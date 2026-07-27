# Quiz — Software Design for Virtual Machines

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

25 multiple-choice questions and 10 short-answer questions. Answer key at the end.

---

## Part A — Multiple choice (25)

1. In this talk, "virtual machine" primarily means:
   a) Type 1 hypervisor VMs  b) Docker containers  c) Language-level VMs (JVM/CLR)  d) Emulators

2. The JVM and CLR execute:
   a) Source code directly  b) Bytecode/IL, later JIT-compiled to native  c) Only interpreted native code  d) Microcode

3. Microsoft's term for automatic memory management is:
   a) ARC  b) Reference counting  c) Managed memory  d) Off-heap allocation

4. The managed-memory mantra is:
   a) Allocate, free, repeat  b) Reduce, reclaim, reuse  c) Retain, release, autorelease  d) Map, reduce, filter

5. Swift's ARC is best described as:
   a) Tracing garbage collection  b) Compiler-inserted retain/release, essentially manual  c) Off-heap allocation  d) Fully automatic like the JVM

6. A JIT can sometimes beat AOT/C++ because it has:
   a) More memory  b) Runtime context information  c) A larger instruction set  d) No garbage collector

7. Which set are all JVM-based systems software?
   a) Kafka, Spark, Hadoop  b) Redis, Nginx, HAProxy  c) Postgres, MySQL, SQLite  d) IIS, HTTP.sys, YARP

8. The default design model for coarse-grained business apps is:
   a) Design for Super-Fast Execution  b) Design for Lowering at the Data Level  c) Design for Garbage Collection  d) Design for AOT

9. Almost every network protocol has a "C bias" because:
   a) C is the fastest language  b) Processors are designed to be good targets for C code  c) RFCs are written in C  d) C has the most libraries

10. Protocols almost always require struct alignment of:
    a) 8 bytes  b) 4 bytes  c) 2 bytes  d) 1 byte (packed)

11. Reserved padding bytes are added to a packed struct in order to:
    a) Encrypt data  b) Realign a following multi-byte field to a natural boundary  c) Store pointers  d) Mark the end of the packet

12. Little-endian means:
    a) Most-significant byte first  b) Least-significant byte first  c) Random order  d) Network order by default

13. A C#/Java `String` differs from a C string because it is:
    a) Always UTF-32  b) A compound object with a length, not a null-terminated byte run  c) Immutable so it cannot be sent  d) Always null-terminated already

14. Which guarantees do Java/C# classes give about field layout?
    a) Declaration order always  b) Natural alignment always  c) Neither a default layout nor declaration order  d) Big-endian layout

15. C#'s current edge over Java for protocols is:
    a) async/await  b) LINQ  c) `struct` + explicit layout (StructLayout/FieldOffset)  d) Records

16. Java's modern API for describing memory layouts and calling native code is:
    a) JNA  b) The FFM API (Project Panama)  c) LINQ  d) Reflection

17. In a varint, a leading bit of 1 on a byte means:
    a) The value is negative  b) It is the last byte  c) More bytes follow  d) The value is zero

18. Values 0–127 in a varint occupy:
    a) 8 bytes  b) 4 bytes  c) 2 bytes  d) 1 byte

19. Why do pointers never appear inside protocol packets?
    a) They are too large  b) A pointer is meaningless on the receiving machine  c) They are encrypted  d) C forbids it

20. Off-heap allocation was first seen by the speaker in:
    a) Kafka  b) Spark  c) Hadoop  d) Storm

21. Which statement about NativeAOT is correct?
    a) It removes the GC  b) It keeps IL in the EXE for re-JIT  c) It has no re-JIT and still has a GC  d) It is always faster than warmed JIT

22. Full determinism, per the talk, requires:
    a) AOT alone  b) Off-heap alone  c) AOT + off-heap allocation  d) Tiered compilation

23. The JVM-side AOT option (with licensing caveats) is:
    a) GraalVM native image  b) HotSpot  c) ReJIT  d) C--

