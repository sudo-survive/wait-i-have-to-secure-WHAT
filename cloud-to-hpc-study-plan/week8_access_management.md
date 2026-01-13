# Week 8: Access Management & Authentication
## Modernizing Authentication and Authorization for HPC

---

## Learning Objectives

By the end of this week, you should be able to:
- Assess current HPC authentication methods and their security implications
- Design multi-factor authentication implementation for HPC environments
- Evaluate SSH key management at scale
- Analyze privileged access management for HPC administrators
- Plan integration with organizational identity management systems
- Apply Zero Trust principles to HPC environments

---

## Day 1: Current HPC Authentication Methods

### Morning Session (3 hours)

#### Traditional HPC Authentication

**Unix/Linux Authentication:**
- Local user accounts on all nodes
- Password-based authentication
- SSH key-based authentication
- NIS/NIS+ for centralized account management
- LDAP integration for directory services

**Characteristics of HPC Authentication:**
- Designed for trusted user communities
- Emphasis on usability over security
- Long-lived credentials (SSH keys)
- Shared accounts common (bad practice)
- Limited audit trails

#### Current Authentication Landscape

**Password Authentication:**
- Still common in many HPC environments
- Often weak password policies
- Password reuse across systems
- Limited password complexity requirements
- Infrequent password changes

**SSH Key Authentication:**
- Preferred method for HPC access
- Often no passphrase protection
- Keys rarely rotated
- Difficult to manage at scale
- Limited audit capabilities

**Kerberos Authentication:**
- More secure but complex to implement
- Good for single sign-on
- Requires significant infrastructure
- Not widely adopted in HPC
- Integration challenges with HPC software

### Afternoon Session (2 hours)

#### Hands-On Authentication Assessment

**Current Authentication Analysis:**
```bash
# Check authentication methods
cat /etc/ssh/sshd_config | grep -E "(PasswordAuthentication|PubkeyAuthentication|KerberosAuthentication)"

# Check user account information
getent passwd | wc -l
cat /etc/passwd | grep -E "(nologin|false)$" | wc -l

# Check for centralized authentication
cat /etc/nsswitch.conf | grep passwd
cat /etc/pam.d/sshd

# Check SSH key usage
find /home -name ".ssh" -type d | wc -l
find /home -name "authorized_keys" -exec wc -l {} \; | awk '{sum+=$1} END {print sum}'

# Check for shared accounts
awk -F: '$3 >= 1000 {print $1}' /etc/passwd | sort
```

**Authentication Security Assessment:**
- [ ] What authentication methods are currently in use?
- [ ] How are user accounts managed and provisioned?
- [ ] What password policies exist?
- [ ] How are SSH keys managed?
- [ ] What audit logging exists for authentication events?
- [ ] Are there any shared or service accounts?

#### Authentication Security Issues

**Common Problems:**
- Weak or default passwords
- Unprotected SSH private keys
- Shared accounts and passwords
- No multi-factor authentication
- Poor key management practices
- Limited authentication logging

**Security Implications:**
- Credential compromise leads to system access
- Lateral movement through shared keys
- Difficult to attribute actions to individuals
- Limited ability to revoke access quickly
- Compliance violations

---

## Day 2: Multi-Factor Authentication for HPC

### Morning Session (3 hours)

#### MFA Requirements for HPC

**Why MFA is Critical:**
- HPC systems are high-value targets
- Single-factor authentication is insufficient
- Compliance requirements (NIST 800-53, etc.)
- Protection against credential compromise
- Enhanced audit and accountability

**HPC-Specific MFA Challenges:**
- Batch job authentication
- Automated workflows and scripts
- Performance impact concerns
- User resistance to complexity
- Integration with existing tools

#### MFA Technologies for HPC

**Time-based One-Time Passwords (TOTP):**
- Google Authenticator, Authy, etc.
- Works with existing infrastructure
- No additional hardware required
- Good user experience
- Offline capability

**Hardware Security Keys:**
- FIDO2/WebAuthn standards
- YubiKey, RSA SecurID, etc.
- Highest security level
- Phishing resistant
- Higher cost and complexity

**SMS/Voice-based MFA:**
- Easy to implement
- Works with any phone
- Vulnerable to SIM swapping
- Not recommended for high-security environments

**Push Notifications:**
- Duo, Microsoft Authenticator, etc.
- Good user experience
- Requires internet connectivity
- Potential for push fatigue

### Afternoon Session (2 hours)

#### MFA Implementation Planning

**MFA Architecture Options:**

