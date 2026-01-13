# Week 2: Cloud Identity and Access Management
## From Unix Permissions to IAM Policies

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how cloud IAM differs from traditional Unix/LDAP authentication
- Design and implement cloud IAM policies and roles
- Translate HPC user management concepts to cloud identity models
- Implement multi-factor authentication and federated identity in cloud
- Apply least privilege principles in cloud environments
- Integrate cloud identity with existing organizational systems

---

## Day 1: IAM Fundamentals vs. HPC Authentication

### Morning Session (3 hours)

#### HPC Authentication vs. Cloud IAM

**Traditional HPC Authentication (what you know):**
- Unix user accounts with UID/GID
- LDAP/Active Directory for centralized authentication
- SSH keys for passwordless access
- Sudo for privilege escalation
- Group-based permissions (wheel, admin, etc.)
- NIS/NIS+ for account distribution

**Cloud IAM Paradigm:**
- API-driven identity and access management
- Policies define what actions are allowed on which resources
- Roles for temporary credential assumption
- Service accounts for applications
- Fine-grained, resource-specific permissions
- Programmatic access through APIs

#### Core IAM Concepts Translation

| HPC Concept | Cloud IAM Equivalent | Key Difference |
|-------------|---------------------|----------------|
| Unix user account | IAM User | API access vs. shell access |
| Unix group | IAM Group | Policy attachment vs. file permissions |
| sudo privileges | IAM Role assumption | Temporary vs. persistent elevation |
| SSH key | Access Key / API Key | Programmatic vs. interactive access |
| /etc/passwd | IAM User database | Centralized cloud service vs. local file |
| LDAP authentication | Federated identity | Cloud-native vs. directory service |

#### IAM Components Deep Dive

**Users:**
- **HPC equivalent**: Individual user accounts in /etc/passwd
- **Purpose**: Long-term credentials for people
- **Best practice**: Minimize direct user access, prefer roles
- **Security**: MFA, strong passwords, regular rotation

**Groups:**
- **HPC equivalent**: Unix groups (/etc/group)
- **Purpose**: Collect users for easier policy management
- **Best practice**: Organize by job function, not department
- **Security**: Regular membership reviews

**Roles:**
- **HPC equivalent**: Like sudo, but more granular and temporary
- **Purpose**: Temporary credentials for specific tasks
- **Best practice**: Use roles for applications and cross-account access
- **Security**: Short-lived credentials, assume only when needed

**Policies:**
- **HPC equivalent**: Combination of file permissions and sudo rules
- **Purpose**: Define what actions are allowed on which resources
- **Best practice**: Least privilege, specific resource targeting
- **Security**: Regular policy reviews and testing

### Afternoon Session (2 hours)

#### Hands-On IAM Exploration

**Basic IAM Commands (AWS example):**
```bash
# List current user and permissions
aws sts get-caller-identity
aws iam get-user
aws iam list-attached-user-policies --user-name myuser

# List groups and their policies
aws iam list-groups
aws iam get-group --group-name mygroup
aws iam list-attached-group-policies --group-name mygroup

# List roles (like sudo rules)
aws iam list-roles
aws iam get-role --role-name myrole
aws iam list-attached-role-policies --role-name myrole
```

**Understanding IAM Policies:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-research-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

**HPC Permission Equivalent:**
```bash
# This IAM policy is roughly equivalent to:
# - User can read/write files in /projects/research/
# - Only if files are encrypted
# - No other access allowed

# In HPC, this might be:
chmod 750 /projects/research/
chgrp research-team /projects/research/
# Plus custom encryption requirements (harder to enforce)
```

---

## Day 2: Policy Design and Implementation

### Morning Session (3 hours)

#### Policy Design Principles

**Least Privilege Principle:**
- **HPC approach**: Often give broad access, trust users
- **Cloud approach**: Start with no access, add only what's needed
- **Implementation**: Specific actions on specific resources only
- **Monitoring**: Regular access reviews and unused permission cleanup

**Policy Structure Best Practices:**
- **Explicit deny over allow**: Deny takes precedence
- **Resource-specific permissions**: Target specific buckets, instances, etc.
- **Condition-based access**: Time, IP, MFA requirements
- **Regular policy reviews**: Remove unused permissions

#### Common Policy Patterns

**Research Data Access Pattern:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowResearchDataRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::research-data-*",
        "arn:aws:s3:::research-data-*/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

