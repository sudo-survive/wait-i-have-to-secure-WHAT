# Week 6: Data Management & Archives
## Understanding the Full Data Lifecycle at Your HPC Center

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand the complete scientific data lifecycle and security implications
- Assess tape storage and archival system security
- Evaluate data movement and transfer security mechanisms
- Analyze data sharing and collaboration security requirements
- Design data classification and handling frameworks
- Create comprehensive data security and lifecycle management policies

---

## Day 1: Scientific Data Lifecycle

### Morning Session (3 hours)

#### Understanding Scientific Data Workflows

**Typical Scientific Data Lifecycle:**

1. **Data Acquisition/Generation:**
   - Experimental data collection
   - Simulation output generation
   - External data ingestion
   - Real-time data streams

2. **Active Processing:**
   - Data cleaning and preprocessing
   - Analysis and computation
   - Visualization and exploration
   - Iterative refinement

3. **Collaboration and Sharing:**
   - Internal team sharing
   - External collaborator access
   - Publication preparation
   - Peer review processes

4. **Long-term Storage:**
   - Archive for future reference
   - Compliance retention requirements
   - Backup and disaster recovery
   - Legacy data preservation

5. **Data Publication/Release:**
   - Public dataset publication
   - Repository submission
   - Open science requirements
   - Export control compliance

6. **End-of-Life:**
   - Secure data destruction
   - Compliance with retention policies
   - Legal hold considerations
   - Historical preservation decisions

#### Security Implications at Each Stage

**Data Acquisition Security:**
- Source validation and integrity
- Secure transfer mechanisms
- Access control for raw data
- Audit trails for data provenance

**Active Processing Security:**
- Access controls during analysis
- Data integrity during processing
- Backup of intermediate results
- Version control and change tracking

**Collaboration Security:**
- External access controls
- Data sharing agreements
- Export control compliance
- Intellectual property protection

**Archive Security:**
- Long-term data integrity
- Access controls for archived data
- Disaster recovery capabilities
- Compliance with retention requirements

### Afternoon Session (2 hours)

#### Data Classification Framework

**Scientific Data Sensitivity Levels:**

**Public Data:**
- Already published or intended for publication
- No access restrictions
- Standard backup and integrity protection
- Open science compliance

**Internal Data:**
- Unpublished research data
- Institutional access only
- Standard security controls
- Publication embargo protection

**Restricted Data:**
- Sensitive research data
- Limited access controls
- Enhanced security measures
- Export control considerations

**Controlled Data:**
- Highly sensitive or regulated data
- Strict access controls
- Encryption requirements
- Compliance monitoring

#### Hands-On Data Classification Exercise

**Inventory Current Data Types:**
```bash
# Explore different data storage areas
ls -la /scratch/
ls -la /projects/
ls -la /archive/

# Look for data classification indicators
find /projects -name "README*" -exec grep -l "sensitive\|restricted\|export" {} \;

# Check for data sharing agreements or policies
# (ask operations team for documentation)
```

**Create Data Inventory:**
- [ ] What types of scientific data are stored?
- [ ] What sensitivity levels exist?
- [ ] What compliance requirements apply?
- [ ] What data sharing occurs?
- [ ] What retention requirements exist?

---

## Day 2: Tape Storage and Archival Systems

### Morning Session (3 hours)

#### Understanding Tape Storage in HPC

**Why Tape Storage Still Matters:**
- **Cost-effective**: Cheapest storage per TB for long-term retention
- **Durability**: Tapes can last 30+ years with proper care
- **Offline security**: Air-gapped storage protects against cyber attacks
- **Capacity**: Modern tape cartridges hold 12-18 TB compressed
- **Energy efficient**: No power consumption when not in use

**Tape Storage Architecture:**
- **Tape libraries**: Robotic systems managing hundreds/thousands of tapes
- **Tape drives**: Read/write mechanisms for accessing tape data
- **Media management**: Software tracking tape locations and contents
- **HSM (Hierarchical Storage Management)**: Automated data migration

#### Common Tape Technologies

