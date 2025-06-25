# Deployment Guide

Complete guide for deploying BunkerChain in various environments, from local development to production radio networks. This documentation covers setup, configuration, and operational considerations for different deployment scenarios.

{% hint style="warning" %}
**Experimental Software:** BunkerChain is experimental blockchain technology. Do not use for production financial systems or critical infrastructure. Intended for research, education, and amateur radio experimentation only.
{% endhint %}

## Local Setup

### Prerequisites

**System Requirements:**

* **Operating System:** Linux, macOS, or Windows 10+
* **Memory:** 512MB RAM minimum, 2GB recommended
* **Storage:** 100MB for blockchain data (grows over time)
* **Network:** Internet connection for peer synchronization

**Development Dependencies:**

```bash
# Rust toolchain (required)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Additional tools (optional)
sudo apt-get install git curl wget  # Ubuntu/Debian
brew install git curl wget          # macOS
```

### Quick Start

```bash
# Generate wallet
bunkercoin keygen

# Start mining node
bunkercoin node --mine

# In another terminal, send test transaction
bunkercoin send <recipient-pubkey> 1000

# Monitor node activity
curl http://localhost:10000/events
```

## Configuration

### Data Directory Structure

```
data/
├── wallet.json         # Ed25519 keypair and nonce state
├── chain.json          # Complete blockchain data
└── mempool/            # Pending transactions
    ├── tx1.tx
    ├── tx2.tx
    └── ...
```

### Configuration Options

**CLI Arguments:**

```bash
bunkercoin node \
    --mine                    # Enable mining
    --port 7001              # TCP P2P port (default: 7000)
    --peers "peer1,peer2"    # Comma-separated peer list
    --data-dir ./custom-data # Custom data directory
```

**Environment Variables:**

```bash
export BUNKERCOIN_DATA_DIR=/opt/bunkercoin/data
export BUNKERCOIN_LOG_LEVEL=debug
export BUNKERCOIN_PEERS="peer1.example.com,peer2.example.com"
```

## HF Radio Bridge

### Radio Interface Setup

BunkerChain is specifically designed for HF radio deployment using digital modes:

#### JS8Call Integration

```bash
# Configure JS8Call for blockchain relay
# 1. Set up JS8Call with your radio
# 2. Configure automated messaging
# 3. Create blockchain relay scripts

#!/bin/bash
# js8-relay.sh - Automated blockchain-to-JS8Call bridge
while true; do
    # Get latest block
    BLOCK=$(curl -s http://localhost:10000/chain | jq '.[-1]')
    
    # Fragment into JS8Call messages
    echo "$BLOCK" | split -b 200 - block_fragment_
    
    # Transmit fragments via JS8Call
    for fragment in block_fragment_*; do
        js8call_send "BLOCKCHAIN:$(cat $fragment)"
        sleep 30  # Respect band usage
    done
    
    sleep 3600  # Hourly blockchain sync
done
```

#### VARA Modem Support

```python
# Python bridge for VARA HF modems
import requests
import socket
import json

def vara_blockchain_sync():
    # Connect to VARA modem
    vara_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    vara_socket.connect(('localhost', 8300))  # VARA port
    
    # Get blockchain sync data
    response = requests.get('http://localhost:10000/sync')
    sync_data = response.json()
    
    # Transmit via VARA
    message = f"BUNKERCOIN SYNC {sync_data['height']} {sync_data['latest_hash']}"
    vara_socket.send(message.encode())
    
if __name__ == "__main__":
    vara_blockchain_sync()
```

### Radio Network Topology

```
        [Internet Backbone]
               |
         [Gateway Station]
               |
    ┌─────────┼─────────┐
    │         │         │
[HF Radio] [VHF Net] [Mesh Node]
    │         │         │
[Mobile]  [Repeater] [Emergency]
[Station]   [Site]    [Shelter]
```

**Station Roles:**

* **Gateway:** Internet-connected station providing network bridge
* **Relay:** Packet forwarding between radio segments
* **End Node:** Transaction origination and wallet operations

