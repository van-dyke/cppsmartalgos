# Some smart algorithms based on modern C++ semantics

***1. Sum of tuple elements (using variadic templates and std::apply implementations, C++14 version)***

Let's consider case: we have an Tuple and we need to sum up all the components of that Tuple. 
To do that, we need to invoke callable object f with Tuple elements as arguments and sum up their values. In C++17 it's simple - std::apply function is implemented. 
Below, possible implementation of std::apply is proposed...

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
