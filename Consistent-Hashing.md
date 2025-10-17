# Consistent Hashing

## What is Consistent Hashing?

Consistent hashing is a distributed hashing technique used to distribute data across multiple servers in a way that minimizes reorganization when servers are added or removed.

## The Problem with Traditional Hashing

### Simple Hash Function Approach

Traditional approach: `server = hash(key) % number_of_servers`

**Problems:**
- When servers are added/removed, almost all keys need to be remapped
- Causes massive data movement
- Results in cache invalidation and performance degradation

**Example:**
- 3 servers: key "user123" → hash("user123") % 3 = Server 1
- Add 1 server (now 4): hash("user123") % 4 = Server 2
- Almost all keys get remapped!

## How Consistent Hashing Works

### The Hash Ring Concept

1. **Create a Ring:** Imagine a circular space from 0 to 2^32-1
2. **Place Servers:** Hash each server and place it on the ring
3. **Place Keys:** Hash each key and place it on the ring
4. **Assignment Rule:** Each key is assigned to the first server found moving clockwise

### Key Properties

- **Minimal Remapping:** When a server is added/removed, only keys near that server are affected
- **Even Distribution:** Keys are distributed roughly evenly (with virtual nodes)
- **Scalability:** Easy to add or remove servers

## Virtual Nodes (Vnodes)

### The Problem

With real nodes only, distribution may be uneven, especially with few servers.

### The Solution

Each physical server is represented by multiple virtual nodes on the ring.

**Benefits:**
- More uniform distribution of keys
- Better load balancing
- Smoother transitions when servers are added/removed

**Implementation:**
```
For each server S:
    For i = 1 to num_virtual_nodes:
        virtual_node = hash(S + "#" + i)
        place virtual_node on ring
```

## Algorithm Details

### Adding a Server

1. Hash the new server to get its position on the ring
2. Insert it into the ring
3. Only keys between the new server and its predecessor need to be moved
4. **Result:** Minimal data movement (approximately 1/N of keys, where N is total servers)

### Removing a Server

1. Find the server's position on the ring
2. Remove it from the ring
3. Keys that were assigned to this server are now assigned to the next server clockwise
4. **Result:** Again, minimal data movement

### Finding a Key

1. Hash the key to get its position on the ring
2. Move clockwise from that position
3. The first server encountered is responsible for that key

## Advantages

1. **Scalability:** Easy to add or remove nodes without major disruption
2. **Minimal Data Movement:** Only K/N keys need to be remapped (K = total keys, N = servers)
3. **Load Distribution:** With virtual nodes, load is distributed evenly
4. **Fault Tolerance:** When a node fails, its load is distributed to neighbors
5. **Heterogeneous Systems:** Can assign more virtual nodes to powerful servers

## Disadvantages

1. **Complexity:** More complex than simple modulo hashing
2. **Cascading Failures:** If a node fails, its neighbor gets overloaded
3. **Non-uniform Distribution:** Without virtual nodes, distribution can be skewed
4. **Overhead:** Need to maintain and search the ring structure

## Use Cases

### 1. Distributed Caching

**Systems:** Memcached, Redis Cluster

**Why:** Minimize cache invalidation when cache servers are added/removed

### 2. Distributed Databases

**Systems:** Cassandra, DynamoDB, Riak

**Why:** Distribute data evenly across database nodes

### 3. Load Balancing

**Systems:** Content Delivery Networks (CDNs)

**Why:** Route requests consistently to the same server

### 4. Distributed File Systems

**Systems:** Amazon S3, Ceph

**Why:** Determine which storage node holds specific data

### 5. Peer-to-Peer Networks

**Systems:** BitTorrent DHT, Chord

**Why:** Locate resources in a distributed network

## Real-World Implementations

### Amazon DynamoDB

- Uses consistent hashing for data partitioning
- Each node is responsible for a range of hash values
- Automatic rebalancing when nodes are added/removed
- Virtual nodes for better distribution

### Apache Cassandra

- Token ring architecture based on consistent hashing
- Each node owns a range of tokens
- Vnodes (virtual nodes) enabled by default (256 per node)
- Handles node failures gracefully

### Memcached

- Client-side consistent hashing
- Ketama algorithm (specific implementation)
- Reduces cache misses during topology changes

