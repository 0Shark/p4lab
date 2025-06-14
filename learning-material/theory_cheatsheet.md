# DATACENTER NETWORKING THEORY CHEAT SHEET

## 1. CORE NETWORKING PRINCIPLES
**Packet Switching** • Statistical multiplexing • **Best-Effort Delivery** (packets can be lost/delayed/out-of-order)  
**Internet Hourglass** • IP is universal waist • **Scalability** via hierarchy + indirection

### OSI LAYERS
| L | Name | PDU | Function |
|---|------|-----|----------|
| 7 | Application | Message | HTTP, SMTP |
| 4 | **Transport** | Segment | **End-to-end** (TCP/UDP) |
| 3 | **Network** | Packet | **Host-to-host routing** (IP) |
| 2 | **Data Link** | Frame | **Hop-to-hop** (Ethernet, MAC) |
| 1 | Physical | Bit | Raw transmission |

**L2**: MAC addresses (48-bit), ARP, DHCP  
**L3**: IP addresses (32/128-bit), TTL, fragmentation  
**L4**: Port multiplexing, UDP (connectionless), TCP (reliable, flow/congestion control)

### ROUTING
**Control Plane** (brain) → routing table → **Data Plane** (muscle) → forwarding table  
**IGP** (intra-domain): OSPF (link-state, Dijkstra)  
**EGP** (inter-domain): BGP (path-vector, policy-based)

## 2. DATACENTER FUNDAMENTALS
**Problem**: Web-scale distributed computation requires massive compute/storage/network  
**Impact**: 500ms latency = 20% traffic drop (Google), 100ms = 1% revenue loss (Amazon)

### TRAFFIC PATTERNS
**Coflows** • Collection of parallel flows • Job completion = slowest flow  
**TCP Incast** • Many→one overwhelms ToR buffers → 90% throughput collapse  
**Elephants vs Mice** • Few large flows carry most data, many small flows need low latency  
**East-West** >> North-South traffic

### ARCHITECTURES
**Traditional 3-tier**: Core→Aggregation→Edge (ToR) • **Problems**: STP inefficiency, oversubscription bottlenecks  
**Leaf-Spine (Fat-Tree)**: Every leaf↔every spine • **Benefits**: massive bisection bandwidth, multiple paths

## 3. LOAD BALANCING
**Challenge**: Utilize multiple equal-cost paths without packet reordering

### GRANULARITY
**Per-Packet**: Perfect load distribution but causes TCP reordering  
**Per-Flow (ECMP)**: Hash 5-tuple, prevents reordering but coarse (elephant flows)  
**Per-Flowlet (CONGA/HULA)**: Split flows at idle gaps, best compromise

### ECMP (Equal-Cost Multipath)
Static hash mod num_paths • Hash = f(5-tuple) % paths  
**Problems**: congestion-agnostic, hash collisions, poor with asymmetry

### CONGA (Congestion-Aware Load Balancing)
**Global congestion awareness** • Piggyback congestion info in VXLAN • Real-time congestion tables  
**Stability**: μs feedback loop + flowlet-level adjustments  
**Limitation**: Doesn't scale (leaf-to-leaf state = O(n²))

### HULA (Hop-by-hop Utilization-aware Load-balancing)
**Scalable congestion awareness** • Distance-vector for utilization instead of hops  
**Phases**: (1) Path probing with max_util tracking (2) Flowlet forwarding  
**Key**: Each switch only tracks best next-hop per destination (O(n) state)  
**Algorithm**: Forward probes track max congestion, reverse probes update best-hop tables

## 4. MONITORING & TELEMETRY
**Traditional Problems**: Polling too slow, CPU stress, sampling loses detail, no end-to-end view

### IN-BAND NETWORK TELEMETRY (INT)
**"Million Minions"** • Data packets collect telemetry at line rate  
**Roles**: Source (adds INT header), Transit (appends metadata), Sink (strips & reports), Monitor (analyzes)  
**Metadata Stack**: LIFO, first entry = last hop  
**Challenge**: Where/When/How to collect (overhead scales linearly)

## 5. CACHING ALGORITHMS

### TRADITIONAL PROBLEMS
**Incast Problem**: Single request → hundreds of cache servers → response flood  
**Fan-out**: High latency due to multiple server contacts

### NETCACHE (In-Network Caching)
**Concept**: Move caching into P4 switches (ToR as front-line cache)  
**Benefits**: Absorbs hot traffic, μs latency, reduces fan-out  
**Challenge**: Variable-length values → split across multiple register arrays

### PROBABILISTIC DATA STRUCTURES
**Bloom Filter**: Membership testing, k hash functions → k bits  
- Properties: No false negatives, allows false positives, space-efficient  
- Cannot delete items (use Counting Bloom Filter for deletion)

**Count-Min Sketch**: Frequency estimation, d arrays with hash functions  
- Update: Increment d counters, Query: Return minimum of d values  
- Used for hot-key detection (min because collisions only over-count)

**Cache Update Flow**: (1) P4 reports hot keys (2) Control plane decides (3) Fetch values (4) Insert/evict

### CACHE REPLACEMENT POLICIES
**LRU**: Least Recently Used - good temporal locality  
**LFU**: Least Frequently Used - good for stable workloads  
**Random**: Simple, surprisingly effective for some workloads

## 6. ADVANCED CONCEPTS
**In-Network ML**: Inference in switches, distributed training acceleration  
**Security**: DDoS protection, encryption, access control

### KEY ALGORITHMS
**Dijkstra**: Single-source shortest path (O(V²) or O(E+V log V) with heap)  
**ECMP Hash**: hash(5-tuple) % num_paths for consistent flow→path mapping  
**Sketch-based**: Probabilistic monitoring (FlowRadar, UnivMon)  
**Bloom Filter**: Insert: set k bits, Query: check all k bits  
**Count-Min Sketch**: Update: increment d counters, Query: min(d counters)  
**HULA Probing**: Forward (update max_util), Reverse (update best-hop tables)

### PERFORMANCE METRICS
**Latency**: End-to-end delay (RTT)  
**Throughput**: Bits per second  
**Packet Loss**: Congestion, buffer overflow  
**Jitter**: Latency variation  
**Bisection Bandwidth**: Worst-case cross-section capacity 