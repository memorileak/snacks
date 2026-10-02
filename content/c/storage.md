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

## 13.1. malloc and friends

- Dynamic allocation allows reclaiming storage instances for objects on the fly and releasing them when no longer needed.
- Functions include `malloc`, `calloc`, `realloc`, `aligned_alloc`, and `free` (from `<stdlib.h>`), plus POSIX additions `strdup` and `strndup` (from `<string.h>`).

**Takeaways:**

- **Takeaway 13.1 #1:** Only use the allocation functions with a size strictly greater than zero.
- **Takeaway 13.1 #2:** Failed allocations result in a null pointer.
- **Takeaway 13.1 #3:** Prefer the use of `strndup` over `strdup` (prevents reading beyond buffer if not 0-terminated).
- **Takeaway 13.1 #4:** Don’t cast the return of `malloc` and friends. (They return `void*`, which automatically converts to any pointer type).
- **Takeaway 13.1 #5:** Storage allocated through `malloc` is uninitialized and has no type.

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

int main(void) {
    // Takeaway 13.1 #1 & #4: Size > 0, no casting of the void* return.
    double* largeVec = malloc(100 * sizeof(double));

    // Takeaway 13.1 #2 & #5: Check for null. Storage has no type until written to.
    if (largeVec != NULL) {
        largeVec[0] = 3.14; // Now the effective type is double
    }

    const char* source = "Hello, World!";
    // Takeaway 13.1 #3: Prefer strndup to limit max read length.
    char* copy = strndup(source, 5);
    if (copy) {
        printf("%s\n", copy); // Prints "Hello"
        free(copy);
    }

    free(largeVec);
    return 0;
}
```

## 13.1.1. A complete example with varying array size

- Demonstrates managing a dynamically allocated structure (like a circular buffer).
- The `[[nodiscard]]` attribute ensures the returned pointer replaces the old one.

**Takeaway:**

- **Takeaway 13.1.1 #1:** `malloc` indicates failure by returning a null pointer value.

```c
#include <stdlib.h>

typedef struct circular circular;
struct circular {
    size_t cap;
    double* tab;
};

circular* circular_init(circular* c, size_t cap) {
    if (c) {
        if (cap) {
            *c = (circular){
                .cap = cap,
                .tab = malloc(sizeof(double[cap])),
            };
            // Takeaway 13.1.1 #1: Checking if malloc failed and returned null
            if (!c->tab) c->cap = 0;
        } else {
            *c = (circular){ };
        }
    }
    return c;
}

int main(void) {
    circular c;
    circular_init(&c, 10);
    free(c.tab);
    return 0;
}
```

## 13.1.2. Ensuring consistency of dynamic allocations

- Every allocated block must be paired with exactly one deallocation to avoid memory leaks.

**Takeaways:**

- **Takeaway 13.1.2 #1:** For every allocation, there must be a `free`.
- **Takeaway 13.1.2 #2:** For every `free`, there must be a `malloc`, `calloc`, `aligned_alloc`, or `realloc`.
- **Takeaway 13.1.2 #3:** Only call `free` with pointers as they are returned by `malloc`, `calloc`, `aligned_alloc`, or `realloc`.

```c
#include <stdlib.h>

int main(void) {
    // Takeaway 13.1.2 #2: Using malloc to get a valid pointer
    int* data = malloc(sizeof(int));

    if (data) {
        *data = 42;
    }

    // Takeaway 13.1.2 #1 & #3: Using free exactly once, on the returned pointer
    free(data);

    return 0;
}
```

## 13.1.3. Flexible array members

- A flexible array member (FLA) must be the last member of a struct, defined as an incomplete array `[]`.

**Takeaways:**

- **Takeaway 13.1.3 #1:** A structure object with a flexible array member must have enough storage to access the structure as a whole.
- **Takeaway 13.1.3 #2:** Consistency between a length member and a flexible array member must be maintained manually.

```c
#include <stdlib.h>
#include <stdint.h>
#include <stddef.h>

