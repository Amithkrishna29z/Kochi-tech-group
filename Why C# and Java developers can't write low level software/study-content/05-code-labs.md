# Code Labs — Low-Level Design on the JVM and CLR

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

Hands-on exercises with worked solutions. Each lab targets one of the four interop rules or the GC-avoidance technique from the talk. Code is illustrative and compiles against modern Java (21+, FFM API) and .NET (8+).

The running example struct (a file-transfer metadata header) matches the war story in the source: a C-shaped, 1-byte-packed PDU.

```c
// The C definition every managed implementation must match, byte-for-byte.
#pragma pack(1)
struct FileHeader {
    unsigned char  msgType;      // 1 byte
    unsigned char  reserved1;    // explicit padding to realign fileSize
    unsigned char  reserved2;
    unsigned char  reserved3;
    unsigned int   fileSize;     // 4 bytes, now at offset 4 (natural)
    // followed by: null-terminated file name (variable length)
};
```

---

## Lab A — Java: pack a C-compatible struct with ByteBuffer + FFM

Goal: emit exactly 8 header bytes in declaration order, little-endian, 1-byte packed.

Verify: the emitted bytes equal `01 00 00 00 2A 00 00 00` for `msgType=1, fileSize=42`.

### A.1 ByteBuffer solution

```java
import java.nio.ByteBuffer;
import java.nio.ByteOrder;

public class HeaderByteBuffer {
    static byte[] encode(int msgType, int fileSize) {
        ByteBuffer buf = ByteBuffer.allocate(8).order(ByteOrder.LITTLE_ENDIAN);
        buf.put((byte) msgType);   // offset 0
        buf.put((byte) 0);         // reserved1
        buf.put((byte) 0);         // reserved2
        buf.put((byte) 0);         // reserved3
        buf.putInt(fileSize);      // offset 4, 4 bytes
        return buf.array();
    }
    public static void main(String[] a) {
        byte[] b = encode(1, 42);
        for (byte x : b) System.out.printf("%02X ", x);   // 01 00 00 00 2A 00 00 00
    }
}
```

### A.2 FFM API solution (layout described programmatically)

The FFM API lets you describe the layout explicitly — but note the limitation from the talk: Java cannot declare layout at the language-syntax level, so you build it programmatically.

```java
import java.lang.foreign.*;
import java.lang.invoke.VarHandle;

public class HeaderFFM {
    static final StructLayout LAYOUT = MemoryLayout.structLayout(
        ValueLayout.JAVA_BYTE.withName("msgType"),
        ValueLayout.JAVA_BYTE.withName("reserved1"),
        ValueLayout.JAVA_BYTE.withName("reserved2"),
        ValueLayout.JAVA_BYTE.withName("reserved3"),
        ValueLayout.JAVA_INT.withOrder(java.nio.ByteOrder.LITTLE_ENDIAN).withName("fileSize")
    ); // total byteSize() == 8, no hidden padding

    static final VarHandle MSG  = LAYOUT.varHandle(MemoryLayout.PathElement.groupElement("msgType"));
    static final VarHandle SIZE = LAYOUT.varHandle(MemoryLayout.PathElement.groupElement("fileSize"));

    public static void main(String[] a) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment seg = arena.allocate(LAYOUT);   // off-heap, native memory
            MSG.set(seg, 0L, (byte) 1);
            SIZE.set(seg, 0L, 42);
            byte[] bytes = seg.toArray(ValueLayout.JAVA_BYTE);
            for (byte x : bytes) System.out.printf("%02X ", x);
        }
    }
}
```

Key point: `LAYOUT.byteSize()` is 8 because the reserved bytes we declared realign `fileSize` to offset 4 — the FFM layout carries no automatic padding of its own here.

---

## Lab B — C#: the same struct with StructLayout/FieldOffset + unsafe

Goal: produce the identical 8 bytes using C#'s declarative layout control — its current edge over Java.

Verify: `01 00 00 00 2A 00 00 00`.

### B.1 Attribute-driven layout

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Explicit, Size = 8)]
struct FileHeader
{
    [FieldOffset(0)] public byte MsgType;
    // offsets 1..3 are the reserved padding bytes (left as zero)
    [FieldOffset(4)] public uint FileSize;   // C# defaults to little-endian on x86
}

class Program
{
    static byte[] Encode(in FileHeader h)
    {
        byte[] buf = new byte[8];
        MemoryMarshal.Write(buf, in h);   // blits the struct's exact bytes
        return buf;
    }
    static void Main()
    {
        var h = new FileHeader { MsgType = 1, FileSize = 42 };
        foreach (var b in Encode(h)) System.Console.Write($"{b:X2} ");
    }
}
```

`LayoutKind.Explicit` + `FieldOffset` pins each field to the offset C++ expects. `LayoutKind.Sequential, Pack = 1` is the alternative when you want declaration-order packing without hand-writing every offset.

### B.2 unsafe / pointer equivalent

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct FileHeaderSeq
{
    public byte MsgType, Reserved1, Reserved2, Reserved3;
    public uint FileSize;
}

class ProgramUnsafe
{
    static unsafe void Main()
    {
        var h = new FileHeaderSeq { MsgType = 1, FileSize = 42 };
        byte* p = (byte*)&h;                 // "a C compiler hiding inside unsafe"
        for (int i = 0; i < sizeof(FileHeaderSeq); i++)
            System.Console.Write($"{p[i]:X2} ");
    }
}
```

`sizeof(FileHeaderSeq)` is 8 (not 5) precisely because we declared three reserved bytes to realign `FileSize`.

---

## Lab C — Varint (ClickHouse-style) encode/decode in Java and C#

