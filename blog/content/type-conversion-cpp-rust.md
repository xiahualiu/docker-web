+++
title = "Type Conversions in C++ and Rust"
date = 2024-10-19
draft = false
[taxonomies]
  tags = ["Rust", "C++"]
[extra]
  toc = true
	keywords = "Rust, C++"
+++

Both C++ and Rust provide type conversion mechanism at the language syntax level. But they work differently. C++ allows implicit type conversion, powered by the object constructors, whereas Rust provides `AsRef` and `Deref` for different scenarios.

## C++ Implicit Type Conversion

It is well written in the C++ standards, you can find [here](https://en.cppreference.com/w/cpp/language/implicit_conversion). In general, the implicit conversion will try to construct a rvalue object from the given type to the required type, and the implicit conversion fails if it either didn't find the constructor, or found more than 1 candidate constructors.

There are also extra implicit promotion and narrrow rules for interger and float data types.

Something worth noting is, there are possible conflicts between implicit type conversion and the overload functions.

For example there maybe 2 overload functions with different input signatures, however the compiler finds both input signatures can be satisfied after the implicit type conversions, this will cause the compiler to throw an ambiguity error. 

However C++ also provides a way to disable implicity type conversion, we can take advantage of this fact: template use the exact types for function instatiation. This give us the power to make sure one function only accepts the exact type we want, for example:

```cpp
template<typename T>
T only_double(T f) = delete;

template<>
double only_double<double>(double f) {
    return f;
}
```

So when we call this `only_double` with any parameter type other than `double` will trigger a compilation error.

```cpp
double fd=1.0;
float ff=1.0;
only_double(ff); // error: use of deleted function 'T only_double(T) [with T = float]'
only_double(fd);
```

You can also do explicit type conversion with the help of `static_cast` `reinterpret_cast` and `dynamic_cast`, etc. cast functions.

### Conclusion

In general C++ implicit type conversion is very useful and flexible.

The only problem is that for integer and float types, we don't want these data types changed implicitly without warnings.

There is a compiler flag `-Wnarrowing` but overall this is a big safety issue, especially to inexperienced C++ programmers.

However this type of problem can be found easily with SAST tools like `clangd`, and I highly suggest any serious C++ programmer should use it while doing their projects.

## Rust Type Conversion

Rust provides 2 methods of type conversion that happens during referencing and dereferencing, which are `Deref` and `AsRef`.

* `Deref` allows you to do type cast from reference type to another object type. It is used implicitly when you `*ref`
* `AsRef` needs to be used with `as_ref` keyword. It returns a reference to the original type, but the reference type is not same as the orignal type. `a.as_ref()`

There are also 2 common methods to convert the object type without referencing or dereferencing. These 2 are like `static_cast` in C++.

* `From` trait allows an object to create from another type.
* `Into` trait allows an object to be explicitly converted to another type.

| Trait | Usage |
| :---: | :---: |
| Deref | &A -> &B |
| AsRef | A.as_ref() -> &B |
| From | A::from(B) -> A |
| Into | A.into() -> B |

### Conclusion

In Rust there is no implicit type conversion, all conversions happen only if you have already defined the traits that provides them. And except `Deref`, all the other conversion must be used with their own function names.

It greatly increase the safety of the code, however it is more complicated than C++ and the learning curve is quite steep for these traits.