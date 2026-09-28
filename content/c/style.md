+++
title = "Modern C: Style"
description = "A guide to readable C23 style, covering consistent formatting, identifier naming and namespace rules, meaningful names, and Unicode considerations."
date = 2026-09-28T03:44:12Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 9: Style** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the focus opens **Level 2: Cognition** by addressing code readability, human constraints, formatting automation, identifier naming conventions, and internationalization.

Here is a detailed breakdown of the key takeaways and fundamental concepts from Chapter 9:


### 1. Prime Directives of Code Style
Programs serve a dual purpose: instructing the executable and documenting intended behavior for human maintainers.
* **Readability First:** **Takeaway 9 #1** establishes that ***All C code must be readable***.
* **Human Constraints:** **Takeaway 9 #2** notes that ***Short-term memory and the field of vision are small***. A typical code view displays roughly 30 lines of 80 columns (~2,400 characters); everything outside this window must be held in human memory.
* **Cultural Context:** **Takeaway 9 #3** states ***Coding style is not a question of taste but of culture***, and **Takeaway 9 #4** adds ***Each established project constitutes its own cultural space***. Developers should adapt to the established rules of the repository they join.


### 2. Formatting (Section 9.1)
White space and visual structure align code with human visual habits.
* **Consistency:** **Takeaway 9.1 #1** mandates: ***Choose a consistent strategy for white space and other text formatting***.
* **Book Conventions:** Gustedt illustrates several specific formatting practices:
  * Use **prefix notation for blocks**, placing the opening brace `{` at the end of the line.
  * **Bind type modifiers and qualifiers to the left** (e.g., `char* name;` or `char const* const path_name[[deprecated]];`) to separate the type visually from the identifier.
  * Attach function parentheses `()` directly to the identifier, but separate control condition parentheses `()` with a space.
* **Automatic Formatting:** **Takeaway 9.1 #2** advises: ***Have your text editor automatically format your code correctly***. Tools like `astyle`, `clang-format`, or Emacs automatically maintain consistent team layout and prevent noisy diffs.

#### Example

Code formatting aligns C source code with human visual constraints, such as small short-term memory and limited fields of vision. Gustedt lays out specific formatting conventions:
* **Takeaway 9.1 #1:** *Choose a consistent strategy for white space and other text formatting*.
* **Left-binding qualifiers:** Bind type qualifiers (`const`) and pointer operators (`*`) to the left of the type.
* **Parentheses placement:** Attach function parentheses directly to the function name, but place a space after control flow keywords (`if`, `while`, `for`).

```c
#include <stdio.h>
#include <stdbool.h>

// 1. LEFT-BINDING TYPE QUALIFIERS & MODIFIERS:
// Binding qualifiers to the left clearly separates the type specification from the identifier name.
// 'char const* const' -> const pointer to const char
void print_path(char const* const path_name) {
    // 2. PARENTHESES ATTACHMENT:
    // Control flow 'if' has a space before '(': if (cond)
    // Function call 'puts' attaches '(' directly: puts(path_name)
    if (path_name) { // Prefix block notation: opening '{' stays on the same line
        puts(path_name);
    } // Closing '}' starts on a new line at the parent indentation level
}

void demo_formatting(void) {
    double value = 42.0;
    double const* ptr = &value; // Left-binding: 'const' modifies 'double', '*' binds to type

    if (ptr) {
        printf("Value: %g\n", *ptr);
    }
}
```

### 3. Naming Conventions & Identifiers (Section 9.2)
Naming requires balancing technical language constraints with semantic clarity.

#### Technical Restrictions & API Protection
* **Consistency:** **Takeaway 9.2 #1** requires: ***Choose a consistent naming policy for all identifiers***.
* **Conforming Headers:** **Takeaway 9.2 #2** states ***Any identifier that is visible in a header file must be conforming***.
* **Reserved Name Rules:** 
  * Identifiers starting with `__` or `_` followed by a capital letter are reserved for internal compiler use.
  * Single leading underscores `_` are reserved for file-scope items and struct/union/enum tags.
  * Macro names must be written in **ALL_CAPS**.
  * Identifiers starting with `str` or ending in `_t` are reserved by standard headers and POSIX.
* **Global Namespace Protection:** **Takeaway 9.2 #3** warns: ***Don’t pollute the global space of identifiers***. Expose only necessary API types and functions, using distinct project prefixes (e.g., `pthread_` or `p99_`). Use private prefixes (e.g., `p00_`) for internal header parameters to avoid macro collision bugs.

#### Example

To prevent naming collisions in larger projects or standard headers, identifier names must follow strict technical constraints:
* **Takeaway 9.2 #2:** *Any identifier that is visible in a header file must be conforming*.
* **Takeaway 9.2 #3:** *Don’t pollute the global space of identifiers*.
* **Reserved names:** Identifiers starting with `__` or `_` followed by an uppercase letter are reserved for compilers. Identifiers ending in `_t` or starting with `str` are reserved by POSIX and standard headers. Prefix API exports with a unique project tag (e.g., `myproj_`), and internal header macros with a private prefix (e.g., `p00_`).

```c
#ifndef MYPROJ_HEADER_H
#define MYPROJ_HEADER_H

#include <stddef.h>

// BAD / NON-CONFORMING EXAMPLES (Avoid!):
// int __internal_var;   // RESERVED: Leading '__' is reserved for compiler implementation
// typedef int my_type_t; // RESERVED: Trailing '_t' is reserved for POSIX/C standard library
// void strcmp(void);    // RESERVED: 'str' prefix is reserved by <string.h>

// GOOD PRACTICE (Takeaway 9.2 #3): Protect global namespace using a clear project prefix 'myproj_'
typedef struct myproj_sensor myproj_sensor; // Forward declaration tag and alias use project prefix

struct myproj_sensor {
    double value;
    size_t id;
};

// API Function: Uses project prefix and clear verb action
void myproj_sensor_init(myproj_sensor* sensor_ptr, size_t id);

#endif // MYPROJ_HEADER_H
```

