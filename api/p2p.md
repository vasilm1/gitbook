# P2P Protocol

Direct TCP communication protocol for efficient peer synchronization.

## Overview

The P2P protocol enables direct node-to-node communication via persistent TCP connections, complementing the HTTP interface with real-time block and transaction propagation.

**Default Port:** 7000
**Protocol:** TCP with JSON messaging
**Connection Model:** Persistent bidirectional

## Message Types

### GetChain
Request peer's complete blockchain

### Chain(Vec<Block>)
Response containing blockchain data

### GetMempool
Request peer's pending transactions

### Mempool(Vec<Transaction>)
Response containing mempool data

### NewBlock(Block)
Real-time block announcement

### NewTransaction(Transaction)
Real-time transaction propagation

_Detailed protocol documentation coming soon..._ 