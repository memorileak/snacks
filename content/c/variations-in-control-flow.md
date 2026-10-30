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

## 19. Variations in control flow

### Overview of Control Flow Variations

**What it is:** Control flow dictates the order in which individual statements in a C program are executed. While standard code executes sequentially or relies on structured conditionals and loops, exceptional situations require exceptional measures that drastically alter this flow.

**Why it matters / how it works:** Understanding standard sequence execution (basic blocks) is the foundation of predicting a program's state. When this sequence is interrupted by features like `goto`, `longjmp`, signals, or threads, the abstract state machine becomes difficult to track.

**Key details:**

- **Basic Block:** A maximal sequence of statements executed unconditionally from top to bottom.
- **Language features:** Standard conditional statements (`if/else`, `switch/case`), loop statements (`do{}while()`, `while()`, `for()`), function calls, and short jumps (`goto`).
- **C library interfaces:** Long jumps (`setjmp`/`longjmp`), interrupts (signal handlers), and threads (`thrd_create`).

**Pitfalls:** Unexpected control flows can lead to objects being used outside their lifetimes, uninitialized objects, misinterpretations by optimizers, and race conditions (especially with unsequenced updates).

**Connections:** Links back to Chapter 2 (control flow basics) and forward to Chapter 20 (threads) and Chapter 21 (atomic data).

### Recap

- Control flow can be simple (loops, conditionals) or complex (signals, jumps, threads).
- Complexity increases as we move from language syntaxes to C library features.
- Managing state consistency is the primary challenge in non-linear control flow.

## 19.1. A detailed example

### The `descend` Function Case Study

**What it is:** The chapter introduces a recursive descent parser (a function named `descend`) to parse basic blocks and code indentation.

**Why it matters / how it works:** This 36-line example packs in multiple complex control flow variations: short jumps (`goto`), recursion, long jumps (`longjmp`), and asynchronous interrupt handling. It demonstrates how these constructs can be used in tandem to manage deeply nested state and handle critical failures without cascading return values.

**Key details:**

- Uses `goto` for state transitions (`NEW_LINE`, `ASCEND`).
- Recursively calls itself to handle nested scopes (e.g., encountering `{`).
- Relies on `longjmp` for fatal, unrecoverable errors (e.g., nesting depth exceeded).
- Checks a global interrupt flag to safely terminate execution if an external signal is caught.

**Pitfalls:** This density of control flow transitions is hard to read and optimize. If pointer parameters or global variables aren't handled meticulously, state corruptions occur.

**Connections:** Serves as the running example for sections 19.3, 19.4, 19.5, and 19.6.

### Recap

- Complex C code often combines multiple control flow strategies.
- The recursive descent parser serves as a masterclass in using `goto`, recursion, and `longjmp` effectively.

---

## 19.2. Sequencing

### Evaluation Order and Sequence Points

**What it is:** In C, expressions are not always evaluated exactly in the order they are written from left to right. Sequence points are specific points in the program execution where all side effects of previous evaluations are guaranteed to be complete.

**Why it matters / how it works:** The compiler has the freedom to reorder, interleave, or delay the evaluation of expressions to optimize execution, provided it respects sequence points. If code modifies the same object multiple times between sequence points, the behavior is undefined or indeterminate.

**Key details:**

- Sequence points occur at the end of a statement (`;`), before a comma operator (`,`), at the end of a declaration, and at the end of controlling expressions (`if`, `switch`, `for`, `while`).
- Short-circuit evaluation (`&&`, `||`, `?:`) introduces sequence points.
- Function calls introduce a sequence point _after_ the evaluation of the function designator and arguments, but _before_ the body executes.

**Pitfalls:** Assuming left-to-right evaluation of function arguments is a major pitfall.

**Connections:** Relates to Chapter 4 (expressions) where evaluation order was first introduced.

> Takeaway 19.2 #1 _Side effects in functions can lead to indeterminate results._

**Explanation:** If a function mutates an object, and that function is called multiple times within an unsequenced context (like arguments to another function call), the order of those mutations is unpredictable. The C standard calls this "indeterminately sequenced".

```c
// Demonstrating Takeaway 19.2 #1
#include <stdio.h>
#include <stdlib.h>

// Function with a side effect on *x
unsigned add(unsigned* x, unsigned const* y) {
    return *x += *y;
}

int main(void) {
    unsigned a = 3;
    unsigned b = 5;

    // The evaluation order of add(&a, &b) and add(&b, &a) is not fixed.
    // They are indeterminately sequenced.
    printf("a = %u, b = %u\n", add(&a, &b), add(&b, &a));
    return EXIT_SUCCESS;
}
/* Expected behavior/output:
   It may print "a = 8, b = 13" OR "a = 11, b = 8" depending on the compiler.
   Both are standard-compliant.
   What goes wrong if... you rely on this ordering for critical logic? Your program
   will have hidden bugs that only appear when switching compilers or optimization flags. */
```

