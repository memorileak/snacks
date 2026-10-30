+++
title = "Modern C: The principal structure of a program"
description = "A practical introduction to the grammar, declarations, definitions, scopes, initialization, statements, iteration, function calls, and control flow that shape a C23 program."
date = 2026-09-27T16:20:11Z

[taxonomies]
tags = ["modernc", "basic"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 2.1. Grammar

### Syntactical Elements of a C Program

**What it is:**
The grammar of a C program refers to the syntactical aspects that allow the compiler to understand the code. A C program is composed of various types of text elements, categorized into special words, punctuation, comments, literals, identifiers, functions, operators, and attributes.

**Why it matters / how it works:**
Just like natural languages, C has strict lexical elements and grammar rules. These elements are rigidly specified to convey an unambiguous meaning to the compiler, allowing it to translate human-readable source code into machine-executable instructions. Understanding these elements is the first step in reading and writing valid C code.

**Key details:**

- **Special words (Keywords):** Predefined words imposed by the language that cannot be changed or redefined (e.g., `#include`, `int`, `char`, `void`, `double`, `for`, `return`).
- **Punctuation:** C uses brackets for grouping (`{ }`, `( )`, `[ ]`, `[[ ]]`, `/* */`, `< >`) and separators/terminators (`,`, `;`).
- **Comments:** Ignored by the compiler, used for documentation. `/* ... */` spans multiple lines or portions of a line, while `//` extends to the end of the line.
- **Literals:** Fixed values hardcoded into the program (e.g., `0`, `9.0`, `"element %zu is %g\n"`).
- **Identifiers:** Names given to entities like variables (`A`, `i`), type aliases (`size_t`), functions (`main`, `printf`), and constants (`EXIT_SUCCESS`).
- **Operators:** Symbols that perform operations on operands (e.g., `=` for initialization/assignment, `<` for comparison, `++` for increment, `*` for multiplication).
- **Attributes (C23 Feature):** Placed in double square brackets `[[...]]` to provide supplemental information to the compiler (e.g., `[[maybe_unused]]`).

**Pitfalls:**

- Forgetting a terminator (`;`) at the end of a statement is a frequent syntax error.
- Using a keyword as an identifier (e.g., naming a variable `int`) will cause a compilation failure.
- Mismatched brackets (e.g., forgetting a closing `}`) will confuse the compiler, often resulting in errors far away from the actual mistake.

**Connections:**

- Connects to **Lexical Structure** and the concept of an abstract state machine (Chapter 5), where these grammatical tokens represent the abstract state transitions.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// This is a C++-style comment (grammar: comments)
int main(int argc, [[maybe_unused]] char* argv[]) {
    // int, char, return are keywords.
    // main, argc, argv are identifiers.
    // [[maybe_unused]] is a C23 attribute.

    size_t count = 5; // '=' is an operator, 5 is a literal, ';' is a punctuator

    /*
       This is a multi-line comment.
       We are using brackets () and {} to structure the code.
    */
    for(size_t i = 0; i < count; ++i) { // '<' and '++' are operators
        printf("Count: %zu\n", i); // "Count: %zu\n" is a string literal
    }

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ The program compiles and prints "Count: 0" through "Count: 4" on separate lines.

_What goes wrong if:_ If we omit the `;` after `size_t count = 5`, the compiler throws a syntax error on the next line, failing to parse the grammar.

**Recap:**

- C code is built from specific lexical tokens: keywords, identifiers, literals, operators, punctuation, and attributes.
- Punctuation like brackets groups code, while semicolons terminate statements.
- Attributes like `[[maybe_unused]]` are new to C23 and provide extra hints to the compiler without changing the core logic.

---

## 2.2. Declarations

### Specifying Identifiers

**What it is:**
A declaration specifies what a particular identifier is supposed to represent before it is used in a program. It tells the compiler the properties (like type) associated with a name, distinguishing it from predefined keywords.

**Why it matters / how it works:**
The compiler needs to know the type and nature of every identifier to verify that it is used correctly in statements and expressions. This strict requirement prevents arbitrary or undeclared names from polluting the program, ensuring memory safety and logical consistency.

**Key details:**

- Declarations usually start with a **type** (e.g., `int`, `double`, `size_t`).
- They can declare variables (e.g., `size_t i;`), arrays (e.g., `double A[5];`), or functions (e.g., `int main(void);`).
- Predeclared identifiers (like `printf` or `size_t`) are brought into the program via include/header files (`<stdio.h>`, `<stdlib.h>`).

> Takeaway 2.2 #1: _All identifiers in a program have to be declared._

_Explanation:_ You cannot simply invent a variable name mid-program and start using it. The compiler must see a formal declaration of the identifier beforehand so it knows its type and how to handle it.

> Takeaway 2.2 #2: _Identifiers may have several consistent declarations._

_Explanation:_ You can declare the same function or variable multiple times (e.g., in different header files or repeated in the same file), as long as the declarations perfectly match (are consistent). A declaration merely describes a feature; it does not allocate it.

**Pitfalls:**

- Trying to use a variable before declaring it results in a compiler error.
- Providing _inconsistent_ declarations for the same identifier (e.g., declaring `int a;` and then `double a;` in the same scope) causes an error.

**Connections:**

- Declarations are deeply tied to **Types** (Chapter 5 & 6) because every declaration must specify a type for the identifier.

### Scope and Visibility

**What it is:**
Scope defines the region of the program text where an identifier is visible and can be legally referenced.

**Why it matters / how it works:**
Scope prevents naming collisions. By restricting visibility, you can safely reuse common variable names (like `i` or `count`) in different parts of your program without them interfering with each other.

**Key details:**

- **Block scope:** Variables declared inside `{ ... }` or within a function's parameter list. Visibility ends at the closing `}` of the block.
- **Primary block:** The block associated with a control structure (like `for`).
- **Secondary block:** The block containing the actual body of the control structure.
- **File scope (Globals):** Identifiers declared outside any block or parameter list. They are visible from their declaration down to the end of the file.

> Takeaway 2.2 #3: _Declarations are bound to the scope in which they appear._

_Explanation:_ An identifier is only "known" to the compiler within the specific block (or file) where it is declared. Once the execution leaves that block, the identifier is conceptually destroyed or hidden.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// Consistent redeclaration (Takeaway 2.2 #2)
int my_function(void);
int my_function(void);

// 'global_var' has file scope (visible everywhere below here)
size_t global_var = 10;

int main(void) {
    // 'local_var' has block scope (visible only inside main)
    int local_var = 5;

    // 'i' has block scope bound to the 'for' loop (primary block)
    for(size_t i = 0; i < 2; ++i) {
        // Secondary block scope
        printf("i: %zu, local_var: %d\n", i, local_var);
    }

    // printf("%zu", i); // Error: 'i' is out of scope here (Takeaway 2.2 #3)

    return EXIT_SUCCESS;
}

int my_function(void) {
    return 0;
}
```

_Expected behavior/output:_ Prints the loop output successfully.

_What goes wrong if:_ If we uncomment the `printf` accessing `i` outside the loop, the compiler throws an "undeclared identifier" error because `i`'s scope ended with the `for` loop.

**Recap:**

- Every identifier must be declared with a type before use.
- Multiple consistent declarations are harmless, often occurring via header files.
- The placement of a declaration dictates its scope (block vs. file scope), determining where the identifier can be used.

---

## 2.3. Definitions

### Defining and Initializing Objects

**What it is:**
While a declaration describes an identifier, a definition brings the corresponding object into existence by instructing the compiler to allocate storage for it. An initialization provides the initial value that goes into that storage.

**Why it matters / how it works:**
Without definitions, identifiers are just names without physical memory backing them. A program cannot run without actual storage for its variables and actual executable code for its functions. In modern C practice, defining a variable and initializing it are typically done together to prevent "garbage" values.

> Takeaway 2.3 #1: _Declarations specify identifiers, whereas definitions specify objects._

_Explanation:_ A declaration tells the compiler "there is a thing called `X` of type `int`." A definition tells the compiler "make a box in memory for `X`."

> Takeaway 2.3 #2: _An object is defined at the same time it is initialized._

_Explanation:_ In C, when you provide an initial value (e.g., `size_t i = 0;`), you are definitively instructing the compiler to allocate the storage and place that value inside it.

### Array Initialization and Uniqueness

**What it is:**
Arrays are aggregates of several items of the same type. C23 allows designated initializers to selectively initialize specific array elements. Functions are defined when their declaration is accompanied by a block `{ ... }` containing their executable code.

**Key details:**

- **Designated Initializers:** Syntax like `[4] = 3.E+25` directly targets a specific index in an array.

> Takeaway 2.3 #3: _Missing elements in initializers default to 0._

_Explanation:_ If you initialize an array but do not specify values for every single index, the C compiler automatically fills all the unspecified positions with `0` (or `0.0` for floats, etc.).

> Takeaway 2.3 #4: _For an array with n elements, the first element has index 0, and the last has index n-1._

_Explanation:_ C uses 0-based indexing. The index represents the distance from the start of the array.

> Takeaway 2.3 #5: _Each object or function must have exactly one definition._

_Explanation:_ While you can _declare_ `printf` or a variable `A` a hundred times, there can be only one definitive place in the entire program (across all files) where the memory is allocated (for objects) or the code is provided (for functions). Multiple definitions cause linker errors.

**Pitfalls:**

- Accessing an array out of bounds (e.g., `A[5]` in a 5-element array) is Undefined Behavior.
- Forgetting to initialize local variables before use can lead to reading garbage data from memory (though Takeaway 2.3 #2 advises initializing immediately).

**Connections:**

- Definitions bridge the gap between lexical syntax (Chapter 2) and the **Abstract State Machine** (Chapter 5), where defined objects hold the dynamic state of the program.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// Declaration ONLY (not a definition)
extern double global_array[5];

int main(void) {
    // Declaration AND Definition (allocates memory, initializes)
    // Using designated initializers
    double A[5] = {
        [0] = 9.0,
        [1] = 2.9,
        [4] = 3.E+25
        // [2] and [3] are missing (Takeaway 2.3 #3)
    };

    // Takeaway 2.3 #4: Index goes from 0 to 4.
    for(size_t i = 0; i < 5; ++i) {
        printf("A[%zu] = %g\n", i, A[i]);
    }

    return EXIT_SUCCESS;
}

// Exactly one definition of the global object (Takeaway 2.3 #5)
double global_array[5] = { 0 };
```

_Expected behavior/output:_ Prints values for indices 0 to 4. `A[2]` and `A[3]` will output `0` because missing elements in an initializer default to 0.

_What goes wrong if:_ If we accidentally define `double global_array[5] = {0};` again in another file, the linker will throw a "multiple definition" error.

**Recap:**

- Definitions allocate actual memory for objects or provide the executable body for functions.
- It is best practice to define objects by initializing them simultaneously.
- Array initializers default missing elements to zero, and arrays are strictly zero-indexed.
- Every object/function must have one and only one definition to avoid linker errors.

---

## 2.4. Statements

### Iteration (for)

**What it is:**
Statements are instructions that dictate what the compiler should do with the declared identifiers. The `for` statement is a domain iteration structure that repeats a block of code based on a condition.

**Why it matters / how it works:**
Statements are the active verbs of the programming language. Without them, a program is just a static collection of memory allocations. The `for` loop provides a controlled way to execute repetitive tasks without writing redundant code.

**Key details:**

- A `for` loop has 4 parts:
  1.  Initialization (executed once).
  2.  Loop condition (checked before each iteration).
  3.  Update statement (executed after each iteration).
  4.  Loop body (the secondary block containing the code to repeat).

> Takeaway 2.4.1 #2: _The loop variable should be defined in the initial part of a `for`._

_Explanation:_ Defining the loop variable (e.g., `size_t i = 0`) directly inside the `for` statement tightly restricts its scope to the loop itself. This prevents the variable from accidentally being modified or causing naming conflicts elsewhere in the function.

### Function Calls and Function Returns

**What it is:**
A function call (like `printf(...)`) suspends the current function, passes control and arguments to the called function, and resumes once the called function finishes. A function return (like `return EXIT_SUCCESS;`) hands control back to the caller, optionally passing back a value.

**Why it matters / how it works:**
Function calls enable modularity and code reuse. C uses **call by value**, meaning that arguments passed to a function are copied. The called function cannot directly modify the original variable in the caller's scope (unless pointers are used, a topic for later). The return statement is the mechanism for completing a function's execution and yielding a result.

**Key details:**

- **Call by value:** Arguments evaluate to values, and these values are what the function receives.
- **Return statement:** Must provide a value that matches the declared return type of the function (e.g., `int` for `main`).

**Pitfalls:**

- Assuming a function can modify an argument directly (e.g., `my_func(x)` where `x` changes) is a common misunderstanding of C's pass-by-value mechanics.
- Forgetting to return a value in a function declared to return a non-void type leads to undefined behavior.

**Connections:**

- Control flow statements dictate the **Execution Order** (Chapter 3), laying the foundation for more complex branching (`if`, `switch`) and looping (`while`, `do`).

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// A simple function demonstrating return and call-by-value
int square_value(int val) {
    val = val * val; // Modifies the local copy 'val'
    return val;      // Returns control and a value to the caller
}

int main(void) {
    int my_num = 5;

    // Takeaway 2.4.1 #2: Loop variable 'i' defined in the initial part
    for(size_t i = 0; i < 3; ++i) {
        // Function call: suspends main, runs square_value
        // my_num is passed BY VALUE.
        int result = square_value(my_num);
        printf("Iteration %zu: my_num = %d, result = %d\n", i, my_num, result);
    }

    // Function return: passes EXIT_SUCCESS back to the OS
    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "my_num = 5, result = 25" three times. Notice that `my_num` remains 5 in `main` despite being multiplied in `square_value`.

_What goes wrong if:_ If we declared `i` outside the loop and forgot to reset it, or if we mistakenly believed `square_value(my_num)` would permanently change `my_num` to 25, our program logic would be flawed.

**Recap:**

- Statements are instructions that manipulate data and control execution flow.
- `for` loops manage domain iterations, and loop variables should be tightly scoped inside the `for` initialization.
- Function calls suspend the current execution and pass arguments strictly by value.
- `return` ends a function's execution, passing control (and optionally a typed value) back to the caller.

---

## Summary

A C program is a structured text document combining distinct lexical elements (keywords, punctuation, identifiers) guided by strict grammar. Programmers interact with this structure by establishing semantic meaning in three layers: **declarations** (which tell the compiler the type and scope properties of identifiers), **definitions** (which instruct the compiler to allocate physical storage or define function logic), and **statements** (which instruct the compiler on how to manipulate those defined objects and control the execution flow through iterations and function calls). By strictly separating the declaration of a name from the definition of its physical object, and managing execution through block-scoped logic, C enables modular, portable, and efficient programming.

---

## Self-Check Questions

1.  **What is the difference between a declaration and a definition in C?**
    A declaration specifies the properties and type of an identifier so the compiler recognizes it, whereas a definition actually allocates memory (storage) for an object or provides the code block for a function.

2.  **What happens to array elements that are not explicitly initialized in a designated initializer?**
    Any missing elements in an array initializer automatically default to a value of 0.

3.  **Why is it recommended to define a loop variable inside the initial part of a `for` statement?**
    It restricts the variable's scope to the loop block, preventing unintended side effects or naming collisions elsewhere in the surrounding function.

4.  **Can an identifier have multiple declarations? Multiple definitions?**
    An identifier can have multiple _consistent declarations_ (e.g., in various header files), but it must have exactly _one definition_ across the entire program.

5.  **In C, does a called function receive the original variable or a copy of its value?**
    C uses "call by value," meaning the function receives a copy of the argument's value, not the original variable itself.

6.  **What are the indices of an array declared as `double arr[10];`?**
    The indices range from `0` to `9` (n-1).

7.  **What is a C23 attribute and how is it formatted syntactically?**
    An attribute provides supplemental information or hints to the compiler. It is enclosed in double square brackets, such as `[[maybe_unused]]`.

8.  **What is the difference between block scope and file scope?**
    Block scope limits an identifier's visibility to the `{ ... }` block (or loop) where it is declared. File scope (globals) makes an identifier visible from its declaration point down to the end of the file.
