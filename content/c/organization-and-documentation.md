+++
title = "Modern C: Organization and documentation"
description = "A guide to structuring C23 projects, documenting interfaces, clarifying implementations, writing safe macros, and using pure functions and standard attributes."
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

#### Example

Interfaces describe **what** a unit does and belong in header files (`.h`). To protect public headers against macro collisions and multiple inclusions:
* **Takeaway 10 #5:** Separate interface (`.h`) and implementation (`.c`).
* **Takeaway 10.1 #1:** Document interfaces thoroughly using structured Doxygen tags (`@brief`, `@param`, `@return`).
* **Takeaway 10.2.3 #2:** Use the double-underscore forms of attributes in header files (e.g., `[[__nodiscard__]]`, `[[__unsequenced__]]`) so user-defined preprocessor macros do not accidentally override attribute keywords.

```c
/**
 * @file math_utils.h
 * @brief Public interface for core numerical utilities.
 *
 * Takeaway 10 #1: Interfaces describe WHAT a module does.
 * Takeaway 10.1 #2: Group functions with strong semantic connections together.
 */

#ifndef MATH_UTILS_H
#define MATH_UTILS_H

#include <stddef.h>

/**
 * @brief Computes the cube of a given double-precision floating-point number.
 *
 * @param x The input value to be cubed.
 * @return The result of x * x * x.
 *
 * Takeaway 10.2.3 #2: Use double-underscore attribute forms in public headers
 * to prevent collisions with potential user macros (e.g., [[__nodiscard__]], [[__unsequenced__]]).
 */
[[__nodiscard__("Discarding pure computation result"), __unsequenced__]]
double math_cube(double x);

/**
 * @brief Legacy integer squaring function.
 *
 * @deprecated Use math_cube or direct arithmetic instead.
 */
[[__deprecated__("Use direct multiplication or math_cube instead")]]
int math_old_square(int x);

#endif // MATH_UTILS_H
```

### 3. Implementation Clarity & Control Flow (Section 10.2)
Good implementation code should read as a literal, unambiguous description of the task.

* **Takeaway 10.2 #1:** **Implement literally**.
* **Takeaway 10.2 #2:** **Control flow must be obvious**. Use compound statements (`{}`) and clear control statements to ensure execution paths are visually distinct and predictable.

#### Example

Implementation files (`.c`) contain the code showing **how** tasks are executed.
* **Takeaway 10 #3 & #6:** Code shows *how* the function is organized; explain implementation details locally.
* **Takeaway 10.2.2 #3:** Express small tasks as **pure functions** (`[[unsequenced]]`). Pure functions have no side effects, and their return value depends strictly on input parameters passed by value.

```c
/**
 * @file math_utils.c
 * @brief Internal implementation of numerical utilities.
 *
 * Takeaway 10 #5: Implementation details stay inside translation units (.c).
 */

#include "math_utils.h"

// Pure function implementation:
// Has no side effects, does not read/modify global state, and output depends strictly on 'x'.
double math_cube(double x) {
    // Takeaway 10.2 #1: Implement literally and clearly
    return x * x * x;
}

int math_old_square(int x) {
    return x * x;
}
```

### 4. Taming Macros (Section 10.2.1)
Macros perform textual replacement prior to compilation, making them prone to subtle bugs if misused.

* **Takeaway 10.2.1 #1:** **Macros should not change the control flow in a surprising way**. Avoid defining macro aliases that conceal control constructs (such as hiding `return` statements or loop headers).
* **Takeaway 10.2.1 #2:** **Function-like macros should syntactically behave like function calls**.
  * Enclose macro parameters and expressions in parentheses to preserve operator precedence.
  * Avoid multiple evaluations of macro arguments with side effects.
  * Wrap multi-statement macros in `do { ... } while (false)` constructs to allow safe semicolon termination.

#### Example

Macros perform raw text replacement. If improperly written, they cause precedence bugs or syntax failures.
* **Takeaway 10.2.1 #1:** Macros must not change control flow in surprising ways.
* **Takeaway 10.2.1 #2:** Function-like macros should syntactically behave like standard function calls:
  1. Enclose all macro parameters and full expressions in parentheses `()`.
  2. Enclose multi-statement macros in `do { ... } while (false)` so they safely require a trailing semicolon `;` when used inside `if-else` blocks.

```c
#include <stdio.h>
#include <stdbool.h>

// BAD MACRO (Avoid!): Unparenthesized arguments cause precedence bugs!
// #define BAD_SQUARE(x) x * x  --> BAD_SQUARE(1 + 2) expands to 1 + 2 * 1 + 2 = 5!

// GOOD FUNCTION-LIKE MACRO:
// Parenthesizes arguments and full expression to preserve precedence
#define SAFE_SQUARE(x) ((x) * (x))

// MULTI-STATEMENT MACRO:
// Enclosed in do { ... } while (false) so it acts as a single compound statement
// requiring a trailing semicolon.
#define SWAP_INTEGERS(a, b)             \
    do {                                \
        int temp_val = (a);             \
        (a) = (b);                      \
        (b) = temp_val;                 \
    } while (false)

void demo_taming_macros(void) {
    int x = 1 + 2;
    // Expands to ((1 + 2) * (1 + 2)) = 9
    printf("SAFE_SQUARE(1 + 2) = %d\n", SAFE_SQUARE(x));

    int val1 = 10, val2 = 20;

    // do-while(false) allows safe inclusion inside if-else blocks with semicolons:
    if (val1 < val2)
        SWAP_INTEGERS(val1, val2); // Semicolon cleanly terminates the do-while wrapper
    else
        puts("No swap needed");

    printf("Swapped values: val1 = %d, val2 = %d\n", val1, val2);
}
```

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

#### Example

C23 introduces attributes (`[[...]]`) to guide compiler optimizations and suppress warnings cleanly.
* **`[[nodiscard]]`:** Warns if the return value of a function is ignored.
* **`[[maybe_unused]]`:** Suppresses unused variable or parameter compiler warnings.
* **`[[fallthrough]]`:** Informs the compiler that falling through `case` labels in a `switch` is deliberate.

```c
#include <stdio.h>
#include <stdlib.h>
#include "math_utils.h"

// Attribute [[maybe_unused]] on parameters that are intentionally unused
int main([[maybe_unused]] int argc, [[maybe_unused]] char* argv[]) {

    // 1. [[nodiscard]] DEMONSTRATION:
    // math_cube is tagged with [[nodiscard]]; assigning its return value suppresses warnings.
    double result = math_cube(3.0);
    printf("Cube of 3.0 = %g\n", result);

    // 2. [[maybe_unused]] ON LOCAL VARIABLES:
    [[maybe_unused]] int debug_code = 404; // Suppresses "unused variable" warning

    // 3. [[fallthrough]] IN SWITCH STATEMENTS:
    int option = 1;
    switch (option) {
        case 1:
            puts("Mode 1 activated: Initializing base settings...");
            [[fallthrough]]; // Informs compiler that fall-through to case 2 is intentional!

        case 2:
            puts("Mode 2 activated: Running standard execution pipeline.");
            break;

        default:
            puts("Default mode");
            break;
    }

    return EXIT_SUCCESS;
}
```
