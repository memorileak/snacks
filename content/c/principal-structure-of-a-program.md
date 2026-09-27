+++
title = "Modern C: The principal structure of a program"
description = "An introduction to C23 program structure, covering grammar, declarations, definitions, scopes, initialization, statements, iteration, function calls, and control flow."
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


### 3. Definitions (Section 2.3)
While declarations describe what an identifier represents, **definitions** specify objects or functions by providing their actual values or storage locations in memory.

* **Declarations vs. Definitions:** **Takeaway 2.3 #1** highlights that *declarations specify identifiers, whereas definitions specify objects*.
* **Initialization as Definition:** **Takeaway 2.3 #2** establishes that *an object is defined at the same time it is initialized*. Initializing a variable instructs the compiler to allocate storage for its value.
* **Array Rules & Designated Initializers:**
  * **Takeaway 2.3 #4:** For an array with \\(n\\) elements, the first element has index `0`, and the last has index `n-1`.
  * Designated initializers allow selective element initialization (e.g., `double A = {  = 9.0, = 2.9 }`).
  * **Takeaway 2.3 #3:** Missing elements in initializers default to `0` (or `0.0` for floating point).
* **The Single Definition Rule:** **Takeaway 2.3 #5** dictates that *each object or function must have exactly one definition* across the program.


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
