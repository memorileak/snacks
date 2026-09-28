+++
title = "Modern C: More involved processing and IO"
description = "A guide to C23 text processing, formatted input, multibyte and UTF conversions, binary streams, and compile-time resource embedding."
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

#### Example

Standard string conversion functions (`strtod`, `strtoul`) and classic search interfaces (`memchr`, `strchr`) historically lacked `const`-safety because they accepted `char const*` inputs but returned non-const `char*` pointers. C23 resolves this issue by introducing type-generic macros that preserve pointer qualifiers. Additionally, while `sprintf` provides no protection against buffer overflows, `snprintf` guarantees buffer safety and enables exact buffer size calculations when passed `nullptr` and a length of `0`.

* **Takeaway 14.1 #1:** *The string `strto...` conversion functions are not `const`-safe*.
* **Takeaway 14.1 #2 & #3:** *The function interfaces for `memchr` and `strchr` search functions are not `const`-safe; C23 type-generic interfaces resolve this*.
* **Takeaway 14.1 #4:** *The `strspn` and `strcspn` search functions are `const`-safe*.
* **Takeaway 14.1 #5 & #6:** *`sprintf` makes no provision against buffer overflow; use `snprintf` when formatting output of unknown length*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void demo_text_processing_and_formatting(void) {
  // 1. CONST-SAFETY IN C23 SEARCH FUNCTIONS
  char const read_only_text[] = "C23 Standard Text";

  // C23 type-generic macro returns 'char const*' when passed 'char const*' (Takeaway 14.1 #3)
  char const* found = strchr(read_only_text, 'S');
  if (found) {
    printf("Found character: %c\n", *found);
  }

  // Index search functions (strspn, strcspn) return size_t offsets and are inherently const-safe
  size_t span = strspn(read_only_text, "C23 "); // Takeaway 14.1 #4
  printf("Initial matching span length: %zu\n", span);

  // 2. BUFFER SAFETY & EXACT SIZING WITH snprintf (Takeaway 14.1 #6)
  size_t val1 = 1024, val2 = 2048;
  char const* label = "Buffer Sizing Test";

  // SIZING TRICK: Passing (nullptr, 0) calculates exact required bytes without writing memory
  int required_len = snprintf(nullptr, 0, "%s: %zu + %zu", label, val1, val2);

  if (required_len > 0) {
    size_t buf_size = (size_t)required_len + 1; // Include '\0' terminator
    char* dynamic_buf = malloc(buf_size);

    if (dynamic_buf) {
      // snprintf bounds memory writes to buf_size, preventing overflows (Takeaway 14.1 #5)
      snprintf(dynamic_buf, buf_size, "%s: %zu + %zu", label, val1, val2);
      printf("Formatted output: %s\n", dynamic_buf);
      free(dynamic_buf);
    }
  }
}
```

### 2. Formatted Input: `scanf`, `fscanf`, and `sscanf` (Section 14.2)
The `scanf` family (`fscanf` for streams, `scanf` for `stdin`, `sscanf` for strings) parses structured text into values, but possesses tricky edge cases.

* **Pointer Arguments:** Every target parameter passed to `scanf` must be a valid pointer to the receiving variable.
* **Whitespace & Newline Behavior:**
  * A space character `' '` in a `scanf` format string matches any sequence of zero or more whitespace characters (spaces, tabs, newlines).
  * `%s` skips leading whitespace and scans a sequence of non-whitespace characters, placing a terminating `0` byte at the end. `%c` reads exact character counts without skipping whitespace.
* **Scan Sets (`%[...]`):** Scan sets match specific character sets (e.g., `%` for digits or `%[^\n]` to read up to a newline).
* **Fragility:** Because `scanf` functions read across newline boundaries and easily desynchronize on unexpected input, Gustedt recommends reading full lines using `fgets` (or `fgetline`) first, followed by numeric conversion functions like `strtod` or `strtoul`.

#### Example

Functions in the `scanf` family (`scanf`, `fscanf`, `sscanf`) parse input streams using format specifiers and require valid pointer arguments to store output. However, `scanf` can easily desynchronize when encountering unexpected newlines or invalid characters. A safer approach combines full-line reads using `fgets` with robust numerical parsing via `strtoul` or `strtod`.

```c
#include <stdio.h>
#include <stdlib.h>

