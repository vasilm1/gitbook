# API Overview

Complete documentation for Bunkercoin's HTTP, P2P, and CLI interfaces. The API is designed for both programmatic integration and manual operation in radio-constrained environments.

## Interface Overview

Bunkercoin provides three primary interfaces for interaction:

### [HTTP Endpoints](http.md)

RESTful API for network communication and real-time monitoring

* **Port:** 10000 (fixed)
* **Protocol:** HTTP/1.1 (no TLS required)
* **Format:** JSON request/response
* **Features:** Server-Sent Events for real-time updates

### [P2P Protocol](p2p.md)

Direct TCP communication for efficient peer synchronization

* **Port:** 7000 (configurable)
* **Protocol:** TCP with JSON messaging
* **Features:** Persistent connections, real-time block propagation

### [CLI Commands](cli.md)

Command-line interface for local operations and node management

* **Binary:** `bunkercoin`
* **Configuration:** File-based with sensible defaults
* **Features:** Wallet management, node operation, transaction creation

## Quick Start Examples

### Start a Mining Node

```bash
# Start mining with HTTP interface
bunkercoin node --mine --peers chain.bunkerchain.dev

# Mining with custom port and peers
bunkercoin node --mine --port 7001 --peers "peer1.example.com,peer2.example.com"
```

### Create and Send Transaction

```bash
# Generate wallet
bunkercoin keygen

# Send transaction
bunkercoin send a1b2c3d4e5f6... 1000
```

### Monitor Node via HTTP

```bash
# Check node status
curl http://localhost:10000/sync

# Get blockchain data
curl http://localhost:10000/chain

# Monitor real-time events
curl http://localhost:10000/events
```

## Radio-Optimized Design

All APIs are designed with radio constraints in mind:

### Bandwidth Efficiency

* **Compact JSON:** Minimal overhead in all responses
* **Incremental sync:** Only transfer necessary data
* **Binary alternatives:** Bincode serialization available

### Intermittent Connectivity

* **Stateless HTTP:** No persistent session requirements
* **Atomic operations:** All API calls complete independently
* **Graceful degradation:** Partial failures don't affect system

### Manual Operation

* **Human-readable formats:** JSON and hex encoding throughout
* **Voice-friendly:** All data can be transmitted verbally if needed
* **Error tolerance:** Robust handling of malformed requests

## Integration Patterns

### Blockchain Explorer

```javascript
// Real-time blockchain monitoring
const eventSource = new EventSource('http://node:10000/events');
eventSource.onmessage = function(event) {
    console.log('Node event:', event.data);
};

// Fetch current chain state
fetch('http://node:10000/sync')
    .then(response => response.json())
    .then(data => console.log('Chain height:', data.height));
```

### Radio Bridge Integration

```python
# Python script for radio transmission
import requests
import json

# Get pending transactions for radio broadcast
response = requests.get('http://localhost:10000/mempool')
transactions = response.json()

for tx in transactions:
    # Convert to compact format for radio
    radio_data = json.dumps(tx, separators=(',', ':'))
    print(f"TX for radio: {radio_data}")
```

### Peer Discovery

```bash
# Discovery script for network peers
#!/bin/bash
PEERS="manhattanproject.onrender.com peer1.local peer2.local"

for peer in $PEERS; do
    echo "Checking peer: $peer"
    curl -s "http://$peer:10000/sync" || echo "Peer unavailable"
done
```

## Error Handling

All APIs use consistent error handling patterns:

### HTTP Status Codes

| Code    | Meaning      | Usage                        |
| ------- | ------------ | ---------------------------- |
| **200** | Success      | Normal operation             |
| **400** | Bad Request  | Invalid transaction or block |
| **404** | Not Found    | Unknown endpoint             |
| **500** | Server Error | Internal node error          |

### Error Response Format

```json
{
    "error": "Invalid transaction signature",
    "code": "INVALID_SIGNATURE",
    "details": {
        "transaction_hash": "a1b2c3...",
        "expected_sender": "d4e5f6..."
    }
}
```

### CLI Error Handling

* **Exit codes:** Standard Unix exit codes for script integration
* **Error messages:** Human-readable descriptions
* **Graceful degradation:** Partial failures don't stop execution

## Security Considerations

### HTTP Interface Security

* **No authentication:** All endpoints are public (suitable for radio networks)
* **Rate limiting:** Not implemented (trusted network assumption)
* **Input validation:** Cryptographic verification of all blockchain data

### P2P Protocol Security

* **Message validation:** All peer messages cryptographically verified
* **Connection limits:** Bounded number of concurrent connections
* **Malicious peer protection:** Invalid data discarded without affecting node

### CLI Security

* **Local access only:** CLI operates on local data directory
* **File permissions:** Wallet files use standard Unix permissions
* **Private key handling:** Keys stored in plaintext (operator responsibility)

## Performance Characteristics

### HTTP Interface

* **Concurrent requests:** Handles multiple simultaneous connections
* **Memory usage:** Bounded by blockchain size (grows linearly)
* **Response time:** Sub-second for all standard operations

### P2P Protocol

* **Connection overhead:** \~1KB per active peer connection
* **Throughput:** Limited by network bandwidth, not protocol
* **Latency:** Direct TCP provides minimal protocol overhead

### CLI Operations

* **Startup time:** \~100ms for simple operations
* **Disk I/O:** Atomic file operations for safety
* **Memory footprint:** <10MB for typical operations

## Future API Enhancements

### Planned Features

* **WebSocket support:** Real-time bidirectional communication
* **GraphQL endpoint:** Complex queries and subscriptions
* **gRPC interface:** High-performance binary protocol
* **OpenAPI specification:** Formal API documentation

### Radio-Specific Enhancements

* **Compression support:** gzip/deflate for bandwidth conservation
* **Batch operations:** Multiple transactions in single request
* **Delta synchronization:** Incremental blockchain updates
* **Voice encoding:** Audio-friendly data representation