typedef struct ua32 ua32;
struct ua32 {
    size_t length;
    uint32_t data[]; // Flexible array member
};

int main(void) {
    size_t len = 32;
    // Takeaway 13.1.3 #1: Allocate enough for struct offset PLUS the array
    size_t size = offsetof(ua32, data) + sizeof(uint32_t[len]);
    if (size < sizeof(ua32)) size = sizeof(ua32);

    ua32* ap = calloc(size, 1);
    if (ap) {
        // Takeaway 13.1.3 #2: Manually maintaining consistency
        ap->length = len;
        free(ap);
    }
    return 0;
}
```

## 13.2. Storage duration, lifetime, and visibility

- Visibility (scope) and lifetime are distinct. Four storage durations exist: static, automatic, allocated, and thread.

**Takeaways:**

- **Takeaway 13.2 #1:** Identifiers only have visibility inside their scope, starting at their declaration.
- **Takeaway 13.2 #2:** The visibility of an identifier can be shadowed by an identifier of the same name in a subordinate scope.
- **Takeaway 13.2 #3:** Every definition of a variable creates a new, distinct object.
- **Takeaway 13.2 #4:** Read-only object literals may overlap.
- **Takeaway 13.2 #5:** Objects have a lifetime outside of which they can’t be accessed.
- **Takeaway 13.2 #6:** A program execution that refers to an object outside of its lifetime fails.
- **Takeaway 13.2 #7:** A compound literal has the same lifetime as a variable that would be declared with the same storage class within the same context.

```c
#include <stdio.h>

// Static storage duration
unsigned i = 1;

int main(void) {
    // Takeaway 13.2 #1 & #3: New distinct object created inside this scope
    unsigned i = 2;
    if (i) {
        // Takeaway 13.2 #2: Shadows the block-scope 'i', refers to global 'i'
        extern unsigned i;
        printf("%u\n", i); // Prints 1
    }

    // Takeaway 13.2 #7: Compound literal lifetime follows block scope here
    int* p = &(int){ 5 };
    printf("%d\n", *p);

    // Takeaway 13.2 #5 & #6: Avoid accessing out-of-lifetime pointers
    int* bad_ptr;
    {
        int temp = 42;
        bad_ptr = &temp;
    }
    // printf("%d", *bad_ptr); // FAILURE: temp's lifetime has ended

    return 0;
}
```

## 13.2.1. Static storage duration

- Objects defined in file scope and not declared with `thread_local`. Variables and compound literals can have that property.
- Variables and compound literals defined inside a block and have the storage class specifier `static` and no additional `thread_local`.
- String literals, which are arrays of `char` or a wide character type and always have static storage duration.

Such objects have a lifetime that spans the entire program execution. Because they are considered alive before any application code is executed, they can only be initialized with expressions that are known at compile time or can be resolved by the system's
process startup procedure.

**Takeaway:**

- **Takeaway 13.2.1 #1:** Objects with static storage duration are always initialized.

```c
#include <stdio.h>

double A = 37; // Explicitly initialized
double* p = &(static double){ 1.0 }; // Static compound literal

int main(void) {
    // Takeaway 13.2.1 #1: B is implicitly initialized to 0.0
    static double B;
    printf("B is %f\n", B);
    return 0;
}
```

## 13.2.2. Automatic storage duration

- Local variables (without `static`). Can be heavily optimized by compilers if aliasing is restricted.

**Takeaways:**

- **Takeaway 13.2.2 #1:** Unless automatic objects are VLA or temporary objects, they have a lifetime corresponding to the execution of their block of definition.
- **Takeaway 13.2.2 #2:** Each recursive call creates a new local instance of an automatic object.
- **Takeaway 13.2.2 #3:** The `&` operator is not allowed for objects declared with `register`.
- **Takeaway 13.2.2 #4:** Objects declared with `register` can't alias.
- **Takeaway 13.2.2 #5:** Declare local variables that are not arrays in performance-critical code as `register`.
- **Takeaway 13.2.2 #6:** Arrays with storage class `register` are useless.
- **Takeaway 13.2.2 #7:** Objects of temporary lifetime are read-only.
- **Takeaway 13.2.2 #8:** Temporary lifetime ends at the end of the enclosing full expression.

```c
#include <stdio.h>

