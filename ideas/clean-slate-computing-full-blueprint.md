# A Complete Blueprint for a Clean-Slate Computing System

`English | `[中文](clean-slate-computing-full-blueprint.zh-CN.md)

> Collected June 2026. Exploratory discussion. Premise: set aside hardware devices and manufacturing for now.
>
> This is a deepened version building on [clean-slate-computing-stack](clean-slate-computing-stack.md). Two key revisions:
> 1. **The master goal is now "performance first + eliminating the accidental complexity caused by historical baggage."** Security is a by-product of clean design, with trade-offs stated explicitly only where there is a real conflict.
> 2. **The chip is a fresh design, co-designed with the microarchitecture, borrowing only RISC-V's methodology — not its assumptions.**

## 0. The system's "constitution": five overarching principles

Before any layers, five meta-principles that run through every layer.

1. **Co-design, not after-the-fact layering.** Layers still exist (so humans can comprehend), but **the interfaces between layers are designed together**, not built separately and reverse-engineered into a fit. A large share of today's accidental complexity comes from "the interface is a historical accident."
2. **Performance is multi-axis, ordered by "compounding."** Priority: **energy efficiency (useful computation per joule) > scalability (parallelism / scale) > latency > peak throughput.** The first two compound with scale and deserve top priority; clock-speed-style peak performance comes last.
3. **Make the fastest path the most correct path.** Design so that "the highest-performance way to write it happens to also be the safe/correct way," so that **there is no permanent security tax** — security isn't an extra cost, it's a natural result of clean structure.
4. **Mechanism low, policy high.** The lowest layers fix only the minimal, most general mechanisms; everything that will evolve ("policy") stays in a replaceable soft layer.
5. **Inter-layer contracts must be explicit, measurable, and verifiable.** Every interface has a formal contract, enabling both cross-layer correctness verification and cross-layer optimization.

The eight layers follow.

## 1. Instruction set and chip: fresh design, co-designed with microarchitecture

**Methodology borrowed from RISC-V (only this):** open and royalty-free, modular, a formal specification and a "golden reference model" from day one, clean and regular encoding.

**RISC-V assumptions firmly *not* inherited:**

- **The target is reversed.** The new ISA **aims from the start at modern wide-issue out-of-order + SMT + heterogeneity**, with the ISA and microarchitecture designed together — not an ISA frozen first and hardware forced to accommodate it.
- **Kill fragmentation with "Profiles."** No "minimal base + a pile of optional extensions," which fragments. Instead define **a small number of complete profiles**, each a fully guaranteed capability set — no à-la-carte, partial combinations.
- **Encoding designed for modern decoders and macro-op fusion.** Encoding width, alignment, and parallel-decodability are all decided together with the decoder.
- **Exactly one strong memory consistency model.** No "weak/strong optional." Pick a model that is **both high-performance and reasoned-about** (a release-acquire system) and pin it down with a formal spec.
- **Data-parallel / vector / tensor are first-class citizens, not a late extension.** Vector-length-agnostic (VLA) + tensor primitives are unified with scalar from day one.
- **Capability addressing native in hardware (the CHERI route), but the reason is performance.** Pointers carry their own bounds/tags, making out-of-bounds access impossible in hardware. **The key selling point is not "more secure" but that it saves software a whole class of runtime checks** — a win for both performance and correctness; the cost (wider pointers, tag bits, a little silicon area) is stated openly.
- **Unified heterogeneous addressing.** CPU/GPU/NPU/DPU **share one coherent, byte-addressable virtual address space**; accelerators are equal citizens that directly access the same memory, eliminating the "copy back and forth across PCIe" tax.

## 2. Memory and interconnect: collapse the "zoo of parts" into one co-designed fabric

- **A unified coherent memory fabric.** All compute units see one coherent, byte-addressable memory space; **near-memory / in-memory compute primitives** are exposed directly to software.
- **Express the storage hierarchy as "one address space + placement hints."** High-bandwidth, main-memory, and persistent tiers all live in the same address space, scheduled via explicit placement hints.
- **One network only.** Merge today's zoo of independent standards — PCIe / Ethernet / NVLink / CXL — into **one co-designed switched fabric that carries coherence and identity**, continuous from on-chip all the way to rack and datacenter. **Memory interconnect and networking are one continuum.**

## 3. Operating system: a formally verified microkernel, but tuned for performance

