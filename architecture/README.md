# Bunkercoin Architecture Overview

Bunkercoin represents a radical departure from traditional blockchain architectures, designed specifically for operation in hostile, disconnected environments where conventional internet infrastructure is unavailable or unreliable. The system prioritizes deterministic behavior, minimal resource consumption, and radio-friendly communication patterns.

## Core Components

### System Architecture Diagram

**Application Layer**
- CLI Interface (main.rs)
- Command Processing
- Configuration Management

**Core Logic**
- Blockchain State (blockchain.rs)
- Mining Coordinator
- Transaction Pool

**Network Layer**
- HTTP Server (web.rs)
- P2P Protocol (network.rs)
- Event Broadcasting

**Cryptographic Layer**
- Ed25519 Signatures (wallet.rs)
- Blake3 Hashing (transaction.rs)
- VDF Proof-of-Work (vdf.rs)

**Data Structures**
- Block Format (block.rs)
- Transaction Format (transaction.rs)
- Persistent Storage

### Modular Design Philosophy

```rust
// src/lib.rs
pub mod block;
pub mod transaction;
pub mod wallet;
pub mod blockchain;
pub mod network;
pub mod vdf;
pub mod web;
```

Each module encapsulates a specific concern, enabling independent development and testing. The clean separation allows components to be replaced or upgraded without affecting the entire system - crucial for field deployment scenarios.

## Data Flow

### Transaction Lifecycle

1. **Transaction Creation**: Wallet generates Ed25519-signed transaction with deterministic nonce
2. **Mempool Injection**: Transaction added to local mempool with hash-based deduplication
3. **Peer Propagation**: HTTP POST to /tx endpoint broadcasts to network peers
4. **Block Inclusion**: Mining process selects transactions deterministically by hash order
5. **Block Finalization**: VDF computation completes, block added to chain and broadcast to peers

### Mining Process Flow

```
Time-Based Mining Cycle (30-second intervals):

1. Calculate Target Timestamp
   └── Previous Block Time + 30 seconds
   └── Or next 30-second boundary (genesis)

2. Wait Period
   └── Sleep until target timestamp reached
   └── Broadcast countdown to SSE clients

3. Block Construction
   ├── Collect mempool transactions
   ├── Sort deterministically by hash
   ├── Calculate merkle root (Blake3)
   └── Generate deterministic nonce

4. VDF Computation
   ├── Seed: Previous block hash
   ├── Difficulty: 10,000 iterations
   └── Output: Blake3^10000(seed)

5. Block Finalization
   ├── Assemble complete block
   ├── Add to local chain
   ├── Persist to chain.json
   └── Broadcast to peers via HTTP
```

### Network Synchronization Flow

**Peer Discovery**
- Static peer configuration via CLI
- HTTP endpoint scanning on port 10000
- Support for cloud services (render.com)
- DNS resolution for radio bridge nodes

**Chain Synchronization**
- GET /sync for lightweight status check
- Compare local vs peer chain height
- GET /chain for full blockchain download
- Longest chain rule for conflict resolution

## Security Model

{% hint style="danger" %}
**Threat Model & Assumptions**

**Assumptions:**
- Majority of nodes are honest
- Ed25519 cryptography remains secure
- Blake3 hash function is collision-resistant
- Radio transmission errors are detectable

**Threat Scenarios:**
- Network partitioning / radio jamming
- Malicious block/transaction injection
- Replay attacks on radio channels
- Resource exhaustion attacks
{% endhint %}

### Cryptographic Security Layers

#### 1. Transaction Integrity

```rust
// transaction.rs - Signature Verification
pub fn verify(&self) -> bool {
    let pk = PublicKey::from_bytes(&self.sender).unwrap();
    let data = self.data_to_sign();
    let sig_bytes: [u8; 64] = self.signature.clone().try_into().expect("sig length");
    let sig = Signature::from_bytes(&sig_bytes).unwrap();
    pk.verify(&data, &sig).is_ok()
}
```

Ed25519 signatures provide 128-bit security with fast verification, crucial for resource-constrained radio bridge environments.

#### 2. Block Chain Integrity

```rust
// block.rs - Block Hashing
impl Block {
    pub fn hash(&self) -> [u8; 32] {
        let encoded = bincode::serialize(&self.header).unwrap();
        let hash = blake3::hash(&encoded);
        hash.into()
    }
}
```

Blake3 provides cryptographic linking between blocks while being significantly faster than SHA-256, important for low-power radio deployments.

#### 3. Proof-of-Work via VDF

```rust
// vdf.rs - Sequential Computation
pub fn compute(seed: &[u8], difficulty: u32) -> [u8; 32] {
    let mut out = *blake3::hash(seed).as_bytes();
    for _ in 0..difficulty {
        out = *blake3::hash(&out).as_bytes();
    }
    out
}
```

The VDF requires sequential computation that cannot be parallelized, ensuring fair resource consumption and predictable timing for radio synchronization.

### Network Security Considerations

| Security Layer | Implementation | Radio Considerations |
|----------------|---------------|---------------------|
| **HTTP Transport Security** | Unencrypted HTTP with cryptographic signatures | Radio environments often preclude TLS due to overhead constraints |
| **Replay Attack Prevention** | Transaction nonces and block timestamps | Deterministic nonce system ensures each transaction can only be valid once |
| **Resource Exhaustion Protection** | Fixed block intervals, limited block size (10KB) | Hash-based deduplication prevents computational attacks |
| **Network Partition Tolerance** | Longest chain rule | Automatic recovery from network partitions |

### Radio-Specific Security Adaptations

#### HF Radio Threat Model

**Eavesdropping Resistance**
All transaction data is cryptographically signed but not encrypted. While transaction amounts are visible, sender/receiver identity requires knowledge of the corresponding private keys.

**Jamming Tolerance**
The 6-hour block interval provides large transmission windows that can accommodate temporary jamming or propagation blackouts. Nodes automatically resynchronize when communication is restored.

**Error Correction**
JSON serialization with checksums enables detection of transmission errors. The deterministic block structure allows reconstruction of corrupted data from known good fragments.

### Future Security Enhancements

**Planned Improvements**
- Real Wesolowski VDF implementation
- UTXO set validation
- Multi-signature wallet support
- Advanced radio error correction

**Research Areas**
- Zero-knowledge transaction privacy
- Mesh network routing protocols
- Quantum-resistant signatures
- Adaptive difficulty adjustment 