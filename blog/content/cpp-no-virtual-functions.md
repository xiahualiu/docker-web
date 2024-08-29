+++
title = "Avoid virtual functions in C++."
date = 2024-08-27
draft = false
[taxonomies]
  tags = ["Design Patterns", "C++"]
[extra]
  toc = true
	keywords = "Code, C++, Design Patterns, Template"
+++

Virtual function is the core feature of C++, it was introduced to C++ from the very beginning. It gives C++ something that C does not have, which is known as *Run-time Polymorphism*. However, many people often ignore the dark side of it. 

## Example

Let's say you are going to write a database application. For each data type inside the database, it must have a `print()` function associated, which returns a `string` type, so that people can see the data.

We can define a base abstract class, then declare `print()` as a pure virtual function there, so we can redefine this function later on for our data types, for example `Int1`.

Example code can be found [HERE](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjEIADMAOykAA6oCoRODB7evv6p6ZkCIWGRLDFxSbaY9o4CQgRMxAQ5Pn4BdpgOWQ1NBCUR0bEJyQqNza15HeP9oYPlw0kAlLaoXsTI7Bzm8aHI3lgA1CbxbmPEocAn2CYaAII7eweYx6dO55is17cP9/tMCgUhwsAMwABEmI1jokrPdDvDDikvFFaHhkCAfgjDgA3PDNLxiQ5jdAgEDnS6Ii6CCBLQ5oBhjV5gw4aE6wh6JMFsn4/f6Aw4ASUEXEOIERyNRyGBoIhUJMMMxCKRKLRGLhCKFBC4EHpjNCBEOTFpYuxYi8mAgRuhVk5ivhxNJ5KMlP1NLpAkZqGxsQuR3l7KxWOImAI6wYRIIJJARAA%2Bk7gBACAg8AoALTXU3eTBLbnq%2BHyrl5l2mgjsO2B/U4s2YXMcrnxdm8gz8zVma3l5WStV3LGtnUeg2Vq0m6uW2n%2Bgvlh1kghU4Au6m03UGidFoMhsMRqOx%2BOJ5NpjPVnMN8uTospC4lstrhGVzPm2uTk/3H6VlhMUKHN2rnsa4WG7UuA0Y8Az/Ag2yYMwICAkDy0rKIRROZkMgAL0wVAqEtLhYKLeC2yQok8DQjDLTMHDf3tSNSTQLwV1OE43GOMwzCEIiXgwwVhTFcx8Po05DgQ143AYrdSVcWhayxacaLo4T%2BJ41i0MODjW245ihJEqJeLkxjp3EySEWDUNiHDVlnzrDgVloTgAFZeD8bheFQThhMsawiTWDYXh2HhSAITRLJWABrBJ4gAOniSKoui6KADZ9E4SR7IC0hnI4XgFBADQ/IClY4FgJA0BYFI6FichKCKkr6DiYApACGhaFLYhMogKIUqiUImgAT04XyOuYYguoAeSibQun8xzSCKthBCGhhaB6jgtFILAoi8YA3DEWhMsmrB3yMcQlt4fBg26b0duWzBVC6WitmW/UahS1EomIbqPCwFLZzwFhet4b1iCidJwUwfbgFRIxcr4AxgAUAA1PBMAAdyGlJGF%2BmRBBEMR2CkDH5CUNQUt0Lh9EMYxrGsfQ8CiTLIBWVAUjqBlOCc/7fUwWmaWqWoshcBh3E8No9GCOYygqPQ0gyJnJj8EnJaKBgBjF4YSc6bp6hmGW9DVpnemaJWhjiVXNcFvJjb6A2FiNlYFE8zYJCs2zkqO1LOEOVQAA5YtTWLJEOYBkClKQwrbCBcEIEgmPibDeAmrQlmC0KIpilPIvi6yOCS0gHOWtKMqynKjrymBEBANYCCRAhyv7YrSuIcJWC2T3vd9/3A8OYOzF4TB8CIX09H4THRHEXHB/xlR1Bd4nSER16Ul%2Bx2ODs7OUrSobaMr5SqHdr2fb9gOg8kEOvw8WvqqjmPC/jxPIuT1OYoSzPndzlnbALuPAsfruV5dvOr8//6GRnCSCAA%3D%3D):

However, even `Int1` only has 1 `int` member variable, storing it takes 16 bytes of space!

`Int2` which does not have virtual functions at all, only takes 4 bytes to store. Imagine if we are going to store billions of `Int1` in our database, `Int1` will definitely become a poor choice.

Where does this size difference come from? 

If you read the assembly code, inside the `Int1` constructor, we need to store an additional value `vtable for Int1+16`, which points to the `vtable` for `Int1` on memory.

```assembly
Int1::Int1(int) [base object constructor]:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     QWORD PTR [rbp-8], rdi
        mov     DWORD PTR [rbp-12], esi
        mov     rax, QWORD PTR [rbp-8]
        mov     rdi, rax
        call    BaseData::BaseData() [base object constructor]
        mov     edx, OFFSET FLAT:vtable for Int1+16
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], rdx    # Write 8 bytes (vtable address)
        mov     rax, QWORD PTR [rbp-8]
        mov     edx, DWORD PTR [rbp-12]
        mov     DWORD PTR [rax+8], edx  # Write 4 bytes (input int value)
        nop
        leave
        ret
```

However this does not exist for `Int2`, the constructor of `Int2` only writes back 4 bytes:

```assembly
Int2::Int2(int) [base object constructor]:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     DWORD PTR [rbp-12], esi
        mov     rax, QWORD PTR [rbp-8]
        mov     edx, DWORD PTR [rbp-12]
        mov     DWORD PTR [rax], edx  # Wrtie 4 bytes (input int value)
        nop
        pop     rbp
        ret
```

The reason for the extra 8 bytes are that, **for a `Int1` object to call its virtual functions, the program needs to find the `vtable` belongs to this `Int1` first**.

Why we need to store this value?

Imagine if we `static_cast` our `Int1` object back to the base class `BaseData`, even though the cast object is `BaseData` type, the `vtable` pointer is still same as the `Int1` type. In short:

**The vtable pointer of an object is unrelated to the object type during runtime.**

This gives us *Run-time Polymorphism*, but also pain.

### Performance issues

This vtable jump hits our software performance, such as:

* We cannot inline the function anymore. The program has to jump to the vtable first, then jump back to the function definition next.

* For compiler, this 2-step jump limits the compiler's optimization capability. It can be seens in our example, because `Int2::print()` is never called, the compiler didn't even create this function at all. However compiler cannot do the same for `Int1::print()` because it can not predict when and where this function will be called.

* There is memory caching issues as you can imagine, we are reading different memory locations back and forth.

### What about `final`?

Even though you can use `final` keyword to let the compiler do the *devirtualization* optimization when possible, this optimization is not reliable.

The *Run-time Polymorphism* virtual functions provides, is a double edge sword, it gives us flexiblity, but also limits the compiler's optimization capability. 

## How to NOT use virtual function?

We can use a design pattern called Curiously Recurring Template Pattern (CRTP).

[Here is the example code](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAKxcpK4AMngMmAByPgBGmMQgZmakAA6oCoRODB7evgFBaRmOAmER0SxxCUm2mPbFDEIETMQEOT5%2BgTV1WY3NBKVRsfGJyQpNLW15nWN9A%2BWVIwCUtqhexMjsHOYAzOHI3lgA1Cbbbk5jxJisJ9gmGgCCO3sHmMenF%2BHAN3ePDwSYLBSBn%2Bbzc%2ByYCgUhwAKt8HuDIYcLBDMAARJhNY4AdisD0O%2BJSXhitDwyBAP3x%2BLG6BAIA%2BRkOKWI4QIEEW2Ish0uBDWDEOM0cyAA%2BqIxic3NCAFQ3CAEBB4BSLAB0TJZbJOnJMWNRPy1Ou2uN%2B9wRUIAkoIzByKQSiSSydb8eaCGY2YcQIcAG5iLyYCBcDTsrVWbUO/kEGl0gjMhmqwSuoNczA84h86m0ohC%2BnACBe7yYRYa7E6h4O1Ve/7kvGUlme72YDW6kMG3XwgyIp3bN2M22kpEo9FNcUdm5Wqvd4mkyv3SmHDuu925n1%2BgNWkNjsMRrOM6OswM4xPJ1Ph9OoTNRz45usFg1FlvTgnM8vsUM1xf15slpuGn41lhMcLxjioYdocTDbA265OpaTBmBB95UseKCrAQoLivyeAAF6YKgVAQGBganGhaYgK4tBwTOxFoF4KHikRmHYbhMEEW4RGIaR5GUtyvKHBocF6hwyy0Jw/i8H43C8KgnAsZY1j8qs6yvDsPCkAQmgCcsADWIDbNsSo6fpBmGQAbPonCSKJamkJJHC8AoIAaCpanLHAsBIGggJ0PE5CUO5KSeQkwBSMkNC0P8xB2RAMSWTE4TNAAnpwykxcwxBxQA8jE2iYA4iW8O5bCCGlDC0AlHBaKQWAxF4wBuGItB2eJFUAoYwDiGVvD4JcDh4B6mANeVmCqNl1GbOVLK1JZJIxMQ8UeFglnniwuWkL1xAxOkaLNUYJJGE5fAGMACgAGp4JgADuaUpIwy38IIIhiOwUgyIIigqOo7WkLoQQGLtpgyZY%2Bh4DEdmQMsqApPUDUSatzJYCDbJdNl9QuAw7ieO0eihOEgwVMMBTpJkAiTH4%2BNFFkcxDAkQR2EjPTjK06N5NTtS0wIvQtBTuNU7Y9PE3oMwc9j8x48sCjyRsEiCcJFkfdZhyqAAHEZAC0RmSIcwDIMghxSEqloQLghAkMcZjbFwiy8Kp7WLJp2m6YZDv6SZQkcOZpBieV1m2fZjnW6QLmICAyGEgQ3kQL5/mRKwmyKyrasa1rOuSHrvCYPgRCw3ot3CKI4hPdnr1qJZX2kGdM0pLlUscCJ7uWdZaXUSHhw4fLSuq%2Brmva7r%2BseB59DECbZsW77Wg26QWk6XpjsO6Zrsy57nDew5Vuj1XZjzxJi8j%2BpK3xBkziSEAA%3D)

The assembly code for `Int3` and `Int2` is identical, but since `Int3` inherits `BaseData`, user now must implement `print()` for `Int3` or will get a link error. 

The `Int3` implementation is almost identical with the virtual function version but it does not have any performance overhead.

## Generic programming vs virtual functions

The CRTP style of programming is acutally called **generic programming**. Where we consider more generic types than a specific one in our software, for example:
 
To operate on a object that inherits `BaseData<T>` further more, we are going to use template again.

For example we want to have a `print_object()` function that calls the corresponding `print()` function on an object. We can write:

```cpp
template<class T>
std::string print_object(const T& a) {
  return a.print();
}
```

Because we know `T` inherits `BaseData<T>`, `T` must have the `print()` function that we need. Now `T` becomes a generic type, all types that inherits `BaseData<T>` can be accepted here instead of a specific type.

In generic programming, we care more about the interface of an object, rather than the type of it.