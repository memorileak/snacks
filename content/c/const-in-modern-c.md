+++
title = "The `const` in modern C"
description = "The `const` keyword in C is a powerful tool for enforcing data safety and immutability. This article explores its role in modern C programming, particularly with the introduction of `constexpr` in C23."
date = 2026-10-01T03:43:31+00:00

[taxonomies]
tags = ["const"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

An object of a `const`-qualified type in C is read-only, meaning the compiler prevents you from modifying it through that specific identifier. However, this qualification acts as a contractual interface for your code rather than an absolute physical lock on the underlying memory.

To master modern C, particularly with the standard updates in C23, engineering students must distinguish between variables that are protected from modification at runtime and true compile-time constants.

### 1. `const` as a Programming Contract

The primary engineering use of `const` is to enforce data safety across function boundaries. When you pass a pointer to a function, `const` guarantees that the function will not mutate the caller's original data.

Modern C formatting often relies on writing `const` immediately after the type it qualifies, which clarifies exactly what is being protected. For example, `char const*const` specifies both that the characters cannot be altered and that the pointer itself cannot be redirected to a new memory address.

```c
#include <stdio.h>

// The parameter 'str' is a read-only pointer to read-only characters.
void print_message(char const*const str) {
    // str[0] = 'H'; // ERROR: Cannot mutate the characters.
    // str = "New";  // ERROR: Cannot reassign the pointer itself.
    printf("%s\n", str);
}

int main(void) {
    char const*const greeting = "Hello, Engineers";
    print_message(greeting);
    return 0;
}

```

In this scenario, `const` shifts potential runtime memory corruption bugs into immediate compile-time errors.

### 2. C23 and Compile-Time Guarantees: `constexpr`

For decades, developers used `const` or preprocessor `#define` macros to declare constants, but both have flaws. Macros lack type safety, and a `const`-qualified object simply means its value cannot be modified after initialization—it does not mean its value is known during compilation.

C23 introduces the `constexpr` construct, which results in read-only objects that are guaranteed to never change, because their values are fully determined at compile time. An integer constant expression must only evaluate objects that are declared with `constexpr`, making it the required choice for array sizes or `switch` case labels.

```c
// constexpr guarantees the value is embedded at compile-time.
constexpr double PI = 3.14159265358979323846;

// Valid: constexpr objects can be used to size arrays safely.
double circle_areas[ (constexpr int){ 10 } ];

```

C23 enforces strict type safety for `constexpr` initialization. The initializer of a `constexpr` object must fit the target type exactly without precision loss.

```c
// ERROR: Compilation fails because significant digits after
// the decimal point are lost when converting to an unsigned integer.
constexpr unsigned PI_flat = 3.14159265358979323846;

```

### 3. The Reality of Memory Aliasing

A common misconception is that a `const` qualification makes the physical memory completely immutable. In reality, the fact that a specific object is `const`-qualified does not mean the compiler or run-time system might not change its value. Other parts of the program may see that exact same object without the qualification and change it.

This phenomenon is known as aliasing. If two pointers point to the exact same memory address, but only one is `const`-qualified, the data can mutate "behind the back" of the read-only pointer.

```c
#include <stdio.h>

void execute_process(int const* status, int* mutable_alias) {
    printf("Initial status: %d\n", *status); // Output: 0

    // The data is modified through a different pathway.
    *mutable_alias = 1;

    // The const pointer now reads a different value!
    printf("Mutated status: %d\n", *status); // Output: 1
}

int main(void) {
    int hardware_flag = 0;

    // Passing the exact same memory address to both parameters.
    execute_process(&hardware_flag, &hardware_flag);
    return 0;
}

```

Because of aliasing, `const` should be viewed strictly as an access right applied to a specific variable name, not a physical hardware lock.

### Implementation Strategy for Modern C

To write safe and highly optimizable C code, adopt the following rules:

- Use `constexpr` for any fixed limit, mathematical constant, or configuration value known before compiling.
- Use `const` for data fetched at runtime (like sensor readings or user inputs) that should not be altered after its initial assignment.
- Systematically apply `const` to pointer parameters in your function interfaces to prevent accidental memory mutation.

### A Note on Physical Memory Protection

If the variable was originally declared as a global `const`, the compiler might place it in a physical Read-Only section of memory (like `.rodata`). If you try the cast-trick on that, the program will compile but crash at runtime with a Segmentation Fault. This physical memory protection and associated risk of crashing applies to several other unmutable entities in C:

- **String Literals:** String literals are inherently read-only. However, for backward compatibility, their type does not explicitly include the `const` keyword, making them an easy target for accidental mutation attempts that will result in a crash.

- **Literal Overlapping (Shared Memory):** Read-only object literals, including string literals and `const`-qualified compound literals, may overlap in physical memory. The compiler is permitted to map identical literals (or even overlapping suffixes, such as "end" and "friend") to the exact same memory address to save space. Forcing a mutation on one via a pointer cast could corrupt other seemingly unrelated variables throughout your program.

- **Temporary Objects:** Objects with a temporary lifetime, such as those returned by a function call to access a specific struct member, are strictly read-only.

  ```c
  #include <stdio.h>

  typedef struct {
      int x;
      int y;
  } Point;

  Point get_origin() {
      Point p = {0, 0};
      return p; // Returning a temporary object
  }

  int main() {
      // In this example, get_origin() returns a temporary Point object (a rvalue)
      // Reading the value of a temporary object is valid:
      int m = get_origin().x;

      // Writing to a temporary object is INVALID
      // The compiler will report an error:
      get_origin().x = 10;

      // But, if you store the temporary object into a variable, you can modify it:
      Point my_point = get_origin(); // Store the temporary object into a variable
      my_point.x = 10;               // Valid, because 'my_point' is an lvalue
  }
  ```

- **General Access Violations:** Modifying any unmutable object—whether it is a `const`-qualified object, a string literal, or a temporary object—is classified as an access violation and will lead to program failure.
