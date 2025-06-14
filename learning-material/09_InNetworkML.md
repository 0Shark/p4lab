
# InNetworkML

These are summary notes from the "In-Network Support for Distributed ML" lecture, covering the SwitchML and "Do Switches Dream of Machine Learning?" topics.

## Part 1: SwitchML - Accelerating Distributed ML Training

### 1. The Problem: Communication is the Bottleneck

- **Scalability Issues:** Modern AI models and datasets are too large for a single server, requiring distributed training across many "worker" nodes.
- **Data Parallelism:** A common distributed training method where:
    1.  Each worker computes model updates (`U_i`) on a subset of the data.
    2.  All workers must synchronize by exchanging and aggregating these updates in every iteration.
- **The Bottleneck:** This all-to-all communication of model updates (often 100s of MBs to GBs per iteration) consumes a significant portion of training time, making the network the primary performance limiter.

### 2. The Solution: In-Network Aggregation

- **Core Idea:** Instead of workers sending data to each other or a central parameter server, use a P4 programmable switch to perform the aggregation directly in the network.
- **How it Works (SwitchML):**
    1.  **Stream:** Workers stream their model updates (as packets) to the switch.
    2.  **Aggregate:** The switch performs the aggregation (e.g., summation) on the incoming packets in its pipeline.
    3.  **Multicast:** The switch sends the final aggregated result back to all workers simultaneously.

### 3. Challenges & Design of SwitchML

P4 switches have significant limitations that must be addressed:

| Challenge              | SwitchML Solution                                       |
| ---------------------- | ------------------------------------------------------- |
| **No floating-point math** | **Integer Quantization**: Convert floats to fixed-point integers on the host before sending. Accuracy is shown to be comparable. |
| **Limited memory**     | **Pool-based streaming aggregation**: A clever slot-matching mechanism to manage aggregation state within the switch's small memory. |
| **Limited computation**  | **Combined Switch-Host Architecture**: The switch does the simple, fast `fixed-point aggregation`. Hosts handle complex logic like `quantization` and `failure recovery`. |
| **Packet loss**        | A `failure recovery protocol` is implemented on the hosts. |

### 4. Results

- **Speedup:** Up to **2.27x** training throughput improvement on 100Gbps networks.
- **Key Insight:** The speedup is most significant when the computation/communication ratio is low (i.e., when communication is the dominant bottleneck).

---

## Part 2: In-Network ML Inference

### 1. The Question: Can a Switch Run an ML Model?

- This part shifts focus from *training* to *inference* (e.g., classification).
- **Core Idea:** The architecture of a P4 switch pipeline (Parser -> Match-Action stages) is conceptually similar to a **Decision Tree**.
    - **Packet Headers** -> Features
    - **Match Tables** -> Decision Nodes
    - **Actions** -> Classifications / next step in the tree

### 2. Mapping ML Models to a P4 Switch

#### A. Decision Trees
- The mapping is **intuitive**. Each level of the tree can be implemented in a pipeline stage. The switch simply follows the decision path for each packet.

#### B. Other Models (SVM, K-means, Naive Bayes)
- **Challenge:** These models rely on complex math (multiplication, exponents, roots) that P4 switches **cannot** perform.
- **Solution:** **Approximate and Pre-compute**.
    1.  **Don't compute in the switch:** The complex model is fully trained offline on a server.
    2.  **Store classifications, not values:** The switch data plane is populated with lookup tables that represent the *outcome* of the complex math.
    3.  **Example (SVM):** For each hyperplane, a table checks which side a packet's features fall on. The action is to "vote" for a class (A or B). A final stage counts the votes to make the final classification.

### 3. A General Framework

- A framework can map different ML models to the switch:
    - **Control Plane:** Translates a trained model (e.g., from Scikit-learn) into P4 table rules.
    - **Data Plane:** A generic P4 pipeline that performs feature lookups, code/vote aggregation, and a final classification lookup.
    - **Backend:** Packets with low classification confidence can be forwarded to a server for more complex processing.

### 4. Summary & Future Work

- **In-Network Training (SwitchML):** In-network aggregation is a powerful technique to accelerate distributed training by tackling the communication bottleneck.
- **In-Network Inference:** Simple, stateless ML models can be mapped to the P4 pipeline for line-rate inference, especially for tasks like traffic classification. The key is to **pre-compute the model and use lookup tables**.
- **Future Work:** Increasing model size and distributing models across multiple switches.