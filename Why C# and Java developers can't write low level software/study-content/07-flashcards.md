# Flashcards — Software Design for Virtual Machines

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

Anki-importable. Each line is `Question ; Answer`. Import with `;` as the field separator. 70 cards.

```
Which two VMs does "linguistic virtual machine" refer to in this talk ; The CLR (built for C#) and the JVM (built for Java)
Name three languages the JVM hosts besides Java ; Clojure, Scala, Groovy
Name two dynamic languages the CLR hosts ; IronPython and IronRuby
Why can a container be viewed as a lightweight VM ; It is process-level isolation running on top of a lightweight kernel
Around what year did Microsoft introduce .NET/CLR ; About 2002
What dominant language did Java and C# rise against around 2000 ; C++
Two reasons managed VMs displaced C++ ; Too few competent C++ programmers, and most app code did not need C++'s power
What did Microsoft call automatic memory management ; Managed memory
State the managed-memory mantra ; Reduce, reclaim, reuse
How did Objective-C manage memory ; Manually and indirectly via reference counting
What is Swift's ARC ; Automatic Reference Counting: the compiler inserts retain/release, but the model is essentially manual
What runtime advantage lets a JIT sometimes beat C++ ; It has runtime context information
What 1998 source argued GC code could rival C++ ; An IEEE Software paper recalled as by Arthur van Hoff
When did the multi-core revolution occur ; Around 2005
Name five JVM systems/data-processing platforms from the talk ; Kafka, Spark, Samza, Storm, Hadoop
Why can't you build Kafka in a pure design-for-GC attitude ; It manipulates wire-level formats needing a different design posture
Name three wire/serialization formats mentioned ; Avro, Protocol Buffers, Thrift
What are the three software-design models for a language VM ; Design for GC, Design for Lowering at the Data Level, Design for Super-Fast Execution
Which design model is the correct default and for what ; Design for GC, for coarse-grained business/enterprise apps
Give the archetypal design-for-GC application ; A typical Spring Boot database application
Why does design-for-GC fail for protocols ; The protocol won't stay synchronized and the code won't work correctly
What is the core insight of the data-level model ; Almost every network protocol and wire format has a C bias
Why does the C abstract machine dominate ; Unix (first HLL OS) was in C, so every processor since is designed to run C code well
What was C originally described as ; A high-level assembler designed around the PDP machine's assembly
List the four data-aspect interop rules ; Declaration-order layout, byte alignment (pack 1), endianness, null-terminated strings
Default C++ struct alignment vs what protocols need ; Default 4-byte word alignment; protocols need 1-byte (packed) alignment
Two ways to pack a struct to 1 byte in C++/GCC ; #pragma pack(1) or __attribute__((packed))
Why add reserved padding bytes in a packed struct ; To realign a following multi-byte field onto a natural word boundary
Why do protocol/graphics dimensions tend to be powers of two ; For natural byte alignment
What are the two endianness orders ; Little-endian (LSB first) and big-endian (MSB first)
What processor instruction swaps byte order ; BSWAP
Why must you convert a C#/Java String before sending to a C receiver ; It is a compound object with length/Unicode; you must flatten to bytes and add a null terminator
What layout guarantees do Java/C# classes give ; Neither a default layout nor declaration-order field allocation
Java tool for hand-building binary data ; ByteBuffer
What does Java's FFM API do ; Lets you describe memory layouts explicitly and call native code (programmatically, not at syntax level)
What umbrella project contains the FFM API ; Project Panama
What is Project Valhalla expected to do ; Add value types to Java, neutralizing C#'s struct/layout edge
C#'s two attributes for exact struct layout ; StructLayout and FieldOffset
What is C#'s current edge over Java for protocols ; struct + explicit layout producing C-compatible data structures
What is C#'s edge expected to be neutralized by ; Project Valhalla
What compression encoding does the ClickHouse protocol use ; A variable-length integer (varint) encoding
In a varint, what does the leading bit of a byte mean ; Continuation flag: 0 = last byte, 1 = more bytes follow
How many bytes does a varint use for values 0–127 ; One byte
Why can't a varint be laid out as a struct ; It is variable-length; it must be parsed byte-by-byte
How do you keep a proxy compatible with new ClickHouse releases ; Diff the vendor's parsing source file each release ("proxy only up to revision N")
How do Postgres/MySQL protocols differ from ClickHouse's ; They use fixed-length packets in their FSM, easier to interoperate with
Why do pointers never appear inside protocol packets ; A pointer is meaningless on the receiving machine; data is length-prefixed instead
Where was off-heap allocation first seen in the wild ; In Spark
Two rules of the super-fast-execution model ; Do not trigger GC; leverage the JIT better
Between reference-type C# and Java, which is often faster and why ; Java, because simple constructs let HotSpot optimize aggressively
What does "guide the compiler" mean for the JIT ; Write the code pattern the JIT maps to the best instruction sequence instead of hoping
Example of choosing a JIT-friendly pattern ; Writing the pattern the JIT lowers to BSWAP instead of calling native code
What mindset does the speed model fight ; The enterprise computing mindset (better data structures / fewer allocations)
What does the next level of performance require ; Restructuring the whole program around the C abstract machine's data and control aspects
Naive way to avoid GC ; Allocate large chunks up front so generational GC promotes them to old-gen
Better way to avoid GC ; Off-heap allocation (available in both C# and Java)
How does HotSpot decide what to compile ; It interprets, profiles hot spots, JIT-compiles them, and can re-JIT
What is .NET's default compilation strategy ; Tiered compilation: quick JIT then ReJIT of hot methods
Roughly what fraction of Java vs C# runs as native code ; ~90% of Java, ~100% of C#
Corrected fact: does AOT keep the IL in the EXE ; No (unless as debug info), so there is no re-JIT under AOT
Corrected fact: does AOT remove the garbage collector ; No, the GC is still present
What equals full determinism ; AOT + off-heap allocation
What is the JVM's main AOT option and its caveat ; GraalVM native image; not mainstream, parts proprietary / need a license
When can warmed JIT beat AOT ; After JVM warm-up, JIT + ReJIT can out-perform AOT
Who wrote a reverse proxy in pure C# ; Microsoft (YARP)
What does "pure C# but not enterprise C#" use ; unsafe blocks, value types, COM interop, careful patterns
What "hides inside" C#'s unsafe block ; Effectively a C compiler
List C# low-level features to study ; Value types, unsafe blocks, stackalloc, P/Invoke, COM interop, explicit layout
What Windows kernel driver intercepts HTTP ; HTTP.sys
Tomcat vs JBoss ; Tomcat is a servlet container (historically no health management); JBoss is a full EJB app server with health checks
Where must synchronization live in a protocol implementation ; Either in the protocol or in the code
What concurrency style does protocol work need ; Declarative concurrency with fine-grained state, not coarse-grained enterprise idioms
The single biggest problem named in the talk ; Using coarse-grained enterprise idioms (async/await etc.) when designing protocol packets
What was the original teaching intent for protocol code ; Mimic cross-platform C (POSIX/Windows threads) with explicit threading in C#
What converges into algebraic data types (ADTs) ; Record types + union types + pattern matching
The one headline C#/Java feature of the last ~12 years ; Pattern matching
What language is praised for parsimony ; Go
What backend does GHC compile Haskell through ; C-- (C minus minus), a portable assembly
What did the Eta project do ; Replaced GHC's final layer with a JVM bytecode emitter to bring Haskell to the JVM
Two aspects of the C abstract machine ; Data aspect (layout/alignment/endianness/strings) and control aspect (subroutines/stack/calling conventions/pointers)
Why does the JIT run "on top of" the C abstract machine ; Every real processor conforms to it, and ~90–100% of managed code runs as native
The talk's central thesis ; Any developer who understands the C abstract machine can write performant low-level code in their own language
Required low-level curriculum ; bit/nibble/byte/word/dword, integer & IEEE-754 encodings, bit patterns/casting, struct layout, endianness
What verification does the talk insist on ; Empirical, a-posteriori verification ("the proof of the pudding is in the eating")
Which architecture is the C abstract machine a continuation of ; The von Neumann architecture (from the Turing machine)
Which architecture does the GPU map to ; The Harvard architecture
How is the TPU described relative to CPU/GPU ; CPU=scalar/vector, GPU=matrices, TPU=a stack of matrices ("GPU on steroids")
Name three theoretical machine models from algorithm theory ; RAM, PRAM, hypercube/PRAM variants
Which book catalogs those parallel machine models ; Horowitz–Sahni–Rajasekaran
What is the likely near-term shape of quantum hardware ; A quantum coprocessor (a fourth chip alongside CPU/GPU/TPU)
Name an existing quantum programming language ; Q# (Microsoft)
What alternative to new languages might quantum adoption take ; OpenMP-style pragmas
What happens if auto-parallelism isn't achieved ; It degenerates into concurrency (same model, different lowering strategy)
```
