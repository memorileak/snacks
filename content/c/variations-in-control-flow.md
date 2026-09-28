+++
title = "Modern C: Variations in control flow"
description = "A guide to C23 control flow beyond ordinary sequencing, covering sequence points, goto, recursion, setjmp and longjmp, and asynchronous signal handlers."
date = 2026-09-28T07:24:28Z

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

#### Example

Execution between **sequence points** can be reordered or interleaved by the compiler. Function argument evaluations are **indeterminately sequenced**; if function calls produce side effects on shared variables, the outcome depends on the compiler's evaluation order.

* **Takeaway 19.2 #1:** *Side effects in functions can lead to indeterminate results*.
* **Takeaway 19.2 #2 & #3:** *Operator actions and assignment/increment updates are sequenced after the evaluation of their operands*.
* **Takeaway 19.2 #4:** *A function call is sequenced with respect to all evaluations of the caller*.
* **Takeaway 19.2 #5:** *Initialization list expressions for array or structure types are indeterminately sequenced*.

```c
#include <stdio.h>

// Function with side effects on the target pointed to by x
static unsigned add_and_update(unsigned* x, unsigned const* y) {
    return *x += *y; // Takeaway 19.2 #3: Assignment/increment updates are sequenced after operand evaluation
}

void demo_sequencing_and_side_effects(void) {
    unsigned a = 3;
    unsigned b = 5;

    // Takeaway 19.2 #1 & #4: Function calls are sequenced relative to the caller,
    // BUT the two argument expressions below are INDETERMINATELY SEQUENCED.
    // Execution will either evaluate add_and_update(&a, &b) first (a=8, b=13)
    // OR add_and_update(&b, &a) first (b=8, a=11) depending on the compiler.
    printf("a = %u, b = %u\n", add_and_update(&a, &b), add_and_update(&b, &a));

    // Takeaway 19.2 #5: Elements in compound initializers are also indeterminately sequenced.
    unsigned x = 1;
    unsigned arr = { x++, x++ }; // Order of evaluation for array initializers is indeterminate
    printf("arr: {%u, %u}\n", arr, arr);
}
```

### 3. Short Jumps & Object Lifetimes (Section 19.3)
Short jumps are implemented using **`goto`** and target **labels** within the same function body.

* **Takeaway 19.3 #1:** ***Each iteration defines a new instance of a local object***. Entering a loop body creates a fresh instance of automatic variables or compound literals, whereas repeating code using a `goto RETRY:` label retains the existing instance's lifetime.
* **Takeaway 19.3 #2:** ***`goto` should only be used for exceptional changes in control flow***. Its primary legitimate uses are centralized error cleanup (jumping to a single `CLEANUP:` label) or state-machine parsing algorithms.

#### Example

Short jumps transfer control within the same function body using labels. A critical distinction exists between repeating code via a loop versus jumping back with `goto`: entering a loop block creates a **new instance and lifetime** for local objects, whereas `goto` retains the existing instance.

* **Takeaway 19.3 #1:** *Each iteration defines a new instance of a local object*.
* **Takeaway 19.3 #2:** *`goto` should only be used for exceptional changes in control flow*.

```c
#include <stdio.h>
#include <stdbool.h>

static int get_next_val(void) {
    static int val = 0;
    return ++val;
}

void demo_short_jumps_and_lifetimes(void) {
    size_t* ip1 = nullptr;

    // 1. LOOP ITERATION LIFETIME:
    // Takeaway 19.3 #1: Each iteration of while defines a NEW instance of the compound literal.
    size_t iterations = 0;
    while (iterations < 1) {
        ip1 = &(size_t){ (size_t)get_next_val() }; // Lifetime of this compound literal ends at while '}'
        ++iterations;
    }
    // Dereferencing ip1 here is invalid because the object's lifetime has ended!

    // 2. GOTO REPEAT LIFETIME:
    // Jumping to a label does NOT create a new block scope, so the compound literal stays alive.
    size_t* ip2 = nullptr;
    bool retry = true;

RETRY_LABEL:
    ip2 = &(size_t){ (size_t)get_next_val() }; // Life continues across goto jump
    if (retry) {
        retry = false;
        // Takeaway 19.3 #2: Use goto primarily for exceptional conditions or specific state-machine parsing
        goto RETRY_LABEL;
    }

    // ip2 is still valid here because the definition block was not exited
    printf("Value via goto object: %zu\n", *ip2);
}
```

### 4. Functions & Recursion (Section 19.4)
Function calls suspend caller execution and create isolated frame contexts.

* **Takeaway 19.4 #1:** ***Each function call defines a new instance of a local object***. Recursive invocations operate on independent copies of local variables on the call stack. Modifying shared state via pointers across recursive calls weakens this isolation and complicates compiler optimization.

#### Example

When a function recurses, each call operates on its own independent instance of local automatic variables.

* **Takeaway 19.4 #1:** *Each function call defines a new instance of a local object*.

```c
#include <stdio.h>

static void recursive_depth_counter(unsigned current_level, unsigned max_level) {
    // Takeaway 19.4 #1: Every recursive invocation creates a distinct instance of 'local_frame_id'
    unsigned local_frame_id = current_level;

    printf("Entering recursion level %u (Frame ID: %u)\n", current_level, local_frame_id);

    if (current_level < max_level) {
        recursive_depth_counter(current_level + 1, max_level);
    }

    // When returning, local_frame_id retains the exact value of this call's frame instance
    printf("Unwinding recursion level %u (Frame ID: %u)\n", current_level, local_frame_id);
}

void demo_function_recursion(void) {
    recursive_depth_counter(1, 3);
}
```

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

