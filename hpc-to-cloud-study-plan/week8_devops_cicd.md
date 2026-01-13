# Week 8: DevOps and CI/CD Security
## From Manual Builds to Automated Secure Pipelines

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how CI/CD differs from manual HPC software builds and deployments
- Implement secure CI/CD pipelines with automated security testing
- Apply Infrastructure as Code (IaC) principles with security controls
- Design GitOps workflows for secure configuration management
- Implement secrets management and secure artifact handling
- Create comprehensive testing strategies including security testing

---

## Day 1: CI/CD Pipeline Security

### Morning Session (3 hours)

#### HPC vs. DevOps Deployment Models

**Traditional HPC Software Deployment:**
- Manual compilation and installation
- System administrator deploys software
- Infrequent updates and patches
- Manual testing and validation
- Shared software installations

**CI/CD Pipeline Approach:**
- Automated build, test, and deployment
- Developers trigger deployments
- Frequent, incremental updates
- Automated testing at every stage
- Containerized, isolated deployments

#### Secure Pipeline Design

**GitHub Actions Security Pipeline:**
```yaml
name: Secure CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run SAST Scan
      uses: github/super-linter@v4
      env:
        DEFAULT_BRANCH: main
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Run Dependency Check
      run: |
        pip install safety
        safety check --json --output safety-report.json
    
    - name: Container Security Scan
      run: |
        docker build -t research-app:${{ github.sha }} .
        trivy image research-app:${{ github.sha }}

  deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Deploy to Production
      run: |
        # Secure deployment steps
        echo "Deploying to production..."
```

### Afternoon Session (2 hours)

#### Pipeline Security Controls

**Secrets Management:**
```yaml
# Secure secrets handling
- name: Deploy Application
  env:
    DATABASE_PASSWORD: ${{ secrets.DB_PASSWORD }}
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    # Use secrets securely without logging
    kubectl create secret generic app-secrets \
      --from-literal=db-password="$DATABASE_PASSWORD" \
      --from-literal=api-key="$API_KEY"
```

**Artifact Security:**
```bash
# Sign and verify artifacts
cosign sign --key cosign.key research-app:v1.0
cosign verify --key cosign.pub research-app:v1.0

# SBOM generation
syft research-app:v1.0 -o spdx-json > sbom.json
```

---

## Day 2: Infrastructure as Code Security

### Morning Session (3 hours)

#### IaC Security Best Practices

**Terraform Security Configuration:**
```hcl
# Secure S3 bucket configuration
resource "aws_s3_bucket" "research_data" {
  bucket = "research-data-${random_id.bucket_suffix.hex}"
}

resource "aws_s3_bucket_encryption" "research_data" {
  bucket = aws_s3_bucket.research_data.id
  
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_public_access_block" "research_data" {
  bucket = aws_s3_bucket.research_data.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Afternoon Session (2 hours)

#### IaC Security Scanning

**Automated Security Scanning:**
```bash
# Terraform security scanning
tfsec .
checkov -f main.tf
terrascan scan -t terraform

# CloudFormation scanning
cfn-lint template.yaml
cfn_nag_scan --input-path template.yaml
```

---

## Day 3: GitOps and Configuration Management

### Morning Session (3 hours)

#### GitOps Security Model

**ArgoCD Secure Configuration:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: research-app
  namespace: argocd
spec:
  project: research
  source:
    repoURL: https://github.com/research/app-config
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: research
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Afternoon Session (2 hours)

#### Secure Configuration Management

**Sealed Secrets Implementation:**
```bash
# Create sealed secret
echo -n mypassword | kubectl create secret generic mysecret \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml
```

---

## Day 4: Testing Strategies

### Morning Session (3 hours)

#### Comprehensive Testing Pipeline

**Security Testing Integration:**
```yaml
test:
  stage: test
  script:
    - pytest tests/
    - bandit -r src/
    - safety check
    - docker run --rm -v $(pwd):/app owasp/zap2docker-stable zap-baseline.py -t http://localhost:8080
```

### Afternoon Session (2 hours)

#### Performance and Load Testing

**Load Testing with Security Focus:**
```python
from locust import HttpUser, task, between

class SecurityLoadTest(HttpUser):
    wait_time = between(1, 3)
    
    @task
    def test_authenticated_endpoint(self):
        # Test with valid authentication
        headers = {"Authorization": f"Bearer {self.token}"}
        self.client.get("/api/data", headers=headers)
    
    @task
    def test_rate_limiting(self):
        # Test rate limiting behavior
        for i in range(100):
            response = self.client.get("/api/compute")
            if response.status_code == 429:
                break
```

---

## Day 5: Monitoring and Compliance

### Morning Session (3 hours)

#### Pipeline Monitoring

**Security Metrics Collection:**
```python
import prometheus_client
from prometheus_client import Counter, Histogram, Gauge

# Security metrics
security_scan_failures = Counter('security_scan_failures_total', 'Total security scan failures')
deployment_duration = Histogram('deployment_duration_seconds', 'Deployment duration')
vulnerability_count = Gauge('vulnerabilities_found', 'Number of vulnerabilities found')

def collect_security_metrics():
    # Collect and expose security metrics
    vulnerability_count.set(get_vulnerability_count())
    security_scan_failures.inc()
```

### Afternoon Session (2 hours)

#### Compliance Automation

**Automated Compliance Checking:**
```bash
#!/bin/bash
# Compliance validation script
inspec exec compliance-profile/ --reporter json:compliance-report.json
```

---

## Week 8 Deliverable: Secure CI/CD Implementation Plan

Create a comprehensive document (10-15 pages) covering:

### Executive Summary (1-2 pages)
- DevOps security approach vs. manual HPC processes
- Key benefits and security improvements
- Implementation timeline and resource requirements

### CI/CD Security Architecture (3-4 pages)
- Secure pipeline design and implementation
- Security testing integration
- Secrets management and artifact security
- Access controls and approval processes

### Infrastructure as Code Security (3-4 pages)
- IaC security best practices and scanning
- Configuration management and GitOps
- Compliance automation and validation
- Change management and audit trails

### Testing and Quality Assurance (2-3 pages)
- Comprehensive testing strategies
- Security testing automation
- Performance and load testing
- Compliance and regulatory testing

### Monitoring and Continuous Improvement (2-3 pages)
- Pipeline monitoring and metrics
- Security incident response
- Continuous improvement processes
- Training and skill development

---

## Success Metrics for Week 8

You should be able to:
- [ ] Design and implement secure CI/CD pipelines
- [ ] Apply Infrastructure as Code with security controls
- [ ] Implement comprehensive security testing
- [ ] Design GitOps workflows with proper security
- [ ] Create monitoring and compliance automation

---

## Week 8 Wrap-Up

By the end of week 8, you should understand how DevOps and CI/CD practices can significantly improve security through automation, consistency, and comprehensive testing. The shift from manual processes to automated pipelines enables better security controls and faster response to security issues.

**Next week**: We'll explore databases and data services in the cloud, including managed databases, data warehouses, and analytics services for scientific data.