# web.rs - HTTP Interface & Server-Sent Events

The web module provides the HTTP interface layer for Bunkercoin, implementing both RESTful endpoints for peer-to-peer communication and real-time event streaming via Server-Sent Events (SSE). This design enables firewall-friendly networking and live monitoring capabilities essential for radio-based deployments.

## Global Event Broadcasting System

```rust
// src/web.rs (lines 10-22)
// Global broadcast channel for log lines / events
pub static EVENT_TX: Lazy<Sender<String>> = Lazy::new(|| {
    let (tx, _rx) = channel(100);
    tx
});

/// Broadcast helper prints to stdout and pushes to SSE listeners
pub fn broadcast<S: Into<String>>(msg: S) {
    let text: String = msg.into();
    println!("{}", text);
    let _ = EVENT_TX.send(text);
}
```

{% hint style="info" %}
**Event Architecture:** The system uses a global broadcast channel that simultaneously outputs to stdout and pushes events to all connected SSE clients. This dual approach ensures both console logging and real-time web monitoring.
{% endhint %}

## Server-Sent Events Implementation

```rust
// src/web.rs (lines 25-33)
// /events
let sse_route = warp::path("events").and(warp::get()).map(|| {
    let rx = EVENT_TX.subscribe();
    let stream = BroadcastStream::new(rx)
        .filter_map(|result| async move { result.ok() })
        .map(|msg| Ok::<Event, Infallible>(Event::default().data(msg)));
    warp::sse::reply(warp::sse::keep_alive().stream(stream))
});
```

The SSE endpoint creates a new subscription to the global broadcast channel for each client connection. The stream filtering ensures only successful messages are forwarded, with automatic keep-alive to maintain connections through firewalls and proxies.

## P2P Synchronization Endpoints

### Chain Data Access

```rust
// src/web.rs (lines 35-37)
// /chain – serve snapshot file directly
let chain_file = data_dir.join("chain.json");
let chain_route = warp::path("chain").and(warp::get()).and(warp::fs::file(chain_file));
```

The chain endpoint serves the blockchain data directly from the filesystem. This approach eliminates memory overhead and provides atomic consistency since the JSON file is only updated after successful block validation.

### Peer Synchronization Info

```rust
// src/web.rs (lines 39-46)
// /sync – return latest block info for peers to check
let blockchain_sync = blockchain.clone();
let sync_route = warp::path("sync")
    .and(warp::get())
    .map(move || {
        let bc = blockchain_sync.clone();
        let info = bc.get_sync_info();
        warp::reply::json(&info)
    });
```

{% hint style="success" %}
**Lightweight Synchronization:** The sync endpoint returns only essential metadata (height and latest hash), allowing peers to quickly determine if they need to fetch the full chain without transferring large amounts of data.
{% endhint %}

## Block Broadcasting Protocol

```rust
// src/web.rs (lines 48-56)
// /broadcast – accept new blocks from peers
let blockchain_broadcast = blockchain.clone();
let broadcast_route = warp::path("broadcast")
    .and(warp::post())
    .and(warp::body::json())
    .map(move |block: Block| {
        let bc = blockchain_broadcast.clone();
        match bc.add_block(block) {
            Ok(_) => warp::reply::with_status("OK", warp::http::StatusCode::OK),
            Err(_) => warp::reply::with_status("Invalid block", warp::http::StatusCode::BAD_REQUEST),
        }
    });
```

## Transaction Processing

### Mempool Inspection

```rust
// src/web.rs (lines 58-65)
// /mempool – list current pending transactions
let blockchain_mempool = blockchain.clone();
let mempool_route = warp::path("mempool")
    .and(warp::get())
    .map(move || {
        let bc = blockchain_mempool.clone();
        let txs = bc.get_mempool();
        warp::reply::json(&txs)
    });
```

### Transaction Submission

```rust
// src/web.rs (lines 67-75)
// /tx – accept single transaction from peer
let blockchain_tx = blockchain.clone();
let tx_route = warp::path("tx")
    .and(warp::post())
    .and(warp::body::json())
    .map(move |tx: Transaction| {
        let bc = blockchain_tx.clone();
        bc.add_transaction(tx);
        warp::reply::with_status("OK", warp::http::StatusCode::OK)
    });
```

## Server Configuration

```rust
// src/web.rs (lines 77-85)
let routes = sse_route
    .or(chain_route)
    .or(sync_route)
    .or(broadcast_route)
    .or(mempool_route)
    .or(tx_route);

// Force port to 10000
let port: u16 = 10000;
println!("Web interface listening on :{}", port);

warp::serve(routes).run(([0, 0, 0, 0], port)).await;
```

{% hint style="warning" %}
**Fixed Port Strategy:** Hardcoding port 10000 simplifies configuration and enables predictable peer discovery in radio environments where dynamic port allocation would be problematic.
{% endhint %}

## Complete API Reference

### GET /events
- **Purpose:** Real-time event streaming via Server-Sent Events
- **Response:** text/event-stream
- **Use Case:** Provides live updates for mining progress, transaction processing, and peer synchronization events. Essential for monitoring radio-bridge operations.

### GET /chain
- **Purpose:** Download complete blockchain data
- **Response:** JSON array of blocks
- **Use Case:** Peer synchronization - allows nodes to fetch the entire chain when joining the network or recovering from partitions.

### GET /sync
- **Purpose:** Lightweight synchronization status
- **Response:** `{"height": number, "latest_hash": "hex"}`
- **Use Case:** Quick check to determine if peer has newer blocks without downloading the full chain.

### POST /broadcast
- **Purpose:** Accept new blocks from peers
- **Request Body:** Single Block object in JSON
- **Use Case:** Network-wide block propagation when a node successfully mines a new block.

### GET /mempool
- **Purpose:** List pending transactions
- **Response:** JSON array of transactions
- **Use Case:** Peer synchronization of pending transactions to ensure all nodes have the same transaction pool.

### POST /tx
- **Purpose:** Accept single transaction from peer
- **Request Body:** Single Transaction object in JSON
- **Use Case:** Transaction propagation throughout the network before block inclusion.

## Radio-Optimized Design Features

### HTTP-Only Protocol
- **Firewall Friendly:** HTTP traverses NAT and firewalls without configuration
- **Stateless:** No persistent connections that can break during radio blackouts
- **Simple:** Minimal protocol overhead for bandwidth-constrained radio links

### Event Broadcasting for Monitoring
The SSE system enables real-time monitoring of radio bridge operations:

```javascript
// Client-side monitoring
const eventSource = new EventSource('http://node:10000/events');
eventSource.onmessage = function(event) {
    console.log('Node status:', event.data);
};
```

### Atomic Operations
All endpoints are designed for atomic operations that can safely handle connection failures:
- GET operations are idempotent
- POST operations either succeed completely or fail cleanly
- No partial state updates that could corrupt the blockchain

## Integration with Blockchain Core

The web module acts as the network interface for the blockchain:

```rust
// Integration pattern throughout web.rs
let blockchain_clone = blockchain.clone();
let route = warp::path("endpoint")
    .map(move || {
        let bc = blockchain_clone.clone();
        // Perform blockchain operation
        bc.some_operation()
    });
```

This pattern ensures:
- **Thread Safety:** All blockchain access goes through Arc<Mutex>
- **Performance:** No blocking of HTTP responses during blockchain operations
- **Reliability:** Failed operations don't affect the web server 