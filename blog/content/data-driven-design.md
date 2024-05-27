
+++
title = "Data-Oriented Design Pattern"
date = 2024-05-27
draft = true
[taxonomies]
  tags = ["Design Patterns", "C++"]
[extra]
  toc = true
	keywords = "Code, C++, Design Patterns, Reflection, ECS"
+++

Data-oriented design pattern has already been used in some of the new generation game engines (such as [Bevy](https://bevyengine.org/), [Flecs](https://www.flecs.dev/flecs/), [Unity DOTS](https://unity.com/dots)) and this concept is also known as ECS (Entity, Component, System).

However, the same design pattern can be used for many applications which run heavy I/O operations. The advantages of the data oriented design pattern are:

* It can achieve very high I/O throughput due to all data are stored in a cache friendly way.
* Data can be easily processed in parallel in multi-threads automatically even without locks.

## Overview

Simply speaking the data-oriented design pattern containes 3 important parts:

* Entities
* Components
* Systems