#### Example

`<setjmp.h>` allows transferring execution across arbitrary nested function frames directly back to a saved calling context. Because optimization assumes local variables in `setjmp`'s frame are unmodified across the jump, **any variable modified between `setjmp` and `longjmp` must be qualified as `volatile`**.

* **Takeaway 19.5 #1:** *`longjmp` never returns to the caller*.
* **Takeaway 19.5 #2 & #4:** *`setjmp` returns 0 when reached normally, and returns the condition value when reached via `longjmp`*.
* **Takeaway 19.5 #3:** *Leaving the scope of a call to `setjmp` invalidates the jump target*.
* **Takeaway 19.5 #5:** *A 0 passed as a condition parameter to `longjmp` is replaced by 1*.
* **Takeaway 19.5 #6:** *`setjmp` may be used only in simple comparisons inside controlling expressions of conditionals*.
* **Takeaway 19.5 #7 & #8:** *Optimization interacts badly with `setjmp`; objects modified across `longjmp` must be `volatile`*.
* **Takeaway 19.5 #9 & #A:** *`volatile` objects are reloaded from memory on read and stored on write*.
* **Takeaway 19.5 #B:** *The `typedef` for `jmp_buf` hides an array type* (creating implicit pass-by-reference semantics).

```c
#include <stdio.h>
#include <setjmp.h>

enum parse_status {
    STATUS_OK = 0,
    STATUS_TOO_DEEP,
    STATUS_INVALID_SYNTAX
};

// Functions calling longjmp must receive jmp_buf (which decays to a pointer)
static void parse_nested_element(jmp_buf target_buf, unsigned level) {
    if (level > 5) {
        // Takeaway 19.5 #1: longjmp never returns to its caller.
        // Takeaway 19.5 #4: Returns control back to setjmp with the condition argument.
        longjmp(target_buf, STATUS_TOO_DEEP);
    }
}

void demo_long_jumps(void) {
    jmp_buf jump_target;

    // Takeaway 19.5 #8: Variables modified between setjmp and longjmp MUST be volatile!
    // Non-volatile variables modified in caller frames will be improperly cached by compiler optimizations.
    unsigned volatile nesting_depth = 0;

    // Takeaway 19.5 #6: setjmp may ONLY be called inside simple conditional controlling expressions!
    // Takeaway 19.5 #2: Returns 0 during normal execution flow when setting up the jump target.
    switch (setjmp(jump_target)) {
        case STATUS_OK:
            nesting_depth = 10; // Modifying volatile variable
            parse_nested_element(jump_target, nesting_depth);
            puts("Parsing completed successfully.");
            break;

        case STATUS_TOO_DEEP:
            // Takeaway 19.5 #9 & #A: Volatile guarantees we read the updated value (10) from memory
            printf("Error recovery: Nesting level %u was too deep!\n", nesting_depth);
            break;

        default:
            puts("Unknown error condition encountered.");
            break;
    }

    // Takeaway 19.5 #3: Returning from this function invalidates jump_target!
}
```

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

#### Example

Signal handlers execute asynchronously in response to hardware traps or software events. Because signals can interrupt statement execution mid-instruction, **signal handlers must only modify atomic objects or `volatile sig_atomic_t` flags**, and must refrain from calling non-reentrant C library functions (like `printf` or `malloc`).

* **Takeaway 19.6 #1:** *C's signal-handling interface is minimal and should only be used for elementary situations*.
* **Takeaway 19.6 #2 & #3:** *Signal handlers can kick in at any point; execution resumes where interrupted*.
* **Takeaway 19.6 #4 & #5:** *Statements correspond to multiple CPU instructions; signal handlers need types with uninterruptible operations*.
* **Takeaway 19.6 #6:** *Objects of type `sig_atomic_t` should not be used as counters* (read-modify-write operations like `++` are not uninterruptible).
* **Takeaway 19.6 #7:** *Unless specified otherwise, C library functions are NOT asynchronous-signal-safe*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

// Takeaway 19.6 #5: Global flags modified by signal handlers MUST be volatile sig_atomic_t
static volatile sig_atomic_t global_interrupt_flag = 0;

static void minimal_signal_handler(int sig) {
    // Takeaway 19.6 #6: sig_atomic_t objects should NOT be used as counters (avoid ++)!
    // Simple assignment = is guaranteed to be an uninterruptible store operation.
    global_interrupt_flag = sig;

    // Takeaway 19.6 #7: C library functions (printf, malloc, exit) are NOT async-signal-safe!
    // Handlers should do as little work as possible and return immediately.
    // Allowed safe terminations: quick_exit(), _Exit(), or raise().
    signal(sig, SIG_DFL); // Reset signal disposition to default
}

void demo_signal_handlers(void) {
    // Takeaway 19.6 #1: Register custom handler using signal()
    if (signal(SIGINT, minimal_signal_handler) == SIG_ERR) { // Interactive interrupt (Ctrl+C)
        fputs("Failed to install SIGINT signal handler.\n", stderr);
        return;
    }

    printf("Simulating work... Raising SIGINT signal.\n");

    // Takeaway 19.6 #2: Signal handler kicks in asynchronously
    raise(SIGINT);

    // Takeaway 19.6 #3: After returning from signal handler, execution resumes right here
    if (global_interrupt_flag != 0) {
        printf("Main thread captured interrupt flag: %d\n", global_interrupt_flag);
    }
}
```

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

