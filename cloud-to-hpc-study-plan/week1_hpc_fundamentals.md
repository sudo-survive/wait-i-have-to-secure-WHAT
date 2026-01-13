# Week 1: Core HPC Concepts
## Understanding What Makes HPC Different from Cloud Computing

---

## Learning Objectives

By the end of this week, you should be able to:
- Explain the fundamental differences between HPC and cloud computing architectures
- Identify the key security implications of HPC's unique characteristics
- Understand why traditional cloud security approaches need adaptation for HPC
- Speak intelligently about HPC concepts in security discussions

---

## Day 1: HPC Architecture Basics

### Morning Session (2-3 hours)

#### Core Concepts to Learn

**1. HPC System Components**
- **Login nodes**: Where users SSH to access the cluster (like bastion hosts, but everyone uses them)
- **Compute nodes**: Where jobs actually run (users never directly access these)
- **Storage nodes**: Serve the parallel filesystem
- **Management nodes**: Run job scheduler, monitoring, etc.

**2. Key Architectural Differences from Cloud**

| Aspect | Cloud | HPC |
|--------|-------|-----|
| **Compute Model** | On-demand VMs | Fixed cluster, batch jobs |
| **User Access** | Direct to instances | Through login nodes only |
| **Resource Allocation** | Self-service | Centrally scheduled |
| **Network** | Software-defined | Physical high-speed interconnects |
| **Storage** | Multiple services | One big parallel filesystem |

#### Security Translation Exercise

For each HPC component, identify the cloud equivalent and security implications:

**Login Nodes = Bastion Hosts**
- *Security implication*: Single point of failure, must be heavily hardened
- *Cloud difference*: In cloud, you might have multiple bastion hosts; in HPC, often just 1-2 login nodes for entire cluster

**Compute Nodes = EC2 Instances**
- *Security implication*: Shared sequentially between users, data cleanup critical
- *Cloud difference*: In cloud, you get dedicated instances; in HPC, nodes are shared over time

#### Hands-On Activity
- Draw a simple HPC architecture diagram
- Label each component with its cloud equivalent
- Note security boundaries and trust relationships

### Afternoon Session (1-2 hours)

#### Reading Assignment
- NIST HPC Security Workshop Overview: https://csrc.nist.gov/projects/high-performance-computing-security
- Focus on: "What makes HPC security different from enterprise IT security?"

#### Reflection Questions
1. What assumptions from cloud security don't apply to HPC?
2. Where are the new attack surfaces in HPC vs. cloud?
3. What security controls from cloud computing would be hardest to implement in HPC?

---

## Day 2: Parallel Processing and MPI Fundamentals

### Morning Session (2 hours)

#### Conceptual Understanding (Not Deep Technical)

**What is Parallel Processing?**
- Breaking big problems into smaller pieces
- Running pieces simultaneously on many cores/nodes
- Combining results back together

**Message Passing Interface (MPI)**
- How parallel programs communicate between nodes
- Think of it like microservices talking to each other, but MUCH more chatty
- Generates tons of network traffic between compute nodes

#### Security Implications

**1. Network Traffic Patterns**
- Normal HPC jobs generate massive amounts of inter-node communication
- This makes detecting malicious network activity harder
- "Unusual network traffic" baselines are very different in HPC

**2. Process Coordination**
- MPI jobs spawn processes across many nodes simultaneously
- Process monitoring tools need to understand this is normal
- Malicious processes could hide among legitimate MPI processes

**3. Shared Memory and Data**
- Parallel jobs often share data structures across nodes
- Data isolation between jobs becomes more complex
- Memory cleanup between jobs is critical

#### Cloud Translation
- **MPI communication** = Like having hundreds of Lambda functions that need to constantly talk to each other
- **Parallel job** = Like a Kubernetes job that spans multiple nodes and requires tight coordination

### Afternoon Session (1 hour)

#### Practical Exercise
- Look up your HPC center's job examples
- Identify which ones use MPI (parallel) vs. serial processing
- Note the resource requests (how many nodes, cores, memory)

