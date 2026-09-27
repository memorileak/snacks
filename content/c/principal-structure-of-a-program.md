+++
title = "Modern C: The principal structure of a program"
description = "A practical introduction to the grammar, declarations, definitions, scopes, initialization, statements, iteration, function calls, and control flow that shape a C23 program."
date = 2026-09-27

[taxonomies]
tags = ["modernc"]

# [extra]
# math = false
# cover.image = "images/cover.png"
+++

Chapter 2 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"The principal structure of a program,"** lays out the foundational syntax, semantics, and structural rules required to write and understand portable C programs. 

Here is a detailed breakdown of the most important concepts and key takeaways organized by the chapter's four core sections:


### 1. Grammar (Section 2.1)
A C program is built from specific textual elements assembled according to strict syntactical rules:

* **Special Words:** Words like `int`, `double`, `for`, `return`, `void`, and `char` are language keywords that specify fixed features imposed by C and cannot be changed or redefined.
* **Punctuation & Brackets:** C uses six types of paired brackets for grouping and structure: `{...}`, `(...)`, `[...]`, `[[...]]`, `/*...*/`, and `<...>`. It also uses commas `,` and semicolons `;` as separators and terminators.
* **Comments:** Code documentation is enclosed in block comments `/*...*/` or C++-style single-line comments `//`, both of which are ignored by the compiler.
* **Literals:** Fixed values embedded directly in the program text, such as numbers (`0`, `9.0`, `3.E+25`) or character string literals.
* **Identifiers:** Names given to entities in the program. They represent:
  * **Variables/Data objects** (e.g., `A`, `i`).
  * **Type aliases** (e.g., `size_t`, where the trailing `_t` convention signals a type).
  * **Functions** (e.g., `main`, `printf`).
  * **Constants** (e.g., `EXIT_SUCCESS`).
* **Operators:** Fundamental tokens that perform actions, such as `=` (initialization/assignment), `<` (comparison), `++` (increment), and `*` (multiplication).
* **Attributes (C23):** Constructs like `[[maybe_unused]]` placed inside double square brackets to pass supplementary information to the compiler.

#### Example

C grammar is built from special keywords, brackets, punctuation, comments, literals, operators, and attributes.

```c
/* Block comment: Explains high-level module behavior to human readers */
#include <stdio.h>   // Preprocessor directive importing system standard I/O library
#include <stdlib.h>  // Header providing EXIT_SUCCESS macro identifier

// C23 attribute syntax [[maybe_unused]] informs compiler that parameter may intentionally be unused
int main(int argc, [[maybe_unused]] char* argv[]) {
    
  // Literals: Fixed values directly embedded in program text (0, 5, 2.9, "text\n")
  // Identifiers: Names given to entities like variables ('A'), type aliases ('size_t'), functions ('printf')
  // Operators: Action symbols like '=' (assignment/init), '<' (comparison), '*' (multiplication)

  size_t count = 5; // 'size_t' uses '_t' convention indicating it is a type alias
    
  printf("Lexical demo completed for count = %zu\n", count); // Function call statement
  return EXIT_SUCCESS; // 'EXIT_SUCCESS' is a predefined macro constant identifier
}
```

### 2. Declarations (Section 2.2)
Declarations inform the compiler what an identifier represents before it can be used in statements.

* **The Mandatory Declaration Rule:** **Takeaway 2.2 #1** states that *all identifiers in a program have to be declared*.
* **Properties Specified by Type:** A declaration binds an identifier to a specific type (e.g., `int argc`, `size_t i`, `double A`). Parentheses `()` designate a function, brackets `[]` declare an array of elements, and an asterisk `*` indicates a pointer.
* **The "Box" Analogy:** An object can be visualized as a memory box where the **box** is the object, the **type** is the specification, the **value** is the contents inside, and the **identifier** is the label on the outside.
* **Scopes and Visibility:** **Takeaway 2.2 #3** specifies that *declarations are bound to the scope in which they appear*. Scope defines where an identifier is visible:
  * **Block Scope:** Identifiers declared inside `{}` compound statements or loop headers (`for`) are restricted to those local blocks.
  * **Function Parameter Scope:** Parameters like `argc` and `argv` are visible throughout the entire body of the function.
  * **File Scope (Globals):** Identifiers declared outside any function (like `main` itself) are visible from their declaration point to the end of the source file.
* **Consistent Declarations:** **Takeaway 2.2 #2** notes that *identifiers may have several consistent declarations*, provided they do not contradict one another within the same scope.

  #### Example

  Declarations introduce identifiers to the compiler and specify their properties (such as type, array dimensions, or function signatures) before they are used. 

  * **Takeaway 2.2 #1:** *All identifiers in a program have to be declared*.
  * **Takeaway 2.2 #2:** *Identifiers may have several consistent declarations*.
  * **Takeaway 2.2 #3:** *Declarations are bound to the scope in which they appear*.

  ```c
  #include <stdio.h>
  #include <stddef.h>

  // 1. FILE SCOPE (Globals): Visible from declaration point to end of source file
  extern int global_counter; // Declaration only (specifies identifier without defining storage)
  extern int global_counter; // Takeaway 2.2 #2: Redeclarations are allowed if consistent

  int global_counter = 100;   // Definition allocating actual storage

  // Function parameter scope: 'a' and 'b' are visible throughout the function body
  void demo_scopes(double a) {
    
    // 2. BLOCK SCOPE (Outer Block): Visible inside this compound statement block
    double x = a * 2.0; 

    if (x > 5.0) {
      // 3. NESTED BLOCK SCOPE (Inner Primary/Secondary Block):
      // Variable 'i' is visible ONLY inside this if-block
      size_t i = 1; 
      printf("Inner scope: x = %g, i = %zu\n", x, i);
    } 
    // 'i' is no longer visible or accessible here!
    
    // The "Box" Analogy:
    // Identifier label = 'x'
    // Type specifier   = 'double'
    // Stored object    = memory allocated for double
    // Value contents   = computed result of (a * 2.0)
  }
  ```

