+++
title = "Avoid virtual functions in C++"
date = 2024-08-27
draft = false
[taxonomies]
  tags = ["Design Patterns", "C++"]
[extra]
  toc = true
	keywords = "Code, C++, Design Patterns, Template"
+++

Virtual function is the core feature of C++, it was introduced to C++ from the very beginning as the fundation of Object Oriented Programming. It gives C++ something that C does not have, which is known as *Run-time Polymorphism*.

However, it is also one of the most controversial feature up-to-date. It is so easy to use in C++ that many people often ignore the dark side of it. 

## Example

Let's say you are going to write a database application. For each data type inside the database, it must have a `print()` function associated with itself, which returns a `string` type, so the user can see the data.

We can define a base abstract class, then declare `print()` as a pure virtual function there, so we can redefine this function later on for our data types, for example `Int1`, see [this](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjEIADMAOykAA6oCoRODB7evv6p6ZkCIWGRLDFxSbaY9o4CQgRMxAQ5Pn4BdpgOWQ1NBCUR0bEJyQqNza15HeP9oYPlw0kAlLaoXsTI7Bzm8aHI3lgA1CbxbmPEocAn2CYaAII7eweYx6dO55is17cP9/tMCgUhwsAMwABEmI1jokrPdDvDDikvFFaHhkCAfgjDgA3PDNLxiQ5jdAgEDnS6Ii6CCBLQ5oBhjV5gw4aE6wh6JMFsn4/f6Aw4ASUEXEOIERyNRyGBoIhUJMMMxCKRKLRGLhCKFBC4EHpjNCBEOTFpYuxYi8mAgRuhVk5ivhxNJ5KMlP1NLpAkZqGxsQuR3l7KxWOImAI6wYRIIJJARAA%2Bk7gBACAg8AoALTXU3eTBLbnq%2BHyrl5l2mgjsO2B/U4s2YXMcrnxdm8gz8zVma3l5WStV3LGtnUeg2Vq0m6uW2n%2Bgvlh1kghU4Au6m03UGidFoMhsMRqOx%2BOJ5NpjPVnMN8uTospC4lstrhGVzPm2uTk/3H6VlhMUKHN2rnsa4WG7UuA0Y8Az/Ag2yYMwICAkDy0rKIRROZkMgAL0wVAqEtLhYKLeC2yQok8DQjDLTMHDf3tSNSTQLwV1OE43GOMwzCEIiXgwwVhTFcx8Po05DgQ143AYrdSVcWhayxacaLo4T%2BJ41i0MODjW245ihJEqJeLkxjp3EySEWDUNiHDVlnzrDgVloTgAFZeD8bheFQThhMsawiTWDYXh2HhSAITRLJWABrBJ4gAOniSKoui6KADZ9E4SR7IC0hnI4XgFBADQ/IClY4FgJA0BYFI6FichKCKkr6DiYApACGhaFLYhMogKIUqiUImgAT04XyOuYYguoAeSibQun8xzSCKthBCGhhaB6jgtFILAoi8YA3DEWhMsmrB3yMcQlt4fBg26b0duWzBVC6WitmW/UahS1EomIbqPCwFLZzwFhet4b1iCidJwUwfbgFRIxcr4AxgAUAA1PBMAAdyGlJGF%2BmRBBEMR2CkDH5CUNQUt0Lh9EMYxrGsfQ8CiTLIBWVAUjqBlOCc/7fUwWmaWqWoshcBh3E8No9GCOYygqPQ0gyJnJj8EnJaKBgBjF4YSc6bp6hmGW9DVpnemaJWhjiVXNcFvJjb6A2FiNlYFE8zYJCs2zkqO1LOEOVQAA5YtTWLJEOYBkClKQwrbCBcEIEgmPibDeAmrQlmC0KIpilPIvi6yOCS0gHOWtKMqynKjrymBEBANYCCRAhyv7YrSuIcJWC2T3vd9/3A8OYOzF4TB8CIX09H4THRHEXHB/xlR1Bd4nSER16Ul%2Bx2ODs7OUrSobaMr5SqHdr2fb9gOg8kEOvw8WvqqjmPC/jxPIuT1OYoSzPndzlnbALuPAsfruV5dvOr8//6GRnCSCAA%3D%3D):

However, as shown above, even `Int1` only has 1 `int` member variable, storing it takes 16 bytes of space!

Compared with `Int2` which does not have virtual functions at all, `Int2` only takes 4 bytes to store. Imagine if we are going to store billions of `Int1` in our database, `Int1` will definitely become a poor choice.

Where does this size difference come from? If you read the assembly code carefully you will notice the only difference between `Int1` and `Int2` is that. Inside `Int1` constructor, we need to store an additional value `vtable for Int1+16`, which points to the `vtable` for `Int1` on memory.

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

The reason for the extra 8 bytes are that, **for a `Int1` object to call its virtual functions, the program needs to find the `vtable` belongs to this `Int1` first**. Because the compiler cannot predict what `vtable` is pointed by this `Int1` object during the runtime.

Why the compiler cannot predict it? 

Imagine if we `static_cast` our `Int1` object back to the base class `BaseData`, even though the cast object is `BaseData` type, the `vtable` pointer is still same as the `Int1` type. In short:

**The vtable pointer of an object is unrelated to the object type during runtime.**

### Other performance issues

This vtable jump creates another problem for software performance, we cannot inline the function no more.

The program has to jump to the vtable first, then jump back to the function definition next.

For compiler, this 2-step jump limits the compiler's optimization capability. It can be seens in our example, because `Int2::print()` is never called, the compiler didn't even create this function at all. However compiler cannot do the same for `Int1::print()` because it can not predict when and where this function will be called.

And also as you can imagine there is cache issue at the same time.

Even though you can use `final` keyword to let the compiler do the *devirtualization* optimization when possible, this optimization is not reliable, the reason is same as above: the vtable associated with a given object is usually not predictable at the compilation time.

## How to NOT use virtual function?

