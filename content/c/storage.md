+++
title = "Modern C: Storage"
description = ""
date = 2026-09-28

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


### 4. Lifetime vs. Definition Encounter & VLAs (Section 13.3)

* **Takeaway 13.3 #1:** *For an object that is not a VLA, lifetime starts when the scope of the definition is entered, and it ends when that scope is left*.
* **Takeaway 13.3 #2:** *Initializers of automatic variables and compound literals are evaluated each time the definition is met*.
* **Takeaway 13.3 #3 (VLAs):** *For a VLA, lifetime starts when the definition is encountered and ends when the visibility scope is left*. Because Variable-Length Array sizes depend on runtime calculations, VLA memory cannot be allocated upon scope entry.


### 5. Initialization Rules & Systematic Construction APIs (Section 13.4)

* **Static/Thread Objects:** **Takeaway 13.4 #1:** Automatically default-initialized to all-bits zero (`0` or `nullptr`).
* **Automatic/Allocated Objects:** **Takeaway 13.4 #2:** Must be **initialized explicitly**. Memory returned by `malloc` contains uninitialized garbage.
* **Systematic Type APIs:** **Takeaway 13.4 #3:** *Systematically provide an initialization function for each of your data types*.
  * Standard naming convention:
    * `toto* toto_init(toto* p, ...)`: Initializes existing memory; returns `nullptr` if `p` is null.
    * `toto* toto_new(...)`: Allocates memory via `malloc` and passes it to `toto_init`.
    * `void toto_destroy(toto* p)` & `void toto_delete(toto* p)`: Reclaims internal memory and frees the object.


### 6. Machine Model & Execution Stack (Section 13.5)

* **Stack Frame Allocation:** On real hardware architectures (such as x86_64), automatic objects are created en masse when entering a function by subtracting the total required frame size from the stack pointer register (`%rsp`).
* **Stack Memory is Garbage:** Because adjusting the stack pointer simply reserves space without writing bytes, uninitialized local variables contain residual stack garbage.
* **As-If Compiler Optimizations:** Under full compiler optimization, the "as-if" rule allows compilers to reorder, eliminate, or store automatic variables directly in hardware registers, bypassing physical stack writes entirely.
