+++
title = "Modern C: Storage"
description = "A guide to C23 storage duration and object lifetimes, dynamic allocation and deallocation, initialization APIs, VLAs, and the execution stack."
date = 2026-09-28T04:53:34Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 13: Storage** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book explores how data objects spring into existence, reside in memory, and are destroyed. This chapter covers dynamic memory allocation (`malloc` and friends), object lifetimes, visibility vs. scope, storage duration, initialization rules, and low-level stack machine models.

Here is a detailed breakdown of the key concepts, design rules, and takeaways from Chapter 13:


### 1. Dynamic Allocation: `malloc` and Friends (Section 13.1)

Dynamic storage allocation provides a mechanism to create and reclaim storage instances on the fly to handle unpredictable or varying collections of data (such as user input, irregular structures, or dynamic matrices).

* **Untyped Storage Instances:** Functions in `<stdlib.h>` operate with **`void*`**. Allocated memory represents raw storage bytes that initially have **no type and no value**; objects acquire an effective type and value only when data is assigned to them.
* **Core Allocation Functions:**
  * **`malloc(size)`**: Creates a storage instance of the requested byte size containing uninitialized garbage.
  * **`calloc(nmemb, size)`**: Allocates storage and automatically sets all bits to `0`. It is the preferred function for allocating arrays of integer or basic types.
  * **`realloc(ptr, size)`**: Resizes an existing allocation.
    * If `ptr` is `nullptr`, it acts like `malloc`.
    * Preserves existing content up to the smaller of the old and new sizes.
    * If memory must be moved, the old pointer is freed automatically and a new pointer is returned.
    * If reallocation fails, it returns `nullptr` while leaving the original memory intact.
  * **`aligned_alloc(alignment, size)`**: Allocates memory with a non-default alignment requirement.
  * **`strdup(s)` & `strndup(s, n)`**: Standardized in C23 from POSIX, these allocate memory and duplicate string contents.
* **Key Takeaways & Best Practices:**
  * **Takeaway 13.1 #1:** *Only use the allocation functions with a size strictly greater than zero*.
  * **Takeaway 13.1 #2 & 13.1.1 #1:** *Failed allocations result in a null pointer (`nullptr`)*. Always test allocation returns before dereferencing.
  * **Takeaway 13.1 #3:** *Prefer the use of `strndup` over `strdup`* because `strndup` bounds the scan length and guarantees `0`-termination.
  * **Takeaway 13.1 #4:** ***Don’t cast the return of `malloc` and friends***. Automatic conversion from `void*` to any target pointer type occurs cleanly without explicit casts.
  * **Idiom for Size Calculation:** Use `sizeof *p` (e.g., `double* vec = malloc(len * sizeof *vec);`) or `sizeof(double[len])` to keep allocations resilient if pointer types change.

#### Example

Dynamic storage allocation provides untyped, uninitialized raw storage bytes that acquire an effective type and value only when assigned.

