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

## 13. Storage (Introduction)

**What it is:** Storage in C deals with how and where objects are kept in memory during the execution of a program. This chapter focuses on variables and compound literals, exploring their **lifetime** (how long they exist), **storage duration** (the rules governing their creation and destruction), and **identifier visibility** (where in the code they can be accessed). It also introduces **dynamic allocation**, a mechanism to request storage at runtime.

**Why it matters / how it works:** Programs often handle varying amounts of data (like user inputs or large files) where the required memory size is unknown at compile time. Dynamic allocation solves this by creating storage instances on the fly. Understanding storage duration and object lifetime is essential to prevent accessing invalid memory, which can lead to undefined behavior or crashes.

**Key details:**

- Objects have a lifetime dictated by their storage duration.
- Different instances of a variable can exist simultaneously, such as during recursive function calls.
- Storage instances created dynamically initially have no interpretation as objects; they only acquire a type once a value is stored in them.

**Pitfalls:**

- Confusing identifier visibility (where a name is valid) with object lifetime (when the memory is valid).

**Connections:**

- Builds on the concepts of variables and compound literals introduced in earlier chapters.
- Relates closely to C's memory model and pointers, as dynamically allocated memory is accessed exclusively via pointers.

**Recap:**

- Storage duration defines when an object is created and destroyed.
- Dynamic allocation reclaims storage on the fly.
- Variables and compound literals have specific lifetimes depending on their syntactical context.

---

## 13.1. malloc and friends

### Dynamic Memory Allocation Functions

**What it is:** The `<stdlib.h>` header provides functions to manually manage memory at runtime: `malloc`, `free`, `calloc`, `realloc`, and `aligned_alloc`. `malloc` creates a new storage instance, `free` annihilates it, `calloc` allocates and zeroes all bits, `realloc` resizes existing storage, and `aligned_alloc` enforces strict alignment boundaries. Additionally, C23 incorporates `strdup` and `strndup` from POSIX into `<string.h>` to simplify duplicating strings.

**Why it matters / how it works:** Arrays and standard variables require sizes known largely at compile time (except for Variable-Length Arrays, which live on the stack). For large or irregularly sized data, we must allocate storage dynamically from the "heap". These functions operate using `void*`, meaning they handle raw byte arrays without a predefined type.

**Key details:**

- `malloc(size_t size)` takes the number of bytes to allocate.
- `calloc(size_t nmemb, size_t size)` allocates space for an array of `nmemb` elements, zeroing the memory.
- These functions return `void*`, which automatically converts to any other object pointer type.

> Takeaway 13.1 #1: Only use the allocation functions with a size strictly greater than zero.

_Explanation:_ Requesting zero bytes can yield implementation-defined behavior. It's safer to always ensure the requested size is at least 1 byte.

> Takeaway 13.1 #2: Failed allocations result in a null pointer.

_Explanation:_ If the system exhausts available memory, the allocation functions return a null pointer (`nullptr` in C23).

> Takeaway 13.1 #3: Prefer the use of strndup over strdup.

_Explanation:_ `strdup` requires a strictly 0-terminated string. `strndup` adds a length parameter, preventing out-of-bounds reads if the source string lacks a null terminator.

> Takeaway 13.1 #4: Don’t cast the return of malloc and friends.

_Explanation:_ In C, `void*` converts implicitly to any other object pointer. Casting is redundant and can hide errors (like forgetting to include `<stdlib.h>`).

> Takeaway 13.1 #5: Storage allocated through malloc is uninitialized and has no type.

_Explanation:_ The memory returned by `malloc` contains garbage values. It only acquires an effective type once you write data into it.

**Code Demonstration:**

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    // Correct usage of malloc without a cast.
    // Allocates an array of 10 doubles.
    size_t size = 10;
    double* my_array = malloc(sizeof(double) * size);

    // Always check for failed allocation (Takeaway 13.1 #2)
    if (!my_array) {
        return EXIT_FAILURE;
    }

    // Initialize the storage so it acquires a type (Takeaway 13.1 #5)
    for (size_t i = 0; i < size; ++i) {
        my_array[i] = 0.0;
    }

    // Prefer strndup (Takeaway 13.1 #3)
    char const* original = "Hello World";
    char* copy = strndup(original, 5); // Only copies "Hello"

    printf("Copied string: %s\n", copy);

    // Free all allocated memory
    free(my_array);
    free(copy);
    return EXIT_SUCCESS;
}

