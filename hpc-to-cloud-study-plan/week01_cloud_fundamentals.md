# Week 1: Cloud Computing Fundamentals
## Understanding What Makes Cloud Different from HPC

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand the fundamental differences between cloud and HPC computing models
- Translate HPC concepts to cloud equivalents and understand key differences
- Navigate basic cloud services and understand their security implications
- Recognize how cloud's "shared responsibility model" differs from HPC's full ownership
- Speak intelligently about cloud concepts in security discussions

---

## Day 1: Cloud Computing Models and Architecture

### Morning Session (2-3 hours)

#### Core Cloud Concepts vs. HPC

**What Makes Cloud Different:**
- **On-demand self-service**: Users provision resources instantly via APIs/consoles
- **Elastic scaling**: Resources scale up/down automatically based on demand
- **Pay-per-use**: Only pay for what you consume, when you consume it
- **Managed services**: Provider handles infrastructure, you focus on applications
- **Global reach**: Deploy anywhere in the world in minutes

**Fundamental Mindset Shift:**

| Aspect | HPC Mindset | Cloud Mindset |
|--------|-------------|---------------|
| **Resources** | Fixed cluster, maximize utilization | Infinite resources, optimize cost |
| **Capacity Planning** | Plan for peak, buy hardware | Scale elastically, pay as you go |
| **Failure Handling** | Prevent failures, maintain uptime | Expect failures, design for resilience |
| **Security Model** | Full control, physical security | Shared responsibility, API security |
| **Operations** | Long-term stability | Continuous deployment and change |

#### Cloud Service Models

**Infrastructure as a Service (IaaS):**
- **HPC equivalent**: Like having a data center you can provision instantly
- **Examples**: AWS EC2, Azure VMs, Google Compute Engine
- **Security**: You manage OS, applications, data; provider manages hardware
- **Use case**: Lift-and-shift from HPC, custom configurations

**Platform as a Service (PaaS):**
- **HPC equivalent**: Like having pre-configured software stacks ready to use
- **Examples**: AWS Lambda, Azure App Service, Google Cloud Run
- **Security**: You manage code and data; provider manages runtime and OS
- **Use case**: Focus on application logic, not infrastructure

**Software as a Service (SaaS):**
- **HPC equivalent**: Like using centrally-managed scientific applications
- **Examples**: Office 365, Salesforce, Google Workspace
- **Security**: Provider manages everything; you manage user access and data
- **Use case**: Standard business applications

### Afternoon Session (1-2 hours)

#### Cloud Deployment Models

**Public Cloud:**
- **HPC equivalent**: Like using a national supercomputing center
- **Characteristics**: Shared infrastructure, internet-accessible, cost-effective
- **Security considerations**: Data leaves your premises, compliance challenges
- **Examples**: AWS, Azure, Google Cloud

**Private Cloud:**
- **HPC equivalent**: Your current HPC center with cloud-like interfaces
- **Characteristics**: Dedicated infrastructure, on-premises or hosted
- **Security considerations**: Full control, but higher cost and complexity
- **Examples**: VMware vSphere, OpenStack, Azure Stack

**Hybrid Cloud:**
- **HPC equivalent**: Using both local HPC and external cloud resources
- **Characteristics**: Mix of public and private, workload portability
- **Security considerations**: Complex security boundaries, data movement
- **Examples**: AWS Outposts, Azure Arc, Google Anthos

#### Hands-On Cloud Exploration

**If you have access to a cloud account:**
```bash
# AWS CLI basics (if available)
aws ec2 describe-instances --region us-east-1
aws s3 ls
aws iam get-user

# Azure CLI basics (if available)
az vm list
az storage account list
az account show

# Google Cloud basics (if available)
gcloud compute instances list
gcloud storage ls
gcloud auth list
```

**Cloud Console Exploration:**
- Navigate the web console interface
- Explore the service catalog
- Look at billing and cost management
- Examine identity and access management
- Review security and compliance features

---

## Day 2: Cloud vs. HPC Computing Models

### Morning Session (3 hours)

#### Compute Models Comparison

**HPC Batch Jobs vs. Cloud Services:**

| HPC Batch Jobs | Cloud Equivalent | Key Difference |
|----------------|------------------|----------------|
| Submit job to queue | Launch EC2 instance | Instant vs. queued |
| Job runs on assigned nodes | Instance runs until stopped | Temporary vs. persistent |
| Resource allocation by scheduler | Self-service resource selection | Centralized vs. self-service |
| Shared nodes over time | Dedicated instances | Sequential vs. concurrent sharing |
| Fixed cluster capacity | Elastic scaling | Limited vs. unlimited resources |

