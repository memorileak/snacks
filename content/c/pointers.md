+++
title = "Modern C: Pointers"
description = "An overview of C23 pointers, covering address and dereference operators, pointer arithmetic and bounds, null pointers, struct and array access, and function callbacks."
date = 2026-09-28T04:05:59Z

[taxonomies]
tags = ["modernc", "pointer"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## Chapter 11. Pointers

Pointers often represent the first significant hurdle to achieving a deep understanding of C. They are essential for accessing objects from different points in a program's execution, structuring data dynamically, and abstracting hardware specifics to write portable code.

The term **pointer** refers to a specially derived type construct that "points" or "refers" to another object. Pointers break the barrier between the caller of a function and the code inside a function, allowing you to write functions that can directly change the variables of the calling function (i.e., functions that are not _pure_).

---

## 11.1 Pointer Operations

This section details the specific C operations and features designed exclusively for pointers. Pointers are considered **scalars**, meaning arithmetic operations like offset addition and subtraction are defined for them.

### 11.1.1 The Address-of and Object-of Operators

**What it is:**
To use pointers to interact with external variables, we need two fundamental operators. The **address-of** operator (`&`) returns the memory address of an object. Its inverse, the **object-of** operator (or dereference operator, `*`), takes a pointer and refers back to the original object.

**Why it matters / how it works:**
By passing the address of an object (e.g., `&d0`) to a function, the function can receive that address in a pointer variable (e.g., `double* p0`). The function doesn't know the name of the original variable `d0`, but by using the object-of operator (`*p0`), it can read and modify the value stored at that address directly. Note that the `*` character plays two roles: in a declaration (like `double* p0;`), it modifies a type to create a pointer type; in an expression (like `*p0 = *p1;`), it dereferences a pointer to access its underlying object.

**Key details:**

- The `&` operator generates a pointer type from an existing object.
- The `*` operator dereferences a pointer to access the target object.
- For clarity, the `*` is typically flushed left with the type in declarations (e.g., `double* p0;`) and flushed right with the variable in expressions (e.g., `*p0`).

**Pitfalls:**
Dereferencing pointers requires extreme caution. If a pointer is null or invalid, attempting to dereference it will cause the program to crash or behave unpredictably.

> _A program execution that uses `*` with an invalid or null pointer fails._

If the pointer is invalid (e.g., uninitialized), it might access a random memory object, leading to bugs that are incredibly difficult to trace. If it is null, the program will typically crash immediately, which is often considered a helpful feature during debugging.

**Connections:**
This connects directly to the concept of pointer states introduced earlier in the book (valid, null, invalid).

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// Function that receives pointers to doubles
void double_swap(double* p0, double* p1) {
    // *p0 refers to the object pointed to by p0
    auto tmp = *p0; // Stores the value of the first object
    *p0 = *p1;      // Overwrites the first object with the second object's value
    *p1 = tmp;      // Overwrites the second object with the stored temporary value
}

int main(void) {
    double d0 = 3.5;
    double d1 = 10.0;

    // Passing the addresses of d0 and d1 using the '&' operator
    double_swap(&d0, &d1);

    printf("d0: %g, d1: %g\n", d0, d1);

    // What goes wrong if...
    // double* bad_ptr = nullptr;
    // double_swap(&d0, bad_ptr);
    // ^ This would cause a crash when double_swap tries to evaluate *p1.

    return EXIT_SUCCESS;
}
// Expected output:
// d0: 10, d1: 3.5

```

**Recap:**

- Use `&` to get the address of an object.
- Use `*` to dereference a pointer and access the object.
- Never dereference a null or invalid pointer.

---

### 11.1.2 Pointer Addition

**What it is:**
Pointer addition allows a program to calculate offsets from a specific memory address. When you add an integer to a valid pointer, the pointer shifts by that many elements of its base type.

**Why it matters / how it works:**
In C, a valid pointer isn't restricted to pointing at a single, isolated instance of its reference type; it can point to the first element of an array of unknown length _n_.

> _A valid pointer refers to the first element of an array of the reference type._

This means pointer addition is intrinsically linked to array traversal.

> _The length of an array object cannot be reconstructed from a pointer._

Because the pointer merely holds an address, it has no built-in knowledge of how large the underlying array is.

> _Pointers are not arrays._

Despite their syntactic similarities, they are fundamentally distinct: arrays contain the data directly and have a fixed size, while pointers only refer to data locations and have no inherent size bounds.

**Key details:**

- A pointer essentially acts as an entry point into an array.
- The syntax and mechanics entangle the concepts of pointers and arrays tightly.

**Pitfalls:**
Because a pointer loses the size information of the original array, performing pointer addition without external bounds-checking can easily lead to out-of-bounds access.

**Connections:**
This is the foundational logic that allows C to rewrite array parameters in functions into simple pointers.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    double A[3] = { 1.1, 2.2, 3.3 };
    double* p = &A[0]; // p refers to the first element

    // Pointer addition advances the pointer by elements, not bytes
    double* next_p = p + 1;

    printf("First element: %g\n", *p);
    printf("Second element: %g\n", *next_p);

    return EXIT_SUCCESS;
}
// Expected output:
// First element: 1.1
// Second element: 2.2

```

**Recap:**

- Adding to a pointer advances it sequentially through an array structure.
- Pointers do not retain the length of the array they point to.
- Pointers and arrays are conceptually distinct, even when used similarly.

---

### 11.1.3 Pointer Subtraction and Differences

**What it is:**
C permits the subtraction of one pointer from another, provided both point to elements within the same array object.

**Why it matters / how it works:**
Subtracting two pointers calculates the scalar "distance" between them in terms of array elements, not bytes.

> _Only subtract pointers to elements of the same array object._

If you subtract pointers pointing to completely unrelated objects, the operation is meaningless and leads to undefined behavior.

> _All pointer differences have type `ptrdiff_t`._

> _Use `ptrdiff_t` to encode signed differences of positions or sizes._

Because pointer subtraction can yield negative values (if the first pointer precedes the second), the result must be stored in a signed integer type, which the standard defines as `ptrdiff_t`.

> _For printing, cast pointer values to `void*` and use the format `%p`._

When printing the memory address stored inside a pointer for debugging, you must cast it to `void*` and use the `%p` format specifier to ensure it is handled correctly by `printf`.

**Key details:**

- Pointer difference represents element counts.
- `ptrdiff_t` is the standard signed type for handling these differences.
- Pointer formatting uses `%p` paired with a `(void*)` cast.

**Pitfalls:**
Attempting to measure the distance between two distinct, unassociated variables by subtracting their pointers results in undefined behavior.

**Connections:**
The `ptrdiff_t` type acts as the signed counterpart to `size_t` when dealing with memory distances.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h> // Required for ptrdiff_t

int main(void) {
    double A[5] = { 0.0, 1.0, 2.0, 3.0, 4.0 };
    double* p_start = &A[0];
    double* p_end = &A[4];

    // Calculating the distance between two pointers in the same array
    ptrdiff_t diff = p_end - p_start;

    printf("Distance: %td elements\n", diff);

    // Printing a pointer value safely
    printf("Address of p_start: %p\n", (void*)p_start);

    return EXIT_SUCCESS;
}
// Expected behavior: diff will be 4.
// What goes wrong if...
// double x = 0, y = 0;
// ptrdiff_t bad_diff = &x - &y;
// ^ Undefined behavior because x and y are not part of the same array object.

```

**Recap:**

- Pointer subtraction measures distance in elements.
- Use `ptrdiff_t` to capture this distance.
- Never subtract pointers from different array objects.

---

### 11.1.4 Pointer Validity

**What it is:**
Pointers hold values (addresses), and those values can change. It is vital to manage whether a pointer holds a valid address, an invalid address, or a null state.

**Why it matters / how it works:**
Controlling pointer state prevents catastrophic memory access errors.

> _Pointers have a truth value._

A valid pointer evaluates to `true` in a logical expression, while a null pointer evaluates to `false`. This makes it simple to check if a pointer is ready to be dereferenced without using clunky comparisons.

> _Set pointer variables to null as soon as you can._

We must ensure pointer variables remain null unless they deliberately point to an object we want to manipulate. Explicitly initializing pointers guarantees this baseline safety.

> _A program execution that accesses an object that has a non-value representation for its type fails._

If a pointer interprets memory incorrectly, interpreting bits as a type when they form a non-value (or trap) representation for that type, the program will fail.

> _When dereferenced, a pointed-to object must be of the designated type._

You cannot use a pointer to arbitrarily treat a `size_t` object as a `double`; C's type rules forbid this mixup.

> _A program execution that computes a pointer value outside the bounds of an array object (or one element beyond) fails._

Accessing beyond the bounds of an array refers to a memory region whose type and state are entirely unknown. Even just _computing_ an invalid pointer address offset (e.g., adding 3 to a 2-element array pointer) can cause a failure before dereferencing even occurs.

**Key details:**

- Pointers implicitly resolve to boolean values (`false` if null).
- Null initialization is critical.
- Type matching during dereferencing is mandatory.

**Pitfalls:**
Failing to initialize a pointer leaves it in an invalid, indeterminate state. Dereferencing it, or adding out-of-bounds offsets to it, triggers immediate program failure or subtle corruption.

**Connections:**
This expands upon the "invalid pointers lead to program failure" concept from earlier chapters.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    char const* name = nullptr; // Initialized immediately to null

    // Testing the truth value of the pointer
    if (name) {
        printf("Today's name is %s\n", name);
    } else {
        printf("Today we are anonymous\n");
    }

    double A[2] = { 0.0, 1.0 };
    double* p = &A[0];

    // Valid pointer addition
    ++p;

    // What goes wrong if...
    // p += 2; // Valid pointer computation (one past the end), but...
    // printf("Element: %g\n", *p); // Referencing a non-object -> Program failure!

    // p += 3; // Invalid pointer addition -> Program failure before dereferencing!

    return EXIT_SUCCESS;
}
// Expected output:
// Today we are anonymous

```

**Recap:**

- Test pointers logically to ensure they aren't null before using them.
- Always initialize unused pointers to `nullptr`.
- Never compute pointer addresses beyond an array's bounds.

---

### 11.1.5 Null Pointers

**What it is:**
A null pointer represents a pointer that is intentionally pointing to "nothing." C23 introduces the keyword `nullptr` for this purpose.

**Why it matters / how it works:**
Historically, C used the `NULL` macro (often expanding to `0`, `0L`, or `(void*)0`) to represent a "null pointer constant". However, this concept was flawed because `NULL` was not strictly a pointer type—it could just be an integer constant. Depending on the platform, passing `NULL` to certain functions (especially those with a variable number of arguments) could lead to dangerous undefined behavior or crashes.

> _Use `nullptr` instead of `NULL`._

The introduction of `nullptr` in C23 provides a true, generic null pointer value that avoids the ambiguities and backward-compatibility messes of the old `NULL` macro.

**Key details:**

- `NULL` expansions historically varied (`0U`, `0`, `0LL`, `(void*)0`).
- `nullptr` is standardized and strictly treated as a pointer type.

**Pitfalls:**
Using `NULL` can hide the actual underlying type from the compiler, leading to subtle bugs on platforms where pointer widths differ from integer widths.

**Connections:**
This resolves the "generic pointer of value 0" ambiguity that plagued pre-C23 programming.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    // Modern C23 way to declare a null pointer
    double const* const nix = nullptr;

    if (!nix) {
        printf("The pointer is cleanly null.\n");
    }

    return EXIT_SUCCESS;
}
// Expected output:
// The pointer is cleanly null.