24. YARP is described as:
    a) Enterprise C#  b) Pure C#, but not enterprise C# (unsafe, value types, COM interop)  c) Written in C++  d) A Java servlet

25. The talk's central thesis is that low-level performance comes from understanding:
    a) Assembly only  b) The C abstract machine (data + control aspects)  c) The garbage collector  d) LINQ internals

---

## Part B — Short answer (10)

26. Name the three software-design models for a language VM and the domain each targets.

27. List the four data-aspect interop rules of the C abstract machine.

28. Explain why designing protocol packets with async/await and coarse-grained enterprise idioms tends to produce incorrect protocols.

29. Describe how the ClickHouse-style varint encoding works and one reason it saves space.

30. Why must a proxy author diff the vendor's parsing source against every new ClickHouse release?

31. State the two corrected conclusions about AOT from the JIT-vs-AOT debate.

32. Explain when warmed JIT can out-perform AOT, and what AOT is therefore best used for.

33. Contrast HotSpot's profile-guided JIT with .NET's tiered compilation.

34. What are the two aspects of the C abstract machine, and which is needed for protocol design?

35. Describe two possible near-term shapes for quantum hardware raised in the talk, and how the multi-core transition is used as a precedent.

---

## Answer key

Part A: 1-c, 2-b, 3-c, 4-b, 5-b, 6-b, 7-a, 8-c, 9-b, 10-d, 11-b, 12-b, 13-b, 14-c, 15-c, 16-b, 17-c, 18-d, 19-b, 20-b, 21-c, 22-c, 23-a, 24-b, 25-b.

Part B:

26. Design for Garbage Collection (coarse-grained business/enterprise apps, e.g. Spring Boot); Design for Lowering at the Data Level (network protocols and wire formats with a C bias); Design for Super-Fast Execution (low-latency systems: trading, raw reverse proxies, firewalls).

27. Declaration-order struct layout; 1-byte (packed) alignment with explicit reserved bytes to realign; endianness (little- vs big-endian) on the wire; null-terminated strings.

28. The coarse-grained enterprise mindset does not transfer to protocol work. Protocols need declarative concurrency with fine-grained state, and synchronization must live in the protocol or the code; sprinkling async/await over packet handling leaves the protocol unsynchronized, so the code need not work correctly.

29. Each byte carries 7 data bits plus a leading continuation flag (0 = last byte, 1 = more follow). Small values (0–127) fit in one byte; larger values chain across bytes. Because most real values are small and millions of records are sent, using one byte instead of eight for the common case saves enormous space.

30. The varint/protocol parsing logic lives in a specific source file that changes across releases; to stay byte-compatible ("proxy only up to revision N") the proxy's parser must be kept in sync by diffing that file each release.

31. (a) Under AOT the IL is not embedded in the EXE (unless as debug info), so there is no re-JIT. (b) AOT does not remove the garbage collector — the GC is still present. Hence full determinism = AOT + off-heap allocation.

32. After JVM warm-up, JIT + ReJIT can exploit live runtime profiles the AOT compiler never sees, so it may out-perform AOT — as is tolerable for long-running warmed servers like Kafka. AOT is therefore best where determinism and startup matter: raw reverse proxies, trading systems, short-lived/serverless processes.

33. HotSpot interprets, profiles hot spots, JIT-compiles them, and re-JITs as profiles change (~90% of code ends native). .NET tiered compilation does a quick low-optimization JIT at load/run time, then ReJITs hot methods at higher optimization (~100% ends native).

34. The data aspect (struct layout, alignment, endianness, strings) and the control aspect (subroutines, the stack, calling conventions, pointer passing). The data aspect is what protocol design requires.

35. Possible shapes: (a) a faster instruction set inside classical CPUs (microcode-level change); (b) a quantum coprocessor — a fourth chip alongside CPU/GPU/TPU; (c) a multi-computer approach with concurrent quantum units. The multi-core precedent: the OS abstracted cores as threads and runtimes built green threads/tasks on top, so the programming model survived and only the lowering strategy changed; auto-parallelism otherwise degenerates into concurrency.
