# Week 02: Connectivity & Networking Between Environments

## Learning Objectives
By the end of this week, you will be able to:
- Design secure, high-performance network connections between HPC and cloud environments
- Configure VPN gateways and direct connections for hybrid architectures
- Optimize network performance for data-intensive HPC workloads
- Implement network security best practices for hybrid environments
- Troubleshoot common connectivity issues in HPC-cloud integrations

## Overview
Network connectivity is the backbone of any successful hybrid HPC-cloud implementation. This week focuses on the networking technologies, protocols, and best practices that enable seamless communication between on-premises HPC systems and cloud resources.

## Key Concepts

### Network Requirements for HPC-Cloud Hybrid

#### Bandwidth Considerations
- **Data Transfer Volumes**: HPC workloads often involve terabytes of data
- **Burst Requirements**: Sudden spikes during job submission/completion
- **Sustained Throughput**: Long-running data synchronization processes
- **Bidirectional Flows**: Input data to cloud, results back to HPC

#### Latency Sensitivity
- **Interactive Workloads**: Real-time visualization, debugging sessions
- **Tightly Coupled Applications**: MPI jobs spanning environments
- **Storage Operations**: Distributed filesystem performance
- **Control Plane**: Job scheduling and monitoring communications

#### Reliability and Availability
- **Redundant Paths**: Multiple connectivity options for failover
- **Quality of Service**: Prioritization of critical traffic
- **Monitoring and Alerting**: Proactive issue detection
- **Disaster Recovery**: Network-level business continuity

## Connectivity Technologies Deep Dive

### 1. VPN-Based Connections

#### Site-to-Site VPN
**Advantages**:
- Quick to deploy and configure
- Cost-effective for moderate bandwidth needs
- Encrypted by default
- Works over existing internet connections

**Disadvantages**:
- Limited bandwidth (typically 1-10 Gbps)
- Variable latency due to internet routing
- Potential security concerns with shared infrastructure
- Performance affected by internet congestion

**Implementation Example**:
```bash
# Configure IPsec VPN tunnel (Linux/strongSwan)
conn hpc-to-aws
    left=192.168.1.1          # HPC gateway IP
    leftsubnet=10.0.0.0/16    # HPC network
    right=52.1.2.3            # AWS VPN gateway
    rightsubnet=172.16.0.0/16 # AWS VPC
    authby=secret
    keyexchange=ikev2
    ike=aes256-sha256-modp2048
    esp=aes256-sha256
    auto=start
```

#### Client VPN
**Use Cases**:
- Remote researcher access to hybrid resources
- Administrative access from multiple locations
- Temporary project collaborations

### 2. Direct Connections

#### AWS Direct Connect
- **Bandwidth**: 1 Gbps to 100 Gbps dedicated connections
- **Latency**: Consistent, predictable performance
- **Cost**: Reduced data transfer charges
- **Security**: Private connection, not over internet

#### Azure ExpressRoute
- **Global Reach**: Connect to multiple Azure regions
- **Bandwidth**: 50 Mbps to 100 Gbps options
- **SLA**: 99.95% availability guarantee
- **Integration**: Native Azure service integration

#### Google Cloud Interconnect
- **Dedicated Interconnect**: 10 Gbps or 100 Gbps circuits
- **Partner Interconnect**: Flexible bandwidth options
- **Cross-Cloud**: Connect to other cloud providers
- **Hybrid Connectivity**: On-premises to multiple GCP regions

### 3. Software-Defined Networking (SDN)

#### SD-WAN Solutions
- **Centralized Management**: Single pane of glass for all connections
- **Dynamic Path Selection**: Automatic failover and load balancing
- **Application Awareness**: QoS based on application requirements
- **Security Integration**: Built-in firewall and encryption

