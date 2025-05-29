Absolutely! Here’s how you can implement **consistent hashing with backup failover** for something like Redis in **Rust**.

---

## **1. Consistent Hashing in Rust**

We’ll build a basic consistent hash ring with virtual nodes and a method to get the primary and backup nodes for a key.

**We’ll use:**
- [`md5`](https://crates.io/crates/md5) for hashing
- [`ordered_float`](https://crates.io/crates/ordered-float) for sortable floats (not strictly needed with integers, but helpful if you extend)
- Standard Rust collections

**Add to your `Cargo.toml`:**
```toml
md5 = "0.7"
```

---

## **2. Implementation**

```rust
use md5::{Digest, Md5};
use std::collections::BTreeMap;

#[derive(Debug, Clone)]
struct ConsistentHashRing {
    ring: BTreeMap<u128, String>, // hash -> node address
    replicas: usize,
}

impl ConsistentHashRing {
    fn new(nodes: &[String], replicas: usize) -> Self {
        let mut ring = BTreeMap::new();
        for node in nodes {
            for i in 0..replicas {
                let virtual_node = format!("{}:{}", node, i);
                let hash = Self::hash(&virtual_node);
                ring.insert(hash, node.clone());
            }
        }
        Self { ring, replicas }
    }

    fn hash(key: &str) -> u128 {
        let mut hasher = Md5::new();
        hasher.update(key.as_bytes());
        let result = hasher.finalize();
        // Use first 16 bytes (128 bits) as u128
        u128::from_be_bytes(result.into())
    }

    /// Returns a vector of node addresses: [primary, backup1, ...]
    fn get_nodes(&self, key: &str, num_replicas: usize) -> Vec<String> {
        let key_hash = Self::hash(key);

        let mut selected = Vec::new();
        let mut seen = std::collections::HashSet::new();

        // Start from the first node >= key_hash
        for (_, node) in self.ring.range(key_hash..) {
            if seen.insert(node) {
                selected.push(node.clone());
                if selected.len() == num_replicas {
                    return selected;
                }
            }
        }

        // Wrap around the ring if needed
        for (_, node) in self.ring.range(..) {
            if seen.insert(node) {
                selected.push(node.clone());
                if selected.len() == num_replicas {
                    break;
                }
            }
        }
        selected
    }
}

// Example usage:
fn main() {
    let nodes = vec![
        "redis1:6379".to_string(),
        "redis2:6379".to_string(),
        "redis3:6379".to_string(),
    ];
    let ring = ConsistentHashRing::new(&nodes, 100);

    let key = "user:12345";
    let backends = ring.get_nodes(key, 2); // 1 primary + 1 backup

    println!("For key '{}', use nodes: {:?}", key, backends);

    // Pseudo code for failover
    for node in backends {
        // Try connecting to Redis at `node`
        // If success, break
        // If fail, try next
    }
}
```

---

## **3. Integrating with Redis**

- Use [redis-rs](https://crates.io/crates/redis) for Redis connections.
- On a real error, try the next node in the list returned by `get_nodes`.
- In production, you’d want to cache dead nodes for some interval to avoid hammering them.

---

## **4. Summary**

- The hash ring lets you get an **ordered list of nodes per key**.
- On failure, **try each node in order** (primary, then backups).
- This pattern is widely used for building robust distributed caches and sharded databases.

---

**Let me know if you want a more complete example with actual Redis connections and error handling!**
