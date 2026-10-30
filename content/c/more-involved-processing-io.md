+++
title = "Modern C: More involved processing and IO"
description = "A guide to C23 text processing, formatted input, multibyte and UTF conversions, binary streams, and compile-time resource embedding."
date = 2026-09-28T05:02:48Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 14.1. Text processing

### String Searching and `const`-Safeness

**What it is:** When processing strings, standard C library functions like `memchr` and `strchr` are used to find characters. However, historically, these functions returned a non-`const` pointer even when passed a `const` string, breaking `const`-correctness.

**Why it matters / how it works:** If a function returns a mutable pointer to an immutable string, it opens the door for accidental modification of read-only memory, leading to undefined behavior or segmentation faults. C23 resolves this by providing type-generic macros that ensure the return type respects the `const`-ness of the input.

**Key details:**

- Standard `strchr` returns `char*`, stripping the `const` qualifier of a `const char*` argument in older C versions.
- C23 type-generic interfaces correctly map `const char*` inputs to `const char*` outputs.
- Functions returning an integer index (like `strspn` and `strcspn`) have always been safe since they don't return pointers.

**Pitfalls:** Compiling pre-C23 code might silently allow a `const` string to be modified via the pointer returned by `strchr`.

**Connections:** This connects to pointer qualifiers (Chapter 6) and type-generic macros (`<tgmath.h>`, Chapter 8).

> **Takeaway 14.1 #2** The function interfaces for `memchr` and `strchr` search functions are not `const`-safe.

> **Takeaway 14.1 #3** The type-generic interfaces for `memchr` and `strchr` search functions are `const`-safe.

> **Takeaway 14.1 #4** The `strspn` and `strcspn` search functions are `const`-safe.

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    const char* text = "Hello, C23!";

    // In C23, this uses a type-generic macro.
    // It returns const char* because 'text' is const char*.
    const char* found = strchr(text, 'C');

    if (found) {
        printf("Found: %s\n", found);
    }

    // Using strspn (always const-safe as it returns size_t)
    size_t span = strspn(text, "Helo, ");
    printf("Initial span matching 'Helo, ': %zu\n", span);

    return 0;
}
/* Expected Output:
Found: C23!
Initial span matching 'Helo, ': 7
*/

/* What goes wrong if... (Pre-C23)
char* bad_ptr = strchr(text, 'C'); // Historically allowed, stripping const!
*bad_ptr = 'X'; // Undefined behavior: modifying a string literal!
*/

```

### Output Formatting and Buffer Overflow

**What it is:** The `sprintf` function writes formatted data into a character string instead of a stream. Its safer counterpart is `snprintf`, which limits the number of characters written.

**Why it matters / how it works:** `sprintf` just writes to the provided buffer until the format string is exhausted, assuming the buffer is large enough. `snprintf` takes a size parameter and guarantees it will never write past that bound (including the null terminator).

**Key details:**

- `sprintf` always ensures a `0` (null terminator) is placed at the end of the string.
- `snprintf(NULL, 0, format, ...)` is a common idiom to calculate the exact buffer size needed before allocating memory.

**Pitfalls:** Using `sprintf` with untrusted input or dynamically sized data almost inevitably leads to buffer overflow vulnerabilities.

**Connections:** Relates to `printf` and formatted output (Chapter 8).

> **Takeaway 14.1 #5** `sprintf` makes no provision against buffer overflow.

> **Takeaway 14.1 #6** Use `snprintf` when formatting output of unknown length.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    double val = 12345.6789;

    // 1. Determine required size using snprintf with NULL/0
    int req_size = snprintf(NULL, 0, "The value is %f", val);

    if (req_size >= 0) {
        // 2. Allocate exact memory (+1 for null terminator)
        char* safe_buf = malloc(req_size + 1);
        if (safe_buf) {
            // 3. Write safely
            snprintf(safe_buf, req_size + 1, "The value is %f", val);
            printf("%s\n", safe_buf);
            free(safe_buf);
        }
    }
    return 0;
}
/* Expected Output:
The value is 12345.678900
*/

/* What goes wrong if...
char small_buf[10];
sprintf(small_buf, "The value is %f", val);
// Buffer overflow! small_buf only holds 10 bytes, but the string is much longer.
*/

```