**Workload Patterns:**

**HPC Workloads:**
- Long-running simulations (hours to weeks)
- Batch processing with defined start/end
- High-performance parallel computing
- Predictable resource requirements
- Scientific and research applications

**Cloud Workloads:**
- Always-on web services and APIs
- Event-driven serverless functions
- Microservices architectures
- Variable and unpredictable load
- Business and consumer applications

#### Service-Oriented Architecture

**From Monolithic HPC to Microservices:**
- **HPC**: Large, monolithic scientific applications
- **Cloud**: Small, independent services that communicate via APIs
- **Benefits**: Scalability, resilience, independent deployment
- **Challenges**: Complexity, network latency, distributed system issues

**API-First Design:**
- Everything is accessible via REST APIs
- Infrastructure as code
- Programmatic resource management
- Integration and automation capabilities

### Afternoon Session (2 hours)

#### Hands-On Compute Comparison

**Create Simple Compute Comparison:**

**HPC Job Script (what you know):**
```bash
#!/bin/bash
#SBATCH --job-name=hello-world
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "Hello from HPC node: $(hostname)"
echo "Job ID: $SLURM_JOB_ID"
echo "Running at: $(date)"
sleep 60
echo "Job completed at: $(date)"
```

**Cloud Equivalent (AWS EC2 with user data):**
```bash
#!/bin/bash
# EC2 User Data Script
echo "Hello from cloud instance: $(hostname)" > /tmp/hello.log
echo "Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)" >> /tmp/hello.log
echo "Running at: $(date)" >> /tmp/hello.log
sleep 60
echo "Job completed at: $(date)" >> /tmp/hello.log
# Instance continues running until terminated
```

**Cloud Serverless (AWS Lambda):**
```python
import json
import datetime
import time

def lambda_handler(event, context):
    print(f"Hello from Lambda function")
    print(f"Function name: {context.function_name}")
    print(f"Running at: {datetime.datetime.now()}")
    
    # Simulate work
    time.sleep(5)  # Lambda has time limits
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Function completed',
            'timestamp': datetime.datetime.now().isoformat()
        })
    }
```

#### Key Differences Analysis

**Resource Lifecycle:**
- **HPC**: Submit → Queue → Run → Complete → Resources freed
- **Cloud IaaS**: Launch → Run continuously → Terminate when done
- **Cloud PaaS/Serverless**: Trigger → Execute → Auto-terminate

**Cost Model:**
- **HPC**: Fixed cost, maximize utilization
- **Cloud**: Variable cost, optimize for actual usage

---

## Day 3: Cloud Storage Models

### Morning Session (3 hours)

#### Storage Architecture Comparison

**HPC Storage vs. Cloud Storage:**

| HPC Storage | Cloud Equivalent | Key Difference |
|-------------|------------------|----------------|
| Parallel filesystem (GPFS/Lustre) | Block storage (EBS) + File storage (EFS) | Single massive filesystem vs. multiple storage types |
| /home directories | Persistent block storage | Personal vs. instance-attached |
| /scratch space | Ephemeral storage | High-performance temporary vs. instance-local |
| Tape archives | Object storage (S3) + Glacier | Hierarchical vs. flat namespace |
| Shared project space | Shared file systems (EFS/FSx) | POSIX filesystem vs. managed service |

#### Cloud Storage Types Deep Dive

**Object Storage (S3, Azure Blob, Google Cloud Storage):**
- **HPC equivalent**: Like a massive, globally accessible tape archive
- **Characteristics**: Flat namespace, REST API access, unlimited scale
- **Use cases**: Backup, archival, static websites, data lakes
- **Security**: Bucket policies, IAM, encryption at rest/transit

**Block Storage (EBS, Azure Disks, Persistent Disks):**
- **HPC equivalent**: Like a dedicated disk attached to a compute node
- **Characteristics**: Raw block device, high IOPS, attached to single instance
- **Use cases**: Database storage, file systems, any high-performance storage
- **Security**: Encryption, snapshots, access controls

**File Storage (EFS, Azure Files, Filestore):**
- **HPC equivalent**: Like NFS-mounted shared storage
- **Characteristics**: POSIX-compliant, shared across instances, managed service
- **Use cases**: Shared application data, content repositories, home directories
- **Security**: Network access controls, encryption, POSIX permissions

### Afternoon Session (2 hours)

#### Hands-On Storage Exploration

