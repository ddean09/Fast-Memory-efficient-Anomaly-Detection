
Overview of StreamSpot:
StreamSpot is designed for real-time anomaly detection in streaming heterogeneous graphs, with applications like detecting advanced persistent threats (APT). The approach handles temporal graphs with typed nodes and edges and maintains a memory-efficient representation of evolving graphs.

Key Components of StreamSpot:
Graph Representation:

Graphs are represented using shingle vectors, which are generated from the k-hop neighborhood of a node by traversing its outgoing edges. These shingles capture the local structure and the temporal order of edges.
The similarity between two graphs is computed using cosine similarity between their shingle vectors.
Shingle Sketching:

Instead of storing the shingle vectors explicitly, StreamSpot constructs sketches of the graphs. Sketches are compact representations that approximate the pairwise similarities of graphs, providing memory efficiency.
The sketches are updated dynamically as new edges arrive.
StreamHash:

StreamSpot introduces StreamHash to maintain a summary of graph representations online with constant-time updates and bounded memory consumption.
StreamHash uses a family of hash functions that map shingles to {+1, -1}. These hash functions are chosen to be uniform and pairwise-independent.
Each graph has a projection vector that is updated using these hash functions, allowing StreamSpot to compute the similarity of graphs based on their projection vectors.
Clustering:

StreamSpot uses a centroid-based clustering approach to capture normal behavior. It maintains clusters of graphs based on their sketch similarities.
New graphs are dynamically added to clusters or flagged as anomalies if they deviate significantly from the nearest cluster.
Anomaly Detection:

Anomalies are detected in real time by measuring the distance between a graph’s sketch and the nearest cluster centroid. If the distance exceeds a threshold, the graph is flagged as anomalous.
Each cluster has an anomaly threshold set to 3 standard deviations greater than the mean distance between its graphs and the centroid.
Time and Space Complexity:

StreamSpot’s time complexity for updating a graph’s sketch is O(L + |s|²max) per edge, where L is the sketch length and |s|max is the maximum size of a shingle.
The space complexity is O(cL + N), where c is the number of graphs, L is the sketch length, and N is the maximum number of edges retained in memory.
​

1. Maintaining Sketches Incrementally:
When a new edge arrives, it modifies both the outgoing and incoming shingles of the graph.
For each new shingle, the projection vector is updated by adding (for incoming shingle) or subtracting (for outgoing shingle) the corresponding hash value.
The sketch is updated by checking if the projection vector changes sign after an update. If so, the corresponding bit in the sketch is flipped.
2. Clustering and Anomaly Detection:
Initially, StreamSpot is bootstrapped using a training set of benign graphs, which are clustered using the K-medoids algorithm.
New edges update the graph sketches in real-time, and the graph’s distance to each cluster centroid is computed.
If the graph is close enough to a cluster centroid, it is assigned to that cluster. Otherwise, it is flagged as an anomaly.
StreamSpot Evaluation:
Datasets: The system was evaluated on flow-graphs derived from normal and attack scenarios. These graphs represent real-world system logs, where nodes are typed (e.g., process, file) and edges represent system calls (e.g., read, fork).
Detection Performance: StreamSpot demonstrated high anomaly detection accuracy, even under strict memory constraints. The average precision of the system was over 90%, indicating effective ranking of anomalous graphs.
Key Contributions of StreamSpot:
Memory-Efficient Sketching: StreamSpot’s sketch-based representation enables memory-efficient handling of streaming heterogeneous graphs.
Real-Time Clustering and Anomaly Detection: It maintains clusters dynamically and detects anomalies in real time.
Scalable and Fast: StreamSpot processes over 100,000 edges per second, making it suitable for real-time applications with low latency requirements.
### Problem Description:
The primary problem StreamSpot addresses is **anomaly detection in streaming heterogeneous graphs**, which consist of multiple types of nodes and edges. This type of anomaly detection is particularly important in **Advanced Persistent Threat (APT)** detection at the host level. In this scenario, **system logs** capturing events like memory accesses and system calls are used to generate **information flow graphs**. These graphs, which evolve over time, must be monitored in real-time to detect abnormal behavior indicating malicious activity.

### Challenges:
1. **Heterogeneity**: The system needs to handle various types of nodes and edges.
2. **Streaming Nature**: Graphs evolve over time, and new graphs are continuously generated. These must be processed in real-time without the benefit of storing the entire graph.
3. **Memory Constraints**: The solution must be memory-efficient, as storing the entire graph structure in memory is infeasible due to the potentially large size of the data.
4. **Real-Time Detection**: The goal is to detect anomalies in real-time with minimal delay.

