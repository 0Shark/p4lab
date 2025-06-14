# Caching

These notes summarize the main points from the "In-Network Caching (NetCache)" lecture.

### 1. The Problem with Traditional Datacenter Caching

Cloud providers require low latency, high scalability (billions of reads/sec), and efficient handling of popular ("hot") content.

- **Traditional Architecture:** A `Memcache` layer sits between web servers and databases to speed up reads.
- **Scaling Problem:** To handle large amounts of data, the cache is distributed across many `memcached` servers using consistent hashing.
- **The Incast Problem:** A single client request often needs to fetch data from *hundreds* of different cache servers. The simultaneous flood of responses can overwhelm the requesting web server, increasing tail latency. This high "fan-out" is a major bottleneck.

### 2. NetCache: An In-Network Solution

NetCache proposes moving the caching logic directly into the network data plane using P4-programmable switches.

- **Core Idea:** Use the Top-of-Rack (ToR) switch as a "front-line" cache. The switch intercepts key-value requests, serves hits for "hot" items directly, and forwards misses to the backend servers.
- **Benefits:**
    - **Absorbs Hot Traffic:** Prevents requests for the most popular items from ever reaching the server fleet.
    - **Ultra-Low Latency:** Serves cached items at line rate (microsecond latency).
    - **Reduces Fan-Out:** Drastically reduces the number of servers a web server needs to contact.
- **Target Workload:** Ideal for workloads with small objects, more reads than writes, and rapidly changing key popularity (skewed workloads).

### 3. NetCache Architecture

NetCache is composed of two main components:

- **Data Plane (P4 Switch):**
    - **Parser:** Parses custom key-value request packets.
    - **Registers & Tables:** Stores the actual key-value data.
    - **Statistics Engine:** Tracks request frequencies to identify hot items.
    - **Actions:** Serves cached items or forwards requests.
- **Control Plane (CPU):**
    - **Cache Management:** Runs the cache replacement policy (e.g., insert new hot items, evict cold items).
    - **Communication:** Communicates with the data plane via an API to get hot-key reports and to update the switch's cache tables.

#### Key Implementation Challenge: Storing Variable-Length Values

P4 has limitations (no loops, no complex data structures). Storing values larger than a single register is a challenge.

- **Solution:** Split a large value across multiple `register arrays`.
- **Mechanism:** A lookup table maps a key to a `bitmap` and an `index`. The bitmap indicates which arrays hold parts of the value, and the index specifies the slot within those arrays. The control plane is responsible for managing this allocation (a bin-packing problem).

### 4. Core Enabling Technology: Probabilistic Data Structures in P4

To efficiently track hot items without using excessive memory, NetCache relies on probabilistic data structures.

#### a. Bloom Filter (Membership Testing)

Answers the question: "Have I seen this item before?"

- **How it works:** Uses `k` hash functions to map an item to `k` bits in a bit array. To insert, set all `k` bits to 1. An item *probably* exists if all its `k` bits are 1. It *definitely does not* exist if at least one bit is 0.
- **Properties:**
    - **Space-efficient.**
    - **No False Negatives.**
    - **Allows False Positives.**
- **Problem:** Cannot delete items from a standard Bloom Filter.

#### b. Counting Bloom Filter (Membership with Deletion)

- **How it works:** Instead of a bit array, it uses an array of small counters (e.g., 4 bits).
    - **Insert:** Increment the `k` counters.
    - **Delete:** Decrement the `k` counters.
- **Use Case:** Useful when items need to be removed from the tracked set.

#### c. Count-Min Sketch (Frequency Estimation)

Answers the question: "How many times have I seen this item?"

- **How it works:** Uses `d` arrays of counters, each with its own hash function.
    - **Update:** On seeing an item, increment its corresponding counter in all `d` arrays.
    - **Query:** To get an item's frequency, read its `d` counters and return the **minimum** value. The minimum is used because hash collisions can only cause over-counting, making it the most accurate estimate.
- **Use Case in NetCache:** To identify which non-cached keys are becoming "hot" and should be added to the cache.

### 5. The NetCache Cache Update Flow

1.  **Report:** The P4 data plane uses a **Count-Min Sketch** to track request frequencies for non-cached keys. It reports the hottest keys to the control plane. A **Bloom Filter** is used to prevent sending duplicate reports for the same hot key.
2.  **Compare:** The control plane receives the report and decides which new keys to cache.
3.  **Fetch:** The control plane fetches the full values for these new hot keys from the backend key-value stores.
4.  **Insert/Evict:** The control plane installs the new key-value pairs into the P4 switch's registers, evicting older or less popular items to make space.

### 6. Conclusion

- P4 is a powerful tool for creating stateful network applications like in-network caching.
- By moving caching logic into the network, **NetCache** can dramatically improve the performance and scalability of distributed key-value stores.
- **Probabilistic data structures** (Bloom Filters, Count-Min Sketches) are essential for implementing these systems efficiently in the constrained environment of a switch ASIC.# Study Notes: Datacenter Network Programming - In-Network Caching (NetCache)

