+++
title = "Modern C: Variations in control flow"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 19: Variations in control flow** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book explores how execution moves beyond simple sequential statement lists and basic block structures. It examines non-local jumps, signal handlers, evaluation order guarantees, and the interaction between unusual control flow and compiler optimizations.

Here is a detailed breakdown of the fundamental concepts, execution mechanisms, and key takeaways from Chapter 19 across its six core sections:


### 1. Basic Blocks & Control Flow Overview (Intro & Section 19.1)
Normally, C program execution can be analyzed in terms of **basic blocks**—maximal sequences of statements executed unconditionally from first to last once entered. Structured programming organizes basic blocks hierarchically using conditionals (`if/else`, `switch/case`) and loops (`for`, `while`, `do`). However, exceptional conditions or external events require non-hierarchical control flow mechanisms:

* **Language-level constructs:** Conditional statements, loops, function calls/returns, and short jumps (`goto`).
* **Library-level mechanisms:** Long jumps across functions (`setjmp`/`longjmp`), signal interrupts (`signal`/`raise`), and concurrent thread execution (`thrd_create`/`thrd_exit`).
* **State Risks:** Non-standard control flow can cause objects to be accessed outside their lifetime, used uninitialized, modified partially, or misinterpreted due to compiler optimization assumptions.


### 2. Sequencing & Sequence Points (Section 19.2)
C code execution is formally structured around **sequence points** and **sequencing relations**. Code between sequence points may be reordered or interleaved by the compiler as long as observable side effects match the abstract machine.

#### Sequence Points Definition
A sequence point is a synchronization barrier in execution. Major sequence points occur at:
* The end of a statement (terminated by `;` or `}`).
* The end of an expression before the comma operator `,`.
* The end of a declaration.
* The evaluation of controlling expressions in `if`, `switch`, `for`, `while`, `?:`, `&&`, and `||`.
* Immediately after evaluating function designators and function arguments, before entering the function body.
* The end of a `return` statement.
* Specific library boundaries (e.g., format specifier actions in I/O, before library returns, and before/after comparison callbacks in `qsort`/`bsearch`).

#### Sequencing Rules & Indeterminate Evaluation
* **Takeaway 19.2 #1:** ***Side effects in functions can lead to indeterminate results***. Function arguments are **indeterminately sequenced**—they are executed in some order, but calling functions with side effects as arguments (e.g., `printf("%u, %u\n", add(&a, &b), add(&b, &a))`) yields platform-dependent results.
* **Takeaway 19.2 #2:** ***The specific operation of any operator is sequenced after the evaluation of all its operands***.
* **Takeaway 19.2 #3:** ***The effect of updating an object with any of the assignment, increment, or decrement operators is sequenced after the evaluation of its operands***.
* **Takeaway 19.2 #4:** ***A function call is sequenced with respect to all evaluations of the caller***.
* **Takeaway 19.2 #5:** ***Initialization list expressions for array or structure types are indeterminately sequenced***.


### 3. Short Jumps & Object Lifetimes (Section 19.3)
Short jumps are implemented using **`goto`** and target **labels** within the same function body.

* **Takeaway 19.3 #1:** ***Each iteration defines a new instance of a local object***. Entering a loop body creates a fresh instance of automatic variables or compound literals, whereas repeating code using a `goto RETRY:` label retains the existing instance's lifetime.
* **Takeaway 19.3 #2:** ***`goto` should only be used for exceptional changes in control flow***. Its primary legitimate uses are centralized error cleanup (jumping to a single `CLEANUP:` label) or state-machine parsing algorithms.


### 4. Functions & Recursion (Section 19.4)
Function calls suspend caller execution and create isolated frame contexts.

* **Takeaway 19.4 #1:** ***Each function call defines a new instance of a local object***. Recursive invocations operate on independent copies of local variables on the call stack. Modifying shared state via pointers across recursive calls weakens this isolation and complicates compiler optimization.


### 5. Long Jumps: `setjmp` and `longjmp` (Section 19.5)
The `<setjmp.h>` header provides a mechanism to restore execution to an earlier calling context across arbitrary nested function calls, bypassing regular `return` cascades.

#### Mechanics of `setjmp` and `longjmp`
* `int setjmp(jmp_buf target)`: Saves the execution environment (stack pointer, instruction pointer, registers) into `target`.
* `[[noreturn]] void longjmp(jmp_buf target, int condition)`: Restores the saved context and transfers control back to `setjmp`.

