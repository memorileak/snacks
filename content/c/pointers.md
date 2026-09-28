+++
title = "Modern C: Pointers"
description = "An overview of C23 pointers, covering address and dereference operators, pointer arithmetic and bounds, null pointers, struct and array access, and function callbacks."
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

#### Example

Pointers store memory addresses of objects or functions. The unary **address-of operator (`&`)** retrieves an object's memory location, while the unary **object-of operator (`*`)** dereferences a valid pointer to access or modify the underlying value.

```c
#include <stdio.h>
#include <stddef.h> // Provides ptrdiff_t and size_t
#include <stdbool.h>

// Modifies variables in the caller's stack frame via address references
void double_swap(double* p0, double* p1) {
    // Takeaway 11.1.1 #1: Dereferencing an invalid or null pointer causes program failure.
    if (!p0 || !p1) return; // Guard against nullptr

    double tmp = *p0; // Dereference p0 to read its value
    *p0 = *p1;        // Write value of *p1 into the location referenced by p0
    *p1 = tmp;        // Write original value into the location referenced by p1
}

void demo_pointer_operations(void) {
    double d0 = 3.5;
    double d1 = 10.0;

    // Pass object memory addresses using unary '&'
    double_swap(&d0, &d1);
    printf("Swapped: d0 = %g, d1 = %g\n", d0, d1); // Yields d0 = 10, d1 = 3.5

    // POINTER ARITHMETIC & DIFFERENCE:
    double weights = {1.0, 2.0, 3.0, 4.0};

    // Takeaway 11.1.2 #1: A valid pointer refers to the first element of an array.
    double const* p = &weights; // Points to element index 1
    double const* q = &weights; // Points to element index 3

    // Takeaway 11.1.3 #1: Only subtract pointers to elements of the same array.
    // Takeaway 11.1.3 #2 & #3: Pointer differences have type ptrdiff_t.
    ptrdiff_t diff = q - p; // Yields 2 (the index distance between elements)
    printf("Pointer distance (q - p): %td elements\n", diff);

    // Takeaway 11.1.3 #4: Cast pointers to (void*) when printing with %p.
    printf("Address of weights: %p\n", (void*)weights);
}
```

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

#### Example

Pointers have three states: **valid**, **null**, or **invalid**. In C23, `nullptr` replaces `NULL` to provide a type-safe null pointer constant of type `nullptr_t`.

```c
#include <stdio.h>
#include <stdbool.h>

void demo_pointer_validity_and_bounds(void) {
        // Takeaway 11.1.5 #1: Use C23 nullptr instead of the legacy NULL macro.
        double* ptr = nullptr;

        // Takeaway 11.1.4 #1: Pointers have a truth value (null evaluates to false).
        if (ptr) {
                printf("Pointer is valid.\n");
        } else {
                printf("Pointer is nullptr (evaluates to false in boolean context).\n");
        }

        double arr = {10.0, 20.0};
        double* p = &arr;

        // Takeaway 11.1.4 #5: A pointer must point to a valid object, one position beyond, or be null.
        p += 2; // Valid pointer pointing ONE element past array end (&arr)

        // Takeaway 11.1.4 #6: Computing a pointer beyond "one past the end" is illegal!
        // p += 2; // UNDEFINED BEHAVIOR: Out-of-bounds pointer calculation!

        // Takeaway 11.1.4 #4: Dereferencing a "one past the end" pointer is illegal!
        // double val = *p; // UNDEFINED BEHAVIOR: No object exists at index 2!

        // Takeaway 11.1.4 #2: Reset pointers to nullptr as soon as they become invalid.
        p = nullptr;
}
```

### 4. Pointers and Structures (Section 11.2)
* **Member Access Operator (`->`):** The arrow operator `rp->member` is a convenient shorthand equivalent to dereferencing and accessing a field (`(*rp).member`).
* **Modifying State via Pointers:** Passing pointers to structures allows functions to update the structure's fields directly without copying the entire object.
* **Takeaway 11.2 #1:** *Don't hide pointer types inside a `typedef`*. Hiding pointers behind a `typedef` (e.g., `typedef struct node* node;`) conceals the fact that `nullptr` is a valid input. Instead, declare `typedef struct node node;` and explicitly use `node*` in signatures.

#### Example

The **arrow operator (`->`)** provides convenient access to struct fields through a pointer. Gustedt warns against concealing pointers behind `typedef` definitions because doing so hides whether `nullptr` is a valid input.

```c
#include <stdio.h>
#include <stdbool.h>

// Forward declaration and typedef alias (Takeaway 11.2 #1: Do NOT hide pointers in typedefs!)
typedef struct rational rational;

struct rational {
        bool sign;
        size_t num;
        size_t denom;
};

// Functions mutating struct state receive pointers (e.g., rational* rp)
rational* rational_init(rational* rp, bool sign, size_t num, size_t denom) {
        if (!rp) return nullptr; // Guard against null pointer input

        // Arrow operator (rp->member) dereferences pointer and accesses member field
        rp->sign = sign;
        rp->num = num;
        rp->denom = (denom != 0) ? denom : 1;

        return rp; // Returns pointer for function chaining
}

void demo_pointers_and_structures(void) {
        rational r1 = {};

        // Explicit pointer visibility in function signature: rational_init(&r1, ...)
        if (rational_init(&r1, false, 3, 4)) {
                printf("Rational: %zu/%zu\n", r1.num, r1.denom);
        }
}
```