```

_Expected behavior/output:_ The program safely allocates memory, checks for success, initializes it, copies a string safely, prints "Copied string: Hello", and frees the resources.
_What goes wrong if..._ If you skip the `!my_array` check and `malloc` fails, assigning `my_array[i] = 0.0` will attempt to dereference a null pointer, causing a crash.

**Recap:**

- Use `<stdlib.h>` tools for dynamic memory.
- Always test returned pointers against null.
- Memory from `malloc` is uninitialized raw bytes.

### 13.1.1. A complete example with varying array size

**What it is:** The book illustrates dynamic arrays using a `circular` buffer structure for `double` values. It uses dynamic allocation to assign memory for both the control structure and the underlying data array.

**Why it matters / how it works:** Building scalable data structures requires coupling dynamic arrays with metadata (like capacity and current length). The `realloc` function is heavily utilized to resize the internal array when capacity limits are hit, necessitating careful manipulation of the memory chunks (e.g., using `memcpy` or `memmove`).

> Takeaway 13.1.1 #1: malloc indicates failure by returning a null pointer value.

_Explanation:_ As highlighted in the code examples for the circular buffer, any function allocating memory must prepare for the eventuality that memory is exhausted.

### 13.1.2. Ensuring consistency of dynamic allocations

**What it is:** Consistency means managing memory such that every byte requested from the system is eventually returned to the system when no longer needed.

**Why it matters / how it works:** Failing to return memory creates a **memory leak**, which depletes system resources and degrades performance.

> Takeaway 13.1.2 #1: For every allocation, there must be a free.

_Explanation:_ Simple counting of allocation calls and free calls should generally yield the same number.

> Takeaway 13.1.2 #2: For every free, there must be a malloc, calloc, aligned_alloc, or realloc.

_Explanation:_ You cannot `free` variables declared on the stack or statically; `free` is exclusively for pointers yielded by dynamic allocation functions.

> Takeaway 13.1.2 #3: Only call free with pointers as they are returned by malloc, calloc, aligned_alloc, or realloc.

_Explanation:_ Modifying the pointer value (e.g., `ptr++`) before passing it to `free` will crash the program because the memory allocator metadata will not align.

**Code Demonstration:**

```c
#include <stdlib.h>

int main(void) {
    int* ptr = malloc(sizeof(int) * 5);
    if (!ptr) return 1;

    // Do work...

    // CORRECT: Freeing the exact pointer returned by malloc
    free(ptr);

    return 0;
}

```

_What goes wrong if..._ If you do `free(ptr + 1);`, the allocator will fail to find the hidden accounting metadata it stored just before `ptr`, resulting in memory corruption or an immediate crash.

### 13.1.3. Flexible array members

**What it is:** A Flexible Array Member (FLA) allows the last member of a `struct` to be an array of unspecified size (e.g., `uint32_t data[];`).

**Why it matters / how it works:** It couples array metadata (like length) intimately with the array data in a single continuous memory block, improving cache locality and avoiding a secondary pointer dereference.

> Takeaway 13.1.3 #1: A structure object with a flexible array member must have enough storage to access the structure as a whole.

_Explanation:_ You must allocate enough memory to hold both the base struct members (and potential padding) PLUS the desired length of the flexible array.

> Takeaway 13.1.3 #2: Consistency between a length member and a flexible array member must be maintained manually.

_Explanation:_ C does not automatically update your `length` member when you allocate the struct. The programmer must manually assign `obj->length = size;`.

**Code Demonstration:**

```c
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>

typedef struct {
    size_t length;
    int data[]; // Flexible array member
} IntArray;

int main(void) {
    size_t elements = 10;

    // Calculate total required size.
    // offsetof helps find where the array actually starts.
    size_t total_size = offsetof(IntArray, data) + sizeof(int) * elements;

    // Ensure size is at least the size of the base struct
    if (total_size < sizeof(IntArray)) {
        total_size = sizeof(IntArray);
    }

    IntArray* arr = malloc(total_size);
    if (arr) {
        // Manually maintain consistency (Takeaway 13.1.3 #2)
        arr->length = elements;

        arr->data[0] = 42;
        printf("Element 0: %d\n", arr->data[0]);
        free(arr);
    }
    return 0;
}