**LTO (Linear Tape-Open):**
- Industry standard, multiple vendors
- LTO-8: 12TB native, 30TB compressed
- LTO-9: 18TB native, 45TB compressed
- Hardware encryption available

**Enterprise Tape:**
- IBM TS1160: 20TB native, 60TB compressed
- Oracle T10000: 8.5TB native, 25TB compressed
- Proprietary but high performance

#### Security Features of Modern Tape Systems

**Hardware Encryption:**
- AES-256 encryption built into tape drives
- Key management through external systems
- Transparent to applications
- FIPS 140-2 compliance available

**Access Controls:**
- Physical access controls to tape libraries
- Logical access controls through HSM software
- Audit trails for tape access
- Write-once, read-many (WORM) capabilities

### Afternoon Session (2 hours)

#### Hands-On Tape System Assessment

**If Your HPC Center Has Tape Storage:**
```bash
# Check for tape storage systems
mount | grep -i tape
df -h | grep -i tape

# Look for HSM commands (varies by system)
which dmget dmput dmls  # Generic HSM commands
which hsi              # HPSS interface
which dsmc             # IBM Spectrum Protect

# Check tape storage policies
# (ask operations team for documentation)
```

**Tape Security Assessment:**
- [ ] What tape technologies are in use?
- [ ] What encryption capabilities exist?
- [ ] How are tapes physically secured?
- [ ] What access controls exist for archived data?
- [ ] How is tape integrity verified?
- [ ] What disaster recovery procedures exist?

**If No Tape Storage:**
- Document current archival approach
- Consider future tape storage security requirements
- Evaluate cloud archival alternatives (Glacier, etc.)
- Review long-term data retention strategies

---

## Day 3: Data Movement and Transfer Security

### Morning Session (3 hours)

#### Data Transfer Technologies in HPC

**High-Performance Data Transfer:**
- **Globus**: Grid-based data transfer service
- **GridFTP**: High-performance, secure file transfer protocol
- **Aspera**: Commercial high-speed transfer solution
- **bbcp**: Parallel data transfer utility
- **rsync**: Traditional but still widely used

**Data Transfer Nodes:**
- Dedicated nodes for external data transfers
- High-bandwidth network connections
- Specialized transfer software
- Security boundary between HPC and external networks

#### Globus Security Deep Dive

**Globus Architecture:**
- **Globus Connect**: Software running on endpoints
- **Globus Auth**: OAuth-based authentication
- **Globus Transfer**: Managed file transfer service
- **Globus Sharing**: Secure data sharing capabilities

**Security Features:**
- OAuth 2.0 authentication
- Encrypted data transfers
- Audit logging and monitoring
- Fine-grained access controls
- Integration with institutional identity systems

#### Hands-On Data Transfer Assessment

**Examine Current Transfer Capabilities:**
```bash
# Check for data transfer nodes
# (usually separate from login nodes)

# Look for transfer software
which globus-url-copy
which gridftp
which bbcp
which rsync

# Check network connectivity
# (ask about dedicated transfer networks)

# Examine transfer logs
# (locations vary by system)
```

### Afternoon Session (2 hours)

#### Secure Data Transfer Best Practices

**Transfer Security Controls:**
- **Authentication**: Strong user authentication
- **Authorization**: Proper access controls
- **Encryption**: Data encrypted in transit
- **Integrity**: Data integrity verification
- **Audit**: Complete transfer logging
- **Rate limiting**: Prevent bandwidth abuse

**Data Transfer Monitoring:**
- Transfer volume and patterns
- Failed transfer attempts
- Unusual transfer destinations
- Large or sensitive data transfers
- Automated vs. manual transfers

#### External Collaboration Security

**Data Sharing Agreements:**
- Legal framework for data sharing
- Technical security requirements
- Access control specifications
- Audit and monitoring requirements
- Data retention and destruction

**Export Control Compliance:**
- ITAR (International Traffic in Arms Regulations)
- EAR (Export Administration Regulations)
- Deemed export considerations
- Foreign national access restrictions
- Technology transfer controls

---

## Day 4: Data Sharing and Collaboration Requirements

