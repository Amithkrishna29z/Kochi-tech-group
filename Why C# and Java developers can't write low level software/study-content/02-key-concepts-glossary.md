# Key Concepts Glossary

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

Definitions for every technical term in the discussion. Each entry is 2–4 sentences and self-contained.

---

Linguistic (language-level) virtual machine — A process VM that executes bytecode/IL emitted by a language compiler rather than virtualizing hardware. The JVM and the CLR are the two canonical examples; both are now polyglot, hosting many languages beyond Java and C#.

Platform VM (hypervisor VM) — A virtual machine created by a Type 1 (bare-metal) or Type 2 (hosted) hypervisor that virtualizes an entire machine. These became prominent with the rise of cloud computing.

Container — Process-level isolation (Docker and similar). Because a container runs on top of a lightweight kernel, it can be regarded as a lightweight virtual machine, though it is not a full platform VM.

CLR (Common Language Runtime) — Microsoft's language-level VM, introduced around 2002, built primarily to run C# but also hosting F#, IronPython, IronRuby and others. It executes IL and JIT-compiles it to native code.

JVM (Java Virtual Machine) — The language-level VM built to run Java, now also hosting Clojure, Scala, Groovy and more. It executes Java bytecode and JIT-compiles hot code to native.

IL / bytecode — The intermediate instruction format a language compiler emits for a VM (.NET IL; Java bytecode). The VM later interprets and/or JIT-compiles it to native machine code.

Managed memory — Microsoft's term for automatic memory management: the runtime allocates objects, tracks their scope, and a background garbage collector reclaims unused memory. Summarized by the mantra reduce, reclaim, reuse.

Garbage collection (GC) — Automatic reclamation of memory no longer reachable by the program, removing the need for manual free/delete. It is the defining advantage managed VMs had over C++, but its non-deterministic pauses make it problematic for low-latency systems.

Reference counting — A manual-ish memory scheme where each object tracks how many references point to it and is freed when the count hits zero. Used by Objective-C.

ARC (Automatic Reference Counting) — Swift's model in which the compiler automatically inserts retain/release calls. It automates the bookkeeping but the underlying model is still essentially manual memory management, not tracing GC.

HotSpot — Java's mature JIT/optimization technology. It interprets code, profiles which spots are "hot," JIT-compiles those, and can re-optimize (re-JIT) as profiles change.

JIT (just-in-time compilation) — Compiling bytecode/IL to native code at run time. Because it has runtime context information, a JIT can sometimes produce better code than ahead-of-time compilation.

ReJIT — .NET's ability to re-compile a method after initial JIT, typically to apply deeper optimization to performance-critical methods based on profiling. HotSpot has an analogous re-JIT capability.

Tiered compilation — .NET's default strategy: a quick, low-optimization JIT at load/run time, followed by ReJIT of hot methods at higher optimization. Balances fast startup against peak throughput.

AOT (ahead-of-time compilation) — Compiling to native code before run time. It can yield strong optimization and fast, deterministic startup, but forfeits runtime re-optimization.

NativeAOT (.NET) — .NET's ahead-of-time native compiler. The IL is not embedded in the produced EXE (unless kept as debug info), so there is no re-JIT; crucially, the garbage collector is still present.

GraalVM — A JVM-side option for AOT "native image" compilation, considered the best current choice for AOT on the JVM. It is not mainstream and parts are proprietary / require a license.

Native image — An ahead-of-time-compiled, self-contained native executable (e.g., produced by GraalVM), offering fast startup and determinism at the cost of runtime re-optimization.

Off-heap allocation — Allocating memory outside the GC-managed heap so the collector never scans or moves it. Available in both Java and C#; combined with AOT it is the path to full determinism. Spark was an early real-world user.

Determinism (in this context) — Predictable, pause-free execution timing, required by low-latency systems such as trading engines and raw reverse proxies. Achieved by AOT + off-heap allocation, since AOT alone still leaves GC in place.

C abstract machine — The conceptual machine model defined by the C language (data + control aspects). Because Unix and later OSes were written in C, processors have been designed as good targets for C code, making this model the de-facto default architecture the JIT ultimately runs on.

Data aspect (of the C abstract machine) — Struct layout in declaration order, byte alignment, endianness, and null-terminated strings. This is what gives network protocols their "C bias."

Control aspect (of the C abstract machine) — Subroutines, the call stack, calling conventions, and pointer passing. Restructuring a program around this (stack allocation, pointer passing) is part of squeezing top-tier performance.

C bias — The property that almost every network protocol and wire format assumes C/C++ data conventions (declaration-order layout, packing, endianness, null-terminated strings), because the computing world is built around the C abstract machine.

PDU (Protocol Data Unit) — A packet/unit of data exchanged by a protocol. On the wire it must be coherent with C/C++ struct layout, so it inherits the C bias.

