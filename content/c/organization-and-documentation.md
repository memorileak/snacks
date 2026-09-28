+++
title = "Modern C: Organization and documentation"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 10: Organization and documentation** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the focus centers on structuring software projects, separating interface from implementation, documenting code clearly for maintainers, and leveraging C23 features like attributes and pure functions.

Below is a detailed breakdown of the key concepts, design rules, and takeaways from Chapter 10.


### 1. The Core Directives of Code Organization & Documentation
Gustedt outlines four fundamental rules that categorize the purpose of different code components:

* **Takeaway 10 #1 (What):** **Function interfaces describe *what* is done**.
* **Takeaway 10 #2 (What for):** **Interface comments document the purpose of a function**.
* **Takeaway 10 #3 (How):** **Function code shows *how* the function is organized**.
* **Takeaway 10 #4 (In which manner):** **Code comments explain the manner in which function details are implemented**.
* **Takeaway 10 #5:** **Separate interface and implementation**. Interfaces belong in **header files (`.h`)**, while implementations belong in **translation units (`.c`)**.
* **Takeaway 10 #6:** **Document the interface; explain the implementation**. Most users of a library read the interface specification, whereas far fewer need to inspect the internal implementation details.


### 2. Interface Documentation & Doxygen (Section 10.1)
Although C has no built-in documentation tool, standard industry practice relies on **Doxygen** to automatically generate manuals, web pages, and call graphs.

* **Takeaway 10.1 #1:** **Document interfaces thoroughly**. Use structured Doxygen tags such as `@brief` (summary), `@param` (parameter requirements), `@a` (parameter reference), and `@see` (related symbols) inside block comments (`/** ... **/`).
* **Takeaway 10.1 #2:** **Structure your code in units that have strong semantic connections**. Group functions and types that manage a specific structure or data abstraction together in a single header file.
* **Include Guards:** Protect header files against multiple inclusion using preprocessor macros (e.g., `#ifndef MYHEADER_H`, `#define MYHEADER_H 1`, `#endif`).


### 3. Implementation Clarity & Control Flow (Section 10.2)
Good implementation code should read as a literal, unambiguous description of the task.

* **Takeaway 10.2 #1:** **Implement literally**.
* **Takeaway 10.2 #2:** **Control flow must be obvious**. Use compound statements (`{}`) and clear control statements to ensure execution paths are visually distinct and predictable.


### 4. Taming Macros (Section 10.2.1)
Macros perform textual replacement prior to compilation, making them prone to subtle bugs if misused.

* **Takeaway 10.2.1 #1:** **Macros should not change the control flow in a surprising way**. Avoid defining macro aliases that conceal control constructs (such as hiding `return` statements or loop headers).
* **Takeaway 10.2.1 #2:** **Function-like macros should syntactically behave like function calls**.
  * Enclose macro parameters and expressions in parentheses to preserve operator precedence.
  * Avoid multiple evaluations of macro arguments with side effects.
  * Wrap multi-statement macros in `do { ... } while (false)` constructs to allow safe semicolon termination.


### 5. Pure Functions (Section 10.2.2)
A **pure function** operates purely on values without relying on or modifying external program state.

* **Two Defining Properties of Pure Functions:**
  1. The function has **no side effects** other than returning a value.
  2. The return value depends **strictly on its input parameters**.
* **Takeaway 10.2.2 #1:** **Function parameters are passed by value**.
* **Takeaway 10.2.2 #2:** **Global variables are frowned upon**.
* **Takeaway 10.2.2 #3:** **Express small tasks as pure functions whenever possible**.
* **Optimization Benefits:** Compilers can reorder, parallelize, or optimize pure functions easily because their impact on the abstract state machine is entirely local.


### 6. C23 Standard Attributes (Section 10.2.3)
C23 introduces standardized **attributes** (`[[...]]`) to annotate declarations, supplying compiler hints that enhance diagnostics and optimization.

* **Key C23 Attributes:**
  * **`[[deprecated]]` / `[[deprecated("reason")]]`:** Flags obsolete features so the compiler issues diagnostics upon use.
  * **`[[fallthrough]]`:** Suppresses compiler warnings when execution intentionally falls through `case` labels in a `switch`.
  * **`[[maybe_unused]]`:** Suppresses unused variable or parameter warnings.
  * **`[[nodiscard]]` / `[[nodiscard("reason")]]`:** Warns if the caller discards the returned value (vital for memory allocations or pure function results).
  * **`[[noreturn]]`:** Indicates that a function never returns to its caller (e.g., `exit` or `abort`).
  * **`[[unsequenced]]`:** Specifies that a function is pure and unsequenced, unlocking compiler optimizations.
  * **`[[reproducible]]`:** Indicates a function whose repeated execution with identical arguments yields identical results, even if it temporarily inspects state.
* **Takeaway 10.2.3 #1:** **Identifiers in attributes can be replaced by preprocessing**.
* **Takeaway 10.2.3 #2:** **Use the double underscore forms of attributes in header files** (e.g., `[[__unsequenced__]]`, `[[__nodiscard__]]`, `[[__deprecated__]]`). This prevents user-defined preprocessor macros from colliding with attribute names in public headers.
