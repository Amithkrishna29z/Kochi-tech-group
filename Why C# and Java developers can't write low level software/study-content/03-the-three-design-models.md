# The Three Design Models for Language VMs

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

The central framework of the talk: software for the JVM/CLR can be designed in one of three modes. Most developers only ever use the first and assume it is the whole language. The other two are what make Kafka-class and proxy/trading-class software possible.

---

## At a glance

| Dimension | Design for Garbage Collection | Design for Lowering at the Data Level | Design for Super-Fast Execution |
|---|---|---|---|
| Goal | Developer productivity, maintainable business logic | Correct binary interop with C-shaped wire formats | Minimal, deterministic latency |
| Typical domain | Coarse-grained enterprise/business apps | Network protocols, wire formats, serialization | Trading engines, raw reverse proxies, firewalls |
| Memory posture | Rely on the GC; use Dispose/finalizers as discipline | Precise byte layout; length-prefixed data | Avoid GC entirely; off-heap allocation |
| Key techniques | Idiomatic value/reference types, collections, LINQ | Packed structs, explicit layout, endianness, null-terminated strings, varint parsing | Off-heap + AOT, JIT-friendly code patterns, `stackalloc`, `unsafe` |
| Concurrency style | async/await, tasks, coarse-grained | Fine-grained, follows the protocol's synchronization | Explicit threading, declarative concurrency |
| C# tools | Core idioms | `struct`, `StructLayout`/`FieldOffset`, `ByteBuffer`-equivalents | `unsafe`, `stackalloc`, P/Invoke, COM interop, value types |
| Java tools | Core idioms | `ByteBuffer`, FFM API, (historically JNA); Valhalla incoming | Off-heap buffers, JIT-recognized patterns, FFM |
| Exemplar systems | Typical Spring Boot DB app | Kafka wire format, ClickHouse protocol, Avro/Protobuf/Thrift | YARP, trading systems, (hypothetically) a C# firewall |
| Main pitfall | Doesn't transfer to protocols — they desync and break | Java/C# guarantee no default/declaration-order layout | Fighting the enterprise mindset; whole-program restructuring needed |
| Verification | Business tests | Byte-for-byte compatibility with a C receiver | Latency/pause measurement; empirical, a-posteriori |

---

## 1. Design for Garbage Collection — the default model

If you write Java or .NET using the language's core idioms — the default, most-used, abstraction-first features — you are already in this model, whether you named it or not. It is the correct default for the bulk of software: coarse-grained business and enterprise applications, the archetype being a typical Spring Boot database application.

This model is not "wrong" — it is right for its domain, and it carries its own disciplines:

- Choosing value types vs reference types deliberately.
- Managing collections (sizing, copying, churn).
- Deciding whether to implement the Dispose pattern and finalizers for unmanaged resources.

When to use it: business logic, CRUD services, orchestration, anything where developer productivity and maintainability dominate and where GC pauses are irrelevant to correctness.

The pitfall: this attitude does not transfer to protocol work. If you design protocol packets while thinking in coarse-grained enterprise idioms, the protocol will not stay synchronized and the code will not work correctly. That failure is the bridge to the second model.

---

## 2. Design for Lowering at the Data Level — the C-bias model

Core insight: almost every network protocol and wire format has a C bias. This is not an accident. C defines an abstract machine; Unix (the first OS in a high-level language) was written in C as a "high-level assembler" for the PDP; and ever since, every processor architecture has been designed to be a good target for C-generated code. The C abstract machine is therefore the de-facto default architecture, and wire formats inherit its data conventions.

To design at this level you must respect the four data-aspect rules of the C abstract machine:

1. Structure layout / declaration order — fields are laid out in the order declared; packets must match C/C++ struct layout.
2. Byte alignment — protocols almost always require 1-byte (packed) alignment, unlike the default 4-byte word alignment; designers then add explicit reserved padding bytes to keep multi-byte fields on natural boundaries.
3. Endianness — little- vs big-endian byte order must be honored on the wire.
4. Null-terminated strings — C strings end in a zero byte; a Java/C# `String` is a length-carrying compound object and must be converted to bytes with an explicit terminating zero.