### Morning Session (3 hours)

#### Scientific Collaboration Models

**Internal Collaboration:**
- Within research groups
- Across departments
- Multi-institutional projects
- Industry partnerships

**External Collaboration:**
- International research partnerships
- Open science initiatives
- Public-private partnerships
- Commercial collaborations

**Data Publication:**
- Journal data requirements
- Repository submissions
- Open access mandates
- Reproducibility requirements

#### Security Challenges in Scientific Collaboration

**Access Control Complexity:**
- Multiple institutions with different identity systems
- Temporary access for visiting researchers
- Project-based access that changes over time
- Balancing openness with security

**Data Sovereignty:**
- Data residency requirements
- Cross-border data transfer restrictions
- Jurisdictional compliance issues
- Export control implications

**Intellectual Property Protection:**
- Protecting unpublished research
- Patent considerations
- Commercial confidentiality
- Attribution and credit

### Afternoon Session (2 hours)

#### Collaboration Security Framework

**Identity and Access Management:**
- Federated identity systems
- Guest account management
- Multi-factor authentication
- Regular access reviews

**Data Sharing Controls:**
- Project-based access controls
- Time-limited access
- Data use agreements
- Audit and monitoring

**Technical Security Measures:**
- Secure collaboration platforms
- Encrypted communication channels
- Data loss prevention
- Endpoint security requirements

#### Hands-On Collaboration Assessment

**Current Collaboration Analysis:**
- [ ] What external collaborations exist?
- [ ] How is external access managed?
- [ ] What data sharing agreements are in place?
- [ ] What technical controls exist for collaboration?
- [ ] How is collaborative access monitored?

**Compliance Requirements:**
- [ ] What export control requirements apply?
- [ ] What data sovereignty issues exist?
- [ ] What institutional policies govern collaboration?
- [ ] What audit requirements exist?

---

## Day 5: Data Security Framework Development

### Morning Session (3 hours)

#### Comprehensive Data Security Assessment

**Data Inventory and Classification:**
- Complete inventory of data types and locations
- Sensitivity classification for all data
- Compliance requirements mapping
- Risk assessment for each data category

**Lifecycle Security Controls:**
- Security controls for each lifecycle stage
- Data handling procedures
- Access control mechanisms
- Monitoring and audit capabilities

**Gap Analysis:**
- Missing or inadequate security controls
- Compliance violations or risks
- Operational challenges and inefficiencies
- Technology limitations and constraints

### Afternoon Session (2 hours)

#### Data Security Policy Development

**Data Classification Policy:**
- Clear classification criteria
- Handling requirements for each classification
- Marking and labeling requirements
- Regular classification reviews

**Data Lifecycle Management Policy:**
- Procedures for each lifecycle stage
- Retention and disposal requirements
- Backup and recovery procedures
- Archive and long-term storage policies

**Data Sharing and Collaboration Policy:**
- Requirements for external data sharing
- Approval processes and authorities
- Technical security requirements
- Monitoring and audit procedures

#### Implementation Planning

**Short-term Actions (0-6 months):**
- Data classification implementation
- Policy development and approval
- Basic security controls enhancement
- Staff training and awareness

**Medium-term Improvements (6-18 months):**
- Advanced data protection technologies
- Automated policy enforcement
- Enhanced monitoring and alerting
- Compliance automation

**Long-term Strategic Items (18+ months):**
- Next-generation data management platforms
- Advanced analytics and AI for data security
- Comprehensive data governance program
- Cultural and organizational transformation

---

## Week 6 Deliverable: Data Security and Classification Framework

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Current data security posture
- Key risks and compliance issues
- Recommended data classification framework
- Implementation timeline and resources

### Data Lifecycle Analysis (3-4 pages)

**Scientific Data Workflows:**
- Complete data lifecycle mapping
- Security requirements for each stage
- Current controls and their effectiveness
- Gap analysis and improvement opportunities

**Data Classification Framework:**
- Proposed classification levels and criteria
- Handling requirements for each classification
- Implementation procedures and timelines
- Training and awareness requirements

