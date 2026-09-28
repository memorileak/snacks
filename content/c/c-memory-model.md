+++
title = "Modern C: The C memory model"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 12: The C memory model** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book explores C's abstract execution environment, object representations in memory, strict aliasing rules, and hardware alignment constraints. 

Here is a detailed breakdown of the fundamental concepts and key takeaways from Chapter 12 across its seven main sections:


### 1. A Uniform Memory Model & Byte Representation (Section 12.1)
* **Objects as Byte Collections:** At the memory model level, all objects—regardless of type—are constructed from an assemblage of **bytes**.
* **The Atom of Storage:** **`sizeof(char)` is 1 by definition**. The C standard specifies **`unsigned char`** as the fundamental atomic building block for all object types in memory.
* **Byte Inspection:** Every object `A` can be viewed, inspected, and processed as an array of `unsigned char[sizeof A]`.
* **Character Pointer Exemption:** Pointers to character types (`unsigned char*`) are special because they are allowed to inspect and alias any object representation in memory.


### 2. Unions & Object Overlaying (Section 12.2)
* **Overlaying Types:** A **`union`** overlays different type interpretations over the **exact same physical memory location** rather than collecting sequential fields like a `struct`.
* **Inspecting Byte Endianness:** Unions provide a safe mechanism to inspect the **object representation** and byte order (**endianness**) of arithmetic or pointer types.
* **Implementation-Defined Storage Order:** The in-memory order of representation digits (little-endian vs. big-endian) for arithmetic types is **implementation-defined** by the target platform.


### 3. Memory, State, and Aliasing Rules (Section 12.3)
* **Aliasing Defined:** **Aliasing** occurs when two or more pointers address the same object in memory. Uncontrolled aliasing prevents compiler optimization because writing through one pointer might silently alter values read through another.
* **Strict Aliasing Rule:** With the exclusion of character types, **only pointers of the same base type are allowed to alias**. The compiler assumes pointers of different base types (e.g., `double*` and `size_t*`) never point to the same memory location.
* **Avoiding the `&` Operator:** Minimizing the use of the address-of operator `&` prevents local variables from being inadvertently aliased, allowing the compiler to optimize register usage.


### 4. Pointers to Unspecific Objects (`void*`) (Section 12.4)
* **Untyped Memory References:** A **`void*`** represents a pointer to an unspecific object or generic storage instance.
* **Implicit Conversion:** Any object pointer converts implicitly to and from `void*`. Converting an object pointer to `void*` and back to its original type is an **identity operation** that preserves the original memory address.
* **Avoid `void*` in User Code:** Avoid using `void*` when possible because converting to `void*` strips away all type safety, leaving address access unmonitored by the compiler.


### 5. Explicit Conversions & Casts (Section 12.5)
* **Strict Conversion Limits:** Implicit pointer conversions are limited to adding qualifiers (e.g., adding `const`) or converting to/from `void*`.
* **The Danger of Casts:** **Don't use casts `(T*)`**. Explicit pointer casts strip away compiler warnings and type safety checks, risking invalid pointer access or runtime memory corruption.


### 6. Effective Types (Section 12.6)
* **Strict Effective Type Rule:** Objects in memory **must be accessed through their effective type or through a character pointer** (`unsigned char*`).
* **Immutable Declared Types:** The effective type of a declared variable or compound literal is determined permanently by its declaration. Attempting to read or write a declared variable (e.g., `unsigned char A`) through an incompatible pointer cast (e.g., `(unsigned*)A`) violates the effective type rule and results in **undefined behavior**.
* **Union Exemption:** Members of an object with an effective `union` type can be accessed at any time, provided the stored bit pattern represents a valid value for the accessed member type.


### 7. Memory Alignment & Word Boundaries (Section 12.7)
* **Hardware Word Boundaries:** Objects of non-character types cannot start at arbitrary byte boundaries; hardware architectures require them to start at aligned **word boundaries**.
* **Misalignment Crashes:** Forcing pointer conversions to an incompatible, misaligned byte offset (e.g., casting `&buf` to `complex double*`) leads to **misaligned memory access or runtime bus errors**.
* **C23 Alignment Keywords:** C23 standardizes **`alignof`** (retrieves a type's alignment requirement) and **`alignas`** (forces a specific byte alignment boundary during allocation).

