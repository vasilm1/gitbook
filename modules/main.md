# main.rs - CLI Interface & Application Coordination

The main.rs module serves as the application entry point and command-line interface for Bunkercoin. It orchestrates all system components, handles user commands, and implements the HTTP-based peer synchronization protocol that enables radio-friendly blockchain networking.

## CLI Command Structure

```rust
// src/main.rs (lines 8-42)
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
    /// Path to local data directory (blockchain & wallet). Defaults to ./data
    #[arg(long, short)]
    data_dir: Option<PathBuf>,
}

#[derive(Subcommand)]
enum Commands {
    /// Generate new wallet keypair
    Keygen,
    /// Show wallet address & balance
    Balance,
    /// Send coins to recipient
    Send {
        /// Recipient public key (hex)
        to: String,
        /// Amount in satoshi-like units
        amount: u64,
    },
    /// Run full node (p2p + optional mining)
    Node {
        /// Enable mining loop (CPU-based)
        #[arg(long)]
        mine: bool,
        /// TCP port to listen on
        #[arg(long, default_value_t = 7000)]
        port: u16,
        /// Bootstrap peer list
        #[arg(long)]
        peers: Option<String>,
    },
}
```

{% hint style="info" %}
**Design Philosophy:** The CLI uses clap's derive macros for type-safe command parsing. Each command encapsulates a complete workflow, from wallet operations to full node deployment, making the system accessible to both developers and operators.
{% endhint %}

## Application Initialization

```rust
// src/main.rs (lines 44-50)
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let data_dir = cli.data_dir.unwrap_or_else(|| PathBuf::from("data"));
    std::fs::create_dir_all(&data_dir)?;

    match cli.command {
```

The main function demonstrates Bunkercoin's resilient design. The data directory is created automatically, and the application gracefully handles missing configuration through sensible defaults.

## Command Implementations

### Wallet Operations

```rust
// src/main.rs (lines 51-66)
Commands::Keygen => {
    let wallet = Wallet::generate();
    wallet.save(&data_dir)?;
    println!("New address: {}", wallet.address_hex());
}
Commands::Balance => {
    let wallet = Wallet::load_or_generate(&data_dir)?;
    println!("Address: {}", wallet.address_hex());
    // In this prototype we don't track UTXO; balance is placeholder
    println!("Balance tracking not yet implemented");
}
Commands::Send { to, amount } => {
    let mut wallet = Wallet::load_or_generate(&data_dir)?;
    let recipient = hex::decode(to)?;
    let tx = wallet.create_tx(recipient, amount);
    // Save to mempool folder for node to broadcast
    let mp_dir = data_dir.join("mempool");
    std::fs::create_dir_all(&mp_dir)?;
    let fname = mp_dir.join(format!("{}.tx", blake3::hash(&bincode::serialize(&tx)?)));
    std::fs::write(fname, bincode::serialize(&tx)?)?;
}
```

## Node Orchestration

```rust
// src/main.rs (lines 71-85)
Commands::Node { mine, port, peers } => {
    let chain = Blockchain::load_or_init(&data_dir)?;
    // Spawn auxiliary tasks
    let web_task = tokio::spawn(web::serve_http(data_dir.clone(), chain.clone()));

    // Spawn mempool watcher to import .tx files dropped by CLI 'send'
    let mp_chain = chain.clone();
    let mp_dir_clone = data_dir.clone();
    let mp_watcher = tokio::spawn(async move {
        use std::time::Duration;
        use std::fs;
        use bunkercoin::transaction::Transaction;
        loop {
            let mp_path = mp_dir_clone.join("mempool");
            if let Ok(entries) = fs::read_dir(&mp_path) {
                for entry in entries.filter_map(|e| e.ok()) {
```

{% hint style="success" %}
**Async Task Coordination:** The node command spawns multiple concurrent tasks using tokio::spawn, enabling simultaneous web serving, mining, peer synchronization, and mempool monitoring without blocking operations.
{% endhint %}

## HTTP Peer Synchronization

```rust
// src/main.rs (lines 155-185)
async fn sync_with_peer(blockchain: &bunkercoin::blockchain::Blockchain, peer_url: &str) -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();
    
    // Get peer's sync info
    let sync_url = format!("{}/sync", peer_url);
    let resp: bunkercoin::blockchain::SyncInfo = client.get(&sync_url).send().await?.json().await?;
    
    let our_info = blockchain.get_sync_info();
    
    // Always fetch peer's chain if they have blocks
    if resp.height > 0 {
        let chain_url = format!("{}/chain", peer_url);
        let peer_chain: Vec<bunkercoin::block::Block> = client.get(&chain_url).send().await?.json().await?;
        
        // Use longest chain rule
        if blockchain.replace_if_longer(peer_chain)? {
            println!("Adopted longer chain from {}", peer_url);
        }
    }

    // Fetch peer mempool
    let mempool_url = format!("{}/mempool", peer_url);
    if let Ok(peer_txs) = client.get(&mempool_url).send().await?.json::<Vec<bunkercoin::transaction::Transaction>>().await {
        for tx in peer_txs {
            blockchain.add_transaction(tx);
        }
    }
    
    // Push our mempool to peer
    let our_txs = blockchain.get_mempool();
    for tx in &our_txs {
        let _ = client.post(&format!("{}/tx", peer_url)).json(tx).send().await;
    }
    
    Ok(())
}
```

### Synchronization Protocol Breakdown

| Step | Operation | Purpose |
|------|-----------|---------|
| **1. Lightweight Status Check** | GET /sync returns only height and latest hash | Minimizes bandwidth for radio links while enabling quick chain comparison |
| **2. Chain Download** | GET /chain fetches complete blockchain if peer has more blocks | Uses longest chain rule for automatic conflict resolution |
| **3. Mempool Exchange** | Bidirectional transaction propagation via GET /mempool and POST /tx | Ensures all nodes have the same pending transactions |
| **4. Error Tolerance** | Failures logged but don't stop the sync loop | Critical for unreliable radio links with intermittent connectivity |

## Task Lifecycle Management

The main.rs module implements sophisticated task coordination for multi-threaded operation:

### Background Task Spawning

```rust
// Task coordination pattern
let web_task = tokio::spawn(web::serve_http(data_dir.clone(), chain.clone()));
let mining_task = if mine {
    Some(tokio::spawn(async move { chain_clone.mining_loop().await }))
} else { None };
let peer_sync_task = tokio::spawn(periodic_peer_sync(chain.clone(), peers));
```

### Graceful Shutdown

All tasks run indefinitely until the process receives a termination signal. The async runtime ensures proper cleanup of resources and network connections.

## Integration with Radio Systems

The main.rs architecture is specifically designed for radio deployment scenarios:

- **Fixed port 10000** for HTTP server simplifies firewall configuration
- **30-second peer sync interval** accommodates HF propagation delays
- **Atomic file operations** prevent corruption during power failures
- **HTTP-only networking** eliminates complex NAT traversal issues

## Error Handling Strategy

```rust
// Resilient error handling pattern throughout main.rs
if let Err(e) = sync_with_peer(&chain, peer_url).await {
    eprintln!("Sync failed with {}: {}", peer_url, e);
    // Continue with other peers rather than stopping
}
```

The system prioritizes availability over perfection, logging errors but continuing operation to maintain network connectivity even when individual components fail. 