#### Security Checklist Development
Start building your "HPC Security Questions" list:
- How do we monitor inter-node MPI traffic for anomalies?
- What does normal vs. suspicious parallel job behavior look like?
- How do we ensure data isolation between parallel jobs?

---

## Day 3: Interconnect Fabrics (InfiniBand/Omni-Path)

### Morning Session (2 hours)

#### Why High-Speed Networks Matter

**The Problem HPC Solves:**
- Scientific simulations need to process HUGE amounts of data
- Thousands of CPU cores need to coordinate constantly
- Regular Ethernet would be too slow and high-latency

**InfiniBand/Omni-Path Characteristics:**
- 100-200 Gbps speeds (vs. 1-10 Gbps Ethernet)
- Microsecond latency (vs. millisecond Ethernet)
- Hardware-level message passing
- Bypasses kernel networking stack for performance

#### Security Implications

**1. Limited Security Controls**
- InfiniBand often has minimal built-in security
- "It's internal network" mentality = less security focus
- Hardware-level communication harder to inspect

**2. Lateral Movement Risk**
- High-speed interconnect connects ALL compute nodes
- Compromised node could potentially reach any other node
- Traditional network security appliances can't inspect InfiniBand traffic

**3. Monitoring Challenges**
- Specialized tools needed to monitor InfiniBand traffic
- High-speed networks generate massive amounts of log data
- Performance impact of monitoring is a bigger concern

#### Cloud Translation
- **InfiniBand** = Like AWS Placement Groups with dedicated 100Gbps networking, but physical hardware
- **Security challenge** = Like having a flat network between all your instances with minimal firewall controls

### Afternoon Session (1 hour)

#### Research Your Environment
- Find out what interconnect your HPC center uses
- Identify what security monitoring exists for the high-speed network
- Note any network segmentation or isolation controls

#### Add to Security Questions List
- What monitoring exists for InfiniBand/high-speed network traffic?
- How do we detect lateral movement across the interconnect?
- Are there any network segmentation controls on the compute network?

---

## Day 4: Scientific Workflows vs. Cloud Workloads

### Morning Session (2 hours)

#### Understanding Scientific Computing Patterns

**Typical Scientific Workflows:**
1. **Data ingestion**: Load large datasets (terabytes)
2. **Preprocessing**: Clean, format, subset data
3. **Simulation/Analysis**: Run compute-intensive algorithms
4. **Postprocessing**: Analyze results, generate visualizations
5. **Data archival**: Store results for future use

**Characteristics:**
- Long-running (hours to weeks)
- Resource-intensive (thousands of cores, terabytes of RAM)
- Data-intensive (terabytes of I/O)
- Iterative (run many variations of same analysis)

#### Comparison to Cloud Workloads

| Aspect | Cloud Workloads | Scientific Workloads |
|--------|----------------|---------------------|
| **Duration** | Minutes to hours | Hours to weeks |
| **Resource Usage** | Bursty, variable | Sustained, predictable |
| **Data Patterns** | Request/response | Bulk processing |
| **Failure Handling** | Retry, graceful degradation | Checkpoint/restart |
| **Scaling** | Horizontal (more instances) | Vertical (more cores per job) |

#### Security Implications

**1. Baseline Establishment**
- "Normal" resource usage is MUCH higher in HPC
- Jobs that consume 1000 cores for a week are legitimate
- Traditional "resource abuse" detection doesn't work

**2. Data Handling**
- Scientific data often has export control restrictions
- Data sharing with external collaborators is common
- Data retention requirements may be decades

**3. User Behavior**
- Scientists run similar jobs repeatedly with variations
- Trial-and-error approach to optimization
- Less security-conscious than typical IT users

### Afternoon Session (1 hour)

#### Case Study Analysis
Pick 2-3 scientific domains relevant to your HPC center:
- Climate modeling
- Genomics/bioinformatics  
- Physics simulations
- Machine learning