* **Takeaway 13.1 #1:** Only call allocation functions with a size strictly greater than `0`.
* **Takeaway 13.1 #2 & #5 / 13.1.1 #1:** Failed allocations return `nullptr`. Allocated storage is uninitialized and has no type.
* **Takeaway 13.1 #3:** Prefer `strndup` over `strdup` because it bounds the string scan and guarantees `0`-termination.
* **Takeaway 13.1 #4:** **Don't cast the return of `malloc` and friends**. Conversion from `void*` is automatic in C.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void demo_dynamic_allocation(void) {
    size_t length = 10;

    // IDIOM 1 (Takeaway 13.1 #4): Do NOT cast malloc!
    // Uses 'sizeof *large_vec' so the allocation adapts automatically if large_vec's type changes.
    double* large_vec = malloc(length * sizeof *large_vec); //

    // IDIOM 2: Allocation using array type size
    // double* large_vec = malloc(sizeof(double[length])); //

    // Takeaway 13.1 #2 & 13.1.1 #1: Always check for nullptr on allocation failure
    if (!large_vec) { //
        perror("Allocation failed");
        return;
    }

    // Takeaway 13.1 #5: Storage is uninitialized garbage until explicitly written to
    for (size_t i = 0; i < length; ++i) {
        large_vec[i] = (double)i * 1.5; // Assignment sets effective type and value
    }

    // Takeaway 13.1 #3: Prefer strndup over strdup for bounded string duplicating
    char const* user_input = "Modern C23 Standard";
    char* string_copy = strndup(user_input, 10); // Clamps copy to 10 chars + '\0'

    if (string_copy) {
        printf("Bounded string copy: %s\n", string_copy);
        free(string_copy); // Clean up allocated string copy
    }

    free(large_vec); // Freeing vector allocation
}
```

### 2. Consistency & Deallocation Rules (Section 13.1.2)

Because the runtime heap is simple, proper memory management requires strict symmetry:

* **Takeaway 13.1.2 #1:** *For every allocation, there must be a `free`*. Failing to free memory leads to **memory leaks** and platform resource exhaustion.
* **Takeaway 13.1.2 #2:** *For every `free`, there must be a `malloc`, `calloc`, `aligned_alloc`, or `realloc`*.
* **Takeaway 13.1.2 #3:** *Only call `free` with pointers as they are returned by `malloc`, `calloc`, `aligned_alloc`, or `realloc`*.
  * Calling `free` on addresses of local variables, compound literals, or internal array offsets causes severe memory corruption and program crashes.
  * **`free(nullptr)` is safe** and performs no action.
* **Flexible Array Members (FAM) (Section 13.1.3):**
  * Structure types can declare an incomplete array as their last member (e.g., `struct ua32 { size_t length; uint32_t data[]; };`).
  * **Takeaway 13.1.3 #1:** A structure with a flexible array member must be allocated with enough space for both the base `struct` and the trailing array elements using `offsetof`.
  * **Takeaway 13.1.3 #2:** Consistency between the length field and the array must be maintained manually.

#### Example

Proper dynamic memory management requires exact balance between allocations and deallocations.

* **Takeaway 13.1.2 #1 & #2:** For every allocation, there must be a `free` (and vice-versa) to prevent memory leaks.
* **Takeaway 13.1.2 #3:** Only pass pointers to `free` that were returned by `malloc`, `calloc`, `aligned_alloc`, or `realloc` (or `nullptr`).
* **Takeaway 13.1.3 #1 & #2:** A structure with a **flexible array member (FAM)** must allocate space for both the base struct and trailing elements using `offsetof`; consistency must be maintained manually.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stddef.h> // Provides offsetof

// Structure containing a Flexible Array Member (uint32_t data[]) as its last field
typedef struct dynamic_array dynamic_array;
struct dynamic_array {
    size_t length;     // Metadata tracking element count
    uint32_t data[];   // Flexible Array Member (incomplete array)
};

dynamic_array* dynamic_array_create(size_t len) {
    // Takeaway 13.1.3 #1: Compute total bytes using offsetof + payload size
    size_t total_size = offsetof(dynamic_array, data) + sizeof(uint32_t[len]);
    if (total_size < sizeof(dynamic_array)) {
        total_size = sizeof(dynamic_array); // Ensure enough space for struct headers
    }

    // Use calloc to zero-initialize metadata and payload bytes
    dynamic_array* arr = calloc(total_size, 1);
    if (!arr) return nullptr; // Handle allocation failure

    // Takeaway 13.1.3 #2: Maintain length consistency manually
    arr->length = len;
    return arr;
}

void demo_flexible_array_members(void) {
    dynamic_array* arr = dynamic_array_create(5);
    if (!arr) return;

    for (size_t i = 0; i < arr->length; ++i) {
        arr->data[i] = (uint32_t)(i * 10);
        printf("arr->data[%zu] = %u\n", i, arr->data[i]);
    }

    // Takeaway 13.1.2 #1 & #3: Free allocated storage
    free(arr);

    // safe: free(nullptr) is a valid no-op
    free(nullptr);
}
```

### 3. Storage Duration, Lifetime, and Visibility (Section 13.2)

C strictly distinguishes between **identifier visibility** (lexical scope) and **object lifetime** (temporal accessibility).

