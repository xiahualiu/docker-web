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

Let's say you are going to write a database application. For each data type inside the database, it must have a `print()` function associated, which returns a `std::string` type, so that people can see the data.

We can define a base abstract class, then declare `print()` as a pure virtual function there, so we can redefine this function later on for our data types, for example `Int1`.

Example code can be found [HERE](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjEIADMAOykAA6oCoRODB7evv6p6ZkCIWGRLDFxSbaY9o4CQgRMxAQ5Pn4BdpgOWQ1NBCUR0bEJyQqNza15HeP9oYPlw0kAlLaoXsTI7Bzm8aHI3lgA1CbxbmPEocAn2CYaAII7eweYx6dO55is17cP9/tMCgUhwsAMwABEmI1jokrPdDvDDikvFFaHhkCAfgjDgA3PDNLxiQ5jdAgEDnS6Ii6CCBLQ5oBhjV5gw4aE6wh6JMFsn4/f6Aw4ASUEXEOIERyNRyGBoIhUJMMMxCKRKLRGLhCKFBC4EHpjNCBEOTFpYuxYi8mAgRuhVk5ivhxNJ5KMlP1NLpAkZqGxsQuR3l7KxWOImAI6wYRIIJJARAA%2Bk7gBACAg8AoALTXU3eTBLbnq%2BHyrl5l2mgjsO2B/U4s2YXMcrnxdm8gz8zVma3l5WStV3LGtnUeg2Vq0m6uW2n%2Bgvlh1kghU4Au6m03UGidFoMhsMRqOx%2BOJ5NpjPVnMN8uTospC4lstrhGVzPm2uTk/3H6VlhMUKHN2rnsa4WG7UuA0Y8Az/Ag2yYMwICAkDy0rKIRROZkMgAL0wVAqEtLhYKLeC2yQok8DQjDLTMHDf3tSNSTQLwV1OE43GOMwzCEIiXgwwVhTFcx8Po05DgQ143AYrdSVcWhayxacaLo4T%2BJ41i0MODjW245ihJEqJeLkxjp3EySEWDUNiHDVlnzrDgVloTgAFZeD8bheFQThhMsawiTWDYXh2HhSAITRLJWABrBJ4gAOniSKoui6KADZ9E4SR7IC0hnI4XgFBADQ/IClY4FgJA0BYFI6FichKCKkr6DiYApACGhaFLYhMogKIUqiUImgAT04XyOuYYguoAeSibQun8xzSCKthBCGhhaB6jgtFILAoi8YA3DEWhMsmrB3yMcQlt4fBg26b0duWzBVC6WitmW/UahS1EomIbqPCwFLZzwFhet4b1iCidJwUwfbgFRIxcr4AxgAUAA1PBMAAdyGlJGF%2BmRBBEMR2CkDH5CUNQUt0Lh9EMYxrGsfQ8CiTLIBWVAUjqBlOCc/7fUwWmaWqWoshcBh3E8No9GCOYygqPQ0gyJnJj8EnJaKBgBjF4YSc6bp6hmGW9DVpnemaJWhjiVXNcFvJjb6A2FiNlYFE8zYJCs2zkqO1LOEOVQAA5YtTWLJEOYBkClKQwrbCBcEIEgmPibDeAmrQlmC0KIpilPIvi6yOCS0gHOWtKMqynKjrymBEBANYCCRAhyv7YrSuIcJWC2T3vd9/3A8OYOzF4TB8CIX09H4THRHEXHB/xlR1Bd4nSER16Ul%2Bx2ODs7OUrSobaMr5SqHdr2fb9gOg8kEOvw8WvqqjmPC/jxPIuT1OYoSzPndzlnbALuPAsfruV5dvOr8//6GRnCSCAA%3D%3D):

However, even `Int1` only has 1 `int` member variable, storing it takes 16 bytes of space!

`Int2` which does not have any virtual functions at all, only takes 4 bytes to store. Imagine if we are going to store billions of `Int1` in our database, `Int1` will definitely become a poor choice.

Where does this size difference come from? 

If you read the assembly code on godbolt, inside the `Int1` constructor, the program needs to store an additional value `vtable for Int1+16`, which points to the `vtable` for `Int1` on memory.

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

Why we need to store this value? Can't we just hardcode it in the `Int1::print()` function?

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

[Here is the example code](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIMwBspK4AMngMmAByPgBGmMT%2BAMykAA6oCoRODB7evv5BaRmOAmER0SxxCWbJdpgOWUIETMQEOT5%2BgbaY9sUMjc0EpVGx8Um2TS1teZ0KE4PhwxWj1QCUtqhexMjsHOaJ4cjeWADUJoluTrPEmKxn2CYaAIJ7B0eYp%2BdX4cB3D89PBEwLBSBkBHzchyYCgUxwAKr8npDoccLFDMAARJhNU4AdisT2OhJSXhitDwyBAf0JhNm6BAIC%2BRmOKWI4QIEBWxzQDFmuPxj2pguO1wImwYx1mWPJAH1RLMzhCBLzYQAqO4QAgIPAKFYAWjuXiUxGlLLZHLO/MFJhx6L%2B1ttiX5fyRMIAkoIzHyqUSSWSKd7Ce6CGYOccQMcAG5iLyYCBcDSc61WG0BiUEOkMgispmmwSh7m8pPCzCi4ji2n0ojSxnACBR7yYFYW3G2p4B01RwGUgnUtmR6OYC12lOOu2IgzIoOJMPM33klFozFNBVTu5enuz0nk7sC6lT0Ph%2BsxuMJr0pjdpjM146G%2BIm7PszkFgh84ul8vpyuoatZ751gdNo6LZjruzKsp27Cpn2R6DqObYjk6TwAPRIccQisO8UJcmIZJMjEaJchOMJUF4DD1AIxwAO6EAgkZ4C0XhiMcJFkb0Ch/ICwKgu8CounCCKPBWmbZsAYFstKqAxNodTss%2BC5KEuTAKvCiT3IExxMImeKpiKYoaQAdLmj5DvBrb/I8fYsEw4ShkmqZThpiQmaBxxBp6TBmM5gpCWgXgvgqCoSngABemCoFQEBMIkibnIFQmuLQXnUj5Gz%2BbF5xBaF4WRWYMVuHFn4gAlSU0oVvlpflGVRYZD62elbiXvSxVwS5KV%2BeCgUeTVZp5QVGbNZa1Iod5ZWpR1GVGRJUkyTlvUZfFDDoIlQHDdgxDECQqZtRVgWTZJ0kOJF0XjQ1C1LSVb56Rozn2hway0JwACsvB%2BNwvCoJw%2BWWNYEobFsPHVDwpAEJod1rAA1iAiSJPp0Nw/DCNBA9HCSC9oOkB9HC8AoIAaMDoNrHAsBIGgwJ0PE5CUKTKTkwkwBSGYfB0ICxA4xAMTozE4TNAAnpwQNc8wxA8wA8tNDj87wpNsIIIsMLQfMcFopBYDEXjAG4OE429KtAoYwDiErvD4Nc9QRpg2vK5gqh1H5OzK2y3To2SMTELzHhYOjv4sJLpDm8QMTpBietGLhoBG2sVAGMACgAGp4JglEiykjC%2B/wggiGI7BSDIgiKCo6hG6QuhcPo%2BsoNY1j6HgMQ45AayoCkbGcO9/uslgdccl0PRZC4i1TH4pehAs5SVHohSZAIA/j%2Bkk8MEMo%2BjKXtTkX0czT8v3QyQ0cwLyMCTL%2BvnjtHokotHvSwH2sCh/dsEj3U9aNF5jxyqAAHAEuoBJIxzAMgyDHCkPpT0EBcCEBIKcQGKxeAgwjhDKGMMEZILhkjTgqNSCvWVpjbGuN8YR1IETRAIBUrEgIJTCA1NaaRAwpwd%2Bn9v6/3/oAyQwDeCYHwEQdueh07CFEOIHOPD85qHRiXUglE3YpElg/Dgz0MHo0xiLPypDjjhVfh/L%2BP8/4AKASAjwZN6DEEgYkLg0C8FaBWPA6GsNkFIP0Ggp%2BWCW62FwbA8x0izAOPek41xYM/bxAyM4SQQA)

The assembly code for `Int3` and `Int2` is identical, but since `Int3` inherits `BaseData`, user now must implement `user_print()` for `Int3` or will get a compilation error. 

The `Int3` implementation is almost identical with the virtual function version but it does not have any performance overhead.

### Little heads up

You may have noticed that `Int3` has a different `user_print()` function name than the `print()` in `BaseData<Int3>`.

This is because in CRTP, you MUST differ the function names (here in our example `base_print()` and `print()`) that are used in the base class and the derived class.

Otherwise the program will run into segment fault issue, since the compiler now links these 2 functions in a circle.

## Generic programming vs virtual functions

The CRTP style of programming is acutally the fundation of the **generic programming** style in C++, where we consider more generic types than a specific one in our software.

For example we want to have a `print_object()` function that calls the corresponding `print()` function on an object that inherits the `BaseData` interface. We can write:

```cpp
template<class T>
std::string print_object(const BaseData<T>& a) {
  return a.print();
}
```

Writting this way, we ask the compiler to determine whether `T` inherits `BaseData<T>` during the compile time.

Note `BaseData<T>` in this function template now becomes a generic type, all types that inherits `BaseData<T>` can be accepted here instead of a specific type one.

In generic programming, we care more about the interface of an object, rather than the type of it, or its private content.

The generic interface of a type is provided by its parenet classes (one can have more than one interfaces of course), at zero runtime cost.