**SSH-based MFA:**
- PAM modules for MFA integration
- Google Authenticator PAM module
- Duo Unix integration
- Custom MFA solutions

**VPN + MFA:**
- MFA at VPN level
- All HPC access through VPN
- Centralized MFA enforcement
- Network-level protection

**Jump Host + MFA:**
- MFA on bastion/jump hosts
- Single point of MFA enforcement
- Easier to implement and manage
- Potential single point of failure

#### Hands-On MFA Assessment

**Current MFA Capabilities:**
```bash
# Check for existing MFA modules
ls /lib/security/ | grep -E "(pam_google|pam_duo|pam_oath)"
ls /usr/lib/x86_64-linux-gnu/security/ | grep -E "(pam_google|pam_duo|pam_oath)"

# Check PAM configuration
cat /etc/pam.d/sshd | grep -E "(google|duo|oath)"

# Check for VPN infrastructure
ps aux | grep -E "(openvpn|strongswan|ipsec)"

# Check for jump host architecture
# (ask operations team about access patterns)
```

**MFA Implementation Assessment:**
- [ ] What MFA technologies are currently deployed?
- [ ] What infrastructure exists for MFA implementation?
- [ ] What are user preferences and requirements?
- [ ] What compliance requirements exist for MFA?
- [ ] What budget and resources are available?

---

## Day 3: SSH Key Management at Scale

### Morning Session (3 hours)

#### SSH Key Management Challenges

**Scale Issues:**
- Thousands of users with multiple keys
- Keys distributed across hundreds/thousands of nodes
- No centralized key management
- Difficult to track key usage
- Hard to revoke compromised keys

**Security Issues:**
- Keys without passphrases
- Keys never rotated
- Orphaned keys from former users
- Shared keys between users
- Keys with excessive privileges

**Operational Issues:**
- Manual key distribution
- No automated key rotation
- Difficult troubleshooting
- Poor documentation
- Inconsistent key formats

#### SSH Key Management Solutions

**Centralized Key Management:**
- LDAP-based key storage
- Database-driven key management
- Configuration management tools
- Custom key management systems

**SSH Certificate Authorities:**
- Short-lived certificates instead of keys
- Centralized certificate issuance
- Automatic certificate rotation
- Better audit and control
- Reduced key distribution complexity

**Commercial Solutions:**
- SSH.COM Universal SSH Key Manager
- CyberArk Privileged Access Manager
- BeyondTrust Privileged Remote Access
- Venafi SSH Protect

### Afternoon Session (2 hours)

#### Hands-On SSH Key Analysis

**SSH Key Inventory:**
```bash
# Find all SSH keys
find /home -name "*.pub" -o -name "authorized_keys" | head -20

# Analyze key types and sizes
find /home -name "*.pub" -exec ssh-keygen -l -f {} \; | head -10

# Check for weak keys
find /home -name "*.pub" -exec ssh-keygen -l -f {} \; | grep -E "(1024|512)"

# Look for duplicate keys
find /home -name "authorized_keys" -exec cat {} \; | sort | uniq -d | head -5

# Check key age (if possible)
find /home -name "*.pub" -exec stat -c "%Y %n" {} \; | sort -n | head -10
```

**SSH Key Security Assessment:**
- [ ] How many SSH keys exist in the environment?
- [ ] What key types and sizes are in use?
- [ ] Are there weak or duplicate keys?
- [ ] How are keys distributed and managed?
- [ ] What processes exist for key rotation?
- [ ] How are keys revoked when users leave?

#### SSH Key Management Best Practices

**Key Generation:**
- Strong key algorithms (RSA 2048+, Ed25519)
- Mandatory passphrases
- Secure key generation processes
- Key naming conventions
- Documentation requirements

**Key Distribution:**
- Automated key distribution
- Secure key transfer methods
- Version control for authorized_keys
- Regular key synchronization
- Audit trails for key changes

**Key Lifecycle Management:**
- Regular key rotation (annually)
- Automated key expiration
- Key revocation procedures
- Orphaned key cleanup
- Key usage monitoring

---

## Day 4: Privileged Access Management

### Morning Session (3 hours)

#### Privileged Access in HPC

**Types of Privileged Access:**
- Root access on compute nodes
- Administrative access to job schedulers
- Storage system administration
- Network infrastructure management
- Security system administration

**HPC-Specific Privileged Access Challenges:**
- Shared administrative responsibilities
- Emergency access requirements
- Automated system processes
- Service account management
- Cross-system dependencies