**Recap:**

- C23 type-generic macros make string searches `const`-safe.
- Never use `sprintf`; always prefer `snprintf` for string formatting to prevent buffer overflows.

---

## 14.2. Formatted input

### The `scanf` Family

**What it is:** The C library provides `fscanf` (stream), `scanf` (`stdin`), and `sscanf` (string) to parse formatted text into typed variables.

**Why it matters / how it works:** These functions do the reverse of `printf`. They rely on format strings (e.g., `"%lg"`) and require pointers to variables so they can modify them.

**Key details:**

- All arguments corresponding to specifiers _must_ be pointers.
- Whitespace handling is tricky: a space `' '` in the format matches _any_ sequence of whitespace (spaces, tabs, newlines).
- `%s` matches non-whitespace characters and appends a `0` (null terminator), while `%c` reads raw characters without adding a terminator.
- Character classes like `%[aeiou]` can match specific sets, and `%[^ \n]` acts as a negated set.
- The `*` modifier suppresses assignment (e.g., `%*[\n]` skips newlines without storing them).

**Pitfalls:** Failing to check the return value (the number of successfully assigned variables) can result in using uninitialized variables if parsing fails. `%s` without a width limit (like `%20s`) is a buffer overflow risk.

**Connections:** Counterpart to `printf` formats (Chapter 8.4) but has divergence in types (e.g., `%f` for `float` and `%lf` or `%lg` for `double` in `scanf`, unlike `printf`).

```c
#include <stdio.h>

int main(void) {
    const char* input = "  42 3.1415 text\n";
    int i;
    double d;
    char str[10];

    // sscanf reads from a string.
    // %d reads integer, %lg reads double, %9s reads up to 9 non-whitespace chars.
    int parsed = sscanf(input, "%d %lg %9s", &i, &d, str);

    if (parsed == 3) {
        printf("Parsed: %d, %g, %s\n", i, d, str);
    } else {
        printf("Failed to parse all items.\n");
    }
    return 0;
}
/* Expected Output:
Parsed: 42, 3.1415, text
*/

/* What goes wrong if...
sscanf(input, "%d %lg", i, d); // Missing '&' pointers! Will cause a memory fault.
*/

```

**Recap:**

- `scanf` requires pointers to write to.
- Check the return value to confirm assignments.
- Beware of whitespace nuances and use width limits on `%s`.

---

## 14.3. Extended character sets

### Multibyte Strings and `locale`

**What it is:** Multibyte strings allow the representation of extended character sets (like Unicode) using sequences of one or more basic `char` bytes.

**Why it matters / how it works:** Traditional C was built around 8-bit ASCII. Multibyte strings allow global text representation. Before printing extended characters, the program must synchronize with the environment's native encoding using `setlocale(LC_ALL, "")`.

**Key details:**

- A **multibyte character** is a sequence of bytes representing one extended character.
- They are stored in standard `char` arrays.
- `strlen` returns the number of _bytes_, not the number of _characters_.
- Parsing state is tracked via the `mbstate_t` type from `<wchar.h>`.

**Pitfalls:** Forgetting `setlocale` will result in terminal garbage. Standard string functions like `strchr` won't find extended characters correctly because they look at single bytes.

**Connections:** Relates to basic character types (Chapter 5) and string library functions (Chapter 8.5).

> **Takeaway 14.3 #1** Multibyte characters don't contain null bytes.

> **Takeaway 14.3 #2** Multibyte strings are null terminated.

