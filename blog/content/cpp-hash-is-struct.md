+++
title = "Why std::hash is a struct"
date = 2024-08-28
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
	keywords = "Code, C++"
+++

If you check the C++ standard you will find that `std::hash` is a struct instead of a function. Reference is [here](https://en.cppreference.com/w/cpp/utility/hash).

It has the `()` operator overloaded so that we can use it just like a function (`std::size_t std::hash()`).

## Why it is a struct?

### Store meta information

I think the initial reason (before C++17) is that, struct can store meta information such as types for static deduction unlike function.

Before C++17, you can do:

```cpp
std::hash<T>::argument_type // T type
std::hash<T>::result_type // std::size_t type
```

After C++17 you can use `decltype(T)` and `decltype(std::hash<T>())` so these types are no longer needed.

### Salted hash

Because it is a template struct, you can add member varibles in your type specification, such as seed value to enable salted hash.

Even the default specification for all `std` data types are stateless, you can and hash states as well for your own application.

Salted hash also requires the `std::hash` to be created as a discret object to store the internal seed value, instead of being a conceptual function and never created in the software.

## Conclusion

`std::hash` is designed to be a struct so that it gives the user more choice when designing the hash steps.