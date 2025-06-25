# wallet.rs - Key Management

The wallet module implements Ed25519 cryptographic key management, transaction creation, and secure wallet persistence for Bunkercoin. It provides the cryptographic foundation for all user operations while maintaining compatibility with radio-constrained environments.

## Wallet Structure and Key Management

```rust
// src/wallet.rs (lines 6-12)
pub struct Wallet {
    secret_key: SecretKey,
    public_key: PublicKey,
    current_nonce: u64,
}
```

## Ed25519 Key Generation

```rust
// src/wallet.rs (lines 14-23)
impl Wallet {
    pub fn generate() -> Self {
        let mut csprng = OsRng;
        let secret_key = SecretKey::generate(&mut csprng);
        let public_key: PublicKey = (&secret_key).into();
        Self {
            secret_key,
            public_key,
            current_nonce: 0,
        }
    }
}
```

{% hint style="info" %}
**Cryptographically Secure Random Generation:** The wallet uses OsRng (Operating System Random Number Generator) which provides cryptographically secure randomness from the OS entropy pool, ensuring unpredictable private key generation.
{% endhint %}

## Wallet Persistence and Recovery

### JSON Serialization Format

```rust
// src/wallet.rs (lines 32-50)
pub fn save(&self, data_dir: &Path) -> anyhow::Result<()> {
    let wallet_path = data_dir.join("wallet.json");
    let wallet_data = WalletData {
        secret_key_bytes: self.secret_key.to_bytes().to_vec(),
        current_nonce: self.current_nonce,
    };
    let json = serde_json::to_string_pretty(&wallet_data)?;
    fs::write(wallet_path, json)?;
    Ok(())
}

pub fn load_or_generate(data_dir: &Path) -> anyhow::Result<Self> {
    let wallet_path = data_dir.join("wallet.json");
    if wallet_path.exists() {
        let json = fs::read_to_string(wallet_path)?;
        let wallet_data: WalletData = serde_json::from_str(&json)?;
        let secret_key = SecretKey::from_bytes(&wallet_data.secret_key_bytes)?;
        let public_key: PublicKey = (&secret_key).into();
        Ok(Self {
            secret_key,
            public_key,
            current_nonce: wallet_data.current_nonce,
        })
    } else {
        Ok(Self::generate())
    }
}
```

### Security Considerations

{% hint style="warning" %}
**Private Key Storage:** The wallet stores private keys in plain JSON format on disk. In production deployments, this should be enhanced with encryption, hardware security modules, or other secure storage mechanisms.
{% endhint %}

## Transaction Creation and Signing

```rust
// src/wallet.rs (lines 52-70)
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

### Deterministic Nonce Management

The wallet implements a simplified nonce system for transaction ordering:

| Feature | Implementation | Radio Benefits |
|---------|---------------|----------------|
| **Sequential Nonces** | Incremented for each transaction | Prevents replay attacks |
| **Persistent State** | Saved with wallet data | Survives node restarts |
| **Deterministic Ordering** | Enables consistent transaction processing | Critical for disconnected networks |

## Public Key Operations

### Address Generation

```rust
// src/wallet.rs (lines 25-30)
pub fn address_hex(&self) -> String {
    hex::encode(self.public_key.to_bytes())
}

pub fn address_bytes(&self) -> Vec<u8> {
    self.public_key.to_bytes().to_vec()
}
```

The wallet provides both hexadecimal and binary address formats:
- **Hex format:** Human-readable for CLI operations and radio transmission
- **Binary format:** Efficient for programmatic use and blockchain storage

## Integration with CLI Commands

### Wallet Command Implementations

```rust
// From main.rs - wallet command integration
Commands::Keygen => {
    let wallet = Wallet::generate();
    wallet.save(&data_dir)?;
    println!("New address: {}", wallet.address_hex());
}

Commands::Send { to, amount } => {
    let mut wallet = Wallet::load_or_generate(&data_dir)?;
    let recipient = hex::decode(to)?;
    let tx = wallet.create_tx(recipient, amount);
    // Transaction saved to mempool for broadcast
}
```

## Radio-Optimized Features

### Compact Key Format
- **32-byte public keys:** Ed25519 provides optimal size-to-security ratio
- **64-byte signatures:** Minimal signature overhead for radio transmission
- **Hex encoding:** Human-readable format suitable for voice transmission

### Offline Operation Capability
The wallet is designed for disconnected operation:
- **No network dependencies:** All operations work offline
- **File-based persistence:** Survives power failures and restarts  
- **Manual transaction export:** Transactions can be saved to files for radio transmission

## Error Handling and Recovery

### Graceful Degradation
```rust
// Robust wallet loading pattern
pub fn load_or_generate(data_dir: &Path) -> anyhow::Result<Self> {
    match Self::load(data_dir) {
        Ok(wallet) => Ok(wallet),
        Err(_) => {
            eprintln!("Could not load wallet, generating new one");
            Ok(Self::generate())
        }
    }
}
```

### Backup and Recovery Strategies
- **JSON export:** Wallet data can be manually copied for backup
- **Key derivation:** Future enhancement for BIP32-style HD wallets
- **Paper backup:** Hex-encoded private keys suitable for paper storage

## Security Model

### Cryptographic Properties
| Property | Implementation | Security Level |
|----------|---------------|----------------|
| **Private Key Security** | Ed25519 scalar (32 bytes) | 128-bit security equivalent |
| **Signature Algorithm** | EdDSA with Curve25519 | Post-quantum resistant candidate |
| **Random Number Generation** | OS entropy pool (OsRng) | Cryptographically secure |
| **Key Derivation** | Direct from secret scalar | No key stretching (future enhancement) |

### Threat Model Considerations
- **Physical access:** Private keys stored in plaintext on disk
- **Memory attacks:** Private keys held in process memory during operation
- **Side-channel attacks:** No specific protections against timing attacks
- **Quantum resistance:** Ed25519 provides some quantum resistance but not full protection

## Future Enhancements

### Planned Wallet Features
- **Hardware security module support:** Integration with HSMs for enhanced security
- **Multi-signature wallets:** m-of-n signature schemes for shared control
- **Hierarchical deterministic keys:** BIP32-style key derivation for better backup
- **Encrypted storage:** Password-protected wallet files

### Radio Integration Enhancements
- **QR code generation:** Visual encoding for air-gapped key transfer
- **Voice encoding:** Audio-friendly private key backup formats
- **Radio-specific addressing:** Ham radio callsign integration for addressing
- **Mesh network routing:** Wallet-aware routing for packet radio networks 