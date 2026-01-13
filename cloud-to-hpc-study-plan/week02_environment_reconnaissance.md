# Week 2: HPC Center-Specific Deep Dive
## Understanding YOUR Specific Environment

---

## Learning Objectives

By the end of this week, you should be able to:
- Describe your HPC center's specific architecture and security posture
- Identify current security controls and gaps
- Understand the user community and their workflows
- Document the current state for future security improvements
- Ask informed questions about security to HPC operations staff

---

## Day 1: Documentation Review and Information Gathering

### Morning Session (3 hours)

#### Request Essential Documentation

**From HPC Operations Team:**
- [ ] System architecture diagrams
- [ ] Network topology diagrams
- [ ] User account management procedures
- [ ] Current security policies and procedures
- [ ] Incident response procedures (if they exist)
- [ ] Compliance documentation (FISMA, etc.)
- [ ] System monitoring and logging configuration
- [ ] Backup and disaster recovery procedures

**From Organizational Security Team:**
- [ ] Current security assessments of HPC environment
- [ ] Applicable compliance frameworks
- [ ] Security incident history
- [ ] Organizational security policies that apply to HPC
- [ ] Risk assessment documentation

#### Documentation Review Checklist

For each document, note:
- **Date**: How current is this information?
- **Completeness**: What's missing or unclear?
- **Security focus**: How much security detail is included?
- **Gaps**: What questions does this raise?

### Afternoon Session (2 hours)

#### Create Your Information Inventory

**System Information:**
- HPC system name and model
- Number of compute nodes, cores, total capacity
- Job scheduler type and version
- Operating system and versions
- Storage systems and capacity
- Network infrastructure

**Security Information:**
- Authentication methods
- Authorization mechanisms
- Monitoring and logging systems
- Security tools deployed
- Compliance requirements
- Known security issues

#### Initial Gap Analysis

Create three lists:
1. **What I Know**: Clear, documented information
2. **What I Think I Know**: Assumptions that need verification
3. **What I Don't Know**: Critical gaps requiring investigation

---

## Day 2: Shadow Operations Team

### Preparation (30 minutes)

#### Questions to Prepare
- How do users get accounts and access?
- What does a typical day look like for operations?
- What are the most common user issues?
- What keeps you up at night (security-wise)?
- What would you change about current security if you could?

### Shadowing Session (4-6 hours)

#### Observe and Document

**User Support Activities:**
- How are user requests handled?
- What access do support staff have?
- How are user issues escalated?
- What security checks are performed?

**System Administration:**
- How are systems monitored?
- What alerts exist and how are they handled?
- How are patches and updates applied?
- What backup and recovery procedures are used?

**Security-Related Activities:**
- How are security incidents detected and handled?
- What security monitoring is automated vs. manual?
- How are user accounts managed and audited?
- What compliance activities are performed?

#### Take Notes On
- Processes that seem ad-hoc or manual
- Security controls that are missing or weak
- Areas where staff seem frustrated or overworked
- Opportunities for automation or improvement

### Debrief Session (1 hour)

#### Key Questions to Ask
- What are your biggest security concerns?
- What security improvements would help you most?
- What compliance requirements are hardest to meet?
- What would break if key staff left?
- What security tools would you like to have?

---

## Day 3: User Community Analysis

### Morning Session (2 hours)

#### Understand Your User Base

**User Demographics:**
- How many active users?
- What scientific domains are represented?
- Internal vs. external users
- Student vs. faculty vs. staff vs. industry
- Geographic distribution

**User Behavior Patterns:**
- Peak usage times
- Typical job sizes and durations
- Data storage patterns
- Collaboration patterns
- Support request patterns

#### Security-Relevant User Characteristics

**Access Patterns:**
- How do users typically access the system?
- What locations do they access from?
- What devices do they use?
- How often do they change passwords/SSH keys?

**Data Handling:**
- What types of data do users work with?
- How sensitive is the data?
- Do users share data with external collaborators?
- What export control restrictions apply?

**Technical Sophistication:**
- How security-aware are users?
- Do users follow security best practices?
- How do users react to security controls?
- What security training do users receive?

### Afternoon Session (2 hours)

#### Interview Key Stakeholders

**HPC Center Director/Manager:**
- Strategic priorities and challenges
- Budget and resource constraints
- Compliance and regulatory requirements
- Relationship with organizational security