**Compute Resource Management Pattern:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowComputeManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:InstanceType": ["t3.micro", "t3.small", "t3.medium"]
        }
      }
    }
  ]
}
```

### Afternoon Session (2 hours)

#### Hands-On Policy Creation

**Create Research Team Policy:**
```bash
# Create a policy for research team
cat > research-team-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::research-project-*/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::research-project-*"
    }
  ]
}
EOF

# Create the policy
aws iam create-policy \
  --policy-name ResearchTeamDataAccess \
  --policy-document file://research-team-policy.json

# Attach to a group
aws iam attach-group-policy \
  --group-name research-team \
  --policy-arn arn:aws:iam::123456789012:policy/ResearchTeamDataAccess
```

**Policy Testing and Validation:**
```bash
# Test policy with IAM Policy Simulator
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/researcher \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::research-project-alpha/data.txt

# Check effective permissions
aws iam get-account-authorization-details \
  --filter User \
  --max-items 10
```

#### Policy Troubleshooting

**Common Policy Issues:**
- **Overly broad permissions**: Using "*" for actions or resources
- **Missing conditions**: Not restricting by time, location, or MFA
- **Conflicting policies**: Explicit deny overriding intended allow
- **Resource ARN mistakes**: Incorrect resource specification

**Debugging Tools:**
- CloudTrail for access logging
- IAM Policy Simulator for testing
- Access Analyzer for unused permissions
- AWS Config for compliance monitoring

---

## Day 3: Roles and Service Accounts

### Morning Session (3 hours)

#### Understanding IAM Roles

**Roles vs. Users:**
- **Users**: Long-term credentials for people
- **Roles**: Temporary credentials for specific tasks
- **HPC equivalent**: Like sudo, but more granular and auditable

**Role Types:**
- **Service roles**: For AWS services (EC2, Lambda, etc.)
- **Cross-account roles**: For access between AWS accounts
- **Identity provider roles**: For federated access
- **Instance roles**: For EC2 instances (like service accounts)

#### Service Accounts in Cloud

**HPC Service Account Pattern:**
```bash
# In HPC, you might create:
useradd -r -s /bin/false backup-service
# Give it specific permissions for backup tasks
# Use SSH keys or passwords for authentication
```

**Cloud Service Account Pattern:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Role-Based Access Control (RBAC)

**Designing Role Hierarchy:**
- **Administrative roles**: Full access for emergencies
- **Operational roles**: Day-to-day management tasks
- **Developer roles**: Application deployment and debugging
- **Read-only roles**: Monitoring and auditing
- **Service roles**: Automated processes and applications

### Afternoon Session (2 hours)

#### Hands-On Role Management

**Create Service Role for Compute Instances:**
```bash
# Create trust policy (who can assume this role)
cat > ec2-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name ComputeInstanceRole \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach policies to the role
aws iam attach-role-policy \
  --role-name ComputeInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile (AWS-specific concept)
aws iam create-instance-profile \
  --instance-profile-name ComputeInstanceProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name ComputeInstanceProfile \
  --role-name ComputeInstanceRole
```

**Role Assumption and Temporary Credentials:**
```bash
# Assume a role (get temporary credentials)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/ResearcherRole \
  --role-session-name research-session

# Use temporary credentials
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Test access with temporary credentials
aws s3 ls s3://research-data/
```

#### Cross-Account Access

**HPC Equivalent:**
- Like giving users from another institution access to your cluster
- Requires trust relationships and careful permission management

**Cloud Implementation:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PARTNER-ACCOUNT:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

---

## Day 4: Multi-Factor Authentication and Federation

### Morning Session (3 hours)

#### MFA in Cloud vs. HPC

**HPC MFA Challenges:**
- SSH doesn't natively support MFA well
- PAM modules for Google Authenticator or Duo
- VPN-based MFA for network access
- Limited integration with scientific workflows

**Cloud MFA Advantages:**
- Built into IAM system
- API and console access protection
- Conditional access based on MFA status
- Integration with identity providers
- Programmatic MFA for automation

#### MFA Implementation Patterns

**User MFA:**
```bash
# Enable MFA for a user
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name researcher-mfa \
  --path /

# Associate MFA device with user
aws iam enable-mfa-device \
  --user-name researcher \
  --serial-number arn:aws:iam::123456789012:mfa/researcher-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

**MFA-Required Policies:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

#### Federated Identity

**HPC Federation Equivalent:**
- Like using your institutional login for multiple HPC centers
- XSEDE/ACCESS portal authentication
- InCommon federation for research

**Cloud Federation Benefits:**
- Single sign-on with organizational identity
- No separate cloud credentials to manage
- Centralized access control and auditing
- Integration with existing identity providers

