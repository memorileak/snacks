+++
title = "Modern C: Derived data types"
description = "An overview of C23 arrays, pointers, structures, bit-fields, and typedefs, with examples of how these derived types are declared and used."
date = 2026-09-28T02:42:25Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 6: Derived data types** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the focus shifts from basic numerical and scalar types to constructing complex data structures. C provides four main strategies for deriving data types (Arrays, Structures, Pointers, and Unions) plus a mechanism for creating type aliases (`typedef`).

Here is a detailed breakdown of the fundamental concepts and key takeaways from Chapter 6 across its four main sections:


### 1. Grouping Objects into Arrays (Section 6.1)
Arrays combine multiple subobjects of the **same base type** into a single encapsulating object.

* **Arrays Are Not Pointers:** Gustedt strongly emphasizes that arrays and pointers are distinct concepts. While array names decay to pointers in many contexts, treating them as identical creates deep confusion.
* **Array Objects vs. Array Values:** 
  * There are array objects, but **there are no array values**. 
  * Because arrays lack values, **arrays cannot be assigned to** (e.g., `A = B` is illegal) and **cannot be compared** using `==` or `!=`.
  * In boolean conditional contexts, an array always evaluates to `true` due to array-to-pointer decay.
* **Array Lengths (FLAs vs. VLAs):**
  * **Fixed-Length Arrays (FLAs):** Length is determined by an Integer Constant Expression (ICE) at compile time or inferred directly from an initializer.
  * **Variable-Length Arrays (VLAs):** Length is evaluated at runtime (non-ICE). In C23, automatic VLAs are optional (`__STDC_NO_VLA__`), can only use default initializers `{}`, and cannot be declared at file scope (globals).
  * **Array Size Formula:** For any array `A`, its element count is computed as `(sizeof A) / (sizeof A[0])`.
* **Array Parameters to Functions:**
  * When passed to a function, the innermost dimension of an array parameter is lost and **rewritten to a pointer**.
  * Inside function bodies, array parameters behave as if passed by reference, and using `sizeof` on an array parameter yields the size of the pointer, not the array.
* **Strings:**
  * A string in C is a **`0`-terminated array of `char`**. For example, the string literal `"hello"` has a length of 5 visible characters, but occupies 6 bytes in memory (ending with `'\0'`).
  * Passing a character array that is not null-terminated to standard string functions (like `strlen` or `strcpy`) causes program failure or undefined behavior.

#### Example

Arrays combine objects of the same type. **Arrays are not pointers**, and **there are no array values**—meaning arrays cannot be directly assigned or compared using `==`.

```c
#include <stdio.h>
#include <stddef.h>

// Function taking an array parameter:
// Note: In function headers, 'double A[len]' is rewritten by the compiler to 'double* A'.
void print_array_info(size_t len, double const A[len]) {
  // INSIDE A FUNCTION: sizeof A yields the size of the POINTER, not the array!
  printf("Inside function: sizeof A = %zu bytes (size of pointer!)\n", sizeof A);

  for (size_t i = 0; i < len; ++i) {
    printf("%g ", A[i]);
  }
  printf("\n");
}

void demo_arrays(void) {
  // 1. Fixed-Length Array (FLA) initialized with C23 universal initializer {}
  double weights = {1.5, 2.5, 3.5, 4.5};

  // Array element count formula: (sizeof A) / (sizeof A)
  size_t count = sizeof weights / sizeof weights;
  printf("Array element count: %zu (Total bytes: %zu)\n", count, sizeof weights);

  // Arrays decay to pointers when passed to functions:
  print_array_info(count, weights);

  // 2. Variable-Length Array (VLA): Length evaluated at runtime
  size_t vla_len = 3;
  double vla_arr[vla_len]; // VLA allocated on stack for dynamic size
  vla_arr = 10.0;

  // 3. Strings: 0-terminated character arrays
  // "C23" has 3 visible characters, but occupies 4 bytes ending in '\0'
  char const str[] = "C23";
  printf("String '%s' has sizeof = %zu (includes null terminator '\\0')\n", str, sizeof str);

  // ARRAYS ARE NOT ASSIGNABLE OR COMPARABLE:
  // double copy; copy = weights;  // COMPILER ERROR! Cannot assign arrays
  // if (weights == copy) ...       // ILL-ADVISED: Compares pointer addresses, not array elements!
}
```

### 2. Pointers as Opaque Types (Section 6.2)
Before delving into address arithmetic in later levels, Gustedt introduces pointers strictly as **opaque reference types**.

* **Pointer States:** A pointer does not hold data directly; it refers or "points" to data elsewhere in memory. A pointer is always in one of three states: **valid**, **null**, or **invalid**.
* **Null Pointers:**
  * Initializing or assigning a pointer with C23's **`nullptr`** keyword puts it in a null state.
  * In boolean logical expressions, null pointers evaluate to `false`.
* **Invalid Pointers & Mandatory Initialization:**
  * Uninitialized local pointers contain arbitrary, indeterminate addresses (invalid pointers). Dereferencing or evaluating an invalid pointer leads to program failure.
  * **Rule:** Always explicitly initialize pointers (either to a valid target address or to `nullptr`).

#### Example

Pointers are reference types that refer to objects in memory. A pointer is always in one of three states: **valid**, **null**, or **invalid**.

