# Notes
### Loops
- set metadata with a loop index and then decrement it in the parser
- repeat ttl= ttl -1; to decrement the ttl 5 times

### P4 simple_switch_CLI
- `table_dump <table_name>`: Dumps the contents of the specified table.
- `table_add <table_name> <action_name> <match_key> <action_params>`: Adds an entry to the specified table with the given action and parameters (in hex format, so 0x appended)
```
table_add MyIngress.ipv4_lpm MyIngress.ipv4_forward 0x0a000303/32 => 0x080000000300 7
```
- `table_delete <table_name> <match_key>`: Deletes an entry from the specified table that matches the given key.
- `table_clear <table_name>`: Clears all entries from the specified table.

### Debugging
You create a new table to debug the pipeline. You can use the `debug` action to print out the values of specific fields or metadata at various points in the pipeline.


### Congestion Control (TCP - New Reno Method)
https://www.geeksforgeeks.org/tcp-congestion-control/

##### Slow Start
- Start with 1 packet and send double the number of packets each round trip time (RTT) until a packet is lost.
- The sender starts with a congestion window (cwnd) of 1 packet and increases it exponentially until it reaches a threshold (ssthresh).

##### Congestion Avoidance
- After reaching the threshold, the sender increases the congestion window linearly (by 1 packet per RTT) to avoid overwhelming the network.

##### Cases of congestion:
1. Timeout: When a sender does not receive an acknowledgment (ACK) for a sent packet within a certain time frame, it assumes that the packet was lost due to congestion and reduces its sending rate by 50%.
2. Duplicate ACKs: When a sender receives multiple duplicate ACKs for the same packet, it indicates that the packet was likely lost, and the sender reduces its sending rate by 50% as well.

##### When congestion occurs:
- Timeout: The sender reduces its congestion window (cwnd) to 1 packet and enters the slow start phase again, but it sets the slow start threshold (ssthresh) to half of the max cwnd before the timeout.
- Duplicate ACKs: The sender reduces its congestion window (cwnd) to half of the max cwnd before the duplicate ACKs and enters the fast recovery phase, where it continues to send packets until it receives a new ACK.
- Fast Recovery: After a timeout or receiving duplicate ACKs, the sender enters the fast recovery phase, where it continues to send packets until it receives a new ACK for the lost packet. Once the new ACK is received, the sender exits fast recovery and resumes normal operation.


##### Explicit Congestion Notification (ECN)
- TOS (8bits): Type of Service in the IPv4 header is used to indicate the priority of the packet and can be used for congestion control.
- ECN (2 bits) marks packets that are experiencing congestion without dropping them. The sender and receiver can then adjust their sending rates based on the ECN marks.
- Check queue length and if it exceeds a certain threshold, mark the packets with ECN instead of dropping them. This allows the sender to adjust its sending rate without losing packets.
- Congestion information is sent back to the sender using the ACK packets, which can then adjust its sending rate based on the congestion information, since the congestion bit is available only in the IP header sent to the receiver.
- The sender can then adjust its sending rate based on the congestion information received in the ACK packets.

###### ECN Marking meanings (CE - Congestion Experienced bits)

CWR (Congestion Window Reduced) - Indicates that the sender has reduced its sending rate in response to congestion.
ECE (ECN Echo) - Indicates that the sender has received a packet with the ECN bit set, indicating congestion.

CWR | ECE | Meaning
---|-----|----------------
0  | 0   | No congestion experienced
0  | 1   | Congestion experienced, sender should reduce its sending rate
1  | 0   | Congestion window reduced, sender has reduced its sending rate
1  | 1   | Queue size larger than a congestion threshold, sender should reduce its sending rate

Based on RFC 3168, the ECN field is used to indicate whether a packet is ECN-capable and whether congestion has been experienced. The sender and receiver can then use this information to adjust their sending rates based on the congestion information received in the ACK packets.

##### Congetion Control in P4


### State management in P4
- By using the `register` primitive, you can store and retrieve values in a stateful manner.

#### Counter 
- Only stored but can't be read inside P4 program
- bytes type tracks the number of bytes processed by the switch (packet size)
- packets type tracks the number of packets processed by the switch
- packets_and_bytes type tracks both the number of packets and the number of bytes processed by the switch

- Direct counter example:
```p4
control MyIngress {
    direct_counter<packets_and_bytes> my_counter;

    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
        }
        counters
        size = 1024;
    }
}
```