* **Visibility & Shadowing:** An identifier is visible only within its scope. Inner scopes can **shadow** outer declarations of the same name. **Takeaway 13.2 #3:** *Every definition of a variable creates a new, distinct object*.
* **Read-Only Object Overlapping:** **Takeaway 13.2 #4:** *Read-only object literals may overlap*. Compilers are permitted to share memory locations for identical string literals or `const`-qualified compound literals.
* **The 4 Storage Durations:**
  1. **Static Storage Duration:** Lifetime spans the **entire program execution**. Applies to global variables, file-scope definitions, block-scope `static` variables, `constexpr` objects, and string literals.
     * **Takeaway 13.2.1 #1:** *Objects with static storage duration are always initialized* (implicitly defaulted to `0` or `nullptr` if no explicit initializer is provided).
  2. **Automatic Storage Duration:** Bound to the execution of a statement block or function. Applies to local variables, function parameters, and non-`static` compound literals.
     * **Takeaway 13.2.2 #1:** Lifetime corresponds to the execution of their block of definition.
     * **Takeaway 13.2.2 #2:** *Each recursive call creates a new local instance of an automatic object*.
     * **`register` Keyword:** **Takeaway 13.2.2 #3 & #4:** The address-of operator `&` is prohibited on `register` objects, guaranteeing they never alias and allowing storage in CPU registers. (Useless for arrays since arrays decay to pointers).
     * **Temporary Lifetime:** Returned objects containing array fields live until the end of the enclosing full expression.
  3. **Allocated Storage Duration:** Managed manually via `malloc` and `free`.
  4. **Thread Storage Duration (`thread_local` in C23):** Variable instances are tied strictly to the lifetime of an individual thread.

#### Example

C categorizes storage into four distinct durations: **static**, **automatic**, **allocated**, and **thread**.

* **Takeaway 13.2 #3:** Every definition creates a distinct object in memory.
* **Takeaway 13.2 #4:** Read-only object literals (e.g. string literals or `const` compound literals) may overlap in memory.
* **Takeaway 13.2.1 #1:** Objects with **static storage duration** are always initialized (defaulted to `0` or `nullptr`).
* **Takeaway 13.2.2 #3, #4 & #5:** Taking the address (`&`) of a **`register`** variable is illegal, guaranteeing it cannot alias and allowing register allocation. Arrays declared `register` are useless because arrays decay to pointers.
* **Takeaway 13.2.2 #7 & #8:** Objects with **temporary lifetime** (e.g. struct returns containing arrays) are read-only and expire at the end of the enclosing full expression.

```c
#include <stdio.h>
#include <stdbool.h>

// Static storage duration: Alive during entire program execution, defaulted to 0.0
static double global_static_val; // Automatically initialized to 0.0

struct array_holder { unsigned values; };
struct array_holder get_holder(void) { return (struct array_holder){ {10, 20} }; }

void demo_storage_durations(void) {
        // Takeaway 13.2 #4: Compilers may reuse identical read-only string literal locations
        char const* str1 = "Modern C";
        char const* str2 = "Modern C"; // str1 and str2 may point to identical memory addresses!

        // Takeaway 13.2.2 #5: Declare non-array local variables as 'register' in critical code
        register size_t fast_counter = 0; // Guarantees fast_counter will not alias

        // &fast_counter; // COMPILER ERROR! Cannot take address of register variable!

        // Takeaway 13.2.2 #7 & #8: Temporary Lifetime of member access on returned struct
        // The array returned by get_holder() exists ONLY for the duration of this printf expression!
        printf("Temporary array member: %u\n", get_holder().values);

        printf("Read-only literals match? %s\n", (str1 == str2) ? "Yes (Overlapped)" : "No");
}
```

### 4. Lifetime vs. Definition Encounter & VLAs (Section 13.3)

* **Takeaway 13.3 #1:** *For an object that is not a VLA, lifetime starts when the scope of the definition is entered, and it ends when that scope is left*.
* **Takeaway 13.3 #2:** *Initializers of automatic variables and compound literals are evaluated each time the definition is met*.
* **Takeaway 13.3 #3 (VLAs):** *For a VLA, lifetime starts when the definition is encountered and ends when the visibility scope is left*. Because Variable-Length Array sizes depend on runtime calculations, VLA memory cannot be allocated upon scope entry.

#### Example

* **Takeaway 13.3 #1 & #2:** For non-VLA automatic variables, lifetime begins upon **entering the block scope**, while initializers are evaluated each time the definition is encountered.
* **Takeaway 13.3 #3:** For **Variable-Length Arrays (VLAs)**, lifetime begins only when the **definition statement is encountered** during execution.

```c
#include <stdio.h>

void demo_lifetime_and_vlas(size_t n) {
    // Non-VLA automatic variable: Object lifetime exists as soon as block is entered
    size_t count = 0;

    AGAIN:
    if (count < 2) {
        // Compound literal initializer is evaluated EACH time line 18 is executed
        int* p = &((int){ (int)count + 10 });
        printf("Pass %zu: *p = %d\n", count, *p);

        ++count;
        goto AGAIN; // Jumping back re-evaluates initializer
    }

    // Takeaway 13.3 #3: VLA lifetime starts ONLY when this line is executed!
    // Memory cannot be pre-allocated upon block entry because 'n' is evaluated at runtime.
    double vla_buffer[n] = {};
    vla_buffer = 3.14;
    printf("VLA element 0: %g\n", vla_buffer);
}
```

