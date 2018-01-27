---
layout: post
title:  "C++ Wishlist"
date:   2018-27-01
categories: c++
---

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

Lets assume that we want to convert 32-bit integral values (unsigned integer, signed integer, floating point) to the corresponding 64-bit integral values. 
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

None of these three versions looks very readable nor pleasing at all.
If we stick to SFINAE, we will probably have to wait until C++20 which will introduce the `requires` keyword.

Alternatively use C++17's [`if constexpr`](http://en.cppreference.com/w/cpp/language/if):
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
inline Foo &Create< Foo >(ConstructorArgsT &&...args) {
    return g_foos.emplace_back(std::forward< ConstructorArgsT >(args)...);
}

template< typename... ConstructorArgsT >
inline Bar &Create< Bar >(ConstructorArgsT &&...args) {
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





