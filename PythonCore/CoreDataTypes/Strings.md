# PythonCore → Datatypes → Strings

# 1️⃣ Conceptual Model

Python 3 `str` is:

> Immutable sequence of Unicode code points.

It is NOT: - ASCII - UTF-8 - UTF-16

Internally it uses a flexible representation (PEP 393).

------------------------------------------------------------------------

# 2️⃣ CPython Internal Representation (PEP 393)

Defined in:

-   `Include/unicodeobject.h`
-   `Objects/unicodeobject.c`

Python 3.3 introduced Flexible String Representation.

Three layouts:

-   PyASCIIObject
-   PyCompactUnicodeObject
-   Full PyUnicodeObject

The layout depends on content.

------------------------------------------------------------------------

# 3️⃣ Flexible Storage Strategy

Python dynamically selects storage width:

  Max Code Point   Storage           Bytes per Char
  ---------------- ----------------- ----------------
  ≤ 0x7F           ASCII / Latin-1   1 byte
  ≤ 0xFF           UCS1              1 byte
  ≤ 0xFFFF         UCS2              2 bytes
  otherwise        UCS4              4 bytes

Stored in `state.kind`.

Implication:

ASCII string → 1 byte per char\
Emoji string → 4 bytes per char

Memory optimized per string.

------------------------------------------------------------------------

# 4️⃣ Memory Layout

Compact ASCII string:

\[Header\]\[Characters inline\]

Benefits: - No extra allocation - Better cache locality - Reduced memory
fragmentation

------------------------------------------------------------------------

# 5️⃣ Immutability Mechanics

Strings are immutable.

Example:

``` python
s = "abc"
s += "d"
```

Creates a new object.

No in-place modification occurs.

------------------------------------------------------------------------

# 6️⃣ String Interning

Python interns:

-   Identifiers
-   Some literals
-   Small constant strings

Example:

``` python
a = "hello"
b = "hello"
a is b  # Often True
```

Manual interning:

``` python
import sys
sys.intern("some_string")
```

Critical for: - Dictionary key performance - Compiler symbol tables

------------------------------------------------------------------------

# 7️⃣ Unicode Complexity

Code point ≠ glyph ≠ byte.

Example:

``` python
len("é")   # 1
len("é")  # 2 (combining accent)
```

Normalization matters:

-   NFC
-   NFD
-   NFKC
-   NFKD

------------------------------------------------------------------------

# 8️⃣ Encoding / Decoding

`str` ↔ `bytes`

Encoding:

``` python
"hello".encode("utf-8")
```

Decoding:

``` python
b"hello".decode("utf-8")
```

Errors:

-   strict
-   ignore
-   replace
-   backslashreplace

UTF-8 is variable-width encoding.

------------------------------------------------------------------------

# 9️⃣ f-Strings Internals

f-strings are compiled at parse time.

Example:

``` python
f"{x + 2}"
```

Converted into:

-   AST nodes
-   Runtime expression evaluation
-   Format protocol calls

Equivalent to:

``` python
format(x + 2)
```

No runtime parsing of template string.

------------------------------------------------------------------------

# 🔟 String Performance Engineering

### ❌ Bad

``` python
s = ""
for i in range(100000):
    s += str(i)
```

O(n²) behavior.

### ✅ Good

``` python
parts = []
for i in range(100000):
    parts.append(str(i))
result = "".join(parts)
```

O(n) behavior.

------------------------------------------------------------------------

# Memory Summary (64-bit CPython Approximate)

  Type            Memory
  --------------- ------------
  Empty string    \~49 bytes
  10-char ASCII   \~59 bytes
  10-char Emoji   \~89 bytes

Numbers vary by build.

------------------------------------------------------------------------

# Production Engineering Patterns

Use:

-   `str` for application-level text
-   `bytes` for I/O boundaries
-   Normalize user input when comparing Unicode
-   Avoid repeated concatenation
-   Use `.join()` for large builds

------------------------------------------------------------------------

# Key Takeaways

-   Strings use flexible internal storage (PEP 393)
-   Memory per character depends on max code point
-   Immutable → concatenation is expensive
-   Interning improves dictionary performance
-   Encoding boundary must be explicit

------------------------------------------------------------------------

This file provides systems-level understanding of Python string
internals.