The core difficulty: Java and C# classes guarantee neither a default layout nor declaration-order field allocation. So a naive FTP-style transfer of a binary file plus its metadata from Java/.NET breaks against a C-struct-shaped receiver.

How the platforms cope:

- Java — `ByteBuffer` for hand-built binary data; historically JNA/JNI for native details; now the FFM API to describe memory layouts programmatically (not declaratively at the syntax level). Project Valhalla's value types are expected to close the gap with C#.
- C# — `struct` plus `StructLayout`/`FieldOffset` to pin field order and offsets exactly as C++ expects, with default packing set to 1 — producing C-compatible data structures. This is C#'s current edge.

Two shapes of protocol you will meet:

- Fixed-length FSM protocols (Postgres, MySQL) — fixed-size packets, easier to interoperate with.
- Variable-length / compressed protocols (ClickHouse) — use varint encoding: each byte's leading bit is a continuation flag, so values 0–127 take one byte and only large values grow. This saves enormous space across millions of records but cannot be laid out as a struct — you must parse it byte-by-byte, and you must diff the vendor's parsing source against every release to keep a proxy compatible.

A key consequence: pointers never appear inside packets, because a pointer is meaningless on the receiving machine; variable-length data is always length-prefixed instead.

Exemplars: Kafka's wire format, ClickHouse's protocol, and the Avro/Protobuf/Thrift serialization formats. Spark was where off-heap allocation was first seen in the wild.

Conclusion: as JITs got better at producing correct data layouts, you can now write most low-level code in pure Java or C#/CLR at this level — rather than dropping to JNA/P-Invoke.

---

## 3. Design for Super-Fast Execution — the low-latency model

For low-latency systems (trading, proxies), two rules govern everything:

1. Do not trigger garbage collection.
2. Leverage the JIT better.

Avoiding GC. The naive first instinct is to allocate large chunks up front so generational GC promotes them into the old-generation pool where they sit untouched. The better answer is off-heap allocation (available in both C# and Java): manage one large memory area as a unit and take responsibility for sub-allocation yourself, so the collector never touches it.

Leveraging the JIT. The philosophy is: keep language constructs simple so the JIT can generate better object code. Concretely:

- Between two programs using reference types, Java is often faster than C# because simple constructs let HotSpot optimize aggressively; C#'s advantage is `struct`/explicit layout (which Valhalla should neutralize for Java).
- Guide the compiler rather than hope. For a single task there may be ~40 candidate code patterns; the JIT optimizes only some of them well. A knowledgeable programmer chooses the pattern the JIT maps to the best instruction sequence — e.g., writing the exact pattern the JIT lowers to the `BSWAP` byte-swap instruction instead of calling native code. There are canonical code-pattern → instruction mappings; learn them. User judgment beats compiler guessing when competing patterns exist.

The mindset fight. This model is a fight against the enterprise computing mindset. Enterprise programmers optimize at the level of "use better data structures/algorithms, reduce allocations." Squeezing the next level requires restructuring the whole program around the C abstract machine's data variant and control variant — stack allocation, subroutine calls, pointer passing.

Where determinism comes from. AOT compilation improves optimization and startup but does not remove the GC and offers no re-JIT. Full determinism therefore requires AOT + off-heap allocation. With that combination you could in principle write even a firewall in C# — the honest open question being whether C#/Java is the natural language for it. YARP shows the shape of the answer: "pure C#, but not enterprise C#" — `unsafe` blocks, value types, COM interop, careful patterns, effectively a C compiler hiding inside `unsafe`.

C# features to study for this model: value types (structs), `unsafe` blocks, `stackalloc`, P/Invoke, COM interop, explicit layout attributes.

Verification note: rational design assumptions are never sufficient here — you must verify empirically, a posteriori ("the proof of the pudding is in the eating"): measure latency and pauses on the real workload.

---

## How the three models relate

- Model 1 is the default and the majority of code. Models 2 and 3 are specializations you drop into only where the domain demands them.
- Model 2 (data-level) is a prerequisite for Model 3 (speed): you cannot build low-latency protocol software without first getting the byte layout right.
- Both 2 and 3 ultimately rest on the C abstract machine — data aspect for Model 2, data + control aspects for Model 3 — which is why understanding that machine is the master key for every VM-hosted language, not just C#/Java.