```c
#include <stdio.h>
#include <string.h>
#include <locale.h>

int main(void) {
    // 1. Switch to native locale for proper terminal rendering
    setlocale(LC_ALL, "");

    // 2. Multibyte string containing a 2-byte UTF-8 character (ä)
    char mb_str[] = "b\u00E4r"; // 'b', 'ä', 'r', '\0'

    printf("String: %s\n", mb_str);

    // 3. strlen counts bytes, not glyphs. 'ä' is 2 bytes in UTF-8.
    printf("Byte length (strlen): %zu\n", strlen(mb_str));

    // Because they don't contain inner nulls, standard copy works.
    char copy[10];
    strcpy(copy, mb_str);
    printf("Copied correctly: %s\n", copy);

    return 0;
}
/* Expected Output:
String: bär
Byte length (strlen): 4
Copied correctly: bär
*/

/* What goes wrong if...
// If we omit setlocale(LC_ALL, ""), the terminal might interpret the
// multibyte sequence incorrectly and print "bÃ¤r".
*/

```

### Wide Characters (`wchar_t`)

**What it is:** A type (`wchar_t`) that encodes exactly one extended character per array element, making character counting and iteration uniform.

**Why it matters / how it works:** Since multibyte strings have variable length characters, `wchar_t` (often 16-bit or 32-bit) allows constant-time access to the nth character.

**Key details:**

- Prefixed with `L` (e.g., `L'ä'` or `L"text"`).
- Functions generally prefixed with `wcs` (Wide Character String) map to standard string functions.

**Pitfalls:** Wide strings consume significantly more memory than UTF-8 encoded multibyte strings.

**Connections:** `<wctype.h>` provides wide-character classification functions analogous to `<ctype.h>`.

```c
#include <stdio.h>
#include <wchar.h>
#include <locale.h>

int main(void) {
    setlocale(LC_ALL, "");

    // Wide string literal
    wchar_t w_str[] = L"b\u00E4r";

    // %ls is used to print wide character strings
    printf("Wide String: %ls\n", w_str);

    // wcslen correctly counts characters (glyphs), not bytes
    printf("Character length (wcslen): %zu\n", wcslen(w_str));

    return 0;
}
/* Expected Output:
Wide String: bär
Character length (wcslen): 3
*/

```

**Recap:**

- Initialize `setlocale(LC_ALL, "")` before printing extended characters.
- Multibyte strings act like normal C strings (null-terminated, no inner nulls) but `strlen` measures bytes, not visual characters.
- `wchar_t` provides fixed-width character representation.

---

## 14.4. UTF character encodings

**What it is:** Unicode defines character concepts globally. UTF-8, UTF-16, and UTF-32 are encoding formats.

**Why it matters / how it works:** Direct locale reliance is unstable across systems. C11/C23 provide explicit types and literals for direct UTF encodings, guaranteeing behavior independent of the OS locale.

**Key details:**

- **UTF-8:** `char8_t`, prefixed with `u8` (e.g., `u8"text"`). Variable width.
- **UTF-16:** `char16_t`, prefixed with `u` (e.g., `u"text"`). Uses surrogate pairs for large code points.
- **UTF-32:** `char32_t`, prefixed with `U` (e.g., `U"text"`). Fixed width, one word per code point.
- C23 fixed `__STDC_UTF_16__` and `__STDC_UTF_32__` to exactly 1, guaranteeing that these types physically use the UTF encodings.

**Pitfalls:** UTF-16 is notoriously complicated due to surrogate pairs (the basic multilingual plane is 16-bit, but others require two 16-bit words). Prefer UTF-8 or UTF-32.

```c
#include <stdio.h>
#include <uchar.h> // for char8_t, char16_t, char32_t (C23)

int main(void) {
    // Explicit UTF-8 encoding
    char8_t utf8_str[] = u8"A\u00E4";

    // Explicit UTF-32 encoding
    char32_t utf32_str[] = U"A\u00E4";

    printf("UTF-8 byte size: %zu\n", sizeof(utf8_str));   // 'A'(1) + 'ä'(2) + '\0'(1) = 4
    printf("UTF-32 byte size: %zu\n", sizeof(utf32_str)); // 3 chars * 4 bytes = 12

    return 0;
}
/* Expected Output:
UTF-8 byte size: 4
UTF-32 byte size: 12
*/

```

