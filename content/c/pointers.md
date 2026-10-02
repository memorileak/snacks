+++
title = "Modern C: Pointers"
description = "An overview of C23 pointers, covering address and dereference operators, pointer arithmetic and bounds, null pointers, struct and array access, and function callbacks."
date = 2026-09-28T04:05:59Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Pointers are a crucial C feature that allow access to objects from different points in code and enable dynamic data structures. They break the barrier of pure functions, allowing functions to modify variables in the caller's scope.

## 11.1. Pointer operations

Pointers are scalar types that support specific operators to deal with the "pointing-to" and "pointed-to" relationships, as well as arithmetic operations.

### 11.1.1. Address-of and object-of operators

- **Address-of operator (`&`):** Retrieves the memory address of an object.
- **Object-of (dereference) operator (`*`):** Accesses the object located at the pointer's address.
- **Takeaway 11.1.1 #1:** A program execution that uses `*` with an invalid or null pointer fails.

```c
#include <stdio.h>

// Concept: Pointers allow functions to modify caller variables
void double_swap(double* p0, double* p1) {
    double tmp = *p0; // '*' (object-of) retrieves the value
    *p0 = *p1;
    *p1 = tmp;
}

int main(void) {
    double d0 = 3.5, d1 = 10.0;

    // Concept: '&' (address-of) passes the variable's address
    double_swap(&d0, &d1);

    // Takeaway 11.1.1 #1: Dereferencing a valid pointer is safe.
    // Dereferencing an invalid/null pointer would crash the program here.
    double* p_invalid = nullptr;
    // *p_invalid = 5.0; // ERROR: Program execution fails.

    return 0;
}
```

### 11.1.2. Pointer addition

- **Takeaway 11.1.2 #1:** A valid pointer refers to the first element of an array of the reference type.
- **Takeaway 11.1.2 #2:** The length of an array object cannot be reconstructed from a pointer.
- **Takeaway 11.1.2 #3:** Pointers are not arrays.
- **Concept:** Because pointers lose array length information, it is best practice to pass the length alongside the array and use array notation in function parameters to clarify intent.

```c
#include <stdio.h>

// Concept: Prefer array notation with size to clarify intent,
// even though 'a' decays to a pointer.
double sum(size_t len, double const a[len]) {
    double total = 0;
    for(size_t i = 0; i < len; ++i) {
        total += a[i];
    }
    return total;
}

int main(void) {
    double arr[3] = {1.0, 2.0, 3.0};

    // Takeaway 11.1.2 #1 & #3: 'ptr' refers to the first element, but is NOT the array itself.
    double* ptr = &arr[0];

    // Takeaway 11.1.2 #2: We cannot determine 'arr' has 3 elements just from 'ptr'.
    // We must pass the length explicitly.
    printf("Sum: %f\n", sum(3, ptr));
    return 0;
}
```

### 11.1.3. Pointer subtraction and difference

- **Takeaway 11.1.3 #1:** Only subtract pointers to elements of the same array object.
- **Takeaway 11.1.3 #2:** All pointer differences have type `ptrdiff_t`.
- **Takeaway 11.1.3 #3:** Use `ptrdiff_t` to encode signed differences of positions or sizes.
- **Takeaway 11.1.3 #4:** For printing, cast pointer values to `void*` and use the format `%p`.

```c
#include <stdio.h>
#include <stddef.h> // For ptrdiff_t

int main(void) {
    double arr[5] = {0};
    double* p1 = &arr[1];
    double* p2 = &arr[4];

    // Takeaway 11.1.3 #1, #2, & #3: Subtracting pointers in the same array
    // yields a ptrdiff_t representing the distance in elements.
    ptrdiff_t diff = p2 - p1;
    printf("Difference in elements: %td\n", diff);

    // Takeaway 11.1.3 #4: Cast to void* and use %p to print pointer addresses.
    printf("Address of p1: %p\n", (void*)p1);

    return 0;
}
```

### 11.1.4. Pointer validity

- **Takeaway 11.1.4 #1:** Pointers have a truth value (evaluate to `false` if null, `true` if valid).
- **Takeaway 11.1.4 #2:** Set pointer variables to null as soon as you can.
- **Takeaway 11.1.4 #3:** A program execution that accesses an object that has a non-value representation for its type fails.
- **Takeaway 11.1.4 #5:** A pointer must point to a valid object, one position beyond, or be null.
- **Takeaway 11.1.4 #6:** A program execution that computes a pointer value outside the bounds of an array object (or one element beyond) fails.

