+++
title = "Modern C: Program failure"
description = "A guide to C23 program failures, from invalid operations and resource exhaustion to race conditions and livelocks, with strategies for prevention, error handling, and cleanup."
date = 2026-09-28T05:09:19Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Here is a detailed, comprehensive study note for **Chapter 15: Program failure**, generated directly from the provided text of _Modern C: A Guide to the C23 Standard_.

## 15.1 Wrongdoings

### What it is

Wrongdoings represent the most straightforward category of program failures. These are specific actions, events, or omissions during execution that directly cause a failure. The book compares wrongdoings to traffic accidents where the driver is clearly at fault—such as speeding or drunk driving—rather than blaming the road or the environment.

### Why it matters / how it works

When a program commits a wrongdoing, it immediately violates the abstract state machine's rules. This often leads to what the C jargon calls _Undefined Behavior_ (UB), shifting the program into an unreliable state. Understanding wrongdoings allows programmers to check preconditions and avoid these fatal operations entirely.

### Key details

- Wrongdoings generally manifest during program execution and are often undetectable at compile time.
- Unlike syntax errors that violate language constraints and halt compilation, wrongdoings survive compilation and corrupt the runtime state.

### Pitfalls

- Focusing too much on the _results_ of the failure (what happens after UB is invoked) rather than the _causes_ (why the failure occurred) is a major pitfall.

### Connections

- Connects to the **Abstract State Machine** (Section 5.1), which dictates that operations must be strictly defined to transition properly.

### 15.1.1 Arithmetic violations

**What it is:**
Arithmetic violations happen when a program attempts an operation that has no mathematically defined result for the represented numbers, known in the standard as an _exceptional condition_.

**Why it matters / how it works:**
The CPU or floating-point environment cannot process operations like division by zero or modulo by zero for integers. In floating-point arithmetic, the environment dictates how these are handled (e.g., yielding `INFINITY` or raising a signal).

**Key details:**

- Integer division by zero directly results in program failure.
- Floating-point exceptions do not always result in program failure; they depend on the platform's floating-point environment (e.g., `<fenv.h>`).

**Pitfalls:**

- Assuming floating-point arithmetic fails the same way integer arithmetic does. Systems with `FE_DIVBYZERO` might seamlessly return `INFINITY` instead of crashing.

> Takeaway 15.1.1 #2: The floating-point environment of the platforms determines the floatingpoint operations that result in program failure.

_Explanation:_ Your hardware and the standard library implementation (such as whether `INFINITY` is defined) control what happens when a math violation occurs. You must query this environment using tools like `fetestexcept()` to know if an error occurred.

> Takeaway 15.1.1 #4: Where possible, use array indexing instead of pointer arithmetic combined with dereferencing.

_Explanation:_ Relying on array indexing (`A[i]`) clearly communicates bounds and intent to the compiler, avoiding the arithmetic pitfalls and out-of-bounds risks associated with raw pointer arithmetic (`*(A + i)`).

```c
#include <stdio.h>
#include <fenv.h>
#include <stdlib.h>

// Code Demonstration: Arithmetic Violations
int main(void) {
    // Expected behavior: Integer division by zero triggers UB (often a crash/trap).
    // What goes wrong if... we uncomment the next line:
    // int x = 5 / 0;

    // Floating point division by zero.
    // The environment dictates the behavior.
    feclearexcept(FE_ALL_EXCEPT); // Clear previous exceptions
    double num = 5.0;
    double zero = 0.0;

    double result = num / zero; // Floating point exception

    if (fetestexcept(FE_DIVBYZERO)) {
        printf("Division by zero occurred! Result: %g\n", result);
    }

    return 0;
}

```

### 15.1.3 Value violations

**What it is:**
Value violations occur when standard C library functions are called with arguments that are invalid for the requested operation, or when the operation's result cannot be represented.

**Why it matters / how it works:**
The C library trusts the programmer to pass valid values (e.g., non-null pointers, appropriately sized bounds). Passing invalid values directly causes failure because the library functions generally do not check preconditions for you.

**Key details:**

