
+++
title = "Data-Oriented Software Architecture"
date = 2024-05-27
draft = false
[taxonomies]
  tags = ["Design Patterns", "C++"]
[extra]
  toc = true
	keywords = "Code, C++, Design Patterns, Reflection, ECS"
+++

Data-oriented design pattern has already been used in some of the new generation game engines (such as [Bevy](https://bevyengine.org/), [Flecs](https://www.flecs.dev/flecs/), [Unity DOTS](https://unity.com/dots)) and this concept is also known as ECS (Entity, Component, System).

However, the same design pattern can also be used for many applications which run heavy I/O operations. The advantages of the data oriented design pattern are:

* It can achieve very high I/O throughput due to all data are stored in a cache friendly way.
* Data can be easily processed in parallel in multi-threads automatically even without locks.

## Overview

Simply speaking the data-oriented design pattern containes 3 important parts:

* Entity
* Component
* System

Let's start with the **Component** first.

### Component

**Component** is the fine grain of the data piece that a **System** that operates on. It can be a primitive data types such as `int` or `float`, or it can be user defined data types like `struct`.

In the data-oriented architecture, a certain **Component** type can have many instances in memory. All these instances that belong to the same **Component** type, will be allocated consecutively like an array in the memory.

For example, if we defined a **Component** as:

```cpp
struct MessageStruct {
	uint8_t buffer[256];
}
```

Then we can create as many `MessageStruct` as we want, but all these `MessageStruct` instances will be put one by one next to each other in memory. This ensure when we read them, the CPU cache can work effectively by loading multiple of them at once.

### Entity

In the traditional software OOP architecture, the object (from a class type) contains all data associated with it in the consecutive memory blocks. Whereas in the data-oriented architecture, the **Entity** is merely a lookup table for finding all **Components** that construct it. This means the actually data that associated with a **Entity** is not stored continuously but rather spreaded out in memory space.

This is the most biggest difference between OOP and data-oriented architecture. Data-oriented design focuses on packing the same type of data (**Component**) tightly in the memory address to maximize the cache hit rate.

An important about **Entity** is the software logic doesn't access **Components** thru **Entity**, which is a common practice in OOP. Let's talk about **System** next.

### System

The **System** is the actually function that read and write the data. The immediate question will be, how a **System** know which piece of **Component** it needs to read and write on?

If the answer is looking up the **Entities**, then you cannot guarantee data read by the **System** will be continuous, as a result it will basically beat the purpose of data-oriented architecture. In reality the system access the data through a **Query**.

#### Query

The **Query** is something like `Query<const CpA, CpB>.with<CpC>().without<CpD>()`, it means the **System** will look for an **Entity** that has `CpA` and `CpB`, `CpC` and NOT `CpD`, then the **System** is going to read `CpA` and read/write `CpB`.

The interesting part is how **Queries** are implemented, all **Queries** are actually a tree starts from matching everything to matching a certain **Entity**. The root node has all **Components**, where the more constraints added to it, the fewer **Components** the node contains.

A reference **Query** design can be found from [Flecs Queries](https://www.flecs.dev/flecs/md_docs_2Queries.html). The most important detail of implementing **Query** is to construct `tables` of **Components**.

And likewise, when a new **Component** is inserted to the memory, it needs to find the correct **Component** table to append to.

#### Parallel Systems with Reflection

One thing that is special about the data-oriented design is that all data are stored together, the **Systems**, are just functions with **Queries** as inputs.

This makes concurrent processing of the data really easy, because the system can check whether 2 **Queries** return mutual exclusive **Components** based on the static types given by the **Queries**.

For example **Query** `foo` searches all **Components** `A`, another **Query** `bar` searches all **Components** `B`, then the system that has `foo` as the input can run along side with a system that has `bar` as the input at the same time.

This requires the problem can statically or dynamically checks the types of the parameter, which is a kind of meta programming capabilities, and this feature is called **Reflection**.
