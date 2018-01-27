---
layout: post
title:  "C++ Wishlist"
date:   2018-01-27
categories: c++
---

This post contains a running list of C++ language features and standard library (`std`) extensions, I would like to see in future C++ standards (C++20 or beyond).

## Full and partial template specilization for alias templates

[Alias templates](http://en.cppreference.com/w/cpp/language/type_alias) are introduced in C++11. 
They basically extent [typedefs](http://en.cppreference.com/w/cpp/language/typedef) 
(which are equivalent to [type aliases](http://en.cppreference.com/w/cpp/language/type_alias)) by adding a template parameter list.

```c++
// Type alias
// This is equivalent to the less intuitive:
// typedef unsigned int uint
using uint = unsigned int;

// Alias template
template< typename T >
using Vector = std::vector< T >;
```

Suppose we want to specialize the return type of a template function based on the template parameter, 
we somehow need to map template parameters to return types. To allow arbitrary mappings, template specialization is required.

Lets assume that we want to convert 32-bit numeric values (unsigned integer, signed integer, floating point) to the corresponding 64-bit numeric values. 
In order to specify this mapping in C++, we can use a class template that will be specialized for each element of the mapping:

```c++
#include <stdint.h>

//  int32_t maps to  int64_t
// uint32_t maps to uint64_t
//    float maps to double

template< typename T >
struct ReturnType {
    using type = T;
};

template<>
struct ReturnType< int32_t > {
    using type = int64_t;
};

template<>
struct ReturnType< uint32_t > {
    using type = uint64_t;
};

template<>
struct ReturnType< float > {
    using type = double;
};

template< typename T >
using return_t = typename ReturnType< T >::type;

template< typename T >
return_t< T > convert(T value) {
    return return_t< T >(value);
}
```

*For a more advanced example, see [resource_manager.hpp](https://github.com/matt77hias/MAGE/blob/master/MAGE/MAGE/src/resource/resource_manager.hpp), 
containing a resource manager that manages mutable/immutable resources of different types through different resource pools of different types via one interface for all resources.*

This code seems pretty verbose for what it actually tries to achieve (e.g. why do we need structs for type mappings?), making our intentions less clear for fellow programmers. 
Ideally, we would like to write something like this:

```c++
#include <stdint.h>

//  int32_t maps to  int64_t
// uint32_t maps to uint64_t
//    float maps to double

template< typename T >
using return_t = T;

template<>
using return_t< int32_t > = int64_t;

template<>
using return_t< uint32_t > = uint64_t;

template<>
using return_t< float > = double;

template< typename T >
return_t< T > convert(T value) {
    return return_t< T >(value);
}
```

Unfortunately, (full) specialization of alias templates is not permitted in C++ (<= C++17). 
This also implies that partial specialization of alias templates is not permitted either.

**Therefore, I like to be able to define full and partial template specializations of alias templates in future C++.**

## Partial template specilization for functions and member methods

Assume that we want to create resources in a uniform way by calling a function `Create` 
with a template parameter matching the resource we want to create and 
with a list of arguments matching the constructor of the resource we want to invoke.

If we use [SFINAE](http://en.cppreference.com/w/cpp/language/sfinae) on the return type, we can write the following code:

* **C++11**
```c++
#include <type_traits>
#include <vector>

struct Foo {
    // Contains various constructors
};
struct Bar {
    // Contains various constructors
};

std::vector< Foo > g_foos;
std::vector< Bar > g_bars;

template< typename ResourceT, typename... ConstructorArgsT >
typename std::enable_if< std::is_same< Foo, ResourceT >::value, ResourceT & >::type 
    Create(ConstructorArgsT &&...args) {
    
    g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
    return *g_foos.end();
}

template< typename ResourceT, typename... ConstructorArgsT >
typename std::enable_if< std::is_same< Bar, ResourceT >::value, ResourceT & >::type 
    Create(ConstructorArgsT &&...args) {
    
    g_bars.emplace_back(std::forward< ConstructorArgsT >(args)...);
    return *g_bars.end();
}

int main() {
    auto &foo = Create< Foo >();
    auto &bar = Create< Bar >();
}
```

* **C++14** (using [`std::enable_if_t`](http://en.cppreference.com/w/cpp/types/enable_if))
```c++
#include <type_traits>
#include <vector>

struct Foo {
    // Contains various constructors
};
struct Bar {
    // Contains various constructors
};

std::vector< Foo > g_foos;
std::vector< Bar > g_bars;

template< typename ResourceT, typename... ConstructorArgsT >
std::enable_if_t< std::is_same< Foo, ResourceT >::value, ResourceT & > 
    Create(ConstructorArgsT &&...args) {
    
    g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
    return *g_foos.end();
}

template< typename ResourceT, typename... ConstructorArgsT >
std::enable_if_t< std::is_same< Bar, ResourceT >::value, ResourceT & > 
    Create(ConstructorArgsT &&...args) {
    
    g_bars.emplace_back(std::forward< ConstructorArgsT >(args)...);
    return *g_bars.end();
}

int main() {
    auto &foo = Create< Foo >();
    auto &bar = Create< Bar >();
}
```

* **C++17** (using the new [`std::vector::emplace_back`](http://en.cppreference.com/w/cpp/container/vector/emplace_back) and [`std::is_same_v`](http://en.cppreference.com/w/cpp/types/is_same))
```c++
#include <type_traits>
#include <vector>

struct Foo {
    // Contains various constructors
};
struct Bar {
    // Contains various constructors
};

std::vector< Foo > g_foos;
std::vector< Bar > g_bars;

template< typename ResourceT, typename... ConstructorArgsT >
std::enable_if_t< std::is_same_v< Foo, ResourceT >, ResourceT & > 
    Create(ConstructorArgsT &&...args) {
    
    return g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
}

template< typename ResourceT, typename... ConstructorArgsT >
std::enable_if_t< std::is_same_v< Bar, ResourceT >, ResourceT & > 
    Create(ConstructorArgsT &&...args) {
    
    return g_bars.emplace_back(std::forward< ConstructorArgsT >(args)...);
}

int main() {
    auto &foo = Create< Foo >();
    auto &bar = Create< Bar >();
}
```
Note that multiple type traits exist in C++. 
For example: if you rather want collections of pointers instead of values to exploit polymorphism, 
you can use [`std::base_of`](http://en.cppreference.com/w/cpp/types/is_base_of) instead of `std::is_same`.

None of these three versions look very readable or pleasing at all.
If we stick to SFINAE, we will probably have to wait until C++20 which will introduce the `requires` keyword.

Alternatively, we can use C++17's [`if constexpr`](http://en.cppreference.com/w/cpp/language/if):
```c++
#include <type_traits>
#include <vector>

struct Foo {
    // Contains various constructors
};
struct Bar {
    // Contains various constructors
};

std::vector< Foo > g_foos;
std::vector< Bar > g_bars;

template< typename ResourceT, typename... ConstructorArgsT >
ResourceT &Create(ConstructorArgsT &&...args) {
    if constexpr (std::is_same_v< Foo, ResourceT >) {
        return g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
    } 
    else if constexpr (std::is_same_v< Bar, ResourceT >) {
        return g_bars.emplace_back(std::forward< ConstructorArgsT >(args)...);
    } 
}

int main() {
    auto &foo = Create< Foo >();
    auto &bar = Create< Bar >();
}
```
This looks much compacter, but unfortunately requires us to know all resource types in advance which will not always be the case.

Ideally, we would like to write something like this:
```c++
#include <type_traits>
#include <vector>

struct Foo {
    // Contains various constructors
};
struct Bar {
    // Contains various constructors
};

std::vector< Foo > g_foos;
std::vector< Bar > g_bars;

template< typename ResourceT, typename... ConstructorArgsT >
ResourceT &Create(ConstructorArgsT &&...args);

template< typename... ConstructorArgsT >
inline Foo &Create(ConstructorArgsT &&...args) {
    return g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
}

template< typename... ConstructorArgsT >
inline Bar &Create(ConstructorArgsT &&...args) {
    return g_bars.emplace_back(std::forward< ConstructorArgsT >(args)...);
}

int main() {
    auto &foo = Create< Foo >();
    auto &bar = Create< Bar >();
}
```

Unfortunately, partial specialization of template functions is not permitted in C++ (<= C++17). 
This is also the case when we want to encapsulate our resource collections inside a class: 
partial specialization of member methods is not permitted in C++ (<= C++17). 

**Therefore, I like to be able to define partial template specializations of template functions and member methods in future C++.**

## Anonymous structs
C11 added anonymous unions (originally only a GNU extension) and anonymous structs to the C standard.
C++98 (the first C++ standard) includes anonymous unions but not anonymous structs.

Though, anonymous structs work out-of-the-box with MVC++, gcc and Clang for all C++ standards and are ubiquitous in APIs aiming at both a C and C++ audience such as the Windows API (try to disable C++ language extensions in the compiler and see for yourself). 
So the only reason I can imagine for not adding anonymous structs to the C++ standard, is that one simply forgot that this language feature is non-standard.

**Therefore, I like to be able to define anonymous structs in future C++.**

## Integral prefixes

There exist no integral suffix for (un)signed chars and (un)signed shorts. 
Therefore, (implicit/explicit) casts are required to initialize these types:
```c++
int main() {
    auto a = 1;    //   signed int
    auto b = 2u;   // unsigned int
    auto c = 3l;   //   signed long
    auto d = 4ul;  // unsigned long
    auto e = 5ll;  //   signed long long
    auto f = 6ull; // unsigned long long
    auto g = 7.0f; // float
    auto h = 8.0;  // double
    // Note that we can use capitals as well.
    
    auto i = static_cast<   signed short >(9);
    auto j = static_cast< unsigned short >(10);
    auto k = static_cast<   signed char  >(11);
    auto l = static_cast< unsigned char  >(12);
}
```
This can become quite verbose when using simple arithmetic functions:
```
#include <algorithm>

int main() {
    auto value  = static_cast< signed short >(9); // Assume that this value is not known at compile time...
    auto result = std::max(i, static_cast< signed short >(5));
}
```

**Therefore, I like to be able to define (un)signed chars and (un)signed shorts using prefixes in future C++.**

## `noexcept` move constructor and assignment operators in std
Move constructors and move assignment operators should be declared `noexcept`, especially in the `std`.
Unfortunately, this is not true for all `std` classes:
* [std::function](http://en.cppreference.com/w/cpp/utility/functional/function)
* [std::map](http://en.cppreference.com/w/cpp/container/map/map), [std::unordered_map](http://en.cppreference.com/w/cpp/container/unordered_map/unordered_map), [std::multimap](http://en.cppreference.com/w/cpp/container/multimap/multimap), [std::unordered_multimap](http://en.cppreference.com/w/cpp/container/unordered_multimap/unordered_multimap)
* ...

## Remove and replace `[[nodiscard]]` with `[[maybe_discard]]`.
I add C++17's `[[nodiscard]]` attribute everywhere it makes sense. For functions returning error codes, the returned value should be used, since I return these codes for a reason and do not intent or allow the continuation of the program without handling them appropriately (where the user may use his preferred programming style: total, normal, defensive, etc.). For functions returning values that could leak resources (allocators) or could break the goal (async) in case of not using them, the returned values should be used.  

Most of the remaining functions or member methods (i.e. getters) that return a value are pure or nearly pure (if no exceptions are thrown or assertions are failed). This last category comprises more than 90% of my codebase. Calling these functions without using the returned value makes no sense and should be dealt with to obtain proper code. So in that sense, I like to be warned or even receive an error in such cases. On the other hand, interfaces become quite verbose since you will nearly see the attribute once in every two functions. 

Ideally C++17 should have broken backwards compatibility by adding the opposite keyword [[maybe_discarded]]. Note that this is not strictly breaking backwards compatibility, but merely adds extra warnings to existing codebases (compilers will always become better at analyzing code, so you should expect more warnings anyway).

**Therefore, I like to be able to use a [[maybe_discarded]] instead of [[nodiscard]] attribute in future C++.**

## CPU clock
I really like the [`#include <chrono>`](http://en.cppreference.com/w/cpp/chrono) and especially the way it handles different storage types (e.g. integer or floating point) with varying degrees of precision. The available clocks can only handle wall clock time. I would like to handle kernel and/or user mode time as well, since these are more accurate. *(If you have a Windows operating system, I encourage you to compare the difference for yourself with this [project](https://github.com/matt77hias/Timing/blob/master/Timing/Timing/src/Timing.cpp). More particularly, compare the results obtained after running the program once versus twice at the same time.)*

It would be handy to have a clock that handles kernel and/or user mode timestamps in the `std`, since it could be used on all platforms. The surprising thing is that C++ already has such a clock from its C legacy: [`clock`](http://www.cplusplus.com/reference/ctime/clock/). 

> Returns the processor time consumed by the program.

Unfortunately, there is a rather large caveat when looking at the documentation of Microsoft's implementation of [`clock`](https://msdn.microsoft.com/en-us/library/4e2ess30.aspx):

> The clock function tells how much **wall-clock time** has passed since the CRT initialization during process start. Note that this function does not strictly conform to ISO C, which specifies net CPU time as the return value. To obtain CPU times, use the Win32 [`GetProcessTimes`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683223(v=vs.85).aspx) function.

The aforementioned `GetProcessTimes` is, as one would guess, only available for Windows operating systems. The `std` leaves us thus empty-handed. But even if C's `clock` worked the way it is supposed to on all platforms, we still need a clock inside `<chrono>` with an interface similar to the other clocks (`std::chrono::system_clock`, `std::chrono::steady_clock`, `std::chrono::high_resolution_clock`).