### 5. Initialization Rules & Systematic Construction APIs (Section 13.4)

* **Static/Thread Objects:** **Takeaway 13.4 #1:** Automatically default-initialized to all-bits zero (`0` or `nullptr`).
* **Automatic/Allocated Objects:** **Takeaway 13.4 #2:** Must be **initialized explicitly**. Memory returned by `malloc` contains uninitialized garbage.
* **Systematic Type APIs:** **Takeaway 13.4 #3:** *Systematically provide an initialization function for each of your data types*.
  * Standard naming convention:
    * `toto* toto_init(toto* p, ...)`: Initializes existing memory; returns `nullptr` if `p` is null.
    * `toto* toto_new(...)`: Allocates memory via `malloc` and passes it to `toto_init`.
    * `void toto_destroy(toto* p)` & `void toto_delete(toto* p)`: Reclaims internal memory and frees the object.

#### Example

* **Takeaway 13.4 #1 & #2:** Static/thread objects are initialized by default. Automatic/allocated objects must be **initialized explicitly**.
* **Takeaway 13.4 #3:** Systematically provide initialization and destruction APIs for custom types:
  * `toto_init(toto* p, ...)`: Initializes existing memory; returns `p` (or `nullptr` on error).
  * `toto_new(...)`: Allocates memory via `malloc` and passes it to `toto_init`.
  * `toto_destroy(toto* p)` & `toto_delete(toto* p)`: Reclaims internal memory and frees the container instance.

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct rational rational;
struct rational {
    long long num;
    unsigned long long denom;
};

// 1. toto_init: Initializes existing storage passed by caller
rational* rational_init(rational* r, long long num, unsigned long long denom) {
    if (!r) return nullptr; // Guard against null pointer
    r->num = num;
    r->denom = (denom != 0) ? denom : 1;
    return r; //
}

// 2. toto_destroy: Reclaims inner resources of an existing object
void rational_destroy(rational* r) {
    if (r) {
        *r = (rational){}; // Clear internal fields
    }
}

// 3. toto_new: Combines malloc with rational_init
[[nodiscard]] inline rational* rational_new(long long num, unsigned long long denom) { //
    return rational_init(malloc(sizeof(rational)), num, denom); //
}

// 4. toto_delete: Combines rational_destroy with free
inline void rational_delete(rational* r) { //
    rational_destroy(r);
    free(r); //
}

void demo_systematic_api(void) {
    // Safe initialization idiom:
    // If malloc fails, rational_init returns nullptr, leaving my_rat cleanly in a null state.
    rational const* my_rat = rational_new(3, 4); //

    if (my_rat) {
        printf("Rational fraction: %lld/%llu\n", my_rat->num, my_rat->denom);
        rational_delete((rational*)my_rat); // Clean deletion
    }
}
```

### 6. Machine Model & Execution Stack (Section 13.5)

* **Stack Frame Allocation:** On real hardware architectures (such as x86_64), automatic objects are created en masse when entering a function by subtracting the total required frame size from the stack pointer register (`%rsp`).
* **Stack Memory is Garbage:** Because adjusting the stack pointer simply reserves space without writing bytes, uninitialized local variables contain residual stack garbage.
* **As-If Compiler Optimizations:** Under full compiler optimization, the "as-if" rule allows compilers to reorder, eliminate, or store automatic variables directly in hardware registers, bypassing physical stack writes entirely.

#### Example

At the low-level machine assembly level, automatic storage for block-scoped variables is created en masse upon entering a function by adjusting the stack pointer register (`%rsp`).

```c
#include <stdio.h>

void demo_machine_stack_model(void) {
    // MACHINE MODEL EXECUTION STACK:
    // 1. Function entry adjusts stack pointer register (%rsp) by total required frame bytes.
    // 2. Adjusting %rsp does NOT zero memory, so local uninitialized variables contain stack garbage.
    int uninit_stack_garbage; // Contains residual values leftover from prior function stack frames

    // 3. Compiler 'as-if' optimization: Under optimization (-O2), automatic variables that do not
    // have their addresses taken (&) are stored directly in CPU registers (e.g. %ebx, %ebp),
    // bypassing physical stack writes entirely.
    int register_cached_var = 42;

    printf("Stack variable address offset: %p (Value: %d)\n",
           (void*)&uninit_stack_garbage, register_cached_var);
}
```
