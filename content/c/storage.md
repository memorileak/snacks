+++
title = "Modern C: Storage"
description = "A guide to C23 storage duration and object lifetimes, dynamic allocation and deallocation, initialization APIs, VLAs, and the execution stack."
date = 2026-09-28T04:53:34Z

[taxonomies]
tags = ["modernc", "malloc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

This chapter covers how C manages memory through dynamic allocation, the rules of storage duration and initialization, and the precise definitions of object lifetimes.

## 13.1 malloc and friends

Dynamic allocation allows programs to request memory on the fly to handle varying data sizes. The standard library `<stdlib.h>` provides `malloc`, `free`, `calloc`, `realloc`, and `aligned_alloc`. These functions operate with `void*`, meaning the memory is just raw bytes until used.

- **Takeaway 13.1 #1 & #5:** Only use allocation functions with a size strictly greater than zero. Storage allocated through `malloc` is uninitialized and has no type.
- **Takeaway 13.1 #2 & 13.1.1 #1:** Failed allocations result in a null pointer. Always check the return value!
- **Takeaway 13.1 #3:** Prefer the use of `strndup` over `strdup` to prevent out-of-bounds reading.
- **Takeaway 13.1 #4:** Do not cast the return of `malloc` and friends (e.g., `(int*)malloc(...)` is bad practice in C).

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    // Takeaway 13.1 #1, #4, #5: Allocate memory without casting.
    size_t length = 5;
    double* largeVec = malloc(length * sizeof *largeVec);

    // Takeaway 13.1 #2: Check for failure
    if (largeVec) {
        for(size_t i = 0; i < length; ++i) {
            largeVec[i] = 0.0; // Now the memory acquires an effective type
        }
        free(largeVec);
    }

    // Takeaway 13.1 #3: Prefer strndup
    const char* source = "Hello World";
    char* safe_str = strndup(source, 5); // Copies exactly 5 chars
    if (safe_str) {
        printf("%s\n", safe_str); // Prints "Hello"
        free(safe_str);
    }

    return 0;
}
```

## 13.1.2 Ensuring consistency of dynamic allocations

Memory leaks (loss of allocated objects) and memory corruption occur when allocations and deallocations do not match perfectly.

- **Takeaway 13.1.2 #1 & #2:** For every allocation (`malloc`, `calloc`, etc.), there must be exactly one `free`.
- **Takeaway 13.1.2 #3:** Only call `free` with pointers exactly as they were returned by the allocation functions. Never free variables, compound literals, or offsets of pointers.

```c
#include <stdlib.h>

int main(void) {
    int* ptr = calloc(10, sizeof(int));
    if (ptr) {
        // Do work...

        // Takeaway 13.1.2: Matching free.
        // Freeing (ptr + 1) or forgetting this step is catastrophic.
        free(ptr);
    }
    return 0;
}
```

## 13.1.3 Flexible array members

A Flexible Array Member (FLA) is an array without a specified size declared as the _last_ member of a struct. It lets you allocate the struct and the array buffer in a single block of memory.

- **Takeaway 13.1.3 #1:** You must manually allocate enough storage to access the structure AND the array elements.
- **Takeaway 13.1.3 #2:** Consistency between a length member and the flexible array must be maintained manually.

```c
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>

// Struct with a flexible array member
typedef struct {
    size_t length;
    uint32_t data[]; // Flexible array must be the last member
} ua32;

int main(void) {
    size_t len = 10;
    // Takeaway 13.1.3 #1: Calculate total size (struct + array elements)
    size_t size = offsetof(ua32, data) + sizeof(uint32_t[len]);
    if (size < sizeof(ua32)) { size = sizeof(ua32); }

    ua32* ap = calloc(1, size);
    if (ap) {
        // Takeaway 13.1.3 #2: Manually store the length to maintain consistency
        ap->length = len;
        ap->data[0] = 42;
        printf("First element: %u\n", ap->data[0]);
        free(ap);
    }
    return 0;
}
```

## 13.2 Storage duration, lifetime, and visibility

Visibility dictates where a name is valid. Lifetime dictates when an object actually exists in memory.

- **Takeaway 13.2 #3:** Every definition creates a new, distinct object (even if shadowed).
- **Takeaway 13.2 #5 & #6:** Objects have a strict lifetime. Accessing an object outside its lifetime fails.
- **Takeaway 13.2 #7:** A compound literal has the same lifetime as a variable declared in the same scope.

```c
#include <stdio.h>

