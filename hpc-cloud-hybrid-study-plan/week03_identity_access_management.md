# Week 03: Identity & Access Management Across Boundaries

## Learning Objectives
- Implement federated identity systems spanning HPC and cloud environments
- Configure single sign-on (SSO) for hybrid access
- Design role-based access control (RBAC) across platforms
- Secure service-to-service authentication in hybrid architectures
- Implement compliance and audit frameworks for hybrid identity

## Overview
Identity and Access Management (IAM) in hybrid HPC-cloud environments presents unique challenges. This week covers federated identity, cross-platform authentication, and security best practices for managing user and service identities across boundaries.

## Key Concepts

### Federated Identity Architecture
- **Identity Providers (IdP)**: Centralized authentication sources
- **Service Providers (SP)**: Applications trusting the IdP
- **Federation Protocols**: SAML, OAuth 2.0, OpenID Connect
- **Trust Relationships**: Establishing secure identity federation

### Hybrid Identity Challenges
- **Protocol Differences**: HPC systems vs cloud-native authentication
- **Legacy Integration**: Connecting older HPC systems with modern cloud IAM
- **Performance Requirements**: Low-latency authentication for HPC workloads
- **Compliance**: Meeting regulatory requirements across environments

## Hands-On Lab: Federated Identity Implementation

### Lab Setup: Multi-Environment Identity
- Configure Active Directory as primary IdP
- Integrate with AWS IAM using SAML federation
- Setup LDAP authentication for HPC cluster
- Implement cross-platform role mapping

### Exercise 1: SAML Federation Setup (90 minutes)

#### Configure SAML Identity Provider
```xml
<!-- SAML Assertion example -->
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
      user@hpc-domain.edu
    </saml:NameID>
  </saml:Subject>
  <saml:AttributeStatement>
    <saml:Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
      <saml:AttributeValue>
        arn:aws:iam::123456789012:role/HPCResearcher,
        arn:aws:iam::123456789012:saml-provider/HPCIdentityProvider
      </saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

#### AWS IAM Role Configuration
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/HPCIdentityProvider"
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
```

### Exercise 2: Cross-Platform Role Mapping (60 minutes)

#### HPC Cluster Integration
```bash
# Configure SSSD for LDAP authentication
# /etc/sssd/sssd.conf
[domain/hpc.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://hpc-dc.local
ldap_search_base = dc=hpc,dc=local
ldap_user_search_base = ou=users,dc=hpc,dc=local
ldap_group_search_base = ou=groups,dc=hpc,dc=local

# Map LDAP groups to local groups
[nss]
override_gid = 1001:researchers
override_gid = 1002:administrators
```

#### SLURM Integration
```bash
# Configure SLURM for LDAP authentication
# /etc/slurm/slurm.conf
AuthType=auth/munge
AuthInfo=/var/run/munge/munge.socket.2

# Account mapping configuration
# /etc/slurm/slurm_accounts.conf
researchers:MaxJobs=100:MaxSubmitJobs=200
administrators:MaxJobs=1000:MaxSubmitJobs=2000
```

## Security Best Practices

### Multi-Factor Authentication
```python
# Python example for MFA integration
import pyotp
import qrcode

def setup_mfa(user_id, secret_key):
    totp = pyotp.TOTP(secret_key)
    provisioning_uri = totp.provisioning_uri(
        user_id,
        issuer_name="HPC-Cloud Hybrid"
    )
    
    # Generate QR code for mobile app
    qr = qrcode.QRCode(version=1, box_size=10, border=5)
    qr.add_data(provisioning_uri)
    qr.make(fit=True)
    return qr.make_image(fill_color="black", back_color="white")
```

### Service Account Management
```yaml
# Kubernetes ServiceAccount for hybrid workloads
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hpc-cloud-service
  namespace: hybrid-compute
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/HPCCloudServiceRole
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hpc-cloud-binding
subjects:
- kind: ServiceAccount
  name: hpc-cloud-service
  namespace: hybrid-compute
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## Week 3 Deliverables

### Implementation Report
1. **Federation Architecture Design** (2 pages)
   - Identity flow diagrams
   - Trust relationship documentation
   - Security boundary analysis

2. **Lab Implementation Results** (2 pages)
   - Configuration screenshots
   - Test results and validation
   - Performance measurements

3. **Security Assessment** (1 page)
   - Threat model analysis
   - Risk mitigation strategies
   - Compliance considerations

### Practical Exercise
- Configure federated authentication for your lab environment
- Test cross-platform access scenarios
- Document troubleshooting procedures

---

*Secure identity management is critical for hybrid success. Get authentication right, and authorization becomes much simpler.*