```

**Recap:**

- The `NULL` macro is historically inconsistent.
- Always use `nullptr` in modern C23 development.

---

## 11.2 Pointers and Structures

**What it is:**
Pointers are frequently used to pass structures to functions, allowing the functions to modify the structure's contents directly.

**Why it matters / how it works:**
To manipulate a structure via a pointer, you would normally have to dereference the pointer and then use the dot operator (e.g., `(*a).tv_sec`). Because this is syntactically clumsy, C provides the `->` operator. The `->` operator combines dereferencing and member access: `a->tv_sec` is perfectly equivalent to `(*a).tv_sec`.
Additionally, C allows pointers to **opaque structures**—structures that are declared but not fully defined. This is heavily used to separate a library's interface from its implementation, strictly hiding the data layout from the user while still allowing pointers to be passed around.

**Key details:**

- `->` represents a pointer on the left and a structure member on the right.
- Expressions like `a->tv_nsec` evaluate to the member's object type (e.g., `long`), not a pointer.
- Opaque structs hide the structure definition from the header file, only providing a pointer to it.

**Pitfalls:**
A common mistake is using the `.` operator on a structure pointer instead of `->`, which causes a compile-time error. Another pitfall is hiding pointers inside `typedef`s (e.g., `typedef struct toto_s* toto;`), which obscures the fact that the type can accept a null pointer.

**Connections:**
Using structure pointers avoids the overhead of copying large structs by value (pass-by-value), aligning with the performance goals of modern C.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// A simple structure
struct time_data {
    long tv_sec;
    long tv_nsec;
};

// Function taking a pointer to a struct
void time_normalize(struct time_data* t) {
    if (t) {
        // Using the -> operator for clean access
        if (t->tv_nsec >= 1000000000L) {
            t->tv_sec += t->tv_nsec / 1000000000L;
            t->tv_nsec = t->tv_nsec % 1000000000L;
        }
    }
}

int main(void) {
    struct time_data my_time = { .tv_sec = 1, .tv_nsec = 1500000000L };

    time_normalize(&my_time);
    printf("Seconds: %ld, Nanoseconds: %ld\n", my_time.tv_sec, my_time.tv_nsec);

    // What goes wrong if...
    // my_time.tv_sec // correct
    // (&my_time).tv_sec // Compile error! Must be (&my_time)->tv_sec

    return EXIT_SUCCESS;
}
// Expected output:
// Seconds: 2, Nanoseconds: 500000000

```