- Examples include passing null pointers, excessively large numbers, or zero sizes to allocation functions.

**Pitfalls:**

- Relying on the C library to safely reject bad inputs. Most functions will blindly process invalid values and crash.

### 15.1.4 Function pointers and Call Prototypes

**What it is:**
This concept addresses the dangers of misusing function pointers, casting them improperly, or calling functions without consistent prototypes.

**Why it matters / how it works:**
If a function is called through a pointer that was cast away from its original type, or if different translation units use conflicting prototypes for the same function name, the execution environment may corrupt the call stack or misinterpret return values.

> Takeaway 15.1.4 #1: Don’t convert pointers unless you must.

_Explanation:_ Pointer conversions hide the true type of the object or function, bypassing the compiler's strict type checking and inviting severe runtime failures.

> Takeaway 15.1.4 #2: Always call a function with the prototype with which it is defined.

_Explanation:_ Mismatched prototypes mean the caller and callee disagree on how parameters are passed or returned, which corrupts the execution state.

> Takeaway 15.1.4 #3: Call a function by its name.

_Explanation:_ Calling a function directly by its name (rather than through a function pointer) is the simplest and safest way to avoid casting errors and prototype mismatches.

```c
#include <stdio.h>

void print_int(int a) {
    printf("Integer: %d\n", a);
}

int main(void) {
    // Expected behavior: Calling directly by name is safe.
    print_int(42);

    // What goes wrong if... we cast the function pointer to a mismatched type:
    // void (*bad_ptr)(double) = (void (*)(double))print_int;
    // bad_ptr(3.14); // UB: Prototype mismatch during call!

    return 0;
}

```

### 15.1.5 Access violations

**What it is:**
Access violations are improper uses of memory interfaces, occurring when a program reads or writes to memory it shouldn't. The text likens this to ignoring a "No Entry" sign in traffic.

**Why it matters / how it works:**
Because C's type system cannot always track pointer bounds and lifetimes at compile time, illegal memory accesses are usually caught by the operating system (e.g., a segmentation fault) or silently corrupt data.

**Key details:**

- **Common cases:** Null pointer dereference, accessing freed storage, out-of-bounds array access, or modifying a `const`-qualified object (like a string literal).
- Other cases include unsequenced modifications, accessing atomic structures improperly, or calling `free` twice.

**Pitfalls:**

- Believing that access violations will immediately crash the program. Often, they silently corrupt adjacent data, delaying the failure.

### 15.1.6 Value misinterpretation

**What it is:**
Value misinterpretation (formerly known as a _trap representation_) happens when an object stores a bit pattern that is invalid for the type attempting to access it. C23 refers to this as an _indeterminate representation_.

**Why it matters / how it works:**
When memory is copied or casted inappropriately, a type might be forced to interpret raw bytes that don't constitute a valid value for that type.

> Takeaway 15.1.6 #1: Don’t store values other than 0 or 1 in a bool object.

_Explanation:_ A boolean object expects only a strictly binary state. Forcing other byte representations into a `bool` compromises the abstract state machine.

> Takeaway 15.1.6 #2: Don’t change the representation bytes of objects directly.

_Explanation:_ Manually shifting or injecting bytes into standard types (especially floating-point or booleans) creates non-value representations, leading to immediate program failure when accessed.

```c
#include <stdio.h>
#include <stdbool.h>
#include <string.h>

int main(void) {
    bool my_bool = true;

    // What goes wrong if... we change the representation bytes directly:
    // unsigned char bad_val = 42;
    // memcpy(&my_bool, &bad_val, sizeof(bool));
    // printf("Bool is: %d\n", my_bool); // UB: Value misinterpretation!

    return 0;
}

```

### 15.1.7 Explicit invalidation

**What it is:**
Explicit invalidation utilizes the C23 `unreachable()` macro to annotate control paths that the programmer guarantees will never be executed.

**Why it matters / how it works:**
It is a tool for the optimizer. If the control flow reaches an `unreachable()` macro, the abstract state machine treats it as a program failure. It differs from `exit()` or `abort()`, which prescribe specific termination behavior; `unreachable()` simply tells the compiler the path doesn't exist.