These notes summarize the main points from the "In-Network Caching (NetCache)" lecture.

### 1. The Problem with Traditional Datacenter Caching

Cloud providers require low latency, high scalability (billions of reads/sec), and efficient handling of popular ("hot") content.

- **Traditional Architecture:** A `Memcache` layer sits between web servers and databases to speed up reads.
- **Scaling Problem:** To handle large amounts of data, the cache is distributed across many `memcached` servers using consistent hashing.
- **The Incast Problem:** A single client request often needs to fetch data from *hundreds* of different cache servers. The simultaneous flood of responses can overwhelm the requesting web server, increasing tail latency. This high "fan-out" is a major bottleneck.

### 2. NetCache: An In-Network Solution

NetCache proposes moving the caching logic directly into the network data plane using P4-programmable switches.

- **Core Idea:** Use the Top-of-Rack (ToR) switch as a "front-line" cache. The switch intercepts key-value requests, serves hits for "hot" items directly, and forwards misses to the backend servers.
- **Benefits:**
    - **Absorbs Hot Traffic:** Prevents requests for the most popular items from ever reaching the server fleet.
    - **Ultra-Low Latency:** Serves cached items at line rate (microsecond latency).
    - **Reduces Fan-Out:** Drastically reduces the number of servers a web server needs to contact.
- **Target Workload:** Ideal for workloads with small objects, more reads than writes, and rapidly changing key popularity (skewed workloads).

### 3. NetCache Architecture

NetCache is composed of two main components:

- **Data Plane (P4 Switch):**
    - **Parser:** Parses custom key-value request packets.
    - **Registers & Tables:** Stores the actual key-value data.
    - **Statistics Engine:** Tracks request frequencies to identify hot items.
    - **Actions:** Serves cached items or forwards requests.
- **Control Plane (CPU):**
    - **Cache Management:** Runs the cache replacement policy (e.g., insert new hot items, evict cold items).
    - **Communication:** Communicates with the data plane via an API to get hot-key reports and to update the switch's cache tables.

#### Key Implementation Challenge: Storing Variable-Length Values

P4 has limitations (no loops, no complex data structures). Storing values larger than a single register is a challenge.

- **Solution:** Split a large value across multiple `register arrays`.
- **Mechanism:** A lookup table maps a key to a `bitmap` and an `index`. The bitmap indicates which arrays hold parts of the value, and the index specifies the slot within those arrays. The control plane is responsible for managing this allocation (a bin-packing problem).

### 4. Core Enabling Technology: Probabilistic Data Structures in P4

To efficiently track hot items without using excessive memory, NetCache relies on probabilistic data structures.

#### a. Bloom Filter (Membership Testing)

Answers the question: "Have I seen this item before?"

- **How it works:** Uses `k` hash functions to map an item to `k` bits in a bit array. To insert, set all `k` bits to 1. An item *probably* exists if all its `k` bits are 1. It *definitely does not* exist if at least one bit is 0.
- **Properties:**
    - **Space-efficient.**
    - **No False Negatives.**
    - **Allows False Positives.**
- **Problem:** Cannot delete items from a standard Bloom Filter.

#### b. Counting Bloom Filter (Membership with Deletion)

- **How it works:** Instead of a bit array, it uses an array of small counters (e.g., 4 bits).
    - **Insert:** Increment the `k` counters.
    - **Delete:** Decrement the `k` counters.
- **Use Case:** Useful when items need to be removed from the tracked set.

#### c. Count-Min Sketch (Frequency Estimation)

Answers the question: "How many times have I seen this item?"

- **How it works:** Uses `d` arrays of counters, each with its own hash function.
    - **Update:** On seeing an item, increment its corresponding counter in all `d` arrays.
    - **Query:** To get an item's frequency, read its `d` counters and return the **minimum** value. The minimum is used because hash collisions can only cause over-counting, making it the most accurate estimate.
- **Use Case in NetCache:** To identify which non-cached keys are becoming "hot" and should be added to the cache.

### 5. The NetCache Cache Update Flow

1.  **Report:** The P4 data plane uses a **Count-Min Sketch** to track request frequencies for non-cached keys. It reports the hottest keys to the control plane. A **Bloom Filter** is used to prevent sending duplicate reports for the same hot key.
2.  **Compare:** The control plane receives the report and decides which new keys to cache.
3.  **Fetch:** The control plane fetches the full values for these new hot keys from the backend key-value stores.
4.  **Insert/Evict:** The control plane installs the new key-value pairs into the P4 switch's registers, evicting older or less popular items to make space.

### 6. Conclusion

- P4 is a powerful tool for creating stateful network applications like in-network caching.
- By moving caching logic into the network, **NetCache** can dramatically improve the performance and scalability of distributed key-value stores.
- **Probabilistic data structures** (Bloom Filters, Count-Min Sketches) are essential for implementing these systems efficiently in the constrained environment of a switch ASIC.