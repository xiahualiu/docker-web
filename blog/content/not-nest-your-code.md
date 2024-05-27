
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

Here we used a bin-bucket hash table, but you can use any hash table types, like in C++ `std::map` or `std::unordered_map`, in Python `dict` object.

## Early Return

Early return requires a function to return (or throw exceptions) as early as possible if a certain situation is met.

For some languages like Rust you can use things like `unwrap` or `assert` functions to throw the exception easily. 

This is easy to understand since the earlier a function returns the fewer control flow branches it needs to deal with, which as a result, reduce the code nesting.

## State Design Pattern

C++ uses vtable to achieve polymorphism, and it acts exactly like a `switch-case` or look up table we saw above.

So we can use this feature, to hide the `switch-case` style functions by overriding the functions in different sub classes.

The most known design pattern for this is called **State** pattern, which can be found [here](https://refactoring.guru/design-patterns/state).

Let's assume you need to write the function for a ceiling fan, the user can input up/down and on/off to control the speed.

When the fan is the OFF, of course the user input does nothing and the speed is zero. But if the fan is ON, the user can control the speed of the fan.

With the state design pattern, you can write the code like this:

```cpp
#include <iostream>

class FanContext
{
    float speed;

    public:
    void increaseSpeed(float a) { this->speed += a; }
    void decreaseSpeed(float a) { this->speed -= a; }
    void setZeroSpeed() { this->speed = 0.0; }
    void printSpeed() { std::cout << "The current speed is: " << this->speed << std::endl; }
};

class BaseFanState
{
    FanContext *context;

    public:
    enum STATE { ON, OFF };

    FanContext& getContext() { return *context; }
    void setContext(FanContext *newContext) { context = newContext; }

    virtual void increaseSpeed(float a) = 0;
    virtual void decreaseSpeed(float a) = 0;
    virtual enum STATE getState() = 0;
};


class FanONState : public BaseFanState
{
    void increaseSpeed(float a) override { getContext().increaseSpeed(a); }
    void decreaseSpeed(float a) override { getContext().decreaseSpeed(a); }
    enum STATE getState() override { return ON; }
} onState;

class FanOFFState : public BaseFanState
{
    void increaseSpeed(float a) override { ; }
    void decreaseSpeed(float a) override { ; }
    enum STATE getState() override { return OFF; }
} offState;

class Fan
{
    BaseFanState* currentFanState;
    FanContext context;

    public:
    Fan(BaseFanState* initState, FanContext initContext):
    currentFanState(initState),
    context(initContext)
    {};

    void changeState(BaseFanState* newState)
    {
        this->currentFanState = newState;
        this->context.setZeroSpeed();
        this->currentFanState->setContext(&this->context);
    }

    void printSpeed() { context.printSpeed(); }
    void printState() { std::cout << "Current state is: " << currentFanState->getState() << std::endl; }
    void increaseSpeed(float a) { currentFanState->increaseSpeed(a); }
    void decreaseSpeed(float a) { currentFanState->decreaseSpeed(a); }
};

int main ()
{
    FanContext initState;
    Fan A = Fan(&offState, initState);
    A.printSpeed();
    A.printState();
    A.changeState(&onState);
    A.printSpeed();
    A.printState();
    A.increaseSpeed(5.0);
    A.printSpeed();
    A.printState();
    A.changeState(&offState);
    A.printSpeed();
    A.printState();
    A.increaseSpeed(5.0);
    A.printSpeed();
    A.printState();
    return 0;
}
```

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

If you are familiar with the design pattern, the factory method is based on the same polymorphism philosophy, so you can also use factory method to create different types of object with the same interface functions. In language like Rust, these design pattern is built into the language itself.

## Use Higher-Order Functions

Almost all modern programming language provides certain higher-order functions for array manipulation.

The most famous examples are `map()` functions from languages like Python or Rust. This function is not available in C++, however.