> Takeaway 15.1.7 #1: Only use unreachable() where you have proof.

_Explanation:_ If you cannot mathematically or logically prove the path is impossible, the compiler's optimizations around `unreachable()` will actively break your program.

> Takeaway 15.1.7 #2: Don’t use other operations than unreachable() to mark a control path that will never be taken.

_Explanation:_ Do not artificially trigger undefined behavior (like dividing by zero) to hint to the compiler that a path is dead. Use the explicit standard macro meant for this purpose.

_Recap of 15.1:_

- Wrongdoings are direct causes of UB.
- They encompass math errors, library misuses, pointer mishandling, and memory violations.
- Explicit hints like `unreachable()` must be used strictly logically.

---

## 15.2 Program state degradation

### What it is

Program state degradation is a systemic failure resulting from the cumulative interplay of multiple actions over time, rather than a single explicit wrongdoing. The text equates this to being part of a traffic jam: no single driver caused it, but everyone contributes to the gridlock.

### Why it matters / how it works

Degradation exhausts system resources. Because the standard environment has finite capacity for memory and execution contexts, failing to manage resources degrades the state until the execution inevitably crashes.

### 15.2.1 Unbounded recursion

**What it is:**
Unbounded recursion occurs when a recursive function fails to make progress toward its base case, exhausting the platform's capacity to provide function call contexts.

**Why it matters / how it works:**
Each function call pushes a new context onto the _stack_. If the recursion never bottoms out, it leads to a _stack overflow_, which is a special form of state degradation that crashes the program.

### 15.2.2 Storage exhaustion

**What it is:**
Storage exhaustion happens when the dynamically allocated memory system (the _heap_) or other finite system resources (files, threads, mutexes) are depleted.

**Why it matters / how it works:**
Functions like `malloc`, `calloc`, and `realloc` return a null pointer when memory is exhausted. While the C standard doesn't provide a way to predict stack exhaustion, heap exhaustion is detectable and must be handled.

**Key details:**

- Other scarce resources include streams (`FOPEN_MAX`), threads (`thrd_create`), and mutexes (`mtx_init`).

_Recap of 15.2:_

- Degradation is a gradual, resource-based failure.
- The Stack is vulnerable to unbounded recursion.
- The Heap and system resources are vulnerable to unmonitored allocations.

---

## 15.3 Unfortunate incidents

### What it is

Unfortunate incidents are rare, complex failures caused by the fatal alignment of distant, seemingly unrelated events in space or time. The book compares this to independent cars colliding at an intersection due to blind spots.

### Why it matters / how it works

Because the components involved might be perfectly valid in isolation, diagnosing these failures is notoriously difficult.

### 15.3.1 Escalating state degradation

**What it is:**
This occurs when a program ignores the warning signs of state degradation (like a null pointer from `malloc`) and proceeds anyway.

**Why it matters / how it works:**
Continuing execution after resource exhaustion causes erratic system reactions. Like ignoring a brake malfunction light in a car, it jeopardizes not only the program but potentially the host system and innocent bystanders.

### 15.3.2 Collisions and race conditions

**What it is:**
Collisions happen when subexpressions with side effects access and modify the same memory locations in unsequenced ways.

**Why it matters / how it works:**
Because C expression evaluation does not strictly prescribe execution order (sequencing), a compiler might interleave reads and writes arbitrarily.

> Takeaway 15.3.2 #1: Don’t read and modify the same object within the same arithmetic expression.

_Explanation:_ An expression like `x++ + x` is a wrongdoing because the modification of `x` is unsequenced relative to the secondary read of `x`, resulting in unpredictable output.

**Key details:**

- With pointers, this becomes a _race condition_ (e.g., `(*p)++ + (*q)` where `p == q`).
- Signal handlers and multithreaded concurrent access are major sources of uncontrollable race conditions.

```c
#include <stdio.h>

int main(void) {
    int x = 5;
    // What goes wrong if... we violate sequencing rules:
    // printf("%d\n", x++ + x); // UB: Unsequenced read and modify

    // Expected behavior: Separate the operations.
    x++;
    printf("%d\n", x + x); // Safe and defined

    return 0;
}

```