void demo_formatted_input(void) {
  // 1. FORMATTED SCANNING WITH sscanf
  char const* input_line = "2026 0xFF 3.14159";
  int year;
  unsigned int hex_val;
  double pi;

  // sscanf requires pointers to receiving variables (&year, &hex_val, &pi)
  int scanned = sscanf(input_line, "%d %x %lg", &year, &hex_val, &pi);
  if (scanned == 3) {
    printf("Parsed via sscanf: Year=%d, Hex=0x%X, Pi=%g\n", year, hex_val, pi);
  }

  // 2. ROBUST ALTERNATIVE: fgets COMBINED WITH strtoul/strtod (Takeaway 8.4.5 #1 & 14.2)
  char buffer = {};
  printf("Simulated line input processing...\n");

  // Reading full lines prevents line-desynchronization issues common to scanf
  char const* test_str = "  42000 remaining";
  char* end_ptr = nullptr;

  // strtoul cleanly parses numbers, skips leading whitespace, and reports leftover text
  unsigned long value = strtoul(test_str, &end_ptr, 0);
  if (end_ptr != test_str) {
    printf("Parsed number: %lu, Remaining text suffix: '%s'\n", value, end_ptr);
  }
}
```

### 3. Extended Character Sets & Multibyte Strings (Section 14.3)
To support internationalization beyond ASCII, C provides **multibyte character strings** and **wide characters** (`wchar_t`).

* **Locale Configuration:** Printing and processing multibyte strings correctly requires invoking `setlocale(LC_ALL, "")` at application startup.
* **Properties of Multibyte Strings:**
  * **Takeaway 14.3 #1:** *Multibyte characters don’t contain null bytes* inside their byte sequences.
  * **Takeaway 14.3 #2:** *Multibyte strings are null terminated*.
  * Because multibyte strings contain no internal null bytes, standard string functions like `strcpy` work seamlessly. However, `strlen` returns the raw *byte count*, whereas computing displayed character glyph counts requires `mbsrlen`.
* **Wide Characters:** Wide characters (`wchar_t`, classified via `<wctype.h>`) use fixed-width integer encodings prefixed with `L` (e.g., `L'ä'`, `L"string"`).

#### Example

Standardizing text output for international character sets requires calling `setlocale(LC_ALL, "")` at process startup to match the system's execution locale. **Multibyte strings** represent extended characters using variable-length byte sequences that are null-terminated and contain no internal zero bytes. While standard string functions like `strcpy` work on multibyte strings, `strlen` returns raw byte counts rather than visible displayed character glyphs.

* **Takeaway 14.3 #1:** *Multibyte characters don’t contain null bytes*.
* **Takeaway 14.3 #2:** *Multibyte strings are null terminated*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <locale.h> // For setlocale
#include <wchar.h>  // For wchar_t and mbsrtowcs

void demo_extended_character_sets(void) {
  // Enable system default locale for proper multibyte text rendering (Section 14.3)
  setlocale(LC_ALL, "");

  // Multibyte string containing UTF-8 characters (Takeaway 14.3 #1 & #2)
  char const* mb_str = "C23 Standard: €100";

  // strlen computes total RAW BYTES (including multi-byte UTF-8 sequences)
  size_t byte_count = strlen(mb_str);

  // mbsrtowcs calculates the actual number of WIDE CHARACTERS / displayed glyphs
  mbstate_t state = {};
  char const* src_ptr = mb_str;
  size_t wchar_count = mbsrtowcs(nullptr, &src_ptr, 0, &state);

  printf("String: '%s'\n", mb_str);
  printf("Raw Byte Count (strlen): %zu bytes\n", byte_count);
  printf("Character Count (mbsrtowcs): %zu characters\n", wchar_count);

  // WIDE CHARACTERS (wchar_t) and 'L' prefixed literals
  wchar_t const* wide_str = L"Wide Character Literal: €";
  wprintf(L"%ls\n", wide_str);
}
```

### 4. UTF Encodings & Restartable Conversions (Sections 14.4 & 14.5)
C23 standardizes explicit Unicode UTF encodings and stateful conversion utilities.

* **C23 UTF Types:** `<uchar.h>` defines fixed UTF character types (`char8_t` for UTF-8, `char16_t` for UTF-16, `char32_t` for UTF-32) accompanied by string literal prefixes (`u8"..."`, `u"..."`, `U"..."`).
* **Restartable Conversions (`XXXrtoYYY`):** Functions such as `mbrtoc8` and `c8rtomb` perform restartable conversions between multibyte encodings and UTF characters.
* **State Tracking (`mbstate_t`):**
  * **Takeaway 14.5 #2:** *The multibyte `mb` encoding of a code point may be collected piecewise from the input*.
  * By passing a state tracking object (`mbstate_t*`), restartable functions can process incomplete multibyte character sequences split across buffer boundaries without losing parsing context.

