# The C Abstract Machine — One-Page Cheat Sheet

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

The master key. Every real processor is designed to be a good target for C, so the JIT ultimately runs on top of the C abstract machine. Understand this and you can write low-level code in any VM-hosted language.

---

## Data widths

```
bit     1 bit                         smallest unit (0/1)
nibble  4 bits                        one hex digit (0x0–0xF)
byte    8 bits                        0–255 unsigned; two nibbles
word    16 bits (classic)             two bytes
dword   32 bits (double word)         four bytes
qword   64 bits (quad word)           eight bytes
```

Word alignment: CPUs prefer multi-byte values placed at offsets that are multiples of the word size (commonly 4 or 8). This is why protocol/graphics dimensions gravitate to powers of two.

---

## Integer encoding

```
unsigned byte 0xB4 = 1011 0100 = 180
signed byte (two's complement): high bit = sign
  0xB4 = 1011 0100 -> negative -> -(~0xB4 + 1) = -76
```

Two's complement is the standard signed-integer representation: negate by inverting all bits and adding 1.

---

## IEEE 754 floating point (single precision, 32-bit)

```
[ sign : 1 ][ exponent : 8 ][ mantissa/fraction : 23 ]
 bit 31       bits 30..23       bits 22..0

value = (-1)^sign x 1.mantissa x 2^(exponent - 127)
```

Double precision (64-bit): 1 sign, 11 exponent (bias 1023), 52 mantissa. Know that floats are not laid out like integers — decode field-by-field when parsing wire data.

---

## Struct layout and packing

Rule: C/C++ lay out struct fields in declaration order. Java/C# classes guarantee neither default layout nor declaration order — this is the root of the interop problem.

```
struct S { char a; int b; };

Default (pack = 4)               Packed  #pragma pack(1)
+----+----+----+----+            +----+----+----+----+----+
| a  | .. | .. | .. |  <- 3 pad  | a  | b0 | b1 | b2 | b3 |
+----+----+----+----+            +----+----+----+----+----+
| b0 | b1 | b2 | b3 |            = 5 bytes, b unaligned
+----+----+----+----+
= 8 bytes, b aligned

Designer's fix under pack(1): insert explicit reserved bytes
struct S { char a; char r1; char r2; char r3; int b; };  // b realigned to offset 4
```

- Enable packing: `#pragma pack(1)` or `__attribute__((packed))` (GCC).
- Protocols almost always require 1-byte alignment (no hidden padding on the wire).

---

## Endianness

```
value 0x11223344 in memory / on the wire:

little-endian:  44 33 22 11     (least-significant byte first; x86)
big-endian:     11 22 33 44     (most-significant first; "network order")
```

Convert with the `BSWAP` instruction. In managed code, write the pattern the JIT lowers to `BSWAP` rather than calling native code.

---

## Null-terminated strings

```
C string "Hi":  48 69 00        (bytes + terminating zero)

Java/C# String is a compound object (length + Unicode data),
NOT a bare null-terminated byte run. To send it to a C receiver:
  1. encode to bytes (e.g., UTF-8/ASCII)
  2. append an explicit 0x00 terminator
```

---

## Length-prefixed data & varints

Pointers never appear in packets (meaningless on the other machine). Variable-length data is length-prefixed. For compression, protocols use varints:

```
varint byte = [ C | 7 data bits ]     C = continuation (MSB)
  C = 0  -> last byte
  C = 1  -> more bytes follow

0..127     -> 1 byte      (covers most real values -> huge savings)
128..16383 -> 2 bytes     ...

Cannot be a struct: must be PARSED byte-by-byte.
```

---

## Why protocols have a C bias

- Unix, the first OS in a high-level language, was written in C (a "high-level assembler" for the PDP).
- Every processor since has been designed to run C code well → the C abstract machine is the default architecture.
- Wire formats and RFCs therefore assume C data conventions.

---

## The four interop rules (memorize)

1. Declaration-order layout — match the C/C++ struct field order exactly.
2. 1-byte alignment — pack the struct; add explicit reserved bytes where a field must realign.
3. Endianness — convert to the protocol's byte order on the wire.
4. Null-terminated strings — encode to bytes and append the terminating zero.

Managed tooling: C# — `struct` + `StructLayout`/`FieldOffset`, packing = 1, `unsafe`, `stackalloc`. Java — `ByteBuffer`, FFM API (programmatic layout); Project Valhalla to come.

---

## Two aspects of the machine

- Data aspect — struct layout, alignment, endianness, strings. (Needed for protocol design.)
- Control aspect — subroutines, the stack, calling conventions, pointer passing. (Needed for the last level of performance: restructure the program around it.)

Final reminder: verify empirically. Rational assumptions are not enough — measure. "The proof of the pudding is in the eating."
