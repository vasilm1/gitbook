# blockchain.rs - Core Blockchain Logic

The blockchain module serves as the central nervous system of Bunkercoin, orchestrating all blockchain operations including block validation, mining coordination, transaction management, and peer synchronization. It implements a thread-safe, deterministic blockchain with specialized optimizations for disconnected environments.

## Architecture Overview

### Key Design Principles
- **Deterministic Mining:** Blocks are produced at predictable 30-second intervals
- **Thread Safety:** Arc<Mutex> wrapper enables safe concurrent access
- **Persistent State:** Automatic JSON serialization for blockchain persistence
- **Longest Chain Rule:** Automatic chain replacement for network consensus

## Core Data Structures

```rust
// src/blockchain.rs (lines 7-26)
#[derive(Clone)]
pub struct Blockchain {
    inner: Arc<Mutex<State>>,
}

struct State {
    chain: Vec<Block>,
    mempool: Vec<Transaction>,
    data_dir: PathBuf,
    difficulty: u32,
}

#[derive(Serialize, Deserialize)]
pub struct SyncInfo {
    pub height: usize,
    pub latest_hash: String,
}
```

{% hint style="info" %}
**Thread Safety Architecture:** The blockchain uses Arc<Mutex> to wrap the internal state, allowing safe sharing across multiple async tasks while maintaining data consistency during concurrent mining, networking, and transaction processing operations.
{% endhint %}

## Initialization and Persistence

```rust
// src/blockchain.rs (lines 28-44)
impl Blockchain {
    pub fn load_or_init(dir: &Path) -> anyhow::Result<Self> {
        let chain_path = dir.join("chain.json");
        let chain: Vec<Block> = if chain_path.exists() {
            let data = fs::read_to_string(&chain_path)?;
            serde_json::from_str(&data)?
        } else {
            vec![]
        };
        Ok(Blockchain {
            inner: Arc::new(Mutex::new(State {
                chain,
                mempool: Vec::new(),
                data_dir: dir.to_path_buf(),
                difficulty: 10000,
            })),
        })
    }
}
```

The initialization process demonstrates Bunkercoin's resilient design. If no existing blockchain is found, it starts with an empty chain. The difficulty is hardcoded to 10,000 iterations, balancing security with the need for deterministic timing in radio environments.

## Deterministic Mining Algorithm

```rust
// src/blockchain.rs (lines 76-98)
pub async fn mining_loop(self) {
    loop {
        // Determine the target timestamp for the next block (prev + 30s)
        let target_ts = {
            let s = self.inner.lock().unwrap();
            if let Some(prev) = s.chain.last() {
                prev.header.timestamp + 30
            } else {
                // If no blocks yet, align to next 30-second boundary from Unix time
                let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
                (now / 30 + 1) * 30
            }
        };

        // Sleep until we reach or pass the target timestamp
        let now_secs = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        if now_secs < target_ts {
            let wait = target_ts - now_secs;
            web::broadcast(format!("Waiting {} seconds until next scheduled block", wait));
            tokio::time::sleep(std::time::Duration::from_secs(wait)).await;
        }

        if let Err(e) = self.mine_once() {
            eprintln!("mine error: {:?}", e);
        }
    }
}
```

{% hint style="success" %}
**Innovation:** Unlike traditional competitive mining, Bunkercoin uses deterministic scheduling. Each block is produced exactly 30 seconds after the previous one, eliminating energy waste and enabling predictable radio transmission windows.
{% endhint %}

## Block Construction Process

