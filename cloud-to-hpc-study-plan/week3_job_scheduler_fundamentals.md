# Week 3: Job Scheduler Fundamentals
## Understanding How Users Submit and Run Jobs

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how job schedulers work and their security implications
- Submit and monitor test jobs on your HPC system
- Identify security controls needed for job scheduling
- Recognize normal vs. suspicious job submission patterns
- Design monitoring strategies for job scheduler security

---

## Day 1: Job Scheduler Architecture and Concepts

### Morning Session (3 hours)

#### Core Job Scheduler Concepts

**What is a Job Scheduler?**
- Central resource manager for the entire HPC cluster
- Decides which jobs run where and when
- Enforces resource limits and fair-share policies
- Tracks resource usage for accounting and billing

**Key Components:**
- **Job queue**: Where submitted jobs wait for resources
- **Scheduler daemon**: Makes scheduling decisions
- **Resource manager**: Tracks available nodes and resources
- **Accounting system**: Records job usage for billing/reporting

#### Common Job Schedulers

**Slurm (Simple Linux Utility for Resource Management)**
- Most common in modern HPC
- Open source, highly scalable
- Good security features when configured properly

**PBS/Torque (Portable Batch System)**
- Older but still widely used
- Commercial (PBS Pro) and open source (Torque) versions
- More basic security model

**LSF (Load Sharing Facility)**
- Commercial scheduler from IBM
- Enterprise features and support
- Complex but powerful

#### Cloud Translation

| Job Scheduler Concept | Cloud Equivalent | Key Difference |
|----------------------|------------------|----------------|
| Job queue | SQS + Auto Scaling Group | Jobs wait for compute resources, not just processing |
| Job script | Kubernetes Job YAML | Specifies resources needed, not just what to run |
| Partition/Queue | Instance type category | But managed centrally, not self-service |
| Fair-share | Resource quotas | But enforced by scheduler, not per-tenant |
| Job accounting | CloudWatch billing | But tracks compute time, not just cost |

### Afternoon Session (2 hours)

#### Security Implications of Job Scheduling

**Authentication and Authorization:**
- How does scheduler verify user identity?
- What prevents users from submitting jobs as other users?
- How are resource limits enforced?
- What audit trail exists for job submissions?

**Resource Isolation:**
- How does scheduler ensure job isolation on shared nodes?
- What prevents jobs from accessing each other's data?
- How are resources cleaned up between jobs?
- What happens if a job tries to use more resources than allocated?

**Privilege Management:**
- What privileges does the scheduler run with?
- How are user jobs executed (setuid, containers, etc.)?
- What prevents privilege escalation through job submission?
- How are administrative functions protected?

#### Initial Security Questions for Your Environment

Document answers to these questions:
- [ ] What job scheduler does your HPC center use?
- [ ] How are users authenticated to submit jobs?
- [ ] What resource limits and quotas exist?
- [ ] What job accounting and logging is performed?
- [ ] What security controls exist around job submission?

---

## Day 2: Hands-On Job Submission and Monitoring

### Morning Session (3 hours)

#### Get Hands-On with Job Submission

**Basic Job Submission (Slurm example):**
```bash
# Interactive job for testing
srun --pty bash

# Batch job submission
sbatch myjob.sh

# Check job status
squeue -u $USER

# Cancel a job
scancel <jobid>

# Job history
sacct -u $USER
```

**Basic Job Submission (PBS example):**
```bash
# Interactive job
qsub -I

# Batch job submission
qsub myjob.pbs

# Check job status
qstat -u $USER

# Cancel a job
qdel <jobid>

# Job history
qstat -x
```

#### Create Test Job Scripts

**Simple Test Job (Slurm):**
```bash
#!/bin/bash
#SBATCH --job-name=security-test
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --output=test-%j.out

echo "Job started at: $(date)"
echo "Running on node: $(hostname)"
echo "Job ID: $SLURM_JOB_ID"
echo "User: $(whoami)"
echo "Working directory: $(pwd)"
sleep 60
echo "Job completed at: $(date)"
```