#### Privileged Access Management (PAM) Solutions

**Traditional Approaches:**
- Sudo configuration management
- Shared root passwords
- Individual administrative accounts
- Role-based access control
- Manual access reviews

**Modern PAM Solutions:**
- Just-in-time access provisioning
- Session recording and monitoring
- Automated access reviews
- Password vaulting
- Privileged session management

**Commercial PAM Tools:**
- CyberArk Privileged Access Manager
- BeyondTrust Privileged Remote Access
- Thycotic Secret Server
- HashiCorp Vault
- AWS Systems Manager Session Manager

### Afternoon Session (2 hours)

#### Hands-On Privileged Access Assessment

**Current Privileged Access Analysis:**
```bash
# Check sudo configuration
cat /etc/sudoers
ls -la /etc/sudoers.d/

# Check for shared accounts
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Check for service accounts
awk -F: '$3 < 1000 && $3 > 0 {print $1}' /etc/passwd

# Check for privileged group memberships
getent group sudo
getent group wheel
getent group admin

# Check for SUID/SGID files
find /usr -perm -4000 -o -perm -2000 | head -20
```

**Privileged Access Security Assessment:**
- [ ] How is privileged access currently managed?
- [ ] What shared administrative accounts exist?
- [ ] How are sudo privileges configured?
- [ ] What service accounts exist and how are they managed?
- [ ] What audit logging exists for privileged access?
- [ ] How are privileged credentials stored and managed?

#### Service Account Management

**Service Account Challenges:**
- Automated processes requiring authentication
- Long-lived credentials
- Shared service accounts
- Difficult to rotate credentials
- Limited audit visibility

**Service Account Best Practices:**
- Dedicated service accounts for each service
- Principle of least privilege
- Regular credential rotation
- Secure credential storage
- Comprehensive audit logging

---

## Day 5: Identity Management Integration and Zero Trust

### Morning Session (3 hours)

#### Integration with Organizational Identity Systems

**Identity Management Integration Benefits:**
- Centralized user management
- Consistent access policies
- Automated provisioning/deprovisioning
- Single sign-on capabilities
- Enhanced audit and compliance

**Common Identity Systems:**
- Active Directory
- LDAP directories
- SAML identity providers
- OAuth/OpenID Connect
- Cloud identity services (Azure AD, AWS IAM)

**Integration Challenges:**
- Legacy HPC software compatibility
- Performance impact of remote authentication
- Network connectivity requirements
- Complex attribute mapping
- User experience considerations

#### Federated Identity for HPC

**Research Identity Federations:**
- InCommon (US higher education)
- eduGAIN (global research federation)
- ORCID (researcher identifiers)
- Institutional identity providers

**Benefits of Federation:**
- Simplified external collaboration
- Reduced account management overhead
- Enhanced security through professional identity providers
- Compliance with research community standards

### Afternoon Session (2 hours)

#### Zero Trust Principles for HPC

**Zero Trust Core Principles:**
- Never trust, always verify
- Least privilege access
- Assume breach mentality
- Continuous monitoring and validation
- Identity-centric security

**Applying Zero Trust to HPC:**
- Identity verification for every access
- Device trust and compliance
- Application-level access controls
- Continuous risk assessment
- Micro-segmentation of resources

**Zero Trust Implementation Challenges:**
- Performance impact of continuous verification
- Complexity of HPC environments
- User experience considerations
- Legacy system compatibility
- Cultural resistance to change

#### Hands-On Identity Integration Assessment

**Current Identity Integration:**
```bash
# Check for LDAP integration
cat /etc/nsswitch.conf | grep ldap
cat /etc/pam.d/system-auth | grep ldap

# Check for Kerberos integration
klist
cat /etc/krb5.conf

# Check for SSSD configuration
systemctl status sssd
cat /etc/sssd/sssd.conf

# Check for federation capabilities
# (ask about SAML, OAuth integration)
```

**Identity Integration Assessment:**
- [ ] What organizational identity systems exist?
- [ ] How is HPC currently integrated with identity systems?
- [ ] What federation capabilities exist?
- [ ] What are the integration gaps and challenges?
- [ ] What zero trust principles can be applied?

---

## Week 8 Deliverable: Access Management Modernization Plan

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Current access management security posture
- Key authentication and authorization risks
- Recommended modernization approach
- Implementation timeline and resource requirements

### Current State Analysis (3-4 pages)

**Authentication Assessment:**
- Current authentication methods and their security
- User account management processes
- SSH key management practices
- Multi-factor authentication status
- Privileged access management approach