Struct layout / declaration order — In C/C++ a struct's fields are laid out in the order declared. Java/C# classes guarantee neither a default layout nor declaration-order allocation, which is why naive managed code breaks C-shaped protocols.

Byte alignment — Placing fields at memory offsets that are multiples of a word size (commonly 4 or 8 bytes) so the CPU accesses them efficiently. Protocols usually demand 1-byte alignment instead.

Packed struct / `#pragma pack(1)` — A struct compiled with 1-byte alignment so there is no automatic padding between fields (`__attribute__((packed))` in GCC). Most protocols require this; designers then insert explicit reserved bytes to restore natural alignment where needed.

Padding / reserved bytes — Filler bytes (`char reserved1, reserved2, ...`) added deliberately to a packed struct so a following multi-byte field lands on a natural word boundary. Explains why protocol/graphics dimensions are often powers of two.

Endianness — The byte order of a multi-byte value: little-endian (least-significant byte first) vs big-endian (most-significant first). Must be respected on the wire; network order is traditionally big-endian.

BSWAP — A processor instruction that reverses byte order (used for endianness conversion). If you write the Java/C# code pattern the JIT maps to BSWAP, you avoid a native call.

Null-terminated string — A C string represented as a byte run ending in a zero byte. Most RFC protocols assume this, so a C#/Java `String` (a length-carrying compound object) must be converted to bytes with an explicit terminating zero.

`String` as compound object — In Java/C#, a string is an object carrying a length and Unicode data, not a bare null-terminated byte array. This mismatch forces explicit conversion when talking to C-shaped receivers.

ByteBuffer (Java) — A Java class for reading/writing raw bytes with control over order and position, used to hand-build binary protocol data.

FFM API (Foreign Function & Memory, Java) — Java's modern API (from Project Panama) for describing memory layouts explicitly and calling native code, replacing older JNA/JNI-style approaches. Layout is constructed programmatically, not declaratively at the syntax level.

Project Panama — The umbrella Java effort improving connections between the JVM and native code/data, of which the FFM API is a part.

Project Valhalla — The Java effort to add value types (and related features) to the JVM, expected to neutralize C#'s remaining `struct`/layout advantage.

