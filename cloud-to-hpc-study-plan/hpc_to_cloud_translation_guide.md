# HPC to Cloud Translation Guide
## For Security Professionals Who Know Cloud But Not HPC

---

## Introduction

You already know cloud computing security. You've secured multi-tenant platforms, managed AWS/Azure environments, and built compliance frameworks. HPC isn't a completely different world - it's just a different computing paradigm with some unique characteristics.

This guide translates every major HPC concept into cloud computing terms you already understand, then explains what's different and why it matters for security.

---

## Core Computing Model

### **Compute Resources**

#### Cloud Computing:
- **EC2 instances / VMs** that you spin up on-demand
- You pick instance type, OS, and configuration
- Elastic - scale up/down as needed
- Pay for what you use
- Each instance is relatively isolated

#### HPC (Example System):
- **Compute nodes** in a fixed cluster
- Pre-configured by admins, users don't pick OS
- Fixed capacity - the cluster is what it is
- Users submit jobs to a queue, wait for resources
- Nodes are shared sequentially (jobs run, then node is freed for next job)

**Security Translation:**
- **Cloud:** Isolation between VMs is handled by hypervisor
- **HPC:** Isolation between jobs is handled by job scheduler (Slurm) and OS-level controls
- **Risk:** In HPC, one user's job runs on the same physical hardware that just ran someone else's job. Data cleanup between jobs is critical.

---

### **Job/Workload Model**

#### Cloud Computing:
- **Long-running services:** Web servers, databases, APIs that run continuously
- **Event-driven functions:** Lambda/Azure Functions that execute on triggers
- **Batch jobs:** Can use batch services, but often run on persistent instances
- You start it, it runs until you stop it (or it crashes)

#### HPC (Discover):
- **Batch jobs ONLY:** Every workload is a submitted job with defined start and end
- Jobs specify: how many nodes needed, how long it will run, what resources required
- **Job queue:** Jobs wait in queue until resources are available
- Jobs are scheduled based on priority, resource requirements, and queue policies
- When job completes (or times out), nodes are freed for next job

**Security Translation:**
- **Cloud:** Monitor running services for anomalies (long-term baselines)
- **HPC:** Monitor job submission patterns, resource requests, and job behavior (transient workloads)
- **Risk:** In HPC, malicious activity often appears as "weird job requests" rather than "compromised running service"

**This is like:** Cloud Batch or Kubernetes jobs, but at massive scale and with MUCH more resource coordination required

---

### **Resource Allocation**

#### Cloud Computing:
- **Auto-scaling groups:** Automatically add/remove instances based on load
- **Instance sizing:** Pick t2.micro vs. c5.24xlarge
- **Spot instances:** Bid for unused capacity
- Pay-per-use model

#### HPC (Example System):
- **Fixed cluster size:** You have X nodes, period
- **Job scheduler (Slurm/PBS/LSF):** Assigns available nodes to waiting jobs
- **Partitions/Queues:** Different node groups for different purposes (like "small jobs" vs "large jobs")
- **Fair-share scheduling:** Ensures no single user/project monopolizes resources
- **Allocations:** Users get X core-hours per quarter, tracked and enforced

**Security Translation:**
- **Cloud:** Monitor spend/usage to detect abuse
- **HPC:** Monitor allocations, job queue patterns, and resource requests to detect abuse
- **Risk:** Resource exhaustion attacks in HPC = submitting tons of jobs to block other users, or requesting resources you don't actually need

**This is like:** Kubernetes resource quotas and limits, but enforced by a centralized scheduler rather than per-namespace

---

## Storage Architecture

### **Storage Types**

#### Cloud Computing (AWS example):
- **S3 / Object Storage:** Cheap, slow, great for archives and big files
- **EBS / Block Storage:** Fast, expensive, attached to specific VMs
- **EFS / Shared File Storage:** Mounted by multiple instances, NFS-like
- **Glacier / Archive:** Super cheap, very slow retrieval

#### HPC (Example System):
- **Home directories:** Small, backed up, permanent (like your EBS root volume)
- **Scratch/Nobackup:** Large, fast, shared parallel filesystem (GPFS/Lustre), NOT backed up, temporary
- **Project space:** Shared scratch for teams
- **Centralized Storage:** Curated datasets everyone can read (like S3 public datasets)
- **Tape archive:** Long-term storage, slow retrieval (like Glacier)