**Simple Test Job (PBS):**
```bash
#!/bin/bash
#PBS -N security-test
#PBS -l walltime=00:05:00
#PBS -l nodes=1:ppn=1
#PBS -o test.out
#PBS -e test.err

echo "Job started at: $(date)"
echo "Running on node: $(hostname)"
echo "Job ID: $PBS_JOBID"
echo "User: $(whoami)"
echo "Working directory: $(pwd)"
sleep 60
echo "Job completed at: $(date)"
```

#### Submit and Monitor Your Test Jobs

1. Submit the test job
2. Monitor its progress through the queue
3. Examine the output files
4. Check job accounting information
5. Look at system logs (if you have access)

### Afternoon Session (2 hours)

#### Job Monitoring and Analysis

**What to Look For:**
- Job submission patterns and timing
- Resource requests vs. actual usage
- Job success/failure rates
- User behavior patterns
- Unusual or suspicious job characteristics

**Security-Relevant Job Information:**
- Who submitted the job and when?
- What resources were requested vs. used?
- What commands were executed?
- What files were accessed?
- What network connections were made?
- How long did the job run?

#### Create Your Job Monitoring Checklist

**Normal Job Characteristics:**
- [ ] Reasonable resource requests for user's typical work
- [ ] Job names and descriptions make sense
- [ ] Execution time matches resource request
- [ ] File access patterns are appropriate
- [ ] Network activity is expected for the application

**Suspicious Job Indicators:**
- [ ] Unusual resource requests (too high or too low)
- [ ] Generic or misleading job names
- [ ] Jobs that finish immediately or run much longer than requested
- [ ] Unexpected file access or network activity
- [ ] Jobs submitted at unusual times
- [ ] Repeated failed job submissions

---

## Day 3: Job Scheduler Security Configuration

### Morning Session (3 hours)

#### Authentication and Access Control

**User Authentication to Scheduler:**
- How are users authenticated when submitting jobs?
- Are there any additional authentication requirements?
- How are user credentials validated?
- What happens if authentication fails?

**Authorization and Permissions:**
- What determines which queues/partitions users can access?
- How are resource limits enforced?
- What prevents users from submitting jobs as other users?
- How are administrative functions protected?

**Account Management:**
- How are user accounts created and managed?
- What information is required for account creation?
- How are accounts disabled or removed?
- What audit trail exists for account changes?

#### Resource Management and Limits

**Resource Quotas:**
- What resource limits exist (CPU time, memory, storage)?
- How are limits enforced?
- What happens when limits are exceeded?
- How are limits monitored and reported?

**Fair-Share Policies:**
- How is resource sharing managed between users/groups?
- What priority systems exist?
- How are resource allocations tracked?
- What prevents resource monopolization?

### Afternoon Session (2 hours)

#### Job Execution Security

**Process Isolation:**
- How are jobs isolated from each other?
- What prevents jobs from interfering with system processes?
- How are temporary files and directories managed?
- What cleanup occurs when jobs complete?

**Privilege Management:**
- What user context do jobs run under?
- How are elevated privileges handled (if needed)?
- What prevents privilege escalation?
- How are system resources protected?

**Environment Security:**
- What environment variables are available to jobs?
- How is the execution environment sanitized?
- What prevents information leakage between jobs?
- How are shared libraries and modules secured?

#### Document Current Security Configuration

Create an inventory of:
- [ ] Authentication mechanisms
- [ ] Authorization policies
- [ ] Resource limits and quotas
- [ ] Job isolation mechanisms
- [ ] Privilege management approaches
- [ ] Environment security controls

---

## Day 4: Job Accounting and Logging

### Morning Session (3 hours)

#### Understanding Job Accounting

**What Information is Collected:**
- Job submission details (user, time, resources requested)
- Job execution details (start time, end time, exit status)
- Resource usage (CPU time, memory, I/O, network)
- Accounting information (project codes, billing)

