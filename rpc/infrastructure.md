# RPC Module Infrastructure Documentation

## Overview

The RPC module deploys and manages blockchain Remote Procedure Call (RPC) nodes that serve as the primary interface for blockchain interactions. This module provides externally-accessible endpoints for transaction submission, blockchain queries, and network synchronization, supporting both Ethereum execution layer and consensus layer protocols.

## Architecture

### Core Components
- **EC2 Instance**: High-performance Ubuntu server for blockchain node
- **Elastic IP**: Static public IP for external accessibility
- **Security Groups**: Controlled network access for P2P and RPC protocols
- **Storage**: High-capacity EBS volume for blockchain data
- **IAM Profile**: Systems Manager integration for management

### System Architecture
```
                    Internet
                        │
                        ↓
                ┌──────────────┐
                │  Elastic IP  │
                │ (Public IP)  │
                └──────────────┘
                        │
                ┌──────────────┐
                │  RPC Node    │
                │  (c5.2xlarge)│
                ├──────────────┤
                │ Geth Client  │ :30303 (P2P)
                │ Consensus    │ :26656 (P2P)
                │ JSON-RPC API │ :8545  (RPC)
                │ WebSocket    │ :8546  (WS)
                └──────────────┘
                        │
                   Internal VPC
                        │
                ┌──────────────┐
                │   Relayer    │
                └──────────────┘
```

## EC2 Instance Configuration

### Instance Specifications
| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Instance Type** | `c5.2xlarge` | 8 vCPUs, 16 GiB RAM - Required for blockchain sync |
| **AMI** | Ubuntu 22.04 LTS | Stable platform for blockchain clients |
| **Architecture** | x86_64 | Standard architecture |
| **Instance Name** | `rpc-goat` | Production RPC identifier |
| **Tenancy** | Default | Standard shared hardware |

### Performance Characteristics
| Resource | Specification | Purpose |
|----------|--------------|---------|
| **vCPUs** | 8 cores | Parallel processing for blockchain operations |
| **Memory** | 16 GiB | State caching and transaction pool |
| **Network** | Up to 10 Gbps | High-throughput P2P communication |
| **EBS Bandwidth** | Up to 4,750 Mbps | Fast blockchain data access |

### Storage Configuration
| Component | Specification | Purpose |
|-----------|--------------|---------|
| **Root Volume** | 100 GB GP3 | OS and blockchain software |
| **Volume Type** | GP3 | Balance of performance and cost |
| **IOPS** | 3000 (baseline) | Adequate for blockchain I/O |
| **Throughput** | 125 MB/s | Standard GP3 throughput |
| **Delete on Termination** | Yes | Automatic cleanup |
| **EBS Optimized** | Disabled | Consider enabling for production |

### Storage Capacity Planning
```
Blockchain Data Growth Estimates:
├── Ethereum Full Node: ~1 TB (current) + ~50 GB/month
├── Ethereum Archive Node: ~15 TB (current) + ~300 GB/month
├── Application Logs: ~10 GB/month
└── System Overhead: ~20 GB
```

## Elastic IP Configuration

### EIP Allocation
| Parameter | Value | Purpose |
|-----------|-------|---------|
| **Domain** | VPC | VPC-scoped allocation |
| **Name Tag** | `rpc-goat-eip` | Resource identification |
| **Association** | `rpc-goat` instance | Static IP binding |

### EIP Association Benefits
1. **Stable Endpoint**: Consistent IP for external clients
2. **DNS Mapping**: Reliable A record configuration
3. **Failover Support**: Re-associate to replacement instance
4. **Whitelist Friendly**: Static IP for firewall rules

### EIP Management
```hcl
resource "aws_eip" "rpc-goat-eip" {
    domain = "vpc"
    instance = aws_instance.rpc-goat.id
    tags = {
        Name = "rpc-goat-eip"
    }
}

resource "aws_eip_association" "rpc-goat-eip-assoc" {
    allocation_id = aws_eip.rpc-goat-eip.id
    instance_id = aws_instance.rpc-goat.id
}
```

## Security Group Configuration

### Security Group: `rpc-sg`

#### Inbound Rules

| Port | Protocol | Source | Service | Purpose |
|------|----------|--------|---------|---------|
| 30303 | TCP | 0.0.0.0/0 | Geth P2P | Ethereum node discovery and sync |
| 30303 | UDP | 0.0.0.0/0 | Geth Discovery | DHT-based peer discovery |
| 26656 | TCP | 0.0.0.0/0 | Consensus P2P | Tendermint/Cosmos consensus layer |

#### Outbound Rules

