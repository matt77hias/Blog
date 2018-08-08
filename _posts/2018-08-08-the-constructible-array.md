---
layout: post
title:  "The Constructible Array"
date:   2018-08-08
---

## Problem
[MAGE](https://github.com/matt77hias/MAGE)'s (mathematical) vector types have come a long way:

Originally, we used separate template classes for vectors of different dimensions (e.g., 1x2, 1x3 and 1x4) with the primitive type (e.g., `F32`, `U32`, `S32`, etc.) as template parameter, using various [alignments](https://en.cppreference.com/w/cpp/language/alignas) (e.g., alignment of template type, [16-byte boundary](https://docs.microsoft.com/en-us/windows/desktop/direct3dhlsl/dx-graphics-hlsl-packing-rules)). These classes provide both explicit and implicit constructors for constructing vectors from vectors with a different dimension and/or template parameter, and overload common arithemtic and logical operators (e.g., vectors may implicitly be converted to vectors with a larger dimension, `F32` vectors may implicitly be converted to `F64` vectors of the same dimension).

## Adding dimension and alignment

A first generalization consists of combining all these different template classes into a single template class by adding two additional template parameters: one for the dimension (`size_t`) and one for the alignment (`size_t`). The latter can use a default value equal to the alignement of the template parameter (e.g., `alignas(T)`) in case no value is provided by the programmer. This approach supports vectors of arbitrary dimensions similar to the [Tungsten](https://github.com/tunabrain/tungsten/blob/master/src/core/math/Vec.hpp) renderer. Overloaded arithemtic and logical operators will typically require a for loop over the coefficients of the vector for each dimension. Since the dimension template parameter/value is known at compile time, these loops can be unrolled by the compilers.

## Separating container from the arithemtic/logic

A second generalization consists of separating the container functionality from the arithemtic/logical functionality (similar to the [MAML](https://github.com/matt77hias/MAML) project). On the one hand, we can think of our vectors rather as packs of data values without imposing any fixed interpretation. This way, the same packing template class and its interface (e.g., constructors, indexing mechanisms, etc.) can be used to store and access color data (e.g., linear RGB, sRGB, XYZ, LogLUV, etc.), geometrical data (e.g., point, normal, direction, plane, etc.) and abstract mathematical data (e.g., real, complex, dual, hyperbolic and quaternion numbers). On the other hand, arithmetic/logical functionality can be implemented using SIMD instructions for better performance: 

1. Load the vector into an SIMD register (e.g., `__m128`);
2. Perform all arithmetic and logical operations using SIMD intrinsics. 
3. Store the result from an SIMD register in some new vector (e.g., cross product) or primitive values (e.g., dot product). 

In our vector classes, we do not want to use `__m128` member variables (e.g., `union`), since this imposes a 16-byte boundary alignment and thus does not support custom alignments, and since the size of not all our vector class template instantiations is a multiple of sizeof(`__m128`), resulting in wasted memory usage. Alternatively, we do not want to perform separate `__m128` conversions upon starting and finishing every elementary arithemtic and logical member method. To reduce the number of instructions and transfers between registers of different type, and thus to obtain the best performance, data should be kept as long as possible into `__m128` variables. Therefore, we will provide separate functions to load vectors to SIMD registers, to store SIMD registers to vectors/primitive values, and to perform various arithmetic and logical operations (e.g., [DirectXMath](https://github.com/Microsoft/DirectXMath)).

## Extending `std::array`

...

## References

The `Array` was designed for and initially used in [MAGE](https://github.com/matt77hias/MAGE/blob/master/MAGE/Utilities/src/collection/array.hpp#L127).
