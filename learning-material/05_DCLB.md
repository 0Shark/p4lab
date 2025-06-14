# DCLB

## 1. Loadbalancing Fundamentals

### Motivation
- Modern datacenters use multi-stage topologies (Clos, fat-tree) that provide multiple **equal-cost paths** between hosts.
- The goal of load balancing is to efficiently distribute traffic across these paths to maximize bandwidth utilization and avoid network hotspots.

### Loadbalancing Granularity
There are three main ways to split traffic:

1.  **Per-Packet:**
    - **Pros:** Very fine-grained, allowing for near-perfect load distribution.
    - **Cons:** Causes **TCP packet reordering**, which is misinterpreted as packet loss by TCP's congestion control, severely degrading throughput.

2.  **Per-Flow (Used by ECMP):**
    - **Pros:** Assigns an entire flow (identified by its 5-tuple) to a single path, which **prevents packet reordering**.
    - **Cons:** Coarse-grained. A single large "elephant" flow can't be split. **Hash collisions** can map multiple flows (including large ones) to the same path, creating congestion while other paths are underutilized.

3.  **Per-Flowlet (Used by CONGA):**
    - A **flowlet** is a burst of packets within a flow, separated by an idle time gap.
    - **Pros:** A good compromise. It allows large flows to be split into smaller chunks, improving load balancing.
    - **Cons:** Can avoid reordering only if the idle gap is larger than the difference in Round-Trip-Time (RTT) between paths. Requires more sophisticated logic in the switch.

---

## 2. ECMP (Equal-Cost Multipath)

### Core Concept
- ECMP is a widely-used, **stateless**, flow-based load balancing mechanism.
- It distributes traffic based on a static hash, not on real-time network conditions.

### How ECMP Works
1.  When a packet arrives, the switch calculates a **hash** over its header fields (typically the 5-tuple: Src/Dst IP, Src/Dst Port, Protocol).
2.  It then performs a **modulo operation** on the hash result (`hash % number_of_available_paths`).
3.  The result of the modulo is an index that deterministically selects which next-hop path to use.
- Because the hash is consistent for a given flow, all its packets take the same path.

### Key Problems with ECMP
- **Static & Congestion-Agnostic:** It has no knowledge of link congestion. It will continue sending traffic down a congested path.
- **Hash Collisions & Coarse Granularity:** Multiple flows, including large "elephant" flows and small "mice" flows, can be hashed to the same path, leading to significant latency increases and unfairness.
- **Poor Performance with Asymmetry:** Inefficient when paths have different capacities or when failures occur.

### ECMP in P4 (Conceptual Implementation)
- **`ecmp_group` table:** Maps a destination IP prefix to a group of potential next-hops.
- **`set_ecmp_select` action:**
    - Takes the 5-tuple from the packet header.
    - Performs a `hash()` operation.
    - Uses the hash result to select a specific member from the ECMP group.
- **`ecmp_nhop` table:** Maps the selected member index to the actual next-hop details (e.g., destination MAC address, egress port).
- **`set_nhop` action:** Rewrites the packet headers (dst MAC, TTL) and forwards it to the chosen egress port.

---

## 3. CONGA (Distributed Congestion-Aware Load Balancing)

### Motivation
- To overcome the limitations of ECMP by making load balancing decisions based on real-time, network-wide congestion information.
- **Key Insight:** Local congestion awareness alone is insufficient and can be worse than ECMP. A **global view** of path congestion is needed. CONGA achieves this with a fast, distributed control loop directly in the **dataplane**.

### How CONGA Works
1.  **Overlay-based:** Operates over a standard DC overlay like **VXLAN**, using its header to carry metadata.
2.  **Leaf-to-Leaf Feedback:**
    - Leaf (Top-of-Rack) switches track congestion to all other leaves.
    - A rate measurement module at each switch port calculates a small congestion value (`CE`).
    - This `CE` value is **piggybacked** onto data packets (e.g., in the VXLAN header) and sent to the destination leaf.
3.  **Real-time Congestion Tables:**
    - Each leaf switch receives `CE` values from all other leaves, continuously updating a **congestion table**.
    - This table provides a near real-time view of the congestion on *every path* to *every other leaf*.
4.  **Flowlet-based LB Decision:**
    - For each new **flowlet**, the source leaf switch consults its congestion table.
    - It sends the flowlet down the path to the destination leaf that is currently the **least congested**.

### Why CONGA is Stable
- **Extremely Fast Feedback Loop:** Congestion information is updated in microseconds, directly via data packets.
- **Fine-Grained Adjustments:** Decisions are made at the flowlet level.
- This combination of **near-zero latency feedback + flowlet-level adjustments** creates a stable and highly efficient load-balancing system without needing a complex, slow control plane.