#### Key Rules & Takeaways
* **Takeaway 19.5 #1:** ***`longjmp` never returns to the caller***.
* **Takeaway 19.5 #2:** ***When reached through normal control flow, a call to `setjmp` marks the call location as a jump target and returns 0***.
* **Takeaway 19.5 #3:** ***Leaving the scope of a call to `setjmp` invalidates the jump target***. Calling `longjmp` on a `jmp_buf` after its defining function has returned causes undefined behavior.
* **Takeaway 19.5 #4:** ***A call to `longjmp` transfers control directly to the position set by `setjmp` as if that had returned the condition argument***.
* **Takeaway 19.5 #5:** ***A 0 as a condition parameter to `longjmp` is replaced by 1***. This ensures `setjmp` never returns `0` upon a long jump.
* **Takeaway 19.5 #6:** ***`setjmp` may be used only in simple comparisons inside controlling expressions of conditionals***. It cannot be assigned to variables in complex expressions.
* **Takeaway 19.5 #7:** ***Optimization interacts badly with calls to `setjmp`***.
* **Takeaway 19.5 #8:** ***Objects modified across `longjmp` must be `volatile`***. Compilers assume local variables in `setjmp`'s frame are unmodified unless qualified with `volatile`.
* **Takeaway 19.5 #9 & #A:** ***`volatile` objects are reloaded from memory each time they are accessed*** and ***stored each time they are modified***.
* **Takeaway 19.5 #B:** ***The `typedef` for `jmp_buf` hides an array type***. Passing `jmp_buf` to functions automatically rewrites to a pointer, creating implicit pass-by-reference behavior.


### 6. Signal Handlers & Asynchronous Control Flow (Section 19.6)
Signals handle hardware interrupts (traps/synchronous signals like `SIGSEGV` or `SIGFPE`) or software events (asynchronous signals like `SIGINT` or `SIGTERM`).

#### Execution Model & Restrictions
* **Takeaway 19.6 #1:** ***C’s signal-handling interface is minimal and should only be used for elementary situations***.
* **Takeaway 19.6 #2:** ***Signal handlers can kick in at any point of execution***.
* **Takeaway 19.6 #3:** ***After return from a signal handler, execution resumes exactly where it was interrupted***.
* **Takeaway 19.6 #4:** ***Any C statement may correspond to several processor instructions***. An interrupt occurring mid-statement can leave variables in partial/zombie states.
* **Takeaway 19.6 #5:** ***Signal handlers need types with uninterruptible operations***.
* **Takeaway 19.6 #6:** ***Objects of type `sig_atomic_t` should not be used as counters***. Read-modify-write operations (like `++`) can be split across machine instructions. Safe cross-handler flags should use `volatile sig_atomic_t` or lock-free atomic types (`atomic_flag`, `_Atomic`).
* **Takeaway 19.6 #7:** ***Unless specified otherwise, C library functions are not asynchronous signal safe***. Signal handlers must avoid I/O (like `printf`), dynamic memory (`malloc`/`free`), or `exit()`; they may only call `_Exit()`, `quick_exit()`, `signal()`, or atomic operations.


### Summary Table of Chapter 19 Takeaways

| Takeaway ID | Rule Description |
| :--- | :--- |
| **19.2 #1** | *Side effects in functions can lead to indeterminate results.* |
| **19.3 #1** | *Each iteration defines a new instance of a local object.* |
| **19.3 #2** | *`goto` should only be used for exceptional changes in control flow.* |
| **19.4 #1** | *Each function call defines a new instance of a local object.* |
| **19.5 #1** | *`longjmp` never returns to the caller.* |
| **19.5 #2** | *When reached through normal control flow, a call to `setjmp` marks the call location as a jump target and returns 0.* |
| **19.5 #3** | *Leaving the scope of a call to `setjmp` invalidates the jump target.* |
| **19.5 #8** | *Objects modified across `longjmp` must be `volatile`.* |
| **19.6 #2** | *Signal handlers can kick in at any point of execution.* |
| **19.6 #6** | *Objects of type `sig_atomic_t` should not be used as counters.* |
| **19.6 #7** | *Unless specified otherwise, C library functions are not asynchronous signal safe.* |

