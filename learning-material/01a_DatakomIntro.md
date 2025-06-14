# DatakomIntro
> Based on "Datacenter Network Programming – Sommersemester 2023"

## 1. Core Network Principles

*   **Packet Switching**: Data is broken into small chunks called **packets**.
    *   Each packet travels independently through the network.
    *   Allows for **statistical multiplexing**, where multiple flows can share a link efficiently.
*   **Best-Effort Delivery**: The network does *not* guarantee that packets will be delivered.
    *   Packets can be **lost**, **delayed**, or arrive **out-of-order**.
    *   This simplifies the core network design. Reliability is a job for higher layers (e.g., TCP).
*   **Internet Hourglass Model**: A layered architecture with a "narrow waist".
    *   Many application protocols on top (HTTP, SMTP, etc.).
    *   Many link layer technologies on the bottom (Ethernet, WiFi, etc.).
    *   **IP (Internet Protocol)** is the universal waist that connects everything.
*   **Scalability**: Achieved through **hierarchy** (e.g., IP addresses with network/host parts, Autonomous Systems) and **indirection** (e.g., DNS mapping names to addresses).

## 2. Layered Architecture (OSI Model)

A model that organizes network functions into 7 distinct layers. Each layer provides services to the layer above it and communicates with its peer layer on another machine using a protocol.

| Layer | Name          | PDU        | Function & Key Concepts                                      |
|-------|---------------|------------|--------------------------------------------------------------|
| 7     | Application   | Message    | User-facing protocols (HTTP, SMTP).                            |
| 6     | Presentation  | -          | Data formatting, encryption. (Often part of Application layer) |
| 5     | Session       | -          | Manages dialogues (sessions). (Often part of Application layer) |
| 4     | **Transport** | Segment    | **End-to-end** communication. Process-to-process delivery. |
| 3     | **Network**   | Packet     | **Host-to-host** delivery across multiple networks. **Routing**.   |
| 2     | **Data Link** | Frame      | **Node-to-node** (hop) delivery on the same link.            |
| 1     | Physical      | Bit        | Transmitting raw bits over a medium.                         |

*   **Encapsulation**: As data moves down the stack, each layer adds its own **header**.

## 3. Data Link Layer (L2)

*   **Function**: Reliable data transfer over a single physical link.
*   **Addressing**: **MAC Address** (e.g., `00-15-C5-49-04-A9`).
    *   A 48-bit, globally unique, hard-coded address on the Network Interface Card (NIC).
    *   Used for communication within a local network (LAN).
*   **Key Protocols**:
    *   **ARP (Address Resolution Protocol)**: Maps a known IP address to its corresponding MAC address.
        *   Broadcasts: "Who has IP `192.168.1.10`?"
        *   Response: "I have that IP, and my MAC is `...`"
    *   **DHCP (Dynamic Host Configuration Protocol)**: Automatically assigns an IP address to a host when it joins a network.
        *   Broadcasts: "I need an IP address!"
        *   Response: "You can have IP `192.168.1.10`."

## 4. Network Layer (L3)

*   **Function**: Routing packets from a source host to a destination host across multiple networks (an internetwork).
*   **Protocol**: **IP (Internet Protocol)** is the dominant L3 protocol.
*   **Addressing**: **IP Address** (e.g., `12.34.158.5`).
    *   A 32-bit (IPv4) or 128-bit (IPv6) logical address.
    *   Hierarchical: `Network part` + `Host part`.
    *   The **Subnet Mask** (or CIDR notation like `/24`) separates the two parts.
*   **IPv4 Header Key Fields**:
    *   `Source/Destination Address`: 32-bit IP addresses.
    *   `TTL (Time-to-Live)`: Hop limit to prevent infinite loops. Decremented by each router.
    *   `Protocol`: ID of the transport layer protocol (6 for TCP, 17 for UDP).
    *   `Fragmentation Fields`: To split packets larger than the link's MTU.

## 5. Routing

Routing is the process of selecting a path for traffic. It occurs at Layer 3.

*   **Control Plane**: The "brain". Decides where to send packets.
    *   Runs routing protocols (e.g., OSPF, BGP) to build a **routing table**.
*   **Data Plane (Forwarding Plane)**: The "muscle". Forwards packets at high speed.
    *   Uses a **forwarding table** (derived from the routing table) to look up the next hop for each incoming packet.

### Routing Protocols

*   **Intra-domain (within an AS)**: Also called Interior Gateway Protocols (IGP).
    *   **OSPF (Open Shortest Path First)**: A **link-state** protocol.
        1.  Routers flood information about their direct links (link state) to all other routers.
        2.  Each router builds a complete map (graph) of the network.
        3.  Each router independently runs **Dijkstra's algorithm** to find the shortest path to all destinations.
*   **Inter-domain (between ASes)**: Also called Exterior Gateway Protocols (EGP).
    *   **BGP (Border Gateway Protocol)**: A **path-vector** protocol.
        *   Routers advertise not just a cost, but the full path of ASes to reach a destination.
        *   This prevents loops and allows for complex **policy-based routing** (e.g., avoiding a competitor's network).

*   **Dijkstra's Algorithm**: A greedy algorithm to find the single-source shortest paths in a graph with non-negative edge weights. Essential for link-state protocols.

## 6. Transport Layer (L4)

*   **Function**: Provides logical communication between application processes on different hosts.
*   **Key Tasks**:
    *   **Demultiplexing**: Delivers data to the correct application using **port numbers**.
    *   **Error Detection**: Uses a **checksum** to detect data corruption.
*   **Protocols**:
    *   **UDP (User Datagram Protocol)**:
        *   Connectionless, unreliable, "fire and forget".
        *   Provides just demultiplexing and basic checksums.
        *   Low overhead, fast. Good for DNS, VoIP, online gaming.
    *   **TCP (Transmission Control Protocol)**:
        *   **Connection-oriented**: Establishes a connection via a 3-way handshake.
        *   **Reliable, in-order delivery**: Uses sequence numbers, acknowledgements (ACKs), and retransmissions to ensure data arrives correctly.
        *   **Flow Control**: Prevents the sender from overwhelming the receiver's buffer.
        *   **Congestion Control**: Adapts sending rate to avoid overwhelming the network.

## 7. Application Layer (L7) & DNS

*   **Function**: Defines protocols that network applications use to communicate, like HTTP for the web.
*   **Relationship to other layers**:
    1.  An application wants to connect to a human-readable **name** (e.g., `www.example.com`).
    2.  **DNS (Domain Name System)** is used to resolve this `name` into a machine-readable IP **address**. This is the **Discovery** step.
    3.  TCP/UDP use this IP address and a port number to create a connection.
    4.  IP uses the destination address to **route** the packet across the internet, finding a **path**.