**Recap:**

- C23 enforces UTF-16 and UTF-32 formats strictly.
- Use `u8`, `u`, and `U` prefixes for explicit standard encodings instead of hoping the local execution charset handles it properly.

---

## 14.5. Restartable text conversion

**What it is:** Conversion functions between multibyte/locale strings and explicit UTF strings that maintain a parsing state (`mbstate_t`), allowing conversion to pause and resume.

**Why it matters / how it works:** When reading from a network or file stream, a multibyte character (like a 3-byte UTF-8 glyph) might be split across two buffer reads. Restartable functions (prefixed/suffixed with `r`, e.g., `mbrtoc8`) check the state to see if they are in the middle of assembling a character.

**Key details:**

- Functions include `mbrtowc`, `mbrtoc8`, `mbrtoc16`, `c8rtomb`, etc.
- They return `mbincomplete` if they need more bytes, or `mbinvalid` on a bad sequence.

**Pitfalls:** Do not discard or reset the `mbstate_t` variable between chunks of the same stream, or the encoding will corrupt.

**Connections:** Relates to Stream IO (Chapter 14.6) when reading chunked data.

> **Takeaway 14.5 #1** The multibyte mb encoding of a code point is written to the output string all at once.

> **Takeaway 14.5 #2** The multibyte mb encoding of a code point may be collected piecewise from the input.

```c
#include <stdio.h>
#include <uchar.h>
#include <wchar.h>

int main(void) {
    // A UTF-8 multibyte sequence for 'ä' is 0xC3 0xA4
    char input[] = {(char)0xC3, (char)0xA4};

    mbstate_t state = {0};
    char32_t output;

    // First call: Give it only the first byte of the 2-byte character
    size_t res1 = mbrtoc32(&output, &input[0], 1, &state);

    // It should return (size_t)-2 meaning "incomplete"
    if (res1 == (size_t)-2) {
        printf("Piecewise: Incomplete, need more bytes.\n");
    }

    // Second call: Give it the second byte, maintaining the state
    size_t res2 = mbrtoc32(&output, &input[1], 1, &state);

    if (res2 == 1) {
        printf("Piecewise: Completed! Code point: U+%04X\n", (unsigned int)output);
    }

    return 0;
}
/* Expected Output:
Piecewise: Incomplete, need more bytes.
Piecewise: Completed! Code point: U+00E4
*/

/* What goes wrong if...
If we cleared `state` to {0} before the second call, `mbrtoc32` would treat
the second byte as an isolated invalid byte, returning an error.
*/

```

**Recap:**

- Use restartable functions to safely parse multibyte inputs from chunked streams.
- Outputs are generated atomically, but inputs can be digested piecewise.

---

## 14.6. Binary streams

**What it is:** Using standard streams (`FILE*`) in a mode that writes and reads raw memory bytes exactly as they appear in RAM, without text interpretation.

**Why it matters / how it works:** Text mode alters data invisibly: it translates `\n` to `\r\n` on Windows, strips trailing spaces, or alters multibyte chars. Binary mode (`"rb"`, `"wb"`) skips all translation. `fread` and `fwrite` dump bulk bytes directly to/from memory arrays.

**Key details:**

- Open files with the `b` modifier (e.g., `fopen("data.bin", "rb")`).
- Only `fgetc` and `fputc` handle binary files portably alongside `fread` and `fwrite`.

**Pitfalls:** **Endianness.** Binary dumps map internal CPU registers to disk. A file written with `fwrite` on a Little-Endian machine will read as garbage if read via `fread` into the same `struct` on a Big-Endian machine. Do not use binary mode for cross-platform sharing without writing custom byte-swapping logic.

**Connections:** Relates to file pointers and basic IO (`fopen`, Chapter 8).