**Recap:**

- Use `->` to access struct members through a pointer.
- Pointers allow opaque structure designs for robust library interfaces.
- Avoid hiding pointer types behind a `typedef`.

---

## 11.3 Pointers and Arrays

**What it is:**
C uses the exact same syntax for pointer and array element access, and automatically rewrites array parameters in function declarations into pointers.

**Why it matters / how it works:**

> _The two expressions `A[i]` and `*(A+i)` are equivalent._

Whether `A` is an array or a pointer, applying the bracket notation `A[i]` is just syntactic sugar for pointer arithmetic `*(A+i)`. If `A` is an array, this demonstrates **array-to-pointer decay**, where the array implicitly converts to a pointer to its first element to satisfy the expression.

> _In a function declaration, any array parameter rewrites to a pointer._

When you declare a function like `size_t strlen(char const s[static 1]);`, C silently rewrites it to `size_t strlen(char const* s);`. This is why arrays passed to functions do not copy their contents; the function only receives a pointer to the original array.

**Key details:**

- Bracket syntax `[]` works identically on pointers and arrays.
- Array parameters in function signatures decay entirely into pointers.

**Pitfalls:**
Novices often believe that passing an array to a function passes the whole array by value. Because of array decay, `sizeof(s)` inside the function will return the size of the _pointer_, not the array.