**Where Information is Stored:**
- Job scheduler databases
- Accounting log files
- System log files
- Custom reporting systems

**How Long Information is Retained:**
- Scheduler database retention policies
- Log file rotation and archival
- Backup and long-term storage
- Compliance requirements for data retention

#### Hands-On Accounting Analysis

**Slurm Accounting Commands:**
```bash
# Job history for user
sacct -u username --format=JobID,JobName,Start,End,State,ExitCode

# Detailed job information
scontrol show job <jobid>

# User resource usage summary
sreport user TopUsage start=2024-01-01 end=2024-01-31

# Cluster utilization
sinfo -o "%P %C %S"
```

**PBS Accounting Commands:**
```bash
# Job history
qstat -x -u username

# Detailed job information
qstat -f <jobid>

# Accounting logs
# (usually in /var/spool/pbs/server_logs/)
```

### Afternoon Session (2 hours)

#### Security Monitoring with Job Data

**Security Events to Monitor:**
- Unusual job submission patterns
- Resource abuse or waste
- Failed authentication attempts
- Jobs with suspicious characteristics
- Privilege escalation attempts
- Data exfiltration indicators

**Creating Security Dashboards:**
- Job submission rates and patterns
- Resource utilization trends
- User behavior analysis
- Failed job analysis
- Security alert correlation

#### Log Analysis Exercise

1. Extract job accounting data for the past week
2. Analyze patterns in job submissions
3. Identify any unusual or suspicious activity
4. Create simple visualizations of the data
5. Document what "normal" looks like for your environment

---

## Day 5: Security Controls Design

### Morning Session (3 hours)

#### Designing Job Scheduler Security Controls

**Preventive Controls:**
- Strong authentication requirements
- Proper authorization and access controls
- Resource limits and quotas
- Job submission validation
- Environment sanitization

**Detective Controls:**
- Comprehensive job accounting and logging
- Real-time monitoring and alerting
- User behavior analysis
- Resource usage monitoring
- Security event correlation

**Responsive Controls:**
- Incident response procedures
- Job termination capabilities
- Account suspension mechanisms
- Forensic data collection
- Recovery procedures

#### Security Control Implementation Plan

