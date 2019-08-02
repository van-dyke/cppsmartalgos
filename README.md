# Some smart algorithms based on modern C++ semantics

***1. Sum of tuple elements (using variadic templates and std::apply implementations, C++14 version)***

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
***2. Universal callable object***
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

***3. Function composition***

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