| Port | Protocol | Destination | Purpose |
|------|----------|------------|---------|
| All | All | 0.0.0.0/0 | Unrestricted egress for blockchain connectivity |

### Additional Ports (Not Configured)
Consider adding these ports for full functionality:

| Port | Protocol | Service | Security Consideration |
|------|----------|---------|------------------------|
| 8545 | TCP | JSON-RPC HTTP | Requires authentication |
| 8546 | TCP | JSON-RPC WebSocket | Requires authentication |
| 9090 | TCP | gRPC | For Cosmos chains |
| 6060 | TCP | Metrics/pprof | Internal only |

## Network Architecture

### Public Connectivity
```
Internet Peers
     │
     ↓ (30303/tcp+udp, 26656/tcp)
┌────────────┐
│ Elastic IP │ (Public Static IP)
└────────────┘
     │
┌────────────┐
│  RPC Node  │
└────────────┘
     │
     ↓ (Internal)
┌────────────┐
│   Relayer  │
└────────────┘
```

### P2P Network Topology
1. **Ethereum Network (Port 30303)**
   - TCP: Reliable message delivery
   - UDP: Peer discovery protocol
   - Typical connections: 25-50 peers
   - Bandwidth: 1-10 Mbps average

2. **Consensus Network (Port 26656)**
   - TCP only: Tendermint protocol
   - Persistent peer connections
   - Typical connections: 10-20 peers
   - Bandwidth: 0.5-5 Mbps average

## Blockchain Client Configuration

### Ethereum Execution Client (Geth)
```bash
# Geth configuration example
geth \
  --datadir /var/lib/ethereum \
  --syncmode full \
  --gcmode archive \
  --http \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --http.api eth,net,web3,txpool \
  --http.vhosts "*" \
  --ws \
  --ws.addr 0.0.0.0 \
  --ws.port 8546 \
  --ws.api eth,net,web3,txpool \
  --maxpeers 50 \
  --cache 8192 \
  --metrics \
  --metrics.addr 127.0.0.1 \
  --metrics.port 6060
```

### Consensus Client Configuration
```yaml
# Consensus client config (Lighthouse/Prysm)
network:
  listen_address: 0.0.0.0
  port: 9000
  discovery_port: 9000
  target_peers: 50

execution:
  endpoint: http://localhost:8545
  jwt_secret_file: /etc/ethereum/jwt.hex

metrics:
  enabled: true
  address: 127.0.0.1
  port: 5054
```

## Terraform Configuration

### Required Variables
| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `vpc_id` | string | Target VPC for deployment | `vpc-0123456789abcdef` |
| `subnet_id` | string | Public subnet for instance | `subnet-0123456789abcdef` |

### Optional Variables
| Variable | Default | Description | Customization |
|----------|---------|-------------|---------------|
| `aws_region` | `us-west-2` | AWS deployment region | Any AWS region |
| `ami_name_pattern` | `ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*` | AMI filter | Custom pattern |
| `ami_owner` | `099720109477` | Canonical's account | Custom AMI owner |
| `key_name` | `rpc-prod` | SSH key pair | Your key name |

## Outputs

### Exported Values
| Output | Type | Description | Usage Example |
|--------|------|-------------|---------------|
| `instance_ip_goat` | string | Public IP via EIP | RPC endpoint configuration |

### Output Usage
```bash
# Get RPC endpoint
RPC_IP=$(terraform output -raw instance_ip_goat)
echo "RPC Endpoint: http://${RPC_IP}:8545"

# Test connectivity
curl -X POST http://${RPC_IP}:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

## Operations

### Deployment Process

1. **Pre-Deployment Checklist**
```bash
# Verify public subnet has internet gateway
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-xxx"

# Check EIP limits
aws service-quotas get-service-quota \
  --service-code ec2 \
  --quota-code L-0263D0A3
```

2. **Terraform Deployment**
```bash
cd rpc/
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars" -auto-approve
```

3. **Post-Deployment Setup**
```bash
# Get public IP
PUBLIC_IP=$(terraform output -raw instance_ip_goat)

# Connect via Session Manager
aws ssm start-session --target $(terraform output -raw instance_id)

# Install blockchain client
sudo apt-get update
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install -y geth
```

### Blockchain Synchronization

#### Initial Sync Process
```bash
# Start Geth with fast sync
geth --syncmode snap --datadir /var/lib/ethereum

# Monitor sync progress
geth attach --datadir /var/lib/ethereum
> eth.syncing
> eth.blockNumber