void use_pointer(double* p) {
    if (p) printf("Value: %f\n", *p);
}

int main(void) {
    double* ptr = NULL;
    {
        // Lifetime of 'x' is bound to this inner block
        double x = 35.0;
        ptr = &x;
        use_pointer(ptr); // Valid
    }
    // use_pointer(ptr); // ERROR (Takeaway 13.2 #6): 'x' is dead here.

    // Takeaway 13.2 #7: Compound literal bound to the main() block
    double* p_comp = &(double){ 99.0 };
    use_pointer(p_comp); // Valid until main() returns

    return 0;
}
```

## 13.2.1 Static storage duration

Objects with static storage duration live for the entire program execution.

- **Takeaway 13.2.1 #1:** Objects with static storage duration are _always_ initialized. If not explicitly initialized, they default to 0 (or null).

```c
#include <stdio.h>

double global_val; // File scope: implicitly 0.0

int main(void) {
    // Block scope, but static lifetime: implicitly 0.0
    static double block_static;

    printf("Global: %f, Static Local: %f\n", global_val, block_static);
    return 0;
}
```

## 13.2.2 Automatic storage duration

Variables defined inside a block (without `static`) have automatic storage duration.

- **Takeaway 13.2.2 #3, #4 & #5:** Use `register` for critical local variables. You cannot use the `&` operator on a `register` variable, preventing aliasing.
- **Takeaway 13.2.2 #7 & #8:** Temporary objects (like arrays returned inside a struct) are read-only, and their lifetime ends immediately after the expression finishes evaluating.

```c
#include <stdio.h>

struct demo { unsigned ory[1]; };
struct demo get_demo(void) {
    return (struct demo){ .ory = {42} };
}

int main(void) {
    // Takeaway 13.2.2 #5: register keyword for optimization
    register int counter = 0;
    counter++;
    // int* p = &counter; // ERROR (Takeaway 13.2.2 #3): Cannot take address

    // Takeaway 13.2.2 #7 & #8: get_demo().ory is a temporary read-only object.
    // It dies completely on the next line.
    printf("Temporary: %u\n", get_demo().ory[0]);

    return 0;
}
```

## 13.3 Object Lifetime

Understanding exactly _when_ an object starts and stops existing is vital, especially inside loops and with Variable Length Arrays (VLAs).

- **Takeaway 13.3 #1 & #2:** For standard block variables and compound literals, lifetime starts when the block scope is entered. Initializers are evaluated _each time_ the definition is met.
- **Takeaway 13.3 #3:** For a VLA, lifetime starts only when the declaration is _encountered_ during execution.

```c
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 3; i++) {
        // Takeaway 13.3 #2: This compound literal is evaluated and initialized
        // to a new value (0, 1, 2) on EVERY iteration.
        size_t* ip = &(size_t){ i };
        printf("%zu ", *ip);
    }
    printf("\n");
    return 0;
}
```

## 13.4 Initialization

C does not automatically clean up memory for you unless it's static/global.

- **Takeaway 13.4 #1 & #2:** Static objects default to 0. Automatic (local) and allocated (`malloc`) objects _must_ be initialized explicitly by you.
- **Takeaway 13.4 #3:** Systematically provide a dedicated initialization function (e.g., `type_init`) for your complex data types.

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    long long numerator;
    unsigned long long denominator;
} rat;

// Takeaway 13.4 #3: Systematic initialization function
rat* rat_init(rat* p, long long num, unsigned long long den) {
    if (p) {
        p->numerator = num;
        p->denominator = den;
    }
    return p; // Return the pointer to allow chaining
}

int main(void) {
    // Takeaway 13.4 #2: Explicitly initializing dynamically allocated memory
    rat* my_rat = rat_init(malloc(sizeof(rat)), 13, 7);

    if (my_rat) {
        printf("Ratio: %lld/%llu\n", my_rat->numerator, my_rat->denominator);
        free(my_rat);
    }
    return 0;
}
```