**Connections:**
This connects to the earlier rule that "pointers are not arrays", explaining why the compiler can interoperate between them despite their underlying differences.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// Array parameter rewritten to a pointer by the compiler
void print_first_two(double const A[static 2]) {
    // Bracket access and pointer arithmetic are identical
    printf("First: %g\n", A[0]);
    printf("Second: %g\n", *(A + 1));
}

int main(void) {
    double data[4] = { 10.0, 20.0, 30.0, 40.0 };

    print_first_two(data); // Array implicitly decays to pointer

    return EXIT_SUCCESS;
}
// Expected output:
// First: 10
// Second: 20

```

**Recap:**

- `A[i]` means `*(A+i)`.
- Arrays naturally decay to pointers when passed to functions.
- Function signatures with array parameters are technically pointer parameters.

---

## 11.4 Function Pointers

**What it is:**
Just as you can point to objects in memory, you can take the address of a function and store it in a **function pointer**.

**Why it matters / how it works:**

> _A function name without following parenthesis decays to a pointer to its start._

If you use a function's name (e.g., `printf`) without invoking it, it automatically decays into a pointer pointing to the executable code's start address. Function pointers allow for dynamic execution choices (e.g., selecting a logging function at runtime).

> _Function pointers must be used with their exact type._

> _The function call operator (...) applies to function pointers._

Technically, all function calls in C operate via function pointers; the normal syntax evaluates the identifier to a function pointer before executing it. Because function types dictate the calling convention and argument sizes, function pointers must strictly match the signature they point to.

**Key details:**

- Function pointers can be stored in arrays to create jump tables.
- Calling through a function pointer introduces a slight overhead (indirection) because the compiler must first fetch the address.

**Pitfalls:**
Assigning a function to a function pointer with a mismatched prototype circumvents the type system and causes undefined behavior when called. Also, using function pointers in highly time-critical code can degrade performance due to the indirection.

**Connections:**
Function pointers enable callback architectures, famously used by C standard library functions like `qsort`, `bsearch`, and `atexit`.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// Typedef for a function pointer that takes an int and returns void
typedef void (*logger_function)(int);

void log_verbose(int code) {
    printf("Verbose log: Exit code %d\n", code);
}

void log_ignore(int code) {
    // Do nothing
}

int main(void) {
    // Array of function pointers
    logger_function loggers[2] = { log_verbose, log_ignore };

    int use_verbose = 1;

    // Dynamically selecting the function to call
    logger_function current_logger = use_verbose ? loggers[0] : loggers[1];

    // Executing the function via the pointer
    current_logger(42);

    return EXIT_SUCCESS;
}
// Expected output:
// Verbose log: Exit code 42

```