> Takeaway 19.2 #2 _The specific operation of any operator is sequenced after the evaluation of all its operands._

**Explanation:** Before an operator (like `+`, `*`, or `==`) does its actual computation, the CPU must first fully evaluate everything on its left and right sides.

> Takeaway 19.2 #3 _The effect of updating an object with any of the assignment, increment, or decrement operators is sequenced after the evaluation of its operands._

**Explanation:** When doing `x = y + z`, `y` and `z` are fully evaluated, the addition happens, and _then_ the assignment to `x` takes place.

> Takeaway 19.2 #4 _A function call is sequenced with respect to all evaluations of the caller._

**Explanation:** Once a function's arguments are fully evaluated, the jump into the function happens. The entire body of the function executes and completes before the caller resumes evaluating the rest of its expression.

> Takeaway 19.2 #5 _Initialization list expressions for array or structure types are indeterminately sequenced._

**Explanation:** When you write `int arr[2] = {f(), g()};`, the compiler can call `g()` before `f()` or vice-versa.

```c
// Demonstrating Takeaways 19.2 #4 and #5
#include <stdio.h>

int call_first(void) {
    printf("1 ");
    return 1;
}
int call_second(void) {
    printf("2 ");
    return 2;
}

int main(void) {
    // Array initialization: Indeterminately sequenced!
    // It might print "1 2 " or "2 1 "
    int arr[] = { call_first(), call_second() };
    return 0;
}
```

### Recap

- C does not guarantee strict left-to-right evaluation.
- Sequence points define the boundaries where side-effects must settle.
- Never depend on the evaluation order of function arguments or initializer lists if they contain side effects.

---

## 19.3. Short jumps

### `goto` and Labels

**What it is:** `goto` is a jump statement that transfers control unconditionally to a specified label within the same function.

**Why it matters / how it works:** It allows breaking out of deeply nested loops or establishing a single exit/cleanup point in a function.

**Key details:**

- Labels have function scope (they are visible everywhere inside the function).
- Jumping out of a block is safe, but jumping _into_ a block bypassing object initialization can lead to undefined behavior or logical errors.

**Pitfalls:** Overusing `goto` results in "spaghetti code", making it impossible to track state. Jumping past a variable length array (VLA) declaration is invalid.

> Takeaway 19.3 #1 _Each iteration defines a new instance of a local object._

**Explanation:** If you define a block-scoped variable in a loop (like `for`), each time the loop restarts, it's a conceptually new object. However, if you use `goto` to jump _backwards_ into the middle of a block without exiting it, the object might not be re-initialized.

> Takeaway 19.3 #2 _goto should only be used for exceptional changes in control flow._

**Explanation:** Use it for error handling (jumping to cleanup code at the end of a function) or breaking out of a nested loop where `break` would only exit one level.

```c
// Demonstrating Takeaway 19.3 #1 and #2
#include <stdio.h>

int main(void) {
    int attempts = 0;

RETRY: // A label for our short jump
    attempts++;
    int loop_scoped_var = attempts; // Re-assigned upon jumping back

    printf("Attempt %d\n", loop_scoped_var);

    if (attempts < 3) {
        // Exceptional control flow: retry logic
        goto RETRY;
    }

    // What goes wrong if we jump over initialization?
    // goto BAD_JUMP;
    // int uninitialized_var = 42;
    // BAD_JUMP:
    // printf("%d", uninitialized_var); // Undefined behavior!

    return 0;
}
/* Expected behavior/output:
   Attempt 1
   Attempt 2
   Attempt 3
*/
```

### Recap

- `goto` provides function-scoped unconditional jumps.
- Use `goto` sparingly, primarily for error cleanup.
- Be cautious of how jumps interact with the lifetimes and initializations of local variables.

---

## 19.4. Functions

### Recursion and Local State

**What it is:** Functions calling themselves directly or indirectly.

**Why it matters / how it works:** Recursion is a natural fit for parsing nested structures. The hardware implements this using the call stack, pushing a new frame with local variables for each call.

**Key details:**

- A pointer passed through recursive layers modifies the single underlying object.

**Pitfalls:** Stack overflow if recursion goes too deep. Pointer aliasing and modification hiding optimization opportunities.