### Radio Protocol Considerations

#### Frequency Planning

```bash
# Example frequency coordination
# 40m: 7.035-7.040 MHz (JS8Call blockchain relay)
# 20m: 14.070-14.073 MHz (Primary network)
# 15m: 21.070-21.073 MHz (Backup network)
# 2m:  144.390 MHz (Local APRS gateway)
```

#### Timing Synchronization

```bash
# GPS-disciplined frequency standards recommended
# Block timing: 30-second intervals aligned to UTC
# Transmission windows: :00, :30 seconds past minute
# Emergency override: Break into schedule for urgent transactions
```

## Cloud Deployment

### Docker Container

```dockerfile
# Dockerfile
FROM rust:1.70 as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bullseye-slim
WORKDIR /app
COPY --from=builder /app/target/release/bunkercoin /usr/local/bin/
EXPOSE 10000 7000
CMD ["bunkercoin", "node", "--mine", "--peers", "$BUNKERCOIN_PEERS"]
```

**Docker Compose:**

```yaml
# docker-compose.yml
version: '3.8'
services:
  bunkercoin:
    build: .
    ports:
      - "10000:10000"  # HTTP interface
      - "7000:7000"    # P2P protocol
    volumes:
      - bunkercoin_data:/app/data
    environment:
      - BUNKERCOIN_PEERS=chain.bunkerchain.dev
    restart: unless-stopped

volumes:
  bunkercoin_data:
```

### Cloud Provider Configuration

#### AWS Deployment

```bash
# EC2 instance with security groups
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --instance-type t3.micro \
    --security-group-ids sg-bunkercoin \
    --user-data file://bunkercoin-userdata.sh

# Security group rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-bunkercoin \
    --protocol tcp \
    --port 10000 \
    --cidr 0.0.0.0/0
```

#### Render.com Deployment

```yaml
# render.yaml
services:
  - type: web
    name: bunkercoin-node
    env: docker
    dockerfilePath: ./Dockerfile
    plan: starter
    envVars:
      - key: BUNKERCOIN_PEERS
        value: "peer1.example.com,peer2.example.com"
```

### Load Balancing and High Availability

```nginx
# nginx.conf - Load balancer for multiple nodes
upstream bunkercoin_nodes {
    server 10.0.1.10:10000;
    server 10.0.1.11:10000;
    server 10.0.1.12:10000;
}

server {
    listen 80;
    server_name blockchain.example.com;
    
    location / {
        proxy_pass http://bunkercoin_nodes;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /events {
        proxy_pass http://bunkercoin_nodes;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

## Network Monitoring

### Health Check Scripts

```bash
#!/bin/bash
# health-check.sh - Monitor node status
NODE_URL="http://localhost:10000"

# Check if node is responding
if ! curl -s "$NODE_URL/sync" > /dev/null; then
    echo "ERROR: Node not responding"
    exit 1
fi

# Check blockchain height
HEIGHT=$(curl -s "$NODE_URL/sync" | jq '.height')
if [ "$HEIGHT" -lt 1 ]; then
    echo "WARNING: No blocks in chain"
fi

# Check peer connectivity
PEERS=$(curl -s "$NODE_URL/mempool" | jq 'length')
echo "Active mempool transactions: $PEERS"

echo "Node healthy at height $HEIGHT"
```

### Prometheus Metrics

```python
# metrics.py - Export Prometheus metrics
from prometheus_client import start_http_server, Gauge, Counter
import requests
import time

# Define metrics
blockchain_height = Gauge('bunkercoin_blockchain_height', 'Current blockchain height')
mempool_size = Gauge('bunkercoin_mempool_size', 'Number of pending transactions')
node_uptime = Gauge('bunkercoin_node_uptime_seconds', 'Node uptime in seconds')