#### Overlay Networks
- **VXLAN**: Layer 2 over Layer 3 tunneling
- **GRE Tunnels**: Generic routing encapsulation
- **MPLS**: Multi-protocol label switching for enterprise networks
- **Container Networking**: Kubernetes CNI plugins for hybrid clusters

## Network Architecture Patterns

### Pattern 1: Hub-and-Spoke
```
HPC Cluster ←→ Central Hub ←→ Cloud Provider A
                    ↕
               Cloud Provider B
```
**Benefits**: Centralized management, simplified routing
**Drawbacks**: Single point of failure, potential bottleneck

### Pattern 2: Mesh Connectivity
```
HPC Cluster ←→ Cloud Provider A
     ↕              ↕
Cloud Provider B ←→ Cloud Provider C
```
**Benefits**: Redundancy, optimal routing
**Drawbacks**: Complex management, higher costs

### Pattern 3: Tiered Architecture
```
HPC Cluster ←→ Edge Cloud ←→ Public Cloud
                   ↕
              Regional Cloud
```
**Benefits**: Latency optimization, data locality
**Drawbacks**: Increased complexity, multiple management planes

## Hands-On Lab: Building Hybrid Network Connectivity

### Lab Environment Setup
- **On-Premises Simulation**: AWS EC2 instances in "HPC" VPC
- **Cloud Environment**: Separate VPC representing cloud resources
- **Connectivity**: VPN and simulated Direct Connect

### Exercise 1: VPN Configuration (60 minutes)

#### Step 1: Create VPN Gateway
```bash
# AWS CLI commands for VPN setup
aws ec2 create-vpn-gateway --type ipsec.1 --amazon-side-asn 65000

# Create customer gateway (representing HPC site)
aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip 203.0.113.12 \
    --bgp-asn 65001
```

#### Step 2: Configure Routing
```bash
# Create route table for hybrid traffic
aws ec2 create-route-table --vpc-id vpc-12345678

# Add routes for HPC subnets
aws ec2 create-route \
    --route-table-id rtb-12345678 \
    --destination-cidr-block 10.0.0.0/16 \
    --vpn-gateway-id vgw-12345678
```

#### Step 3: Test Connectivity
```bash
# Test basic connectivity
ping 172.16.1.10  # Cloud instance from HPC

# Measure bandwidth
iperf3 -c 172.16.1.10 -t 60 -P 4

# Test latency
mtr --report --report-cycles 100 172.16.1.10
```

### Exercise 2: Performance Optimization (45 minutes)

#### Network Tuning Parameters
```bash
# Optimize TCP settings for high-bandwidth, high-latency networks
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 134217728' >> /etc/sysctl.conf
sysctl -p
```

#### Parallel Data Transfer
```bash
# Use multiple streams for large file transfers
parallel -j 4 rsync -avz --progress \
    /hpc/data/chunk_{} \
    user@cloud-host:/cloud/data/ ::: {1..4}
```

### Exercise 3: Monitoring and Troubleshooting (30 minutes)

#### Network Monitoring Setup
```bash
# Install monitoring tools
sudo apt-get install iftop nethogs nload

# Monitor real-time bandwidth usage
iftop -i eth0

# Track per-process network usage
sudo nethogs eth0
```

#### Performance Baseline
```bash
# Create performance baseline script
#!/bin/bash
echo "=== Network Performance Baseline ===" > baseline.txt
echo "Date: $(date)" >> baseline.txt
echo "Bandwidth Test:" >> baseline.txt
iperf3 -c $CLOUD_HOST -t 30 >> baseline.txt
echo "Latency Test:" >> baseline.txt
ping -c 100 $CLOUD_HOST >> baseline.txt
echo "Traceroute:" >> baseline.txt
traceroute $CLOUD_HOST >> baseline.txt
```

## Security Implementation

### Network Security Best Practices

#### 1. Encryption in Transit
```bash
# IPsec configuration for strong encryption
conn hpc-cloud-secure
    ike=aes256gcm16-sha384-prfsha384-ecp384!
    esp=aes256gcm16-ecp384!
    keyexchange=ikev2
    rekey=no
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s
```

