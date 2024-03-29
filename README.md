# Some smart algorithms based on modern C++ semantics

# 1. Sum of tuple elements 
***(using variadic templates and std::apply implementations, C++14 version)***

Let's consider case: we have an Tuple and we need to sum up all the components of that Tuple. 
To do that, we need to invoke callable object f with Tuple elements as arguments and sum up their values. In C++17 it's simple - std::apply function is implemented. 
Below, possible implementation of std::apply is proposed for preC++17 compilers...

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <functional>
#include <tuple>

namespace notstd
{
    template <class F, class Tuple, std::size_t... I>
    constexpr decltype(auto) apply_impl(F&& f, Tuple&& t, std::index_sequence<I...>)
    {
        return std::forward<F>(f)( std::get<I>(std::forward<Tuple>(t))... );
    }
     
    template <class F, class Tuple>
    constexpr decltype(auto) apply(F&& f, Tuple&& t)
    {
        return notstd::apply_impl(
            std::forward<F>(f), std::forward<Tuple>(t),
            std::make_index_sequence< std::tuple_size<std::remove_reference_t<Tuple>>::value >{});
    }
};
```

In our examples we are going to use tuples with variadic arguments list. It means that our sum function also depends on variadic parameters list...

```cpp
template<typename T, typename U>
auto mysum(T&& t, U&& u) -> decltype(t+u)
{
    return t + u;
}

template<typename T, typename... U>
auto mysum(T&& t, U&&... u)
{
    return std::forward<T>(t) + mysum(std::forward<U>(u)...);
}
```

And the main...

```cpp
int main()
{
    std::tuple<int, int, float> t1(1, 2, 3.5);   /// Output: 6.5
    std::tuple<int, int, float> t2(1, 2);        /// Output: 3
        
    std::cout <<  notstd::apply( [](auto&&... args) { return mysum(std::forward<decltype(args)>(args)...); }, t1);
    std::cout <<  notstd::apply( [](auto&&... args) { return mysum(std::forward<decltype(args)>(args)...); }, t2);
}
```
***Important note:***
Function ***mysum*** is template function with templated arguments. To be able to deduce type of all variadic arguments we pass to ***apply*** function, we have to use generic lambda (as above) or simple functor wrapper (as below):

```cpp
struct fwrapper
{
   auto operator()(auto&&... args)
   { return mysum(std::forward<decltype(args)>(args)...); }
};
// ...
std::cout <<  notstd::apply(fwrapper(), t);`
```
# 2. Universal callable object
```cpp
template<typename T>
struct Callable
{
    Callable() {};
    
    Callable(T u) : callable(u)
    {};        
    
    void call()
    {
        callable();   
    }
    
    T callable;
};


void fun()
{
    std::cout << "function" << std::endl;   
}


int main()
{
    auto lambda = [](){ std::cout << "lambda" << std::endl; };
    
    std::function<void()> f = [](){ std::cout << "std::function" << std::endl; };
    
    struct Functor
    {
        void operator()(void)
        {   std::cout << "functor" << std::endl; };
    };
        
    Callable<decltype(lambda)> foo(lambda);
    
    Callable<std::function<void()>> boo(f);
    
    Callable<Functor> goo;
    
    Callable<std::function<void()>> zoo(fun);
    
    foo.call();     //Output:   lambda
    boo.call();     //Output:   std::function
    goo.call();     //Output:   functor
    zoo.call();     //Output:   function
    
}

```

# 3. Function composition

```cpp
template<typename R, typename F>
auto compose(R&& r, F&& f)
{
    std::cout << "4: " << __PRETTY_FUNCTION__ << std::endl;
    return [=](auto x){ return r(f(x)); };
}

template<typename R, typename... F>
auto compose(R&& r, F&&... f)
{   
    std::cout << "3: " << __PRETTY_FUNCTION__ << std::endl;
    return [=](auto x){ return r( compose(f...)(x) ); };
}


template<typename R, typename G>
auto operator*(R&& r, G&& g)
{   
    std::cout << "2: " << __PRETTY_FUNCTION__ << std::endl;
    return compose(r, g);
}

template<typename R, typename... G>
auto operator*(R&& r, G&&... g)
{
    std::cout << "1: " << __PRETTY_FUNCTION__ << std::endl;
    return operator*( std::forward<R>(r), std::forward<G...>(g...) );
}

int main()
{
    
    auto l = compose( [](auto x){ return x+x; }, 
                      [](auto x){ return x*x; } )(3); // 3*3, then 9+9
    
    auto r = compose( [](auto x){ return x+x; }, 
                      [](auto x){ return x*x; },                       
                      [](auto x){ return x+1; } )(3); // 3+1, then 4*4, then 16+16
                      
    auto h = ( [](auto x){ return x+x; } * [](auto x){ return x*x; } * [](auto x){ return x+2; } )(3); 
    // 3+2, then 5*5, then 25+25                      
    
    std::cout << l << std::endl;    //Output: 18
    std::cout << r << std::endl;    //Output: 32
    std::cout << h << std::endl;    //Output: 50
}

```

