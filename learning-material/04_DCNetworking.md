# DCNetworking

## 1. The "Why": Datacenters for Web-Scale Services

- **Web Search Problem:** A single search query triggers a massive, distributed computation that cannot be handled by one or even a few servers.
- **Answer:** **Datacenters** are used. They are essentially "warehouse-scale computers" built from millions of commodity machines.
- **Key Function:** Provide massive amounts of **Compute (CPU)**, **Storage (disk)**, and handle massive **network traffic** through a tightly integrated system.

## 2. The Importance of Network Performance

- **Performance is Crucial:** Network latency directly impacts revenue and user engagement.
  - **Google:** An extra 500ms of latency can cause a **20% traffic reduction**.
  - **Amazon:** An extra 100ms of latency can cost **1% in revenue**.
- **Communication is a Bottleneck:** For distributed applications like Facebook analytics, communication can account for **33% of job runtime**. The network MUST scale.

## 3. Understanding Datacenter Communication Patterns

### Coflows and Incast

- **Application Flow:** A sequence of packets between two endpoints.
- **Coflow:** A collection of parallel, related flows that work together to complete a computational job (e.g., the "scatter-gather" pattern in a web search).
- **Coflow Problem:** A job's completion time is determined by its **slowest flow**.
- **TCP Incast:** A critical performance problem where many servers send data to a single receiver simultaneously. This overwhelms the small buffers on the Top-of-Rack (ToR) switch, leading to:
  - Packet loss
  - TCP timeouts (RTO) and bursty retransmits
  - **Throughput collapse (up to 90% drop)**

### Traffic Characteristics

- **Elephants and Mice:**
  - **Mice Flows:** Numerous, small, and often latency-sensitive.
  - **Elephant Flows:** Few, but carry the vast majority of the total data volume.
- **Locality:** Most traffic is **internal** to the datacenter (East-West traffic), rather than going to the outside internet (North-South).
- **Concurrency:** Servers handle hundreds or thousands of concurrent connections.
- **Challenge:** Traffic is highly volatile and unpredictable, requiring frequent and rapid optimization.

## 4. Datacenter Network Architectures

### Main Requirements

- High Availability
- High Efficiency
- Smart Agility & Elasticity
- Security

### Conventional (Tiered) Architecture

- **Structure:** 3-layers of switches: **Core**, **Aggregation**, and **Edge** (or Top-of-Rack, ToR).
- **Problems:**
  - **Layer 2 Issues:** Spanning Tree Protocol (STP) is inefficient and blocks potentially useful paths; ARP broadcast storms don't scale.
  - **Layer 3 Issues:** Complex configuration; migrating VMs/containers is difficult without changing IP addresses.
  - **Oversubscription:** The biggest problem. To save costs, there are fewer high-bandwidth uplinks than downlinks (e.g., 20:1 from ToR to aggregation). This creates a major bottleneck, limiting server-to-server bandwidth.

### Modern Topologies: Leaf-Spine (Fat-Tree)

- **Goal:** Approximate one massive, non-blocking, and affordable "big switch" using cheap commodity hardware.
- **Design:** A multi-rooted tree structure (specifically a **Clos network**) with two layers: **Leaf** (ToR switches) and **Spine** (core switches).
  - Every leaf switch connects to every spine switch.
- **Benefits:**
  - **Massive Bisection Bandwidth:** All servers can communicate with each other at high speed.
  - **Multiple Paths:** Provides many equal-cost paths between any two servers, increasing capacity and fault tolerance.
- **Challenge:** Standard routing can only use one path. **Load balancing** is required to utilize the full capacity.
  - **Packet-based load balancing (e.g., ECMP)** can spray packets from a single flow across multiple paths. This causes **packet reordering**, which severely degrades TCP performance.

## 5. Summary: Implications & Challenges

Datacenter networking must solve for:

1.  **Massive internal traffic** volume.
2.  **Congestion** from coflows (TCP Incast).
3.  The need for **low latency** and **high capacity** simultaneously.
4.  The need for **adaptive and fine-grained load balancing** to avoid packet reordering while maximizing path utilization.
5.  All of this must be **cheap, scalable, and fault-tolerant**.