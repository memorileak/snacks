+++
title = "Modern C: Style"
description = "A guide to readable C23 style, covering consistent formatting, identifier naming and namespace rules, meaningful names, and Unicode considerations."
date = 2026-09-28T03:44:12Z

[taxonomies]
tags = ["modernc", "style", "formatting"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 9. Style

### The Purpose of Source Code

**What it is:**
Source code in C serves a dual purpose. It is simultaneously a set of strict, formal instructions intended for a compiler to generate an executable, and a descriptive document meant to be read, understood, and maintained by human beings (programmers, users, maintainers).

**Why it matters / how it works:**
While the compiler only cares about syntax validity, humans are constrained by physical limitations (like short-term memory and visual field size) and cultural baggage. If code is unreadable, it becomes unmaintainable, prone to bugs, and impossible to collaborate on. The structure, spacing, and naming in the code are the primary tools a programmer has to convey intent and logic.

**Key details:**

- Programs are documentation of the intended behavior of a system.
- The human condition requires visual sync and recognizable patterns to process information efficiently.
- Coding style is a cultural agreement within a project, not just a matter of personal taste.

**Pitfalls:**

- Ignoring the human reader and optimizing strictly for the compiler (e.g., stripping all whitespace) makes the codebase a "write-only" artifact that cannot be updated safely.

> Takeaway 9 #1: All C code must be readable.

**Explanation:** This is the prime directive of C programming style. Because C allows for extreme density and complex pointer/macro manipulations, a programmer must actively choose to write code that prioritizes human comprehension.

> Takeaway 9 #2: Short-term memory and the field of vision are small.

> Takeaway 9 #3: Coding style is not a question of taste but of culture.

> Takeaway 9 #4: Each established project constitutes its own cultural space.

_(Note: Takeaways 9#2 through 9#4 are extracted from the book's summary index to provide complete context on the author's philosophy regarding human cognitive limits and project culture)._

**Code Demonstration: Readable vs. Unreadable Code**

```c
#include <stdio.h>
#include <stdlib.h>

// Bad Style: Obfuscated, ignoring the human reader.
// What goes wrong if... you try to debug this? It's impossible to parse visually.
int f(int a,int b){return a>b?a:b;}

// Good Style: Prioritizing readability (Applying Takeaway 9 #1)
// Clear naming, proper spacing, and standard bracing.
int get_maximum_value(int value_a, int value_b) {
    if (value_a > value_b) {
        return value_a;
    }
    return value_b;
}

int main(void) {
    int max = get_maximum_value(5, 10);
    printf("Max is: %d\n", max);
    return EXIT_SUCCESS;
}
// Expected behavior/output:
// Max is: 10
```

**Recap:**

- Code instructs machines but is read by humans.
- Readability is the prime directive.
- Human cognitive limits necessitate stylistic rules.

---

## 9.1. Formatting

### Whitespace and Layout Strategies

**What it is:**
Formatting encompasses indentation, the placement of parentheses `()` and brackets `{} []`, spaces before and after operators, and the use of new lines. C compilers are largely agnostic to whitespace, meaning they will happily parse a whole program written on a single line.

**Why it matters / how it works:**
The human eye relies on visual cues (like aligned indentation and predictable bracket placement) to understand control flow and scope. Consistent formatting ensures that the brain does not waste cognitive cycles parsing text layout and can focus on the program's logic.

**Key details:**

- **Prefix notation for blocks:** The opening brace `{` is placed at the end of the line that initiates the block.
- **Left-binding of modifiers:** Type modifiers (`*`) and qualifiers (`const`) are bound visually to the left (to the type), separating them from the identifier (e.g., `char const* const path_name`).
- **Function vs. Condition parentheses:** Parentheses for function calls are bound to the left `func()`, whereas parentheses for conditions are separated by a space `if (condition)`.

**Pitfalls:**

- Mixing tabs and spaces can cause alignment to break depending on the viewer's text editor settings.
- Inconsistent placement of `*` in pointer declarations (e.g., `int *a, b;` where `b` is surprisingly just an `int`) causes bugs.

> Takeaway 9.1 #1: Choose a consistent strategy for white space and other text formatting.

**Explanation:** It matters less exactly _which_ strategy you choose (e.g., K&R style vs. Allman style bracing) than it does that you apply it uniformly across the entire codebase.

> Takeaway 9.1 #2: Have your text editor automatically format your code correctly.

**Explanation:** Formatting is tedious and error-prone for humans. Tools like `astyle`, `clang-format`, or editor-specific auto-formatters (like those in Emacs or modern IDEs) should handle this automatically.

**Code Demonstration: Formatting Rules Applied**

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

// Left-binding type modifiers: 'char' is qualified by 'const',
// made a pointer by '*', and that pointer is qualified by 'const'.
// It is visually separated from the identifier 'greeting'.
char const* const greeting = "Hello, Formatted World!";

int main(int argc, char* argv[argc+1]) { // { at the end of the line
    // Space after 'if', but no space before () in function calls like puts()
    if (argc > 1) {
        puts("Arguments provided.");
    } else {
        do {
            puts(greeting);
        } while (false); // condition separated by space
    }
    return EXIT_SUCCESS;
}
// Expected behavior/output:
// "Hello, Formatted World!" (if run without arguments).
// What goes wrong if... we don't bind modifiers cleanly?
// E.g., `int* a, b;` misleadingly makes people think `b` is a pointer, but it is an int.
```

**Recap:**

- Compilers ignore whitespace; humans depend on it.
- Adopt a strict, consistent layout style (e.g., left-binding pointers, prefix bracing).
- Automate formatting using editor tools or external formatters to maintain consistency easily.

---

## 9.2. Naming

### Technical Restrictions and Semantic Conventions

**What it is:**
Naming refers to the identifiers chosen for types, struct/union members, variables, macros, and functions. Naming operates under strict technical rules imposed by the C standard (to avoid collisions) and semantic guidelines chosen by the programmer (to convey meaning).

**Why it matters / how it works:**
C features a single global namespace for functions and variables, and specific namespaces for tags (structs/unions) and macros. If an identifier conflicts with a standard library name, the program might explode unpredictably upon updating the compiler or standard. Semantically, names must quickly communicate what an entity is and how it is used.

**Key details (Technical):**

- **Reserved Identifiers (C Standard):**
  - Names starting with an underscore and a capital letter (e.g., `_Atomic`), or two underscores (`__func__`), are reserved for compiler/language extensions.
  - Names starting with a single underscore are reserved for file scope identifiers and tags.
  - Identifiers ending in `_t` (e.g., `size_t`) are reserved for types.
  - Functions starting with `str`, `E`, `mem`, etc., are often reserved by the standard library.

**Key details (Semantic):**

- **Distinguishability:** Names must be easily recognizable and quickly distinguishable. `l11101l` is terrible; `lowBit` vs `highBit` is excellent.
- **Conventions:**
  - _CamelCase:_ `internalCapitalsToBreakWords`
  - _snake_case:_ `internal_underscores_to_break_words`
  - _Hungarian notation:_ Encoding types into names (e.g., `szName` for string zero-terminated) - generally frowned upon in modern C due to brittleness when types change.

**Pitfalls:**

- Polluting the global namespace means your code will inevitably break when linked with other libraries or newer C standards.
- Using single-letter identifiers for global state makes code unsearchable and unreadable.

> Takeaway 9.2 #1: Choose a consistent naming policy for all identifiers.

**Explanation:** Similar to formatting, having a predictable naming scheme (e.g., all functions in snake_case, all macros in UPPER_CASE) helps the reader immediately deduce the nature of the identifier.

> Takeaway 9.2 #2: Any identifier that is visible in a header file must be conforming.

**Explanation:** Header files are included in multiple translation units. If a header defines a non-conforming (e.g., reserved) identifier, it will corrupt the namespace of every file that includes it.

> Takeaway 9.2 #3: Don't pollute the global space of identifiers.

**Explanation:** Limit exposure. Use prefixes for public APIs (e.g., `pthread_create`, `p99_sum`) to minimize the chance of collision with user code or other libraries.

> Takeaway 9.2 #4: Names must be recognizable and quickly distinguishable.

**Explanation:** Avoid similar-looking names like `myLineNumber` vs `myLimeNumber`. Distinctive names prevent accidental misuse of variables.

> Takeaway 9.2 #5: Naming is a creative act.

**Explanation:** Technical rules only take you so far. Choosing names requires thoughtful design to capture the exact concept, state, or action without making the name overwhelmingly long.

> Takeaway 9.2 #6: File scope identifiers must be comprehensive.

**Explanation:** Globals lack local context, so their names must fully explain what they are (e.g., `application_window_width` instead of just `width`).

> Takeaway 9.2 #7: A type name identifies a concept.

**Explanation:** Types define models (e.g., `struct timespec` models time, `size_t` models sizes). If you can't name the type simply, your data structure design might be flawed.

> Takeaway 9.2 #8: A global constant identifies an artifact.

**Explanation:** Constants stand out because they represent universally true or externally fixed values (e.g., `M_PI` for Pi, `SIZE_MAX` for platform limits).

> Takeaway 9.2 #9: A global variable identifies state.

**Explanation:** Globals track system-wide conditions (e.g., `library_initialized_flag`, `active_user_count`).

> Takeaway 9.2 #A: A function or functional macro identifies an action.

**Explanation:** Functions _do_ things. Their names should generally contain verbs (`strcmp` for string compare, `matrix_multiply`, `get_flag`).

**Code Demonstration: Proper Naming Conventions**

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

// Takeaway 9.2 #8: Global constant identifies an artifact (UPPER_CASE for macros/constants)
#define MAX_BUFFER_CAPACITY 1024

// Takeaway 9.2 #9: Global variable identifies state.
// Uses a prefix 'app_' to avoid global space pollution (Takeaway 9.2 #3)
static bool app_is_initialized = false;

// Takeaway 9.2 #7: Type name identifies a concept.
// Avoids the reserved '_t' suffix for custom types in strictly conforming code,
// though POSIX often reserves it. The book suggests avoiding it to prevent standard clashes.
typedef struct NetworkPacket {
    size_t payload_size;
    // member names should be clear and descriptive
    unsigned char* payload_data;
} network_packet;

// Takeaway 9.2 #A: Function identifies an action (verb-based)
// Uses a clear prefix (app_)
void app_initialize_system(void) {
    if (!app_is_initialized) {
        puts("System initialized.");
        app_is_initialized = true;
    }
}

int main(void) {
    app_initialize_system();

    network_packet packet = {
        .payload_size = MAX_BUFFER_CAPACITY,
        .payload_data = NULL // In a real app, allocate memory here
    };

    printf("Packet capacity: %zu\n", packet.payload_size);
    return EXIT_SUCCESS;
}
// Expected behavior/output:
// System initialized.
// Packet capacity: 1024
// What goes wrong if... we name the type `network_packet_t`?
// A future C standard might introduce `network_packet_t`, causing a compilation error.
```

**Recap:**

- Strictly avoid reserved prefixes (`_`, `__`, `_[A-Z]`) and suffixes (`_t`).
- Prefix public API identifiers to prevent namespace pollution.
- Assign meaning logically: Types = Concepts, Constants = Artifacts, Globals = State, Functions = Actions.

---

## 9.3. Internationalization, so to speak

### Character Sets and Identifiers

**What it is:**
Internationalization in C source code refers to the ability to use non-English, extended character sets (like UTF-8) for strings, characters, and importantly in modern C (C23), for identifiers (variable names, function names).

**Why it matters / how it works:**
While English serves as a _lingua franca_ in tech, teams may need or prefer to use domain-specific mathematical symbols (like `Ω` for resistance) or their native language in identifiers. C23 adopts UAX #31 (Unicode Identifier and Pattern Syntax), allowing developers to use a broad spectrum of Unicode characters in identifiers natively, moving away from awkward `\u` Universal Character Names escapes.

**Key details:**

- **Normalization Form C:** Unicode allows characters to be constructed from multiple parts (e.g., 'e' + '´' = 'é'). C identifiers must use Normalization Form C (the precomposed, single-code-point version of the character) to ensure the compiler treats visually identical characters as the same identifier.
- **Distinctiveness:** Because Unicode contains many characters that look identical (e.g., Latin 'A', Greek 'Alpha' 'Α', Cyrillic 'А'), mixing them creates deceptive, unmaintainable code.
- **Mathematical/Technical symbols:** Allowed if they are classified as alphabetic but distinctly separate from standard natural language letters.

**Pitfalls:**

- Using visually indistinguishable characters (homoglyphs) to create hidden bugs (e.g., defining a variable with a Cyrillic 'a' and trying to access it with a Latin 'a' resulting in an "undeclared identifier" error).
- Using subscripts/superscripts in identifiers, which is allowed as a compiler extension in some environments but is strictly non-portable in standard C.

> Takeaway 9.3 #1: The natural language of a project should be chosen to accommodate the majority of the participants.

**Explanation:** Decide on the team's language (usually English for open-source, perhaps a local language for a regional company) and stick to it strictly for comments and identifiers.

> Takeaway 9.3 #2: Alphabetic letters are only allowed in identifiers if they map to themselves for Normalization Form C.

**Explanation:** To prevent the compiler from seeing two visually identical names as different variables due to underlying byte differences (composed vs precomposed characters), C requires Normalization Form C for all identifiers.

> Takeaway 9.3 #3: Only use alphabetic letters in identifiers if they originate directly from natural languages or they are clearly distinctive from all natural languages.

**Explanation:** Avoid obscure or ambiguous Unicode categories. Stick to clear, standard scripts (like standard Latin extended) or completely distinct mathematical symbols (like the Angstrom symbol `Å`).

> Takeaway 9.3 #4: Only use letters from different scripts or variations of decimal digits in identifiers if they are clearly distinctive from one another.

**Explanation:** If you use a mathematical bold digit zero (`𝟎`) alongside a standard zero (`0`), human readers will be unable to distinguish them quickly, leading to catastrophic logic errors.

> Takeaway 9.3 #5: Using subscript or superscript letters in identifiers is not portable.

**Explanation:** While a specific compiler (like GCC or Clang) might allow naming a variable `a₁`, it is not guaranteed by the C23 standard and will break if compiled on a different platform.

**Code Demonstration: Unicode in Identifiers (C23 Concept)**

```c
#include <stdio.h>
#include <stdlib.h>

// Takeaway 9.3 #3: Using distinctive alphabetic letters from natural languages or math
// Note: This requires a C23 compiler with UTF-8 source file support.
// We are using the Greek Omega symbol for resistance (valid UAX #31 identifier)
double calc_voltage(double current, double Ω_resistance) {
    return current * Ω_resistance;
}

int main(void) {
    // Takeaway 9.3 #2: 'année' must be saved in the source file in Normalization Form C.
    // If it was saved as 'anne\u0301e' (combining accent), standard C wouldn't allow it as an identifier.
    int année = 2026;

    double I = 2.5; // Amperes
    double R = 10.0; // Ohms
    double V = calc_voltage(I, R);

    printf("Year: %d | Voltage: %.2fV\n", année, V);
    return EXIT_SUCCESS;
}
// Expected behavior/output:
// Year: 2026 | Voltage: 25.00V
// What goes wrong if... we use Latin 'A' and Greek 'A' interchangeably?
// int A = 5; // Latin
// int Α = 10; // Greek
// The compiler sees two entirely different variables. The human sees identical letters,
// causing massive confusion.
```

**Recap:**

- C23 embraces UTF-8, allowing non-ASCII identifiers based on UAX #31.
- Always use Normalization Form C to ensure byte-level uniqueness matches visual uniqueness.
- Avoid homoglyphs (indistinguishable characters from different scripts).
- Avoid non-portable extensions like subscript identifiers.

---

## Summary

Chapter 9 addresses the subjective but critical aspect of C programming: Style. It establishes the "prime directive" that source code is a communication tool meant for human readers first and compilers second. To manage the cognitive load on programmers, the chapter advocates for strict, automated formatting rules (like prefix bracing and left-binding modifiers) and careful, disciplined naming conventions. It details how technical restrictions (avoiding reserved namespaces, global pollution) intersect with semantic clarity (using concepts for types, state for globals, and actions for functions). Finally, it explains modern C's handling of internationalization, explicitly outlining how C23 allows Unicode identifiers provided they adhere to Normalization Form C and do not introduce visual ambiguity.

---

## Self-Check Questions

1. **What is the dual purpose of a C program according to the text?**
   It provides instructions for the compiler to build an executable, and it serves as documentation of the system's intended behavior for humans.

2. **Why does C require a strict strategy for whitespace formatting if the compiler ignores it?**
   Because human short-term memory and visual fields are limited; inconsistent whitespace causes cognitive strain and obscures control flow.

3. **What does "left-binding" mean in the context of pointer declarations?**
   It means attaching the `*` visually to the type on the left (e.g., `int* p;`) rather than the identifier on the right, clarifying that the type itself is a pointer.

4. **Why is it dangerous to use identifiers that end in `_t` in your own code?**
   The C standard (and POSIX) reserves identifiers ending in `_t` for future standard types. Using them can cause conflicts when upgrading compilers or standards.

5. **How should you name a variable that represents global state versus a function?**
   A global variable should identify its state (e.g., `is_initialized`), while a function should identify an action, usually incorporating a verb (e.g., `initialize_system`).

6. **What is "namespace pollution" and how do you avoid it in public APIs?**
   It's exposing too many generic names in the global scope. You avoid it by using distinct prefixes (like `pthread_`) for all public functions and types.

7. **Are macros formatted differently than standard variables?**
   Yes, macro names should be written in ALL CAPS to visually distinguish them as textual replacements rather than standard variables.

8. **What is Normalization Form C, and why is it required for C identifiers?**
   It is a Unicode encoding rule that uses single precomposed characters instead of combining characters. It ensures that visually identical characters resolve to the same underlying byte sequence for the compiler.

9. **Why is using Cyrillic 'a' and Latin 'a' in the same codebase considered a pitfall?**
   Because they are homoglyphs—they look identical to the human reader but have different Unicode values, meaning the compiler treats them as distinct variables, leading to invisible bugs.

10. **Is it portable to use subscript characters (like `x₁`) in C identifiers?**
    No, while some specific compilers may allow it as an extension, it is not portable standard C.