**Lead Scientists/Principal Investigators:**
- Research requirements and constraints
- Data sharing and collaboration needs
- Tolerance for security controls
- Past security incidents or concerns

**User Support Staff:**
- Common user issues and complaints
- Security-related support requests
- User training and education needs
- Suggestions for improvement

#### Document User Requirements

Create user personas for security planning:
- **The Careful Researcher**: Follows rules, wants guidance
- **The Impatient Expert**: Knows what they're doing, routes around obstacles
- **The Collaborator**: Needs to share data with external partners
- **The Newcomer**: Needs training and support
- **The Power User**: Pushes system limits, finds edge cases

---

## Day 4: Current Security Controls Assessment

### Morning Session (3 hours)

#### Authentication and Access Control Audit

**Current State Documentation:**
- How do users get accounts?
- What authentication methods are used?
- How are passwords/SSH keys managed?
- What access controls exist on filesystems?
- How are privileged accounts managed?
- What role-based access exists?

**Security Control Evaluation:**
- Are controls documented?
- Are they consistently applied?
- Are they regularly audited?
- Do they meet compliance requirements?
- What are the gaps or weaknesses?

#### Network Security Assessment

**Network Architecture Review:**
- What network segmentation exists?
- How is external access controlled?
- What monitoring exists on network traffic?
- Are there any network-based security controls?
- How is the high-speed interconnect secured?

**External Connectivity:**
- How do users transfer data in/out?
- What external connections exist?
- How are these connections secured?
- What monitoring exists for data transfers?

### Afternoon Session (2 hours)

#### Monitoring and Logging Assessment

**Current Logging:**
- What systems generate logs?
- Where are logs stored and for how long?
- What log analysis is performed?
- Are logs centralized?
- What alerting exists?

**Security Monitoring:**
- What security events are monitored?
- How are security incidents detected?
- What automated monitoring exists?
- What manual monitoring is performed?
- How effective is current monitoring?

#### Compliance and Audit Readiness

**Current Compliance Posture:**
- What frameworks apply (FISMA, NIST 800-53, etc.)?
- What controls are implemented?
- What evidence is collected?
- How are audits conducted?
- What are the compliance gaps?

---

## Day 5: Security Gap Analysis and Prioritization

### Morning Session (3 hours)

#### Comprehensive Gap Analysis

**Critical Security Gaps:**
- Missing security controls
- Weak or ineffective controls
- Compliance violations
- High-risk vulnerabilities
- Single points of failure

**Operational Gaps:**
- Lack of documentation
- Manual processes that should be automated
- Insufficient monitoring or alerting
- Inadequate incident response capabilities
- Staff knowledge or training gaps

**Technical Gaps:**
- Outdated or unpatched systems
- Missing security tools
- Inadequate backup or recovery
- Poor access controls
- Insufficient encryption

#### Risk Assessment

For each gap, evaluate:
- **Likelihood**: How likely is this to be exploited?
- **Impact**: What would happen if this were exploited?
- **Detectability**: Would we know if this were exploited?
- **Effort to Fix**: How hard would this be to address?

### Afternoon Session (2 hours)

#### Prioritization Framework

**Immediate (Fix Now):**
- Critical vulnerabilities
- Compliance violations
- High-risk, easy-to-fix issues

**Short-term (Next 3 months):**
- Important security improvements
- Process improvements
- Tool implementations

**Medium-term (3-12 months):**
- Major architectural changes
- Comprehensive monitoring implementation
- Advanced security capabilities

**Long-term (1+ years):**
- Strategic security initiatives
- Major system upgrades
- Cultural/organizational changes

#### Stakeholder Communication Plan

**For HPC Leadership:**
- Executive summary of findings
- Risk-based prioritization
- Resource requirements
- Timeline for improvements

**For Operations Team:**
- Technical details of gaps
- Specific recommendations
- Implementation guidance
- Support requirements

**For Organizational Security:**
- Compliance status
- Risk assessment
- Integration opportunities
- Shared responsibilities

---

## Week 2 Deliverable: Current State Security Assessment

Create a comprehensive document (5-10 pages) with:

### Executive Summary (1 page)
- Overall security posture assessment
- Top 5 security risks
- Recommended immediate actions
- Resource requirements for improvements

### Current State Analysis (2-3 pages)

