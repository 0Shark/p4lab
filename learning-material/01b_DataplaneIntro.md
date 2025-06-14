# DataplaneIntro - The Data Plane and SDN

## 1. What is the Data Plane?

The **Data Plane** is the part of a network device that handles the processing of packet streams. It's responsible for forwarding, filtering, and transforming packets at very high speeds.

### Core Functions of the Data Plane

- **Processing Packet Streams:**
  - Handles large volumes of packets in real-time.
  - Operations must be super fast (e.g., matching bitfields, simple actions).
  - Happens in network devices (routers, switches, firewalls) and on end-hosts (NICs).

- **Key Functionalities:**
  - **Packet Forwarding:** Using a forwarding table (calculated by the Control Plane) to determine the output port for a packet based on its destination address (e.g., MAC for switches, IP prefix for routers).
  - **Access Control (ACLs):** Packet filtering based on rules that match header fields (Src/Dst IP, ports, protocol). Rules are prioritized and can result in `accept` or `drop` actions.
  - **Network Address Translation (NAT):** Maps internal private IP addresses to external public IP addresses, often using port numbers to keep connections unique. Requires rewriting packet headers.
  - **Traffic Monitoring:** Matching header fields to update packet/byte counters for billing, traffic engineering, or anomaly detection.
  - **Buffering & Queue Management:**
    - **FIFO (First In, First Out):** Simple queue; drops packets when full (Drop Tail).
    - **AQM (Active Queue Management):** Proactively drops or marks packets before the queue is full to signal congestion (e.g., RED, CoDel).
  - **Packet Scheduling:** Determines the serving order when multiple queues exist.
    - **Strict Priority:** Always serve the highest priority queue.
    - **Round Robin:** Cycle through queues, serving one packet from each.
    - **Weighted Fair Scheduling (WFS):** Serve queues proportionally based on assigned weights.
  - **Rate Shaping (e.g., Leaky Bucket):** Modifies the packet rate to conform to a profile, smoothing out bursty traffic and avoiding downstream congestion.
  - **Packet Marking:** Marking packets (e.g., using ECN bits) to signal congestion or classify traffic for QoS purposes.

---

## 2. Software Defined Networking (SDN)

SDN is a network architecture that revolutionizes traditional networking by separating the control plane from the data plane.

### Key Concepts of SDN

1.  **Decouple Control and Data Planes:** The core idea. The "brains" (control plane) are moved off the individual network devices.
2.  **Logically Centralized Controller:** A central software "controller" gets a global view of the network and makes intelligent decisions.
3.  **Programmable Network:** The network becomes programmable through a standardized API between the controller and the data plane devices. Network behavior is defined by applications ("apps") running on the controller, not by vendor-specific protocols on each device.

### SDN Architecture & OpenFlow

-   **Southbound Interface:** The API between the controller and the switches. **OpenFlow** is the most well-known standard protocol for this interface.
-   **Northbound Interface:** The API between the controller and the network applications.

**OpenFlow** works on the principle of **Match-Action Tables**:
-   Switches become simple "dumb" forwarding devices with flow tables.
-   The controller installs rules (flows) into these tables.
-   Each rule consists of:
    -   **Match:** A pattern to match against packet headers (e.g., `src=1.2.3.4`, `dport=80`). Can include wildcards.
    -   **Action:** What to do with a matching packet (e.g., `forward(port_x)`, `drop`, `modify_header`, `send_to_controller`).
    -   **Priority:** To resolve conflicts between overlapping rules.
    -   **Counters:** To track stats for that flow (#packets, #bytes).

### SDN Benefits & Use Cases

-   **Centralized Management:** Simplifies network configuration and management.
-   **Rapid Innovation:** New network functions can be deployed as software apps without needing to change hardware.
-   **Example Applications:**
    -   **Simplified Server Load Balancing:** Split traffic to different servers by installing simple match-action rules.
    -   **Seamless Mobility:** When a host moves, the controller can see the new location and simply update the flow rules to reroute traffic seamlessly.
    -   Dynamic Access Control
    -   Energy-Efficient Networking

### SDN Summary & Issues

-   **Main Contributions:**
    1.  **Standardized Protocol (OpenFlow):** A standard way to interact with the data plane.
    2.  **Standardized Model (Match/Action):** A common abstraction for packet processing.
    3.  **Logically Centralized Control:** Simplifies the design of control plane logic.
-   **Issues:**
    -   **Limited Programmability:** The "match-action" model is powerful but not fully flexible; you can only perform actions predefined by the OpenFlow standard.
    -   **Slow Standards Evolution:** Adding new header fields to match on (e.g., for new protocols) requires updating the OpenFlow standard, which is a slow process.
    -   **Interoperability:** Different vendors have variants and extensions, causing issues.