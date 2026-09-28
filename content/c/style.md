+++
title = "Modern C: Style"
description = ""
date = 2026-09-28

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

#### Semantic Naming Rules
* **Recognizability:** **Takeaway 9.2 #4** states ***Names must be recognizable and quickly distinguishable***. Single-letter loop variables (`i`, `n`) are fine in tight, visible scopes, but broader symbols require clarity.
* **Pragmatic Creativity:** **Takeaway 9.2 #5** notes ***Naming is a creative act***. Standard conventions like CamelCase, Snake_case, or Hungarian notation have trade-offs, so readability in context is key.
* **Scope & Entity Roles:**
  * **Takeaway 9.2 #6:** ***File scope identifiers must be comprehensive***.
  * **Takeaway 9.2 #7:** ***A type name identifies a concept*** (e.g., `timespec` \\(\rightarrow\\) time, `person` \\(\rightarrow\\) individual data structure).
  * **Takeaway 9.2 #8:** ***A global constant identifies an artifact*** (e.g., `M_PI`, `SIZE_MAX`, `false`).
  * **Takeaway 9.2 #9:** ***A global variable identifies state*** (e.g., `toto_initialized`).
  * **Takeaway 9.2 #A:** ***A function or functional macro identifies an action***, typically incorporating a verb (e.g., `strcmp`, `getFlag`, `matrixMult`).


### 4. Internationalization & Unicode Identifiers (Section 9.3)
C23 standardizes extended character set support in source code while providing rules to prevent obfuscation.
* **Project Language:** **Takeaway 9.3 #1** states ***The natural language of a project should be chosen to accommodate the majority of the participants***.
* **Unicode Normalization:**
  * **Takeaway 9.3 #2:** ***Alphabetic letters are only allowed in identifiers if they map to themselves for Normalization Form C***.
  * **Takeaway 9.3 #3:** ***Only use alphabetic letters in identifiers if they originate directly from natural languages or they are clearly distinctive from all natural languages***.
  * **Takeaway 9.3 #4:** ***Only use letters from different scripts or variations of decimal digits in identifiers if they are clearly distinctive from one another*** (preventing visual confusion between lookalike characters across scripts).
  * **Takeaway 9.3 #5:** ***Using subscript or superscript letters in identifiers is not portable***.