#### Semantic Naming Rules
* **Recognizability:** **Takeaway 9.2 #4** states ***Names must be recognizable and quickly distinguishable***. Single-letter loop variables (`i`, `n`) are fine in tight, visible scopes, but broader symbols require clarity.
* **Pragmatic Creativity:** **Takeaway 9.2 #5** notes ***Naming is a creative act***. Standard conventions like CamelCase, Snake_case, or Hungarian notation have trade-offs, so readability in context is key.
* **Scope & Entity Roles:**
  * **Takeaway 9.2 #6:** ***File scope identifiers must be comprehensive***.
  * **Takeaway 9.2 #7:** ***A type name identifies a concept*** (e.g., `timespec` \\(\rightarrow\\) time, `person` \\(\rightarrow\\) individual data structure).
  * **Takeaway 9.2 #8:** ***A global constant identifies an artifact*** (e.g., `M_PI`, `SIZE_MAX`, `false`).
  * **Takeaway 9.2 #9:** ***A global variable identifies state*** (e.g., `toto_initialized`).
  * **Takeaway 9.2 #A:** ***A function or functional macro identifies an action***, typically incorporating a verb (e.g., `strcmp`, `getFlag`, `matrixMult`).

#### Example

Gustedt defines distinct semantic roles based on what the identifier represents:
* **Takeaway 9.2 #7:** *A type name identifies a concept* (e.g., `person`, `timespec`, `sensor`).
* **Takeaway 9.2 #8:** *A global constant identifies an artifact* (e.g., `MYPROJ_MAX_BUFFER`, `SIZE_MAX`, `false`).
* **Takeaway 9.2 #9:** *A global variable identifies state* (e.g., `myproj_initialized`).
* **Takeaway 9.2 #A:** *A function or functional macro identifies an action* (incorporates a verb, e.g., `myproj_compute_square`, `strcmp`).

```c
#include <stdio.h>
#include <stdbool.h>

// Takeaway 9.2 #8: Global constant identifies an ARTIFACT (uses ALL_CAPS macro / constexpr)
#define MYPROJ_MAX_BUFFER_SIZE 1024U

// Takeaway 9.2 #9: Global variable identifies STATE ( frowned upon, but uses explicit state name)
static bool myproj_is_initialized = false;

// Takeaway 9.2 #7: Type name identifies a CONCEPT ('myproj_account' represents a bank account concept)
typedef struct myproj_account {
    size_t account_id;
    double balance;
} myproj_account;

// Takeaway 9.2 #A: Function identifies an ACTION (uses verb 'deposit' or 'get')
bool myproj_account_deposit(myproj_account* account_ptr, double amount) {
    if (!account_ptr || amount <= 0.0) {
        return false;
    }
    account_ptr->balance += amount;
    return true;
}

double myproj_account_get_balance(myproj_account const* account_ptr) {
    return account_ptr ? account_ptr->balance : 0.0;
}
```

### 4. Internationalization & Unicode Identifiers (Section 9.3)
C23 standardizes extended character set support in source code while providing rules to prevent obfuscation.
* **Project Language:** **Takeaway 9.3 #1** states ***The natural language of a project should be chosen to accommodate the majority of the participants***.
* **Unicode Normalization:**
  * **Takeaway 9.3 #2:** ***Alphabetic letters are only allowed in identifiers if they map to themselves for Normalization Form C***.
  * **Takeaway 9.3 #3:** ***Only use alphabetic letters in identifiers if they originate directly from natural languages or they are clearly distinctive from all natural languages***.
  * **Takeaway 9.3 #4:** ***Only use letters from different scripts or variations of decimal digits in identifiers if they are clearly distinctive from one another*** (preventing visual confusion between lookalike characters across scripts).
  * **Takeaway 9.3 #5:** ***Using subscript or superscript letters in identifiers is not portable***.

#### Example

Identifiers must be quickly distinguishable to human eyes, avoiding easily confused symbols (like `l` vs `1` vs `I`, or `O` vs `0`). C23 standardizes Unicode character usage, but limits non-ASCII letters to those mapping strictly to Normalization Form C.
* **Takeaway 9.2 #4:** *Names must be recognizable and quickly distinguishable*.
* **Takeaway 9.3 #4:** *Only use letters from different scripts or variations of decimal digits in identifiers if they are clearly distinctive from one another*.

```c
#include <stdio.h>

void demo_recognizability(void) {
    // BAD / CONFUSING IDENTIFIERS (Avoid!):
    // size_t l1ll1 = 10;     // Unreadable: 'l' and '1' look identical in many terminal fonts
    // size_t myLineNumber;  // Hard to distinguish quickly from 'myLimeNumber' in large files

    // GOOD / DISTINGUISHABLE IDENTIFIERS (Takeaway 9.2 #4):
    size_t low_bit_index = 0;
    size_t high_bit_index = 31;

    // UNICODE / MATH SYMBOLS IN DOCUMENTATION & C23 SOURCE (Section 9.3):
    // Standard Latin identifiers are preferred for maximum portability, but math symbols
    // (like pi) can be used in comments/documentation or clean C23 identifiers if distinctive.
    double const pi = 3.1415926535; // Clean, standard ASCII variable name

    printf("Bit range: [%zu, %zu], Pi: %g\n", low_bit_index, high_bit_index, pi);
}
```
