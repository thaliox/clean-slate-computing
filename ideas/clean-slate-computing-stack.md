# Designing the Entire Information-Age Stack From Scratch

`English | `[中文](clean-slate-computing-stack.zh-CN.md)

> Collected June 2026. Exploratory discussion. Premise: set aside hardware devices and manufacturing for now.

## Original question

> If you could design all the hardware of the information age from scratch — computers, servers, network devices, operating systems, applications — how would you plan it? You may ignore the hardware-device and manufacturing problems for now.

## Answer

The reason today's computing stack is "messy" is mostly not that the people back then were foolish — it's the accumulation of decades of incremental evolution, backward-compatibility baggage, and network effects. Many more elegant designs lost, not on technical merit, but on ecosystem. So the real opportunity in a from-scratch design is to **turn what is "patched on after the fact" today into what is "built in from the foundation."**

### I. Design philosophy that runs through the whole stack

1. **Security is the foundation, not a patch.** Today's security (TLS, sandboxes, permissions, antivirus) is almost all bolted on, because the underlying assumption is "trust by default." Redone, you would make **zero-trust by default, encryption by default, least-privilege by default** the factory state of every layer, rather than something you have to configure.

2. **Replace "identity + permissions" with "capabilities."** Today it's "who are you → check whether you have permission" — a long chain that is easy to misconfigure. A better model: **whoever holds a capability token can do the corresponding thing**, with permissions passed along with objects, revocable, and auditable. This eliminates a huge class of privilege-escalation and unauthorized-access bugs at the root.

3. **Memory safety and formal verification as the baseline.** The vast majority of severe vulnerabilities today come from memory unsafety (C/C++). Redone, the system level defaults to a memory-safe language, and core components (kernel, crypto libraries) are **formally verified** (seL4 has already proven that a microkernel can be mathematically proven bug-free).

4. **Simplicity first; resist "accidental complexity."** Every layer should be understandable by one person in a reasonable amount of time. Complexity is the enemy of security and maintainability.

5. **Data belongs to the user; applications request access.** Today every app hoards its own copy of your data. Flip it: **the user owns their own data vault, and applications request capabilities to read/write it** — data is portable and authorization is revocable.

### II. Layer-by-layer plan

**Instruction set / chip architecture layer**
A single **open, lean, unified** RISC instruction set (in the spirit of RISC-V), avoiding the licensing shackles and historical baggage of x86. The key is to push the security mechanisms that software does today down into hardware: **hardware-level capability addressing and memory safety** (in the direction of CHERI), so that an "out-of-bounds access" is simply impossible in hardware. At the same time, CPU/GPU/NPU share a **unified memory model** from day one, rather than each managing its own and shuffling data back and forth by copying.

**Operating system layer**
**Microkernel + capability security.** The kernel keeps only a minimal trusted base (scheduling, memory, IPC); drivers and services run in isolated user space — a driver crashing won't take down the system. Abandon Unix's root/ordinary-user binary model and the over-generalization of "everything is a file," in favor of **unified objects + capability authorization.** System configuration follows a **declarative, reproducible** route (the NixOS idea): the entire system state is a rollback-able, reproducible description, not a pile of mutable state fighting each other.

**Network layer**
The original sin of IP today is that it has **no built-in identity or encryption**, so TLS, IPsec, VPN, and NAT are all patches. Redone: **identity and end-to-end encryption are network-layer primitives**, every packet inherently authenticatable; **mobility is natively supported** (switch WiFi or cell tower without dropping connections); **content addressing and location addressing coexist** — you want "this piece of data," without caring which machine it's on. This incidentally solves a whole pile of today's problems: DDoS, address exhaustion, content distribution, and more.

**Programming / application layer**
**Memory-safe languages by default.** All programs run on a **portable intermediate representation** (the WASM idea), compiled once and running seamlessly across any ISA — this completely eliminates today's porting nightmare between x86 and ARM. Every application must **explicitly declare the capabilities it needs** (network? camera? which data vault?), the system sandboxes it accordingly, and the user can see at a glance exactly what an app can touch.

**Data / storage layer**
**Content-addressed + immutable + versioned** by default: data is identified by content hash, giving natural deduplication, verifiability, and traceable history; **encryption by default**, with keys belonging to the user. Backup, sync, and collaboration all become natural capabilities of this layer, rather than each application reinventing the wheel.

**Identity and trust layer (cross-cutting)**
**Decentralized identity** built in: users hold their own keys, and identity is not bound to any single platform. Login, signing, and authorization are all based on cryptography, not "account + password + a pile of centralized databases."

### III. Tensions you must face honestly

For an ideal design to actually land, these contradictions are unavoidable:

- **Unification vs. innovation.** A fully unified stack-wide standard eliminates fragmentation, but **over-unification stifles diversity and experimentation.** The healthy approach is to unify "interfaces and protocols," not "implementations."
- **Elegance vs. adoption.** History repeatedly proves "worse is better" — crude but usable and easily spread things often beat elegant designs. No matter how good the architecture, if it can't get over the hurdle of ecosystem and network effects, it's dead.
- **Security/simplicity vs. performance/convenience.** Capability checks, encryption, and isolation all have costs; strictest-by-default sacrifices some performance and usability, requiring repeated trade-offs in hardware acceleration and default values.
- **Centralized coordination vs. decentralized resilience.** Designing from scratch requires strong coordination (to unify), but the finished product is best decentralized (for resilience). "Designing a decentralized system in a centralized way" is itself awkward.