#### 2. Network Segmentation
```yaml
# Kubernetes NetworkPolicy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hpc-cloud-isolation
spec:
  podSelector:
    matchLabels:
      tier: compute
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: hpc-namespace
    ports:
    - protocol: TCP
      port: 22
```

#### 3. Access Control Lists
```bash
# iptables rules for hybrid traffic control
# Allow HPC subnet access to cloud resources
iptables -A FORWARD -s 10.0.0.0/16 -d 172.16.0.0/16 -j ACCEPT

# Block direct internet access from compute nodes
iptables -A FORWARD -s 10.0.100.0/24 -d 0.0.0.0/0 -j DROP

# Allow specific services only
iptables -A FORWARD -p tcp --dport 443 -j ACCEPT  # HTTPS
iptables -A FORWARD -p tcp --dport 22 -j ACCEPT   # SSH
```

## Performance Optimization Strategies

### Bandwidth Optimization

#### 1. Data Compression
```bash
# Use compression for text-based data
rsync -avz --compress-level=9 /hpc/results/ cloud:/storage/

# Parallel compression for large datasets
pigz -p 8 large_dataset.txt
scp large_dataset.txt.gz cloud:/storage/
```

#### 2. Parallel Transfers
```python
# Python script for parallel data transfer
import concurrent.futures
import subprocess
import os

def transfer_chunk(chunk_info):
    source, dest, chunk_id = chunk_info
    cmd = f"rsync -avz {source} {dest}/chunk_{chunk_id}/"
    return subprocess.run(cmd, shell=True, capture_output=True)

# Split large dataset into chunks
chunks = [(f"/hpc/data/chunk_{i}", "cloud:/storage", i) 
          for i in range(1, 9)]

# Transfer in parallel
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(transfer_chunk, chunks))
```

#### 3. Protocol Optimization
```bash
# GridFTP for high-performance data transfer
globus-url-copy -p 8 -tcp-bs 2097152 \
    file:///hpc/data/large_file.dat \
    gsiftp://cloud-host/storage/large_file.dat

# Aspera for ultra-high-speed transfers
ascp -T -l 1000M /hpc/data/ user@cloud-host:/storage/
```

### Latency Optimization

#### 1. Connection Pooling
```python
# HTTP connection pooling for API calls
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retry_strategy = Retry(total=3, backoff_factor=1)
adapter = HTTPAdapter(pool_connections=10, pool_maxsize=20, 
                     max_retries=retry_strategy)
session.mount("https://", adapter)
```

#### 2. Caching Strategies
```bash
# Local caching for frequently accessed data
# Setup local cache directory
mkdir -p /hpc/cache/cloud-data

# Cache management script
#!/bin/bash
CACHE_DIR="/hpc/cache/cloud-data"
CLOUD_PATH="cloud:/storage/reference-data"
MAX_AGE=86400  # 24 hours

if [ ! -f "$CACHE_DIR/timestamp" ] || 
   [ $(($(date +%s) - $(cat $CACHE_DIR/timestamp))) -gt $MAX_AGE ]; then
    rsync -avz $CLOUD_PATH/ $CACHE_DIR/
    date +%s > $CACHE_DIR/timestamp
fi
```

## Real-World Implementation Examples

### Case Study 1: Genomics Research Pipeline
**Challenge**: 100TB genomic datasets, 48-hour processing windows
**Solution**: 
- 10 Gbps Direct Connect for primary data transfer
- VPN backup for control traffic
- Parallel transfer protocols (GridFTP)
- Local caching for reference genomes

**Results**:
- Data transfer time reduced from 72 hours to 8 hours
- 99.9% network availability
- 60% reduction in cloud data transfer costs