```c
#include <stdio.h>

void demo_pointers_opaque(void) {
  // 1. Null Pointer State: Explicitly initialized using C23 'nullptr'
  double* ptr1 = nullptr;

  // In boolean logical contexts, null pointers evaluate to 'false'
  if (!ptr1) {
    printf("ptr1 is nullptr (evaluates to false in boolean context)\n");
  }

  // 2. Valid Pointer State: Refers to an existing object's memory address
  double val = 42.0;
  double* ptr2 = &val; // Valid pointer referring to 'val'

  if (ptr2) { // Evaluates to 'true'
    printf("ptr2 is valid, points to value: %g\n", *ptr2);
  }

  // 3. Invalid Pointer State:
  // double* uninit_ptr; // DANGER: Uninitialized local pointer contains arbitrary address.
  // Evaluating or dereferencing an uninitialized pointer leads to UNDEFINED BEHAVIOR!
  // RULE: Always initialize pointers upon definition (e.g., to nullptr or a valid object).
}
```

### 3. Combining Objects into Structures (Section 6.3)
Structures (`struct`) group items that may have **different base types** into a single cohesive data unit accessed by named members.

* **Member Access & Passing Semantics:**
  * Structure fields/members are accessed using the dot operator (e.g., `today.tm_year`).
  * Structure arguments are **passed by value** to functions (the entire structure object is copied).
* **Assignment vs. Comparison:**
  * Structures **can be assigned** using `=` (copying all fields).
  * Structures **cannot be compared** using `==` or `!=`.
* **Padding & Memory Alignment:**
  * Compilers place structure fields in the order declared, but may insert **padding bytes** between fields to align members on hardware word boundaries.
  * There is **no padding at the start** of a structure. Reordering structure members (e.g., placing larger types first) can significantly reduce wasted padding memory.
* **Bit-Fields in C23:**
  * Bit-fields allow specifying exact bit widths for structure members (e.g., for flags or compact data).
  * Avoid bare `int` for bit-fields due to implementation-defined signedness issues. Use **`_BitInt(N)`** for numerical bit-fields of width \\(N\\), and **`bool`** for single-bit flags.

#### Example

Structures group items of **different base types** into a cohesive unit. Structs can be assigned, are passed by value, and can use padding/bit-fields.

```c
#include <stdio.h>
#include <stdbool.h>

// Struct definition with member alignment padding considerations:
// Reordering larger fields first reduces wasted padding bytes.
struct sensor_reading {
  double value;              // 8 bytes
  unsigned long timestamp;   // 8 bytes
    
  // C23 Bit-Fields: Use _BitInt(N) for bit-width numbers and bool for flags
  unsigned _BitInt(4) status_code : 4; // 4-bit status code (0 to 15)
  bool is_active                  : 1; // 1-bit boolean flag
};

// Functions receive struct parameters BY VALUE (copies the entire struct)
void print_reading(struct sensor_reading s) {
  printf("Sensor [TS: %lu]: %g (Status: %u, Active: %s)\n",
       s.timestamp, s.value, (unsigned int)s.status_code, s.is_active ? "yes" : "no");
}

void demo_structures(void) {
  // Designated Initializers: Explicitly initialize named fields; unlisted fields zero out
  struct sensor_reading r1 = {
    .value = 98.6,
    .timestamp = 1600000000UL,
    .status_code = 3wb, // C23 bit-precise integer literal
    .is_active = true
  };

  print_reading(r1);

  // STRUCTURE ASSIGNMENT (=): Copies all fields directly from r1 to r2
  struct sensor_reading r2 = r1;
  r2.value = 100.2;

  printf("r1.value = %g, r2.value = %g (Structs are independent copies!)\n", r1.value, r2.value);

  // NOTE: Structs CANNOT be compared directly!
  // if (r1 == r2) ... // COMPILER ERROR! Struct comparison is illegal in C.
}
```

### 4. Type Aliases (`typedef`) (Section 6.4)
The `typedef` keyword introduces user-defined names for existing types.

* **Aliases, Not New Types:** A `typedef` creates a new alias/name for an existing type; it **never creates a new distinct type** in C's type system.
* **Forward-Declaring Structs:** To avoid repeatedly typing the `struct` keyword, forward-declare the structure tag name inside a `typedef` using the identical identifier (e.g., `typedef struct person person;`).
* **Reserved Naming Conventions:** Identifiers ending with **`_t`** (such as `size_t`, `ptrdiff_t`) are reserved by the standard library and POSIX. Avoid naming custom types with a `_t` suffix to prevent future standard header conflicts.

#### Example

`typedef` creates a convenient **alias** for an existing type; it **does not create a new distinct type**.

```c
#include <stdio.h>

// Forward-declaring a struct tag name with a typedef using the exact same identifier
typedef struct bird bird;

// Complete struct definition matching the forward declaration
struct bird {
  char const* name;
  double wingspan;
};

// Creating a type alias for a standard type
typedef double velocity_mps; // 'velocity_mps' is an alias for 'double'

void demo_typedef(void) {
  // Clean usage without needing the 'struct' keyword repeatedly
  bird raven = { .name = "Raven", .wingspan = 1.2 };
    
  velocity_mps speed = 15.5; // Exactly equivalent to 'double speed = 15.5;'

  printf("Bird: %s, Wingspan: %g m, Speed: %g m/s\n",
       raven.name, raven.wingspan, speed);

  // BEST PRACTICE: Avoid naming custom typedefs with a '_t' suffix (e.g., bird_t),
  // because identifiers ending in '_t' are reserved by standard libraries and POSIX!
}
```