another implementation:

```cpp
template<class Root, class... Branches> 
auto comp(Root &&root, Branches &&... branches) {
     return [root, branches...](auto &&...args) {
         return root(branches(std::forward<decltype(args)>(args)...)...);
     };
}

//...

std::cout << comp([](auto x, auto y){ return x+y; }, [](auto x){ return x+2; }, [](auto x){ return x+1; } )(3) << std::endl;
```

# 4. The Abuse of Variadic Expansion Rules And std::initializer_list

***(taken from https://articles.emptycrate.com/2016/05/14/folds_in_cpp11_ish.html)***

Someone (not me) figured out that we can abuse the std::initializer_list type to let us guarantee the order of execution of some statements while not having to recursively instantiate templates. For this to work we need to create some temporary object to let us use the braced initializer syntax and hold some result values for us.

Only problem is, our print function doesn’t return a result value. So what do we do? Give it a dummy return value!

```cpp
#include <iostream>

template<typename T>
void print(const T &t)
{
  std::cout << t << '\n';
}

template<typename ... T>
  void print(const T& ... t)
{
  std::initializer_list<int>{ (print(t), 0)... };
}

int main()
{
  print(1, 2, 3.4, "Hello World");
}
```
The statement (print(t), 0)... takes advantage of the comma operator to say “execute the first thing then return the second thing.” Even though print(t) returns void, we are giving a 0 to be pushed into the std::initializer_list<int> and the compiler just magically compiles that away. This code ends up being no less efficient than the previous version at runtime and much more efficient at compile time.

# 5. The overload Pattern
***(taken from https://www.bfilipek.com/2019/02/2lines3featuresoverload.html)***

Let’s write a simple type that derives from two base classes:

```cpp
#include <iostream>
#include <string>

struct BaseInt
{
    void Func(int) { std::cout << "BaseInt...\n"; }
};

struct BaseDouble
{
    void Func(double) { std::cout << "BaseDouble...\n"; }
};

struct Derived : public BaseInt, BaseDouble
{
    using BaseInt::Func;        // without: error: request for member 'Func' is ambiguous
    using BaseDouble::Func;
};

int main()
{
    Derived d;
    d.Func(10.0);   // call Func in BaseDouble
    d.Func(5);      // call Func in BaseInt
}
```
We have two bases classes that implement Func. We want to call that method from the derived object. To do that, we have to bring the functions into the scope of the derived class like that:
```cpp
struct Derived : public BaseInt, BaseDouble
{
    using BaseInt::Func;
    using BaseDouble::Func;
};
```
It is possible to ommit this limitation using variadic templates.
```cpp

template <typename T, typename... Ts>
struct Overloader : T, Overloader<Ts...> 
{
    using T::Func;
    using Overloader<Ts...>::Func;
};

template <typename T> 
struct Overloader<T> : T 
{
    using T::Func;
};

```
And now we are able to use ***Overloader*** class like that:
```cpp
    Overloader<BaseInt, BaseDouble> overloader;
    overloader.Func(10.0);  // call Func in BaseDouble
    overloader.Func(5);     // call Func in BaseInt
```

# 6. Visitor pattern with std::variant (C++17)
***( taken from https://www.bfilipek.com/2018/06/variant.html )***

We have a variant that represents a package with four various types, and then we use super-advanced VisitPackage structure to detect what’s inside. The example is also quite interesting as you can invoke a polymorphic operation over a set of classes that are not sharing the same base type.

```cpp
struct Fluid { };
struct LightItem { };
struct HeavyItem { };
struct FragileItem { };

struct VisitPackage
{
    void operator()(Fluid& ) { cout << "fluid\n"; }
    void operator()(LightItem& ) { cout << "light item\n"; }
    void operator()(HeavyItem& ) { cout << "heavy item\n"; }
    void operator()(FragileItem& ) { cout << "fragile\n"; }
};

int main()
{
    std::variant<Fluid, LightItem, HeavyItem, FragileItem> package { FragileItem() };
    std::visit(VisitPackage(), package);
}
```
Output:
***fragile***

We can also use the “overload pattern” from ***5.*** to use several separate lambda expressions:
```cpp
template<class... Ts> struct overload : Ts... { using Ts::operator()...; };
template<class... Ts> overload(Ts...) -> overload<Ts...>;

int main()
{
    std::variant<Fluid, LightItem, HeavyItem, FragileItem> package;

    std::visit(overload{
        [](Fluid& ) { cout << "fluid\n"; },
        [](LightItem& ) { cout << "light item\n"; },
        [](HeavyItem& ) { cout << "heavy item\n"; },
        [](FragileItem& ) { cout << "fragile\n"; }
    }, package);
}
```

But std::visit can accept more variants! It returns the type from that selected overload.

For example we can call it on two packages:
```cpp
std::variant<LightItem, HeavyItem> basicPackA;
std::variant<LightItem, HeavyItem> basicPackB;

std::visit(overload{
    [](LightItem&, LightItem& ) { cout << "2 light items\n"; },
    [](LightItem&, HeavyItem& ) { cout << "light & heavy items\n"; },
    [](HeavyItem&, LightItem& ) { cout << "heavy & light items\n"; },
    [](HeavyItem&, HeavyItem& ) { cout << "2 heavy items\n"; },
}, basicPackA, basicPackB);
```
***Output:***
```cpp
2 light items
```
***remember: Default variant constructor constructs a variant holding the value-initialized value of the first alternative (index() is zero)***

std::visit not only can take many variants, but also those variants might be of a different type.

To illustrate that functionality I came up with the following example:

Let’s say we have an item (fluid, heavy, light or something fragile) and we’d like to match it with a proper box (glass, cardboard, reinforced box, a box with amortization).

In C++17 with variants and std::visit we can try with the following implementation:

```cpp
struct Fluid { };
struct LightItem { };
struct HeavyItem { };
struct FragileItem { };

struct GlassBox { };
struct CardboardBox { };
struct ReinforcedBox { };
struct AmortisedBox { };

variant<Fluid, LightItem, HeavyItem, FragileItem> item { 
    Fluid() };
variant<GlassBox, CardboardBox, ReinforcedBox, AmortisedBox> box { 
    CardboardBox() };

std::visit(overload{
    [](Fluid&, GlassBox& ) { 
        cout << "fluid in a glass box\n"; },
    [](Fluid&, auto ) { 
        cout << "warning! fluid in a wrong container!\n"; },
    [](LightItem&, CardboardBox& ) { 
        cout << "a light item in a cardboard box\n"; },
    [](LightItem&, auto ) { 
        cout << "a light item can be stored in any type of box, "
                "but cardboard is good enough\n"; },
    [](HeavyItem&, ReinforcedBox& ) { 
        cout << "a heavy item in a reinforced box\n"; },
    [](HeavyItem&, auto ) { 
        cout << "warning! a heavy item should be stored "
                "in a reinforced box\n"; },
    [](FragileItem&, AmortisedBox& ) { 
        cout << "fragile item in an amortised box\n"; },
    [](FragileItem&, auto ) { 
        cout << "warning! a fragile item should be stored "
                "in an amortised box\n"; },
}, item, box);
```
Output:
```cpp
warning! fluid in a wrong container!
```

We have four types of items and four types of boxes. We’d like to match the correct box with the item.

std::visit takes two variants item and box and then invokes a proper overload and shows if the types are compatible or not.
The types are very simple, but there’s no problem with extending them and adding features like wight, size or other important members.

In theory, we should write all combinations of overloads: it means 4*4 = 16 functions.
How to Skip Overloads in std::visit?

```cpp
std::variant<int, float, char> v1 { 's' };
std::variant<int, float, char> v2 { 10 };

std::visit(overloaded{
        [](int a, int b) { },
        [](int a, float b) { },
        [](int a, char b) { },
        [](float a, int b) { },
        [](auto a, auto b) { }, // << default!
    }, v1, v2);
```
In the example above you can see that only four overloads have specific types - let’s say those are the “valid” (or “meaningful”) overloads. The rest is handled by generic lambda (available since C++14).

In our main example with items and boxes I use this technique also in a different form. For example:
```cpp
[](FragileItem&, auto ) { 
    cout << "warning! a fragile item should be stored "
            "in an amortised box\n"; },
```
The generic lambda will handle all overloads taking one concrete argument ***FragileItem*** and then the second argument is not “important”.

# 7. std::visit implementation (C++14)
***( taken from https://www.bfilipek.com/2018/06/variant.html )***

In this chapter we would try to write our own implementation of ***std::visit***. This excersise should help us better understand some c++ features.

```cpp
#include <iostream>
#include <string>
#include <functional>
#include <variant>
#include <algorithm>
#include <type_traits>

template<typename T>
using remove_cv_ref = std::remove_cv<typename std::remove_reference<T>::type>;

template<size_t StartIndex, size_t EndIndex>
struct compile_time_loop
{
    template<typename VisitorType, typename VariantType>
    static void visit_visitor(VisitorType&& visitor, VariantType&& vt)
    {
        if constexpr(StartIndex < EndIndex)
        {
            if( auto ptr = std::get_if<StartIndex>(&vt) )   
            {
                std::forward<VisitorType>(visitor)(*ptr);   
                return;
            }
        }

        if constexpr(StartIndex + 1 < EndIndex)
        {
            compile_time_loop<StartIndex+1, EndIndex>::visit_visitor( std::forward<VisitorType>(visitor),          
                std::forward<VariantType>(vt) );
        }

    }
};

template<typename VisitorType, typename VariantType>
void visit(VisitorType&& visitor, VariantType&& vt)
{    
    using variant_t = typename remove_cv_ref<VariantType>::type;
    constexpr std::size_t VariantSize = std::variant_size_v<variant_t>;
    
    compile_time_loop<0, VariantSize>::visit_visitor( std::forward<VisitorType>(visitor), std::forward<VariantType>(vt) );
}
```
We defined helper struct ***compile_time_loop*** which use recursive variadic template mechanism to visit all input variadic  types. Note that we use here ***if constexpr*** semantics which simplify struct calling in each iteration but requires at least C++14 compiler. Each iteration new StartIndex is calculated and pass into template parameter of the struct.

Function ***visit*** calculate input std::variant size and pass it as constexpr expression to ****compile_time_loop*** (as template parameter). 

In below, two lines we created helper code 
```cpp
template<typename T>
using remove_cv_ref = std::remove_cv<typename std::remove_reference<T>::type>;
...
using variant_t = typename remove_cv_ref<VariantType>::type;
```
which remove const and reference qulifiers to provide pure type of used variadic template and pass it into ***std::variant_size_v*** to get its size (count of stored elements)

***Test 1 - struct of override operator()***
```cpp
struct VisitPackage
{
    void operator()(int) { std::cout << "INT\n"; }
    void operator()(float) { std::cout << "FLOAT\n"; }
};

int main()
{
     std::variant<int, float> package { 1.0f };
     visit(VisitPackage(), package);            // Output: FLOAT
}
```
***Test 2 - overload pattern***

Let's create helper struct which override all ***operator()***'s. We want to call ***operator()*** method from the derived object. To do that, we have to bring the functions into the scope of the derived class like that (see 5.): 
```cpp
template<typename... Ts> 
struct overload : Ts... 
{ 
    using Ts::operator()...; 
};

...

auto lambda1 = [](float) { std::cout << "float\n"; };
auto lambda2 = [](int) { std::cout << "int\n"; };
overload<decltype(lambda1), decltype(lambda2)> ov{lambda1, lambda2};    // (***) -> see NOTES below
ov(1.0f);   // Output: float

// or even:
overload<decltype(lambda1), decltype(lambda2)>{lambda1, lambda2}(1); // Output: int
```

Let's addidtional helper code, which create our own type deduction policy. After that we can use ***overload*** helper struct with inline lambda expression

```cpp
template<typename... Ts> 
overload(Ts...) -> overload<Ts...>;

...

overload{ [](float) { std::cout << "float\n"; }, [](int) { std::cout << "int\n"; } }(1);    // Output: int

```
***NOTES!!!***

Only below expression is valid. 
```cpp
overload<decltype(lambda1), decltype(lambda2)> ov{lambda1, lambda2};
```
***Lambda closure do not implement default constructor!!!***
so...
```cpp
overload<decltype(lambda1), decltype(lambda2)> ov{};
```
will generate an error:

***error: use of deleted function 'main()::<lambda(float)>::<lambda>()'***
    
***overload<decltype(lambda1), decltype(lambda2)>{}(1.0f);***

***main.cpp:94:21: note: a lambda closure type has a deleted default constructor***

***auto lambda1 = [](float) { std::cout << "float\n"; };***


# 8. Create a Functor that calls a group of functors in a given inheritance sequence

***( taken from https://stackoverflow.com/questions/38024024/what-does-it-mean-to-inherit-from-lambda )***

[...] because the constructor of the lambda isn't accessible (other than copy/move constructors). The only constructors guaranteed by a lambda type is a defaulted copy/move constructor. And there's no default constructor

The closure type associated with a lambda-expression has no default constructor and a deleted copy assignment operator. It has a defaulted copy constructor and a defaulted move constructor ([class.copy]). [ Note: These special member functions are implicitly defined as usual, and might therefore be defined as deleted. — end note ]

Example:

```cpp
template<typename TFirst, typename... TRemaining>
class FunctionSequence : public TFirst, FunctionSequence<TRemaining...>
{
    public:
    FunctionSequence(TFirst first, TRemaining... remaining)
        : TFirst(first), FunctionSequence<TRemaining...>(remaining...)
    {}

    template<typename... Args>
    decltype(auto) operator () (Args&&... args){
        return FunctionSequence<TRemaining...>::operator()
            (    TFirst::operator()(std::forward<Arg>(args)...)     );
    }
};

template<typename T>
class FunctionSequence<T> : public T
{
    public:
    FunctionSequence(T t) : T(t) {}

    using T::operator();
};


template<typename... T>
auto make_functionSequence(T... t){
    return FunctionSequence<T...>(t...);
}
```

And test

```cpp
int main(){

    //note: these lambda functions are bug ridden. Its just for simplicity here.
    //For correct version, see the one on coliru, read on.
    auto trimLeft = [](std::string& str) -> std::string& { str.erase(0, str.find_first_not_of(' ')); return str; };
    auto trimRight = [](std::string& str) -> std::string& { str.erase(str.find_last_not_of(' ')+1); return str; };
    auto capitalize = [](std::string& str) -> std::string& { for(auto& x : str) x = std::toupper(x); return str; };

    auto trimAndCapitalize = make_functionSequence(trimLeft, trimRight, capitalize);
    std::string str = " what a Hullabaloo     ";

    std::cout << "Before TrimAndCapitalize: str = \"" << str << "\"\n";
    trimAndCapitalize(str);
    std::cout << "After TrimAndCapitalize:  str = \"" << str << "\"\n";

    return 0;
}
```
With output:
***Before TrimAndCapitalize: str = " what a Hullabaloo     "***

***After TrimAndCapitalize:  str = "WHAT A HULLABALOO"***

Another cool trick: Since the resulting type from make_functionSequence(...) is a callable class. You can append more lambda's or callable to it at a later time.

```cpp
//.... As previously seen

    auto trimAndCapitalize = make_functionSequence(trimLeft, trimRight, capitalize);

    auto replace = [](std::string& str) -> std::string& { str.replace(0, 4, "Whaaaaat"); return str; };

    //Add more Functors/lambdas to the original trimAndCapitalize
    auto replaced = make_functionSequence(trimAndCapitalize, replace /*, ... */);
    replaced(str2);
```

# 9. Pretty printing numbers on the fly differently per context on the fly

***Source: "C++17 STL Cookbook" written by Jacek Galowicz***

Class *format_guard* can automatically revert any stream format settings. Its constructor saves the formating flags, which *std::cout* has set at moment. Its destructor restores them to the state it had when constractor was called.

```cpp
#include <iostream>
#include <iomanip>

using namespace std;

class format_guard {
    decltype(cout.flags()) f {cout.flags()};

public:
    ~format_guard() { cout.flags(f); }
};

// *** helper test struct ***
template <typename T>
struct scientific_type {
    T value;

    explicit scientific_type(T val) : value{val} {}
};
// *** helper test struct ***

template <typename T>
ostream& operator<<(ostream &os, const scientific_type<T> &w) {
    format_guard _;
    os << scientific << uppercase << showpos;
    return os << w.value;
}

int main()
{
    {
        format_guard _;
        cout << hex << scientific << showbase << uppercase;

        cout << "Numbers with special formatting:\n";
        cout << 0x123abc << '\n';
        cout << 0.123456789 << '\n';
    }

    cout << "Same numbers, but normal formatting again:\n";
    cout << 0x123abc << '\n';
    cout << 0.123456789 << '\n';

    cout << "Mixed formatting: "
         << 123.0 << " "
         << scientific_type{123.0} << " "
         << 123.456 << '\n';

}
```

# 10. Redirection output to consloe/files for specific code region

***Source: "C++17 STL Cookbook" written by Jacek Galowicz***

We are going to implement helper class that solves the problem of redirecting a stream and reverting that redirection again using RAII concept.
Every stream buffer object has an internal buffer for which it acts as a front end. If we have a stream object *s*, and want to save its buffer into a variable a, and ibstall a new buffer *b*, this looks like following: *a = s.rdbuf(b)*. 
And restoring it can be simple done with *s = rdbuf(a)*.

```cpp
#include <iostream>
#include <fstream>

using namespace std;

class redirect_cout_region {
    using buftype = decltype(cout.rdbuf());

    ofstream ofs;
    buftype buf_backup;

public:
    explicit redirect_cout_region(const string &filename)
        : ofs{filename}, buf_backup{cout.rdbuf(ofs.rdbuf())}
    {}

    redirect_cout_region()
        : ofs{}, buf_backup{cout.rdbuf(ofs.rdbuf())}
    {}

    ~redirect_cout_region() { cout.rdbuf(buf_backup); }
};

void my_output_heavy_function()
{
    cout << "some output\n";
    cout << "this function does really heavy work\n";
    cout << "... and lots of it...\n";
    // ...
}

int main()
{
    cout << "Readable from normal stdout\n";

    {
        redirect_cout_region _ {"output.txt"};
        cout << "Only visible in output.txt\n";
        my_output_heavy_function();
        redirect_cout_region _2 {"fofo.txt"};
        cout << "fofo\n";
    }

    {
        redirect_cout_region _;
        cout << "This output will completely vanish\n";
    }

    cout << "Readable from normal stdout again\n";
}
```

The constructor of class *redirect_cout_region* accepts a filename string as its parameter. The filename is used to initialize the file stream member *ofs*.
After initializing it, we can feed it into *cout* as a new stream buffer. The same function that accepts the new buffer also returns a pointer to the old one, so we can save it in order to restore it later:

```cpp
    
    explicit redirect_cout_region(const string &filename)
        : ofs{filename}, buf_backup{cout.rdbuf(ofs.rdbuf())}
    {}
    
```

The default constructor does the same. The difference is, that it does not open any file. It will just drop its input we give it for printing:

```cpp
   redirect_cout_region()
        : ofs{}, buf_backup{cout.rdbuf(ofs.rdbuf())}
    {}
```

The destructor just restore our change.

    
# 11. CRTP to Avoid Code Duplication

***(taken from: http://www.vishalchovatiya.com/crtp-c-examples/ )***

Let say you have a set of containers that support the functions begin() & end(). But, the standard library’s requirements for containers require more functionalities like front(), back(), size(), etc.
We can design such functionalities with a CRTP base class that provides common utilities solely based on derived class member function i.e. begin() & end() in our cases:

```cpp

template <typename T>
class Container {
    T &actual() { return *static_cast<T *>(this); }
    T const &actual() const { return *static_cast<T const *>(this); }
public:
    decltype(auto) front() { return *actual().begin(); }
    decltype(auto) back() { return *std::prev(actual().end()); }
    decltype(auto) size() const { return std::distance(actual().begin(), actual().end()); }
    decltype(auto) operator[](size_t i) { return *std::next(actual().begin(), i); }
};

```

The above class provides the functions front(), back(), size() and operator[ ] for any subclass that has begin() & end().
For example, subclass could be a simple dynamically allocated array as:

```cpp

template <typename T>
class DynArray : public Container<DynArray<T>> {
    size_t m_size;
    unique_ptr<T[]> m_data;
  public:
    DynArray(size_t s) : m_size{s}, m_data{make_unique<T[]>(s)} {}
    T *begin() { return m_data.get(); }
    const T *begin() const { return m_data.get(); }
    T *end() { return m_data.get() + m_size; }
    const T *end() const { return m_data.get() + m_size; }
};
DynArray<int> arr(10);
arr.front() = 2;
arr[2]        = 5;
asssert(arr.size() == 10);

```
