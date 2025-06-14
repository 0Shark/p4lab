# P4 PROGRAMMING CHEAT SHEET

## 1. P4 PHILOSOPHY & ARCHITECTURE
**Top-Down Design**: Programmer defines packet processing → compiler → target-specific config  
**PISA Model**: Parser → Match-Action Pipeline → Deparser  
**V1Model**: Parser → Ingress → Traffic Manager → Egress → Deparser

## 2. CORE LANGUAGE ELEMENTS

### DATA TYPES & STRUCTURES
```p4
// Basic types
bit<W>        // W-bit unsigned
int<W>        // W-bit signed  
bool          // Boolean

// Headers (ordered fields)
header ethernet_t {
    bit<48> dstAddr;
    bit<48> srcAddr; 
    bit<16> etherType;
}

// Structs (unordered collections)
struct metadata_t {
    bit<32> temp_val;
    bool    processed;
}

// Header stacks (arrays)
header mpls_t {
    bit<20> label;
    bit<3>  tc;
    bit<1>  s;
    bit<8>  ttl;
}
typedef mpls_t[8] mpls_stack_t;  // Stack of 8 MPLS headers
```

### PARSER (State Machine)
```p4
parser MyParser(packet_in packet, out headers hdr, inout metadata meta, 
                inout standard_metadata_t std_meta) {
    state start {
        transition parse_ethernet;
    }
    
    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            0x0800: parse_ipv4;    // IPv4
            0x8847: parse_mpls;    // MPLS
            default: accept;
        }
    }
    
    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}
```

### CONTROL BLOCKS (Match-Action Pipeline)
```p4
control MyIngress(inout headers hdr, inout metadata meta, 
                  inout standard_metadata_t std_meta) {
    
    // Actions
    action drop() {
        mark_to_drop(std_meta);
    }
    
    action forward(bit<9> port) {
        std_meta.egress_spec = port;
    }
    
    action set_ecmp_select(bit<16> ecmp_group_id, bit<16> num_nhops) {
        hash(meta.ecmp_select, HashAlgorithm.crc16,
             (bit<1>)0,
             { hdr.ipv4.srcAddr, hdr.ipv4.dstAddr, 
               hdr.ipv4.protocol, meta.l4_srcPort, meta.l4_dstPort },
             num_nhops);
    }
    
    // Tables
    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;  // Longest prefix match
        }
        actions = {
            forward;
            set_ecmp_select;
            drop;
        }
        size = 1024;
        default_action = drop();
    }
    
    table ecmp_group {
        key = {
            meta.ecmp_group_id: exact;
            meta.ecmp_select: exact;
        }
        actions = {
            forward;
        }
        size = 1024;
    }
    
    // Apply block
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
            if (meta.ecmp_group_id != 0) {
                ecmp_group.apply();
            }
        }
    }
}
```

### DEPARSER
```p4
control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.mpls);      // Only emits if valid
        packet.emit(hdr.ipv4);
    }
}
```

## 3. STATEFUL CONSTRUCTS

### REGISTERS (General-purpose arrays)
```p4
register<bit<32>>(1024) my_register;

action read_register() {
    my_register.read(meta.reg_val, meta.index);
    my_register.write(meta.index, meta.new_val);
}
```

### COUNTERS
```p4
// Direct counter (attached to table)
counter(1024, CounterType.packets_and_bytes) direct_counter;
table my_table {
    // ... table definition
    counters = direct_counter;
}

// Indirect counter
counter(1024, CounterType.packets) packet_counter;
action count_packet() {
    packet_counter.count(meta.counter_index);
}
```

### METERS (Rate limiting)
```p4
meter(1024, MeterType.packets) my_meter;
action meter_action() {
    my_meter.execute_meter(meta.meter_index, meta.meter_result);
    if (meta.meter_result == MeterColor_t.RED) {
        drop();
    }
}
```

## 4. MATCH TYPES & KEY FUNCTIONS
```p4
table my_table {
    key = {
        hdr.ipv4.dstAddr: lpm;        // Longest prefix match
        hdr.ipv4.srcAddr: exact;      // Exact match
        hdr.ipv4.protocol: ternary;   // Ternary (with mask)
        hdr.tcp.dstPort: range;       // Range match
    }
}

// Hash function
hash(output, HashAlgorithm.crc32, base, {fields}, max);

// Header validity
if (hdr.ipv4.isValid()) { ... }
hdr.ipv4.setValid();     // Mark header as valid
hdr.ipv4.setInvalid();   // Mark header as invalid
```