### Afternoon Session (2 hours)

#### Hands-On Federation Setup

**SAML Federation Configuration:**
```bash
# Create SAML identity provider
aws iam create-saml-provider \
  --saml-metadata-document file://metadata.xml \
  --name UniversitySSO

# Create role for federated users
cat > saml-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/UniversitySSO"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name FederatedResearcher \
  --assume-role-policy-document file://saml-trust-policy.json
```

**Testing Federated Access:**
```bash
# Simulate SAML assertion (normally done by identity provider)
aws sts assume-role-with-saml \
  --role-arn arn:aws:iam::123456789012:role/FederatedResearcher \
  --principal-arn arn:aws:iam::123456789012:saml-provider/UniversitySSO \
  --saml-assertion file://saml-assertion.xml
```

#### Identity Provider Integration

**Common Identity Providers:**
- **Active Directory**: Windows-based directory service
- **LDAP**: Lightweight Directory Access Protocol
- **SAML**: Security Assertion Markup Language
- **OpenID Connect**: Modern web-based authentication
- **OAuth 2.0**: Authorization framework

**Integration Patterns:**
- Direct federation with cloud provider
- Identity broker for multiple clouds
- Just-in-time provisioning
- Attribute-based access control

---

## Day 5: Advanced IAM and Security Best Practices

### Morning Session (3 hours)

#### Advanced IAM Features

**Conditional Access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        },
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

**Permission Boundaries:**
- Maximum permissions a user/role can have
- Useful for delegated administration
- Prevents privilege escalation
- Complements regular policies

**Access Analyzer:**
- Identifies resources shared with external entities
- Finds unused access permissions
- Validates policies against best practices
- Generates policies based on access patterns

#### Security Best Practices

**Credential Management:**
- Rotate access keys regularly (90 days max)
- Use roles instead of long-term credentials
- Never embed credentials in code
- Use credential management services

**Monitoring and Auditing:**
- Enable CloudTrail for all API calls
- Monitor failed authentication attempts
- Set up alerts for privilege escalation
- Regular access reviews and cleanup

### Afternoon Session (2 hours)

#### Hands-On Security Implementation

**Implement Security Monitoring:**
```bash
# Create CloudTrail for API logging
aws cloudtrail create-trail \
  --name security-audit-trail \
  --s3-bucket-name security-logs-bucket \
  --include-global-service-events \
  --is-multi-region-trail

# Enable the trail
aws cloudtrail start-logging \
  --name security-audit-trail

# Create CloudWatch alarm for root account usage
aws cloudwatch put-metric-alarm \
  --alarm-name "Root Account Usage" \
  --alarm-description "Alert when root account is used" \
  --metric-name "RootAccountUsage" \
  --namespace "AWS/CloudTrail" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold
```

**Access Review Automation:**
```python
#!/usr/bin/env python3
import boto3
from datetime import datetime, timedelta

def audit_unused_access():
    """Audit unused IAM access keys and roles"""
    iam = boto3.client('iam')
    
    # Check for old access keys
    users = iam.list_users()['Users']
    for user in users:
        keys = iam.list_access_keys(UserName=user['UserName'])['AccessKeyMetadata']
        for key in keys:
            age = datetime.now(key['CreateDate'].tzinfo) - key['CreateDate']
            if age > timedelta(days=90):
                print(f"WARNING: Access key {key['AccessKeyId']} is {age.days} days old")
    
    # Check for unused roles
    roles = iam.list_roles()['Roles']
    for role in roles:
        try:
            last_used = iam.get_role(RoleName=role['RoleName'])['Role'].get('RoleLastUsed')
            if last_used and 'LastUsedDate' in last_used:
                age = datetime.now(last_used['LastUsedDate'].tzinfo) - last_used['LastUsedDate']
                if age > timedelta(days=90):
                    print(f"WARNING: Role {role['RoleName']} unused for {age.days} days")
        except Exception as e:
            print(f"Error checking role {role['RoleName']}: {e}")

if __name__ == "__main__":
    audit_unused_access()
```

#### Migration Planning

**HPC to Cloud IAM Migration:**
1. **Inventory current access**: Document all HPC users and their permissions
2. **Map to cloud roles**: Design cloud roles based on job functions
3. **Implement federation**: Connect to existing identity systems
4. **Gradual migration**: Move users in phases with testing
5. **Cleanup and optimization**: Remove unused permissions and accounts

---

## Week 2 Deliverable: Cloud IAM Implementation Plan

