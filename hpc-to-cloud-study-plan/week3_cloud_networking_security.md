# Week 3: Cloud Networking and Security
## From Physical Networks to Software-Defined Infrastructure

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how Virtual Private Clouds (VPCs) differ from physical network segmentation
- Design and implement cloud network security architectures
- Configure security groups, NACLs, and other cloud network controls
- Implement secure connectivity between cloud and on-premises (hybrid scenarios)
- Apply network security monitoring and logging in cloud environments
- Design network architectures for scientific computing workloads in the cloud

---

## Day 1: Virtual Private Clouds vs. Physical Networks

### Morning Session (3 hours)

#### HPC Network Architecture vs. Cloud Networking

**Traditional HPC Networking (what you know):**
- Physical switches and routers
- VLANs for network segmentation
- Dedicated high-speed interconnects (InfiniBand)
- Physical firewalls and security appliances
- Static IP addressing and routing
- Physical cable management and topology

**Cloud Networking Paradigm:**
- Software-defined networking (SDN)
- Virtual networks and subnets
- API-driven network configuration
- Distributed security controls
- Dynamic IP allocation and routing
- Global network infrastructure

#### Virtual Private Cloud (VPC) Concepts

**VPC vs. Physical Network:**

| HPC Physical Network | Cloud VPC | Key Difference |
|---------------------|-----------|----------------|
| Physical switches | Virtual switches | Software-defined vs. hardware |
| VLANs | Subnets | Logical vs. physical segmentation |
| Physical firewalls | Security groups | Distributed vs. centralized |
| Static routing | Route tables | Dynamic vs. static configuration |
| Physical cables | Virtual connections | Software vs. hardware connectivity |
| Network appliances | Managed services | Cloud-native vs. on-premises |

**VPC Components:**
- **Subnets**: Like VLANs, but software-defined
- **Route tables**: Control traffic routing between subnets
- **Internet gateways**: Provide internet access (like router/firewall)
- **NAT gateways**: Outbound internet access for private resources
- **VPC endpoints**: Private connections to cloud services
- **Peering connections**: Connect VPCs together

#### Network Addressing and Subnets

**IP Address Planning:**
```bash
# HPC network example (what you might know)
# Management network: 10.1.0.0/24
# Compute network: 10.2.0.0/16
# Storage network: 10.3.0.0/24

# Cloud VPC equivalent
# VPC CIDR: 10.0.0.0/16
# Public subnet: 10.0.1.0/24 (internet-accessible)
# Private subnet: 10.0.2.0/24 (no direct internet)
# Database subnet: 10.0.3.0/24 (isolated)
```

### Afternoon Session (2 hours)

#### Hands-On VPC Creation

**Create Basic VPC (AWS example):**
```bash
# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=research-vpc}]'

# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet}]'

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet}]'

# Create internet gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=research-igw}]'

# Attach internet gateway to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345678 \
  --vpc-id vpc-12345678
```

**Configure Routing:**
```bash
# Create route table for public subnet
aws ec2 create-route-table \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Add route to internet gateway
aws ec2 create-route \
  --route-table-id rtb-12345678 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-12345678

# Associate route table with public subnet
aws ec2 associate-route-table \
  --route-table-id rtb-12345678 \
  --subnet-id subnet-12345678
```

#### Network Security Zones

**HPC Security Zones:**
- DMZ for external-facing services
- Internal network for compute and storage
- Management network for administration
- Out-of-band network for hardware management

**Cloud Security Zones:**
- Public subnets for internet-facing resources
- Private subnets for internal applications
- Database subnets for data storage
- Management subnets for administrative access

---

## Day 2: Security Groups and Network ACLs

### Morning Session (3 hours)

#### Cloud Firewall Concepts

**Traditional Firewalls vs. Cloud Security:**

| HPC Firewalls | Cloud Security Groups | Key Difference |
|---------------|----------------------|----------------|
| Physical appliances | Virtual firewalls | Software vs. hardware |
| Network-based rules | Instance-based rules | Per-resource vs. network-wide |
| Stateful inspection | Stateful by default | Similar concept, different implementation |
| Central management | Distributed control | API-driven vs. GUI-based |
| Hardware limitations | Software scalability | Performance vs. flexibility |

#### Security Groups Deep Dive

**Security Group Characteristics:**
- **Stateful**: Return traffic automatically allowed
- **Instance-level**: Applied to network interfaces
- **Allow rules only**: Cannot create deny rules
- **Default deny**: All traffic blocked unless explicitly allowed
- **Multiple groups**: Can apply multiple groups to one instance