## 5. COMMON P4 PATTERNS

### L2 SWITCH
```p4
action set_egress(bit<9> port) {
    std_meta.egress_spec = port;
}

table mac_forward {
    key = { hdr.ethernet.dstAddr: exact; }
    actions = { set_egress; drop; }
    default_action = drop();
}
```

### ECMP LOAD BALANCING
```p4
action ecmp_hash() {
    hash(meta.ecmp_select, HashAlgorithm.crc16, (bit<1>)0,
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr, 
          hdr.tcp.srcPort, hdr.tcp.dstPort, hdr.ipv4.protocol},
         (bit<16>)ECMP_SIZE);
}
```

### TUNNELING (VXLAN)
```p4
// Encapsulation
action vxlan_encap(bit<48> dmac, bit<32> dip, bit<24> vni) {
    hdr.inner_ethernet = hdr.ethernet;
    hdr.inner_ipv4 = hdr.ipv4;
    
    hdr.ethernet.dstAddr = dmac;
    hdr.ipv4.dstAddr = dip;
    hdr.vxlan.vni = vni;
    hdr.vxlan.setValid();
}

// Decapsulation  
action vxlan_decap() {
    hdr.ethernet = hdr.inner_ethernet;
    hdr.ipv4 = hdr.inner_ipv4;
    hdr.vxlan.setInvalid();
}
```

### INT TELEMETRY
```p4
action add_int_hop_latency() {
    hdr.int_metadata.push_front(1);
    hdr.int_metadata[0].data = 
        (bit<32>)(meta.eg_timestamp - meta.ig_timestamp);
}
```

### BLOOM FILTER (Membership Testing)
```p4
#define BLOOM_FILTER_SIZE 1024
#define BLOOM_HASH_COUNT 3

register<bit<1>>(BLOOM_FILTER_SIZE) bloom_filter;

action bloom_insert() {
    bit<32> hash1;
    bit<32> hash2; 
    bit<32> hash3;
    
    hash(hash1, HashAlgorithm.crc16, (bit<1>)0, 
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr}, (bit<32>)BLOOM_FILTER_SIZE);
    hash(hash2, HashAlgorithm.crc32, (bit<1>)0, 
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr}, (bit<32>)BLOOM_FILTER_SIZE);
    hash(hash3, HashAlgorithm.csum16, (bit<1>)0, 
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr}, (bit<32>)BLOOM_FILTER_SIZE);
    
    bloom_filter.write(hash1, 1);
    bloom_filter.write(hash2, 1);
    bloom_filter.write(hash3, 1);
}

action bloom_query() {
    bit<1> bit1, bit2, bit3;
    // Calculate same hashes and read bits
    bloom_filter.read(bit1, hash1);
    bloom_filter.read(bit2, hash2);
    bloom_filter.read(bit3, hash3);
    
    if (bit1 == 1 && bit2 == 1 && bit3 == 1) {
        meta.bloom_hit = 1; // Probably in set
    } else {
        meta.bloom_hit = 0; // Definitely not in set
    }
}
```

### COUNT-MIN SKETCH (Frequency Estimation)
```p4
#define SKETCH_WIDTH 1024
#define SKETCH_DEPTH 4

register<bit<32>>(SKETCH_WIDTH) sketch_row1;
register<bit<32>>(SKETCH_WIDTH) sketch_row2;
register<bit<32>>(SKETCH_WIDTH) sketch_row3;
register<bit<32>>(SKETCH_WIDTH) sketch_row4;

action sketch_update() {
    bit<32> hash1, hash2, hash3, hash4;
    bit<32> count1, count2, count3, count4;
    
    // Calculate 4 different hashes
    hash(hash1, HashAlgorithm.crc16, (bit<1>)0, 
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr}, (bit<32>)SKETCH_WIDTH);
    hash(hash2, HashAlgorithm.crc32, (bit<1>)0, 
         {hdr.ipv4.srcAddr, hdr.ipv4.dstAddr}, (bit<32>)SKETCH_WIDTH);
    // ... similar for hash3, hash4
    
    // Read current counts
    sketch_row1.read(count1, hash1);
    sketch_row2.read(count2, hash2);
    sketch_row3.read(count3, hash3);
    sketch_row4.read(count4, hash4);
    
    // Increment and write back
    sketch_row1.write(hash1, count1 + 1);
    sketch_row2.write(hash2, count2 + 1);
    sketch_row3.write(hash3, count3 + 1);
    sketch_row4.write(hash4, count4 + 1);
}

action sketch_query() {
    // Read all 4 counts and take minimum
    bit<32> min_count = min(count1, min(count2, min(count3, count4)));
    meta.estimated_freq = min_count;
}
```