# Estimate sync time
# Mainnet full sync: 6-12 hours
# Mainnet archive sync: 2-4 weeks
```

#### Sync Optimization
1. **Use Snapshot Sync**: Fastest initial sync method
2. **Increase Cache**: `--cache 8192` for better performance
3. **Peer Optimization**: `--maxpeers 100` for faster sync
4. **Bandwidth Management**: Monitor and adjust based on available bandwidth

### Monitoring

#### System Metrics
| Metric | Command | Alert Threshold |
|--------|---------|-----------------|
| **CPU Usage** | `top` | > 90% sustained |
| **Memory Usage** | `free -h` | > 14 GB used |
| **Disk Usage** | `df -h` | > 85% full |
| **Network I/O** | `iftop` | > 80% bandwidth |
| **Disk I/O** | `iostat -x 1` | > 90% util |

#### Blockchain Metrics
```bash
# Check sync status
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'

# Get peer count
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'

# Check latest block
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

#### CloudWatch Monitoring
```yaml
Metrics:
  - Namespace: CWAgent
    MetricName: disk_used_percent
    Dimensions:
      - Name: InstanceId
        Value: ${instance_id}
  
  - Namespace: AWS/EC2
    MetricName: NetworkIn
    Statistic: Sum
    Period: 300
  
  - Namespace: Custom/Blockchain
    MetricName: blocks_behind
    Unit: Count
```

### Maintenance

#### Regular Updates
```bash
# System updates
sudo apt-get update && sudo apt-get upgrade -y

# Geth updates
sudo apt-get update
sudo apt-get install --only-upgrade geth

# Restart with new version
sudo systemctl restart geth
```

#### Database Maintenance
```bash
# Prune ancient data (if not running archive node)
geth snapshot prune-state --datadir /var/lib/ethereum

# Verify database integrity
geth db inspect --datadir /var/lib/ethereum

# Backup critical data
aws s3 sync /var/lib/ethereum/keystore s3://backup-bucket/keystore/
```

#### Log Management
```bash
# Rotate logs
cat > /etc/logrotate.d/geth <<EOF
/var/log/geth/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 640 ethereum ethereum
    sharedscripts
    postrotate
        systemctl reload geth
    endscript
}
EOF

# Archive old logs
find /var/log/geth -name "*.log.gz" -mtime +30 | \
  xargs -I {} aws s3 mv {} s3://archive-bucket/logs/
```

### Troubleshooting

#### Common Issues

1. **Slow Synchronization**
```bash
# Check peer connectivity
geth attach --exec "admin.peers.length"

# Add bootstrap nodes
geth --bootnodes "enode://[bootstrap-node-list]"

# Check bandwidth usage
iftop -i eth0
```

2. **High Disk Usage**
```bash
# Check blockchain data size
du -sh /var/lib/ethereum/*

# Clean up old logs
find /var/log -name "*.gz" -delete

# Consider pruning if not archive node
geth snapshot prune-state
```

3. **Memory Issues**
```bash
# Check memory usage by process
ps aux | sort -nrk 4 | head

# Adjust Geth cache size
# Edit systemd service file
sudo systemctl edit geth
# Add: --cache 4096

# Monitor swap usage
vmstat 1 5
```

4. **Network Connectivity**
```bash
# Check listening ports
sudo netstat -tlnp | grep -E "30303|26656"

# Verify security groups
aws ec2 describe-security-groups --group-ids sg-xxx

# Test external connectivity
telnet bootstrap.ethereum.org 30303
```

## Security Hardening

### Network Security
1. **DDoS Protection**
```bash
# Rate limiting with iptables
sudo iptables -A INPUT -p tcp --dport 30303 \
  -m connlimit --connlimit-above 100 -j REJECT

# SYN flood protection
echo "net.ipv4.tcp_syncookies = 1" >> /etc/sysctl.conf
sysctl -p
```

2. **RPC Security**
```nginx
# Nginx reverse proxy for RPC
server {
    listen 8545;
    location / {
        proxy_pass http://localhost:8545;
        
        # Rate limiting
        limit_req zone=rpc burst=10;
        
        # IP whitelist
        allow 10.0.0.0/8;
        deny all;
    }
}
```

3. **Firewall Configuration**
```bash
# UFW setup
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
sudo ufw allow 26656/tcp
sudo ufw allow from 10.0.0.0/8 to any port 8545
sudo ufw enable
```

### Access Control
1. **SSH Hardening** (if enabled)
```bash
# Disable password auth
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config

# Restrict SSH access
echo "AllowUsers ubuntu" >> /etc/ssh/sshd_config

# Change default port
echo "Port 2222" >> /etc/ssh/sshd_config
```