### 5. Pointers and Arrays (Section 11.3)
* **Takeaway 11.3.1 #1:** *The two expressions `A[i]` and `*(A + i)` are equivalent*.
* **Takeaway 11.3.1 #2 (Array Decay):** *Evaluation of an array `A` returns `&A`*. Whenever an array is evaluated in a value context, it automatically decays into a pointer to its first element.
* **Takeaway 11.3.2 #1 (Parameter Rewriting):** *In a function declaration, any array parameter rewrites to a pointer*. For instance, `void f(double A)` is rewritten by the compiler to `void f(double* A)`.
* **Takeaway 11.3.2 #2:** *Only the innermost dimension of an array parameter is rewritten*. Multi-dimensional array parameters maintain outer bounds (e.g., `double A[n][m]` becomes `double (*A)[m]`).
* **Takeaway 11.3.2 #3:** *Declare length parameters before array parameters* (e.g., `void process(size_t len, double A[len])`).

#### Example

While **pointers are not arrays**, `A[i]` and `*(A + i)` are syntactically equivalent. When an array expression is evaluated, it undergoes **array-to-pointer decay** and returns a pointer to its first element (`&A`). In function headers, array parameters automatically rewrite to pointers.

```c
#include <stdio.h>
#include <stddef.h>

// Takeaway 11.3.2 #1: Array parameters rewrite to pointers (double A[] becomes double* A).
// Takeaway 11.3.2 #3: Declare length parameters BEFORE array parameters.
// Takeaway 11.3.2 #4: Programmer must guarantee the validity of array arguments.
double sum_array(size_t len, double const A[len]) {
    // Takeaway 11.1.2 #2: Array size cannot be recovered via sizeof on pointer parameters!
    // sizeof A here yields size of pointer (8 bytes), NOT array size!

    double sum = 0.0;
    for (size_t i = 0; i < len; ++i) {
        // Takeaway 11.3.1 #1: A[i] and *(A + i) are strictly equivalent.
        sum += *(A + i);
    }
    return sum;
}

// Takeaway 11.3.2 #2: Only the innermost dimension of multi-dimensional arrays rewrites to a pointer.
// 'double C[n][m]' rewrites to 'double (*C)[m]' (pointer to an array of m doubles)
void matrix_zero(size_t n, size_t m, double C[n][m]) {
    for (size_t i = 0; i < n; ++i) {
        for (size_t j = 0; j < m; ++j) {
            C[i][j] = 0.0; // Indexing 2D VLA parameters cleanly
        }
    }
}

void demo_pointers_and_arrays(void) {
    double data = {10.0, 20.0, 30.0, 40.0};

    // Takeaway 11.3.1 #2: 'data' decays to &data when passed to sum_array
    double total = sum_array(4, data);
    printf("Total sum: %g\n", total);
}
```

### 6. Function Pointers (Section 11.4)
* **Takeaway 11.4 #1 (Function Decay):** *A function name without following parenthesis decays to a pointer to its start*.
* **Takeaway 11.4 #2:** *Function pointers must be used with their exact type*. Calling a function pointer with an incompatible prototype leads to undefined behavior due to platform ABI register conventions.
* **Takeaway 11.4 #3:** *The function call operator `(...)` applies to function pointers*.
* **Callback Dispatch:** Function pointers enable dynamic callback dispatch, such as passing comparison callbacks to standard library sorting (`qsort`) and searching (`bsearch`) utilities.

#### Example

A function name without trailing parentheses undergoes **function decay**, returning a pointer to the function's entry point. Function call operator `(...)` applies directly to function pointers.

```c
#include <stdio.h>
#include <stdlib.h>

// Comparison callback for qsort matching standard signature: int (*)(void const*, void const*)
int compare_doubles(void const* a_ptr, void const* b_ptr) {
    // Cast untyped void pointers back to exact typed pointers
    double const* a = a_ptr;
    double const* b = b_ptr;

    if (*a < *b) return -1;
    if (*a > *b) return +1;
    return 0;
}

void demo_function_pointers(void) {
    double values = {42.0, 3.14, 100.0, 1.0};

    // Takeaway 11.4 #1: Function name 'compare_doubles' decays to a function pointer.
    // Takeaway 11.4 #2: Function pointers must match their exact target signature!
    qsort(values, 4, sizeof(double), compare_doubles);

    // Call through function pointer variable
    int (*cmp_func)(void const*, void const*) = compare_doubles;

    // Takeaway 11.4 #3: Function call operator (...) applies directly to function pointers.
    int res = cmp_func(&values, &values);
    printf("Comparison result: %d (Sorted values = %g)\n", res, values);
}
```
