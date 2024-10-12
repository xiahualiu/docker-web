+++
title = "Float-points precision loss detection for safety critical systems."
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

This article proposes a systematic way to detect the precision loss due to runtime data storage.

## What is float point precision loss?

Computer system uses finite bit width to represent float point numbers like `1.0` and `0.5`.

For float point numbers like $\pi$, there is no way to store all the precision information from this value. The software will round the number to the nearest float point number it can store and this introduces the **error caused by data storage**.

Almost all modern computer architectures now provide `float` and `double` types in their systems, and they have different precision limits:

- A 32-bit float-point uses 23-bit (mantissa) to store the numerator of a fractional number. This means that a float can at most store the 7 most significant digits of a value.
- A 64-bit float-point (usually called `double`) uses 52-bit (mantissa) to store the numerator of a fractional number. This means that a float can at most store the 16 most significant digits of a value.

If we want to store $\pi$ in the above types:

$$float\ A=\pi$$
$$double\ B=\pi$$

$$A=3.1415927+\epsilon_A$$
$$B=3.141592653589793+\epsilon_B$$

The $\epsilon_A$ and $\epsilon_B$ represent the storage error for $\pi$ respectively for `float` and `double`.

However if we only store numbers like `0.5` and `1.0`, since they have fewer significant digits, these values can be stored in the system **losslessly**.

We call this type of error: **Error Due to Data Storage**.

### Storage error is proportional to the value

As you can see, the data storage error depends on the data value, for example, a 32-bit float number `123456789.0` has more storage error than a 32-bit float number `1.0`. Since a float variable can change its value during runtime, this type of error tolerance can also change as well.

We can calculate the approx storage error easily:

$$\epsilon_{float}\approx\frac{x}{2^{23}}$$

$$\epsilon_{double}\approx\frac{x}{2^{52}}$$

## Data Uncertainty (Data Tolerance)

No matter what system you have, and where the input data are coming from, there will always be uncertainty in the input data. 

### Example: sensor reading tolerance

Such as sensor reading error, that comes from the physical inputs. 

For example, an NTC temperature sensor can measure temperate within $\pm 0.1 C$ tolerance, this means that for all data from it, come with this much uncertainty.

The error tolerance of a sensor can be multiple values, usually paired with the probablity, if we use $\epsilon_{prob}$ to represent the error tolerance value:

If a value from an NTC temperature sensor has 99.9% probablity within the 0.1C error tolerance, we can use:

$$\epsilon_{0.999}<0.1C$$

To represent the error of this sensor.

Note this our notation works closely with the failure mode analysis of the system, if the control system is designed to use $\epsilon_{0.999}$ for error bound, the failure rate will be $1-0.999=0.001$.

### Data tolerance can be propagated

If the data come from another system, the provider system should also provide the error tolerance information for the data it provides, with no exceptions.

### Tolerance propagation

Inside a given system, not matter if it is SISO or MIMO, the data uncertainty can be propagated from input to output, and we called this process **tolerance propagation**

If the input data value is $x$. For every arithmetic operations, such as:

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

These calculated result, the output, also carries an updated error tolerance that comes with $x$, the tolerance value can propagate along with all the calculations inside the system until we pass it to another system.

The tolerance propagation function can be found in many statistic research paper. For function $y=f(x, a, b,\dots)$, given $x$, $a$, $b$ are all **un-correlated**, **normal-distributed** random variables (in practice, it still works well even the data variables are weakly correlated to each other.)

$$\sigma_y^2=(\sigma_x\frac{\partial_f}{\partial_x})^2+(\sigma_a\frac{\partial_f}{\partial_a})^2+(\sigma_b\frac{\partial_f}{\partial_b})^2+...$$

There are many papers you can find online, such as [Error Propagation tutorial.doc](https://foothill.edu/psme/daley/tutorials_files/10.%20Error%20Propagation.pdf) list common propagation functions for additive/substraction, etc. arithmetic operations.

For most systems, using the first order approximation is practically good enough for tolerance propagation calculation.

There are also co-variance matrix version for matrix version calculation, which requires us to calculate the **Jacobian matrix** instead of the scalar partial derivative function.

## Precision Loss Detecton Principle

Simply speaking, 

* We calculated data uncertainty of all results that generated by the data processing steps, using the uncertainty propagation formulas shown as above.
* We calculated the error introduced by storage after getting the result values.
* We compare the storage error to the data uncertainty.

If the error due to storage is higher than a certain threshold, saying $1\%$ of the data uncertainty, it triggers a precision loss event. Because we cannot guarantee the data is valid after storing it.

Using the above math notation, for any calculated result $y$, if we set the threshold to be $p$, we compare:

$$If\ \epsilon_y > p\sigma_y$$

If it is true, we need to consider that we are having a precision loss event in our system.

## Why do we need to care about error propagation

### Optimize data process

We want to manipulate the system data processing steps to make sure we can preserve the precision of the original data as much as possible. This indicates we should try to minimize the error tolerance of the data for a given calculation function.

Aside from the time and space complexity, data preservation can also be considered as a performance index for a data processing function.

### Data validity detection

It provides a way for the system to validate the data in real-time, i.e. the software will automatically render the data value invalid and abort the rest calculations after it detects a precision loss event.

### Measure data reliablity

This is crucial for safety critical system. We can measure how reliable the output is from a given system based on the output data tolerance.

## Conclusion

It is very important for the system engineer to not ignore the tolerance propagation for the designed system. Always remember that **all data come with uncertainty**, and we want to make sure that the **error introduced by data storage is WAY LOWER than the inherent data uncertainty**.