Value type / `struct` (C#) — A type stored inline (by value) rather than as a heap reference. C#'s structs plus explicit layout attributes let it produce C-compatible data structures — its current edge over Java.

`StructLayout` / `FieldOffset` (C#) — Attributes that pin a struct's field order and byte offsets to exactly what C++ expects, producing a C-compatible layout. Central to writing protocols in C#.

`unsafe` block (C#) — A C# region permitting pointer arithmetic and direct memory access. Described as "a C compiler hiding inside C#," enabling low-level code such as YARP's hot paths.

`stackalloc` (C#) — A C# feature that allocates a block on the stack rather than the GC heap, avoiding heap allocation in hot code.

P/Invoke — Platform Invocation Services: the .NET mechanism for calling native (C) functions from managed code. A historical route to low-level protocol details.

JNA — Java Native Access: a library for calling native code from Java without hand-written JNI glue; a historical low-level route now largely superseded by FFM.

COM interop — .NET's interoperability with Component Object Model components, one of the low-level tools used in "pure but not enterprise" C#.

Varint (variable-length integer) — An integer encoding that uses only as many bytes as needed. Each byte's leading bit is a continuation flag: 0 means "last byte," 1 means "more follow"; small values (0–127) take a single byte, saving space across millions of records. Must be parsed byte-by-byte, not laid out as a struct.

Continuation bit — The most-significant bit of each varint byte indicating whether more bytes follow. It is why a varint cannot be memory-mapped as a fixed struct.

ClickHouse wire protocol — A binary protocol using fixed strings with lengths plus varint encoding for compression. Its parsing logic lives in a specific source file that a proxy author must diff against every release to stay compatible ("proxy only up to revision N").

Length-prefixed data — Variable-length fields encoded as a length followed by the bytes. Protocols use this instead of pointers, because a pointer value is meaningless on the receiving machine.

Fixed-length protocol FSM — A protocol whose finite-state machine exchanges fixed-size packets (as in Postgres and MySQL), making interoperation easier than variable-length/varint protocols.

Kafka / Spark / Samza / Storm / Hadoop — JVM-based systems-software / data-processing platforms. They exemplify high-performance software impossible under a pure design-for-GC attitude, relying on wire formats and off-heap techniques.

Avro / Protocol Buffers / Thrift — Wire/serialization formats used by such systems to encode structured data compactly for transmission, all reflecting the C-bias data conventions.

YARP — Microsoft's reverse proxy written in "pure C#, but not enterprise C#" — using `unsafe`, value types, COM interop, and careful patterns to meet determinism needs.

Reverse proxy — A server that sits in front of application servers, forwarding client requests. A raw reverse proxy demands determinism, so GC pauses are a problem; an API-gateway-style proxy sees already-throttled traffic.

Firewall — Network infrastructure with hard real-time performance needs. The discussion notes none has ever been written in C#/Java, though AOT + off-heap could in principle enable it.

HTTP.sys — A Windows kernel-level HTTP protocol driver that intercepts HTTP, letting IIS act mainly as a process monitor/host. IIS itself was written in C++.

IIS — Microsoft's Internet Information Services web server (written in C++), which in the modern stack relies on the kernel HTTP.sys driver.

Tomcat — A Java servlet container, historically without real health management. Contrasted with full application servers.

JBoss — A full Java EE / EJB application server (shipped with Red Hat/Oracle stacks) that includes health checks — the "application server" tier above a bare servlet container.

async/await — C#'s declarative asynchrony idioms. Fine for detectable-failure work (e.g., closing connections) but ill-suited to protocol packet design, which needs fine-grained explicit state.

Protocol vs code synchronization — The principle that synchronization must live either in the protocol or in the code: a synchronized protocol permits async code, but an unsynchronized protocol with async code need not work correctly.

Declarative concurrency with fine-grained state — The concurrency style protocol work requires: explicit, fine-grained state management rather than coarse-grained enterprise idioms sprinkled over packet handling.

Explicit threading — Directly managing threads (mirroring POSIX/Windows threads), the intended teaching approach for protocol code, as opposed to relying only on async/await or task frameworks.

Dispose pattern / IDisposable — A .NET pattern for deterministic release of unmanaged resources. Useful in the design-for-GC model but not a substitute for correct low-level design.

Finalizer — A method the GC may call before reclaiming an object, used to release unmanaged resources as a backstop. Part of the design-for-GC discipline.

Coarse-grained software — Business/enterprise applications (e.g., typical Spring Boot DB apps) built from large abstractions; the natural home of the design-for-GC model.

Algebraic Data Type (ADT) — A composite type formed from sums (union types) and products (records), navigated via pattern matching. In C#/Java, record types + union types + pattern matching converge toward F#-style ADTs.

Pattern matching — Deconstructing and branching on the shape of data. Named the one headline C#/Java feature of the last ~12 years, though insufficient on its own without records and unions.

Generics — Parameterized types added to both ecosystems in the same era (C# 2.0 / Java 1.5), an early example of feature parity.

LINQ — Language Integrated Query, introduced in C# 3.0; Java's later `var` type inference is cited as a parallel evolution.

C-- (C minus minus) — A stripped-down, portable-assembly variant of C used as a compiler backend target. GHC type-checks Haskell then compiles via C--.

GHC — The Glasgow Haskell Compiler, which does Haskell type checking and then lowers to C-- for code generation.

Eta — A project that replaced GHC's final backend layer with a JVM bytecode emitter to bring Haskell to the JVM, grown through community/GSoC-style effort.

Turing machine — The foundational abstract model of computation, ancestor of the von Neumann architecture.

von Neumann architecture — The stored-program computer model (shared memory for code and data). The C abstract machine is its continuation for sequential computation.

Harvard architecture — A model with separate code and data paths/memories; the GPU is mapped to this model in the discussion.

CPU / GPU / TPU — Scalar/vector processor (CPU) vs matrix processor (GPU) vs "GPU on steroids" handling a stack of matrices (TPU). Used to frame where a quantum unit might sit.

RAM (Random Access Machine) — A theoretical machine model for analyzing sequential algorithms with constant-time memory access.

PRAM (Parallel RAM) — A theoretical model extending RAM to many processors sharing memory, used to analyze parallel algorithms. The discussion asks whether a quantum PRAM exists.

Hypercube / PRAM variants — Families of parallel machine models (catalogued in Horowitz–Sahni–Rajasekaran) used in algorithm theory.

Quantum coprocessor — A hypothesized fourth chip alongside CPU/GPU/TPU dedicated to quantum computation; considered the likely near-term shape of quantum hardware.

Q# — Microsoft's quantum programming language, cited as evidence that quantum languages already exist; pragmas (à la OpenMP) may appear as an alternative.

OpenMP-style pragmas — Compiler directives that request parallelism without a new language; suggested as a possible path for quantum/parallel adoption.

Green threads / tasks — Runtime-level lightweight threads/task infrastructure built atop OS threads; cited as how the multi-core transition was absorbed without changing the programming model, only the lowering strategy.

bit / nibble / byte / word / dword — The fundamental data widths (1 bit; 4 bits; 8 bits; typically 16/32-bit word; 32-bit double word). Part of the required C-abstract-machine curriculum.

IEEE 754 — The standard for binary floating-point representation (sign, exponent, mantissa). Required knowledge for encoding/decoding numeric data on the wire.
