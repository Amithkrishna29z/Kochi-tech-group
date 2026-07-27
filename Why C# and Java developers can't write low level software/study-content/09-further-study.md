# Further Study — Prioritized Reading & Practice

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

A prioritized path derived directly from the talk. Tiers reflect what unlocks the most: foundations first, then the platform-specific tooling, then the systems and theory.

---

## Tier 1 — Foundations: the C abstract machine (do this first)

The talk's thesis is that this layer is the master key. Everything else builds on it.

- The required curriculum, verbatim from the talk: bit, nibble, byte, word, dword; integer encoding (two's complement) and IEEE-754 floating point; bit patterns and casting; struct layout; little-endian vs big-endian.
- CS:APP-style material — Bryant & O'Hallaron, Computer Systems: A Programmer's Perspective. Chapters on data representation (integers, floats, byte ordering), machine-level representation, and the memory hierarchy map exactly onto the "data aspect" and "control aspect" of the C abstract machine.
- Practice: hand-encode/decode a packed C struct on paper, then verify in code (Lab A/B). Do the same for an IEEE-754 float. Insist on empirical verification — "the proof of the pudding is in the eating."

Success check: given a C `#pragma pack(1)` struct, you can predict its byte layout, size, and wire bytes before running it.

---

## Tier 2 — The four interop rules in practice (managed platforms)

Java side
- Java `ByteBuffer` (order, position, put/get) — the workhorse for hand-built binary data.
- The FFM (Foreign Function & Memory) API under Project Panama — `MemoryLayout`, `MemorySegment`, `VarHandle`, `Arena`; describing C-compatible layouts programmatically.
- Project Valhalla — value types for Java; understand what it changes and why it is expected to neutralize C#'s struct edge.

C# side
- `StructLayout`, `FieldOffset`, `LayoutKind.Sequential/Explicit`, `Pack = 1` — producing C-compatible data structures.
- `unsafe`, pointers, `stackalloc`, `MemoryMarshal`/`Span<T>`, `NativeMemory` — the low-level toolkit.
- P/Invoke and COM interop — calling native code when needed.

Practice: complete Labs A–D (packed struct in Java and C#, varint encode/decode, null-terminated string round-trip to a C receiver).

---

## Tier 3 — JIT, AOT, and determinism

- .NET NativeAOT docs — confirm the two corrections: no re-JIT (IL not in the binary) and the GC still present. Note trimming/startup benefits.
- HotSpot / tiered compilation and ReJIT — how profiling drives hot-spot compilation and re-optimization.
- GraalVM native image — capabilities, licensing caveats, Spring native-image support; open question: does Kafka use Graal?
- Off-heap allocation — Java `ByteBuffer.allocateDirect` / FFM `Arena`; .NET `NativeMemory`. Understand why determinism = AOT + off-heap.

Practice: complete Lab E (off-heap arena in both languages); measure GC behavior with and without it.

---

## Tier 4 — Real systems (read the source)

- Kafka and Spark internals — wire formats and off-heap memory management; Spark is where off-heap was first seen in the wild.
- Serialization formats — Avro, Protocol Buffers, Thrift: how each reflects C-bias data conventions and varint-style encodings.
- ClickHouse wire protocol — fixed strings with lengths plus varint compression; study the parsing source and the "proxy only up to revision N" problem. Contrast with Postgres/MySQL fixed-length FSM protocols.
- YARP source — the concrete example of "pure C#, but not enterprise C#": `unsafe`, value types, COM interop, careful patterns.
- Historical context: IIS (C++) + HTTP.sys kernel driver; Tomcat vs JBoss application-server tiers.

---

## Tier 5 — Language design perspective

- Algebraic data types — how record types + union types + pattern matching converge to F#-style ADTs; compare C#/Java progress.
- Go's parsimony — study Go as a deliberate counterpoint to feature accumulation.
- C-- and GHC — how GHC type-checks Haskell then compiles via C-- (portable assembly); the Eta project's JVM-bytecode backend. Useful for understanding "target x86 assembly or C-- suffices."

---

## Tier 6 — Models of computation & quantum (research thread)

- Machine models — Turing machine → von Neumann (sequential; the C abstract machine's ancestor); Harvard architecture (GPU); CPU/GPU/TPU as scalar-vector / matrices / stack-of-matrices.
- Horowitz–Sahni–Rajasekaran, Fundamentals of Computer Algorithms — the parallel machine-models chapter (RAM, PRAM, hypercube/PRAM variants).
- Quantum overview — Q# and the quantum development kit; open questions on a quantum PRAM, a quantum instruction set, and whether quantum arrives as a coprocessor. Consider whether OpenMP-style pragmas, rather than new languages, will carry adoption.
- Action items from the talk: survey current quantum results/models (a focused week); prepare to "grill" a knowledgeable guest on the computational aspects.

---

## Suggested sequence

1. Tier 1 until you can predict struct/float byte layouts unaided.
2. Tier 2 Labs A–D on your primary platform, then the other platform.
3. Tier 3 + Lab E; internalize determinism = AOT + off-heap.
4. Tier 4 — read one real protocol end-to-end (ClickHouse or Kafka).
5. Tiers 5–6 as breadth/perspective, at your own pace.
