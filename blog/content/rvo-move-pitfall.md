+++
title = "RVO (Return Value Optimization) and deleted move constructors pitfall."
date = 2025-01-14
draft = false
[taxonomies]
  tags = ["Linux"]
[extra]
  toc = true
	keywords = "C++"
+++

## The Surprising Case

```cpp
// filepath: rvo_example.cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) = default;      // Copy constructor
    Widget(Widget&&) = delete;            // Deleted move constructor
    
    int value{42};
};

Widget createWidget() {
    Widget w;
    return w;  // Error: call to deleted move constructor
}
```

## Why This Happens

Based on the C++ standards [here](https://en.cppreference.com/w/cpp/language/copy_elision#:~:text=even%20when%20it%20takes%20place%20and%20the%20copy/move(since%20C%2B%2B11)%20constructor%20is%20not%20called%2C%20it%20still%20must%20be%20present%20and%20accessible%20(as%20if%20no%20optimization%20happened%20at%20all)%2C%20otherwise%20the%20program%20is%20ill%2Dformed%3A):

> Under the following circumstances, the compilers are permitted, but not required to omit the copy and move(since C++11) construction of class objects even if the copy/move(since C++11) constructor and the destructor have observable side-effects. The objects are constructed directly into the storage where they would otherwise be copied/moved to. This is an optimization: **even when it takes place and the copy/move(since C++11) constructor is not called, it still must be present and accessible (as if no optimization happened at all), otherwise the program is ill-formed**:

And because of [this](https://en.cppreference.com/w/cpp/language/copy_elision#:~:text=In%20a%20return%20statement%20or,see%20return%20statement%20for%20details.):

>In a return statement or a throw expression, if the compiler cannot perform copy elision but the conditions for copy elision are met, or would be met except that the source is a function parameter, the compiler will attempt to use the move constructor even if the source operand is designated by an lvalue(until C++23) the source operand will be treated as an rvalue(since C++23); see return statement for details.

### In human words

* RVO applied or not, compiler needs valid move/copy constructors.
* Deleted move constructor is present but invalid.
* Compiler tries move before copy.

### Solution

The solution is simple, we can remove the `= delete` declaration, and because `Widget` has a copy constructor, [the move constructor will not be implicitedly declared](https://en.cppreference.com/w/cpp/language/move_constructor). So the compiler will not use the move constructor of `Widget` in this case, because it is not **present**, and will use the copy constructor instead.

```cpp
// filepath: solution.cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) = default;
    // Remove the deleted move constructor
    // Let compiler handle move operations
    
    int value{42};
};

// Now works fine - uses copy constructor when NRVO not applied
Widget createWidget() {
    Widget w;
    return w;
}
```

### Other notes : Avoid side effects in copy/move constructors

Copy elison allows the compiler to omit the copy/move constructor while returning an object, this means if the copy/move constructor has side effects, the compiler is allowed to ignore those side effects when copy elison happens.

This means in general your C++ code should not have copy/move constructor with side effects, such as the `shared_ptr` copy constructor. Otherwise the behavior will be undefined.