Create a comprehensive document (8-12 pages) covering:

### Executive Summary (1-2 pages)
- Current HPC authentication vs. desired cloud IAM state
- Key benefits and challenges of cloud IAM adoption
- Implementation timeline and resource requirements
- Success metrics and validation approaches

### Current State Analysis (2-3 pages)

**HPC Authentication Assessment:**
- Current user management processes
- Authentication methods and systems
- Permission models and group structures
- Integration with organizational identity systems

**Gap Analysis:**
- Limitations of current HPC authentication
- Security vulnerabilities and risks
- Operational inefficiencies
- Compliance and audit challenges

### Cloud IAM Design (3-4 pages)

**Identity Architecture:**
- User, group, and role design
- Policy structure and hierarchy
- Service account strategy
- Federation and SSO implementation

**Security Controls:**
- Multi-factor authentication requirements
- Conditional access policies
- Permission boundaries and guardrails
- Monitoring and auditing capabilities

### Implementation Roadmap (2-3 pages)

**Phase 1: Foundation (0-3 months):**
- Basic IAM setup and user migration
- Core policy implementation
- MFA enablement
- Basic monitoring setup

**Phase 2: Enhancement (3-6 months):**
- Federation implementation
- Advanced policy features
- Automated access reviews
- Integration with organizational systems

**Phase 3: Optimization (6-12 months):**
- Advanced security features
- Automation and orchestration
- Continuous improvement processes
- Full organizational integration

---

## Key Security Questions Answered This Week

By the end of week 2, you should be able to answer:

**About Cloud IAM:**
- [ ] How does cloud IAM differ from traditional Unix/LDAP authentication?
- [ ] What are the key components of cloud identity management?
- [ ] How do policies and roles work in cloud environments?
- [ ] What are the security benefits and challenges of cloud IAM?

**About Implementation:**
- [ ] How should HPC user management be migrated to cloud IAM?
- [ ] What federation options exist for organizational integration?
- [ ] How should MFA be implemented for cloud access?
- [ ] What monitoring and auditing capabilities are needed?

**About Best Practices:**
- [ ] What are the key security best practices for cloud IAM?
- [ ] How should access be reviewed and managed over time?
- [ ] What automation opportunities exist for identity management?
- [ ] How should cloud IAM integrate with existing security programs?

---

## Resources for Week 2

### Technical Documentation
- AWS IAM User Guide
- Azure Active Directory documentation
- Google Cloud Identity and Access Management
- SAML and OpenID Connect specifications

### Security Best Practices
- Cloud security frameworks (CSA, NIST)
- Identity and access management best practices
- Multi-factor authentication implementation guides
- Federation and SSO deployment guides

### Tools and Services
- Cloud provider IAM services
- Identity provider solutions
- MFA applications and hardware tokens
- Policy management and automation tools

---

## Success Metrics for Week 2

You should be able to:
- [ ] Design cloud IAM architecture for HPC users
- [ ] Create and manage IAM policies and roles
- [ ] Implement MFA and federation for cloud access
- [ ] Plan migration from HPC authentication to cloud IAM
- [ ] Establish monitoring and auditing for cloud identity

---

## Common Week 2 Challenges and Solutions

**"Cloud IAM seems overly complex compared to Unix permissions"**
- Start with basic user and group concepts you know
- Focus on the principle of least privilege
- Use managed policies before creating custom ones
- Practice with simple scenarios before complex ones

**"The policy language is confusing and error-prone"**
- Use policy generators and templates initially
- Test policies with the IAM simulator
- Start with AWS managed policies and customize
- Use version control for policy management

**"Federation setup seems too complex for our environment"**
- Start with basic SAML federation
- Work with your identity team for integration
- Consider cloud-native identity providers initially
- Implement gradually with pilot groups

**"Users will resist changing from SSH keys to cloud authentication"**
- Emphasize security and compliance benefits
- Provide comprehensive training and documentation
- Implement gradually with support available
- Show how cloud access can be more convenient

---

## Week 2 Wrap-Up

By the end of week 2, you should understand how cloud identity and access management differs from traditional HPC authentication and how to implement it effectively. Cloud IAM provides much more granular control and better security than traditional Unix-based systems, but requires a different mindset and approach.

The key insight: Cloud IAM is about API-driven, policy-based access control rather than file-based permissions. It's more complex initially but provides much better security, auditability, and integration capabilities.

**Next week**: We'll explore cloud networking and security, including how Virtual Private Clouds (VPCs) differ from physical network segmentation and how to implement network security in cloud environments.