### 15.3.3 Inappropriate library calls and macro invocations

**What it is:**
Invoking specific C library functions in contexts where they are forbidden, such as calling `signal` in a multithreaded program or placing the `setjmp` macro outside of highly restricted expression contexts.

### 15.3.4 Deadlocks

**What it is:**
A deadlock failure occurs in multithreaded contexts when several threads trap themselves in a cyclic chain of dependencies for shared resources. The text uses a traffic roundabout that is completely locked up as a visual analogy.

_Recap of 15.3:_

- Unfortunate incidents are hard-to-trace interactions between distant code elements.
- Unsequenced expressions cause data collisions.
- Deadlocks trap concurrent operations.

---

## 15.4 Series of unfortunate events

### What it is

This describes a program execution that gets trapped in an endless loop over a finite set of states without making any visible progress or producing side effects.

### Why it matters / how it works

The compiler is allowed to assume that loops eventually terminate or produce observable behavior. If neither happens, the program state is effectively lost, similar to a plane circling endlessly in the eye of a storm.

> Takeaway 15.4 #1: A program execution that loops over a finite set of states with no observable side effects has failed.

_Explanation:_ Infinite loops must do something observable (IO, modifying global state, calling volatile memory). If they do not, the compiler may legally optimize the loop away or trigger a failure.

_Recap of 15.4:_

- Infinite loops without observable behavior are considered program failures.
- Compilers use this rule to aggressively optimize.

---

## 15.5 Dealing with failures

### What it is

This section outlines the strategies required to prevent, detect, and mitigate program failures across the different categories.

### Why it matters / how it works

Different failures require different mitigation tactics. Because byzantine failures (UB) cannot be captured once they happen, proactive design is strictly required.

> Takeaway 15.5 #1: Ensure all preconditions for an operation that could fail.

_Explanation:_ You must actively guard against wrongdoings (null checks, bounds checks, zero division checks) before they execute.

> Takeaway 15.5 #2: The return of operations that might exhaust resources should be checked for errors.

_Explanation:_ Always check if a system call or allocation (like `malloc`) returned a failure indicator (like `NULL` or a specific `errno`) to catch state degradation early.

> Takeaway 15.5 #3: Unfortunate events can only be avoided with a careful algorithm design.

_Explanation:_ Because race conditions and deadlocks are systemic and often un-testable at runtime, you must design your concurrency architecture carefully on paper before writing the code.

**Key details:**

- Systems provide termination functions (`exit`, `abort`, `_Exit`) and interrupt handlers (signals, traps) to shut down gracefully when a failure is detected.

_Recap of 15.5:_

- Prevent wrongdoings via precondition checks.
- Prevent degradation by checking resource return values.
- Prevent unfortunate events through superior architectural design.

---

## 15.6 Error checking and cleanup

### What it is

The practical implementation of capturing runtime errors, managing the `errno` state, and gracefully cleaning up allocated resources before terminating or returning.

### Why it matters / how it works

Functions must safely unwind their state. If a function allocates memory and subsequently fails an IO operation, it must still free that memory. The standard technique in C for this involves using `goto` statements to centralize cleanup code.

### Key details

- C library functions return specific error values (e.g., `-EFAULT`, `-ENOMEM`) or set `errno`.
- If a function detects an error but needs to clean up, it should save the state of `errno`, perform the `free()` calls, and then restore `errno` before returning, ensuring the caller sees the original error.

> Takeaway 15.6 #1: Labels for goto are visible in the entire function that contains them. (Extracted from takeaway list / context).

_Explanation:_ You can safely jump forward to a cleanup label (`CLEANUP:`) from anywhere within the same function block, making resource deallocation linear and avoiding nested `if-else` cascades.

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