```c
#include <stdio.h>

int main(void) {
    // Takeaway 11.1.4 #2: Initialize pointers to null immediately.
    char const* name = nullptr;

    // Takeaway 11.1.4 #1: Pointers have a truth value.
    if (name) {
        printf("Name is %s\n", name);
    } else {
        printf("Pointer is null.\n");
    }

    double A[2] = { 0.0, 1.0 };
    double* p = &A[0];

    p += 2; // Takeaway 11.1.4 #5: Valid pointer, points one position beyond the array.
    // printf("%f", *p); // ERROR: Accessing non-object fails (Takeaway 11.1.4 #3).

    // p += 3; // ERROR: Takeaway 11.1.4 #6: Computing value outside bounds + 1 fails.

    return 0;
}
```

### 11.1.5. Null pointers

- **Concept:** The pre-C23 macro `NULL` hides type implementations (e.g., `0`, `0L`, `(void*)0`) and can cause backward compatibility or variadic function issues.
- **Takeaway 11.1.5 #1:** Use `nullptr` instead of `NULL`.

```c
#include <stdio.h>

int main(void) {
    // Takeaway 11.1.5 #1: Use the C23 standard nullptr instead of NULL macro.
    double const* const nix = nullptr;

    if (!nix) {
        printf("Properly using nullptr.\n");
    }
    return 0;
}
```

## 11.2. Pointers and structures

- **Concept:** Pointers are heavily used to manipulate structs directly. The `->` operator is used to access struct members via a pointer (it is equivalent to `(*ptr).member`).
- **Concept:** Opaque structures allow separating interfaces from implementation by forward-declaring the struct.
- **Takeaway 11.2 #1:** Don’t hide pointer types inside a `typedef`. Doing so obscures the fact that the type is a pointer and can receive `nullptr`.

```c
#include <stdio.h>

// Concept: forward declaration of a struct (Opaque structure)
typedef struct toto toto; // Good: typedef aliases the struct, not the pointer

// Takeaway 11.2 #1: Avoid doing `typedef struct toto_s* toto;`

struct toto {
    unsigned value;
};

void toto_doit(toto* t, unsigned val) {
    if (t) { // Check for nullptr validity
        // Concept: Use -> to access members of a struct pointer
        t->value = val;
    }
}

int main(void) {
    toto my_toto = {0};
    toto_doit(&my_toto, 42);
    printf("Value: %u\n", my_toto.value);
    return 0;
}
```

## 11.3. Pointers and arrays

### 11.3.1. Array and pointer access are the same

- **Takeaway 11.3.1 #1:** The two expressions `A[i]` and `*(A+i)` are equivalent.
- **Concept:** Array-to-pointer decay. When arrays are passed to functions, they seamlessly decay into pointers to their first element.

```c
#include <stdio.h>

int main(void) {
    int A[3] = {10, 20, 30};
    int* ptr = A; // Array decays to pointer

    // Takeaway 11.3.1 #1: A[i] and *(A+i) are completely equivalent.
    printf("A[1]: %d\n", A[1]);
    printf("*(A+1): %d\n", *(A + 1));

    // This applies to the pointer variable as well
    printf("ptr[1]: %d\n", ptr[1]);
    printf("*(ptr+1): %d\n", *(ptr + 1));

    return 0;
}
```

## 11.4. Function pointers

- **Concept:** Pointers can refer to executable code (functions), allowing dynamic dispatch (e.g., callbacks).
- **Takeaway 11.4 #1:** A function name without following parenthesis decays to a pointer to its start.
- **Takeaway 11.4 #2:** Function pointers must be used with their exact type (signatures must perfectly match).
- **Takeaway 11.4 #3:** The function call operator `()` applies to function pointers.

```c
#include <stdio.h>

// Typedef for a function pointer
typedef void logger_function(char const* msg);

void log_console(char const* msg) {
    printf("LOG: %s\n", msg);
}

int main(void) {
    // Takeaway 11.4 #1: function name 'log_console' decays to a pointer.
    // Takeaway 11.4 #2: Exact type match is required.
    logger_function* logger = log_console;

    // Takeaway 11.4 #3: Function call operator applies directly to the pointer.
    logger("System started.");

    return 0;
}
```
