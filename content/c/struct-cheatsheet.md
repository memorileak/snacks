+++
title = "C `struct` cheatsheet"
description = "A quick reference for the `struct` data type in C."
date = 2026-10-01T04:20:07+00:00

[taxonomies]
tags = ["struct"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Structures (`struct`) in C allow you to group objects of different types together into a single, cohesive data type.

## 1. Declaration and Type Aliasing

A structure type explains the layout but doesn't allocate memory until an object is defined.

### Basic Declaration

```c
struct birdStruct {
    char const* jay;
    char const* magpie;
    char const* raven;
};
```

### The `typedef` Idiom

To avoid repeating the `struct` keyword, modern C uses `typedef` to create type aliases. It is standard practice to use the same tag name:

```c
typedef struct birdStruct birdStruct;
struct birdStruct {
    char const* jay;
    char const* magpie;
};
```

### Opaque Structures (Forward Declarations)

You can declare a struct without defining its members to hide implementation details (e.g., in header files). Pointers to opaque structs can be freely passed around.

```c
struct toto; // forward declaration
struct toto* toto_get(void);
void toto_doit(struct toto* t);
```

## 2. Initialization

Modern C relies heavily on **designated initializers** to make code robust and readable.

### Designated Initializers

You explicitly name the members you are initializing. The order of members does not matter.

```c
struct tm today = {
    .tm_year = 2019 - 1900,
    .tm_mon  = 4 - 1,
    .tm_mday = 3,
};
```

### Default and Partial Initialization

- **Omitted fields:** Any members omitted in a designated initializer are automatically forced to `0` (or `nullptr` for pointers).
- **Default Initializer:** In C23, `{ }` is a valid universal initializer that sets all fields to zero.

```c
struct tm unknown_date = { }; // All members initialized to 0
```

## 3. Operations and Access

### Member Access

- **By Value:** Use the dot operator (`.`) to access members of a struct variable.
  ```c
  printf("Year: %d\n", today.tm_year);
  ```
- **By Pointer:** Use the arrow operator (`->`) to access members via a pointer. `a->tv_sec` is equivalent to `(*a).tv_sec`.
  ```c
  void normalize_time(struct timespec* ts) {
      if (ts) {
          ts->tv_sec += ts->tv_nsec / 1000000000;
      }
  }
  ```

### Valid Operations

1.  **Assignment:** Structures can be safely assigned to one another using `=`.
    ```c
    struct tm tomorrow = today;
    ```
2.  **Function Passing:** Structs are **passed by value**. A function receives a full copy of the struct. To modify the original or avoid copying large structs, pass a pointer instead.

### Invalid Operations

- **Comparison:** Structures **cannot** be compared directly using `==` or `!=`. You must compare them member-by-member.

## 4. Memory Layout and Padding

A struct's layout is determined by the compiler and is a critical design decision.

- **Padding:** The compiler may insert invisible "padding" bytes after any member to satisfy alignment requirements (e.g., aligning a 4-byte `int` on a 4-byte memory boundary).
- **Leading Padding:** There is _never_ any padding at the beginning of a struct. A pointer to a struct is guaranteed to point to its first member.

## 5. Advanced Struct Features

### Nested Structures

A struct can contain other structs. However, in C, nested struct definitions **do not create a new scope**.

```c
// BAD: Stardate's scope leaks out anyway, making this visually misleading
struct person {
    struct stardate { struct tm date; } bdate;
};

// GOOD: Declare them flatly on the same level
struct stardate { struct tm date; };
struct person { struct stardate bdate; };
```

### Flexible Array Members (FLA)

An array of unknown size (`[]`) can be placed as the **last member** of a structure to couple an array with its metadata.

```c
typedef struct ua32 ua32;
struct ua32 {
    size_t length;
    uint32_t data[]; // Flexible array member
};
// Must dynamically allocate enough memory for the struct + array elements
size_t len = 32;
ua32* ap = calloc(offsetof(ua32, data) + sizeof(uint32_t[len]), 1);
ap->length = len;
```

### Bit-Fields (C23 Best Practices)

You can pack data tightly by specifying the exact number of bits a member requires using `: width`.

- **Do not** use a bare `int` for bit-fields.
- Use `_BitInt(N)` for numerical bit-fields of width $N$.
- Use `bool` as the type for a flag bit-field of width 1.

```c
struct tib {
    unsigned _BitInt(6) tib_sec : 6;
    bool                tib_isdst : 1;
};
```
