+++
title = "Don't ignore data tolerance in control system design."
date = 2024-10-09
draft = false
[taxonomies]
  tags = ["Math", "Control System"]
[extra]
  math = true
  math_auto_render = true
  toc = true
	keywords = "Float, C++, Error"
+++

When a system engineer is designing a new control system, there is one aspect that is easily overlooked, that is how the data error tolerance will get propagated through out the system.

## Data Error Sources

No matter what system you have, and where the input data are coming from, there will always be error in the data. The data error usually come from 2 sources:

* Precision loss due to data storage.
* Sensor readings or other systemetic restrictions.

**Data storage** error is introduced when a value is stored on the system, such as float/double data types. Because computer can only use finite bit width to store a value, it can only store finite precision, if the data value (such as $pi$) cannot be represented with this given bit width then the value will be cut into fit the data types. Specifically for IEEE 754 float point data representation:

- A 32-bit float-point uses 23-bit (mantissa) to store the numerator of a fractional number. This means that a float can at most store the 7 most significant digit of a value.
- A 64-bit float-point (usually called `double`) uses 52-bit (mantissa) to store the numerator of a fractional number. This means that a float can at most store the 16 most significant digit of a value.

$$float\ A=\pi$$
$$double\ B=\pi$$

$$A=3.1415927+\epsilon_A$$
$$B=3.141592653589793+\epsilon_B$$

**Sensor reading** error comes from the input of the system, usually called sensors. For example, a NTC temperature sensor can measure temperate with a 0.1C tolerance, this means that for all data from it, come with 0.1C uncertainty.

The system engineer should **always** aware of the error tolerance for all the data.

If the data come from another system, the provider system should also provide the error tolerance information for the data it provides, with no exceptions.

The error tolerance of a sensor can be multiple values, usually paired with the probablity, if we use $\epsilon_{prob}$ to represent the error tolerance value:

If a value from an NTC temperature sensor has 99.9% probablity within the 0.1C error tolerance, we can use:

$$\epsilon_{0.999}<0.1C$$

To represent the error of this sensor.

Note this our notation works closely with the failure mode analysis of the system, if the control system is designed to use $\epsilon_{0.999}$ for error bound, the failure rate will be $1-0.999=0.1\%$.

### Storage error is runtime error

As you can see, the data storage error depends on the data value, for example, a 32-bit float number `123456789.0` has more storage error than a 32-bit float number `1.0`. Since a float variable can change its value during runtime, this type of error tolerance can also change as well.

In order for the system to track this kind of error, we need to store the error tolerance value with the data and update the tolerance after any arithmetic operation on the data.

### Sensor reading error is constant error

This type of error is usually the constant type of error, since it is determined by the sensor that the system uses.

## Why do we need to care about error propagation

### Optimize data process

We want to manipulate the system data processing steps to make sure we can preserve the precision of the original data as much as possible. This indicates we should try to minimize the error tolerance of the data for a given calculation function.

### Data validity detection

It provides a way for the system to validate the data in real-time, i.e. if we have a data that come with too much error tolerance, the software will automatically render the data value invalid and abort the rest calculations.

### Measure data reliablity

This is crucial for safety critical system. We can measure how reliable the output is from a given system based on the output data error tolerance.


## Error tolerance propagation in arithmetic operations

If we have data value $x$. For every arithmetic operations, such as:

* Addition/Substraction/Multiplication/Division.

$$x+a$$
$$x-a$$
$$x\times a$$
$$\frac{x}{a}$$

* Trigonometric functions, such as:

$$\sin(x)$$ 
$$\cos(x)$$ 
$$\tan(x)$$

* Power/logarithm calculations:

$$x^a$$
$$log_a(x)$$

These calculated result, also carries an updated error tolerance that comes with $x$, the tolerance value can propagate along with all the calculations inside the system, we call it error tolerance propagation.

## Store data error with data value

Since we know above, data error needs to be during the runtime because storage error depend directly on the data value. We must store the data error tolerance with data value and process the error after any arithmetic operations.

## How to calculate the new tolerance value

### Update new tolerance value from math calculations

The tolerance propagation function can be found in many statistic research paper. For function $y=f(x, a, b,\dots)$, given $x$, $a$, $b$ are all **un-correlated**, **normal-distributed** random variables (in practice, it still works well even the data variables are weakly correlated to each other.)

$$\sigma_y^2=(\sigma_x\frac{\partial_f}{\partial_x})^2+(\sigma_a\frac{\partial_f}{\partial_a})^2+(\sigma_b\frac{\partial_f}{\partial_b})^2+...$$

There are many papers you can find online, such as [Error Propagation tutorial.doc](https://foothill.edu/psme/daley/tutorials_files/10.%20Error%20Propagation.pdf) list common propagation functions for additive/substraction, etc. arithmetic operations.

For most systems, using the first order approximation is practically good enough for tolerance propagation calculation.

There are also co-variance matrix version for matrix version calculation, which requires us to calculate the **Jacobian matrix** instead of the scalar partial derivative function.

### Check tolerance value from storage tolerance

Then we are going to compare the tolerance against the storage types, for example the result is `123456789.0` and the storage type is `float`, we then compare the existing tolerance value with `123456789.0/2^23` and if the storage tolerance is bigger, we will have a precision loss incident if we store the data.

For safety critical system, you can set up a warning here to indicate precision loss in the software and abort the rest operations.

## Conclusion

It is very important for the system engineer to not ignore the tolerance propagation for the designed system. Always remember that **all data come with uncertainty**.