For each, identify:
- Typical resource requirements
- Data sensitivity levels
- External collaboration needs
- Compliance requirements

#### Security Implications Summary
Document how scientific workflows change your security approach:
- Monitoring baselines
- Data classification
- Access controls
- Incident response

---

## Day 5: Hands-On Environment Exploration

### Morning Session (3 hours)

#### Get Access to Your HPC System
- Request test/training account if you don't have one
- SSH to login node
- Explore the environment

#### Basic Commands to Try
```bash
# See system information
uname -a
cat /proc/version

# Check available modules
module avail

# See filesystem layout
df -h
ls /scratch
ls /home
ls /projects

# Check job scheduler
squeue  # (if Slurm)
qstat   # (if PBS)

# See who's using the system
w
users
```

#### Document What You Find
Create a simple inventory:
- What job scheduler is used?
- What filesystems are available?
- What software modules are installed?
- How many users are typically logged in?
- What does normal system activity look like?

### Afternoon Session (1 hour)

#### Security Reconnaissance
Look for:
- Security monitoring tools
- Log locations
- Authentication methods
- Network configuration
- Backup/archival systems

#### Questions to Ask HPC Staff
- What security incidents have occurred?
- What monitoring is currently in place?
- What are the biggest security concerns?
- What compliance requirements apply?
- What security controls are users complaining about?

---

## Week 1 Deliverable: HPC Architecture Summary

Create a 1-2 page document covering:

### Section 1: Architecture Overview
- Diagram of your HPC center's architecture
- Key components and their purposes
- How users interact with the system

### Section 2: Cloud Translation
- Map each HPC component to cloud equivalent
- Highlight key differences
- Note security implications of differences

### Section 3: Initial Security Assessment
- Current security controls you've identified
- Obvious gaps or concerns
- Questions that need answers

### Section 4: Learning Reflections
- What surprised you most about HPC?
- What cloud security practices don't apply?
- What new security challenges do you see?

---

## Resources for Week 1

### Essential Reading
- [NIST HPC Security Workshop Materials](https://csrc.nist.gov/projects/high-performance-computing-security)
- Your HPC center's user documentation
- System architecture diagrams (request from HPC staff)

### Optional Deep Dives
- "Introduction to High Performance Computing" (free online course)
- MPI tutorials (conceptual understanding only)
- InfiniBand architecture overviews

### People to Talk To
- HPC system administrators
- User support staff
- Other security professionals at your organization
- Scientists who use the system regularly

---

## Success Metrics for Week 1

You should be able to:
- [ ] Explain HPC architecture to a cloud security colleague
- [ ] Identify 3 key differences between HPC and cloud security
- [ ] Navigate your HPC system's basic interface
- [ ] Ask intelligent questions about HPC security controls
- [ ] Translate HPC concepts to cloud equivalents

---

## Troubleshooting Common Week 1 Challenges

**"This is overwhelming - there's so much to learn"**
- Focus on concepts, not details
- You don't need to become an HPC expert
- Connect everything back to cloud concepts you know

**"I can't get access to the HPC system"**
- Work with HPC staff to get training account
- Use documentation and diagrams in the meantime
- Shadow an HPC user session if possible

**"The HPC staff are too busy to help"**
- Schedule brief 30-minute sessions instead of long meetings
- Come with specific questions
- Offer to help with security tasks in exchange for knowledge

**"I don't understand the scientific applications"**
- You don't need to understand the science
- Focus on the computing patterns and resource usage
- Think about security implications, not scientific accuracy

---

## Week 1 Wrap-Up

By the end of week 1, you should have a solid conceptual understanding of HPC architecture and how it differs from cloud computing. You're not trying to become an HPC expert - you're building enough understanding to apply your security expertise effectively.

The key insight: HPC is just a different computing model with some unique characteristics. Your security knowledge still applies, but the implementation details are different.

**Next week**: We'll dive into job schedulers and resource management - the heart of how HPC systems actually work.