> Takeaway 19.4 #1 _Each function call defines a new instance of a local object._

**Explanation:** Every time a function is called recursively, it gets its own, fresh copy of local variables (unless they are `static`). These instances do not interact unless addresses are passed explicitly.

```c
// Demonstrating Takeaway 19.4 #1
#include <stdio.h>

void recursive_count(int depth) {
    // Each call gets a unique instance of 'local_val'
    int local_val = depth;

    if (depth > 0) {
        recursive_count(depth - 1);
    }
    // As the stack unwinds, each instance retains its distinct value
    printf("%d ", local_val);
}

int main(void) {
    recursive_count(3);
    return 0;
}
/* Expected behavior/output:
   0 1 2 3
   What goes wrong if... 'local_val' was declared 'static'?
   It would be shared across all calls, printing "0 0 0 0".
*/
```

### Recap

- Function calls (including recursive ones) encapsulate state.
- Pointers passed down the call chain manipulate shared state, requiring careful tracking.

## 19.5. Long jumps

### Cross-Function Exceptional Control Flow (`setjmp` / `longjmp`)

**What it is:** The `<setjmp.h>` header provides a mechanism to save the current execution context (CPU registers, stack pointer) and jump back to it later from a deeply nested function call, bypassing normal `return` paths.

**Why it matters / how it works:** This is C's rudimentary form of exception handling. `setjmp` sets the "catch" point. `longjmp` "throws" an exception back to that point.

**Key details:**

- `jmp_buf`: An opaque array type used to store the context.
- `setjmp(buf)`: Saves context. Returns 0 when initially called directly.
- `longjmp(buf, val)`: Restores context. Causes the matching `setjmp` to appear as if it returned `val`.

**Pitfalls:** Memory leaks (if you jump over `free()` calls), un-restored local variables (due to optimizations), and jumping to a dead context.

> Takeaway 19.5 #1 _longjmp never returns to the caller._

**Explanation:** `longjmp` behaves like a one-way ticket. The function that calls `longjmp` is immediately terminated, and execution resumes at the `setjmp` point. It has the `[[noreturn]]` attribute.

> Takeaway 19.5 #2 _When reached through normal control flow, a call to setjmp marks the call location as a jump target and returns 0._

**Explanation:** The first time `setjmp` is executed, it takes a snapshot and yields `0`. This is how you differentiate between the "setup" phase and the "jumped-back" phase.

> Takeaway 19.5 #3 _Leaving the scope of a call to setjmp invalidates the jump target._

**Explanation:** If the function containing `setjmp` returns, the stack frame is destroyed. Calling `longjmp` with that `jmp_buf` afterward results in undefined behavior (usually a severe crash).

> Takeaway 19.5 #4 _A call to longjmp transfers control directly to the position set by setjmp as if that had returned the condition argument._

**Explanation:** If you call `longjmp(target, 5)`, the program teleports back to the `setjmp(target)` location, and `setjmp` now evaluates to `5`.

> Takeaway 19.5 #5 _A 0 as a condition parameter to longjmp is replaced by 1._

**Explanation:** Because `setjmp` returning `0` signifies the initial setup, `longjmp(buf, 0)` is forbidden. If you try it, the C standard mandates it is converted to `1` so your program doesn't infinitely loop thinking it's setting up again.

> Takeaway 19.5 #6 _setjmp may be used only in simple comparisons inside controlling expression of conditionals._

**Explanation:** You cannot use `setjmp` in complex arithmetic like `a = setjmp(buf) + 5;`. It must be the entire controlling expression, like `if (setjmp(buf) == 0)` or `switch (setjmp(buf))`.

> Takeaway 19.5 #7 _Optimization interacts badly with calls to setjmp._

**Explanation:** Compilers aggressively cache local variables in CPU registers. `setjmp` only saves the registers _at that exact moment_. If a local variable is changed after `setjmp`, and then `longjmp` occurs, the compiler might restore the old, cached value instead of the modified value.

> Takeaway 19.5 #8 _Objects modified across longjmp must be volatile._

**Explanation:** To prevent the optimizer from caching a variable in a register and losing its updated state during a `longjmp`, you must qualify the variable with `volatile`.

> Takeaway 19.5 #9 _volatile objects are reloaded from memory each time they are accessed._

**Explanation:** The compiler is forbidden from caching a `volatile` object. It must fetch it from actual RAM every single time you read it.

> Takeaway 19.5 #A _volatile objects are stored each time they are modified._

**Explanation:** Changes to `volatile` variables are written out to memory immediately, ensuring that if a `longjmp` or hardware interrupt happens right after, the new value is safely in memory.