#### Example

C23 defines standardized UTF character types (`char8_t`, `char16_t`, `char32_t`) along with string literal prefixes (`u8"..."`, `u"..."`, `U"..."`). Restartable conversion functions (`mbrtoc8`, `c8rtomb`) use an `mbstate_t` tracking object to convert multibyte text streams piecewise across buffer boundaries without losing context.

* **Takeaway 14.5 #1:** *The multibyte `mb` encoding of a code point is written to the output string all at once*.
* **Takeaway 14.5 #2:** *The multibyte `mb` encoding of a code point may be collected piecewise from the input*.

```c
#include <stdio.h>
#include <uchar.h>  // Provides char8_t, char16_t, char32_t, mbrtoc8
#include <locale.h>

void demo_utf_and_restartable_conversions(void) {
  setlocale(LC_ALL, "");

  // C23 UTF Literal Prefixes (Section 14.4)
  char8_t const*  utf8_str  = u8"UTF-8 Text: 𝜋"; // UTF-8 encoded
  char16_t const* utf16_str = u"UTF-16 Text";   // UTF-16 encoded
  char32_t const* utf32_str = U"UTF-32 Text";   // UTF-32 encoded

  printf("C23 UTF-8 Literal: %s\n", (char const*)utf8_str);

  // RESTARTABLE MULTIBYTE CONVERSION (mbrtoc8)
  char const* input_bytes = "C23 𝜋";
  mbstate_t state = {};
  char8_t out_char = 0;
  size_t bytes_consumed = 0;

  printf("Piecewise UTF-8 Decoding:\n");
  char const* ptr = input_bytes;

  // mbrtoc8 parses code points piecewise using 'state' (Takeaway 14.5 #2)
  while (*ptr) {
    size_t res = mbrtoc8(&out_char, ptr, 4, &state);

    if (res == (size_t)-1 || res == (size_t)-2) {
      puts("Invalid or incomplete multibyte sequence");
      break;
    } else if (res == 0) {
      break; // Null character reached
    }

    printf("  Consumed %zu byte(s) -> Code Point byte: 0x%02X\n", res, (unsigned int)out_char);
    ptr += res;
  }
}
```

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

#### Example

Binary streams transfer uninterpreted raw bytes without the line-ending transformations used in text mode. Functions like `fread`, `fwrite`, `fseek`, and `ftell` handle binary data, though binary files are not inherently portable across different CPU endianness or alignment architectures. In C23, the **`#embed`** preprocessor directive embeds raw binary file contents directly into character array initializers at compile time.

* **Takeaway 14.6 #1:** *Open streams on which you use `fread` or `fwrite` in binary mode*.
* **Takeaway 14.6 #2:** *Files written in binary mode are not portable between platforms*.
* **Takeaway 14.6 #3:** *`fseek` and `ftell` are not suitable for very large file offsets*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>

void demo_binary_streams_and_embed(void) {
  // 1. BINARY STREAM I/O (Takeaway 14.6 #1)
  char const* filename = "data.bin";
  double dataset = {1.1, 2.2, 3.3, 4.4};

  // Open file in BINARY WRITE mode ("wb") to bypass line-ending translations
  FILE* out_stream = fopen(filename, "wb");
  if (out_stream) {
    // Write raw object bytes using fwrite
    size_t written = fwrite(dataset, sizeof(double), 4, out_stream);
    printf("Written %zu binary items to %s\n", written, filename);
    fclose(out_stream);
  }

  // Open file in BINARY READ mode ("rb")
  FILE* in_stream = fopen(filename, "rb");
  if (in_stream) {
    // Seek to end to calculate byte size using ftell (Takeaway 14.6 #3)
    fseek(in_stream, 0, SEEK_END);
    long file_size = ftell(in_stream);
    rewind(in_stream);

    printf("Binary file size: %ld bytes\n", file_size);

    double read_buf = {};
    fread(read_buf, sizeof(double), 4, in_stream);
    printf("Read element 0: %g\n", read_buf);

    fclose(in_stream);
    remove(filename); // Clean up temporary binary file
  }

  // 2. C23 RESOURCE EMBEDDING (#embed directive) (Section 14.6)
  // Guards embed directive using __has_embed
  unsigned char const embedded_resource[] = {
#if defined(__has_embed) && __has_embed("data.bin")
    #embed "data.bin"
#else
    0x00, 0x01, 0x02, 0x03 // Fallback byte array when file is missing
#endif
  };

  printf("Embedded binary resource size: %zu bytes\n", sizeof embedded_resource);
}
```
