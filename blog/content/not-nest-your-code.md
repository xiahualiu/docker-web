
+++
title = "How to NOT Nest Your Code"
date = 2024-05-10
draft = false
[taxonomies]
  tags = ["Design Patterns", "C++"]
[extra]
  toc = true
	keywords = "Code, C++, Design Patterns, Nest"
+++

At the beginning of the [Linux kernel coding style](https://www.kernel.org/doc/html/v4.10/process/coding-style.html#:~:text=if%20you%20need%20more%20than%203%20levels%20of%20indentation%2C%20you%E2%80%99re%20screwed%20anyway%2C%20and%20should%20fix%20your%20program.) document, it points out that:

> if you need more than 3 levels of indentation, you’re screwed anyway, and should fix your program.

Code nesting exists in almost all programming languages, but code with deep nested structure can soon become very unreadable. Have you ever met this problem and wondered how can I write code that does not nest? Then this article can help! I am going to show you some design patterns that can be used to avoid nesting.

## Table-Driven Methods

Sometimes it is easier to use a look up table (`switch-case`) rather than implementing the complicated `if-else` statements. For some languages such as Python which does not have the `switch-case` syntax, the programmer needs to create a table object to achieve the same goal.

For example we want to write a function that takes an integer input `n` (from 1 to 12) and returns the number of days in the `n`th month. You can easily write function with `if-else` statements, but a better way is to use a look up table instead.

```cpp
int getDaysOfTheMonth(const int n)
{
	int table[12] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
	return table[n-1];
}
```

Of course to make it really work we need to check for leap years and input range checks, the code will look like:

```cpp
int getDaysOfTheMonth(const int n, const int year)
{
	if (n > 12 || n < 1 || year < 0)
		throw std::runtime_error("Input out of range!");
	int isLeapYear = (year % 4 == 0) && (year % 100 != 0 || year % 400 == 0) ? 1 : 0;
	int table[12] = {31, 28 + isLeapYear(year), 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
	return table[n-1];
}
```

The table can be other types, here we only used a bin-bucket hash table. You can use any hash tables as well, like in C++ `std::map` or `std::unordered_map`, in Python `dict` object.

## Early Return

Early return requires a function to return (or throw exceptions) as early as possible if a certain situation is met.

For some languages like Rust you can use things like `unwrap` or `assert` functions to throw the exception easily. 

This is easy to understand since the earlier a function returns the fewer control flow branches it needs to deal with, which as a result, reduce the code nesting.

## State Design Pattern

C++ uses vtable(virtual functions) to achieve runtime polymorphism, and it acts exactly like a `switch-case` or look up table we saw above.

So we can use this feature, to hide the `switch-case` style functions by overriding the functions in different sub classes.

The most known design pattern for this is called **State** pattern, which can be found [here](https://refactoring.guru/design-patterns/state).

Let's assume you need to write the function for a ceiling fan, the user can input up/down and on/off to control the speed.

When the fan is the OFF, of course the user input does nothing and the speed is zero. But if the fan is ON, the user can control the speed of the fan.

With the state design pattern, you can write the code like [this](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGe1wAyeAyYAHI%2BAEaYxHoADqgKhE4MHt6%2BcQlJAkEh4SxRMVy2mPaOAkIETMQEqT5%2BRXaYDskVVQQ5YZHRegqV1bXpDX3twZ353VwAlLaoXsTI7BwmGgCC5gDMwcjeWADUJutuTr3EmKwH2Mtrq9tMCgq7AGKGHoKYqgRXJgDsVqu7AN2VFoqCYBF2ClimEw6AOf2uK0Bu1iXgitDwyBAVyRADdUHh0Lstqc7pghFCYdQQWDdkxJvtfrsCAg8AoALQXSHQwnWA4AEVpcIZfOxgLxBN2WGQJKU5O5VNB4LpDIsTJZ7M5FMJHPWAqYQp%2BIv%2BYvxhKUBAAWtFUHLKfSfqrmayddguTD9rrdhoAHQaA3fI2Ik0S2LEYLCLUQe2M3roEAoWbgg5uZP7MxmAAqCEwu2Qc1OgghWqJChAabMHpThzVzs13Mrqdj8dctH9gcNcK%2BNwM912FlJzwYrQImC%2Bv1FAMHrxHH12ACo0G8Pp3VhPkaj0Zi14wfLshBmVhnsCrdgB5UKkM%2BPR7ClcIpFT2TvT5mABsu2AmBqT4%2BUZPpwIOYGHnRcZ0%2BdZVQ7Y0AXFM0v2nZ8IEfJdwTnEIAHcEI%2BaNVVA58PQFDCsPAyCAy7IMYLwaovDEXZYKJBhpTOWVI2BRVaXtT0/QgtccSowDaPoqUZTJVjqSVTiBW4%2BFcX4mjaF2HcWD3A8jw/L9h0wP9%2BS9O8Ox41dDJWW5e0Hc9NN2MsUTRDE%2BwHQxNLHGTg0JYlmNE%2BU2JpZVUBxaIwz2B11O/FCo29NzSVtdAIDpNteNNSUmhEqKFW8%2BlfP8gkcyCz8QrAsLhPclLYog29oMUhhd33Q9j1yzS/wy4gAuyxkAKAs9Qji1cA12cpKhHPTuzuB4zOvCyrI3Wz%2ByUQdHO65yYISiKWM88SON6vymqyk8uoouiEsKyKxPYnzNuanbSqgvalJUmrgvq9Kzu2oK2uIYDT2vXbDV6qgqEcgyERMkbDCctdpswWb%2BswOdc3zRgCEhsFRwBh8Xh/cE8OXAG12szcsXKwcIHBxGRxh4JCE0y9kLAhjCGIyZ8b2vMmvhkmtPJ4QoemNdMYICAOfptcHX0%2BF4olZAEEMT96uJhyoZhjDNMmIXx3KpEnQ1dZsGZgsEblpGCN2RWobvJF1fVF1ee9c0rWIG1I2VlGzYBDXLbhwQ2Zdc1iIgcxX1di5ecdhayvvFzkTDQQUpw3N0e9UNw2j3bcQShOo6h7SYwIOMEy8JNDlTcwzDcd3wV6A3WTLIuG2rHXWf1kcXTqjPOKrNwIWz5sGHQVtLrI8r6OWjzKS8iSTzrj2G8wF0h%2BK4PQ5TiVDpWke1uVIKJ71ocoZdZfh%2BikrSPbMjsdWcNdhYJhgl2KNQYJtGUNpzmkdNwFB12REdMJv3UF%2Bymn6Vq/AEKx46RwjPKYOa4QFp2fiOW%2BTtgHegllLMkGcf7byRpA8q0CwFJwQR/UBicW5AIIbPSMABWX0WC9o4MTg7EhtD05I3gSHEByCjCoOYT/P%2BXMGGEKjvQ/BjDYFaWoUiEBZD5SUI0GIwEwi8GsP4SIlha5XrAWkmOPkHBpi0E4OQ3gfhuC8FQJwFMlhrAQlmPMbKZh1g8FIAQTQ2jpgAGsQDrHWN6Dx3ifG%2BNfPoTgkgDFONICYjgvBSwaAcU46YcBYBIDQCwWIdBojkEoIk5J9AYjACkGYPgdARzEFLBACIISIjBCqAAT04PY8pzBiCVNPBEbQTRHFGNIIktgghTwMFoNUjgWhSBYAiF4YAbgxC0FLO0rAl8jDiAGbwfApxmh%2BSmYM94TQ86LEGeGEoIT0QRGIFUjwWAQkEDDCwGpvBNoRASJgPkmBZnAHREYGJfADDAAUAANTwJgdCp4oSGPsfwKOohxBSBkIIRQKh1ALNILoIoBhXmmHMZYfQeAIilkgNMVAsQygMCmcYp6WAsVRmKKUZILhu4DHqKQQIow8gFAyIkfFNLmVZAYB0RlExyWtJaMMNlDQSh8r6m0LlXRCi2AFZ4OoPRhjivGIUaYCgrELAkDovRwS4VhN2KoAAHK%2BNkr5JAfmQMgXYUhvQVggLgQgJA0x2MmLwNpWhJiuPcZ43xXrvH%2BN0RwIJpBDGDLCREkAUSXXaNIHExAIBEwogIGkiAGSUnEFCKwRY%2BrDXGtNeay1ZheAwjtQFPQILhBgvYBC0t0K1AhIRaQdCRzYhXI1RwfRgaQlhNPHneNP1dUGqNSa4AZqLWSCtTfDwSSU0OqmM6mJ7qPFeO9V6gJ/qtXBs4KG8Nc6V35vbdqjd0SFlutIJtRIzhJBAA%3D%3D%3D).

Output of the test code:

```txt
The current speed is: 0
Current state is: 1
The current speed is: 0
Current state is: 0
The current speed is: 5
Current state is: 0
The current speed is: 0
Current state is: 1
The current speed is: 0
Current state is: 1
```

You may notice that in our `Fan` class code, there is no `if-else` and `switch-case` lines, but the code behaves like it has such things.

When we changed the state, we changed the override function for the interface function `increaseSpeed()` and `decreaseSpeed()` declared in the `BaseFanState` class.

### Factory Method

If you are familiar with the design pattern, the factory method is based on the same polymorphism philosophy, so you can also use factory method to create different types of object with the same interface functions. In language like Rust, this design pattern is built into the language itself.

## Use Higher-Order Functions

Almost all modern programming language provides certain higher-order functions for array manipulation.

The most famous examples are `map()` functions from languages like Python or Rust. This function is not available in C++, however.

