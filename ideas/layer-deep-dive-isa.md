# Single-Layer Deep Dive: Instruction Set and Chip Architecture

`English | `[中文](layer-deep-dive-isa.zh-CN.md)

> Collected June 2026. Exploratory discussion. Premise: set aside hardware devices and manufacturing for now.
>
> This is an in-depth expansion of layer 1 of the [complete blueprint](clean-slate-computing-full-blueprint.md), focused on "why this choice, and what it costs" rather than just listing features.

## 1. First, the contract goals of this layer

An ISA is fundamentally a **long-term contract between software and hardware**. It is the hardest thing to change, so it must be designed for "the high-end implementation decades from now," not for the simplest implementation. Concrete goals:

- The default implementation form is **wide-issue out-of-order + SMT + heterogeneous accelerators**, not a single-issue in-order core;
- Encoding, registers, the memory model, and the exception model are **all defined together with the microarchitecture**;
- The specification is an **executable formal golden model** (in the spirit of Sail), the single source of truth, from which conformance tests and formal verification are generated.

Making "co-design" concrete means: every decision below answers both "what does software see" and "is it good for hardware to build."

## 2. Instruction encoding: designed for wide decode

This is one of the most direct points of departure from RISC-V (whose C compression extension complicates decode). The choices:

- **Primary instructions are a fixed 32 bits**, making "cutting instruction boundaries" trivial — an N-wide decoder can split them in parallel with no dependencies, which is the crux of a wide front end. None of x86's 1–15-byte variable length that must be scanned serially to find boundaries.
- **Want density, but not via arbitrary variable length.** Use a **decode-friendly "bundle" mechanism**: length information is encoded at fixed bit positions in each slot, so the decoder knows the length without parsing the content.
- **Register fields are at fixed positions across all instruction formats** (RISC-V does this well — borrow it directly): register reads can start before full decode completes.
- Reserve **formal encodings for architectural fused instructions**, making "fusion" a software-visible contract rather than macro-op fusion that each microarchitecture does privately with inconsistent behavior.

Cost: fixed 32-bit has somewhat worse code density than extreme variable length (slightly more instruction-cache pressure), in exchange for deterministic high-bandwidth decode. For a high-end target, the trade is worth it.

## 3. Registers and "whether to have flags"