def collect_metrics():
    try:
        # Get node status
        sync_response = requests.get('http://localhost:10000/sync')
        sync_data = sync_response.json()
        blockchain_height.set(sync_data['height'])
        
        # Get mempool status
        mempool_response = requests.get('http://localhost:10000/mempool')
        mempool_data = mempool_response.json()
        mempool_size.set(len(mempool_data))
        
    except Exception as e:
        print(f"Error collecting metrics: {e}")

if __name__ == '__main__':
    start_http_server(8000)  # Prometheus metrics port
    while True:
        collect_metrics()
        time.sleep(30)
```

## Security Hardening

### System Security

```bash
# Firewall configuration
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 10000/tcp  # HTTP interface
sudo ufw allow 7000/tcp   # P2P protocol
sudo ufw enable

# User isolation
sudo useradd -r -s /bin/false bunkercoin
sudo mkdir -p /opt/bunkercoin/data
sudo chown bunkercoin:bunkercoin /opt/bunkercoin
```

### Service Configuration

```ini
# /etc/systemd/system/bunkercoin.service
[Unit]
Description=Bunkercoin Node
After=network.target

[Service]
Type=simple
User=bunkercoin
WorkingDirectory=/opt/bunkercoin
ExecStart=/usr/local/bin/bunkercoin node --mine --data-dir /opt/bunkercoin/data
Restart=always
RestartSec=10

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/bunkercoin/data

[Install]
WantedBy=multi-user.target
```

### Backup and Recovery

```bash
#!/bin/bash
# backup.sh - Automated backup script
BACKUP_DIR="/backup/bunkercoin/$(date +%Y%m%d)"
DATA_DIR="/opt/bunkercoin/data"

mkdir -p "$BACKUP_DIR"

# Backup critical files
cp "$DATA_DIR/wallet.json" "$BACKUP_DIR/"
cp "$DATA_DIR/chain.json" "$BACKUP_DIR/"

# Create compressed archive
tar -czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"

# Upload to cloud storage (optional)
# aws s3 cp "$BACKUP_DIR.tar.gz" s3://bunkercoin-backups/

echo "Backup completed: $BACKUP_DIR.tar.gz"
```

## Troubleshooting

### Common Issues

**Node Won't Start:**

```bash
# Check data directory permissions
ls -la data/
sudo chown -R $USER:$USER data/

# Verify binary installation
which bunkercoin
bunkercoin --version
```

**Peer Connection Problems:**

```bash
# Test peer connectivity
curl http://peer.example.com:10000/sync

# Check local firewall
sudo ufw status
netstat -tlnp | grep 10000
```

**Blockchain Sync Issues:**

```bash
# Reset blockchain (CAUTION: loses local data)
rm data/chain.json
bunkercoin node --peers "trusted-peer.com"

# Manual peer sync
curl http://trusted-peer.com:10000/chain > data/chain.json
```

### Logging and Diagnostics

```bash
# Enable debug logging
RUST_LOG=debug bunkercoin node --mine

# Monitor real-time events
curl http://localhost:10000/events

# Analyze blockchain data
jq '.[-1]' data/chain.json  # Latest block
jq 'length' data/chain.json # Chain height
```

## Production Considerations

### Performance Tuning

```bash
# System limits
echo "bunkercoin soft nofile 4096" >> /etc/security/limits.conf
echo "bunkercoin hard nofile 8192" >> /etc/security/limits.conf

# Kernel parameters
echo "net.core.somaxconn = 1024" >> /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 1024" >> /etc/sysctl.conf
sysctl -p
```

### Monitoring and Alerting

```bash
# Disk space monitoring
df -h /opt/bunkercoin/data

# Process monitoring
ps aux | grep bunkercoin
systemctl status bunkercoin

# Network monitoring
ss -tulpn | grep -E ':(7000|10000)'
```

### Maintenance Procedures

```bash
# Graceful shutdown
systemctl stop bunkercoin

# Update binary
cp new-bunkercoin /usr/local/bin/
systemctl start bunkercoin

# Validate upgrade
curl http://localhost:10000/sync
```