struct demo { unsigned ory[1]; };
struct demo mem(void) {
    return (struct demo){ .ory = {42} };
}

void recurse(int depth) {
    // Takeaway 13.2.2 #1 & #2: local_var has automatic lifetime, new instance per call
    int local_var = depth;
    if (depth > 0) recurse(depth - 1);
}

int main(void) {
    // Takeaway 13.2.2 #3, #4, #5: Use register for performance-critical scalars. Cannot take address.
    register int fast_counter = 0;
    fast_counter++;

    // Takeaway 13.2.2 #7 & #8: Temporary lifetime object array access. Read-only. Lifetime ends after printf.
    printf("mem().ory[0] is %u\n", mem().ory[0]);

    return 0;
}
```

## 13.3. Digression: using objects before their definition

- Explains edge cases where a block-scope object exists conceptually before its declaration is executed due to how the scope is entered.

**Takeaways:**

- **Takeaway 13.3 #1:** For an object that is not a VLA, lifetime starts when the scope of the definition is entered, and it ends when that scope is left.
- **Takeaway 13.3 #2:** Initializers of automatic variables and compound literals are evaluated each time the definition is met.
- **Takeaway 13.3 #3:** For a VLA, lifetime starts when the definition is encountered and ends when the visibility scope is left.

```c
#include <stdio.h>

int main(void) {
    int j = 0;
    // Takeaway 13.3 #1: 'x' lifetime starts at the beginning of the block, even though defined below.
    goto SKIP_INIT;

    int x = 42;

SKIP_INIT:
    // This is valid memory, but uninitialized because we skipped the initializer line.
    x = 10;

    // Takeaway 13.3 #2: If we looped back, the initializer would run each time.
    printf("x is %d\n", x);

    return 0;
}
```

## 13.4. Initialization

- Summarizes initialization state based on storage duration and promotes consistent patterns.

**Takeaways:**

- **Takeaway 13.4 #1:** Objects of static or thread-storage duration are initialized by default.
- **Takeaway 13.4 #2:** Objects of automatic or allocated storage duration must be initialized explicitly.
- **Takeaway 13.4 #3:** Systematically provide an initialization function for each of your data types.

```c
#include <stdlib.h>

typedef struct rat rat;
struct rat { int num, den; };

// Takeaway 13.4 #3: Systematically provide an init function
rat* rat_init(rat* p, int num, int den) {
    if (p) {
        p->num = num;
        p->den = den;
    }
    return p;
}

static rat global_rat; // Takeaway 13.4 #1: Initialized by default (0, 0)

int main(void) {
    rat local_rat; // Takeaway 13.4 #2: Uninitialized! Must explicitly initialize.
    rat_init(&local_rat, 1, 2);

    rat* alloc_rat = malloc(sizeof(rat)); // Also uninitialized
    rat_init(alloc_rat, 3, 4);

    free(alloc_rat);
    return 0;
}
```

## 13.5. Digression: A machine model

- Provides a glimpse into how C programs map to actual hardware execution (like x86_64 architecture).
- Functions utilize a reserved memory area called "the stack" to hold local variables.
- Stack layout uses registers (like `%rbp`) to manage addresses via negative offsets (e.g., `-36(%rbp)`).
- Automatic variables spring to life simply by the compiler adjusting the stack pointer when a function is entered.
- No explicit takeaways are numbered in this section, but the core concept is understanding that C's abstract machine memory model maps efficiently onto physical Von Neumann architectures (Registers, Stack, RAM).

```c
int fgoto(int n) {
    // Concept: The machine creates 'n' and 'x' on the call stack.
    // They are accessed via offsets from the base pointer register.
    int x = n * 2;
    return x;
}
```
