# DCLB_Advanced

These are study notes summarizing the presentation on HULA (Hop-by-hop Utilization-aware Load-balancing Architecture).

### 1. The Problem: Scalable Datacenter Load Balancing

- **Goal:** Distribute traffic evenly across network paths to avoid congestion, minimize flow completion times (FCT), and maximize throughput.
- **Challenges:**
    - **Scalability:** Solutions must work in large, multi-tier network topologies.
    - **Reordering:** Packet-level balancing can reorder packets within a TCP flow, causing performance degradation.
    - **Congestion Blindness:** Simple methods like ECMP are unaware of network congestion, leading to poor path choices.

### 2. Load Balancing Granularity (Recap)

- **Flow-based:** Hashes a flow's 5-tuple to a path.
    - **Pros:** No packet reordering.
    - **Cons:** Inflexible. Large "elephant" flows can congest a single path while others are underutilized (hash collisions). Lowest granularity.
- **Packet-based:** Spreads individual packets across paths.
    - **Pros:** Highest granularity, fine-grained balancing.
    - **Cons:** Causes packet reordering, which harms TCP performance.
- **Flowlet-based:** A practical compromise.
    - A **flowlet** is a burst of packets from the same flow, separated by a sufficiently large time gap.
    - A switch can send different flowlets from the same flow on different paths without causing reordering, provided the gap is larger than the maximum path delay difference.
    - This strikes a balance between granularity and reordering.

### 3. HULA vs. Other Architectures

| Scheme | Congestion Aware | Scalable | Programmable | Key Idea |
| :--- | :---: | :---: | :---: | :--- |
| **ECMP** | No | Yes | No | Static hashing of flows to paths. |
| **CONGA** | Yes | No | Yes | Measures and maintains congestion for *all* paths between leaf pairs. State doesn't scale to large topologies. |
| **HULA** | Yes | **Yes** | Yes | Uses **summarized state** and **hop-by-hop decisions**. Each switch only needs to know the *best next hop* to a destination, not the state of all possible paths. |

HULA's main innovation is achieving congestion-aware load balancing that **scales to large, arbitrary topologies**.

### 4. How HULA Works

HULA operates in two main phases: path probing to gather congestion information and data forwarding to route flowlets.

#### Phase 1: Path Probing & Best-Path Identification

This phase works like a **distance-vector routing protocol** but for path utilization instead of hop count.

1.  **Probe Origination:** Each Top-of-Rack (ToR) switch periodically generates special **HULA probe packets** for every other ToR in the network.
2.  **Probe Contents:** A probe carries information like:
    - `origin_ToR_ID`: The source of the probe.
    - `max_util`: The *maximum link utilization* seen so far on its path (initialized to 0). In the exercise, this is `qdepth`.
3.  **Forward Propagation (`dir=0`):**
    - Probes are forwarded upstream towards the destination ToR.
    - At each hop, the switch **compares its egress link utilization** to the `max_util` value in the probe. If its local utilization is higher, it **updates the `max_util` field in the probe header**.
    - When a probe reaches the destination ToR, the `max_util` field contains the utilization of the most congested link on that specific path.
4.  **Reverse Propagation (`dir=1`) & Table Updates:**
    - The destination ToR sends the probe back towards the origin ToR.
    - As this reverse probe travels, each switch on the path inspects it.
    - A switch will receive multiple reverse probes for the same `origin_ToR` (one from each downstream path). It **compares their `max_util` values**.
    - It identifies the probe with the **minimum `max_util`**, as this represents the least-congested path.
    - The switch then updates its local "best hop table" (a P4 register), mapping `Destination_ToR -> best_next_hop_port`.

This mechanism ensures every switch dynamically learns the best *next hop* to reach any destination ToR based on real-time congestion, without needing global topology knowledge.

#### Phase 2: Data Packet Forwarding

1.  Data packets (organized as flowlets) travel in the opposite direction of the initial probes.
2.  When a switch receives a packet, it identifies the flowlet.
3.  **If it's a new flowlet** (or the flowlet timer has expired):
    - The switch looks up the packet's destination in its **best hop table** (populated by the probes).
    - It forwards the packet out the corresponding best next-hop port.
    - It caches this decision (flow hash -> egress port) in a **flowlet table**.
4.  **If it's part of an existing flowlet:**
    - The switch uses the cached egress port from the flowlet table to ensure all packets in the same flowlet follow the same path.

### 5. P4 Implementation Concepts

HULA's logic is implemented directly in the P4 data plane.

-   **Custom Header:** A `hula_t` header is defined for probe packets, containing fields like `dir`, `qdepth` (utilization), and `digest` (path ID).
-   **Control Logic (`apply` block):**
    - **Differentiates traffic:** The logic first checks if a packet is a HULA probe or a regular IPv4 data packet.
    - **Probe Handling:**
        - **Forward (`dir==0`):** At the destination ToR, compare queue depths and prepare a reverse probe. In the core, update the `qdepth` field if local queue depth is higher.
        - **Backward (`dir==1`):** Update the best-hop routing table (`hula_bwd`) and drop the probe if it reaches its origin (`hula_src`).
    - **Data Packet Handling:**
        - Implement flowlet switching logic.
        - Use the `hula_nhop` table to look up the best next hop from the `dstindex_nhop_reg` register.
-   **Stateful Memory (Registers):**
    - `dstindex_nhop_reg`: The "Best hop table". Stores the best egress port for each destination. Updated by probes at line rate.
    - `flow_port_reg`: The "Flowlet table". Caches the chosen path for active flowlets.

### 6. Conclusion

HULA is a powerful data plane load balancing mechanism that is:
-   **Adaptive:** Reacts to real-time network congestion.
-   **Scalable:** Avoids centralized bottlenecks and large state tables by using summarized, hop-by-hop information.
-   **Programmable:** Can be implemented on modern programmable switches using P4.

It provides near-optimal performance by intelligently routing flowlets along the least-congested paths available in the network.