```

_Expected behavior/output:_ Allocates exactly enough memory for the struct plus 10 integers, sets the length, and prints 42.

**Recap:**

- Match every allocation with exactly one `free`.
- Flexible array members provide an inline dynamic array at the end of a struct.
- You must manually calculate allocation sizes for flexible arrays and maintain their length metadata.

---

## 13.2. Storage duration, lifetime, and visibility

**What it is:** This section differentiates between an identifier's visibility (where a name can be used lexically) and an object's lifetime (when the memory state is actively valid). C categorizes storage into four durations: **static**, **automatic**, **allocated**, and **thread**.

**Why it matters / how it works:** Using a pointer to access an object whose lifetime has ended yields undefined behavior. Identifiers can also be hidden (shadowed) by identically named identifiers in inner scopes. The `extern` and `static` keywords manipulate **linkage**, which allows the system linker to connect identifiers across different translation units.

> Takeaway 13.2 #1: Identifiers only have visibility inside their scope, starting at their declaration.

_Explanation:_ You cannot use a variable name before the line it is declared on, nor outside the curly braces `{}` that enclose it.

> Takeaway 13.2 #2: The visibility of an identifier can be shadowed by an identifier of the same name in a subordinate scope.

_Explanation:_ If you declare an `int x` inside a block, and then another `int x` inside a nested `if` block, the inner `x` hides the outer `x` for the duration of the `if` block.

> Takeaway 13.2 #3: Every definition of a variable creates a new, distinct object.

_Explanation:_ Shadowed variables do not overwrite the original variable; they exist as entirely separate objects in memory with their own distinct addresses.

> Takeaway 13.2 #4: Read-only object literals may overlap.

_Explanation:_ For efficiency, compilers are permitted to store identical string literals or `const`-qualified compound literals at the exact same memory address.

> Takeaway 13.2 #5: Objects have a lifetime outside of which they can’t be accessed.

_Explanation:_ The memory for an object is only valid within specific temporal boundaries determined by its storage duration.

> Takeaway 13.2 #6: A program execution that refers to an object outside of its lifetime fails.

_Explanation:_ Returning a pointer to a local variable from a function and then dereferencing it later will access dead memory, breaking the abstract state machine.

> Takeaway 13.2 #7: A compound literal has the same lifetime as a variable that would be declared with the same storage class within the same context.

_Explanation:_ A compound literal created inside a block without any storage class acts just like a local automatic variable. Once the block ends, the compound literal is destroyed.

### 13.2.1. Static storage duration

**What it is:** Objects with static storage duration live for the entire execution of the program. This includes global variables, static local variables, and string literals.

**Why it matters / how it works:** Because they exist before application code runs, their initial values must be known at compile time.

> Takeaway 13.2.1 #1: Objects with static storage duration are always initialized.

_Explanation:_ If you do not provide an explicit initializer, the compiler implicitly initializes them to zero (or `nullptr`).

### 13.2.2. Automatic storage duration

**What it is:** Automatic objects are dynamically managed by the execution environment (usually via a call stack).

**Why it matters / how it works:** They spring into existence when their enclosing block executes and die when the block exits. The `register` keyword is a legacy specifier that hints to the compiler that the variable will be heavily used and its address will never be needed.

> Takeaway 13.2.2 #1: Unless automatic objects are VLA or temporary objects, they have a lifetime corresponding to the execution of their block of definition.

_Explanation:_ The object is conceptually alive from the moment the block is entered until the block is left.

> Takeaway 13.2.2 #2: Each recursive call creates a new local instance of an automatic object.

_Explanation:_ Recursion relies on this property. Every active level of recursion holds its own independent copy of local variables.

> Takeaway 13.2.2 #3: The & operator is not allowed for objects declared with register.

_Explanation:_ Because `register` objects might literally live in CPU registers (which do not have memory addresses), you cannot extract a pointer to them.

> Takeaway 13.2.2 #4: Objects declared with register can’t alias.

_Explanation:_ Since you cannot take their address, no pointer can ever point to them. The compiler can optimize aggressively knowing no external code is modifying them behind its back.

> Takeaway 13.2.2 #5: Declare local variables that are not arrays in performance-critical code as register.

_Explanation:_ This acts as a strong optimization guarantee for the compiler regarding aliasing.

> Takeaway 13.2.2 #6: Arrays with storage class register are useless.

_Explanation:_ Because you cannot use the `&` operator, and arrays inherently decay into pointers (which is an address operation), you basically cannot use a `register` array in C.

> Takeaway 13.2.2 #7: Objects of temporary lifetime are read-only.

_Explanation:_ When a function returns a struct containing an array, the compiler creates a temporary object so you can access array members (e.g., `func().array[0]`). You cannot modify this temporary object.

> Takeaway 13.2.2 #8: Temporary lifetime ends at the end of the enclosing full expression.

_Explanation:_ The temporary return struct is immediately destroyed as soon as the statement (ending with `;`) finishes.

**Code Demonstration:**

```c
#include <stdio.h>