> Takeaway 19.5 #B _The typedef for jmp_buf hides an array type._

**Explanation:** `jmp_buf` is declared something like `typedef struct context jmp_buf[1];`. Because it's an array, when you pass it to functions, it decays to a pointer automatically. You can't assign one `jmp_buf` to another using `=`.

```c
// Demonstrating Takeaways 19.5 #1 to #A
#include <stdio.h>
#include <setjmp.h>
#include <stdlib.h>

jmp_buf exception_env;

void deep_function(void) {
    printf("Inside deep_function. Throwing error...\n");
    // #1: Never returns. #4: Transfers control returning '2'
    // #5: If we passed 0 here, it would become 1.
    longjmp(exception_env, 2);
    printf("This will never print.\n");
}

int main(void) {
    // #8, #9, #A: Must be volatile if modified between setjmp and longjmp
    volatile int state = 0;

    // #2 & #6: Normal flow returns 0. Used in a switch conditional.
    // #B: exception_env is passed (decays to pointer)
    switch (setjmp(exception_env)) {
        case 0:
            printf("Setup complete. state = %d\n", state);
            state = 1; // Modified across the jump
            deep_function();
            break;
        case 1:
            // Would trigger if longjmp(..., 0) was called.
            printf("Caught condition 1\n");
            break;
        case 2:
            printf("Caught error 2. state is guaranteed to be %d\n", state);
            break;
    }

    // #3: If main returns, exception_env is dead.
    return 0;
}
/* Expected behavior/output:
   Setup complete. state = 0
   Inside deep_function. Throwing error...
   Caught error 2. state is guaranteed to be 1

   What goes wrong if... 'state' was not volatile? An aggressive optimizing compiler
   might have printed "state is guaranteed to be 0" after the jump because it
   cached the initial value.
*/
```

### Recap

- `setjmp`/`longjmp` allow deep stack unwinding.
- They interact poorly with compiler optimizations.
- Local variables modified between `setjmp` and `longjmp` _must_ be `volatile`.

---

## 19.6. Signal handlers

### Handling External Interrupts

**What it is:** Signals are software interrupts delivered to a process by the OS or user (e.g., `Ctrl+C`). C provides `<signal.h>` to catch these events via a callback function.

**Why it matters / how it works:** Programs need to respond to fatal errors (Segmentation Fault) or user requests to terminate smoothly. The OS pauses the normal thread, executes the registered signal handler, and then resumes the program.

**Key details:**

- `SIGINT`, `SIGTERM`, `SIGSEGV` are standard signals.
- Dispositions can be custom handlers, `SIG_DFL` (default action), or `SIG_IGN` (ignore).

**Pitfalls:** Signal handlers execute asynchronously. Doing almost _anything_ inside a signal handler (like calling `printf` or allocating memory) is strictly unsafe because the main program might have been interrupted right in the middle of modifying internal locks/state.

> Takeaway 19.6 #1 _C’s signal-handling interface is minimal and should only be used for elementary situations._

**Explanation:** Standard C signal handling is heavily restricted. Modern applications often rely on POSIX-specific extensions (like `sigaction`) for robust handling, but standard C implies keeping handlers extremely basic.

> Takeaway 19.6 #2 _Signal handlers can kick in at any point of execution._

**Explanation:** A signal doesn't wait for a sequence point or the end of a line. It can happen between CPU instructions representing a single C statement.

> Takeaway 19.6 #3 _After return from a signal handler, execution resumes exactly where it was interrupted._

**Explanation:** Unless the handler calls `abort()` or `longjmp()`, returning from the callback gives control back to the exact CPU cycle that was interrupted.

> Takeaway 19.6 #4 _Any C statement may correspond to several processor instructions._

**Explanation:** `x = y + z;` involves loading `y`, loading `z`, adding them, and storing to `x`. An interrupt can occur after the load but before the store.

> Takeaway 19.6 #5 _Signal handlers need types with uninterruptible operations._

**Explanation:** If a signal interrupts a multi-byte write to a standard integer, and the handler reads that integer, it might read a "zombie" half-written value. C provides `sig_atomic_t` (and atomic types in `<stdatomic.h>`) guaranteed to be read/written in one uninterruptible CPU instruction.

> Takeaway 19.6 #6 _Objects of type sig_atomic_t should not be used as counters._

**Explanation:** While reading/writing `sig_atomic_t` is indivisible, `count++` is not (Read, Modify, Write). If you need a counter, use atomic locks (`_Atomic`); `sig_atomic_t` is strictly a flag.