**Security Group Rules:**
```bash
# Create security group
aws ec2 create-security-group \
  --group-name research-compute \
  --description "Security group for research compute instances" \
  --vpc-id vpc-12345678

# Allow SSH from specific IP range
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/24

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow communication between instances in same group
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol -1 \
  --source-group sg-12345678
```

#### Network Access Control Lists (NACLs)

**NACLs vs. Security Groups:**
- **Subnet-level**: Applied to entire subnets
- **Stateless**: Must explicitly allow return traffic
- **Allow and deny rules**: Can create both types
- **Rule evaluation**: Processed in order by rule number
- **Default allow**: Default NACL allows all traffic

**NACL Configuration:**
```bash
# Create network ACL
aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=research-nacl}]'

# Create inbound rule (allow SSH)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=22,To=22 \
  --cidr-block 203.0.113.0/24 \
  --rule-action allow

# Create outbound rule (allow all)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress
```

### Afternoon Session (2 hours)

#### Hands-On Security Configuration

**Design Security Group Strategy:**
```bash
# Web tier security group
aws ec2 create-security-group \
  --group-name web-tier \
  --description "Web servers" \
  --vpc-id vpc-12345678

aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Application tier security group
aws ec2 create-security-group \
  --group-name app-tier \
  --description "Application servers" \
  --vpc-id vpc-12345678

aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 8080 \
  --source-group sg-web

# Database tier security group
aws ec2 create-security-group \
  --group-name db-tier \
  --description "Database servers" \
  --vpc-id vpc-12345678

aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp \
  --port 3306 \
  --source-group sg-app
```

**Security Group Best Practices:**
- Use descriptive names and descriptions
- Follow principle of least privilege
- Reference other security groups instead of IP ranges
- Regular review and cleanup of unused rules
- Document the purpose of each rule

#### Network Security Monitoring

**VPC Flow Logs:**
```bash
# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name VPCFlowLogs

# Query flow logs (example)
aws logs filter-log-events \
  --log-group-name VPCFlowLogs \
  --filter-pattern "{ $.action = \"REJECT\" }"
```

**Traffic Analysis:**
- Monitor rejected connections
- Identify unusual traffic patterns
- Detect potential security threats
- Optimize security group rules

---

## Day 3: Hybrid Connectivity and VPNs

### Morning Session (3 hours)

#### Connecting Cloud to HPC Infrastructure

**Hybrid Scenarios:**
- **Cloud bursting**: Extend HPC capacity to cloud during peak demand
- **Data processing**: Move data to cloud for analysis, results back to HPC
- **Disaster recovery**: Use cloud as backup site for HPC workloads
- **Development/testing**: Use cloud for development, production on HPC
- **Gradual migration**: Move workloads to cloud over time

**Connectivity Options:**

| Connection Type | HPC Equivalent | Use Case | Bandwidth | Latency |
|----------------|----------------|----------|-----------|---------|
| Site-to-Site VPN | VPN tunnel | Basic connectivity | Up to 1.25 Gbps | Variable |
| Direct Connect | Dedicated circuit | High bandwidth | Up to 100 Gbps | Consistent |
| Transit Gateway | Network hub | Multiple connections | Scalable | Low |
| VPC Peering | Direct connection | VPC-to-VPC | High | Low |

#### Site-to-Site VPN Configuration

**VPN Components:**
- **Customer Gateway**: Your side of the connection
- **Virtual Private Gateway**: Cloud side of the connection
- **VPN Connection**: The tunnel between them
- **Route Propagation**: Automatic route updates

**VPN Setup:**
```bash
# Create customer gateway (your HPC center's public IP)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.12 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=hpc-center-cgw}]'

# Create virtual private gateway
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=cloud-vgw}]'

# Attach VPN gateway to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-12345678 \
  --vpc-id vpc-12345678

# Create VPN connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-12345678 \
  --vpn-gateway-id vgw-12345678 \
  --tag-specifications 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=hpc-to-cloud-vpn}]'
```

### Afternoon Session (2 hours)

#### Direct Connect for High-Performance

**When to Use Direct Connect:**
- High bandwidth requirements (>1 Gbps)
- Consistent network performance needed
- Large data transfers between HPC and cloud
- Latency-sensitive applications
- Compliance requirements for private connectivity

**Direct Connect Setup:**
```bash
# Create Direct Connect gateway
aws directconnect create-direct-connect-gateway \
  --name hpc-dx-gateway

# Create virtual interface
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-12345678 \
  --new-private-virtual-interface \
  vlan=100,virtualInterfaceName=hpc-vif,asn=65000,customerAddress=192.168.1.1/30,amazonAddress=192.168.1.2/30,directConnectGatewayId=dx-gw-12345678
```