**Storage Service Comparison:**
```bash
# HPC storage commands (what you know)
df -h /home /scratch /projects
ls -la /scratch/
quota -u $USER

# Cloud storage equivalents
# AWS S3 (object storage)
aws s3 ls s3://my-bucket/
aws s3 cp file.txt s3://my-bucket/

# AWS EFS (file storage)
mount -t efs fs-12345678:/ /mnt/efs
ls -la /mnt/efs/

# Check storage costs and performance
aws s3api get-bucket-location --bucket my-bucket
aws ec2 describe-volumes --volume-ids vol-12345678
```

**Storage Security Models:**
- **HPC**: Unix permissions, ACLs, physical security
- **Cloud**: IAM policies, bucket policies, encryption, network controls

#### Data Movement Patterns

**HPC Data Movement:**
- Globus for high-performance transfers
- SCP/SFTP for smaller files
- Dedicated data transfer nodes
- Research network connections

**Cloud Data Movement:**
- REST APIs for programmatic access
- Web consoles for manual operations
- CDNs for global distribution
- Direct connect for high-volume transfers

---

## Day 4: Cloud Security Models

### Morning Session (3 hours)

#### Shared Responsibility Model

**The Fundamental Difference:**
- **HPC**: You're responsible for everything (hardware, OS, applications, data)
- **Cloud**: Responsibility is shared between you and the cloud provider

**Shared Responsibility Breakdown:**

| Component | HPC (You) | Cloud IaaS (Shared) | Cloud PaaS (Provider More) | Cloud SaaS (Provider Most) |
|-----------|-----------|---------------------|----------------------------|----------------------------|
| **Physical Security** | You | Provider | Provider | Provider |
| **Hardware** | You | Provider | Provider | Provider |
| **Hypervisor** | You | Provider | Provider | Provider |
| **Operating System** | You | You | Provider | Provider |
| **Applications** | You | You | You/Provider | Provider |
| **Data** | You | You | You | You |
| **Identity & Access** | You | You | You | You |
| **Network Controls** | You | Shared | Shared | Provider |

#### Cloud-Native Security Services

**Identity and Access Management (IAM):**
- **HPC equivalent**: Like LDAP/Active Directory but for everything
- **Capabilities**: Users, roles, policies, temporary credentials
- **Key difference**: API-driven, fine-grained, service-specific

**Network Security:**
- **Virtual Private Clouds (VPCs)**: Software-defined networks
- **Security Groups**: Instance-level firewalls
- **Network ACLs**: Subnet-level firewalls
- **HPC equivalent**: Physical network segmentation + iptables

**Encryption and Key Management:**
- **Managed encryption services**: Hardware Security Modules (HSMs)
- **Key rotation and lifecycle management**
- **Integration with all services**
- **HPC equivalent**: Manual key management + software encryption

### Afternoon Session (2 hours)

#### Hands-On Security Exploration

**IAM Concepts (AWS example):**
```bash
# List users and roles
aws iam list-users
aws iam list-roles

# Check your permissions
aws iam get-user
aws sts get-caller-identity

# Example policy (JSON)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Network Security (AWS example):**
```bash
# List VPCs and security groups
aws ec2 describe-vpcs
aws ec2 describe-security-groups

# Example security group rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr 10.0.0.0/8
```

#### Security Monitoring and Compliance

**Cloud Security Monitoring:**
- **CloudTrail**: API call logging (like audit logs)
- **GuardDuty**: Threat detection (like IDS/IPS)
- **Security Hub**: Centralized security findings
- **Config**: Configuration compliance monitoring

**HPC vs. Cloud Monitoring:**
- **HPC**: Manual log collection, custom SIEM integration
- **Cloud**: Built-in logging, managed security services, automated compliance

---

## Day 5: Hands-On Cloud Environment Exploration

### Morning Session (3 hours)

#### Cloud Account Setup and Navigation

**Getting Started (if you have access):**
1. **AWS**: Create free tier account, explore console
2. **Azure**: Create free account, explore portal
3. **Google Cloud**: Create free tier, explore console

**Key Areas to Explore:**
- **Compute services**: EC2, Lambda, containers
- **Storage services**: S3, EBS, databases
- **Networking**: VPC, load balancers, CDN
- **Security services**: IAM, encryption, monitoring
- **Management tools**: CloudFormation, monitoring, billing

#### Basic Cloud Operations

**Launch Your First Cloud Instance:**
```bash
# AWS EC2 example
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --count 1 \
  --instance-type t2.micro \
  --key-name my-key-pair \
  --security-group-ids sg-12345678

