# transaction.rs - Transaction Structure

The transaction module defines the fundamental value transfer mechanism in Bunkercoin, implementing Ed25519 digital signatures with deterministic nonce generation and Blake3 hashing for radio-optimized financial operations.

## Transaction Structure

```rust
// src/transaction.rs (lines 4-13)
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Transaction {
    pub sender: Vec<u8>,        // Ed25519 public key
    pub recipient: Vec<u8>,     // Ed25519 public key  
    pub amount: u64,            // Amount in base units
    pub nonce: u64,             // Deterministic nonce
    pub signature: Vec<u8>,     // Ed25519 signature
}
```

## Ed25519 Cryptographic Implementation

### Digital Signature Verification

```rust
// src/transaction.rs (lines 15-23)
impl Transaction {
    pub fn verify(&self) -> bool {
        let pk = PublicKey::from_bytes(&self.sender).unwrap();
        let data = self.data_to_sign();
        let sig_bytes: [u8; 64] = self.signature.clone().try_into().expect("sig length");
        let sig = Signature::from_bytes(&sig_bytes).unwrap();
        pk.verify(&data, &sig).is_ok()
    }
}
```

### Signing Data Generation

```rust
// src/transaction.rs (lines 25-33)
fn data_to_sign(&self) -> Vec<u8> {
    let mut data = Vec::new();
    data.extend_from_slice(&self.sender);
    data.extend_from_slice(&self.recipient);
    data.extend_from_slice(&self.amount.to_le_bytes());
    data.extend_from_slice(&self.nonce.to_le_bytes());
    data
}
```

{% hint style="info" %}
**Ed25519 Advantages:** Ed25519 provides 128-bit security equivalent with fast verification and small signature sizes (64 bytes), making it ideal for bandwidth-constrained radio transmissions while maintaining cryptographic strength.
{% endhint %}

## Blake3 Transaction Hashing

```rust
// src/transaction.rs (lines 35-39)
pub fn hash(&self) -> [u8; 32] {
    let serialized = bincode::serialize(self).unwrap();
    blake3::hash(&serialized).into()
}
```

The Blake3 hash function provides:
- **Deterministic hashing:** Same transaction always produces same hash
- **Collision resistance:** Cryptographically secure transaction identification
- **Fast computation:** Optimized for low-power radio bridge environments

## Deterministic Nonce System

Unlike Bitcoin's UTXO model, Bunkercoin uses a simplified account-based system with deterministic nonces:

### Nonce Generation Strategy
```rust
// From wallet.rs - deterministic nonce calculation
let nonce = self.current_nonce; // Incremented for each transaction
self.current_nonce += 1;
```

### Anti-Replay Protection
- **Sequential nonces:** Each transaction must have the next expected nonce
- **State validation:** Prevents double-spending through nonce verification
- **Deterministic ordering:** Ensures consistent transaction processing across nodes

## Transaction Lifecycle

### 1. Creation and Signing
```rust
// Transaction creation pattern (from wallet.rs)
pub fn create_tx(&mut self, recipient: Vec<u8>, amount: u64) -> Transaction {
    let nonce = self.current_nonce;
    self.current_nonce += 1;
    
    let mut tx = Transaction {
        sender: self.public_key.to_bytes().to_vec(),
        recipient,
        amount,
        nonce,
        signature: Vec::new(),
    };
    
    let data = tx.data_to_sign();
    let signature = self.secret_key.sign(&data);
    tx.signature = signature.to_bytes().to_vec();
    tx
}
```

### 2. Network Propagation
- **HTTP broadcast:** POST /tx endpoint accepts individual transactions
- **Mempool injection:** Transactions stored pending block inclusion
- **Peer synchronization:** GET /mempool shares pending transactions

### 3. Block Inclusion
- **Deterministic ordering:** Transactions sorted by hash for consistent blocks
- **Merkle root calculation:** Blake3 hash of all transactions
- **Finalization:** Transactions become immutable once included in blocks

## Radio-Optimized Features

### Compact Binary Format
```rust
// Typical transaction size breakdown
struct TransactionSize {
    sender: [u8; 32],      // 32 bytes - Ed25519 public key
    recipient: [u8; 32],   // 32 bytes - Ed25519 public key
    amount: u64,           // 8 bytes  - Value transfer amount
    nonce: u64,            // 8 bytes  - Anti-replay protection
    signature: [u8; 64],   // 64 bytes - Ed25519 signature
}
// Total: 176 bytes per transaction
```

### JS8Call Compatibility
- **Single frame transmission:** Most transactions fit in one 256-byte JS8Call frame
- **Error detection:** Built-in checksums enable corruption detection
- **Human readable:** JSON format allows manual verification during radio operations

## Validation and Security

### Cryptographic Validation Chain
```rust
// Complete transaction validation
fn validate_transaction(tx: &Transaction) -> Result<(), ValidationError> {
    // 1. Signature verification
    if !tx.verify() {
        return Err(ValidationError::InvalidSignature);
    }
    
    // 2. Public key format validation
    if tx.sender.len() != 32 || tx.recipient.len() != 32 {
        return Err(ValidationError::InvalidKeyFormat);
    }
    
    // 3. Amount validation
    if tx.amount == 0 {
        return Err(ValidationError::ZeroAmount);
    }
    
    // 4. Nonce validation (state-dependent)
    // Checked during mempool injection
    
    Ok(())
}
```

### Security Properties
| Property | Implementation | Purpose |
|----------|---------------|---------|
| **Authentication** | Ed25519 signature verification | Proves transaction authorization |
| **Integrity** | Blake3 hash validation | Detects transmission corruption |
| **Non-repudiation** | Immutable blockchain storage | Prevents transaction denial |
| **Replay protection** | Sequential nonce system | Prevents transaction reuse |

## Error Handling and Recovery

### Malformed Transaction Handling
```rust
// Graceful error handling pattern
match Transaction::from_json(&received_data) {
    Ok(tx) => {
        if tx.verify() {
            blockchain.add_transaction(tx);
        } else {
            eprintln!("Invalid signature, discarding transaction");
        }
    }
    Err(e) => {
        eprintln!("Malformed transaction data: {}", e);
        // Continue processing other transactions
    }
}
```

### Radio Transmission Errors
- **Checksum validation:** JSON deserialization detects corruption
- **Signature verification:** Cryptographic proof of integrity
- **Graceful degradation:** Invalid transactions discarded without affecting system
- **Retransmission:** Failed transactions can be rebroadcast by sender

## Future Enhancements

### Planned Transaction Features
- **Multi-signature support:** m-of-n signature schemes for enhanced security
- **Smart contract integration:** Programmable transaction conditions
- **Privacy enhancements:** Zero-knowledge proofs for transaction privacy
- **UTXO migration:** Transition to unspent transaction output model

### Backward Compatibility
Transaction format extensions will maintain compatibility through:
- **Version fields:** Enable protocol upgrades
- **Optional fields:** New features added as optional extensions
- **Legacy support:** Older transaction formats remain valid 