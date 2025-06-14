# P4Advanced

## 1. Core P4 Concepts

### P4 Philosophy: Top-Down Design
- **Goal**: Allow programmers to define *how* a networking device should process packets.
- **Process**:
  1. The programmer writes a **P4 program** describing the desired packet processing logic.
  2. A **P4 compiler** translates this program into a target-specific configuration for a programmable device.
  3. A **control plane** application interacts with the device at runtime to populate tables and manage state, effectively telling the data plane *what* to match on.

### PISA: Protocol Independent Switch Architecture
PISA is the abstract model of a P4-programmable device. It consists of three main stages:

1.  **Programmable Parser**:
    - A state machine that reads the incoming packet byte-stream.
    - Identifies and extracts headers into a structured representation.
2.  **Programmable Match-Action Pipeline**:
    - A series of tables that perform lookups on header fields or metadata.
    - When a packet matches a rule in a table, the corresponding action is executed.
    - Actions can modify headers, add/remove headers, and change metadata.
3.  **Programmable Deparser**:
    - Reassembles the modified headers back into a packet byte-stream to be sent out.

---

## 2. P4 Language Basics

P4 has several key language elements that map to the PISA model.

- **Parsers**: Written as state machines (`start`, `accept`, `reject` are built-in states) that perform bitfield extraction to populate header `structs`.
- **Headers & Data Types**:
  - `header`: A collection of named fields with specified bit-widths (e.g., `bit<48>`).
  - `struct`: A collection of headers and other data.
  - Basic types: `bit<W>`, `int<W>`. No `float` or `string`.
- **Controls**:
  - Similar to C functions (but without loops). They define the logic of the match-action pipeline.
  - The main logic is within an `apply {}` block.
  - Can contain `tables`, `actions`, and other control flow.
- **Tables**:
  - The fundamental unit of the match-action pipeline.
  - A table definition specifies:
    - `key`: One or more fields to match on (e.g., `hdr.ipv4.dstAddr: lpm;`).
    - `actions`: A list of possible actions that can be executed.
  - At runtime, the control plane inserts **entries** (rules) into the table, which map a specific key to a specific action.
- **Actions**:
  - A block of code that modifies headers or metadata.
  - `action drop() {}`
  - `action set_egress_port(port_t port) { ... }`

---

## 3. Stateful P4 Constructs

P4 supports stateful objects that persist across multiple packets. State can **only be modified** from the control plane, but the data plane can read/update it.

- **Header Stacks**: An array of a header type (e.g., `mpls_h[8] mpls;`). Allows for processing a variable number of headers, essential for protocols like MPLS and In-Band Telemetry.
- **Registers**: A general-purpose array for storing stateful data. The data plane can read *and* write to a register at a specific index.
  - `register<bit<32>>(1024) my_register;`
  - `my_register.write(index, value);`
- **Counters**: An array used to count packets and/or bytes.
  - **Indirect Counter**: Declared in a control block and explicitly incremented by an action.
  - **Direct Counter**: Attached directly to a table. Each table entry gets its own counter, which is automatically incremented on a match.
- **Meters**: Used for rate-limiting. A meter "colors" a packet (e.g., GREEN, YELLOW, RED) based on whether it conforms to configured rates (CIR/PIR). An action can then use this color to decide whether to forward or drop the packet.
  - Like counters, can be indirect or direct.

---

## 4. P4 Development & Control Plane Workflow

There are two primary workflows for compiling and running a P4 program.

### Workflow 1: `simple_switch_CLI` (Thrift-based)
1.  **Compile**: `p4c-bm2-ss -o program.json program.p4`
    - Compiles the P4 source into a JSON file that the `simple_switch` software target understands.
2.  **Run Target**: `sudo simple_switch --log-console program.json -i 0@veth0 ...`
    - Starts the software switch and loads the JSON configuration.
3.  **Control**: `simple_switch_CLI`
    - A command-line interface connects to the switch via a Thrift port.
    - Used to manually add table entries: `table_add my_table my_action 10.0.1.1 => 1`

### Workflow 2: P4Runtime (Modern gRPC-based)
1.  **Compile**: `p4c-bm2-ss --p4v 16 -o program.json --p4runtime-file program.p4info ... program.p4`
    - Produces both the target JSON **and** a `p4info` file.
    - The `p4info` file describes the programmable elements (tables, actions, etc.) in a target-independent way.
2.  **Run Target**: `sudo simple_switch_grpc ... program.json`
    - Starts a software switch with a gRPC server running.
3.  **Control**: An external controller application (e.g., a Python script).
    - The controller acts as a gRPC client.
    - It uses the `p4info` file to learn the switch's API and can push rules, read counters, etc., in a programmatic and protocol-independent manner.

---

## 5. Key Use Cases & Examples

- **Tunneling**: Demonstrates adding a new protocol.
  1. Define a new `header` for the tunnel.
  2. Update the `parser` to recognize the tunnel protocol's ethertype.
  3. Add `tables` and `actions` to encapsulate (add header) at an ingress switch and decapsulate (remove header) at an egress switch.
  4. Update the `deparser` to correctly serialize the packet with the new header.
- **Explicit Congestion Notification (ECN)**: Demonstrates monitoring and modifying packets based on device state.
  1. In the egress pipeline, P4 has access to metadata like queue depth (`standard_metadata.enq_qdepth`).
  2. Define an `action` that checks if the queue depth exceeds a threshold.
  3. If it does, modify the ECN bits in the packet's IP header to signal congestion to the endpoints.