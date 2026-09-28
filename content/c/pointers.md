+++
title = "Modern C: Pointers"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 11: Pointers** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book enters the core of **Level 2: Cognition**. Pointers represent a fundamental mechanism in C that enables functions to modify caller variables, manage dynamic data structures, and interface efficiently with arrays and functions.


### 1. Pointer Fundamentals & Operators (Section 11.1.1)
* **Pointers as References:** A pointer is a derived type that refers or "points" to another object in memory.
* **Address-of Operator (`&`):** The unary `&` operator retrieves the memory address of an addressable object.
* **Object-of / Dereference Operator (`*`):** The unary `*` operator dereferences a pointer to access or modify the underlying object. 
* **Dual Role of `*`:** In declarations, `*` creates a pointer type (e.g., `double* p0;`); in expressions, `*` dereferences the target object (e.g., `*p0 = 10.0;`).
* **Takeaway 11.1.1 #1:** *A program execution that uses `*` with an invalid or null pointer fails*. Null pointer dereferences typically crash early (which aids debugging), whereas dereferencing invalid pointers can silently corrupt arbitrary memory.


### 2. Pointer Arithmetic & Ranges (Sections 11.1.2 & 11.1.3)
* **Takeaway 11.1.2 #1:** *A valid pointer refers to the first element of an array of the reference type*.
* **Offset Addition:** Adding an integer `i` to a pointer `a` (`a + i`) computes a pointer to the \\(i^{\text{th}}\\) element of the array.
* **Takeaway 11.1.2 #2 & #3:** *The length of an array object cannot be reconstructed from a pointer*, and **pointers are not arrays**. Using `sizeof` on a pointer yields the size of the pointer itself, not the underlying array.
* **Takeaway 11.1.3 #1:** *Only subtract pointers to elements of the same array object*.
* **Pointer Differences (`ptrdiff_t`):** 
  * **Takeaway 11.1.3 #2 & #3:** *All pointer differences have type `ptrdiff_t`*, a signed integer type from `<stddef.h>` designed to encode element distances or relative index offsets.
* **Takeaway 11.1.3 #4:** *For printing, cast pointer values to `void*` and use the format `%p`* (e.g., `printf("%p", (void*)ptr)`).


### 3. Pointer States, Bounds & C23 `nullptr` (Sections 11.1.4 & 11.1.5)
* **Pointer States:** A pointer exists in one of three states: **valid** (points to a valid object), **null** (points to no object), or **invalid** (holds an uninitialized or out-of-bounds address).
* **Takeaway 11.1.4 #1 & #2:** *Pointers have a truth value* (null pointers evaluate to `false`; valid pointers evaluate to `true`). *Set pointer variables to null as soon as you can*.
* **Valid Pointer Bounds:**
  * **Takeaway 11.1.4 #5:** *A pointer must point to a valid object, one position beyond, or be null*. Computing or navigating a pointer beyond one element past an array boundary causes program failure.
  * **Takeaway 11.1.4 #4:** *When dereferenced, a pointed-to object must be of the designated type*.
* **C23 Null Pointer Standard:**
  * **Takeaway 11.1.5 #1:** *Use `nullptr` instead of `NULL`*. Pre-C23 `NULL` macro definitions (such as `0` or `(void*)0`) lacked type safety across differing platform integer widths. C23 standardizes `nullptr` with type `nullptr_t`.


### 4. Pointers and Structures (Section 11.2)
* **Member Access Operator (`->`):** The arrow operator `rp->member` is a convenient shorthand equivalent to dereferencing and accessing a field (`(*rp).member`).
* **Modifying State via Pointers:** Passing pointers to structures allows functions to update the structure's fields directly without copying the entire object.
* **Takeaway 11.2 #1:** *Don't hide pointer types inside a `typedef`*. Hiding pointers behind a `typedef` (e.g., `typedef struct node* node;`) conceals the fact that `nullptr` is a valid input. Instead, declare `typedef struct node node;` and explicitly use `node*` in signatures.


### 5. Pointers and Arrays (Section 11.3)
* **Takeaway 11.3.1 #1:** *The two expressions `A[i]` and `*(A + i)` are equivalent*.
* **Takeaway 11.3.1 #2 (Array Decay):** *Evaluation of an array `A` returns `&A`*. Whenever an array is evaluated in a value context, it automatically decays into a pointer to its first element.
* **Takeaway 11.3.2 #1 (Parameter Rewriting):** *In a function declaration, any array parameter rewrites to a pointer*. For instance, `void f(double A)` is rewritten by the compiler to `void f(double* A)`.
* **Takeaway 11.3.2 #2:** *Only the innermost dimension of an array parameter is rewritten*. Multi-dimensional array parameters maintain outer bounds (e.g., `double A[n][m]` becomes `double (*A)[m]`).
* **Takeaway 11.3.2 #3:** *Declare length parameters before array parameters* (e.g., `void process(size_t len, double A[len])`).


### 6. Function Pointers (Section 11.4)
* **Takeaway 11.4 #1 (Function Decay):** *A function name without following parenthesis decays to a pointer to its start*.
* **Takeaway 11.4 #2:** *Function pointers must be used with their exact type*. Calling a function pointer with an incompatible prototype leads to undefined behavior due to platform ABI register conventions.
* **Takeaway 11.4 #3:** *The function call operator `(...)` applies to function pointers*.
* **Callback Dispatch:** Function pointers enable dynamic callback dispatch, such as passing comparison callbacks to standard library sorting (`qsort`) and searching (`bsearch`) utilities.

