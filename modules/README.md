# Bunkercoin Module Reference

Bunkercoin is architected as a modular system where each Rust module encapsulates specific functionality. This design enables independent development, testing, and potential replacement of components without affecting the overall system stability.

## Module Dependency Graph

```
Module Dependencies:

main.rs
├── blockchain.rs ──────┐
│   ├── block.rs        │
│   ├── transaction.rs  │
│   ├── vdf.rs          │
│   └── web.rs ─────────┼─── Global Event System
├── wallet.rs           │
│   └── transaction.rs  │
├── network.rs ─────────┤
│   ├── block.rs        │
│   └── transaction.rs  │
└── web.rs ─────────────┘
    ├── blockchain.rs
    ├── block.rs
    └── transaction.rs
```

## Core Modules

### [main.rs - Application Entry Point](main.md)
Command-line interface, application coordination, and peer synchronization logic. Handles all user interactions and orchestrates the various subsystems.
- **Tags:** CLI, Coordination, HTTP Sync
- **Size:** 210 lines

### [blockchain.rs - Core Blockchain Logic](blockchain.md)
Central blockchain state management, deterministic mining algorithm, transaction pool, and peer synchronization. The heart of the Bunkercoin system.
- **Tags:** Mining, State, Sync
- **Size:** 204 lines

### [web.rs - HTTP Interface & SSE](web.md)
HTTP server with RESTful API endpoints and Server-Sent Events for real-time monitoring. Enables firewall-friendly P2P communication.
- **Tags:** HTTP, SSE, API
- **Size:** 94 lines

### [wallet.rs - Key Management](wallet.md)
Ed25519 keypair generation, transaction signing, and wallet persistence. Handles all cryptographic operations for user transactions.
- **Tags:** Crypto, Ed25519, Signing
- **Size:** 70 lines

### [transaction.rs - Transaction Structure](transaction.md)
Transaction data structure, cryptographic signing, verification, and serialization. Defines the format for value transfers in the system.
- **Tags:** Structure, Verification, Hashing
- **Size:** 45 lines

### [block.rs - Block Structure](block.md)
Block and block header data structures with cryptographic hashing. Defines the fundamental unit of the blockchain.
- **Tags:** Block, Header, Blake3
- **Size:** 31 lines

### [network.rs - TCP P2P Protocol](network.md)
TCP-based peer-to-peer networking for direct node communication. Complements the HTTP interface for full mesh networking.
- **Tags:** TCP, P2P, Async
- **Size:** 89 lines

### [vdf.rs - Verifiable Delay Function](vdf.md)
Sequential proof-of-work computation using Blake3 iterations. Provides deterministic timing for radio-synchronized mining.
- **Tags:** VDF, PoW, Sequential
- **Size:** 11 lines

## Module Interaction Patterns

### Data Flow Patterns
- **Transaction Creation:** wallet.rs → transaction.rs → blockchain.rs
- **Block Mining:** blockchain.rs → vdf.rs → block.rs → web.rs
- **Network Sync:** main.rs → web.rs → blockchain.rs
- **P2P Communication:** network.rs ↔ blockchain.rs

### Event Broadcasting
- **Global Events:** web.rs provides system-wide event channel
- **Mining Updates:** blockchain.rs → web.rs → SSE clients
- **Transaction Events:** All modules → web.rs broadcast
- **Error Reporting:** Distributed logging via web.rs

## Architectural Principles

### Separation of Concerns
Each module has a clearly defined responsibility. Cryptographic operations are isolated in wallet.rs and transaction.rs, network communication is handled by web.rs and network.rs, and core logic resides in blockchain.rs.

### Minimal Dependencies
Cross-module dependencies are kept to the absolute minimum. The lib.rs file shows the complete module structure, and most modules only depend on shared data structures rather than complex functionality.

### Radio-Optimized Design
All modules are designed with radio communication constraints in mind. Data structures are compact, operations are deterministic, and timing is predictable to enable reliable HF radio propagation.

### Thread Safety
Shared state is protected by Arc<Mutex> wrappers, enabling safe concurrent access across async tasks. This allows the HTTP server, mining loop, and peer synchronization to operate simultaneously.

## Development Guidelines

### Adding New Features
- Follow existing module patterns
- Minimize cross-module dependencies
- Use web.rs for event broadcasting
- Maintain deterministic behavior

### Testing Strategy
- Unit test individual modules
- Integration test module interactions
- Verify radio-compatible behavior
- Test network partition scenarios 