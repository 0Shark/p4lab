# P4Intro

These are study notes summarizing the "Introduction to Data Plane Programming with P4" presentation.

## 1. The "Why" of P4: Top-Down vs. Bottom-Up Design

### Traditional Approach: Bottom-Up (Fixed-Function)
- **The Hardware Dictates:** The network device (e.g., a fixed-function ASIC) defines how packets can be processed. Its capabilities are fixed.
- **Inflexible:** Adding new protocols or features requires new hardware or firmware updates from the vendor.
- **Example:** The ASIC datasheet defines the rules. OpenFlow was a step forward but still required specification updates for new header matches.



### The P4 Approach: Top-Down (Programmable)
- **The Programmer Dictates:** You, the programmer or network operator, define *how you want* packets to be processed.
- **Flexible:** The program is compiled and loaded onto a programmable network device (ASIC, FPGA, CPU).
- **Benefits:**
  - **New Features:** Add new protocols without changing hardware.
  - **Reduced Complexity:** Remove unused protocols from the pipeline.
  - **Efficiency:** Flexible and efficient use of hardware resources (e.g., tables).
  - **Visibility:** Implement custom telemetry and diagnostics (e.g., In-band Network Telemetry).
  - **Agility:** Use software development cycles for network features, including fixing data plane bugs in the field.



---

## 2. The P4 Abstract Model: PISA

P4 programs target an abstract architecture called **PISA (Protocol-Independent Switch Architecture)**.

PISA has three main stages:

1.  **Programmable Parser:**
    - Examines the incoming packet byte-by-byte.
    - Identifies and extracts headers into a structured representation.
    - Implemented as a state machine.

2.  **Programmable Match-Action Pipeline:**
    - The core processing logic.
    - Takes extracted headers and metadata as input.
    - Performs lookups in tables (`Match`).
    - Executes code based on the lookup result (`Action`).
    - Can modify headers, add/remove headers, and update metadata.

3.  **Programmable Deparser:**
    - Takes the final set of headers.
    - Re-assembles them into a packet to be sent on the wire.



---

## 3. P4_16 Language Fundamentals

### Key Concepts

- **P4 Architecture:** A vendor-supplied file that defines the programmable blocks of a target (e.g., the number of parsers/pipelines, available externs). Your P4 program is written *against* an architecture. A common learning architecture is `v1model.p4`.
- **P4 Target:** A specific hardware or software implementation that can run a P4 program (e.g., Barefoot Tofino ASIC, `bmv2` software switch).
- **Control Plane:** The "brain" that runs on a CPU. It manages the data plane by inserting/deleting entries in the tables defined by the P4 program.
- **Data Plane:** The "workhorse" that processes packets at line rate according to the P4 program's logic and the tables populated by the control plane.

### Main Language Elements

| Element         | Purpose                                                                                                 | Maps to PISA Stage     |
| --------------- | ------------------------------------------------------------------------------------------------------- | ---------------------- |
| **`header`**    | Defines the structure of a protocol header (e.g., Ethernet, IPv4). An ordered collection of fields.       | All                    |
| **`struct`**    | Defines a collection of data, often for metadata. An unordered collection.                               | All                    |
| **`parser`**    | Extracts headers from a packet. Written as a state machine with `state` and `transition` statements.      | Parser                 |
| **`control`**   | Defines a match-action pipeline stage. Contains `table` and `action` definitions.                       | Match-Action & Deparser |
| **`action`**    | A block of imperative code that modifies headers/metadata. Can have parameters.                         | Match-Action           |
| **`table`**     | The core of a `control` block. Defines a `key` to match, a list of `actions`, and other properties.     | Match-Action           |

### V1Model Architecture

A standard architecture for learning P4. It refines the PISA model into more specific blocks:

`Parser` -> `Ingress Pipeline` -> `Traffic Manager` -> `Egress Pipeline` -> `Deparser`

- It provides a `standard_metadata_t` struct that carries important packet information like `ingress_port` and `egress_spec` through the pipeline.

### Example: Basic L2 Switch Logic in P4

This `control` block implements a simple L2 learning switch.

```p4
control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t std_meta) {

    // Action to set the egress port
    action set_egress(bit<9> port) {
        std_meta.egress_spec = port;
    }

    // Action to do nothing and drop the packet
    action drop() {
        // mark_to_drop() is a v1model primitive
        mark_to_drop();
    }

    // Table that matches on destination MAC address
    table mac_forward {
        key = {
            hdr.ethernet.dstAddr: exact; // Match key: exact match on destination MAC
        }
        actions = {
            set_egress; // Possible actions
            drop;
        }
        size = 1024;
        default_action = drop(); // If no match, drop
    }

    // The logic block for this control
    apply {
        mac_forward.apply(); // Apply the table lookup
    }
}
```

- The **Control Plane** is responsible for adding entries to the `mac_forward` table, mapping MAC addresses to output ports.
- The **Data Plane** (this P4 code) executes the lookup for every packet and runs the corresponding action.

### Parsers & Select Statement

Parsers use a `state` machine. You can branch based on an extracted value using `transition select`.

```p4
parser MyParser(...) {
    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        // Branch based on the EtherType field
        transition select(hdr.ethernet.etherType) {
            0x0800: parse_ipv4; // If IPv4, go to the next state
            default: accept;    // Otherwise, we are done parsing
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept; // Done
    }
}
```

### Deparsing

The deparser is a `control` block that uses an `emit` call to write headers back to the packet buffer in the correct order.

```p4
control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4); // This will only emit if hdr.ipv4 is valid
    }
}
```