### Redis Cluster

- 16384 hash slots distributed across nodes
- Similar concept to consistent hashing
- Automatic resharding support

## Implementation Considerations

### Data Structures

**Sorted Map/Tree:**
- Store server positions in sorted order
- Binary search to find the next server clockwise
- Time complexity: O(log N) for lookups

**Example (conceptual):**
```
TreeMap<Long, Server> ring = new TreeMap<>()
```

### Number of Virtual Nodes

**Too few:** Uneven distribution

**Too many:** More memory overhead, slower lookups

**Typical values:** 100-500 virtual nodes per physical server

### Hash Function Selection

**Requirements:**
- Uniform distribution
- Fast computation
- Low collision rate

**Common choices:**
- MD5 (128-bit)
- SHA-1 (160-bit)  
- MurmurHash (fast, good distribution)

## Consistent Hashing vs. Other Approaches

### vs. Simple Modulo Hashing

| Aspect | Modulo | Consistent Hashing |
|--------|---------|--------------------|
| Simplicity | Simple | More complex |
| Remapping on change | All keys | Minimal (1/N) |
| Load balance | Perfect | Good (with vnodes) |
| Scalability | Poor | Excellent |

### vs. Rendezvous Hashing (HRW)

**Rendezvous Hashing:**
- Calculate hash for each server for a given key
- Select server with highest hash
- No ring structure needed
- Better for small numbers of servers

**Consistent Hashing:**
- Better for large clusters
- More efficient lookups with sorted structure
- Industry standard for distributed systems

## Variants and Extensions

### Jump Consistent Hash

- Google's space-efficient algorithm
- O(1) space complexity
- No virtual nodes needed
- Trade-off: can only add servers sequentially

### Bounded Load Consistent Hashing

- Addresses "hot spot" problem
- Limits maximum load on any server
- If a server is overloaded, route to next available

### Multi-Probe Consistent Hashing

- Check multiple positions on the ring
- Choose best server based on load
- Better load balancing

## Best Practices

1. **Use Virtual Nodes:** Essential for even distribution (100-500 per server)
2. **Monitor Distribution:** Track key distribution across servers
3. **Replication:** Store copies on multiple consecutive servers
4. **Handle Failures:** Implement health checks and automatic failover
5. **Gradual Changes:** Add/remove servers gradually to avoid overload
6. **Choose Good Hash Functions:** Balance speed and distribution quality
7. **Consider Load:** Assign more vnodes to more powerful servers
8. **Test Thoroughly:** Simulate various failure scenarios

## Example: Key Assignment Process

**Setup:**
- 3 physical servers: A, B, C
- Each with 3 virtual nodes
- Ring positions (simplified): 0-999

**Server positions after hashing:**
- A#1: 100, A#2: 300, A#3: 700
- B#1: 200, B#2: 500, B#3: 900
- C#1: 400, C#2: 600, C#3: 800

**Key assignment:**
- Key "user1" hashes to 250 → assigned to B#2 (next clockwise at 500)
- Key "user2" hashes to 150 → assigned to A#2 (next clockwise at 300)
- Key "user3" hashes to 950 → assigned to A#1 (wraps around to 100)

**If Server B fails:**
- Only keys between 900 and 200 are affected
- They get reassigned to the next available server (A or C)
- Other keys remain unchanged

## Common Pitfalls

1. **Too Few Virtual Nodes:** Results in uneven distribution
2. **Not Handling Edge Cases:** Ring wraparound, empty ring, single server
3. **Ignoring Replication:** Single server failure loses data
4. **Poor Hash Function:** Creates hotspots
5. **Not Monitoring:** Undetected load imbalances
6. **Synchronization Issues:** Ring state inconsistencies in distributed systems

## Conclusion

Consistent hashing is a fundamental technique in distributed systems that enables:
- Efficient scaling by minimizing data movement
- Better fault tolerance through localized impact
- Improved load distribution with virtual nodes

It's widely used in production systems like Cassandra, DynamoDB, and content delivery networks. Understanding consistent hashing is essential for designing scalable distributed systems.

**Key Takeaway:** When adding or removing servers, consistent hashing ensures only a small fraction (approximately 1/N) of keys need to be remapped, compared to nearly all keys in traditional hashing approaches.