For each control category, document:
- Current state (what exists now)
- Desired state (what should exist)
- Gap analysis (what's missing)
- Implementation approach
- Resource requirements
- Timeline and priorities

### Afternoon Session (2 hours)

#### Integration with Organizational Security

**SIEM Integration:**
- What job scheduler logs should be sent to SIEM?
- How should events be normalized and parsed?
- What correlation rules are needed?
- How should alerts be prioritized and routed?

**Identity Management Integration:**
- How should job scheduler accounts integrate with organizational identity systems?
- What authentication methods should be supported?
- How should account lifecycle be managed?
- What audit requirements exist?

**Compliance Mapping:**
- Which compliance controls relate to job scheduling?
- What evidence needs to be collected?
- How should controls be tested and validated?
- What reporting is required?

#### Create Security Requirements Document

Document specific security requirements for:
- [ ] User authentication and authorization
- [ ] Job submission validation and controls
- [ ] Resource management and limits
- [ ] Job execution isolation and security
- [ ] Logging, monitoring, and alerting
- [ ] Incident response and forensics
- [ ] Compliance and audit support

---

## Week 3 Deliverable: Job Scheduler Security Assessment

Create a comprehensive document (5-8 pages) covering:

### Executive Summary (1 page)
- Current job scheduler security posture
- Key security risks and gaps
- Recommended security improvements
- Resource requirements and timeline

### Job Scheduler Analysis (2-3 pages)

**Architecture and Configuration:**
- Job scheduler type and version
- Current security configuration
- Authentication and authorization mechanisms
- Resource management and limits

**Security Controls Assessment:**
- Preventive controls (what prevents bad things)
- Detective controls (what detects bad things)
- Responsive controls (what responds to bad things)
- Control effectiveness and gaps

**User and Usage Analysis:**
- User community and behavior patterns
- Typical job characteristics and resource usage
- Historical security incidents or issues
- Risk factors and threat scenarios

### Security Recommendations (2-3 pages)

**Immediate Improvements (0-30 days):**
- Critical security gaps to address
- Quick configuration changes
- Enhanced monitoring and alerting

**Short-term Enhancements (1-6 months):**
- Authentication and authorization improvements
- Enhanced logging and monitoring
- Integration with organizational security tools

**Long-term Strategic Items (6+ months):**
- Advanced security features
- Automation and orchestration
- Comprehensive security monitoring

### Implementation Plan (1 page)
- Prioritized list of recommendations
- Resource requirements and dependencies
- Timeline and milestones
- Success metrics and validation

---

## Key Security Questions Answered This Week

By the end of week 3, you should be able to answer:

**About Job Scheduling:**
- [ ] How does your job scheduler authenticate and authorize users?
- [ ] What resource limits and controls exist?
- [ ] How are jobs isolated and secured during execution?
- [ ] What accounting and logging is performed?

**About Security Monitoring:**
- [ ] What job scheduler events should be monitored for security?
- [ ] How can you detect suspicious job submission patterns?
- [ ] What baselines exist for normal job behavior?
- [ ] How should job scheduler logs integrate with your SIEM?

**About Risk Management:**
- [ ] What are the highest security risks in job scheduling?
- [ ] What controls are missing or inadequate?
- [ ] How should security improvements be prioritized?
- [ ] What compliance requirements apply to job scheduling?

---

## Resources for Week 3

### Documentation and Guides
- Slurm Security Guide: https://slurm.schedmd.com/security.html
- PBS Security Documentation
- Your HPC center's job scheduler documentation
- NIST guidelines for batch processing security

### Tools and Commands
- Job scheduler command references (squeue, sacct, qstat, etc.)
- Log analysis tools (grep, awk, Python scripts)
- Monitoring tools (Grafana, Nagios, etc.)
- Security scanning tools (if applicable)

### People to Consult
- HPC system administrators
- Job scheduler experts
- Security team members
- Heavy HPC users (for usage patterns)

---

## Success Metrics for Week 3

You should be able to:
- [ ] Submit and monitor jobs on your HPC system
- [ ] Explain job scheduler security controls to management
- [ ] Identify suspicious job submission patterns
- [ ] Design security monitoring for job scheduling
- [ ] Integrate job scheduler security with organizational security programs

---

## Common Week 3 Challenges and Solutions

**"The job scheduler is too complex to understand quickly"**
- Focus on security-relevant aspects, not all features
- Start with basic job submission and build from there
- Ask HPC staff to explain the most important security features
- Use hands-on practice to reinforce learning

**"I can't get access to submit test jobs"**
- Work with HPC staff to get a test account
- Use job scheduler simulators or documentation examples
- Shadow other users submitting jobs
- Focus on monitoring and analysis of existing jobs

**"There's too much job accounting data to analyze"**
- Start with small time periods (last week or month)
- Focus on security-relevant fields and patterns
- Use simple tools (spreadsheets, basic scripts) initially
- Ask for help identifying what's normal vs. unusual

**"The security implications aren't clear"**
- Think about job scheduling like any other privileged system
- Consider what could go wrong (unauthorized access, resource abuse, etc.)
- Map job scheduler functions to security controls you know
- Ask "what would an attacker do?" for each feature

---

## Week 3 Wrap-Up

By the end of week 3, you should understand how job schedulers work and their critical role in HPC security. Job schedulers are the central control point for resource access - securing them properly is essential for overall HPC security.

The key insight: Job schedulers are like a combination of identity management, resource management, and audit logging systems. They need to be secured with the same rigor as any other critical infrastructure component.

**Next week**: We'll explore resource management security in more depth, including multi-tenancy, compute node isolation, and container security in HPC environments.