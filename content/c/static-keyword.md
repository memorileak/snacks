+++
title = "The `static` keyword in C23"
description = ""
date = 2026-09-30T02:24:24+00:00

[taxonomies]
tags = ["static"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## The `static` keyword

In C, the static keyword modifies either the lifetime (how long a variable stays in memory) or the visibility/linkage (what parts of the program can see it) of a variable or function. [1, 2] 
Its behavior changes completely based on where you place it in your code. [1] 

### 1. Inside a Function (Static Local Variables)
When used inside a function, static changes the variable's storage duration from automatic (stack-allocated) to static storage duration (stored in the data segment). [2, 3] 

* **What it does**: The variable is initialized only once. It retains its value across multiple function calls instead of being destroyed when the function exits.
* **Scope**: It is still private to that function; other functions cannot access it directly. [1, 4, 5, 6, 7] 

```c
#include <stdio.h>

void counter() {
    static int count = 0; // Initialized ONLY on the first call
    count++;
    printf("%d ", count);
}

int main() {
    counter(); // Prints 1
    counter(); // Prints 2
    counter(); // Prints 3
}
```

### 2. Outside a Function / Global Scope (Internal Linkage)
By default, global variables and functions in C have extern (external) linkage, meaning they can be accessed from other source files (.c files) in the same project. [8, 9] 

* **What it does**: Placing static before a global variable or function limits its visibility strictly to the current translation unit (the specific .c file it is compiled in). [1, 9] 
* **Why use it**: It acts like a private modifier in object-oriented languages. It hides internal implementation details and prevents naming collisions if another file happens to use the same variable or function name. [1, 8, 10] 

```c
// file_helper.c
static int internal_state = 0; // Other .c files cannot see or modify this

static void do_background_math() { 
    // Private helper function; invisible to the linker outside this file
}

void public_api_function() {
    do_background_math(); // Allowed here
}
```

### 3. Inside Array Declarations in Function Parameters (Modern C / C99+)
In modern C (C99 and newer), static has a lesser-known third use inside function parameter array brackets.

* What it does: It acts as a hint to the compiler optimizer, guaranteeing that the array argument passed into the function will always contain at least the specified number of elements and will not be NULL.
* Why use it: It allows the compiler to generate highly optimized assembly code (like using SIMD vector instructions) because it doesn't have to worry about null checks or out-of-bounds safety checks for that boundary.

```c
// The compiler assumes 'arr' is never NULL and has AT LEAST 4 elements
void process_coordinates(int arr[static 4]) {
    for (int i = 0; i < 4; i++) {
        printf("%d\n", arr[i]);
    }
}
```

### Summary Table

| Context | What it Affects | Behavior |
|---|---|---|
| Inside a function | Lifetime | Value persists across function calls; allocated once. |
| Global variable / Function | Visibility (Linkage) | Restricts access to the current .c file only (Internal Linkage). |
| Function argument array | Optimization / Safety | Guarantees the array is non-null and meets a minimum size requirement. |

[1] [https://www.youtube.com](https://www.youtube.com/watch?v=3E-r4GfvWOI&t=164)
[2] [https://www.youtube.com](https://www.youtube.com/watch?v=g4Dn2cwSrC4)
[3] [https://en.wikipedia.org](https://en.wikipedia.org/wiki/Static_%28keyword%29)
[4] [https://www.reddit.com](https://www.reddit.com/r/explainlikeimfive/comments/si1ooi/eli5_what_exactly_is_a_static_variable_in/)
[5] [https://medium.com](https://medium.com/@pratikkatkar13/mastering-static-keyword-in-c-from-basics-to-advanced-4bc9107eb80d)
[6] [https://www.scaler.com](https://www.scaler.com/topics/cpp-static/)
[7] [https://www.w3schools.com](https://www.w3schools.com/c/ref_keyword_static.php)
[8] [https://www.beningo.com](https://www.beningo.com/131-using-the-static-keyword-in-c/)
[9] [https://www.youtube.com](https://www.youtube.com/watch?v=7Wx_o2p-6aY&t=56)
[10] [https://www.youtube.com](https://www.youtube.com/watch?v=CvVtkT_5J1s&t=252)

---

## The difference between passing an array with `[N]` and `[static N]`

When you pass an array to a function in C, the compiler ignores any size number you put inside standard brackets like `[N]` and automatically treats it as a plain pointer (*).
The difference between `[static N]` and `[N]` comes down to what the compiler enforces and how it optimizes the code.

### Comparison Breakdown

| Feature | Standard Array Syntax `[N]` | Modern Array Syntax `[static N]` |
|---|---|---|
| What the compiler sees | A standard pointer (int *arr) | A pointer that is never NULL and has at least N elements |
| Allows NULL? | Yes (can cause a crash if you don't check) | No (passing NULL is undefined behavior/compiler warning) |
| Enforces Size? | No (you can pass an array of size 1 or 2 to `[4]`) | Yes (compilers will warn you if the passed array is too small) |
| Compiler Optimization | Standard optimization | High optimization (enables aggressive loop unrolling and SIMD auto-vectorization) |

### Detailed Differences

#### 1. Compiler Warnings & Code Safety
Standard syntax does not protect you from out-of-bounds errors or null pointers, whereas static acts as a static analysis contract.

```c
void process_normal(int arr[4]);   // Compiled exactly as: void process_normal(int *arr);
void process_static(int arr[static 4]); 

int main() {
    int small[2] = {1, 2};

    process_normal(NULL);   // Compiles perfectly fine (might crash at runtime)
    process_normal(small);  // Compiles perfectly fine (might crash at runtime)

    process_static(NULL);   // WARNING: Null pointer passed to a static argument
    process_static(small);  // WARNING: Array argument is smaller than prototype requires
}
```

#### 2. Performance and Optimization (The Main Reason to Use It)
Because `int arr[static 4]` guarantees that the pointer points to a valid, continuous block of memory with at least 4 integers, the compiler can make strong assumptions:

* **No Null Checks**: The compiler completely skips checking if `arr == NULL` before looping.
* **Vectorization**: The compiler knows it can safely read 4 elements at once using CPU vector registers (SIMD) without accidentally triggering a segmentation fault by reading past the end of the array.

### A Common Misconception
Using `[static N]` does not change the memory allocation of the array. It does not make the array live forever like a static local variable does. It is strictly a contract type constraint for function parameters.

---

## No Difference Between `[N]` and `*` in Function Parameters

In C, passing `int A[N]` as a function parameter is just syntactic sugar for passing `int* A`. This behavior is known as array decay.
The compiler completely ignores the number N inside standard brackets. It throws that information away and rewrites the parameter as a plain pointer before it even compiles the code.

### Proof by Compiler Behavior
To the C compiler, the following three function declarations are 100% identical:

```c
void process_array(int *A);     // 1. Plain pointer syntax
void process_array(int A[]);    // 2. Open array syntax (decays to pointer)
void process_array(int A[100]); // 3. Fixed array syntax (STILL decays to pointer)
```

Because they all evaluate to `int *A`, you get zero guarantees:

* You can pass NULL without any compiler warnings.
* You can pass an array that is far too small (e.g., an array of only 2 elements to a function expecting `A[100]`), and the compiler will remain completely silent.

### Why does C do this?
This is a legacy design choice from the earliest days of C. Passing an entire array by value (copying every single element onto the stack) would be incredibly slow for large datasets. To keep the language fast, C was designed to automatically pass arrays by reference (a pointer to the first element).
Because standard brackets give you no protection, you traditionally have to pass the size explicitly:

```c
void legacy_process(int *A, size_t size); // The traditional, manual C way
```

This is exactly why `[static N]` was introduced in modern C—it is the only way to tell the compiler, "Hey, I am actually passing an array of a specific size, please enforce it and optimize around it!"