```rust
// src/blockchain.rs (lines 100-140)
fn mine_once(&self) -> anyhow::Result<()> {
    let s = self.inner.lock().unwrap();
    let prev_hash = if let Some(b) = s.chain.last() {
        b.hash()
    } else {
        [0u8; 32]
    };
    let mut txs: Vec<Transaction> = s.mempool.clone();
    drop(s);
    
    // Sort transactions deterministically by hash
    txs.sort_by(|a, b| a.hash().cmp(&b.hash()));
    
    let merkle_root_hash = blake3::hash(&bincode::serialize(&txs)?);
    
    // Determine timestamp deterministically: previous timestamp + 30 seconds
    let timestamp = {
        let s_check = self.inner.lock().unwrap();
        if let Some(prev_block) = s_check.chain.last() {
            prev_block.header.timestamp + 30
        } else {
            let now_secs = SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs();
            (now_secs / 30) * 30
        }
    };
    
    let difficulty = {
        let s = self.inner.lock().unwrap();
        s.difficulty
    };

    let seed = &prev_hash;
    let vdf_output = vdf::compute(seed, difficulty);

    // Deterministic nonce based on prev_hash and timestamp
    let nonce_seed = blake3::hash(&[&prev_hash[..], &timestamp.to_le_bytes()].concat());
    let nonce = u64::from_le_bytes([nonce_seed.as_bytes()[0], nonce_seed.as_bytes()[1], 
                                   nonce_seed.as_bytes()[2], nonce_seed.as_bytes()[3],
                                   nonce_seed.as_bytes()[4], nonce_seed.as_bytes()[5],
                                   nonce_seed.as_bytes()[6], nonce_seed.as_bytes()[7]]);

    let header = BlockHeader {
        version: 1,
        prev_hash,
        merkle_root: merkle_root_hash.into(),
        timestamp,
        vdf_output,
        difficulty,
        nonce,
        tx_count: txs.len() as u32,
        signature: Vec::new(),
    };
    let block = Block { header, transactions: txs };
    self.add_block(block.clone())?;
    Ok(())
}
```

### Mining Process Breakdown

| Step | Operation | Deterministic Factor |
|------|-----------|---------------------|
| **1. Transaction Ordering** | Sort mempool by transaction hash | Ensures all nodes produce identical blocks |
| **2. Timestamp Calculation** | Previous block + 30 seconds | Predictable timing for radio synchronization |
| **3. VDF Computation** | Blake3^10000(prev_hash) | Proof-of-work with non-parallelizable computation |
| **4. Nonce Generation** | Derived from prev_hash + timestamp | Deterministic but unpredictable values |

## Peer Synchronization

The blockchain module implements sophisticated peer synchronization for network consensus:

### Chain Replacement Logic

```rust
// Longest chain rule implementation
pub fn replace_if_longer(&self, new_chain: Vec<Block>) -> anyhow::Result<bool> {
    let mut s = self.inner.lock().unwrap();
    if new_chain.len() > s.chain.len() {
        s.chain = new_chain;
        self.save_chain(&s)?;
        Ok(true)
    } else {
        Ok(false)
    }
}
```

### Mempool Management

- **Deduplication:** Transactions are stored by hash to prevent duplicates
- **Persistence:** Mempool survives process restarts through file-based storage
- **Broadcasting:** New transactions automatically propagate to HTTP peers

## Thread Safety Implementation

The blockchain uses advanced Rust concurrency patterns:

```rust
// Safe concurrent access pattern
let blockchain_clone = Arc::clone(&blockchain);
tokio::spawn(async move {
    blockchain_clone.mining_loop().await;
});
```

- **Arc (Atomic Reference Counting):** Enables shared ownership across threads
- **Mutex:** Provides exclusive access to mutable state
- **Clone semantics:** Arc cloning is cheap (only increments reference count)

## Radio-Optimized Features

### Deterministic Block Production
Unlike Bitcoin's competitive mining, Bunkercoin produces blocks on a fixed schedule, enabling:
- **Predictable transmission windows** for HF radio
- **Reduced power consumption** (no competitive hashing)
- **Synchronized operation** across disconnected networks

### Compact State Management
- **10KB maximum block size** fits within JS8Call frame limits
- **JSON serialization** enables human-readable debugging
- **Atomic file updates** prevent corruption during power failures

## Error Handling and Recovery

The blockchain module implements comprehensive error handling:

```rust
// Graceful error handling pattern
if let Err(e) = self.mine_once() {
    eprintln!("Mining error: {:?}", e);
    // Continue mining loop rather than crashing
}
```

- **Non-fatal mining errors** don't stop the blockchain
- **Automatic recovery** from file system issues
- **Graceful degradation** during network partitions

## Integration Points

The blockchain module integrates with other system components:

- **web.rs:** Provides HTTP endpoints for peer communication
- **vdf.rs:** Computes proof-of-work for new blocks
- **transaction.rs:** Validates and processes value transfers
- **block.rs:** Manages block structure and hashing 