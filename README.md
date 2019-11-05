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

# 5a. The overload Pattern
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
