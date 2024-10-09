+++
title = "Perfect forwarding in C++."
date = 2024-08-28
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
	keywords = "Code, C++, Template"
+++

Perfect forwarding comes from `std::forward()` is one of the special C++ technique to reduce the code length.

More information can be found on [cppreference.com](https://en.cppreference.com/w/cpp/utility/forward).

## How it works

Basically it forwards a rvalue reference to the nested function inside a function template.

```cpp
template<class T>
void wrapper(T&& arg)
{
    // arg is always lvalue
    foo(std::forward<T>(arg)); // Forward as lvalue or as rvalue, depending on T
}
```

* If `T` is a lvalue reference, `std::forward<T>()` does nothing, `arg` will be passed as lvalue reference as normal.
* If `T` is a rvalue reference, since `arg` is treated as an lvalue in the body of the function, `std::forward<T>()` converts it back to rvalue reference.

For both cases, `foo()`, which takes an rvalue parameter OR lvalue reference, will work correctly.

## Difference from `std::move()`

The difference from `std::move()` is that, `std::forward<T>()` is conventionally used in function template only.

* For case 1 above, `std::forward<T>()` is no-op, it doesn't do anything.
* For case 2 above, `std::forward<T>()` is same as `std::move()` here.

You can think `std::forward<T>()` is a smarter version of `std::move()`, however it requires a typename to work.