#### Network Performance Optimization

**Bandwidth Considerations:**
- Scientific data transfer requirements
- Burst vs. sustained bandwidth needs
- Cost optimization for data transfer
- Multiple path redundancy

**Latency Optimization:**
- Placement groups for low-latency computing
- Enhanced networking features
- Regional selection for optimal performance
- Network topology design

**Performance Testing:**
```bash
# Test network performance between HPC and cloud
iperf3 -c cloud-instance-ip -t 60 -P 4

# Test latency
ping -c 100 cloud-instance-ip

# Test bandwidth with large file transfer
time scp large-dataset.tar.gz user@cloud-instance:/tmp/
```

---

## Day 4: Cloud-Native Security Services

### Morning Session (3 hours)

#### Web Application Firewalls (WAF)

**WAF vs. Traditional Firewalls:**
- **Application-layer protection**: Inspects HTTP/HTTPS traffic
- **Rule-based filtering**: Block common web attacks
- **Managed rules**: Pre-configured protection sets
- **Custom rules**: Organization-specific protections
- **Integration**: Works with load balancers and CDNs

**WAF Configuration:**
```bash
# Create WAF web ACL
aws wafv2 create-web-acl \
  --name research-portal-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules file://waf-rules.json

# Example WAF rule (block SQL injection)
{
  "Name": "SQLInjectionRule",
  "Priority": 1,
  "Statement": {
    "SqliMatchStatement": {
      "FieldToMatch": {
        "AllQueryArguments": {}
      },
      "TextTransformations": [
        {
          "Priority": 0,
          "Type": "URL_DECODE"
        }
      ]
    }
  },
  "Action": {
    "Block": {}
  }
}
```

#### DDoS Protection

**DDoS Protection Layers:**
- **Network layer**: Volumetric attacks
- **Transport layer**: Protocol attacks
- **Application layer**: Sophisticated attacks
- **Managed protection**: Cloud provider services
- **Custom mitigation**: Application-specific defenses

**DDoS Protection Setup:**
```bash
# Enable DDoS protection (AWS Shield Advanced)
aws shield subscribe-to-proactive-engagement \
  --proactive-engagement-status ENABLED

# Create DDoS response team contact
aws shield put-proactive-engagement-details \
  --proactive-engagement-status ENABLED \
  --emergency-contact-list file://emergency-contacts.json
```

#### Network Load Balancers

**Load Balancer Types:**
- **Application Load Balancer**: HTTP/HTTPS traffic
- **Network Load Balancer**: TCP/UDP traffic
- **Gateway Load Balancer**: Third-party appliances
- **Classic Load Balancer**: Legacy option

**Load Balancer Configuration:**
```bash
# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name research-portal-alb \
  --subnets subnet-12345678 subnet-87654321 \
  --security-groups sg-12345678 \
  --scheme internet-facing \
  --type application

# Create target group
aws elbv2 create-target-group \
  --name research-servers \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678 \
  --health-check-path /health

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/research-servers/1234567890123456 \
  --targets Id=i-12345678,Port=80 Id=i-87654321,Port=80
```

### Afternoon Session (2 hours)

#### Content Delivery Networks (CDN)

**CDN Benefits for Scientific Computing:**
- **Global distribution**: Faster access to datasets
- **Caching**: Reduce load on origin servers
- **DDoS protection**: Built-in attack mitigation
- **SSL termination**: Simplified certificate management
- **Compression**: Reduce bandwidth usage

**CDN Configuration:**
```bash
# Create CloudFront distribution
aws cloudfront create-distribution \
  --distribution-config file://distribution-config.json

# Example distribution config
{
  "CallerReference": "research-portal-cdn-2024",
  "Comment": "CDN for research portal",
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "research-portal-origin",
        "DomainName": "research-portal.example.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only"
        }
      }
    ]
  }
}
```

#### API Gateway Security

**API Gateway Features:**
- **Authentication**: IAM, Cognito, custom authorizers
- **Rate limiting**: Throttling and quotas
- **Request validation**: Input validation
- **Monitoring**: CloudWatch integration
- **Caching**: Response caching

**API Gateway Setup:**
```bash
# Create REST API
aws apigateway create-rest-api \
  --name research-data-api \
  --description "API for research data access"

# Create API key for authentication
aws apigateway create-api-key \
  --name research-client-key \
  --description "API key for research client applications"

# Create usage plan with throttling
aws apigateway create-usage-plan \
  --name research-usage-plan \
  --throttle burstLimit=100,rateLimit=50 \
  --quota limit=10000,period=DAY
```