# Check instance status
aws ec2 describe-instances --instance-ids i-1234567890abcdef0

# Connect to instance
ssh -i my-key.pem ec2-user@public-ip-address
```

**Create Cloud Storage:**
```bash
# AWS S3 example
aws s3 mb s3://my-unique-bucket-name
aws s3 cp local-file.txt s3://my-unique-bucket-name/
aws s3 ls s3://my-unique-bucket-name/
```

### Afternoon Session (1 hour)

#### Document Your Cloud Learning

**Create Your Cloud Translation Guide:**
- Map HPC concepts to cloud equivalents you've discovered
- Note key differences and their implications
- Identify areas where cloud might be better/worse than HPC
- Document questions for further exploration

**Cloud vs. HPC Comparison Matrix:**

| Function | HPC Approach | Cloud Approach | Pros/Cons |
|----------|--------------|----------------|-----------|
| **Compute** | Job scheduler + fixed nodes | On-demand instances + auto-scaling | Cloud: More flexible, HPC: More efficient |
| **Storage** | Parallel filesystem | Multiple storage types | Cloud: More options, HPC: Better performance |
| **Security** | Full ownership | Shared responsibility | Cloud: Managed services, HPC: Full control |
| **Cost** | Fixed infrastructure cost | Pay-per-use | Cloud: Variable cost, HPC: Predictable cost |

---

## Week 1 Deliverable: Cloud Computing Assessment

Create a 1-2 page document covering:

### Section 1: Cloud Model Understanding
- Key differences between cloud and HPC computing models
- Advantages and disadvantages of each approach
- When to use cloud vs. HPC for different workloads

### Section 2: Service Model Analysis
- Understanding of IaaS, PaaS, and SaaS models
- How each relates to current HPC operations
- Security implications of each service model

### Section 3: Initial Cloud Security Assessment
- Shared responsibility model understanding
- Key cloud security services and their HPC equivalents
- Areas where cloud security differs from HPC security

### Section 4: Learning Reflections
- What surprised you most about cloud computing?
- What HPC practices don't apply to cloud?
- What new opportunities do you see in cloud?
- What concerns do you have about cloud adoption?

---

## Resources for Week 1

### Essential Reading
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- Cloud provider free tier documentation

### Optional Deep Dives
- "Cloud Computing: Concepts, Technology & Architecture" (book)
- Cloud provider certification study guides
- NIST Cloud Computing Definition (SP 800-145)

### People to Talk To
- Organizational cloud architects or engineers
- Colleagues who have moved from on-premises to cloud
- Cloud vendor representatives or solutions architects
- Other HPC professionals exploring cloud

---

## Success Metrics for Week 1

You should be able to:
- [ ] Explain cloud computing models to an HPC colleague
- [ ] Identify 3 key differences between cloud and HPC approaches
- [ ] Navigate a cloud provider console and basic services
- [ ] Understand the shared responsibility model
- [ ] Translate basic HPC concepts to cloud equivalents

---

## Troubleshooting Common Week 1 Challenges

**"Cloud seems too complex with so many services"**
- Focus on core services first (compute, storage, networking)
- Use the free tiers to get hands-on experience
- Don't try to understand everything at once
- Connect new concepts to HPC equivalents you know

**"I can't get access to a cloud account"**
- Use free tier accounts from major providers
- Explore cloud provider documentation and tutorials
- Use cloud simulators or sandbox environments
- Focus on conceptual understanding initially

**"The cost model is confusing compared to HPC"**
- Start with simple examples and calculators
- Focus on the pay-per-use concept vs. fixed costs
- Use cost monitoring tools to understand spending
- Remember that cloud optimizes for flexibility, not just cost

**"Security seems less controlled than HPC"**
- Focus on the shared responsibility model benefits
- Understand that you gain managed security services
- Learn about cloud-native security tools
- Remember that cloud providers invest heavily in security

---

## Week 1 Wrap-Up

By the end of week 1, you should have a solid conceptual understanding of cloud computing and how it differs from HPC. You're not trying to become a cloud expert overnight - you're building enough understanding to apply your HPC knowledge effectively in cloud environments.

The key insight: Cloud computing is a different paradigm that prioritizes flexibility, scalability, and managed services over the performance and control that HPC emphasizes. Both have their place, and understanding the trade-offs is crucial.

**Next week**: We'll dive into cloud identity and access management - understanding how IAM differs from traditional Unix permissions and LDAP systems you know from HPC.