- **A minimal trusted base + a formally verified kernel (seL4-class)**, but **IPC performance is the make-or-break line**: make **zero-copy, capability-passing IPC** the kernel's central primitive.
- **Capabilities throughout, isomorphic to hardware capabilities.** OS capabilities map directly to hardware capabilities, not software-emulated.
- **Asynchronous, message-passing, batched submission by default.** Drop "synchronous syscall trap" as the hot-path default; **io_uring-style batched async submission/completion** becomes the native ABI.
- **Declarative, reproducible system state (the Nix idea) as the native config model** — the whole machine state is a rollback-able, reproducible description.
- **Schedulers that understand heterogeneity and energy.** When scheduling across big / efficiency cores and accelerators, **treat energy as a first-class optimization objective.**
- Abandon the over-generalization of "everything is a file" in favor of **typed objects with capabilities**; keep a small, uniform naming mechanism.

## 4. Network: identity and encryption are protocol primitives

- **Every endpoint has a cryptographic identity, every packet is inherently authenticatable, and end-to-end encryption is a low-level primitive** — eliminating the whole patch stack of TLS / VPN / NAT.
- **Content addressing and location addressing coexist, with native mobility:** connections are bound to identity rather than IP, so switching networks doesn't drop them.
- **Transport and congestion control are co-designed with the layer-2 fabric**, a continuum from on-chip to wide area.
- The data plane is **programmable but memory-safe by default** (the power of eBPF, but designed in).

## 5. Languages, compilers, runtime: one model from silicon to application

- **One portable, verifiable intermediate representation (IR)** (the WASM idea, but spanning the full range from kernel to app), with **capabilities and that strong memory model compiled directly into the IR.** Compile once, run across any ISA profile.
- **A memory-safe systems language by default** (Rust-class ownership, but with less accidental complexity and better ergonomics). Performance-oriented: zero-cost abstractions, no mandatory GC at the system level; managed runtimes as an optional upper layer.
- **The IR carries capability and effect information**, so the OS/hardware can enforce least privilege at **zero runtime cost** — language, OS, and hardware **share one capability model**, a single mental model end to end.
- **Reproducible builds are an inherent property of the toolchain.**

## 6. Data and storage: storage is a tier of the memory fabric, not another world

- **Content-addressed + immutable + versioned by default; encryption by default, keys owned by the user.**
- **Storage is the persistent tier of the memory fabric**, not "block device + filesystem" stacked up. One data model: **typed, content-addressed objects with history**; "files" and "databases" degrade into two views on top of this layer.
- **Structure and queryability are first-class capabilities** — structured data is no longer an app-private silo.

## 7. Identity, trust, data ownership (cross-cutting)

- **Decentralized identity:** users hold their keys; identity is not bound to a platform.
- **Personal data vaults:** an app receives **scoped, revocable capabilities to the user's data**, rather than each hoarding a copy.
- This is also a **performance and architecture simplification**: less data duplication, less synchronization machinery.

## 8. Application and interaction layer

- **Apps are capability-scoped and sandboxed by construction** — cheap, because the whole stack supports it.
- **Composable, data-centric applications over a shared data model:** an app is a "view/lens" over the user's own data rather than a silo — folding away today's integration / API / sync complexity.
- **A unified, capability-mediated extension and automation model:** making scripting, automation, and AI agents **first-class and safe** citizens.

## 9. Failure risks to face honestly

- **The deepest contradiction: co-design requires strong up-front centralized coordination, and centralization is exactly what kills adoption.** The "worse is better / network effects" hurdle is the thing most likely to stillbirth the whole system.
- **Capability hardware has a real silicon cost** (wider pointers, tag bits, area), offset by "saving software checks" — the net gain depends on the workload.
- **One memory model / one fabric = less room for divergent innovation;** mitigate with spec versioning + profiles.
- **Declarative/reproducible everywhere can feel rigid** — escape hatches are needed.
- **One IR everywhere has an abstraction tax** — must be pressed down with aggressive AOT + profile-guided compilation.
- **Decentralized identity + personal data vaults will hit the "key management / UX" problems** that have repeatedly sunk similar attempts.

## In one sentence

The through-line of the whole system: **co-design the contracts between layers around "energy-efficient performance" and "the elimination of accidental complexity"; let one coherent model — capabilities + a strong memory model + content-addressed data + cryptographic identity — run unbroken from silicon to application; and wherever that model would sacrifice performance, pay that cost only by an explicit decision.** Security, simplicity, and evolvability are all by-products of this through-line, not goals standing alongside performance.
