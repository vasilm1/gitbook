# network.rs - TCP P2P Protocol

The network module implements direct TCP-based peer-to-peer communication for Bunkercoin, complementing the HTTP interface with real-time blockchain and transaction synchronization. This dual-protocol approach provides both firewall-friendly HTTP and efficient TCP connectivity.

## TCP Server Implementation

```rust
// src/network.rs (lines 7-25)
pub async fn start_tcp_server(port: u16, blockchain: Arc<Mutex<Vec<Block>>>) -> anyhow::Result<()> {
    let listener = TcpListener::bind(format!("0.0.0.0:{}", port)).await?;
    println!("TCP P2P server listening on port {}", port);
    
    loop {
        let (socket, addr) = listener.accept().await?;
        println!("New TCP connection from {}", addr);
        
        let blockchain_clone = Arc::clone(&blockchain);
        tokio::spawn(async move {
            if let Err(e) = handle_connection(socket, blockchain_clone).await {
                eprintln!("Connection error with {}: {}", addr, e);
            }
        });
    }
}
```

{% hint style="info" %}
**Async TCP Handling:** The server uses Tokio's async runtime to handle multiple concurrent connections efficiently. Each connection spawns an independent task, enabling simultaneous peer communication without blocking.
{% endhint %}

## Connection Protocol

### Message Format
```rust
// src/network.rs (lines 27-35)
#[derive(Serialize, Deserialize, Debug)]
enum P2PMessage {
    GetChain,
    Chain(Vec<Block>),
    GetMempool,
    Mempool(Vec<Transaction>),
    NewBlock(Block),
    NewTransaction(Transaction),
}
```

### Bidirectional Communication
```rust
// src/network.rs (lines 37-65)
async fn handle_connection(
    socket: TcpStream, 
    blockchain: Arc<Mutex<Vec<Block>>>
) -> anyhow::Result<()> {
    let mut lines = BufReader::new(socket).lines();
    
    while let Some(line) = lines.next_line().await? {
        let message: P2PMessage = serde_json::from_str(&line)?;
        
        match message {
            P2PMessage::GetChain => {
                let chain = blockchain.lock().unwrap().clone();
                let response = P2PMessage::Chain(chain);
                // Send response back to peer
            }
            P2PMessage::NewBlock(block) => {
                // Validate and add block to local chain
                blockchain.lock().unwrap().push(block);
            }
            // Handle other message types...
        }
    }
    Ok(())
}
```

## Peer Connection Management

### Outbound Connection Establishment
```rust
// src/network.rs (lines 67-82)
pub async fn connect_to_peer(
    addr: &str, 
    blockchain: Arc<Mutex<Vec<Block>>>
) -> anyhow::Result<()> {
    let stream = TcpStream::connect(addr).await?;
    let mut writer = BufWriter::new(stream);
    
    // Request peer's blockchain
    let request = P2PMessage::GetChain;
    let json = serde_json::to_string(&request)?;
    writer.write_all(json.as_bytes()).await?;
    writer.write_all(b"\n").await?;
    writer.flush().await?;
    
    Ok(())
}
```

### Connection Lifecycle
| Phase | Operation | Purpose |
|-------|-----------|---------|
| **Discovery** | Connect to known peer addresses | Establish network topology |
| **Handshake** | Exchange protocol version info | Ensure compatibility |
| **Synchronization** | Request chain and mempool data | Achieve network consensus |
| **Real-time Updates** | Push new blocks/transactions | Maintain network coherence |

## Message Types and Protocols

### Blockchain Synchronization
```rust
// Chain synchronization protocol
P2PMessage::GetChain => {
    // Peer requests our blockchain
    let chain = blockchain.lock().unwrap().clone();
    send_message(P2PMessage::Chain(chain)).await?;
}

P2PMessage::Chain(peer_chain) => {
    // Received peer's blockchain
    let mut local_chain = blockchain.lock().unwrap();
    if peer_chain.len() > local_chain.len() {
        *local_chain = peer_chain; // Adopt longer chain
    }
}
```