**Key Difference:**
In cloud, each storage type is its own service. In HPC, they're all part of ONE big parallel filesystem (GPFS/Spectrum Scale), just with different policies and mount points.

**Security Translation:**
- **Cloud:** Secure S3 with bucket policies, encrypt EBS volumes, manage EFS access with security groups
- **HPC:** Secure GPFS with Unix permissions, ACLs, and quotas - it's all ONE filesystem with different "areas"
- **Risk:** In HPC, a compromised account can potentially traverse the entire filesystem unless permissions are strictly enforced

**This is like:** Having a giant NFS mount with different directories that have different performance characteristics and backup policies

---

### **Parallel File System (GPFS)**

#### Cloud Computing:
- Multiple storage services, each accessed via API (S3) or mount (EFS)
- Storage and compute are loosely coupled
- Network attached storage (NAS) if you need shared access

#### HPC (GPFS/Lustre/BeeGFS):
- ONE massive parallel filesystem mounted on ALL compute nodes
- **Designed for concurrent access:** Thousands of nodes reading/writing simultaneously
- **Metadata servers:** Track where files are (like a distributed database index)
- **Storage servers:** Actually hold the data chunks
- **High-performance network:** InfiniBand connecting it all
- Data is striped across multiple storage servers for parallel I/O

**Why This Exists:**
Scientific applications need to read/write HUGE datasets from many nodes at once. Regular NFS would fall over.

**Security Translation:**
- **Cloud:** Each storage service has its own access controls and encryption
- **HPC:** One filesystem = one security boundary, must rely heavily on Unix permissions
- **Risk:** Metadata server compromise could expose all file locations; storage server compromise could expose data chunks

**This is like:** A distributed database (like Cassandra) but for files instead of data records - and every compute node is a client

---

### **Data Movement**

#### Cloud Computing:
- **Within region:** Fast, often free
- **Between regions:** Slower, costs money
- **To/from internet:** Egress fees, rate limits
- **Tools:** aws s3 sync, rsync, SCP, SFTP

#### HPC (Example System):
- **Internal movement:** Between storage systems (scratch to archive) - mediated by admins
- **Data transfer nodes:** Special nodes with external connectivity for moving data in/out
- **Globus:** High-performance data transfer tool (like rsync on steroids)
- **External transfers:** To/from other research institutions, cloud, or laptops

**Security Translation:**
- **Cloud:** Monitor egress, enforce encryption in transit, watch for data exfiltration via API
- **HPC:** Monitor data transfer nodes, watch for unusual Globus transfers, enforce use of approved transfer methods
- **Risk:** Data exfiltration in HPC often happens via data transfer nodes or by copying to external collaborators

**This is like:** Having designated NAT gateways for outbound traffic, except for data movement

---

## Networking

### **Network Architecture**

#### Cloud Computing:
- **VPCs:** Software-defined networks with subnets, route tables, security groups
- **Internet Gateway:** Controlled access to public internet
- **VPN/Direct Connect:** Private connectivity to on-prem
- **Standard Ethernet:** 1-100 Gbps typical
- **Firewall-based security:** Security groups, NACLs, WAFs

#### HPC (Example System):
- **Multiple networks:** Management network, InfiniBand compute network, external network (all separate)
- **InfiniBand:** Ultra high-speed interconnect (100-200 Gbps) connecting compute nodes
- **Low latency critical:** Measured in microseconds, not milliseconds
- **Physical network segmentation:** Different networks for different purposes
- **Limited external connectivity:** Most nodes have NO direct internet access

**Why InfiniBand:**
Parallel computing requires nodes to send TONS of messages to each other (MPI communication). Regular Ethernet would be too slow and high-latency.

**Security Translation:**
- **Cloud:** Software-defined security (security groups filter traffic)
- **HPC:** Physical + software security (networks are physically separate, plus OS-level controls)
- **Risk:** InfiniBand networks often have minimal security controls because "it's all internal" - but lateral movement risk exists

