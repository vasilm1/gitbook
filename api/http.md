# HTTP Endpoints

RESTful API endpoints for blockchain interaction and real-time monitoring.

## Overview

The HTTP interface provides firewall-friendly access to blockchain operations via standard REST endpoints. All endpoints use JSON for data exchange and provide SSE for real-time updates.

**Base URL:** `http://localhost:10000`
**Content-Type:** `application/json`

## Endpoints

### GET /events
Server-Sent Events stream for real-time node monitoring

### GET /chain
Download complete blockchain data

### GET /sync
Lightweight synchronization status check

### POST /broadcast
Accept new blocks from peers

### GET /mempool
List pending transactions

### POST /tx
Accept single transaction from peer

_Detailed endpoint documentation coming soon..._ 