---

## Day 5: Network Monitoring and Troubleshooting

### Morning Session (3 hours)

#### Cloud Network Monitoring

**Monitoring Tools:**
- **VPC Flow Logs**: Network traffic analysis
- **CloudWatch**: Metrics and alarms
- **AWS X-Ray**: Application tracing
- **Third-party tools**: Datadog, New Relic, etc.

**Flow Log Analysis:**
```bash
# Enable detailed flow logs
aws ec2 create-flow-logs \
  --resource-type NetworkInterface \
  --resource-ids eni-12345678 \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination s3://network-logs-bucket/flow-logs/

# Analyze flow logs with AWS CLI
aws s3 cp s3://network-logs-bucket/flow-logs/ . --recursive
grep "REJECT" *.gz | head -20

# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
  --dashboard-name NetworkMonitoring \
  --dashboard-body file://network-dashboard.json
```

**Security Monitoring:**
```python
#!/usr/bin/env python3
import boto3
import json
from datetime import datetime, timedelta

def analyze_security_groups():
    """Analyze security group rules for potential issues"""
    ec2 = boto3.client('ec2')
    
    # Get all security groups
    response = ec2.describe_security_groups()
    
    for sg in response['SecurityGroups']:
        print(f"Analyzing Security Group: {sg['GroupName']} ({sg['GroupId']})")
        
        # Check for overly permissive rules
        for rule in sg['IpPermissions']:
            for ip_range in rule.get('IpRanges', []):
                if ip_range['CidrIp'] == '0.0.0.0/0':
                    print(f"  WARNING: Rule allows access from anywhere (0.0.0.0/0)")
                    print(f"    Protocol: {rule.get('IpProtocol', 'All')}")
                    print(f"    Port: {rule.get('FromPort', 'All')}-{rule.get('ToPort', 'All')}")

def monitor_network_performance():
    """Monitor network performance metrics"""
    cloudwatch = boto3.client('cloudwatch')
    
    # Get network metrics for EC2 instances
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='NetworkIn',
        Dimensions=[
            {
                'Name': 'InstanceId',
                'Value': 'i-12345678'
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )
    
    for datapoint in response['Datapoints']:
        print(f"Network In: {datapoint['Average']:.2f} bytes/sec at {datapoint['Timestamp']}")

if __name__ == "__main__":
    analyze_security_groups()
    monitor_network_performance()
```

### Afternoon Session (2 hours)

#### Network Troubleshooting

**Common Network Issues:**
- **Connectivity problems**: Security group misconfigurations
- **Performance issues**: Bandwidth limitations, latency
- **DNS resolution**: Route 53 configuration problems
- **Load balancer issues**: Health check failures
- **VPN connectivity**: Tunnel status and routing

**Troubleshooting Tools:**
```bash
# Test connectivity
aws ec2 describe-instances --instance-ids i-12345678
aws ec2 describe-security-groups --group-ids sg-12345678
aws ec2 describe-route-tables --route-table-ids rtb-12345678

# Check VPN status
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-12345678

# Test DNS resolution
nslookup research-portal.example.com
dig research-portal.example.com

# Check load balancer health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/research-servers/1234567890123456
```

**Performance Optimization:**
- **Instance types**: Choose appropriate network performance
- **Placement groups**: Optimize for low latency
- **Enhanced networking**: Enable SR-IOV
- **Bandwidth allocation**: Monitor and adjust as needed

#### Network Security Best Practices

**Security Checklist:**
- [ ] Use least privilege for security group rules
- [ ] Enable VPC Flow Logs for monitoring
- [ ] Implement network segmentation with subnets
- [ ] Use NACLs for additional subnet-level protection
- [ ] Enable DDoS protection for public-facing resources
- [ ] Monitor network traffic for anomalies
- [ ] Regular security group audits and cleanup
- [ ] Implement WAF for web applications
- [ ] Use private subnets for sensitive resources
- [ ] Enable encryption in transit for all communications

---

## Week 3 Deliverable: Cloud Network Security Architecture

Create a comprehensive document (10-15 pages) covering:

### Executive Summary (1-2 pages)
- Cloud networking approach vs. traditional HPC networking
- Key security benefits and considerations
- Implementation timeline and resource requirements
- Integration with existing HPC infrastructure

### Network Architecture Design (3-4 pages)