### Storage and Archive Security (3-4 pages)

**Tape Storage Assessment (if applicable):**
- Current tape storage architecture
- Security controls and capabilities
- Physical and logical access controls
- Disaster recovery and business continuity

**Archive Security Requirements:**
- Long-term data integrity requirements
- Access controls for archived data
- Compliance and retention requirements
- Migration and technology refresh planning

### Data Transfer and Collaboration Security (3-4 pages)

**Transfer Security Analysis:**
- Current data transfer capabilities
- Security controls and monitoring
- External collaboration requirements
- Export control and compliance issues

**Collaboration Framework:**
- Identity and access management for collaborators
- Data sharing agreements and procedures
- Technical security requirements
- Monitoring and audit capabilities

### Implementation Roadmap (2-3 pages)

**Data Classification Implementation:**
- Classification criteria and procedures
- Data inventory and tagging approach
- Policy development and approval process
- Training and change management

**Security Controls Enhancement:**
- Prioritized list of security improvements
- Technology implementations required
- Process and procedure updates
- Compliance and audit preparations

**Timeline and Resources:**
- Detailed implementation timeline
- Resource requirements and dependencies
- Success metrics and validation approaches
- Risk mitigation strategies

---

## Key Security Questions Answered This Week

By the end of week 6, you should be able to answer:

**About Data Lifecycle:**
- [ ] What is the complete data lifecycle in your HPC environment?
- [ ] What security controls exist at each lifecycle stage?
- [ ] How should scientific data be classified and handled?
- [ ] What compliance requirements apply to different data types?

**About Archives and Long-term Storage:**
- [ ] How is long-term data storage secured?
- [ ] What disaster recovery capabilities exist for archived data?
- [ ] How is data integrity maintained over time?
- [ ] What are the retention and disposal requirements?

**About Data Sharing and Collaboration:**
- [ ] How is external data sharing secured and controlled?
- [ ] What export control and compliance issues exist?
- [ ] How are collaborative access rights managed?
- [ ] What monitoring exists for data sharing activities?

---

## Resources for Week 6

### Standards and Guidelines
- NIST data classification guidelines
- Export control regulations (ITAR, EAR)
- Scientific data management best practices
- Institutional data governance policies

### Technical Documentation
- Tape storage system documentation
- Data transfer tool security guides
- HSM and archive system manuals
- Collaboration platform security guides

### Compliance Resources
- Export control compliance guides
- Data sovereignty requirements
- Scientific data publication requirements
- Institutional review board guidelines

---

## Success Metrics for Week 6

You should be able to:
- [ ] Design a comprehensive data classification framework
- [ ] Assess the security of data storage and archival systems
- [ ] Evaluate data transfer and collaboration security
- [ ] Create data lifecycle security policies
- [ ] Develop implementation plans for data security improvements

---

## Common Week 6 Challenges and Solutions

**"There's too much data to classify effectively"**
- Start with high-value or high-risk data
- Use automated tools where possible
- Implement classification gradually
- Focus on data that's actively used

**"Scientists resist data classification as bureaucratic overhead"**
- Explain security and compliance benefits
- Make classification as simple as possible
- Integrate with existing workflows
- Provide clear guidance and training

**"Export control requirements are too complex to understand"**
- Get training from export control office
- Start with clear, simple guidelines
- Implement technical controls where possible
- Regular consultation with legal/compliance teams

**"Long-term data preservation seems impossible to secure"**
- Focus on data integrity and access controls
- Plan for technology migration and refresh
- Implement multiple backup strategies
- Regular testing of recovery procedures

---

## Week 6 Wrap-Up

By the end of week 6, you should understand the complete data lifecycle in HPC environments and the security challenges at each stage. Data is the most valuable asset in scientific computing, and protecting it requires comprehensive planning and implementation.

The key insight: Scientific data has unique characteristics (long-term value, collaboration requirements, compliance issues) that require specialized security approaches beyond traditional IT data protection.

**Next week**: We'll move into Month 4 and explore HPC network architecture, including InfiniBand, network segmentation, and the unique challenges of securing high-speed scientific networks.