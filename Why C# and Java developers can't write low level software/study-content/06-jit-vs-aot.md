# JIT vs AOT — Notes and Corrected Conclusions

> 🎥 Video: https://www.youtube.com/watch?v=etoBX5RwLjc

A reconstruction of the JIT-vs-AOT debate from the talk, including the mental-model correction one speaker conceded.

---

## The four strategies compared

| | HotSpot JIT (JVM) | .NET tiered / ReJIT (CLR) | NativeAOT (.NET) | GraalVM native image (JVM) |
|---|---|---|---|---|
| When compiled | At run time, hot spots | At run/load time, then hot methods | Before run time (AOT) | Before run time (AOT) |
| Profile-guided | Yes (interpret → profile → JIT) | Yes (quick JIT → profile → ReJIT) | No | No |
| Re-optimization (re-JIT) | Yes, as profiles change | Yes (ReJIT) | No — IL not in the binary | No |
| IL/bytecode in the artifact | Yes | Yes | No (unless kept as debug info) | No |
| Garbage collector present | Yes | Yes | Yes — still there | Yes — still there |
| Startup | Slower (warm-up) | Fast-ish, improves after warm-up | Fast, deterministic | Fast, deterministic |
| Peak throughput | Very high after warm-up | Very high after warm-up | High, no runtime re-tuning | High, no runtime re-tuning |
| Mainstream? | Yes | Yes | Growing | No; parts proprietary / need a license |

---

## HotSpot (JVM) — profile-guided JIT

HotSpot interprets bytecode first, profiles which code is "hot," JIT-compiles those hot spots to native code, and can re-JIT as the profile changes. Roughly 90% of Java code ends up running as native code. The strength is that the compiler has runtime context information, which is exactly why the 1998 IEEE Software argument held that GC + JIT code can rival or beat C++.

## .NET (CLR) — tiered compilation and ReJIT

By default the CLR uses tiered compilation: a quick, low-optimization JIT at load/run time for fast startup, followed by ReJIT of performance-critical methods at higher optimization once profiling identifies them. In practice effectively 100% of C# runs as native code (vs ~90% for Java).

Mental model (from the talk): the runtime behaves like a threaded interpretive language — arrays of functions with control returning to the runtime between units. The runtime has trap points where it can decide to re-JIT or to collect garbage.

## NativeAOT (.NET) — ahead-of-time

NativeAOT compiles IL to a native executable before run time. Two properties matter and one of them corrected a speaker's mental model:

- The mistaken model: the AOT binary carries the IL along, so re-JIT would still be possible.
- The correction (conceded): the IL is not embedded in the EXE (unless kept as debugging information), so there is no re-JIT under AOT. You get full ahead-of-time optimization — sometimes better code — but no runtime re-optimization.

Second key correction: AOT does not remove the garbage collector. The GC is still present. Therefore:

> Full determinism = AOT + off-heap allocation.

AOT removes JIT warm-up and pause-inducing recompilation, but only off-heap allocation removes the GC pauses themselves.

## GraalVM native image (JVM)

GraalVM is the best current choice for AOT on the JVM. Caveats: it is not mainstream, and parts are proprietary / require a license. Spring has native-image support; whether Kafka uses Graal was left as an open research item in the talk.

---

## When AOT actually pays off

The honest trade-off from the discussion:

- After JVM warm-up, JIT + ReJIT may out-perform AOT, because the JIT can exploit live runtime profiles that an AOT compiler never sees.
- So AOT pays off mainly where determinism and startup matter: raw reverse proxies, trading systems, short-lived processes, serverless cold starts.
- For long-running, warmed servers like Kafka, non-determinism is tolerable and warmed JIT is often the better choice — AOT is not automatically superior.

---

## Consequences for the "super-fast execution" model

- With off-heap allocation + AOT you could in principle write even a firewall in C# — the open question being whether C#/Java is the natural language for it (compared, half-jokingly, to the Rust-vs-C++ identity war fought loudest by people who know neither).
- YARP demonstrates the achievable shape: "pure C#, but not enterprise C#" — `unsafe` blocks, value types, COM interop, careful patterns; effectively a C compiler hiding inside `unsafe`.

---

## One-line takeaways

- JIT has runtime context; AOT does not — that is the whole trade.
- AOT: no re-JIT (IL not in the binary), but the GC remains.
- Determinism needs AOT and off-heap allocation together — neither alone.
- Warmed JIT can beat AOT; choose AOT for determinism/startup, not by default.
- GraalVM is the JVM AOT path, with licensing caveats and non-mainstream status.
