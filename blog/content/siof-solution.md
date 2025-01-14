+++
title = "Solve SIOF (Static Initialization Order Fiasco) with Meyer's Singleton."
date = 2025-01-14
draft = false
[taxonomies]
  tags = ["Linux"]
[extra]
  toc = true
	keywords = "C++"
+++

# Static Initialization Order Fiasco (SIOF) and Its Solution

## What is SIOF?
The Static Initialization Order Fiasco (SIOF) is a common C++ issue that occurs when initialization order of static objects across different translation units is undefined. As explained in the C++ standard:

> The static initialization order fiasco refers to the ambiguity in the order that objects with static storage duration in different translation units are initialized in.

## A Simple SIOF Example

```cpp
// filepath: a.cpp
class A {
public:
    A() { value = 42; }
    int value;
};

static A globalA;  // Static object in first translation unit
```

```cpp
// filepath: b.cpp
#include <iostream>
extern A globalA;  // Reference to static object from a.cpp

class B {
public:
    B() { 
        // May crash - globalA might not be initialized yet!
        std::cout << "A's value: " << globalA.value << std::endl;
    }
};

static B globalB;  // Problem: Order of initialization is undefined
```

## Understanding Initialization Stages
There are two key stages in initializing non-local objects:

### 1. Static Initialization
- **Constant Initialization**: Applied to `const`/`constexpr` static objects.
- **Zero Initialization**: Applied to other non-local static/thread-local variables.

### 2. Dynamic Initialization
Occurs in two possible ways:
- **Early Dynamic**: Happens before `main()`.
- **Deferred Dynamic**: Happens after the first usage.

## The Solution: Deferred Initialization
To avoid SIOF, we should use deferred initialization for all the non-const static variables. Here's a practical example:

```cpp
// filepath: a.cpp
class A {
public:
    A() { value = 42; }
    int value;
    
    static A& getInstance() {
        static A instance;  // Safe: initialized on first use
        return instance;
    }
};
```

```cpp
// filepath: b.cpp
class B {
public:
    B() {
        // Safe: A is guaranteed to be initialized before use
        std::cout << "A's value: " << A::getInstance().value << std::endl;
    }
    
    static B& getInstance() {
        static B instance;  // Also using function-local static
        return instance;
    }
};
```

It is called **Meyer's Singleton**.