- **32 general-purpose registers** to start. More would reduce spills but consume encoding bits and enlarge context-switch state; 32 is the proven sweet spot on high-end OoO. Physical registers are piled up via renaming.
- **No global flags register.** Global flags create false "partial-flags" dependencies during renaming — a notorious nuisance in out-of-order design (one of x86's pieces of historical baggage).
- **But restore its benefit**: provide **architectural compare-and-branch and compare-and-select** as single instructions. No global-flags renaming pollution, and no need for microarchitecture to do speculative fusion — fusion is written into the ISA.

## 4. Memory consistency: one model, and "correct by default"

This is the most critical landing point for the "make the fastest path the most correct path" principle.

- **Exactly one model, pinned down by a formal spec**, with no "weak/strong optional."
- **The model is a multi-copy-atomic release/acquire system (RCsc-style)**: ordinary loads/stores carrying acquire/release semantics can express the vast majority of synchronization — clear semantics, verifiable by tools.
- **Strong by default, weak by opt-in**: unannotated memory accesses follow the stronger semantics, so **naive code is correct by nature**; experts who really want to squeeze performance can explicitly use relaxed variants.

Trade-offs laid out against two alternatives: **TSO (the x86 kind)** is stronger and easier to program, but gives microarchitecture less reordering freedom and more cross-core coherence traffic; **a weak model (ARM/RVWMO)** has lots of freedom but is too hard to reason about, easily producing subtle bugs. The choice of RCsc + strong-by-default leans toward reasonability — which does not conflict with performance first, because hard-to-reason bugs are themselves an enormous hidden cost.

## 5. Memory safety: pure-capability architecture (the CHERI route)

This is the foundation that is **isomorphic** to the OS/language capability model above.

- **All pointers are capabilities — a pure-capability architecture** — not a hybrid of "integer pointers + capabilities" (hybrids exist to be compatible with legacy code, and we don't carry that baggage). One capability = address + bounds + permissions, **128 bits**, plus **a 1-bit tag per capability slot** kept out-of-band (the tag prevents forgery). Bounds are squeezed into the 128 bits via compressed encoding (the CHERI Concentrate kind).
- **The payoff is a win for both performance and correctness**: spatial safety (out-of-bounds) becomes impossible in hardware, **saving the pervasive bounds-check instructions in software**; and it doubles directly as the hardware primitive for inter-process / inter-module isolation.
- **Two costs, stated honestly**:
  1. **Pointers widen to 128 bits** → higher cache and memory footprint, bandwidth pressure for pointer-dense structures. This is the most concrete cost of this route.
  2. **Temporal safety (use-after-free) is not free**: spatial safety is solved naturally by capabilities, but dangling references need extra mechanisms — capability revocation + periodic sweeps (the CHERIvoke kind) or load barriers, with runtime overhead.
- **The co-design dividend**: the OS's capabilities and the language IR's capabilities **descend all the way to this same hardware capability**, not a software emulation — one protection model end to end.

## 6. Protection model: replace "privilege rings" with "holding a capability"

- Discard x86's ring 0–3 / complex privilege levels. **Protection rests mainly on "whether you hold the relevant capability," not "which privilege level you're in"** — a flatter, more composable model.
- Still keep **a tiny amount of asymmetry** (a minimal trusted root for that formally verified microkernel); everything else is capability-based.
- **Precise exceptions + structured async**: an out-of-order implementation must still deliver precise exceptions (debuggable, recoverable); interrupts follow a structured model co-designed with the OS layer's "batched async ABI."

## 7. Data parallelism: scalar/vector/tensor unified, not three islands

- **Vector-length-agnostic (VLA)** (the correct idea from SVE / RVV): code doesn't hard-code vector width, and the same binary scales automatically across implementations.
- **Masking/predication are first-class**, not retrofitted.
- **Tensor/matrix as a first-class extension**, sharing the same register and memory model as scalar and vector. There's a real difficulty here: matrix-unit state is huge, so **how to virtualize and context-switch it?** The leaning is to **make matrix instructions operate on "memory descriptors" rather than a giant architectural register file**, minimizing switchable state — trading a bit of peak convenience for switchability.
- The three are unified under one model, instead of today's CPU-scalar, GPU, and NPU each speaking their own language.

## 8. Heterogeneity: one address space, accelerators as equal citizens

- **CPU/GPU/NPU/DPU are all agents on the coherent fabric, sharing one virtual address space — capabilities included.** Page tables / address translation are unified; IOMMU/MMU merged into one.
- Consequences: **eliminate the separate "device memory" API**, eliminate copies across PCIe; capabilities can be **passed safely across the accelerator boundary** — the hardware precondition that makes the upper layers' "unified memory model" promise hold.

## 9. Kill fragmentation with "Profiles"

- No "minimal base + a pile of optional extensions." Define **a small number of complete profiles** (e.g., embedded-class / application-class / server·HPC-class), **each a fully guaranteed set**, with version numbers.
- "This chip is application-class v2" then equals a definite, complete, dependable set of capabilities — software need not face the à-la-carte nightmare of "exists in theory, not on this chip."

## 10. The spec is the truth: a formal golden model

- Write the **single authoritative specification** in an executable formal language (the Sail kind); conformance tests are **generated from the spec**; the hardware implementation and that verified kernel are **formally verified against the same model.**
- This turns "conformance" from "run a pile of tests and hope" into "provable against the spec."

## 11. Trade-offs specific to this layer that must be accounted for

- **Pure-capability 128-bit pointers**: memory/cache pressure is a real and persistent cost; pointer-dense workloads suffer most.
- **Temporal safety** has no zero-cost solution yet; revocation/sweeping has overhead.
- **Matrix-unit virtualization**: using memory descriptors to shrink state sacrifices a bit of peak programming convenience.
- **A single memory model**: the reasonability gained comes at the price of giving up some aggressive-reordering freedom microarchitecture might want.
- **Fixed 32-bit encoding**: the price of simple decode is density worse than extreme variable length.
- **Unified heterogeneous address space**: coherence must reach the accelerators, pushing **the design complexity and scalability pressure of the coherent fabric** down to layer 2 (memory and interconnect), which becomes harder to build.

## In one sentence

Treat "wide out-of-order + heterogeneity + capability safety" as day-one design goals, and make encoding, registers, the memory model, and the protection model co-design with each other; wherever performance or convenience must be sacrificed (128-bit pointers, a single memory model, fixed encoding), pay only after explicitly accounting for it.