**This is like:** Having separate VPCs for different purposes, but they're physically different networks, not just software-defined

---

### **Network Access**

#### Cloud Computing:
- **Public IPs:** Instances can have public IPs
- **Elastic IPs:** Static public IPs
- **Load Balancers:** Distribute traffic to backends
- **Bastion hosts:** Jump boxes for admin access
- Users SSH directly to instances (if allowed)

#### HPC (Example System):
- **Login nodes:** Special gateway nodes where users SSH to access the cluster
- **Compute nodes:** NO direct user access, only job scheduler can submit work
- **Data transfer nodes:** Special nodes for moving data in/out
- **Admin nodes:** Only admins can access for maintenance
- Regular users NEVER touch compute nodes directly

**Security Translation:**
- **Cloud:** Harden each instance that's internet-facing
- **HPC:** Harden login nodes heavily (they're the entry point), compute nodes are more protected
- **Risk:** Compromised login node = access to the entire cluster's filesystem and ability to submit malicious jobs

**This is like:** Bastion hosts, but everyone uses them and they're the ONLY way in

---

### **Authentication**

#### Cloud Computing:
- **IAM:** Identity and Access Management with roles and policies
- **Service accounts:** For applications to authenticate
- **API keys:** For programmatic access
- **MFA:** Multi-factor authentication optional but recommended

#### HPC (Example System):
- **Unix accounts:** Traditional user accounts on all nodes
- **SSH keys:** Primary authentication method
- **LDAP/Active Directory:** Centralized user directory
- **Kerberos:** Sometimes used for additional authentication
- **Two-factor:** May or may not exist (often needs to be added)

**Security Translation:**
- **Cloud:** IAM policies define who can do what
- **HPC:** Unix permissions + Slurm account limits define who can do what
- **Risk:** SSH key management at scale is hard; lost/stolen keys are common; privilege escalation via sudo misconfigurations

**This is like:** EC2 instances using SSH keys and IAM instance profiles, but there's no cloud IAM - it's all traditional Unix/Linux

---

## User Model

### **Access Patterns**

#### Cloud Computing:
- **Developers:** Deploy code, manage infrastructure via API/console
- **Applications:** Run continuously, serve requests
- **Admins:** Manage cloud resources via console/CLI
- Access is typically through APIs and web interfaces

#### HPC (Example System):
- **Researchers/Scientists:** The actual users
- **Submit jobs:** Write job scripts, submit to queue, wait for results
- **Develop code:** Write/compile scientific applications on login nodes
- **Analyze results:** Download data from scratch space, process locally or on cluster
- **Long-running analysis:** Jobs might run for days or weeks

**Security Translation:**
- **Cloud:** Monitor API calls, service logs, application logs
- **HPC:** Monitor SSH sessions, job submissions, file access, and job behavior
- **Risk:** Insider threat is bigger concern - these are scientists with legitimate access doing legitimate work, but might misuse resources

**This is like:** A mix between a CI/CD environment (submitting jobs) and a development environment (writing code)

---

### **User Support**

#### Cloud Computing:
- **Self-service:** Users provision their own resources
- **Documentation:** How to use AWS/Azure services
- **Support tickets:** For when things break
- Minimal hand-holding expected

#### HPC (Example System):
- **High-touch support:** Users need help compiling code, optimizing jobs, understanding errors
- **User services team:** Dedicated support staff
- **Training:** Regular workshops and office hours
- **Consulting:** Help users optimize scientific applications
- Users expect a lot of support (they're scientists, not IT professionals)

**Security Translation:**
- **Cloud:** Security is often users' problem (shared responsibility model)
- **HPC:** Users expect security to be transparent and not slow down their science
- **Risk:** Balancing security with usability is HARD - scientists will route around security that gets in their way

**This is like:** Managed services where users expect things to "just work" and get frustrated when security adds friction

---

## Software Environment

### **Applications**

#### Cloud Computing:
- **Containers:** Docker images with your application
- **AMIs:** Pre-configured machine images
- **Configuration management:** Ansible, Puppet, Chef
- Standard OS packages (yum, apt)
- You control the entire software stack

#### HPC (Example System):
- **Modules:** Environment management system (like Python virtualenvs but for everything)
- **Compilers:** Multiple versions of GCC, Intel, PGI compilers
- **Scientific libraries:** MPI, BLAS, LAPACK, HDF5, NetCDF, etc.
- **Pre-built applications:** Climate models, physics simulators, bioinformatics tools
- Users don't install software system-wide - they load modules or install in home directory

**Module System Example:**
```bash
module load gcc/11.2.0
module load openmpi/4.1.1
module load netcdf/4.8.1
```
This sets up environment variables so the user has access to specific versions of tools.

**Security Translation:**
- **Cloud:** Scan container images, patch AMIs, manage dependencies
- **HPC:** Scan module files, patch system packages, manage scientific software vulnerabilities (which often have NO CVEs)
- **Risk:** Scientific software is notoriously insecure and unmaintained; legacy code from the 1990s still in use

**This is like:** Having a package manager that doesn't install system-wide, plus tons of legacy applications you can't easily update

---

### **Development Environment**

#### Cloud Computing:
- **CI/CD pipelines:** Automated build, test, deploy
- **Version control:** Git repos, pull requests, code review
- **Testing:** Unit tests, integration tests before deployment

#### HPC (Discover):
- **Login nodes:** Where users compile code
- **Test jobs:** Submit small jobs to test before running big jobs
- **Trial and error:** Lots of debugging by submitting jobs and checking results
- **Scientific workflow tools:** Increasingly using things like Snakemake, Nextflow
- Version control less common (but improving)

**Security Translation:**
- **Cloud:** Scan code in CI/CD, enforce security checks before deploy
- **HPC:** Limited ability to scan user code before execution; rely on post-execution monitoring
- **Risk:** Users compile and run custom code constantly; detecting malicious code is very hard

**This is like:** Letting users run arbitrary code without much automated security scanning (scary, I know)

---

## Security Primitives

### **Access Control**

#### Cloud Computing:
- **IAM policies:** JSON documents defining permissions
- **Security groups:** Firewall rules for instances
- **Resource tags:** For organizing and controlling access
- **Service Control Policies:** Organization-wide restrictions

#### HPC (Example System):
- **Unix permissions:** rwxrwxrwx on files and directories
- **ACLs:** Extended permissions for more granular control
- **Job scheduler accounts:** Control who can submit jobs to which queues
- **Group membership:** Unix groups for shared project access
- **Quotas:** Disk space and file count limits

**Security Translation:**
- **Cloud:** Fine-grained, attribute-based access control
- **HPC:** Coarse-grained, identity-based access control
- **Risk:** HPC access controls are much less flexible; group explosion is common

**This is like:** Going back to traditional Linux permissions instead of cloud IAM

---

### **Monitoring & Logging**

#### Cloud Computing:
- **CloudWatch/Azure Monitor:** Centralized metrics and logs
- **CloudTrail:** Audit log of all API calls
- **VPC Flow Logs:** Network traffic logs
- **Application logs:** Whatever your app writes
- Rich dashboards and alerting

#### HPC (Example System):
- **Job scheduler accounting:** Every job submitted and its resource usage
- **System logs:** Standard Linux logs (syslog, auth.log, etc.)
- **Filesystem logs:** Filesystem operations and errors
- **Application logs:** Scientific applications write to stdout/stderr, captured with job
- Often logs are scattered across many systems

**Security Translation:**
- **Cloud:** Centralized logging is standard
- **HPC:** You'll probably need to BUILD centralized logging (use your Kibana/Elasticsearch/etc skills)
- **Risk:** Without centralized logging, detecting attacks across the cluster is nearly impossible

**This is like:** Managing logging for hundreds of EC2 instances with local logs - you need to ship them somewhere central

---

### **Encryption**

#### Cloud Computing:
- **At-rest:** EBS encryption, S3 encryption (KMS keys)
- **In-transit:** TLS everywhere, VPN for private connectivity
- **Key management:** KMS, HSMs, key rotation policies

#### HPC (Example System):
- **At-rest:** Often NONE by default (performance penalty)
- **In-transit:** SSH for user access, but InfiniBand traffic usually unencrypted
- **Key management:** Manual SSH key management
- Encryption is often seen as "too slow" for HPC workloads

**Security Translation:**
- **Cloud:** Encryption everywhere is standard
- **HPC:** Encryption is rare; you'll face pushback about performance
- **Risk:** Data at rest and in transit is often unencrypted; this will be a modernization battle

**This is like:** Pre-2015 cloud computing before encryption became default

---

## Workload Characteristics

### **Compute Patterns**

#### Cloud Computing:
- **Web services:** Responding to HTTP requests
- **Databases:** CRUD operations on data
- **Event processing:** Responding to messages/events
- **Batch processing:** ETL jobs, data processing
- Highly variable, often bursty

#### HPC (Example System):
- **Simulations:** Physics, chemistry, climate models
- **Data analysis:** Processing observational data
- **Machine learning:** Training models on large datasets
- **Visualization:** Rendering scientific visualizations
- Predictable, long-running, computationally intensive

**Security Translation:**
- **Cloud:** Monitor for anomalous API patterns, unusual traffic spikes
- **HPC:** Monitor for anomalous job patterns, unusual resource requests
- **Risk:** "Normal" in HPC = jobs that run for days and consume thousands of cores; distinguishing malicious from legitimate is hard

**This is like:** Batch processing on steroids

---

### **Performance Requirements**

#### Cloud Computing:
- **Latency:** Milliseconds matter for web services
- **Throughput:** Requests per second, MB/s
- **Scalability:** Auto-scale to handle load
- **Availability:** 99.9% uptime goals
- Performance is important but not everything

#### HPC (Example System):
- **Raw compute power:** Petaflops (quadrillions of calculations per second)
- **Parallel efficiency:** How well does code scale to thousands of cores?
- **I/O performance:** Terabytes per second for data-intensive jobs
- **Time to science:** How fast can we get results?
- Performance is EVERYTHING - security is often seen as the enemy

**Security Translation:**
- **Cloud:** Security controls that add 10% overhead are usually acceptable
- **HPC:** Security controls that add 0.1% overhead get complaints
- **Risk:** Any security control that impacts performance will be fought tooth and nail

**This is like:** Optimizing for bare-metal performance rather than cloud flexibility

---

## Operational Model

### **Change Management**

#### Cloud Computing:
- **Rapid iteration:** Deploy multiple times per day
- **Blue-green deployments:** Easy to roll back
- **Infrastructure as code:** Terraform, CloudFormation
- **Automated testing:** CI/CD pipelines

#### HPC (Example System):
- **Slow, scheduled changes:** Maintenance windows planned months in advance
- **User notification:** Email all users about upcoming changes
- **Testing in dev clusters:** Test on smaller systems first
- **Minimal automation:** Often manual processes
- Users hate change (breaks their workflows)

**Security Translation:**
- **Cloud:** Patch frequently, automate security updates
- **HPC:** Patching is slow and painful; must balance security with stability
- **Risk:** Long patch cycles = more time vulnerable

**This is like:** Managing legacy enterprise infrastructure rather than modern cloud

---

### **Availability Requirements**

#### Cloud Computing:
- **High availability:** Multi-AZ deployments
- **Disaster recovery:** Backup regions
- **Auto-healing:** Instances replaced automatically
- Services should always be available

#### HPC (Example System):
- **Scheduled downtime:** Monthly or quarterly maintenance windows
- **No HA:** If login node goes down, users can't access cluster
- **Long recovery times:** Fixing a broken cluster might take days
- **Best effort:** Users expect some downtime
- Science can usually wait (but they prefer not to)

**Security Translation:**
- **Cloud:** Security updates can be rolled out with minimal downtime
- **HPC:** Security updates require scheduling downtime windows and angry users
- **Risk:** Pressure to defer security patches because "we can't take downtime right now"

**This is like:** On-prem data centers with scheduled maintenance windows

---

## Threat Model

### **Common Threats**

#### Cloud Computing:
- **External attacks:** Internet-facing services getting exploited
- **Credential compromise:** Stolen API keys, passwords
- **Misconfiguration:** Public S3 buckets, overly permissive security groups
- **Supply chain:** Compromised containers, malicious packages
- **Crypto mining:** Attackers using your compute for profit

#### HPC (Example System):
- **Insider threats:** Legitimate users misusing resources
- **Crypto mining:** Users running mining on "free" compute
- **Data theft:** Stealing scientific data or IP
- **Resource theft:** Using allocations for personal projects
- **Lateral movement:** Compromised account accessing other users' data
- **External attacks:** Less common (not internet-facing)

**Security Translation:**
- **Cloud:** Focus on perimeter security and access control
- **HPC:** Focus on insider threat detection and resource abuse monitoring
- **Risk:** Traditional security tools expect external threats; HPC threats are often internal and subtle

**This is like:** Enterprise insider threat but with users who have legitimate reasons to use massive resources

---

### **Detection Challenges**

#### Cloud Computing:
- **Normal vs. anomalous:** Well-understood baselines
- **Security tools:** Mature ecosystem (GuardDuty, SecurityHub, etc.)
- **Threat intelligence:** Lots of known attack patterns
- Automated detection is mature

#### HPC (Example System):
- **Normal is weird:** Legitimate jobs look bizarre (thousands of cores, days long, terabytes of data)
- **Limited tooling:** Few purpose-built security tools for HPC
- **Unknown baselines:** What's normal for scientific workflows?
- Manual detection required

**Security Translation:**
- **Cloud:** Use standard tools and threat intel
- **HPC:** Build custom detection, learn what "normal science" looks like
- **Risk:** It's REALLY hard to tell legitimate science from malicious activity

**This is like:** Building a SIEM from scratch without good threat intel

---

## Compliance & Governance

### **Regulatory Environment**

#### Cloud Computing:
- **FedRAMP:** Cloud services authorization
- **NIST 800-53:** Control framework
- **SOC 2:** Attestation reports
- **Shared responsibility:** Cloud provider handles some controls
- Well-defined compliance frameworks

#### HPC (Example System):
- **FISMA:** Federal information security (if applicable)
- **NIST 800-53:** Same control framework
- **Export controls:** Scientific data might have export restrictions
- **Data sharing agreements:** With other research institutions
- **Full responsibility:** No cloud provider to handle infrastructure controls
- Compliance is entirely on you

**Security Translation:**
- **Cloud:** Many controls inherited from cloud provider
- **HPC:** You implement ALL controls yourself
- **Risk:** More work, but also more control

**This is like:** On-prem data center compliance, not cloud

---

### **Audit & Assessment**

#### Cloud Computing:
- **Automated compliance:** AWS Config, Security Hub scanning
- **Continuous monitoring:** Real-time compliance checks
- **Audit reports:** Download from provider
- **Evidence collection:** API logs, screenshots, automated reports

#### HPC (Discover):
- **Manual assessment:** Walk through controls with auditors
- **Evidence collection:** Screenshots, config files, documentation
- **Continuous monitoring:** You build this (hello RADAR!)
- **Auditor education:** Have to teach auditors how HPC works

**Security Translation:**
- **Cloud:** Automated compliance checking is standard
- **HPC:** You're building automated compliance (this is where your RADAR methodology shines)
- **Risk:** Auditors often don't understand HPC; expect to do a lot of explaining

**This is like:** Being an early cloud adopter in 2010 when auditors didn't understand cloud yet

---

## Key Differences Summary

### What's the Same:
- **Core security principles:** Confidentiality, integrity, availability
- **Defense in depth:** Multiple layers of security
- **Least privilege:** Give minimum necessary access
- **Monitoring & detection:** Watch for anomalies
- **Compliance frameworks:** NIST 800-53 applies to both
- **Incident response:** Detect, contain, eradicate, recover

### What's Different:
- **Scale:** HPC is MUCH bigger (hundreds of thousands of cores)
- **Performance sensitivity:** Security can't slow things down
- **User expectations:** Scientists expect support, not self-service
- **Access model:** Batch jobs vs. persistent services
- **Tooling maturity:** Cloud security tools are more advanced
- **Threat focus:** Insider threats vs. external attacks
- **Change velocity:** HPC moves slower
- **Network architecture:** Physical separation vs. software-defined

### What's Harder in HPC:
- **Encryption:** Performance penalties
- **Patching:** Requires downtime
- **Detection:** Normal looks suspicious
- **Tooling:** Have to build your own
- **Compliance:** No inherited controls

### What's Easier in HPC:
- **Perimeter security:** Not internet-facing
- **User population:** Known, vetted scientists
- **Workload predictability:** Batch jobs are more predictable
- **Blast radius:** Physical segmentation limits lateral movement

---

## Mental Models for Common Scenarios

### Scenario: User wants to run a job

**Cloud thinking:**
"User deploys a container to ECS, it runs until stopped"

**HPC thinking:**
"User submits a job script to job scheduler, waits in queue, runs on assigned nodes for specified time, then terminates"

**Security thinking:**
Monitor job submission (authentication), job script contents (what's it doing?), resource request (reasonable?), job behavior (matches request?), job completion (clean up data?)

---

### Scenario: Data breach suspected

**Cloud thinking:**
"Check CloudTrail for unusual API calls, GuardDuty for findings, VPC flow logs for unusual network traffic"

**HPC thinking:**
"Check job scheduler accounting for unusual jobs, auth logs for unusual logins, filesystem logs for unusual file access, network traffic on data transfer nodes"

**Security thinking:**
Incident response process is the same, but data sources and investigation tools are different

---

### Scenario: Applying a security patch

**Cloud thinking:**
"Update AMI, blue-green deploy new instances, terminate old ones. Downtime: 0 minutes"

**HPC thinking:**
"Email users 4 weeks in advance, schedule maintenance window, take system offline, patch all nodes, test, bring back online. Downtime: 4-8 hours"

**Security thinking:**
Risk vs. operations trade-off is MUCH harder in HPC

---

### Scenario: Monitoring for crypto mining

**Cloud thinking:**
"Watch for unusual CPU usage, unexpected network traffic to mining pools, new processes on instances"

**HPC thinking:**
"Watch for jobs with suspicious names, unusual core counts, jobs that request GPUs but do no scientific computation, users burning through allocations quickly"

**Security thinking:**
Same goal, different indicators

---

## Quick Reference Card

When you encounter HPC jargon, translate it like this:

| HPC Term | Cloud Equivalent | Key Difference |
|----------|------------------|----------------|
| Compute node | EC2 instance | Shared sequentially, not dedicated |
| Login node | Bastion host | Everyone uses it, critical security boundary |
| Job | Batch job | Queued, scheduled, temporary |
| Job scheduler | Kubernetes | But for batch jobs, not services |
| Partition | Instance type category | But managed by scheduler |
| Allocation | Reserved capacity | But measured in core-hours |
| GPFS/Lustre | EFS on steroids | Parallel I/O, much faster |
| InfiniBand | AWS Placement Group | But actual physical high-speed network |
| Module | Python virtualenv | But for all software, not just Python |
| Queue | SQS + ASG | Jobs wait for resources, not messages |
| Scratch space | Ephemeral storage | But huge and shared |
| Data transfer node | NAT gateway | For data, not just network traffic |

---

## Your Superpower

**The fact that you DON'T know HPC intimately is actually an advantage.**

You won't accept:
- "We can't encrypt because performance"
- "We can't patch because science"
- "We can't monitor because overhead"
- "We've always done it this way"

You'll ask uncomfortable questions:
- "Why is this different from cloud?"
- "How do we know this is secure?"
- "What's the actual performance impact?" (measure it)
- "Can we do this better?"

Your cloud security experience gives you:
- Modern security practices
- Automation mindset
- Compliance frameworks
- Defense-in-depth thinking
- Risk-based decision making

Apply all of that to HPC. Don't let them tell you "HPC is special and different." It IS different, but it ALSO needs security.

---

## Final Translation

**HPC is just:**
- Cloud computing with fixed capacity
- Batch jobs instead of services
- Physical networks instead of SDN
- Unix permissions instead of IAM
- Scientists instead of developers
- Really, really fast storage and networking

**Security is still:**
- Defense in depth
- Least privilege
- Monitoring and detection
- Incident response
- Compliance frameworks
- Risk management

**You've got this.** You know security. You know cloud. HPC is just another computing model that needs securing. Now go do it.