// Code Demonstration: Error checking and cleanup
int process_file(const char* filename) {
    int prev_errno = errno;
    int status = -1;

    // Check preconditions
    if (!filename) return -EINVAL;

    FILE* file = fopen(filename, "r");
    if (!file) {
        return -errno;
    }

    char* buffer = malloc(1024);
    if (!buffer) {
        status = -ENOMEM;
        goto CLEANUP_FILE;
    }

    // Simulate work that fails
    if (fgets(buffer, 1024, file) == NULL) {
        status = -EIO;
        goto CLEANUP_ALL;
    }

    status = 0; // Success

CLEANUP_ALL:
    free(buffer);
CLEANUP_FILE:
    fclose(file);

    // Restore errno to whatever failed during the process
    if (status != 0) errno = prev_errno;
    return status;
}

```

_Recap of 15.6:_

- Use `goto` for unified cleanup blocks.
- Manage `errno` carefully so the caller gets accurate diagnostics.
- Never skip `free()` calls when erroring out early.

---

## Summary

Chapter 15 establishes that C programs fail via three primary avenues: wrongdoings, program state degradation, and unfortunate incidents. Wrongdoings are explicit violations of the abstract state machine, such as dividing by zero, accessing uninitialized pointers, or misusing C library functions, which immediately invoke undefined behavior. Program state degradation acts like a slow leak, where unmonitored recursion or unchecked memory allocations eventually exhaust the platform's finite resources. Finally, unfortunate incidents are complex architectural failures, such as deadlocks, race conditions, and unsequenced memory access, where otherwise valid code collides fatally in time or space. Because the C language fundamentally prioritizes speed over safety, the compiler rarely protects the programmer from these failures; instead, you must rigorously enforce preconditions, meticulously check the return values of all resource-allocating functions, and architect concurrent environments to structurally preclude race conditions. When errors are detected, functions must rely on structured cleanups (often using `goto`) to gracefully release resources without corrupting the global error state.

---

## Self-Check Questions

1. **What is the difference between a wrongdoing and program state degradation?**
   _Answer:_ A wrongdoing is a direct, singular invalid operation (like dividing by zero) that immediately violates standard rules. State degradation is a gradual failure over time (like a memory leak or stack overflow) due to resource exhaustion.

2. **Does an integer division by zero and a floating-point division by zero behave identically in C?**
   _Answer:_ No. Integer division by zero always causes program failure. Floating-point division behavior depends entirely on the platform's specific floating-point environment implementation (e.g., it might return `INFINITY`).

3. **Why is it dangerous to cast function pointers before calling them?**
   _Answer:_ Calling a function through a pointer with an incompatible prototype causes undefined behavior because the compiler misaligns the parameters and return types on the call stack.

4. **What is an "indeterminate representation" (or trap representation)?**
   _Answer:_ It is a sequence of bits in memory that does not represent a mathematically or logically valid value for the type reading it, such as storing the byte value `42` into a `bool` object.

5. **How should a programmer communicate to the compiler that a specific `if` branch is mathematically impossible to reach?**
   _Answer:_ By placing the `unreachable()` macro inside that branch. However, this must only be used if there is absolute logical proof.

6. **What is the consequence of an unbounded recursive function in C?**
   _Answer:_ It continually pushes new contexts onto the call stack until the stack is exhausted, resulting in a fatal program state degradation (stack overflow).

7. **Why is evaluating `x++ + x` considered a program failure?**
   _Answer:_ It creates a collision/race condition within a single expression because the modification of `x` and the reading of `x` are unsequenced by the C standard.

8. **What happens if an infinite loop performs no observable side effects (no IO, no global state changes)?**
   _Answer:_ The C standard dictates this is a program failure. The compiler is legally permitted to optimize the loop away entirely.

9. **How should a function cleanly handle multiple points of failure without leaving memory allocated?**
   _Answer:_ By using `goto` statements to jump to a centralized cleanup block at the end of the function where all allocated resources are freed before returning.

10. **Why is it necessary to save and restore `errno` during a function's cleanup phase?**
    _Answer:_ `errno` is a global state variable. If a cleanup function (like `fclose`) temporarily sets `errno`, it overwrites the original error that triggered the cleanup, depriving the caller of accurate diagnostic information.