### 3. Definitions (Section 2.3)
While declarations describe what an identifier represents, **definitions** specify objects or functions by providing their actual values or storage locations in memory.

* **Declarations vs. Definitions:** **Takeaway 2.3 #1** highlights that *declarations specify identifiers, whereas definitions specify objects*.
* **Initialization as Definition:** **Takeaway 2.3 #2** establishes that *an object is defined at the same time it is initialized*. Initializing a variable instructs the compiler to allocate storage for its value.
* **Array Rules & Designated Initializers:**
  * **Takeaway 2.3 #4:** For an array with `n` elements, the first element has index `0`, and the last has index `n-1`.
  * Designated initializers allow selective element initialization (e.g., `double A = {  = 9.0, = 2.9 }`).
  * **Takeaway 2.3 #3:** Missing elements in initializers default to `0` (or `0.0` for floating point).
* **The Single Definition Rule:** **Takeaway 2.3 #5** dictates that *each object or function must have exactly one definition* across the program.

#### Example

While declarations describe an identifier's properties, **definitions** allocate storage for objects or provide function bodies.

* **Takeaway 2.3 #1:** *Declarations specify identifiers, whereas definitions specify objects*.
* **Takeaway 2.3 #2:** *An object is defined at the same time it is initialized*.
* **Takeaway 2.3 #3:** *Missing elements in initializers default to 0*.
* **Takeaway 2.3 #4:** *For an array with `n` elements, the first element has index 0, and the last has index `n-1`*.
* **Takeaway 2.3 #5:** *Each object or function must have exactly one definition*.

```c
#include <stdio.h>

void demo_definitions_and_initializers(void) {
    
  // Variable Definition via Initialization: Allocates storage and sets initial value
  size_t total = 0; // Takeaway 2.3 #2

  // Designated Array Initializer (Takeaway 2.3 #3 & #4):
  // Array of 5 double elements (valid indices: 0 to 4)
  double A = {
     = 9.0,      // Index 0 set to 9.0
    = 2.9,      // Index 1 set to 2.9
    = 3.E+25    // Index 4 set to 3.0e25
    // Indices and are unlisted and default to 0.0
  };

  printf("A = %g, A (defaulted) = %g, A = %g\n", A, A, A);
  printf("Total initialized count = %zu\n", total);
}
```

### 4. Statements & Program Execution (Section 2.4)
Statements are instructions that tell the computer what actions to perform on declared objects.

* **Domain Iteration (`for` loops):**
  * **Takeaway 2.4.1 #1:** *Domain iterations should be coded with a `for` statement*.
  * **Takeaway 2.4.1 #2:** *The loop variable should be defined in the initial part of a `for`* (e.g., `for (size_t i = 0; i < 5; ++i)`), constraining its scope tightly to the loop block.
* **Function Calls & Call-by-Value:**
  * A function call (e.g., `printf(...)`) temporarily suspends execution of the current function and passes argument *values* to the called function.
  * C strictly uses **call-by-value**: a called function receives local copies of argument values and cannot modify the caller's variables directly.
* **Function Returns & Control Flow:**
  * A `return` statement terminates execution of the current function and returns control (along with a return value) back to the caller.
  * Execution flows from the operating system's startup routine to `main()`, cascades out to library functions like `printf()`, and returns back up the stack until `main()` returns `EXIT_SUCCESS` or `EXIT_FAILURE` to the system.

  #### Example

  Statements instruct the computer on how to manipulate declared objects.

  * **Takeaway 2.4.1 #1:** *Domain iterations should be coded with a `for` statement*.
  * **Takeaway 2.4.1 #2:** *The loop variable should be defined in the initial part of a `for`*.

  ```c
  #include <stdio.h>
  #include <stdlib.h>

  // Pure helper function: Parameters received via Call-by-Value (copies of argument values)
  double compute_square(double val) {
    return val * val; // Returns control and value to the caller
  }

  int main(void) {
    double data = {1.5, 2.5, 3.5};

    // Domain Iteration over array domain:
    // Loop variable 'i' is declared directly inside the 'for' header (Takeaway 2.4.1 #2)
    // Scope of 'i' is restricted strictly to the loop body
    for (size_t i = 0; i < 3; ++i) { // Takeaway 2.4.1 #1
        
      // Function Call: Temporarily suspends main(), passes data[i] by value to compute_square()
      double sq = compute_square(data[i]); 
        
      printf("Element %zu: %g squared is %g\n", i, data[i], sq); // Call to C library function printf
    }

    // Function Return: Sends control and exit code back to system process startup
    return EXIT_SUCCESS; 
  }
  ```