> **Takeaway 14.6 #1** Open streams on which you use `fread` or `fwrite` in binary mode.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int data[3] = {255, 1024, 42};

    // 1. Write in Binary Mode
    FILE* out = fopen("test.bin", "wb");
    if (out) {
        // Write the raw bytes of the 3 integers
        fwrite(data, sizeof(int), 3, out);
        fclose(out);
    }

    // 2. Read in Binary Mode
    int input[3] = {0};
    FILE* in = fopen("test.bin", "rb");
    if (in) {
        fread(input, sizeof(int), 3, in);
        fclose(in);

        printf("Read: %d, %d, %d\n", input[0], input[1], input[2]);
    }
    return 0;
}
/* Expected Output:
Read: 255, 1024, 42
*/

/* What goes wrong if...
If we open with "w" and write binary data, any byte that happens to equal the ASCII
newline value (10) might be silently converted to a Windows newline (13, 10) on a
Windows machine, corrupting the binary data integer alignment.
*/

```

**Recap:**

- Binary mode (`"b"`) prevents silent IO translation.
- `fread` and `fwrite` are your primary binary tools.
- Binary files are generally bound to the local machine's endianness and architecture.

---

## Summary

Chapter 14 dives deeply into the robust mechanisms C provides for interacting with the outside world through text and binary data. It highlights the evolution of `const`-safeness in C23's standard string library, addresses the critical security requirement of bound-checked string processing (`snprintf` over `sprintf`), and exposes the intricacies of reading formatted input with `scanf`. A large focus is placed on the modernization of character encodings: utilizing the environment's locale to process multibyte strings natively, manipulating fixed-width wide characters (`wchar_t`), and leaning on C's formalized support for Unicode formats (UTF-8, UTF-16, UTF-32). The chapter bridges the gap between chunked stream IO and character parsing through restartable conversion functions (`mbrtoc8`), and finally draws a firm dividing line between human-readable text IO and exact, unformatted binary streams (`fread`/`fwrite`).

---

## Self-Check Questions

1. **Why should `sprintf` be avoided in modern C?**
   It provides no mechanism to stop writing if the format string evaluates to a length larger than the destination buffer, leading to buffer overflows. Use `snprintf` instead.

2. **In C23, why is `strchr(text, 'A')` safer than in C17?**
   C23 uses type-generic macros that preserve the `const`-qualification of the input string, whereas older standards returned a mutable `char*` regardless of the input.

3. **What does a space character (`' '`) in a `scanf` format string do?**
   It matches any sequence of whitespace characters (including none, spaces, tabs, and newlines).

4. **Are multibyte strings null-terminated?**
   Yes, multibyte strings are null-terminated and contain no null bytes within the actual multibyte character sequences.

5. **If `mb_str` contains UTF-8 text, what does `strlen(mb_str)` measure?**
   It measures the number of _bytes_ in the string, not the number of visual characters (code points).

6. **What function is required to enable correct printing of extended characters to the terminal?**
   `setlocale(LC_ALL, "");` must be called to sync the program with the native environment's encoding.

7. **What problem does UTF-16 have that UTF-8 and UTF-32 handle more gracefully?**
   UTF-16 uses surrogate pairs for characters outside the Basic Multilingual Plane, meaning it is neither space-efficient like UTF-8 nor strictly fixed-width (1 element = 1 character) like UTF-32.

8. **What does the `r` in `mbrtoc32` stand for, and what does it mean?**
   "Restartable". It means the function can maintain an `mbstate_t` object to parse characters that are split across multiple input chunks.

9. **What happens to a newline character (`\n`) written to a file opened in text mode vs binary mode on Windows?**
   In text mode, it is translated to `\r\n`. In binary mode, it is written exactly as the single byte `\n`.

10. **Why are files written with `fwrite` not portable across all machines?**
    Because `fwrite` dumps raw memory, the file's data will be tied to the specific endianness (byte order) and data type sizes of the machine that wrote it.
