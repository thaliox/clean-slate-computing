# Single-Layer Deep Dive: Memory and Interconnect

`English | `[中文](layer-deep-dive-memory-interconnect.zh-CN.md)

> Collected June 2026. Exploratory discussion. Premise: set aside hardware devices and manufacturing for now.
>
> This is an in-depth expansion of layer 2 of the [complete blueprint](clean-slate-computing-full-blueprint.md), directly catching the problem the [ISA deep dive](layer-deep-dive-isa.md) offloaded: a unified heterogeneous address space requires coherence to reach the accelerators, and this layer is responsible for delivering it.

## 1. The contract this layer must deliver

The layer above (the ISA) made three promises, all billed to this layer:

- **One coherent, byte-addressable address space** spanning all compute units (CPU/GPU/NPU/DPU) and all memory tiers (high-bandwidth / main-memory / persistent);
- **Capabilities and tags must survive in memory and be passed along**;
- **It is a continuum with the network layer** — from on-chip NoC to rack to datacenter, semantics are uniform.

The hardest part: **coherence must extend to hundreds or thousands of agents plus high-bandwidth accelerators**, and classic coherence simply doesn't scale to that. This layer lives or dies here.

## 2. The core difficulty: coherence doesn't scale up

Full hardware cache coherence (snoop / directory) is the easiest model to program, but **directory state and coherence traffic explode with the number of agents** — it can't reach thousands of nodes plus accelerators. The reason GPUs today use relaxed/scoped coherence is exactly this.

My choice is **tiered + scoped coherence**, and crucially **co-designed with the ISA memory model** rather than starting over:

- **Within a coherence domain**: full hardware coherence, fast — a node/socket/a few sockets form one domain, and naive code is correct within the domain by nature.
- **Across domains**: explicit acquire/release at the correct **scope**. This "scope" is **an architectural concept visible to software**, not black magic hidden in the microarchitecture.
- The directory is **sparse + hierarchical**, with coherence messages riding the fabric below.
- The overall idea remains "**strong within domain, explicit across domains**," fully consistent with the ISA's "strong by default, weak by opt-in."

## 3. Extending the memory model: scopes

How does the ISA deep dive's "one memory model" survive on a heterogeneous fabric? By **adding scopes to release/acquire**:

- Define hierarchical scopes: **thread / core / coherence domain / whole system**.
- Each release/acquire annotates which level it acts at (like a GPU scoped memory model, but **unified and pinned down by a formal spec**).
- This neither requires "global coherence" (which can't scale) nor exposes the programmer to more than one set of scoped release/acquire rules — reasonability is preserved.

This is the only way "one memory model" can physically hold: **turn "distance" into a scope parameter, not into a second set of semantics.**

## 4. Address translation and tag storage

- **One virtual address space, unified translation**: CPU and all accelerators share page tables/translation; IOMMU and MMU merged. Capabilities operate in virtual space, with translation beneath.
- **The real cost of translation at scale**: accelerators must also translate addresses (like today's ATS/PRI), creating heavy **TLB pressure**; mitigated with huge pages, range translation, and shared translation caches.
- **Tag storage is an unavoidable bill on this route**: 1 tag bit per 128-bit capability slot is roughly **0.8% memory overhead**. Implementation either tucks it into the DRAM ECC sideband or carves out a dedicated **tag table + tag cache**. The tag must survive in physical memory too and must be unforgeable by ordinary writes — the hardware precondition that makes a pure-capability architecture hold.

## 5. Memory tiers: one address space + placement hints

- **The high-bandwidth, main-memory, and persistent tiers all live in the same address space**, not each with its own API.
- **Placement is via hints/policy**, with inter-tier migration handled by hardware + OS (like autonuma, but designed in, not bolted on).
- **Make persistence clean**: define **persistence domains** and fold the ordering semantics of flush/commit **into the memory model** (acquire/release + a persist/commit ordering). The reason pmem programming is painful today is that persistence ordering was jammed in after the fact; here it grows together with the consistency model from the start.

## 6. Near-memory compute: saving data movement is saving energy

This directly serves the "energy efficiency first" master goal.

- **Moving data costs far more than computing on it** (the energy of one cross-chip move dwarfs that of one arithmetic op), so the fabric should allow **pushing compute next to the data**.
- Expose **near-memory primitives**: memory-side atomics, reductions, scatter/gather, even small-scale offload.
- This is one of the biggest energy levers: for many workloads the bottleneck and the power draw are in data movement, not in compute.

Cost: near-memory compute **complicates the consistency story** — computation happening at memory must also participate in the memory model; it can't be a "ghost write" outside the model.

## 7. Interconnect: one layered fabric replacing the zoo

Merge today's pile — PCIe / Ethernet / NVLink / CXL / InfiniBand — into **one layered fabric**:

- **Layered**: a physical/link layer (SerDes/lanes — manufacturing ignored here) + a **transaction layer**. The transaction layer uniformly carries: coherent memory transactions, one-sided RDMA-style ops, and message passing — **all with identity and capabilities.**
- **Switched, not a bus**; flexible topology (mesh / dragonfly, etc.).
- **Identity-bearing**: every agent and every packet is authenticatable (connecting to the network layer's identity primitive). **DMA is capability-checked** — an accelerator can only touch memory it holds a capability for, eliminating at the root the whole class of today's DMA attacks / unauthorized access.
- **A continuum**: from on-chip NoC to rack to datacenter, **one transaction model**; latency and bandwidth differ, but **semantics are uniform.** "Distance" is a performance property, not a semantic one.

## 8. Congestion, QoS, isolation

- At fabric scale there must be **congestion control co-designed with the transport layer**: credit-based flow control, in-network congestion signaling.
- **Inter-tenant QoS and isolation**: naturally supported by capabilities + identity — how much bandwidth someone may use, whether they may touch a memory region, are all decided by capabilities.

## 9. Reliability at scale

- More components, more failures. **End-to-end integrity** (checksums, ECC) is the baseline; the upper layer's (layer 6) content-addressed, immutable data model helps (data is verifiable and re-fetchable).
- The coherence fabric must **gracefully handle link failures/retries**, with clear **fault domains** so one domain going down doesn't drag the whole system.

## 10. Trade-offs that must be accounted for

- **The core tension: full coherence can't scale → scoped coherence pushes complexity back to software.** Although mitigated by "co-design with the memory model," the programmer ultimately faces scopes — a real cost.
- **Unified address space + capabilities + tags = translation and tag-storage overhead**; TLB/tag-cache pressure is concrete.
- **One fabric is a huge coordination/standardization bet** — the "worse is better / incumbents" risk is especially acute at this layer, because the real world has many entrenched interconnect standards.
- **Near-memory compute makes consistency more complex**: memory-side computation must obediently enter the model.
- **Persistence ordering** adds another layer of complexity to the otherwise clean memory model.
- **The latency continuum is "semantically uniform but performance non-uniform"**: programmers can write "correct but catastrophically slow" code that ignores locality (a planet-scale NUMA problem). Good **locality hints and observability** are mandatory, or correctness holds while performance collapses.

## In one sentence

This layer's job is to **deliver the beautiful promise of "one coherent address space" against the physical reality that it can't scale**: catch the ISA's memory model with "strong coherence within domain + scoped across domains," replace the interconnect zoo with "one layered fabric bearing identity and capabilities," and use near-memory compute to save the data movement that dominates energy; wherever physics forces a sacrifice (scope complexity, tag overhead, non-uniform performance), put it out in the open for software to face explicitly.
