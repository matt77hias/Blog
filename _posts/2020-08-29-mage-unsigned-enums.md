---
layout: post
title:  "🧙 MAGE: Unsigned Enums"
date:   2020-08-29
---

One annoying shortcoming of both scoped (i.e. `enum class`) and unscoped enums is the inability to perform bitwise operations while ensuring the same result type.
For scoped enum types no bitwise definition will be found. For unscoped enum types the result type will be the underlying integral type.

[Example](https://godbolt.org/z/saWaev)
```c++
// same_as
#include <concepts>

enum A : unsigned int
{
    Foo,
    Bar
};

enum class B : unsigned int
{
    Foo,
    Bar
};

int main()
{
    auto unscoped = Foo | Bar;
    static_assert(std::same_as< decltype(unscoped), unsigned int >);

    // error: no match for 'operator|'
    // auto scoped = B::Foo | B::Bar;
    
    return 0;
}
```

Given that enums are typically used to compose flags, this behavior is very inconvenient and will require many explicit cast operators to circumvent.

[Example](https://godbolt.org/z/c6fsej)
```c++
// underlying_type_t
#include <type_traits>

enum A : unsigned int
{
    Foo,
    Bar
};

enum class B : unsigned int
{
    Foo,
    Bar
};

int main()
{
    A unscoped = static_cast< A >(Foo | Bar);

    B scoped = static_cast< B >(static_cast< std::underlying_type_t< B > >(B::Foo)
                              | static_cast< std::underlying_type_t< B > >(B::Bar));
    
    return 0;
}
```

Fortunately, we can explicitly overload the bitwise operators for our enum types to encapsulate this boilerplate.
We will need to provide each overloaded operator for each enum type, though.
Luckily, [SFINAE](https://www.strikerx3.dev/cpp/2019/02/27/typesafe-enum-class-bitmasks-in-cpp.html) can be used to define each overloaded operator for all enum types at once.