Goal: implement the continuation-bit varint from the talk — MSB is the continuation flag, 7 data bits per byte.

Verify:
- `100` → `64` (1 byte)
- `300` → `AC 02` (2 bytes)
- round-trip: `decode(encode(x)) == x` for a range of values.

### C.1 Java

```java
import java.util.*;

class Varint {
    static byte[] encode(long value) {
        var out = new ArrayList<Byte>();
        long v = value;
        do {
            int b = (int) (v & 0x7F);   // low 7 bits
            v >>>= 7;
            if (v != 0) b |= 0x80;      // set continuation bit
            out.add((byte) b);
        } while (v != 0);
        byte[] r = new byte[out.size()];
        for (int i = 0; i < r.length; i++) r[i] = out.get(i);
        return r;
    }

    static long decode(byte[] bytes) {
        long value = 0; int shift = 0;
        for (byte b : bytes) {
            value |= (long) (b & 0x7F) << shift;
            if ((b & 0x80) == 0) break;   // continuation clear -> last byte
            shift += 7;
        }
        return value;
    }

    public static void main(String[] a) {
        for (byte x : encode(100)) System.out.printf("%02X ", x); // 64
        System.out.println();
        for (byte x : encode(300)) System.out.printf("%02X ", x); // AC 02
        System.out.println();
        System.out.println(decode(encode(123456789L)));           // 123456789
    }
}
```

### C.2 C#

```csharp
using System.Collections.Generic;

static class Varint
{
    public static byte[] Encode(ulong value)
    {
        var outBytes = new List<byte>();
        do {
            byte b = (byte)(value & 0x7F);
            value >>= 7;
            if (value != 0) b |= 0x80;
            outBytes.Add(b);
        } while (value != 0);
        return outBytes.ToArray();
    }

    public static ulong Decode(byte[] bytes)
    {
        ulong value = 0; int shift = 0;
        foreach (byte b in bytes) {
            value |= (ulong)(b & 0x7F) << shift;
            if ((b & 0x80) == 0) break;
            shift += 7;
        }
        return value;
    }
}
```

Takeaway: a varint cannot be modeled as a fixed struct — you must parse it byte-by-byte. To build a compatible proxy you diff the vendor's parser source against each release ("proxy only up to revision N").

---

## Lab D — Null-terminated string round-trip: C# sender → C receiver

Goal: send a file name so a C receiver reading `char name[]` until `\0` gets the right string.

Verify: the emitted bytes for `"log.txt"` are `6C 6F 67 2E 74 78 74 00` (7 chars + terminator).

### D.1 C# sender

```csharp
using System.Text;

static byte[] ToCString(string s)
{
    byte[] body = Encoding.ASCII.GetBytes(s);   // String is a compound object; flatten it
    byte[] buf = new byte[body.Length + 1];     // room for the terminator
    System.Array.Copy(body, buf, body.Length);
    buf[^1] = 0x00;                             // explicit null terminator
    return buf;
}
```

### D.2 C receiver (reference)

```c
// reads a null-terminated name straight into a char buffer
size_t read_name(const unsigned char *wire, char *out) {
    size_t i = 0;
    while (wire[i] != 0x00) { out[i] = (char) wire[i]; i++; }
    out[i] = '\0';
    return i;   // length, excluding terminator
}
```

### D.3 Java equivalent sender

```java
static byte[] toCString(String s) {
    byte[] body = s.getBytes(java.nio.charset.StandardCharsets.US_ASCII);
    byte[] buf = new byte[body.length + 1];   // last byte defaults to 0 in Java
    System.arraycopy(body, 0, buf, 0, body.length);
    return buf;
}
```

Takeaway: "my C# is not your C#." A managed `String` carries a length and Unicode data; the wire wants raw bytes + a zero terminator, and you must add it yourself.

---

## Lab E — GC avoidance via off-heap allocation

Goal: allocate a large buffer the garbage collector will never scan, and sub-allocate from it yourself.

Verify: allocations/reads happen with no managed-heap growth attributable to the buffer (inspect with a profiler / GC logs; the point is that the region is outside the GC's reach).

### E.1 C# — NativeMemory (off-heap)

```csharp
using System.Runtime.InteropServices;

unsafe class OffHeapArena
{
    readonly byte* _base;
    readonly nuint _size;
    nuint _offset;

    public OffHeapArena(nuint size)
    {
        _base = (byte*)NativeMemory.Alloc(size);   // off the GC heap
        _size = size;
    }

    // programmer-managed sub-allocation ("manage one large area as a unit")
    public byte* Rent(nuint n)
    {
        if (_offset + n > _size) throw new System.OutOfMemoryException();
        byte* p = _base + _offset;
        _offset += n;
        return p;
    }

    public void Dispose() => NativeMemory.Free(_base);
}
```

### E.2 Java — direct (off-heap) ByteBuffer / FFM Arena

```java
import java.nio.ByteBuffer;

// classic off-heap: not on the Java managed heap, not scanned/moved by GC
ByteBuffer offHeap = ByteBuffer.allocateDirect(64 * 1024 * 1024);

// modern FFM equivalent: a shared arena holding native memory
// try (java.lang.foreign.Arena arena = java.lang.foreign.Arena.ofShared()) {
//     var seg = arena.allocate(64L * 1024 * 1024);
//     ... sub-allocate by slicing the segment yourself ...
// }
```

Takeaways from the talk:
- The naive route is to allocate large chunks up front so generational GC promotes them to old-gen where they sit untouched.
- The better route is off-heap allocation (shown here) — the collector never touches this memory at all.
- For full determinism, combine this with AOT compilation (NativeAOT / GraalVM native image): AOT alone does not remove GC.