### Case Study 2: Financial Risk Modeling
**Challenge**: Low-latency requirements, regulatory compliance
**Solution**:
- Dedicated fiber connections to multiple cloud regions
- SD-WAN for intelligent path selection
- Encrypted tunnels for sensitive data
- Real-time network monitoring

**Results**:
- Sub-10ms latency to primary cloud region
- Automatic failover in under 30 seconds
- Full audit trail for compliance

### Case Study 3: Climate Modeling Consortium
**Challenge**: Multi-site collaboration, massive datasets
**Solution**:
- Mesh network connecting 5 HPC centers and 3 cloud regions
- Research and Education Networks (REN) integration
- Data staging areas in each region
- Bandwidth reservation system

**Results**:
- 40 Gbps aggregate bandwidth
- Coordinated global climate simulations
- Reduced data movement by 70% through intelligent staging

## Troubleshooting Common Issues

### Connectivity Problems

#### Symptom: Intermittent Connection Drops
```bash
# Diagnosis steps
1. Check physical layer (if applicable)
   ethtool eth0
   
2. Monitor connection stability
   while true; do
     ping -c 1 $CLOUD_HOST || echo "$(date): Connection failed"
     sleep 5
   done

3. Check VPN tunnel status
   ipsec status
   ipsec statusall
```

#### Symptom: Poor Performance
```bash
# Performance diagnosis
1. Baseline network performance
   iperf3 -c $CLOUD_HOST -t 60 -i 5

2. Check for packet loss
   mtr --report --report-cycles 100 $CLOUD_HOST

3. Analyze TCP window scaling
   ss -i dst $CLOUD_HOST
```

#### Symptom: Authentication Failures
```bash
# VPN troubleshooting
1. Check certificates
   openssl x509 -in /etc/ipsec.d/certs/client.crt -text -noout

2. Verify pre-shared keys
   ipsec verify

3. Check logs
   tail -f /var/log/charon.log
```

## Week 2 Deliverables

### Lab Report
Document your hands-on lab experience including:

1. **Network Architecture Diagram** (1 page)
   - Physical and logical topology
   - IP addressing scheme
   - Security boundaries

2. **Performance Baseline** (1-2 pages)
   - Bandwidth measurements
   - Latency analysis
   - Comparison with requirements

3. **Security Implementation** (1 page)
   - Encryption configuration
   - Access control setup
   - Monitoring implementation

4. **Troubleshooting Guide** (1 page)
   - Common issues encountered
   - Resolution procedures
   - Preventive measures

### Design Exercise
Create a network design for a specific hybrid scenario:
- **Scenario**: Choose from provided options or create your own
- **Requirements**: Define bandwidth, latency, and security needs
- **Solution**: Detailed network architecture with justification
- **Cost Analysis**: Compare different connectivity options

## Additional Resources

### Network Planning Tools
- **Bandwidth Calculators**: Cloud provider tools for sizing connections
- **Latency Estimators**: Geographic distance and routing analysis
- **Cost Calculators**: Data transfer and connection pricing tools

### Monitoring and Management
- **Open Source**: Nagios, Zabbix, LibreNMS
- **Commercial**: SolarWinds, PRTG, Datadog
- **Cloud Native**: AWS CloudWatch, Azure Monitor, GCP Operations

### Performance Testing
- **iperf3**: Bandwidth and latency testing
- **nuttcp**: Network performance measurement
- **Flent**: Flexible network tester
- **MTR**: Network diagnostic tool

## Preparation for Week 3
Next week focuses on Identity & Access Management across hybrid boundaries. Please:

1. **Review your organization's identity systems** (Active Directory, LDAP, etc.)
2. **Research cloud identity services** (AWS IAM, Azure AD, Google Cloud Identity)
3. **Install identity management tools** for hands-on exercises
4. **Read**: "Federated Identity in Hybrid Environments" (course materials)

---

*Network connectivity is the foundation of hybrid success. Invest time in getting it right, and everything else becomes much easier!*