**Recap:**

- Function names decay to pointers.
- Function pointers can be invoked identically to direct functions.
- Signatures must match perfectly.

---

## Summary

Chapter 11 introduces pointers as a powerful abstraction that allows C programs to break the boundaries of pure functions by directly interacting with memory. The chapter outlines the specific operations available for pointers: fetching addresses with `&`, dereferencing with `*`, and calculating bounds with addition and subtraction (`ptrdiff_t`). It heavily emphasizes pointer state management, asserting that pointers must be validated or set to the new C23 standard `nullptr` to prevent undefined access failures. Pointers play critical roles when interacting with structures (using the `->` operator and opaque designs) and arrays (via pointer-array decay equivalence). Finally, the chapter demonstrates that functions themselves can be managed through pointers, facilitating dynamic and flexible execution pathways.

---

## Self-Check Questions

1. **What happens if you apply the dereference operator (`*`) to an invalid or null pointer?**
   _Answer:_ The program execution fails. An invalid pointer may corrupt random memory, while a null pointer typically causes an immediate crash.

2. **Can you determine the size of an array strictly from a pointer to its first element?**
   _Answer:_ No, the length of an array object cannot be reconstructed from a pointer; pointer arrays lose their dimension information.

3. **What is the standard data type for storing the result of subtracting two pointers?**
   _Answer:_ The signed integer type `ptrdiff_t`.

4. **Under what condition is it valid to subtract two pointers?**
   _Answer:_ You may only subtract pointers if they point to elements within the exact same array object.

5. **How should an unassigned pointer variable be initialized in C23?**
   _Answer:_ It should be initialized with the keyword `nullptr` as soon as possible.

6. **If you have a pointer to a structure `p` containing a member `val`, what are the two syntactically equivalent ways to access it?**
   _Answer:_ `(*p).val` and `p->val`.

7. **How does the C compiler interpret the array access `A[i]`?**
   _Answer:_ It interprets it exactly as the pointer arithmetic expression `*(A+i)`.

8. **What does the C compiler automatically do to array parameters declared in a function signature?**
   _Answer:_ It rewrites them into pointer parameters.

9. **What happens to a function identifier if it is used in an expression without parentheses?**
   _Answer:_ It automatically decays into a pointer to its start address.

10. **Why should function pointers be avoided in highly time-critical code?**
    _Answer:_ They introduce a layer of indirection, requiring the program to fetch the memory address before executing the function, which carries an overhead.