### Transaction Broadcasting
```rust
// Transaction propagation protocol
P2PMessage::NewTransaction(tx) => {
    // Validate transaction
    if tx.verify() {
        // Add to local mempool
        mempool.lock().unwrap().push(tx.clone());
        
        // Broadcast to other peers
        broadcast_to_peers(P2PMessage::NewTransaction(tx)).await?;
    }
}
```

## Integration with HTTP Interface

The TCP P2P system complements the HTTP interface:

### Protocol Comparison
| Feature | TCP P2P | HTTP Interface |
|---------|---------|----------------|
| **Connection Model** | Persistent connections | Stateless requests |
| **Firewall Traversal** | Requires port forwarding | Works through NAT/firewalls |
| **Real-time Updates** | Push notifications | Polling required |
| **Resource Usage** | Lower latency | Higher compatibility |
| **Radio Suitability** | Requires stable connections | Better for intermittent links |

### Dual Protocol Benefits
- **HTTP for discovery:** Initial peer contact and firewall traversal
- **TCP for streaming:** Real-time block and transaction propagation
- **Fallback capability:** Either protocol can handle full synchronization
- **Flexibility:** Nodes can use optimal protocol for their environment

## Error Handling and Resilience

### Connection Failure Recovery
```rust
// Robust peer connection management
async fn maintain_peer_connections(peers: Vec<String>) {
    loop {
        for peer in &peers {
            if let Err(e) = connect_to_peer(peer).await {
                eprintln!("Failed to connect to {}: {}", peer, e);
                // Continue with other peers rather than failing
            }
        }
        
        // Wait before retry cycle
        tokio::time::sleep(Duration::from_secs(30)).await;
    }
}
```

### Message Validation
```rust
// Comprehensive message validation
fn validate_p2p_message(msg: &P2PMessage) -> Result<(), ValidationError> {
    match msg {
        P2PMessage::NewBlock(block) => {
            // Validate block structure and cryptography
            validate_block(block)?;
        }
        P2PMessage::NewTransaction(tx) => {
            // Verify transaction signature
            if !tx.verify() {
                return Err(ValidationError::InvalidSignature);
            }
        }
        // Other validations...
    }
    Ok(())
}
```

## Radio Environment Considerations

### TCP Challenges in Radio Networks
- **Connection stability:** Radio links may have frequent disconnections
- **NAT traversal:** Radio networks often use complex routing
- **Bandwidth limitations:** TCP overhead may be significant on slow links
- **Latency variation:** HF propagation introduces variable delays

### Optimization Strategies
```rust
// Radio-aware TCP configuration
fn configure_tcp_for_radio(socket: &TcpStream) -> std::io::Result<()> {
    // Enable keep-alive for connection monitoring
    socket.set_keepalive(Some(Duration::from_secs(30)))?;
    
    // Reduce buffer sizes for low-bandwidth links
    socket.set_recv_buffer_size(4096)?;
    socket.set_send_buffer_size(4096)?;
    
    // Disable Nagle's algorithm for low-latency
    socket.set_nodelay(true)?;
    
    Ok(())
}
```

## Performance and Scalability

### Concurrent Connection Handling
The network module uses async/await patterns for efficient resource utilization:

```rust
// Efficient concurrent processing
async fn process_multiple_peers(peer_addrs: Vec<String>) {
    let mut handles = Vec::new();
    
    for addr in peer_addrs {
        let handle = tokio::spawn(async move {
            connect_and_sync(addr).await
        });
        handles.push(handle);
    }
    
    // Wait for all connections to complete
    for handle in handles {
        let _ = handle.await;
    }
}
```

### Memory Management
- **Streaming parsing:** JSON messages parsed incrementally
- **Connection pooling:** Reuse connections when possible
- **Bounded buffers:** Prevent memory exhaustion from malicious peers

## Future Enhancements

### Planned Network Features
- **Peer discovery protocol:** Automatic peer finding and topology mapping
- **Connection encryption:** TLS/SSL support for secure communication
- **Bandwidth adaptation:** Dynamic protocol selection based on link quality
- **Mesh routing:** Multi-hop communication for radio networks

### Radio-Specific Protocols
- **AX.25 integration:** Native packet radio protocol support
- **VARA modem support:** Integration with VARA HF/FM modems
- **JS8Call messaging:** Direct integration with JS8Call for HF communication
- **APRS gateway:** Bridge to amateur radio APRS networks 