### Solution (StreamSpot Approach):
StreamSpot solves this problem by leveraging **clustering-based anomaly detection** for streaming graphs. Its primary innovations are in the way it:
- Represents graphs using **shingles**, which capture local substructures in a graph.
- Uses **StreamHash**, a sketching algorithm that creates memory-efficient representations (sketches) of these graphs.
- Implements **dynamic clustering** to detect anomalies, updating the clusters as new edges arrive and assigning graphs to clusters based on their sketches.

#### Key Techniques:
1. **Graph Representation**:
   - Graphs are represented using **shingle frequency vectors**. A **k-shingle** is a sequence of edges traversed in the k-hop neighborhood of a node. The shingle frequency vector counts the occurrences of unique shingles.
   
2. **Sketching with StreamHash**:
   - StreamSpot uses **StreamHash** to create compact, memory-efficient sketches of graphs, which allow the system to compute the **cosine similarity** between graphs without storing the entire structure. These sketches are updated in real-time as new edges arrive.

3. **Clustering and Anomaly Detection**:
   - StreamSpot performs clustering using a **centroid-based method**. As new graphs are processed, they are compared to cluster centroids, and anomalies are flagged if a graph's sketch is too far from all centroids.
   - Clustering is updated incrementally as the graphs evolve. If a graph moves too far from its assigned cluster, it is re-assigned or flagged as an anomaly.

### Experiments and Results:

#### Datasets:
StreamSpot was evaluated on **flow-graphs** derived from benign and attack scenarios:
1. **Benign Scenarios**: These involved normal activities like watching YouTube, browsing CNN, checking Gmail, downloading files, and playing a video game.
2. **Attack Scenario**: A drive-by download triggered by visiting a malicious URL that exploited a Flash vulnerability and gained root access to the host.
   
- Each scenario was run on a Linux machine, where **system calls** were traced and used to construct flow-graphs. These datasets consisted of over **24 million edges** and were split into three different sets, each corresponding to different combinations of benign and attack graphs.

#### Evaluation Settings:
StreamSpot was tested under two different settings:
1. **Static Evaluation**: 
   - A portion of benign graphs (e.g., 75%) was used to initialize the clusters, and the remaining benign graphs, along with attack graphs, were used for testing. In this setting, all data was stored in memory, and the goal was to measure the accuracy of StreamSpot before introducing memory and time constraints.
   
2. **Streaming Evaluation**:
   - StreamSpot was initialized with a **bootstrap clustering** and then evaluated as new edges were streamed in. The test graphs were processed online, one edge at a time, and snapshots of the anomaly scores were retained for evaluation every 10,000 edges. StreamSpot was also tested under memory constraints by limiting the sketch size and the number of edges retained in memory.

#### Results:
- **Detection Performance**:
   - StreamSpot achieved **high anomaly detection accuracy** in both static and streaming settings. On the static data, it achieved an **average precision (AP) of 0.9** and a **near-ideal AUC** (Area Under the ROC Curve). 
   - In the streaming setting, performance showed periodic dips when new groups of graphs arrived, but the system recovered quickly as more edges became available, indicating a small detection delay. Over time, as more data was processed, StreamSpot’s clustering became more accurate, and the detection performance improved.

- **Memory Efficiency**:
   - StreamSpot demonstrated **excellent memory efficiency**, processing **over 100,000 edges per second** while keeping memory usage bounded. Even with strict memory constraints, the system retained high detection accuracy, making it well-suited for real-time applications with large data streams.

- **Sketch Size**:
   - The system was able to maintain its detection performance while adjusting sketch sizes, showing robustness to different memory constraints.

#### Comparison to Other Models:
StreamSpot was compared to **Isolation Forest (iForest)**, a popular anomaly detection method that uses a different approach based on isolating anomalies through random partitioning. The following were key findings from the comparison:
1. **Precision-Recall (PR) Curves**: StreamSpot consistently outperformed iForest in both static and streaming settings, achieving a **higher average precision** and **better ranking** of attack graphs.
2. **ROC Curves**: StreamSpot achieved near-perfect ROC curves, significantly outperforming iForest in detecting anomalous graphs.
3. **Efficiency**: While iForest was effective, StreamSpot’s use of **shingle-based sketching** and **dynamic clustering** allowed it to process data faster and with lower memory overhead, making it a better choice for real-time anomaly detection in streaming graphs.

### Conclusion:
StreamSpot is a highly effective system for **real-time anomaly detection** in streaming heterogeneous graphs, offering both high detection accuracy and efficient memory usage. It addresses the critical challenges of heterogeneity, streaming nature, and memory constraints by employing novel techniques like **StreamHash sketching** and **dynamic clustering**. Compared to traditional methods like iForest, StreamSpot demonstrates superior performance, especially in handling large volumes of data in real-time.