// Static storage duration - initialized to 0 automatically
static int global_counter;

void recursive_func(int count) {
    // Automatic storage duration - new instance per call
    int local_val = count;

    // Performance hint: no pointer can point to this
    register int fast_val = count * 2;

    if (count > 0) {
        recursive_func(count - 1);
    }
    printf("Local: %d, Fast: %d, Global: %d\n", local_val, fast_val, global_counter);
}

int main(void) {
    recursive_func(2);
    // int* p = &fast_val; // ERROR: Cannot take address of register variable
    return 0;
}

```

_Expected behavior/output:_ Prints recursive outputs showing independent `local_val` instances while `global_counter` remains 0.

**Recap:**

- Visibility is lexical; lifetime is temporal.
- Static objects live forever and zero-initialize automatically.
- Automatic objects exist per-block and per-call.

---

## 13.3. Digression: using objects before their definition

**What it is:** This section explores the exact moment automatic storage lifetime begins. For non-VLAs, lifetime starts upon _entering_ the block, not at the line of declaration.

**Why it matters / how it works:** By using `goto`, it's possible to skip over a variable's declaration but still access its memory (though its value will be uninitialized).

> Takeaway 13.3 #1: For an object that is not a VLA, lifetime starts when the scope of the definition is entered, and it ends when that scope is left.

_Explanation:_ The memory for standard variables is reserved the moment the `{` is crossed.

> Takeaway 13.3 #2: Initializers of automatic variables and compound literals are evaluated each time the definition is met.

_Explanation:_ If you jump back into a loop or use `goto` to re-execute a declaration line, the variable is re-initialized to the specified value.

> Takeaway 13.3 #3: For a VLA, lifetime starts when the definition is encountered and ends when the visibility scope is left.

_Explanation:_ Variable Length Arrays (VLAs) require knowing a runtime size, so their memory cannot be reserved at the start of the block. You cannot `goto` forward over a VLA declaration.

**Code Demonstration:**

```c
#include <stdio.h>

int main(void) {
    int skip = 1;

    if (skip) {
        goto SKIP_INIT;
    }

    int val = 42; // Lexical declaration and initialization

SKIP_INIT:
    // Memory for 'val' exists here (Takeaway 13.3 #1), but it is uninitialized!
    // printf("%d", val); // DANGEROUS: val has garbage data.

    val = 10; // Perfectly valid to use the object now.
    printf("Val: %d\n", val);
    return 0;
}

```

_Expected behavior/output:_ Prints "Val: 10". The memory for `val` existed before the initialization line was executed.

**Recap:**

- Non-VLA memory is allocated at block entry.
- Initialization code only runs when lexically evaluated.
- VLA lifetime strictly begins at the declaration line.

---

## 13.4. Initialization

**What it is:** Initialization provides an object with its first valid state.

**Why it matters / how it works:** Using uninitialized automatic or allocated objects results in undefined behavior. Structuring code to initialize values robustly prevents erratic crashes.

> Takeaway 13.4 #1: Objects of static or thread-storage duration are initialized by default.

_Explanation:_ They are reliably set to zero before `main()` executes.

> Takeaway 13.4 #2: Objects of automatic or allocated storage duration must be initialized explicitly.

_Explanation:_ Variables on the stack (automatic) or heap (allocated) contain whatever garbage bits were previously left in RAM.

> Takeaway 13.4 #3: Systematically provide an initialization function for each of your data types.

_Explanation:_ To prevent forgetting struct fields, define a generic `_init` function (like `rat_init(rat* p, ...)`) that sets up memory correctly.

**Code Demonstration:**

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    int id;
    double value;
} Item;

// Systematically provide an initialization function (Takeaway 13.4 #3)
Item* item_init(Item* p, int id, double value) {
    if (p) {
        p->id = id;
        p->value = value;
    }
    return p;
}

int main(void) {
    // Explicit initialization of allocated storage (Takeaway 13.4 #2)
    Item* i1 = item_init(malloc(sizeof(Item)), 1, 99.9);

    if (i1) {
        printf("Item %d: %f\n", i1->id, i1->value);
        free(i1);
    }
    return 0;
}

```

_Expected behavior/output:_ Prints "Item 1: 99.900000".
_What goes wrong if..._ If we omit `item_init` and just print `i1->id`, the output will be completely random garbage, changing every time the program runs.

**Recap:**

- Static data zeroes itself.
- Automatic and allocated data must be manually set.
- Use initialization functions for complex data types.

---

## 13.5. Digression: A machine model

**What it is:** This section maps C's abstract rules of automatic storage to concrete computer hardware using assembly language.

**Why it matters / how it works:** C compilers typically use a **stack** to manage automatic objects. A base pointer (like `%rbp` on x86_64) marks the start of the current function's memory frame. Local variables are placed at negative offsets from this register. This physical stack model perfectly mirrors C's rule that automatic storage enters life at the block start (the function prologue allocates the stack frame) and dies at block end (the epilogue unwinds the stack).

**Key details:**

- Stack operations align with block scopes.
- Uninitialized automatic variables simply map to stack memory that is not explicitly overwritten by assembly instructions, validating why they hold garbage data.
- Optimizing compilers might bypass the stack entirely for small variables, storing them purely in CPU registers.

**Recap:**

- C's automatic storage translates cleanly to hardware stacks.
- The hardware model explains the behavior of uninitialized variables and recursion.

---

## Summary

Chapter 13 exhaustively details how the C memory model handles object storage over time. It introduces dynamic memory allocation via `malloc` and its peers, emphasizing the necessity of rigorously pairing allocations with `free` calls to avoid memory leaks. The chapter clearly delineates between identifier visibility (a compile-time concept) and object lifetime (a runtime concept), exposing how static, automatic, and allocated objects exist inside the machine. Finally, it enforces strict practices around data initialization, particularly for dynamic and automatic storage which require manual intervention to prevent undefined behavior from garbage data.

---

## Self-Check Questions

1. **Question:** Why must you never pass a size of 0 to `malloc`?

- **Answer:** A size of zero can result in implementation-defined behavior.

2. **Question:** What does `malloc` return if the system is out of memory?

- **Answer:** It returns a null pointer.

3. **Question:** What happens to the memory returned by `malloc` before you assign to it?

- **Answer:** It remains uninitialized garbage and has no effective type.

4. **Question:** If you allocate memory with `calloc`, can you deallocate it using the `free` function?

- **Answer:** Yes, `free` must be used for pointers returned by `malloc`, `calloc`, `aligned_alloc`, and `realloc`.

5. **Question:** What is the difference between identifier visibility and object lifetime?

- **Answer:** Visibility dictates where in the source code a name can be referenced, while lifetime dictates the temporal duration during execution when the memory object exists.

6. **Question:** If you declare a variable as `static` inside a function block, when is it initialized?

- **Answer:** It is initialized before the program execution starts (static storage duration).

7. **Question:** Why can't you use the `&` operator on a variable declared with `register`?

- **Answer:** Because the variable might be stored in a CPU register, which does not have a memory address.

8. **Question:** When does the lifetime of a non-VLA automatic variable begin?

- **Answer:** It begins as soon as execution enters the block where it is defined, even before the definition line is reached.
