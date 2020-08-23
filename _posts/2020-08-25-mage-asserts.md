---
layout: post
title:  "MAGE: Asserts"
date:   2020-08-25
---

While browsing Microsoft's public [STL](https://github.com/microsoft/STL) implementation, I noticed to my surprise that the [`assert`](https://en.cppreference.com/w/cpp/error/assert) macro ([`<cassert>`](https://en.cppreference.com/w/cpp/header/cassert)) can be evaluated within a constant-evaluated context (e.g., compile-time evaluation). Due to the [Relaxing constraints on constexpr functions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3652.html), it becomes possible to use [`assert`](https://en.cppreference.com/w/cpp/error/assert) in `constexpr` functions in C++14 (some [custom assert workarounds](http://ericniebler.com/2014/09/27/assert-and-constexpr-in-cxx11/) are possible in C++11). This is great! On the one hand, more and more functions could be made `constexpr` as newer C++ standards tend to further relax the constraints and tend to make the already existing functionality of the standard library more `constexpr` as well. On the other hand, my mostly used defensive strategy to validate program correctness depends on asserts checking invariants, pre- and post conditions (as opposed to total and defensive programming).

Unfortunately, MAGE already uses custom asserts (`MAGE_ASSERT`) with logging support through [spdlog](https://github.com/gabime/spdlog), which could not be used in `constexpr` functions. Replacing these with [`assert`](https://en.cppreference.com/w/cpp/error/assert) in `constexpr` functions only is not a viable solution:
* having different assert macros depending on the evaluation context is confusing;
* when [`assert`](https://en.cppreference.com/w/cpp/error/assert) is not evaluated at compile time, our custom logger is bypassed, since `constexpr` functions are not guaranteed to be evaluated at compile time;
* `MAGE_ASSERT` is implemented using `MAGE_ENSURE`, which guarantees to evaluate the given expression independent of the configuration (i.e. [`NDEBUG`](https://en.cppreference.com/w/c/error/assert)) compared to [`assert`](https://en.cppreference.com/w/cpp/error/assert). This allows for more compact code when validating error codes as these do not need to be assigned anymore to potentially unused local variables.

```c++
//-----------------------------------------------------------------------------
// Engine Defines: Ensure
//-----------------------------------------------------------------------------

// The expression is guaranteed to be evaluated in all configurations.

#define MAGE_ENSURE(expression, ...)                                          \
	do                                                                        \
	{                                                                         \
		if ((expression) == false)                                            \
		{                                                                     \
			::mage::details::LogAssert(#expression,                           \
									   MAGE_SOURCE_LOCATION,                  \
									   __VA_ARGS__);                          \
			MAGE_BREAK;                                                       \
		}                                                                     \
	}                                                                         \
	while(false)
	
//-----------------------------------------------------------------------------
// Engine Defines: Assert
//-----------------------------------------------------------------------------

// The expression is not guaranteed to be evaluated in all configurations.
	
#ifdef MAGE_DEBUG
	#define MAGE_ASSERT MAGE_ENSURE
#else  // MAGE_DEBUG
	#define MAGE_ASSERT MAGE_UNEVALUATED_EXPRESSION
#endif // MAGE_DEBUG
```

Fortunately, C++20 added [std::is_constant_evaluated](https://en.cppreference.com/w/cpp/types/is_constant_evaluated) for the purpose of querying whether a function call occurs within a constant-evaluated context, allowing us to refine the previous macro definitions.

```c++
//-----------------------------------------------------------------------------
// Engine Defines: Ensure
//-----------------------------------------------------------------------------

// The expression is guaranteed to be evaluated in all configurations.

#define MAGE_ENSURE(expression, ...)                                          \
	do                                                                        \
	{                                                                         \
		if ((expression) == false) [[unlikely]]                               \
		{                                                                     \
			if (std::is_constant_evaluated())                                 \
			{                                                                 \
				???;                                                          \
			}                                                                 \
			else                                                              \
			{                                                                 \
				::mage::details::LogAssert(#expression,                       \
										   MAGE_SOURCE_LOCATION,              \
										   __VA_ARGS__);                      \
				MAGE_BREAK;                                                   \
			}                                                                 \
		}                                                                     \
	}                                                                         \
	while(false)
```

So what could our custom assert do inside the constant-evaluated context? We cannot use [`assert`](https://en.cppreference.com/w/cpp/error/assert), as Microsoft's STL for example defines it as:

```c++
#ifdef NDEBUG

    #define assert(expression) ((void)0)

#else

    _ACRTIMP void __cdecl _wassert(
        _In_z_ wchar_t const* _Message,
        _In_z_ wchar_t const* _File,
        _In_   unsigned       _Line
        );

    #define assert(expression) (void)(                                                       \
            (!!(expression)) ||                                                              \
            (_wassert(_CRT_WIDE(#expression), _CRT_WIDE(__FILE__), (unsigned)(__LINE__)), 0) \
        )

#endif
```

It is not possible to define the [`NDEBUG`](https://en.cppreference.com/w/c/error/assert) symbol, include [`<cassert>`](https://en.cppreference.com/w/cpp/header/cassert), define `MAGE_ENSURE` using [`assert`](https://en.cppreference.com/w/cpp/error/assert) and undefine the [`NDEBUG`](https://en.cppreference.com/w/c/error/assert) symbol, as `MAGE_ENSURE` itself is a macro definition that will be require the [`NDEBUG`](https://en.cppreference.com/w/c/error/assert) symbol to be defined upon expansion. Furthermore, [`assert`](https://en.cppreference.com/w/cpp/error/assert) does not support [`std::string_view`](https://en.cppreference.com/w/cpp/string/basic_string_view), but requires null-terminated strings.

It is not possible to use [`static_assert`](https://en.cppreference.com/w/cpp/language/static_assert), as that cannot evaluate expressions based on function parameters.

The trick consists of ignoring any logging. For expressions that are not evaluated at compile-time, but at runtime instead, we want to redirect the assert messages to our logger. At compile-time itself, we can obviously not use that same logger. But should we use a logger or output at all? As long as we guarantee the compilation to fail, the compiler will provide us some information about the cause without having us to customize the message. The question then becomes how can we let the compilation fail within a constant-evaluated context? Well, we can try to evaluate something that could never be evaluated within such a context. There are many possible combinations, but my personal favorite is [`std::abort`](https://en.cppreference.com/w/cpp/utility/program/abort).

```c++
//-----------------------------------------------------------------------------
// Engine Defines: Ensure
//-----------------------------------------------------------------------------

// The expression is guaranteed to be evaluated in all configurations.

#define MAGE_ENSURE(expression, ...)                                          \
	do                                                                        \
	{                                                                         \
		if ((expression) == false) [[unlikely]]                               \
		{                                                                     \
			if (std::is_constant_evaluated())                                 \
			{                                                                 \
				std::abort();                                                 \
			}                                                                 \
			else                                                              \
			{                                                                 \
				::mage::details::LogAssert(#expression,                       \
										   MAGE_SOURCE_LOCATION,              \
										   __VA_ARGS__);                      \
				MAGE_BREAK;                                                   \
			}                                                                 \
		}                                                                     \
	}                                                                         \
	while(false)

//-----------------------------------------------------------------------------
// Engine Defines: Assert
//-----------------------------------------------------------------------------

// The expression is not guaranteed to be evaluated in all configurations.

#ifdef MAGE_DEBUG
	#define MAGE_ASSERT MAGE_ENSURE
#else  // MAGE_DEBUG
	#define MAGE_ASSERT MAGE_UNEVALUATED_EXPRESSION
#endif // MAGE_DEBUG
```
