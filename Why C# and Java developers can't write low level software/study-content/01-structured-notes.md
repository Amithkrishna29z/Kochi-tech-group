# Software Design for Virtual Machines — Structured Notes

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

These notes convert a ~75-minute technical discussion into clean, exam-ready study material. They preserve every technical claim, example, and debate from the source and normalize conversational passages into precise prose.

---

## 1. Scope — which "virtual machine"?

The phrase "virtual machine" is overloaded. Three broad meanings exist:

- Platform VMs — Type 1 (bare-metal) and Type 2 (hosted) hypervisor VMs, which became prominent with cloud computing.
- Containers — Docker and other container architectures. Technically these are process-level isolation, but because a container runs on top of a lightweight kernel it can be viewed as a lightweight virtual machine.
- Language-level (linguistic) VMs — the subject of this discussion: the CLR (built primarily to execute C#) and the JVM (built to execute Java).

The focus is linguistic VMs: process VMs that execute bytecode/IL produced by a language compiler (Java bytecode, .NET IL). These VMs are now polyglot hosts:

- JVM hosts Clojure, Scala, Groovy — both static and dynamic languages.
- CLR hosts static languages (C#, F#) and dynamic languages (IronPython, IronRuby).

```
  Source language        Compiler front end        VM runtime
  ---------------        ------------------        ----------
  C# / F#         --->   .NET compiler --> IL  -->  CLR  --> JIT --> native code
  Java/Scala/...  --->   javac/scalac --> bytecode->JVM --> JIT --> native code
```

---

## 2. Historical background — why managed VMs won

Around 2000, most system software, middleware, and even desktop productivity tools were written in C++. Two problems drove change:

1. Producing enough competent C++ programmers was difficult.
2. Much application code did not actually need C++'s power.

Java rose against this backdrop. Around 2002 Microsoft introduced its own VM architecture (.NET / CLR). At that point virtual-machine computation became a mainstream activity.

### 2.1 The decisive advantage: automatic memory management

Microsoft called this managed memory. The model:

- Objects are allocated; the runtime tracks object scope; a background garbage-collection module reclaims unused memory for reuse.
- The mantra: reduce, reclaim, reuse.

Contrast with the alternatives of the era:

- Objective-C — manual memory management done indirectly via reference counting.
- Swift (Objective-C's successor) — ARC (Automatic Reference Counting): the compiler inserts retain/release calls, but the underlying model is still essentially manual.
- C / C++ — fully explicit manual memory management.

In managed VMs, managed memory replaces all of this.

### 2.2 The JIT turning point

As Java's HotSpot optimization technology matured, and Microsoft introduced long-term JIT and ReJIT, the VMs began JIT-compiling to native code excellently.

- As early as 1998, an IEEE Software paper (recalled as by Arthur van Hoff) argued that with garbage collection, generated code could rival or beat C++ in many cases — because the JIT has runtime context information, it can often produce better code than ahead-of-time compilation.
- Both Java and the CLR moved into the mainstream.

### 2.3 The multi-core revolution and systems software

After protocol implementations came the multi-core revolution (~2005) and, around it, functional programming, reactive programming, and parallel programming models. Java became a good platform for systems programming — which is why Kafka, Spark, Samza, Storm, and Hadoop appeared and revolutionized data-processing capacity.

Crucial claim: you cannot build software like Kafka or Spark if you code purely in the default "design for garbage collection" attitude. These systems manipulate wire-level formats (Kafka's own wire format; Avro; Protocol Buffers; Thrift), and that requires a different design posture.

---

## 3. The three variants of software design for the CLR/JVM

Three models of designing software for a language VM:

### 3.1 Design for Garbage Collection (the default model)

- Writing Java or .NET using the language's core idioms — the default, most-used, abstraction-first features — implicitly puts you in this model.
- It is the correct default for the bulk of software: coarse-grained business/enterprise applications (e.g., a typical Spring Boot database application).
- It has its own heuristics and disciplines: value types vs reference types, collection management, whether to implement the Dispose pattern, finalizers, and so on.
- Limitation: designing protocols in this attitude leads to protocols that don't stay synchronized and code that won't work correctly (see §6).

### 3.2 Design for Lowering at the Data Level (the C-bias problem)

Core insight: almost every network protocol and wire format has a C bias, because the C language has an abstract machine and the whole computing world is built around it.

Why C's abstract machine dominates:

- The first operating system written in a high-level language was written in C (Unix). C emerged as a "high-level assembler" designed around the PDP machine's assembly language.
- Once operating systems were written in C, every processor architecture since has been designed to be a good target for C-generated code. The C abstract machine became the de-facto default architecture; every language is expected to target processors that conform to it.

Consequences for protocol data (the data aspect of the C abstract machine):

1. Structure layout / declaration order — in C/C++, a struct lays out fields in declaration order. PDUs/packets must be coherent with C/C++ struct layout.
2. Byte alignment — processors have word alignment (typically 4 bytes, sometimes 8). Default struct alignment in C++ is 4, but almost all protocols require 1-byte alignment (packed structs). In C++/GCC you use `#pragma pack(1)` or `__attribute__((packed))`. With 1-byte packing, a `char` followed by an `int` makes the int spill across a word boundary — so protocol designers explicitly add reserved padding bytes (`char reserved1, reserved2, reserved3`) to keep natural alignment. Structure dimensions in protocols and graphics (even pixel dimensions) tend to be powers of two for this reason.
3. Endianness — little-endian vs big-endian byte order must be respected on the wire.
4. Null-terminated strings — C strings are null-terminated; most protocols (see any RFC) assume this. In C#/Java, `String` is a compound object (with a length; not a bare null-terminated byte run), so sending file-metadata strings to a C-struct-shaped receiver requires converting the Unicode string to bytes and explicitly attaching the terminating zero.

The Java/C# problem: Java and C# classes guarantee neither a default layout nor declaration-order field allocation. So when, e.g., an FTP-style server transfers a binary file and its metadata from Java/.NET, the receiving side expects a C-struct format — and naive managed code breaks.

How the managed platforms cope:

- Java — the `ByteBuffer` class; historically JNA / P/Invoke-style native calls; now the FFM (Foreign Function & Memory) API lets you describe memory layouts explicitly. Project Valhalla (value types for Java) is expected to neutralize C#'s remaining edge. Limitation: Java cannot specify layout declaratively at the language-syntax level; with FFM the layout is constructed programmatically.
- C# — the `struct` feature with explicit control: set default packing to 1, use `StructLayout` / `FieldOffset` attributes to lay out fields in exactly the order/offsets C++ expects, producing C-compatible data structures. This is C#'s current edge over Java.
- The programmer must "think like a C programmer while coding in Java/C#."

```
Packed vs padded struct (C):

struct { char a; int b; };

Default (pack = 4):            Packed (#pragma pack(1)):
+----+----+----+----+          +----+----+----+----+----+
| a  |pad |pad |pad |          | a  | b0 | b1 | b2 | b3 |
+----+----+----+----+          +----+----+----+----+----+
| b0 | b1 | b2 | b3 |          (int b spills across the
+----+----+----+----+           natural word boundary)
= 8 bytes                      = 5 bytes

Protocol designer's fix under pack(1): add explicit reserved bytes
struct { char a; char r1; char r2; char r3; int b; };  // realigns b
```

Real-world war stories:

- The speaker implemented a file transfer protocol in C++ (packets, enums, structs). One Mithun Darwin re-implemented it with better comments/structure, and that version is now used to teach protocol design. Emitting the same protocol from C# exposed the string problem ("my C# is not your C#").
- Given four requirements — 1-byte struct alignment, declaration order, endianness, null-terminated strings — most C# developers could not do it, even though everyone claims to "know binary file manipulation."
- ClickHouse wire protocol — uses fixed strings with lengths plus a variable-length integer (varint) encoding for compression. A 64-bit long doesn't always need 8 bytes: the first byte's leading bit is a continuation flag. If the leftmost bit is 0, the value fits in the remaining 7 bits (values 0–127 take 1 byte); if 1, continue into subsequent bytes, terminating where a 0 flag appears. Because most real values are 0–~100 and millions of records are sent, this saves enormous space. Downside: you cannot lay this out as a struct — you must parse it byte-by-byte, and the parsing logic lives in a particular source file you must diff against every new ClickHouse release to keep a proxy compatible ("proxy only up to revision N"). Such software is hard to write even in C++.
- Postgres and MySQL, by contrast, run their protocol finite-state machines with fixed-length packets — easier to interoperate with.
- Because C strings and C++ data types underlie these protocols, pointers never appear inside packets — variable-length data is length-prefixed instead (a pointer is meaningless on the other machine).
- Spark was where the speaker first saw off-heap allocation in the wild — bypassing security via internal APIs to allocate off-heap.

```
ClickHouse-style varint (LEB128-like, 7 bits per byte):

byte layout:  [C bbbbbbb]   C = continuation flag (MSB)
              C=0 -> last byte; C=1 -> more bytes follow

value 100  -> 0x64          -> [0 1100100]                 (1 byte)
value 300  -> 0b100101100   -> [1 0101100][0 0000010]      (2 bytes)
                               low 7 bits  next 7 bits
```

Conclusion of this variant: as JITs improved at producing correct data layouts, you can now write most low-level code in pure Java or C#/CLR — designing for lowering at the data level — instead of dropping to JNA/P-Invoke.

### 3.3 Design for Super-Fast Execution (low-latency systems)

For low-latency systems (trading, proxies), two rules:

1. Do not trigger garbage collection.
2. Leverage the JIT better.

JIT philosophy: keep language features/constructs simple so the JIT can generate better object code.

- Comparing C# and Java: feature-wise C# is more powerful, but in execution — if both use reference types — Java is often faster, because simple constructs let HotSpot optimize aggressively. C#'s edge is `struct` / explicit layout, which Project Valhalla should neutralize for Java.
- Guide the compiler instead of hoping it figures things out. Example: the `BSWAP` instruction (endianness byte swap). If you know which Java/C# code pattern the JIT recognizes and maps to `BSWAP`, write that pattern instead of calling native code. More generally, for one task there may be ~40 candidate code patterns, of which the JIT optimizes some far better; a knowledgeable programmer chooses the pattern — user judgment beats compiler guessing when competing patterns exist. There are canonical/standard JIT mappings from code patterns to instruction sequences; know them.
- This is a fight against the enterprise computing mindset. Enterprise programmers optimize at the level of "use better data structures/algorithms, reduce object allocation," but the next level of performance requires restructuring the whole program around the C abstract machine's data variant and computation variant (stack allocation, subroutine calls, pointer passing).
- Avoiding GC: the speaker's first instinct was to allocate large chunks up front so generational GC promotes them to the old-gen pool where they sit untouched. The better answer, found in discussion: off-heap allocation, available in both C# and Java. Possibly manage one large memory area as a unit, with sub-allocation being the programmer's responsibility.

---

## 4. JIT vs AOT — the debate (with resolution)

- Java (HotSpot) — profile-guided: interprets, profiles usage, JIT-compiles hot spots; can re-JIT as profiles change.
- .NET (CLR) — by default tiered compilation: quick JIT at load/run time, then ReJIT for performance-critical methods based on profiling. Even with ~90% of Java code JIT-ed, in C# effectively 100% runs as native code.
- AOT (ahead-of-time, e.g., .NET NativeAOT) — one speaker's mental model was that the AOT binary carries the IL/bytecode along, so re-JIT would still be possible. The counter-argument: in AOT the IL is not embedded in the EXE (unless kept as debugging information), so there is no re-JIT with AOT — you get full ahead-of-time optimization, sometimes better code, but no runtime re-optimization. The first speaker conceded the model was wrong.
- Key correction: AOT does not remove garbage collection — the GC is still there. Therefore, to be fully deterministic you need AOT plus off-heap allocation.
- With off-heap allocation + AOT you could even write a firewall in C# — but the honest question remains whether C#/Java is the natural language to write it in (analogy: the Rust-vs-C++ identity war, often loudest among people who know neither).
- Trap-based mental model of the runtime: like a threaded interpretive language — arrays of functions where control returns to the runtime between units; the runtime has trap points where it can decide to re-JIT or collect.
- GraalVM — the best current choice for AOT on the JVM; not mainstream; parts are proprietary / need a license. Spring has native-image support; whether Kafka uses Graal was left as an open research item. Trade-off: after JVM warm-up, JIT + ReJIT may out-perform AOT — so AOT pays off mainly where determinism and startup matter (proxies), not necessarily for long-running warmed servers like Kafka where non-determinism is tolerable.

```
JIT vs AOT pipelines:

HotSpot JIT:   bytecode -> interpret -> profile -> JIT hot spots -> (ReJIT loop)
.NET tiered:   IL -> quick JIT -> profile -> ReJIT hot methods
NativeAOT:     IL -> AOT compile -> native EXE (no IL inside, no ReJIT)  + GC still present
GraalVM:       bytecode -> native image (AOT)                            + GC still present

Full determinism  =  AOT  +  off-heap allocation
```

---

## 5. The reverse-proxy / firewall thread

- Microsoft's claim: a reverse proxy written in pure C# (YARP). Initial reaction: "fine — it's an API gateway; traffic reaching it is already throttled." But a raw reverse proxy demands determinism: GC pauses break it. Observation: no firewall has ever been written in C# or Java — real network infrastructure with hard performance needs (yet Kafka/Spark exist, via the techniques above).
- Resolution: it is pure C#, but "pure C#, not enterprise C#": `unsafe` blocks, COM interop, value types, careful patterns. There is effectively "a C compiler hiding inside C#'s `unsafe` block."
- C# features for low-level coding (study these): value types (structs), `unsafe` blocks, `stackalloc`, P/Invoke, COM interop, explicit layout attributes.
- Deployment context: Node.js is excellent, but nobody exposes Node.js without a good reverse proxy in front; in modern cloud/Kubernetes deployments that layer is always present.
- History: in the IIS era, IIS was written in C++; on Windows, HTTP.sys — a kernel-level HTTP protocol driver — intercepts HTTP, so IIS effectively acts as a process monitor/host. Java's side had Tomcat (servlet container, historically no real health management) vs JBoss (full EJB application server, shipped alongside Red Hat/Oracle stacks, with health checks) — the "application server" tier.

---

## 6. Protocol synchronization vs code synchronization (async/await critique)

- When implementing a protocol you need synchronization either in the protocol or in the code. If the protocol itself is synchronized, the code can be asynchronous; if the protocol has no synchronization and your code is async, the code need not work correctly.
- Using C#'s standard idioms (async/await, `Task.Run`, Dispose everywhere) while designing protocol packets leads to incorrect protocols — the coarse-grained enterprise mindset doesn't transfer to protocol work, which needs declarative concurrency with fine-grained state. Named as the single biggest problem.
- Original teaching intent: take cross-platform C code (POSIX threads / Windows threads) and write C# that mimics it — explicit threading. Most people instead wrote in the design-for-GC model because the protocol wasn't conceptually clear to them; people program at a functional level, not a technical level, and habit takes over.
- Elsewhere (e.g., closing connections) async idioms are fine — failures there are detectable; but in low-level code, people who write VM-level code generally lack the low-level pen.
- Personal style note: the speaker uses explicit threading, rarely async/await ("I wrote it for books, not daily use"); newer developers use only declarative threading and assume all of .NET works that way — but the actual source of kernel/services teams is not all IDisposable/async idioms.

---

## 7. Language evolution rant (C# vs Java feature parity)

"Imitation is the sincerest form of flattery" — the two ecosystems have leapfrogged:

- Java 1.0 stabilized → C# 1.0 appeared (2002).
- Generics arrived in both in the same era (C# 2.0 / .NET 2.0; Java 1.5) — near parity, plus the anonymous-types era.
- C# 3.0 brought LINQ; Java later got type inference (`var`) — a good feature.
- async/await came to C# — genuinely good.
- Practically, in the last ~12 years (post-2012) the one headline feature is pattern matching — and pattern matching alone isn't enough: record types + union types + pattern matching converge into algebraic data types (ADTs), F#-style.
- Ask a C# advocate and they'll list 36 features, most of them syntactic sugar. Language designers ship every six months "so career C++ programmers and career C# developers have jobs."
- Whatever Go's faults, Go is excellent on parsimony — accept a little pain and there's nothing extra to carry. (Joke: that's the problem with Darwinian evolution of languages.)
- Realistic conclusion: high-level idiomatic C#/Java code cannot produce protocols — near-certain; you mix-and-match language levels ergonomically rather than treating every feature as essential.

---

## 8. The C abstract machine as the master key

- The C abstract machine has two aspects: a data aspect (struct layout, alignment, endianness, strings) and a control aspect (subroutines, the stack, calling conventions, pointer passing).
- Since ~90% of Java code and ~100% of C# code executes as JIT-ed native code, and every real processor conforms to the C abstract machine, the JIT runs on top of the C abstract machine. Producing optimized tight-loop code = getting the JIT to produce good code on top of that machine.
- Thesis of the whole talk: every developer — Java, .NET, PHP, Python — who understands the C abstract machine can write performant, low-level code in their own language. Every language has such a layer, explicit or not.
- Required curriculum: bit, nibble, byte, word, dword; integer and IEEE floating-point encodings; bit patterns and casting; struct layout; little-endian/big-endian.
- Caveat: rational design assumptions aren't enough — empirical, a-posteriori verification is always required ("the proof of the pudding is in the eating").
- Evidence of the model's power: the GHC Haskell compiler type-checks Haskell and then compiles via C-- (C-minus-minus, a stripped-down C used as portable assembly). The Eta project replaced GHC's final layer with a JVM bytecode emitter to bring Haskell to the JVM (grown via community effort / GSoC-style hand-holding). If you build a Java/C# trans-compiler to processor-specific code, targeting x86 assembly or C-- suffices.
- Mentoring context: for most mentees this is their first network programming in a non-C/C++ language; the mentors are accomplished programmers with their own strongholds.

---

## 9. Quantum computing digression — models of computation

Triggered by: "will this C-abstract-machine thinking survive quantum computing?"

- Foundations: the Turing machine → von Neumann architecture (stored-program computer) — the C abstract machine is its continuation, for sequential computation. The GPU maps to the Harvard architecture. The TPU is "a GPU on steroids": if the CPU is scalar/vector, the GPU is matrices, the TPU is a stack of matrices.
- Algorithm-theory machine models: RAM (random access machine), PRAM (parallel RAM), hypercube/PRAM variants (as catalogued in the Horowitz–Sahni–Rajasekaran algorithms book).
- Open questions posed: Is there a quantum PRAM model? A quantum instruction set? Will quantum arrive as (a) a faster instruction set inside classical CPUs (microcode-level change), (b) a quantum coprocessor — a fourth chip alongside CPU/GPU/TPU, or (c) a multi-computer approach with concurrent quantum units?
- Parallelism precedent: when multi-core arrived, the OS abstracted cores as threads; runtimes built green threads / task infrastructure on top; you either get auto-parallelism from tools/compilers or it degenerates into concurrency — so possibly the same programming model survives and only the lowering strategy changes.
- Quantum algorithms exist as a class (arguably a class of parallel/hyper-parallel algorithms — "instantaneous state exploration" is the popular claim that intractable problems become practically tractable). Languages like Q# (Microsoft) already exist; perhaps pragmas (à la OpenMP) will appear instead of new languages.
- Even with quantum computers, sequential computing remains mainstream for CPU-level processing, and data is still data — but where to place the abstraction over the underlying machine model is an open philosophical problem. The CPU hasn't disappeared despite tensor hardware; a quantum coprocessor is the likely near-term shape (though "the CPU persisting is partly a sociological phenomenon").
- Action items: research current quantum results/models within a week; invite and "grill" a prepared guest (Abhijith) on computational aspects.

---

## 10. Session summary (as delivered)

1. "Software design for virtual machines" here = language-level VMs (JVM/CLR).
2. Three design modes: (a) Design for Garbage Collection — the default C#/Java model, for coarse-grained business software; (b) Design for Lowering at the Data Level — required for protocol design because every protocol has a C bias; (c) Design for Super-Fast Execution — avoid GC + leverage JIT (and AOT where determinism matters), the way Kafka-class software is possible.
3. AOT (e.g., .NET NativeAOT) has no re-JIT (IL not carried in the binary) but still has GC; full determinism = AOT + off-heap allocation. GraalVM is the JVM-side AOT option (licensing caveats).
4. C# low-level features to study: value types, `unsafe` blocks, P/Invoke, COM interop, explicit layout, `stackalloc`.
5. Study the C abstract machine (data + control aspects) and the JIT's canonical code-pattern mappings, and you can produce performant code and use C#/Java as system programming languages.
6. Open thread: models of computation (RAM/PRAM/hypercube; von Neumann vs Harvard; CPU/GPU/TPU) and whether a quantum machine model / instruction set / language (Q#) reshapes any of this.