**VPC Design:**
- Network topology and subnet design
- IP addressing scheme and CIDR planning
- Routing and connectivity architecture
- Security zone implementation

**Hybrid Connectivity:**
- Connection options analysis (VPN vs. Direct Connect)
- Bandwidth and performance requirements
- Redundancy and failover planning
- Cost optimization strategies

### Security Implementation (3-4 pages)

**Network Security Controls:**
- Security group design and implementation
- Network ACL configuration
- WAF and DDoS protection setup
- Load balancer security configuration

**Monitoring and Logging:**
- VPC Flow Logs implementation
- Security monitoring and alerting
- Performance monitoring setup
- Incident response procedures

### Migration and Integration Plan (2-3 pages)

**Phase 1: Foundation (0-3 months):**
- Basic VPC setup and configuration
- Initial security group implementation
- VPN connectivity establishment
- Basic monitoring setup

**Phase 2: Enhancement (3-6 months):**
- Advanced security services deployment
- Performance optimization
- Comprehensive monitoring implementation
- Integration testing and validation

**Phase 3: Optimization (6-12 months):**
- Advanced networking features
- Automation and orchestration
- Continuous improvement processes
- Full hybrid integration

### Operational Procedures (1-2 pages)
- Network management procedures
- Security monitoring and response
- Performance optimization guidelines
- Troubleshooting and support processes

---

## Key Security Questions Answered This Week

By the end of week 3, you should be able to answer:

**About Cloud Networking:**
- [ ] How do VPCs differ from physical network infrastructure?
- [ ] What are the key components of cloud network security?
- [ ] How should network segmentation be implemented in cloud?
- [ ] What are the trade-offs between different connectivity options?

**About Security Implementation:**
- [ ] How should security groups and NACLs be configured?
- [ ] What cloud-native security services are available?
- [ ] How should network monitoring and logging be implemented?
- [ ] What are the best practices for cloud network security?

**About Hybrid Integration:**
- [ ] How should cloud networks integrate with existing HPC infrastructure?
- [ ] What are the performance and security considerations for hybrid connectivity?
- [ ] How should network security be managed across hybrid environments?
- [ ] What monitoring and troubleshooting capabilities are needed?

---

## Resources for Week 3

### Technical Documentation
- AWS VPC User Guide
- Azure Virtual Network documentation
- Google Cloud VPC documentation
- Network security best practices guides

### Security Frameworks
- Cloud security alliance (CSA) guidance
- NIST cloud networking security guidelines
- Industry-specific networking requirements
- Compliance framework networking controls

### Tools and Services
- Cloud provider networking services
- Network monitoring and analysis tools
- Security scanning and assessment tools
- Hybrid connectivity solutions

---

## Success Metrics for Week 3

You should be able to:
- [ ] Design secure cloud network architectures
- [ ] Configure VPCs, security groups, and network controls
- [ ] Implement hybrid connectivity between cloud and HPC
- [ ] Set up network monitoring and security logging
- [ ] Troubleshoot common cloud networking issues

---

## Common Week 3 Challenges and Solutions

**"Cloud networking seems overly complex compared to physical networks"**
- Start with basic VPC concepts and build complexity gradually
- Use network diagrams to visualize the architecture
- Practice with simple scenarios before complex ones
- Focus on security zones and traffic flow patterns

**"Security groups and NACLs are confusing"**
- Remember: Security groups are stateful, NACLs are stateless
- Start with security groups for most use cases
- Use NACLs for additional subnet-level protection
- Test rules carefully and document their purpose

**"Hybrid connectivity setup seems too complex"**
- Start with simple site-to-site VPN for basic connectivity
- Work with network team for Direct Connect implementation
- Test connectivity thoroughly before production use
- Plan for redundancy and failover scenarios

**"Network performance in cloud is different from HPC"**
- Understand cloud networking performance characteristics
- Use appropriate instance types for network performance
- Implement placement groups for low-latency requirements
- Monitor and optimize based on actual performance metrics

---

## Week 3 Wrap-Up

By the end of week 3, you should understand how cloud networking differs from traditional physical networks and how to implement secure, performant network architectures in the cloud. Cloud networking provides much more flexibility and programmability than physical networks, but requires understanding software-defined networking concepts.

The key insight: Cloud networking is about software-defined infrastructure that can be programmatically configured and managed. It provides more flexibility and scalability than physical networks, but requires different design patterns and security approaches.

**Next week**: We'll explore cloud storage and data management, including how cloud storage services differ from parallel filesystems and how to implement data security and lifecycle management in cloud environments.