## Parameters Setup
`CHUNK_LENGTH = 75
L = 1400
SEED = 23
TARGET_SCORE = 0.32285
STRICT_BUFFER = 0.005
GLOBAL_THRESHOLD = 0.3
ANOMALY_THRESHOLD = 0.49
`

- CHUNK_LENGTH: Length of the shingle chunks for stream hashing, representing the "shingles" or substructures of edges.
- L: Number of hash projections used to create sketches, matching StreamSpot’s concept of creating compact representations of graphs.
- SEED: Seed for reproducibility of random operations.
- TARGET_SCORE, STRICT_BUFFER, GLOBAL_THRESHOLD, ANOMALY_THRESHOLD: These values define clustering behavior, thresholds for anomalies, and buffer for strict assignment.
---
## Random Hash Projections
`MAX_UINT64 = np.iinfo(np.uint64).max
H = [np.random.randint(0, MAX_UINT64, CHUNK_LENGTH+2, dtype=np.uint64) for _ in range(L)]`
- This creates a set of random hash vectors (H), similar to StreamSpot’s approach for generating random projections to sketch graphs.
- Each hash vector corresponds to a shingle projection, crucial for the sketch creation step.
---
## Shingle Hashing Function
`def hash_shingle(shingle, randbits):
    sum_hash = int(randbits[0])
    for i, char in enumerate(shingle):
        sum_hash += int(randbits[i+1]) * ord(char)
    return 2 * ((sum_hash >> 63) & 1) - 1 `
- This function hashes a shingle to a binary value using random projections, similar to how StreamSpot converts graph substructures into a sketch.
- Each character of the shingle is combined with the random hash vector to create a signed projection.
---
## Sketch Creation
` def construct_streamhash_sketch(shingle_vector):
    projection = np.zeros(L, dtype=int)
    for shingle, count in shingle_vector.items():
        for i in range(L):
            projection[i] += count * hash_shingle(shingle, H[i])
    sketch = np.where(projection >= 0, 1, 0)
    return sketch` 

- This function generates a StreamHash sketch by projecting shingles of the graph into binary values, representing the structural patterns of the graph.
- Each shingle’s count is used to weight its contribution to the sketch, capturing graph frequency patterns similar to StreamSpot’s compact "sketches."
---
## Hamming Similarity 
`def hamming_similarity(sketch1, sketch2):
    return 1 - hamming(sketch1, sketch2) `
    
- Calculates the similarity between two sketches using Hamming distance, a technique used by StreamSpot to measure similarity between graph sketches.
---
## Processing Edges into Graphs 
`def process_edges(edge_file):
    graphs = defaultdict(nx.DiGraph)
    with open(edge_file, 'r') as f:
        for line in f:
            src_id, src_type, dst_id, dst_type, e_type, gid = line.strip().split()
            graphs[int(gid)].add_edge((src_id, src_type), (dst_id, dst_type), e_type=e_type)
    return graphs`

- Reads edges from an input file and constructs directed graphs for each graph ID (gid), similar to the preprocessing step in StreamSpot where edges are read from streams and stored in graph data structures.
---
## Shingles Vector Creation
`ef construct_shingle_vectors(graphs):
    shingle_vectors = {}
    for gid, graph in graphs.items():
        shingle_vector = defaultdict(int)
        for u, v, data in graph.edges(data=True):
            src_type, dst_type = u[1], v[1]
            shingle = f"{src_type}{data['e_type']}{dst_type}"
            shingle_vector[shingle] += 1
        shingle_vectors[gid] = shingle_vector
    return shingle_vectors
`

- Converts graphs into shingle vectors, representing edge types as shingles, similar to StreamSpot’s method of capturing structural patterns within graphs.
----
## Updating Centroid Sketches 
`def update_centroid_sketches(clusters, graph_sketches):
    centroid_sketches = {}
    for cluster_id, gids in clusters.items():
        cluster_size = len(gids)
        if cluster_size > 0:
            centroid_projection = np.mean([graph_sketches[gid] for gid in gids], axis=0)
            centroid_sketch = np.where(centroid_projection >= 0, 1, 0)
            centroid_sketches[cluster_id] = centroid_sketch
    return centroid_sketches `
- Updates the centroid sketches by averaging the sketches of graphs in a cluster, similar to how StreamSpot maintains centroids for clusters of sketches.
- This step allows dynamic clustering and anomaly detection by comparing new sketches with existing centroids.
---
## Updating Distances and Clustering
`def update_distances_and_clusters(gid, graph_sketches, centroid_sketches, clusters):
    sketch = graph_sketches[gid]
    min_distance = 1.0
    nearest_cluster = None

    for cluster_id, centroid_sketch in centroid_sketches.items():
        sim = hamming_similarity(sketch, centroid_sketch)
        distance = 1.0 - sim
        if distance < min_distance:
            min_distance = distance
            nearest_cluster = cluster_id

    if abs(min_distance - TARGET_SCORE) <= STRICT_BUFFER:
        clusters[0].append(gid)
        print(f"Graph {gid} assigned to cluster 0 with score: {min_distance}")
    elif nearest_cluster == 0 and min_distance <= ANOMALY_THRESHOLD:
        clusters[1].append(gid)
        print(f"Graph {gid} assigned to cluster 1 with score: {min_distance}")
    else:
        print(f"Graph {gid} is considered an anomaly with score: {min_distance}")
`

- Assigns graphs to clusters based on similarity to centroid sketches. If a graph's sketch is close to the target score, it’s assigned to cluster 0; otherwise, it’s either considered an anomaly or assigned to cluster 1 based on distance thresholds.
- This mimics the clustering and anomaly detection logic of StreamSpot, where sketches that deviate from cluster centroids beyond a threshold are marked as anomalies.
---
## Main 
`def main():
    start_time = time.time()

    graphs = process_edges('randomized_email_edges.txt')
    clusters = defaultdict(list)

    shingle_vectors = construct_shingle_vectors(graphs)
    graph_sketches = {}

    initial_gid = list(shingle_vectors.keys())[0]
    initial_sketch = construct_streamhash_sketch(shingle_vectors[initial_gid])
    graph_sketches[initial_gid] = initial_sketch
    clusters[0] = [initial_gid]

    for gid, shingle_vector in shingle_vectors.items():
        if gid != initial_gid:
            sketch = construct_streamhash_sketch(shingle_vector)
            graph_sketches[gid] = sketch

    centroid_sketches = update_centroid_sketches(clusters, graph_sketches)

    for gid in graph_sketches.keys():
        if gid != initial_gid:
            update_distances_and_clusters(gid, graph_sketches, centroid_sketches, clusters)

    end_time = time.time()
    runtime = end_time - start_time

    print("\n--- Execution Parameters ---")
    print(f"Chunk length: {CHUNK_LENGTH}")
    print(f"Parallel graphs: 10")
    print(f"Max edges: 10000")
    print(f"Runtime: {runtime:.7f} seconds")
`
- Implements the main execution flow: reads edge data, creates shingle vectors, generates sketches, updates clusters, and detects anomalies.
- Represents the core functionality of StreamSpot by processing graphs as sketches, maintaining centroids, and identifying anomalies in real-time.
