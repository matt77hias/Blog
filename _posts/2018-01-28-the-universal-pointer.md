---
layout: post
title:  "The universal pointer"
date:   2018-01-28
categories: c++
---

WIP

## Problem

I want to apply an [Entity–Component–System](https://en.wikipedia.org/wiki/Entity%E2%80%93component%E2%80%93system) architecture to my game engine.
My scene data includes a dynamic array encapsulating a contiguous block of memory (`std::vector`) for each component type that stores components by value.
That way, I can exploit cache coherence and avoid unnecessary jumps between unrelated memory.
This design imposes two immediate problems that need to be handled properly:
* How do you avoid invalidation of pointers and references when components are deleted from their respective collections? 
Assume a `std::vector` contains three elements. 
If we invoke `std::vector::erase` on the vector by passing an iterator to the second element, 
all the following elements need to be moved (via the move assignment operator) to a lower index to avoid fragmentation.
Thus, the third element will become the second element. The implementation of `std::vector` will handle this for us. 
All pointers and reference to the original third element, will become invalid (i.e. they will not point or refer anymore to the correct element). 
* How do you avoid invalidation of pointers and references when components are added to their respective collections?
One would expect that adding elements to a `std::vector` would not invalidate pointers or references, since elements will always stay at the same index. 
`std::vector`, however, manages internally a contiguous block of memory of a finite size, when the size is too small to fit the required number of elements, 
a new block of memory is allocated and all elements will be moved (via in-place construction with the move constructor). 
So if our `std::vector` contains three elements and has a capacity of three elements, adding a fourth will invalidate all pointers and references to elements of this collection.

The first problem can be solved easily by not allowing the removal of elements from a `std::vector`. 
If you want to destroy an element, the element is marked `terminated`. 
If you want to add an element, the elements are iterated until the first `terminated` element is encountered. 
That element will be replaced by the new element. If no `terminated` elements are encountered, a new element is added to the back.
This way elements will always reside at the same index. 
*(If it must be possible to remove elements, you can still use the approach below by adding some extra bookkeeping.)*

The second problem can be solved by using a `std::array` of `std::vector` which not will be resized. 
If we know the maximum number of elements, we can assume worst case behavior.

The first and second problem can be solved together by using the index of elements as a handle instead of a pointer or reference to the element.
This approach is prone to error, since the index itself has no clue of the `std::vector` it belongs to. 
Ok, so why not wrap the index and `std::vector` in some handle class, then we get even the type of element for free.

But what if we use other collections as well? 
If we just encapsulate the collection and "index", we will obtain various different handle classes.
What if we need to store such handles in the same collection? 
Should we create a common abstract base class on top that provides an abstract function to obtain the raw pointer or reference?
This virtual method will certainly incur some additional cost at runtime.

There is an alternative. We can encapsulate all the data we need inside a [`std::function`](http://en.cppreference.com/w/cpp/utility/functional/function).

## ProxyPtr