### Proect idea for CONGA
- Congestion-aware load balancing in a leaf-spine topology.
- The controller monitors the load on each spine and redistributes the traffic to balance the load across all spines.

### Project idea for packet based load balancing
P4 program that has a table for each destinatin IP address and a counter for each table for each spine counting packets and bytes. A controller that can add and remove entries from the table and reset the counters. The controller can also query the counters to get the number of packets and bytes processed by each table. 

Periodically updating the tables and routes based on the counters. The controller can also monitor the load on each spine and redistribute the traffic to balance the load across all spines.

Issues with this approach:
- The controller may not be able to keep up with the rate of incoming packets, leading to delays in updating the tables and routes.
- if we reconfigure in the middle of a packet flow,the new flow might be faster than old one and the old one might be dropped since the new one is faster.

Maybe programmable in mininet to automatically update the topology and the tables in the P4 program. The controller can also monitor the load on each spine and redistribute the traffic to balance the load across all spines.

### Load Balancing
Method | Positive | Negative | Example
---|---------|----------|--------
Packet based | Fine-grained, simple | Reordering, TCP Problem | DRILL
Flow based | Simple, no reordering | Course-load distribution, congestion is not calculated | ECMP
Flowlet based | good load distribution, no reordering | Complex, requires state | HULA, CONGA

#### ECMP (load_balance.p4)
- Equal Cost Multi-Path (ECMP) is a routing strategy that allows multiple paths to be used for forwarding packets to the same destination. It helps distribute traffic evenly across available paths, improving network utilization and reducing congestion.

Process:
1) Hash (IP source, destination, source port, destination port, protocol id) -> 16bit hash 0-65535
2) Port id = hash % number of ports (result is between 0 and number of ports - 1)
3) Send the packet to the port id

Problems:
- Hash collisions: Different packets may hash to the same port, leading to uneven distribution.

#### Conga
- Flowlet
- Cngestion-aware
- Leaf spine topology


VLAN: Virtual Local Area Network
- A VLAN is a logical grouping of devices within a network that allows them to communicate as if they were on the same physical network, even if they are not. VLANs are used to segment networks for improved performance, security, and management. So that you can have multiple VLANs on the same physical network infrastructure.

VXLAN: Virtual Extensible Local Area Network
- A VXLAN is a network virtualization technology that encapsulates Layer 2 Ethernet frames within Layer 4 UDP packets, allowing for the creation of virtual networks over existing Layer 3 infrastructure. VXLANs are used to extend VLANs across large data centers and cloud environments.

#### HULA (hula.p4)

- Hula probes the network to find the best path for each flowlet. It uses a combination of active and passive probing to gather information about the network's state and adjust the forwarding paths accordingly.


### Exam questions

#### **1. Why is a table applied only once in the control flow?**
- A table is applied only once in the control flow to ensure that the packet processing pipeline is efficient and avoids unnecessary re-evaluation of the same table for each packet. This design choice helps maintain a clear and predictable flow of packet processing, allowing for better performance and easier debugging.
- If we want the Traffic Manager can recirculate the packet, we can use the `recirculate` action to send the packet back to the beginning of the pipeline for further processing.
- Or you can clone the packet and send to multiple egress ports.
- You can also `truncate` the packet to send a smaller version of the packet clone to other tables.
- DPI: Deep Packet Inspection -> you can use the `clone` action to send a copy of the packet to a different table for inspection without affecting the original packet's processing.

#### **2. What is tunneling?**
- Tunneling is a technique used to encapsulate packets within another packet format, allowing them to be transmitted over a network that may not support the original packet format. This is often used to create virtual private networks (VPNs) or to transport packets across different network protocols.
- Tunneling is like putting a letter (the original packet) inside an envelope (the new packet format) to send it through the mail (the network). The envelope protects the letter and allows it to be sent through different postal systems (networks) without being opened or altered.

# Exercises
## Exercise 1

