# block.rs - Block Structure

The block module defines the fundamental data structures for Bunkercoin blocks, implementing a radio-optimized blockchain format with Blake3 cryptographic hashing and compact serialization suitable for HF radio transmission.

## Block Structure Overview

```rust
// src/block.rs (lines 4-17)
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Block {
    pub header: BlockHeader,
    pub transactions: Vec<Transaction>,
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct BlockHeader {
    pub version: u32,
    pub prev_hash: [u8; 32],
    pub merkle_root: [u8; 32],
    pub timestamp: u64,
    pub vdf_output: [u8; 32],
    pub difficulty: u32,
    pub nonce: u64,
    pub tx_count: u32,
    pub signature: Vec<u8>,
}
```

## Blake3 Cryptographic Hashing

```rust
// src/block.rs (lines 19-25)
impl Block {
    pub fn hash(&self) -> [u8; 32] {
        let encoded = bincode::serialize(&self.header).unwrap();
        let hash = blake3::hash(&encoded);
        hash.into()
    }
}
```

{% hint style="info" %}
**Blake3 Performance:** Blake3 is significantly faster than SHA-256 while providing equivalent security. This is crucial for resource-constrained radio bridge environments where efficient hashing reduces power consumption and processing delays.
{% endhint %}

## Radio-Optimized Design Features

### Compact Binary Serialization
- **Bincode format:** Minimal overhead binary encoding
- **Fixed-size hashes:** 32-byte Blake3 outputs for predictable transmission
- **Type safety:** Serde ensures consistent serialization across platforms

### 10KB Block Size Limit
The block structure is designed to fit within JS8Call transmission constraints:
- **~40 frames:** Each block fragments into approximately 40 × 256-byte frames
- **Error correction:** Compatible with HF digital mode protocols
- **Atomic transmission:** Complete blocks can be verified independently

## Field-by-Field Analysis

| Field | Size | Purpose | Radio Considerations |
|-------|------|---------|---------------------|
| **version** | 4 bytes | Protocol version identifier | Enables future upgrades without breaking compatibility |
| **prev_hash** | 32 bytes | Links to previous block | Blake3 hash provides cryptographic chain integrity |
| **merkle_root** | 32 bytes | Transaction batch verification | Enables efficient transaction validation |
| **timestamp** | 8 bytes | Block production time | Synchronized with 30-second intervals for radio timing |
| **vdf_output** | 32 bytes | Proof-of-work result | Sequential computation prevents parallel mining |
| **difficulty** | 4 bytes | VDF iteration count | Fixed at 10,000 iterations for predictable timing |
| **nonce** | 8 bytes | Deterministic randomness | Generated from prev_hash + timestamp |
| **tx_count** | 4 bytes | Transaction count | Enables efficient transaction parsing |
| **signature** | Variable | Future extension field | Currently unused, reserved for block signing |

## Integration with Mining Process

The block structure integrates seamlessly with Bunkercoin's deterministic mining:

```rust
// Example block construction (from blockchain.rs)
let header = BlockHeader {
    version: 1,
    prev_hash,
    merkle_root: merkle_root_hash.into(),
    timestamp,
    vdf_output,
    difficulty,
    nonce,
    tx_count: txs.len() as u32,
    signature: Vec::new(), // Reserved for future use
};
let block = Block { header, transactions: txs };
```

## Serialization and Transmission

### JSON Format for HTTP
```json
{
  "header": {
    "version": 1,
    "prev_hash": [0, 0, 0, ...],
    "merkle_root": [142, 245, 78, ...],
    "timestamp": 1704067200,
    "vdf_output": [198, 34, 156, ...],
    "difficulty": 10000,
    "nonce": 15738492847362,
    "tx_count": 3,
    "signature": []
  },
  "transactions": [...]
}
```

### Binary Format for Radio
The bincode serialization produces compact binary data suitable for:
- **JS8Call frames:** Direct frame-by-frame transmission
- **APRS packets:** Integration with amateur radio protocols  
- **FT8/FT4 modes:** Compressed transmission in digital mode protocols

## Error Detection and Recovery

### Cryptographic Integrity
```rust
// Block validation pattern
fn validate_block(block: &Block, prev_block: &Block) -> bool {
    // 1. Verify hash chain
    if block.header.prev_hash != prev_block.hash() {
        return false;
    }
    
    // 2. Verify merkle root
    let computed_merkle = compute_merkle_root(&block.transactions);
    if block.header.merkle_root != computed_merkle {
        return false;
    }
    
    // 3. Verify VDF computation
    let expected_vdf = vdf::compute(&prev_block.hash(), block.header.difficulty);
    if block.header.vdf_output != expected_vdf {
        return false;
    }
    
    true
}
```

### Radio Error Correction
- **CRC checksums:** Built into JSON serialization
- **Frame numbering:** Each JS8Call frame includes sequence numbers
- **Redundant transmission:** Critical blocks can be transmitted multiple times
- **Partial recovery:** Individual transactions can be extracted from corrupt blocks

## Future Enhancements

### Planned Block Structure Extensions
- **Signature field utilization:** Enable block signing for enhanced security
- **Witness data:** Support for segregated witness-style extensions
- **State commitments:** UTXO set hashes for efficient synchronization
- **Radio metadata:** Signal strength and propagation path information

### Backward Compatibility
The version field enables protocol upgrades while maintaining compatibility:
- **Version 1:** Current implementation (deterministic mining, fixed difficulty)
- **Version 2:** Planned adaptive difficulty adjustment
- **Version 3:** Future UTXO commitments and advanced features 