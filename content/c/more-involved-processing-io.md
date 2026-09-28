+++
title = "Modern C: More involved processing and IO"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Chapter 14 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"More involved processing and IO,"** builds upon pointer mechanics to explore sophisticated string processing, formatted stream input, extended character encodings (multibyte and UTF), binary stream manipulation, and compile-time resource embedding.

Here is a detailed breakdown of the most important concepts and key takeaways from Chapter 14 across its six core sections:


### 1. Text Processing & Safe Formatting (Section 14.1)
Pointer arithmetic unlocks flexible text processing, but older C library interfaces introduce subtle type-safety and buffer overflow risks.

* **`const`-Safety in Search Interfaces:**
  * **Takeaway 14.1 #1:** *The string `strto...` conversion functions are not `const`-safe*.
  * **Takeaway 14.1 #2:** *The function interfaces for `memchr` and `strchr` search functions are not `const`-safe*. Historically, passing a `char const*` into these functions allowed returning a non-const `char*` pointer into the string, creating a type-system hole.
  * **Takeaway 14.1 #3:** *The type-generic interfaces for `memchr` and `strchr` search functions are `const`-safe*. C23 resolves this flaw by providing type-generic macros that preserve pointer qualifiers.
  * **Takeaway 14.1 #4:** *The `strspn` and `strcspn` search functions are `const`-safe* because they return index offsets rather than reinterpreted pointers.
* **Buffer Safety with `snprintf`:**
  * **Takeaway 14.1 #5:** *`sprintf` makes no provision against buffer overflow*. If the destination buffer is too small, `sprintf` causes undefined memory corruption.
  * **Takeaway 14.1 #6:** *Use `snprintf` when formatting output of unknown length*.
  * **Sizing Trick:** Calling `snprintf(nullptr, 0, format, ...)` writes nothing to memory but returns the exact number of bytes required to store the formatted string, allowing programs to allocate buffers dynamically with exact precision.


### 2. Formatted Input: `scanf`, `fscanf`, and `sscanf` (Section 14.2)
The `scanf` family (`fscanf` for streams, `scanf` for `stdin`, `sscanf` for strings) parses structured text into values, but possesses tricky edge cases.

* **Pointer Arguments:** Every target parameter passed to `scanf` must be a valid pointer to the receiving variable.
* **Whitespace & Newline Behavior:**
  * A space character `' '` in a `scanf` format string matches any sequence of zero or more whitespace characters (spaces, tabs, newlines).
  * `%s` skips leading whitespace and scans a sequence of non-whitespace characters, placing a terminating `0` byte at the end. `%c` reads exact character counts without skipping whitespace.
* **Scan Sets (`%[...]`):** Scan sets match specific character sets (e.g., `%` for digits or `%[^\n]` to read up to a newline).
* **Fragility:** Because `scanf` functions read across newline boundaries and easily desynchronize on unexpected input, Gustedt recommends reading full lines using `fgets` (or `fgetline`) first, followed by numeric conversion functions like `strtod` or `strtoul`.


### 3. Extended Character Sets & Multibyte Strings (Section 14.3)
To support internationalization beyond ASCII, C provides **multibyte character strings** and **wide characters** (`wchar_t`).

* **Locale Configuration:** Printing and processing multibyte strings correctly requires invoking `setlocale(LC_ALL, "")` at application startup.
* **Properties of Multibyte Strings:**
  * **Takeaway 14.3 #1:** *Multibyte characters don’t contain null bytes* inside their byte sequences.
  * **Takeaway 14.3 #2:** *Multibyte strings are null terminated*.
  * Because multibyte strings contain no internal null bytes, standard string functions like `strcpy` work seamlessly. However, `strlen` returns the raw *byte count*, whereas computing displayed character glyph counts requires `mbsrlen`.
* **Wide Characters:** Wide characters (`wchar_t`, classified via `<wctype.h>`) use fixed-width integer encodings prefixed with `L` (e.g., `L'ä'`, `L"string"`).


### 4. UTF Encodings & Restartable Conversions (Sections 14.4 & 14.5)
C23 standardizes explicit Unicode UTF encodings and stateful conversion utilities.

* **C23 UTF Types:** `<uchar.h>` defines fixed UTF character types (`char8_t` for UTF-8, `char16_t` for UTF-16, `char32_t` for UTF-32) accompanied by string literal prefixes (`u8"..."`, `u"..."`, `U"..."`).
* **Restartable Conversions (`XXXrtoYYY`):** Functions such as `mbrtoc8` and `c8rtomb` perform restartable conversions between multibyte encodings and UTF characters.
* **State Tracking (`mbstate_t`):**
  * **Takeaway 14.5 #2:** *The multibyte `mb` encoding of a code point may be collected piecewise from the input*.
  * By passing a state tracking object (`mbstate_t*`), restartable functions can process incomplete multibyte character sequences split across buffer boundaries without losing parsing context.


### 5. Binary Streams & C23 Resource Embedding (Section 14.6)
Unlike text streams (which convert end-of-line sequences like `\n` to platform-specific conventions), **binary streams** transfer uninterpreted, raw object bytes 1-to-1.

* **Core Binary I/O Functions:**
  * `fread(ptr, size, nmemb, stream)` and `fwrite(ptr, size, nmemb, stream)` read and write contiguous memory blocks.
  * `fseek(stream, offset, whence)` updates file stream positions relative to `SEEK_SET` (file start), `SEEK_CUR` (current position), or `SEEK_END`.
  * `ftell(stream)` retrieves the current byte position.
* **Binary Rules & Limitations:**
  * **Takeaway 14.6 #1:** *Open streams on which you use `fread` or `fwrite` in binary mode* (using mode flags like `"rb"` or `"wb"`).
  * **Takeaway 14.6 #2:** *Files written in binary mode are not portable between platforms* due to differences in platform endianness, type padding, and struct alignment.
  * **Takeaway 14.6 #3:** *`fseek` and `ftell` are not suitable for very large file offsets* exceeding `LONG_MAX` bytes (~2 GiB on 32-bit systems).
* **C23 `#embed` Directive:**
  * C23 introduces `#embed "filename"` (guarded by `__has_embed`), which embeds binary files (such as images, certificates, or datasets) directly into character array initializers at compile time, eliminating manual binary-to-C translation tools.

## Examples