#### **Ingress Control Block**
1. **`ipv4_forward` Action**:
   - **Purpose**: This action is invoked by the `ipv4_lpm` table to forward packets.
   - **Steps**:
     - **Set the Egress Port**: The `standard_metadata.egress_spec` field is set to the port specified by the action parameter `port`. This determines which port the packet will be sent out of.
     - **Update MAC Addresses**: The source MAC address is set to the current destination MAC address (the switch's MAC), and the destination MAC address is updated to the next hop's MAC address (`dstAddr`).
     - **Decrement TTL**: The `hdr.ipv4.ttl` field is decremented by 1 to reflect that the packet has traversed one more hop in the network.

2. **`ipv4_lpm` Table**:
   - **Purpose**: This table performs a longest prefix match (LPM) on the destination IP address (`hdr.ipv4.dstAddr`) to determine the next hop.
   - **Actions**:
     - `ipv4_forward`: Forwards the packet to the next hop.
     - `drop`: Drops the packet.
     - `NoAction`: Does nothing (used as a placeholder).
   - **Default Action**: If no match is found in the table, the packet is dropped.

3. **Apply Block**:
   - Checks if the IPv4 header is valid (`hdr.ipv4.isValid()`).
   - If valid, the `ipv4_lpm` table is applied to determine the appropriate action.

---

#### **Deparser**
- The deparser is responsible for reconstructing the packet before it is sent out of the switch.
- **Headers Emitted**:
  - The Ethernet header (`hdr.ethernet`) is emitted first.
  - The IPv4 header (`hdr.ipv4`) is emitted next.
- The order of emission is crucial to ensure the packet is correctly formatted for the next hop.

---

### Summary
- The `ipv4_forward` action updates the packet's MAC addresses, decrements the TTL, and sets the egress port.
- The `ipv4_lpm` table determines the next hop based on the destination IP address.
- The deparser ensures the packet is correctly reconstructed before being sent out.

This implementation enables basic IPv4 forwarding functionality in the switch.


## Exercise 2

### **1. Set the Egress Port**
- **Requirement**: The egress port of the packet must be set to the port number provided by the control plane.
- **Implementation**:
  ```p4
  standard_metadata.egress_spec = port;
  ```
  - The `port` parameter is passed to the `ipv4_forward` action by the control plane.
  - The `standard_metadata.egress_spec` field is used to specify the output port for the packet.

---

### **2. Update the Source MAC Address**
- **Requirement**: The source MAC address of the packet must be updated to the MAC address of the switch (the current destination MAC address).
- **Implementation**:
  ```p4
  hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
  ```
  - The current destination MAC address (`hdr.ethernet.dstAddr`) is the MAC address of the switch.
  - This value is copied to the source MAC address (`hdr.ethernet.srcAddr`).

---

### **3. Update the Destination MAC Address**
- **Requirement**: The destination MAC address of the packet must be updated to the MAC address of the next hop, which is provided by the control plane.
- **Implementation**:
  ```p4
  hdr.ethernet.dstAddr = dstAddr;
  ```
  - The `dstAddr` parameter is passed to the `ipv4_forward` action by the control plane.
  - This value represents the MAC address of the next hop and is assigned to the destination MAC address (`hdr.ethernet.dstAddr`).

---

### **4. Decrement the TTL Field**
- **Requirement**: The TTL (Time-to-Live) field in the IPv4 header must be decremented by 1.
- **Implementation**:
  ```p4
  hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
  ```
  - The current TTL value (`hdr.ipv4.ttl`) is decremented by 1.
  - This ensures that the packet's TTL decreases as it traverses each hop in the network.

---

### **Final `ipv4_forward` Action**
Here is the complete implementation of the `ipv4_forward` action:
```p4
action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
    // Set the egress port
    standard_metadata.egress_spec = port;

    // Update the source MAC address to the switch's MAC address
    hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;

    // Update the destination MAC address to the next hop's MAC address
    hdr.ethernet.dstAddr = dstAddr;

    // Decrement the TTL field
    hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
}
```

---

### **How It Works**
1. **Control Plane**:
   - The control plane (e.g., a P4 runtime or a configuration file) populates the `ipv4_lpm` table with entries that specify:
     - The destination IP prefix (`hdr.ipv4.dstAddr`).
     - The next hop's MAC address (`dstAddr`).
     - The egress port (`port`).

2. **Ingress Pipeline**:
   - When a packet arrives, the `ipv4_lpm` table is applied.
   - If a match is found, the `ipv4_forward` action is executed with the parameters (`dstAddr` and `port`) provided by the control plane.

3. **Packet Processing**:
   - The `ipv4_forward` action updates the packet's headers and sets the egress port.
   - The packet is then forwarded to the specified port with the updated MAC addresses and decremented TTL.

---

### **Testing**
To verify that the `ipv4_forward` action works as expected:
1. **Add Table Entries**:
   Use the control plane to populate the `ipv4_lpm` table with appropriate entries. For example:
   ```bash
   table_add ipv4_lpm ipv4_forward 10.0.1.1/24 08:00:00:00:01:11 1
   table_add ipv4_lpm ipv4_forward 10.0.2.2/24 08:00:00:00:02:22 2
   ```

2. **Send Packets**:
   Use Mininet or a packet generator to send packets with specific destination IPs.

3. **Capture and Inspect Packets**:
   Use `tcpdump` or Wireshark to verify:
   - The source and destination MAC addresses are updated correctly.
   - The TTL is decremented by 1.
   - The packet is forwarded to the correct egress port.


## Exercise 3

### **1. Understand the Testing Infrastructure**
- **Static Control Plane**:
  - The control plane is pre-configured to add entries to the `ipv4_lpm` table on each switch.
  - For example, `s1-commands.txt` contains the table entries for switch `s1`. These entries map destination IP prefixes to the next hop's MAC address and egress port.

- **`run.sh` Script**:
  - This script automates the following steps:
    1. Compiles your P4 program (`basic.p4`) into a JSON file.
    2. Sets up the triangle topology in Mininet.
    3. Loads the table entries (e.g., from `s1-commands.txt`) into the switches.

- **Traffic Testing**:
  - The `pingall` command in Mininet sends ICMP packets between all hosts in the topology to test connectivity.

---

### **2. Steps to Test Your Program**

#### **Step 1: Run the `run.sh` Script**
Execute the script to compile the P4 program, set up the topology, and configure the switches:
```bash
./run.sh
```
- Ensure the script has executable permissions. If not, make it executable:
  ```bash
  chmod +x run.sh
  ```

#### **Step 2: Verify the Topology**
Once the script completes, you should be in the Mininet CLI. Use the `net` command to verify the topology:
```bash
mininet> net
```
- This will display the connections between hosts and switches.

#### **Step 3: Test Connectivity**
Run the `pingall` command to test connectivity between all hosts:
```bash
mininet> pingall
```
- If your `ipv4_forward` action and `ipv4_lpm` table are working correctly:
  - All hosts should be able to ping each other successfully.
  - You should see `0% packet loss`.

---

### **3. Debugging and Verification**

#### **If `pingall` Fails**:
1. **Check Table Entries**:
   - Ensure the `ipv4_lpm` table on each switch has the correct entries.
   - Use the Mininet CLI to access a switch's runtime CLI and inspect the table:
     ```bash
     simple_switch_CLI --thrift-port <port>
     ```
     Replace `<port>` with the Thrift port of the switch (e.g., `9090` for `s1`).

     Run the following command in the switch CLI to display the table:
     ```bash
     table_dump ipv4_lpm
     ```

2. **Verify Header Modifications**:
   - Use `tcpdump` or Wireshark to capture packets on the switch interfaces.
   - Check if:
     - The source and destination MAC addresses are updated correctly.
     - The TTL is decremented by 1.

3. **Check Logs**:
   - If you're using BMv2, enable logging to see how packets are processed:
     ```bash
     sudo simple_switch --log-console basic.json
     ```

---

### **4. Expected Behavior**
- **Successful `pingall`**:
  - All hosts should be able to communicate with each other.
  - For example:
    ```
    *** Ping: testing ping reachability
    h1 -> h2 h3
    h2 -> h1 h3
    h3 -> h1 h2
    *** Results: 0% dropped (6/6 received)
    ```

- **Packet Inspection**:
  - Captured packets should show:
    - Correct source and destination MAC addresses.
    - Decremented TTL in the IPv4 header.

---

### **5. Summary**
- The `run.sh` script automates the setup and testing process.
- Use `pingall` to verify connectivity.
- Debug using table dumps, packet captures, and logs if issues arise.

Let me know if you encounter any specific issues during testing!- **Packet Inspection**:
  - Captured packets should show:
    - Correct source and destination MAC addresses.
    - Decremented TTL in the IPv4 header.

---

### **5. Summary**
- The `run.sh` script automates the setup and testing process.
- Use `pingall` to verify connectivity.
- Debug using table dumps, packet captures, and logs if issues arise.