**Architecture Overview:**
- System components and their security roles
- Network topology and security boundaries
- User access patterns and controls

**Security Controls Inventory:**
- Authentication and access control
- Network security
- Monitoring and logging
- Compliance controls
- Incident response capabilities

**User Community Profile:**
- User demographics and behavior patterns
- Security awareness and training needs
- Data handling and sharing requirements

### Gap Analysis (2-3 pages)

**Critical Gaps:**
- Missing or inadequate security controls
- Compliance violations or risks
- High-impact vulnerabilities

**Operational Improvements:**
- Process improvements needed
- Automation opportunities
- Documentation gaps

**Technical Enhancements:**
- Security tools needed
- Infrastructure improvements
- Monitoring enhancements

### Recommendations (1-2 pages)

**Immediate Actions (0-30 days):**
- Critical fixes
- Quick wins
- Compliance items

**Short-term Improvements (1-6 months):**
- Important security enhancements
- Process improvements
- Tool implementations

**Long-term Strategic Items (6+ months):**
- Major architectural changes
- Advanced security capabilities
- Cultural improvements

### Implementation Roadmap (1 page)
- Timeline for recommendations
- Resource requirements
- Dependencies and prerequisites
- Success metrics

---

## Key Questions to Answer This Week

By the end of week 2, you should be able to answer:

**About the System:**
- [ ] What is the overall architecture of your HPC environment?
- [ ] What are the key security boundaries and trust relationships?
- [ ] What compliance requirements apply?

**About Current Security:**
- [ ] What security controls are currently in place?
- [ ] What are the most significant security gaps?
- [ ] What security incidents have occurred historically?

**About Users:**
- [ ] Who uses the system and how?
- [ ] What are their security-relevant behaviors?
- [ ] What are their tolerance levels for security controls?

**About Operations:**
- [ ] How are security tasks currently performed?
- [ ] What are the operational pain points?
- [ ] What improvements would have the biggest impact?

**About Priorities:**
- [ ] What are the highest-risk security issues?
- [ ] What are the easiest wins?
- [ ] What requires long-term strategic investment?

---

## Resources for Week 2

### Templates and Checklists
- Security control assessment checklist
- Gap analysis template
- Risk assessment matrix
- Stakeholder interview guide

### Reference Materials
- NIST 800-53 control catalog
- Your organization's security policies
- HPC security best practices guides
- Compliance framework requirements

### Tools You Might Use
- Network scanning tools (if approved)
- Log analysis tools
- Documentation tools (diagrams, wikis)
- Risk assessment tools

---

## Success Metrics for Week 2

You should be able to:
- [ ] Describe your HPC environment's security posture to leadership
- [ ] Identify the top 5 security risks and their business impact
- [ ] Explain current security controls and their effectiveness
- [ ] Recommend specific, prioritized security improvements
- [ ] Communicate effectively with both HPC operations and organizational security teams

---

## Common Week 2 Challenges and Solutions

**"I can't get access to all the documentation I need"**
- Start with what you can get
- Document what's missing as a gap
- Use interviews to fill information gaps
- Work with management to get access

**"The operations team is too busy to spend time with me"**
- Schedule shorter, focused sessions
- Offer to help with tasks while learning
- Come prepared with specific questions
- Show how security improvements will help them

**"There's too much information and it's overwhelming"**
- Focus on security-relevant information
- Use templates and checklists to stay organized
- Don't try to understand everything at once
- Ask for help prioritizing what's most important

**"I'm finding serious security issues and don't know what to do"**
- Document everything carefully
- Assess immediate risk and escalate if necessary
- Don't try to fix everything at once
- Focus on building a plan for systematic improvement

**"The HPC staff don't seem to care about security"**
- Understand their priorities and constraints
- Frame security in terms of protecting their work
- Look for security improvements that also improve operations
- Build relationships before pushing for changes

---

## Week 2 Wrap-Up

By the end of week 2, you should have a clear picture of your HPC environment's current security posture. You're not trying to fix everything yet - you're building the foundation of knowledge needed to make informed security decisions.

The key insight: Every HPC environment is different, but the security principles are the same. Your job is to understand the specific implementation details so you can apply security effectively.

**Next week**: We'll dive into job schedulers and resource management - understanding how users actually interact with the HPC system and where the security controls need to be.