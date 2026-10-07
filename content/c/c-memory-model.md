+++
title = "Modern C: The C memory model"
description = "A guide to the C23 memory model, covering object representations, unions, strict aliasing, void pointers, casts, effective types, and alignment."
date = 2026-09-28T04:47:32Z

[taxonomies]
tags = ["modernc", "object", "cast", "union", "alignment", "voidpointer", "unsignedchar"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 12. The C memory model

### Introduction to the Memory Model

- **What it is:** The C memory model is an abstraction of the environment and state in which a program executes. By using the unary `&` operator, programmers can retrieve an object's address and inspect or change the execution state.

- **Why it matters / how it works:** C does not distinguish the actual physical location of an object (e.g., RAM vs. a sensor on the moon). Modern operating systems provide _virtual memory_, mapping a process's address space to physical addresses. This abstraction ensures that C code is **portable** (ignoring physical addresses) and **safe** (reading/writing virtual memory does not crash the OS or other processes).

- **Key details:**
  - The only thing C inherently cares about is the type of the object a pointer addresses.
  - Each pointer type is derived from another base type, creating a distinct new type.

- **Pitfalls:** Assuming a pointer relates to a direct hardware physical memory address can lead to conceptual errors, as pointers only map to virtual memory provided by the system.

- **Connections:** Relates to pointers introduced in Chapter 11, extending the concept of addressing to how C fundamentally interprets memory.

> Takeaway 12 #1 _Pointer types with distinct base types are distinct._

**Explanation:** In C, a pointer to an `int` (`int*`) and a pointer to a `double` (`double*`) are fundamentally different types in the type system, even if both merely hold a memory address under the hood.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int x = 5;
    double y = 3.14;

    int* p_int = &x;       // Pointer to int base type
    double* p_double = &y; // Pointer to double base type

    // The compiler treats p_int and p_double as strictly distinct types.
    // What goes wrong if: p_int = p_double;
    // This would result in a compiler error or warning about incompatible pointer types.

    printf("p_int points to: %d\n", *p_int);
    printf("p_double points to: %f\n", *p_double);

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
p_int points to: 5
p_double points to: 3.140000
*/

```

### Recap

- The memory model uses virtual memory to abstract physical hardware locations.

- Pointer types in C are uniquely tied to their base types.

- This model guarantees portability and safety across differing architectures.

---

## 12.1. A uniform memory model

### Objects as Assemblages of Bytes

- **What it is:** In C, despite objects being strongly typed, the memory model simplifies memory by treating every object as an assemblage of bytes.

- **Why it matters / how it works:** This allows for a uniform accounting of size. The `sizeof` operator measures any object's size in terms of the bytes it uses. Three distinct character types (`char`, `unsigned char`, and `signed char`) are defined to use exactly 1 byte of memory.

- **Key details:**
- Because every object is composed of bytes, it can inherently be represented as an array of character types.

- **Pitfalls:** Assuming a byte is always exactly 8 bits on all historical architectures, although on most modern architectures it is.

- **Connections:** This builds on the `sizeof` operator introduced earlier with arrays (Chapter 6), explaining its exact unit of measurement.

> Takeaway 12.1 #1 sizeof(char) _is_ 1 _by definition._

> Takeaway 12.1 #2 _Every object A can be viewed as_ `unsigned char[sizeof A]`.

> Takeaway 12.1 #3 _Pointers to character types are special._

> Takeaway 12.1 #4 _Use the type_ `char` _for character and string data._

> Takeaway 12.1 #5 _Use the type_ `unsigned char` _as the atom of all object types._

> Takeaway 12.1 #6 _The_ `sizeof` _operator can be applied to objects and object types._

> Takeaway 12.1 #7 _The size of all objects of type T is given by_ `sizeof(T)`.

**Explanation:** By language definition, `char` is the fundamental unit of memory sizing, so `sizeof(char)` is always exactly 1. This means any arbitrary object `A` is essentially represented in memory as an array of `unsigned char` of length `sizeof A`. This enables generic memory copying and inspection using `unsigned char` as the basic "atom" of memory.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    double my_val = 3.14159;

    // Takeaway 12.1 #1 and #6, #7: sizeof applies to types and objects
    size_t char_size = sizeof(char);
    size_t double_size = sizeof(double);
    size_t obj_size = sizeof(my_val);

    printf("sizeof(char): %zu\n", char_size);
    printf("sizeof(double): %zu\n", double_size);
    printf("sizeof(my_val): %zu\n", obj_size);

    // Takeaway 12.1 #2 & #5: Viewing an object as unsigned char array
    unsigned char* byte_viewer = (unsigned char*)&my_val; // Special character pointer rule

    printf("Bytes of my_val in memory (hex): ");
    for(size_t i = 0; i < sizeof(my_val); ++i) {
        printf("%02X ", byte_viewer[i]);
    }
    printf("\n");

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
sizeof(char): 1
sizeof(double): 8
sizeof(my_val): 8
Bytes of my_val in memory (hex): 6E 86 1B F0 F9 21 09 40  (varies by endianness)
*/

```

### Recap

- `char` is always size 1, defining the fundamental byte in C.

- Any object is merely an array of `unsigned char` in memory.

- `sizeof` accurately measures the byte footprint of any type or object.

---

## 12.2. Unions

### Overlaying Object Types

- **What it is:** A `union` is a construct whose declaration looks similar to a `struct`, but it overlays items of different base types in the same memory location.

- **Why it matters / how it works:** Unions serve as a powerful tool to examine the individual bytes of objects or to reuse memory space when multiple types are not needed simultaneously.

- **Key details:**
  - The in-memory order of representation digits (bytes) for arithmetic types is implementation-defined (e.g., endianness).
  - On most architectures, `CHAR_BIT` is 8 and `UCHAR_MAX` is 255.

- **Pitfalls:** Unions are highly dependent on platform endianness; reading a different member than the one most recently written to can expose hardware-specific object representation details.

- **Connections:** Syntactically similar to `struct` (Chapter 6) but semantically vastly different due to shared storage.

> Takeaway 12.2 #1 _The in-memory order of the representation digits of an arithmetic type is implementation defined._

> Takeaway 12.2 #2 _On most architectures,_ `CHAR_BIT` _is 8 and_ `UCHAR_MAX` _is 255._

**Explanation:** Because architectures organize bytes differently (e.g., Little Endian vs. Big Endian), inspecting a type byte-by-byte via a union will yield implementation-defined orderings.

```c
#include <stdio.h>
#include <stdlib.h>

// Defining a union to inspect the bytes of an unsigned integer
typedef union unsignedInspect unsignedInspect;
union unsignedInspect {
    unsigned val;
    unsigned char bytes[sizeof(unsigned)];
};

int main(void) {
    // Initializing the unsigned value explicitly
    unsignedInspect twofold = { .val = 0xAABBCCDD };

    printf("Value: 0x%X\n", twofold.val);
    printf("Memory representation (byte by byte): ");

    // Accessing the union's array member exposes the in-memory order
    for (size_t i = 0; i < sizeof(unsigned); i++) {
        printf("%02X ", twofold.bytes[i]);
    }
    printf("\n");
    // What goes wrong if: you expect memory layout to be strictly 0xAA 0xBB 0xCC 0xDD on all machines.
    // It will fail on little-endian machines which store it in reverse byte order.

    return EXIT_SUCCESS;
}
/*
Expected behavior/output (on little-endian system):
Value: 0xAABBCCDD
Memory representation (byte by byte): DD CC BB AA
*/

```

### Recap

- `union` maps multiple distinct types to the exact same memory space.

- They are ideal tools for inspecting the byte-level object representation.

- Memory layout (endianness) dictates the order of these bytes and is implementation-defined.

---

## 12.3. Memory and state

### Managing Object Access and Aliasing

- **What it is:** The collective values of all objects define the state of the abstract state machine. Objects have unique locations accessed via pointers, but when multiple pointers point to the same object, it creates _aliasing_.

- **Why it matters / how it works:** Accessing an object through a pointer abstracts the state machine but complicates optimization. If a compiler doesn't know if two pointers alias, it cannot safely optimize accesses. Therefore, C forcibly restricts aliasing mostly to pointers of the same type.

- **Key details:**
  - Aliasing is a common cause for missed compiler optimization.
  - Pointer aliasing rules state that, generally, only pointers of the same base type are allowed to alias.

- **Pitfalls:** Using multiple pointers of different types to write to the same memory block violates strict aliasing rules, leading to undefined behavior. Furthermore, excessive use of the `&` operator can force the compiler to place variables in memory rather than registers, hindering performance.

- **Connections:** Relates to function parameters (Chapter 11) where multiple pointers are passed, leading to potential unseen side-effects.

> Takeaway 12.3 #1 _With the exclusion of character types, only pointers of the same base type may alias._

> Takeaway 12.3 #2 _Avoid the & operator._

**Explanation:** Character pointers are the exception to the aliasing rule and can inspect any object (as seen in Section 12.1). Otherwise, accessing a variable via a pointer of an incompatible type breaks the optimizer's assumptions. The `&` operator should be used sparingly to allow the compiler to heavily optimize local variable state.

```c
#include <stdio.h>
#include <stdlib.h>

// A function where aliasing can occur
double blub(double const* a, double* b) {
    *b = 10.0;
    return *a;
}

int main(void) {
    double c = 35;
    double d = 3.5;

    // Normal usage: pointers do not alias
    printf("blub(&c, &d) is %g\n", blub(&c, &d)); // Returns 35.0
    printf("after blub the sum is %g\n", c + d);  // 35.0 + 10.0 = 45.0

    // Aliased usage: pointers refer to the same object
    double e = 35;
    // Here, *b changing inside blub ALSO changes *a because they are the same address.
    printf("blub(&e, &e) is %g\n", blub(&e, &e)); // Returns 10.0

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
blub(&c, &d) is 35
after blub the sum is 45
blub(&e, &e) is 10
*/

```

### Recap

- Memory state represents the current value execution of the abstract machine.

- Aliasing occurs when two pointers refer to the same memory.

- C restricts aliasing to identical types to enable aggressive compiler optimizations.

---

## 12.4. Pointers to unspecific objects

### Working with `void*`

- **What it is:** C provides a pointer to a "non-type", `void*`, to handle memory generically without strict type information.

- **Why it matters / how it works:** A `void*` pointer conceptually points into a _storage instance_ rather than a typed object. It acts as a universal intermediate address format, conceptually similar to a raw phone number stripped of context.

- **Key details:**
  - Any object pointer can automatically convert to and from a `void*`.
  - Converting a pointer to `void*` and back to its original type guarantees the exact same original pointer value (identity operation).

- **Pitfalls:** While powerful, `void*` strips type safety. Accessing or dereferencing data via a `void*` directly is forbidden because the compiler lacks type constraints (size, alignment). Thus, the text advises minimizing its use where specific types are known.

- **Connections:** Relates to generic programming and dynamic allocation (Chapter 13), where functions like `malloc` return `void*`.

> Takeaway 12.4 #1 _Any object pointer converts to and from_ `void*`.

> Takeaway 12.4 #2 _An object has storage, type, and value._

> Takeaway 12.4 #3 _Converting an object pointer to_ `void*` _and then back to the same type is the identity operation._

> Takeaway 12.4 #4 _Avoid_ `void*`.

**Explanation:** `void*` is C's escape hatch for type-agnostic pointers. You can convert any pointer to it and safely retrieve the exact same typed pointer later. However, it eliminates compiler checks, so it should be avoided unless strictly necessary.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    long original_val = 123456789;
    long* p_long = &original_val;

    // Takeaway 12.4 #1: Object pointer implicitly converts to void*
    void* generic_ptr = p_long;

    // What goes wrong if: printf("%ld\n", *generic_ptr);
    // You cannot dereference a void* directly.

    // Takeaway 12.4 #3: Converting back is the identity operation
    long* restored_ptr = generic_ptr;

    if (p_long == restored_ptr) {
        printf("Restored pointer matches original: %ld\n", *restored_ptr);
    }

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
Restored pointer matches original: 123456789
*/

```

### Recap

- `void*` represents raw storage addresses stripped of type semantics.

- Conversions to and from `void*` to the _same_ object type are safe and reversible.

- `void*` removes compiler safety checks and should be avoided in day-to-day code.

---

## 12.5. Explicit conversions

### Casts and Data Narrowing

- **What it is:** Explicit conversions (casts) force a value from one type into another, potentially altering its representation.

- **Why it matters / how it works:** C performs many conversions implicitly. Using narrow types (e.g., `short` instead of `int`) requires the CPU to cast them back to `int` for arithmetic. Therefore, explicitly casting or using narrow types is only justified to save memory when storing massive arrays of values (millions or billions).

- **Key details:**
  - Integer types smaller than `int` undergo implicit integer promotion before operations.

- **Pitfalls:** Explicit casts hide warnings and can create insidious bugs, notably casting away `const` qualifiers or casting invalid pointers.

- **Connections:** Builds on type conversions from Chapter 5, establishing that explicit casting is generally an anti-pattern in modern C.

> Takeaway 12.5 #1 _Don’t use casts._

**Explanation:** Modern C is designed such that valid conversions (like `void*` to object pointers) happen implicitly. Forcing a cast often masks legitimate compiler errors and degrades code safety.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int val = 300;

    // Implicit conversion is preferred when valid
    double d_val = val;

    // Explicit casting is discouraged (Takeaway 12.5 #1)
    // char c = (char)val; // Avoid this: masks the truncation of 300 into a char

    printf("Double val: %f\n", d_val);

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
Double val: 300.000000
*/

```

### Recap

- Implicit conversions handle most routine type alterations automatically.

- Narrow types only exist to save bulk memory, as they are natively expanded for arithmetic.

- Explicit casts should be strictly avoided as they bypass the type system.

---

## 12.6. Effective types

### Access Restrictions

- **What it is:** The "effective type" rules mandate that memory be accessed according to the type of data actually stored there.

- **Why it matters / how it works:** When storage is allocated dynamically or memory is manipulated via `void*`, the compiler tracks the _effective type_ established when a value is written to that memory.

- **Key details:**
  - Objects must only be read as their effective type (or as character types).

- **Pitfalls:** Writing an `int` to raw memory and then reading it back as a `float` violates effective type rules (strict aliasing), producing undefined behavior.

- **Connections:** Tied heavily to the aliasing constraints introduced in Section 12.3.

> Takeaway 12.6 #1 _Objects must be accessed through their effective type_

**Explanation:** Memory inherits a type based on how it is initialized or declared. Reinterpreting that memory under a different type breaks C's memory model constraints.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    float f = 3.14f;

    // Valid: Accessing through the effective type (float)
    float* pf = &f;
    printf("Valid access: %f\n", *pf);

    // Valid: Accessing via character type
    unsigned char* pchar = (unsigned char*)&f;
    printf("Valid char access (first byte): %02X\n", pchar[0]);

    // What goes wrong if: int* pi = (int*)&f; printf("%d", *pi);
    // Invalid: Breaks effective type rule. The effective type is float, not int.

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
Valid access: 3.140000
Valid char access (first byte): C3 (varies)
*/

```

### Recap

- Memory assumes an effective type upon declaration or assignment.

- Violating effective types prevents valid code optimization.

- Character types are the only universal exception for accessing underlying memory.

---

## 12.7. Alignment

### Platform Memory Constraints

- **What it is:** Alignment dictates that objects of most noncharacter types cannot start at arbitrary byte positions in memory; they are required to start at specific _word boundaries_.

- **Why it matters / how it works:** Hardware processors are optimized to fetch memory in blocks (words). If an `int32_t` is placed at a non-aligned byte address (e.g., an odd address), the CPU may require multiple memory fetches or throw a hardware exception (bus error).

- **Key details:**
  - The inverse direction of pointer conversions (from `char*` back to an object pointer) is highly dangerous because the `char*` might not fall on a valid boundary for the object type.

- **Pitfalls:** Casting an arbitrary `unsigned char` array into a complex object type (like `struct` or `double`) without ensuring alignment can lead to immediate crashes on strict architectures.

- **Connections:** Explains why `sizeof(struct)` often includes "padding" (Chapter 6) to ensure sequential elements obey alignment constraints.

**Explanation:** While every object can be decayed into a character array, an arbitrary character array cannot always be promoted into a larger object due to hardware alignment bounds.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdalign.h>

int main(void) {
    // Ensuring proper alignment explicitly if constructing buffers
    alignas(double) unsigned char buffer[sizeof(double)];

    // Now it is safe to treat this buffer as a double because it aligns properly
    double* pd = (double*)buffer;
    *pd = 2.718;

    printf("Aligned double value: %f\n", *pd);

    // What goes wrong if:
    // unsigned char bad_buffer[sizeof(double) + 1];
    // double* bad_pd = (double*)(&bad_buffer[1]);
    // *bad_pd = 1.0;
    // This could cause a hardware crash (bus error) due to misaligned access.

    return EXIT_SUCCESS;
}
/*
Expected behavior/output:
Aligned double value: 2.718000
*/

```

### Recap

- Hardware requires large data types to be placed at specific address multiples (alignment).

- Pointer conversions from character arrays to wider types risk violating these boundaries.

- Misalignment can cause performance degradation or outright program failure.

---

## Summary

The C memory model provides an elegant, multi-layered abstraction of physical hardware. It relies on virtual memory, shielding the programmer from bare-metal addressing while organizing memory into discrete storage instances. At its lowest level, every object is merely an array of `unsigned char`, allowing binary representation inspection through mechanisms like `union`. However, this raw memory access is governed by strict rules: pointers to distinct types are treated as non-overlapping to optimize execution (aliasing), variables must be queried through their correct effective types, and arbitrary memory locations must respect platform-specific alignment boundaries before wider data types can be assigned to them.

## Self-Check Questions

1. **What does `sizeof(char)` evaluate to?**

- _Answer:_ It is always exactly 1 by definition.

2. **How can you inspect the byte-by-byte memory representation of a `double`?**

- _Answer:_ By casting a pointer to the `double` to an `unsigned char*`, or by using a `union` containing both the `double` and an array of `unsigned char`.

3. **Why does C restrict aliasing mostly to pointers of the same type?**

- _Answer:_ Because allowing differing pointers to refer to the same object complicates the abstract state machine and prevents the compiler from optimizing variable storage.

4. **Is it safe to cast any pointer to a `void*` and back?**

- _Answer:_ Yes, doing so is an identity operation and guarantees the pointer is preserved.

5. **Why should explicit casts be avoided?**

- _Answer:_ Because standard C performs necessary and safe conversions implicitly; casts disable the type system's safeguards and obscure potential bugs.

6. **What is an effective type?**

- _Answer:_ The actual type of the object stored at a memory location, dictating the valid pointer types that can read from or write to it.

7. **What error could occur if an array of `char` is cast to an `int*`?**

- _Answer:_ The resulting address might not satisfy the hardware's alignment boundary for `int`, leading to a crash or undefined behavior.

8. **Are `char` pointers exempt from the strict aliasing rule?**

- _Answer:_ Yes, character pointers are permitted to inspect the bytes of any object type.