2. **API Authentication**
```javascript
// JWT authentication for RPC
const jwt = require('jsonwebtoken');

app.use('/rpc', (req, res, next) => {
  const token = req.headers['authorization'];
  jwt.verify(token, secret, (err, decoded) => {
    if (err) return res.status(401).send('Unauthorized');
    next();
  });
});
```

## Performance Optimization

### Instance Optimization
| Parameter | Current | Optimized | Benefit |
|-----------|---------|-----------|---------|
| **Instance Type** | c5.2xlarge | c5n.2xlarge | Enhanced networking |
| **EBS Optimized** | Disabled | Enabled | Consistent I/O |
| **Enhanced Networking** | Default | SR-IOV | Lower latency |
| **Placement Group** | None | Cluster | Reduced latency |

### Blockchain Optimization
```bash
# Geth performance tuning
geth \
  --cache 12288 \           # 12GB cache
  --maxpeers 100 \          # More peers
  --light.maxpeers 0 \      # No light clients
  --txpool.globalslots 8192 \ # Larger tx pool
  --txpool.accountslots 32 \  # More tx per account
  --txpool.globalqueue 4096   # Larger queue
```

### Storage Optimization
1. **Use io2 for Critical Workloads**
```hcl
root_block_device {
  volume_type = "io2"
  volume_size = 500
  iops = 16000
}
```

2. **Separate Data Volumes**
```hcl
resource "aws_ebs_volume" "blockchain_data" {
  availability_zone = aws_instance.rpc-goat.availability_zone
  size              = 2000
  type              = "gp3"
  iops              = 16000
  throughput        = 1000
}
```

## Cost Analysis

### Monthly Cost Breakdown
| Component | Specification | Cost |
|-----------|--------------|------|
| **EC2 Instance** | c5.2xlarge (730 hrs) | $248.00 |
| **EBS Storage** | 100 GB GP3 | $8.00 |
| **Elastic IP** | Static allocation | $3.65 |
| **Data Transfer** | 5 TB outbound | $450.00 |
| **Total** | | **~$709.65** |

### Cost Optimization Strategies

1. **Reserved Instances**
   - 1-year: ~40% savings
   - 3-year: ~60% savings
   - Convertible: Flexibility for upgrades

2. **Storage Optimization**
   - Use S3 for old logs
   - Implement data lifecycle policies
   - Consider pruning non-essential data

3. **Network Optimization**
   - Use AWS PrivateLink where possible
   - Implement caching layers
   - Compress data transfers

4. **Right-Sizing**
   - Monitor actual usage
   - Consider c5.xlarge for light workloads
   - Use autoscaling for variable loads

## High Availability

### Multi-Node Setup
```hcl
# Deploy multiple RPC nodes
resource "aws_instance" "rpc-nodes" {
  count = 3
  
  # Spread across AZs
  subnet_id = element(var.public_subnet_ids, count.index)
  
  # Common configuration
  instance_type = "c5.2xlarge"
  ami = data.aws_ami.ubuntu_2204.id
}

# Load balancer for distribution
resource "aws_lb" "rpc-lb" {
  name               = "rpc-load-balancer"
  internal           = false
  load_balancer_type = "network"
  subnets            = var.public_subnet_ids
}
```

### Failover Strategy
1. **Health Checks**: Monitor block height
2. **Automatic Failover**: Route53 health checks
3. **Data Consistency**: Shared blockchain data via EFS
4. **Session Persistence**: Sticky sessions for WebSocket

## Disaster Recovery

### Backup Strategy
| Data Type | Backup Method | Frequency | Retention |
|-----------|--------------|-----------|-----------|
| **Blockchain Data** | EBS Snapshots | Daily | 7 days |
| **Configuration** | S3 + Git | On change | Indefinite |
| **Logs** | CloudWatch Logs | Real-time | 30 days |
| **Keys/Secrets** | AWS Secrets Manager | On change | Versioned |

### Recovery Procedures
1. **Instance Failure**
   - Launch from AMI
   - Restore from snapshot
   - Reassociate EIP
   - Resume synchronization

2. **Data Corruption**
   - Stop instance
   - Create new volume from snapshot
   - Attach and mount volume
   - Restart services

3. **Region Failure**
   - Deploy infrastructure in backup region
   - Restore from cross-region snapshots
   - Update DNS records
   - Verify connectivity

### RTO/RPO Targets
- **RTO**: 30 minutes for instance recovery
- **RPO**: 24 hours for blockchain data
- **Availability Target**: 99.9% uptime

---

*Module Version: 1.0*  
*Last Updated: 2025*  
*Terraform Version: >= 0.12*  
*AWS Provider Version: ~> 5.0*