**Gap Analysis:**
- Missing or inadequate security controls
- Compliance violations or risks
- Operational inefficiencies
- User experience issues
- Integration challenges

### Modernization Recommendations (4-6 pages)

**Multi-Factor Authentication Implementation:**
- Recommended MFA technologies and architecture
- Implementation approach and timeline
- User training and change management
- Performance and usability considerations

**SSH Key Management Enhancement:**
- Centralized key management solution
- Key lifecycle management processes
- Automated key rotation and distribution
- Security monitoring and audit capabilities

**Privileged Access Management:**
- PAM solution recommendations
- Just-in-time access implementation
- Service account management improvements
- Session monitoring and recording

**Identity Management Integration:**
- Integration with organizational identity systems
- Federated identity implementation
- Single sign-on capabilities
- Zero trust architecture principles

### Implementation Roadmap (3-4 pages)

**Phase 1: Foundation (0-6 months):**
- Basic MFA implementation
- SSH key management improvements
- Privileged access controls enhancement
- Identity system integration planning

**Phase 2: Enhancement (6-12 months):**
- Advanced MFA deployment
- Centralized key management implementation
- PAM solution deployment
- Federation and SSO implementation

**Phase 3: Optimization (12-18 months):**
- Zero trust architecture implementation
- Advanced monitoring and analytics
- Automation and orchestration
- Continuous improvement processes

### Resource Requirements and Success Metrics (1-2 pages)
- Budget and staffing requirements
- Technology and infrastructure needs
- Training and change management resources
- Success metrics and validation approaches

---

## Key Security Questions Answered This Week

By the end of week 8, you should be able to answer:

**About Authentication:**
- [ ] How secure are current HPC authentication methods?
- [ ] What MFA implementation approach is most appropriate?
- [ ] How should SSH keys be managed at scale?
- [ ] What are the compliance requirements for authentication?

**About Access Management:**
- [ ] How is privileged access currently managed and secured?
- [ ] What integration opportunities exist with organizational identity systems?
- [ ] How can zero trust principles be applied to HPC?
- [ ] What are the highest-priority access management improvements?

**About Implementation:**
- [ ] What are the technical and operational challenges?
- [ ] How should access management modernization be prioritized?
- [ ] What resources are required for implementation?
- [ ] How should success be measured and validated?

---

## Resources for Week 8

### Standards and Guidelines
- NIST 800-63 Digital Identity Guidelines
- NIST 800-53 Access Control requirements
- Zero Trust Architecture (NIST SP 800-207)
- Multi-factor authentication best practices

### Technical Documentation
- SSH key management best practices
- PAM module documentation
- Identity federation standards (SAML, OAuth)
- MFA technology documentation

### Tools and Solutions
- Open source MFA solutions
- Commercial PAM platforms
- SSH key management tools
- Identity integration solutions

---

## Success Metrics for Week 8

You should be able to:
- [ ] Assess current HPC authentication and access management security
- [ ] Design appropriate MFA implementation for HPC environments
- [ ] Plan SSH key management at scale
- [ ] Evaluate privileged access management solutions
- [ ] Create comprehensive access management modernization plans

---

## Common Week 8 Challenges and Solutions

**"Users will resist MFA as too complex or slow"**
- Choose user-friendly MFA technologies
- Implement gradually with pilot groups
- Provide comprehensive training and support
- Communicate security benefits clearly

**"SSH key management seems too complex to implement"**
- Start with basic centralized key storage
- Implement automated key distribution
- Focus on new keys first, migrate existing keys gradually
- Use configuration management tools for automation

**"Privileged access management tools are too expensive"**
- Start with open source solutions
- Focus on highest-risk privileged access first
- Build business case with risk reduction benefits
- Consider cloud-based PAM solutions

**"Integration with organizational identity systems is too complex"**
- Start with basic LDAP integration
- Work with organizational IT team
- Implement federation gradually
- Focus on user experience and benefits

---

## Week 8 Wrap-Up

By the end of week 8, you should understand the critical importance of modernizing access management in HPC environments. Traditional HPC authentication methods are insufficient for current security requirements and compliance needs.

The key insight: Access management modernization is essential for HPC security, but it must be implemented thoughtfully to maintain usability and performance. A phased approach with strong change management is critical for success.

**Next week**: We'll move into Month 5 and explore security monitoring for HPC, including adapting monitoring expertise to HPC-specific threats, log sources, and integration with existing SIEM infrastructure.