### HULA LOAD BALANCING
```p4
header hula_t {
    bit<8>  dir;        // 0=forward, 1=reverse
    bit<32> qdepth;     // Max queue depth seen
    bit<16> digest;     // Path identifier  
}

register<bit<9>>(256) dstindex_nhop_reg;  // Best next-hop per destination
register<bit<32>>(1024) flow_port_reg;    // Flowlet table

action hula_forward_probe() {
    // Update max queue depth if local is higher
    if (std_meta.enq_qdepth > hdr.hula.qdepth) {
        hdr.hula.qdepth = (bit<32>)std_meta.enq_qdepth;
    }
}

action hula_reverse_probe() {
    bit<9> current_best_port;
    bit<32> stored_qdepth;
    
    // Read current best path for this destination
    dstindex_nhop_reg.read(current_best_port, (bit<32>)meta.dst_index);
    
    // If this probe shows better path, update
    if (hdr.hula.qdepth < stored_qdepth) {
        dstindex_nhop_reg.write((bit<32>)meta.dst_index, 
                               std_meta.ingress_port);
    }
}

action hula_route_data() {
    bit<9> best_port;
    dstindex_nhop_reg.read(best_port, (bit<32>)meta.dst_index);
    std_meta.egress_spec = best_port;
    
    // Cache flowlet decision
    flow_port_reg.write(meta.flow_hash, (bit<32>)best_port);
}
```

### NETCACHE IMPLEMENTATION
```p4
register<bit<32>>(1024) cache_keys;     // Cached keys
register<bit<128>>(1024) cache_values;  // Cached values (simplified)
register<bit<1>>(1024) cache_valid;     // Valid bits

action cache_lookup() {
    bit<32> stored_key;
    bit<128> stored_value;
    bit<1> valid;
    
    cache_keys.read(stored_key, meta.cache_index);
    cache_valid.read(valid, meta.cache_index);
    
    if (valid == 1 && stored_key == hdr.cache_req.key) {
        // Cache hit
        cache_values.read(stored_value, meta.cache_index);
        meta.cache_hit = 1;
        meta.cache_result = stored_value;
    } else {
        // Cache miss
        meta.cache_hit = 0;
    }
}

action cache_insert(bit<32> key, bit<128> value) {
    cache_keys.write(meta.cache_index, key);
    cache_values.write(meta.cache_index, value);
    cache_valid.write(meta.cache_index, 1);
}
```

## 6. COMPILATION & WORKFLOW

### BMV2 (simple_switch)
```bash
# Compile
p4c-bm2-ss -o prog.json prog.p4

# Run target
simple_switch -i 0@veth0 -i 1@veth2 prog.json

# Control via CLI
simple_switch_CLI
> table_add ipv4_lpm forward 10.0.1.0/24 => 1
```

### P4Runtime (gRPC)
```bash
# Compile with P4Info
p4c-bm2-ss --p4v 16 -o prog.json --p4runtime-file prog.p4info prog.p4

# Run with gRPC
simple_switch_grpc --device-id 1 prog.json

# Control via Python gRPC client
```

## 7. STANDARD METADATA (V1MODEL)
```p4
standard_metadata.ingress_port     // Input port
standard_metadata.egress_spec      // Output port
standard_metadata.egress_port      // Actual egress port
standard_metadata.packet_length    // Packet size
standard_metadata.enq_timestamp    // Enqueue time
standard_metadata.deq_timedelta    // Queue residence time
standard_metadata.enq_qdepth       // Queue depth at enqueue
```

## 8. DEBUGGING TIPS
- Use `log_msg()` for debug output
- Check header validity with `isValid()`
- Verify table sizes and default actions
- Test incrementally: parser → tables → actions 