> Takeaway 19.6 #7 _Unless specified otherwise, C library functions are not asynchronous signal safe._

**Explanation:** You cannot call `printf`, `malloc`, or `exit` in a signal handler. They use internal global state (like locks) which might have been left in a corrupted half-state when the interrupt hit. You can only safely call `quick_exit`, `_Exit`, `abort`, `signal`, or atomic operations.

```c
// Demonstrating Takeaways 19.6 #1 to #7
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h> // for sleep() in posix environments

// #5: Uninterruptible type flag
volatile sig_atomic_t keep_running = 1;

// #1 & #7: Minimal interface, NO printf allowed here!
void handle_sigint(int sig) {
    // #6: We only assign, we DO NOT do keep_running++
    keep_running = 0;

    // We can also reset the signal to default if we want
    // signal(sig, SIG_DFL);
}

int main(void) {
    // Register the handler
    if (signal(SIGINT, handle_sigint) == SIG_ERR) {
        return EXIT_FAILURE;
    }

    printf("Running... press Ctrl+C to stop.\n");

    // #2 & #3: Main loop can be interrupted anytime, resumes smoothly
    while (keep_running) {
        // Simulating work
    }

    printf("\nGracefully stopped by signal!\n");
    return EXIT_SUCCESS;
}
/* Expected behavior/output:
   Running... press Ctrl+C to stop.
   (User presses Ctrl+C)
   Gracefully stopped by signal!

   What goes wrong if... you put printf() inside handle_sigint?
   If Ctrl+C happens *while* the main loop was doing a printf, the internal
   mutexes inside stdout are deadlocked, and your program freezes forever.
*/
```

### Recap

- Signals represent asynchronous external events.
- Signal handlers must be incredibly simplistic.
- Communicate with the main thread using `volatile sig_atomic_t` flags.
- Do not call standard C library functions (except a select few) inside a handler.

---

## Summary

The execution of C code follows a standard sequence dictated by control flow statements and defined boundaries known as sequence points. However, this linearity can be heavily manipulated using features like `goto` for local jumps, and `setjmp`/`longjmp` for cross-function stack unwinding. These tools, while powerful for error handling and deep parsing (like the recursive descent parser), interfere significantly with compiler optimizations, necessitating the use of the `volatile` keyword to ensure data integrity. Furthermore, asynchronous external events introduce signals, which interrupt the program at unpredictable micro-instruction levels. Handling signals safely requires strict adherence to asynchronous-signal-safe practices, primarily communicating state via indivisible `sig_atomic_t` flags to prevent race conditions and deadlocks.

---

## Self-Check Questions

1.  **What defines a "sequence point" in C?**
    A point in execution where all side effects from previous evaluations are completed and subsequent side effects have not begun (e.g., the end of a statement `;`).

2.  **Why is the evaluation order of function arguments dangerous to rely on?**
    The order is indeterminately sequenced; the compiler can choose any order, meaning side effects in the arguments can lead to unpredictable results.

3.  **How does a `goto` statement interact with local variable lifetimes if it jumps backward in a loop?**
    Each loop iteration conceptually creates new instances of local objects. Jumping back restarts the block, meaning objects will be re-initialized.

4.  **What is the purpose of the `jmp_buf` type?**
    It is an opaque array type used to store the hardware execution context (registers, stack pointer) for `setjmp` and `longjmp`.

5.  **Why does `longjmp` have the `[[noreturn]]` attribute?**
    Because it abandons the current function's control flow and teleports the execution back to the corresponding `setjmp` call location; it never reaches the end of its own function.

6.  **If `setjmp(buf)` is evaluated normally for the first time, what does it return?**
    It returns `0`.

7.  **Why must local variables modified between `setjmp` and `longjmp` be marked `volatile`?**
    To prevent the compiler from caching them in registers. `volatile` forces writes and reads directly to/from memory, ensuring `longjmp` restores the true updated value.

8.  **What happens if a signal handler interrupts a statement like `double x = y;`?**
    Because `double` assignment may take multiple machine instructions, the interrupt might happen halfway through, leaving a "zombie" or partial value if the handler attempts to read it.

9.  **Why shouldn't you use `count++` on a `sig_atomic_t` variable inside a signal handler?**
    `sig_atomic_t` only guarantees that reads and writes are indivisible. `count++` is a read, modify, _and_ write operation, which can still be interrupted.

10. **Name three C library functions that are safe to call inside a signal handler.**
    `_Exit`, `quick_exit`, and `abort`. (Most others, like `printf` or `malloc`, are unsafe).
