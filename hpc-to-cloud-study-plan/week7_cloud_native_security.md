# Week 7: Cloud-Native Application Security
## Securing Microservices, APIs, and Distributed Applications

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how cloud-native application security differs from monolithic HPC applications
- Implement API security controls including authentication, authorization, and rate limiting
- Design secure microservices architectures with service mesh and zero trust principles
- Apply application security testing (SAST/DAST) to cloud-native applications
- Implement runtime protection with WAF, DDoS protection, and behavioral monitoring
- Secure container-based applications and orchestration platforms

---

## Day 1: API Security Fundamentals

### Morning Session (3 hours)

#### HPC vs. Cloud-Native Communication

**HPC Application Communication (what you know):**
- MPI for parallel processing communication
- Shared memory and file-based data exchange
- Direct function calls within monolithic applications
- SSH and secure shell access for remote operations
- Limited external interfaces

**Cloud-Native API Communication:**
- REST APIs for service-to-service communication
- GraphQL for flexible data queries
- gRPC for high-performance service communication
- Event-driven messaging and pub/sub patterns
- Multiple external interfaces and endpoints

#### API Security Controls

**Authentication and Authorization:**
```python
# OAuth 2.0 implementation example
from flask import Flask, request, jsonify
from functools import wraps
import jwt
import requests

app = Flask(__name__)

def require_auth(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        token = request.headers.get('Authorization')
        if not token:
            return jsonify({'error': 'No token provided'}), 401
        
        try:
            # Remove 'Bearer ' prefix
            token = token.replace('Bearer ', '')
            
            # Verify JWT token
            payload = jwt.decode(token, 'your-secret-key', algorithms=['HS256'])
            
            # Check scopes/permissions
            required_scope = 'research:read'
            if required_scope not in payload.get('scopes', []):
                return jsonify({'error': 'Insufficient permissions'}), 403
                
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token expired'}), 401
        except jwt.InvalidTokenError:
            return jsonify({'error': 'Invalid token'}), 401
        
        return f(*args, **kwargs)
    return decorated_function

@app.route('/api/research-data')
@require_auth
def get_research_data():
    return jsonify({'data': 'sensitive research information'})

# Rate limiting implementation
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["1000 per hour"]
)

@app.route('/api/compute-intensive')
@limiter.limit("10 per minute")
def compute_intensive_endpoint():
    # Expensive computation here
    return jsonify({'result': 'computation complete'})
```

### Afternoon Session (2 hours)

#### API Gateway Security

**AWS API Gateway Security Configuration:**
```bash
# Create API Gateway with security
aws apigateway create-rest-api \
  --name research-data-api \
  --description "Secure API for research data access"

# Create authorizer
aws apigateway create-authorizer \
  --rest-api-id abcdef123 \
  --name jwt-authorizer \
  --type TOKEN \
  --authorizer-uri arn:aws:lambda:us-east-1:123456789012:function:jwt-authorizer \
  --authorizer-credentials arn:aws:iam::123456789012:role/APIGatewayAuthorizerRole

# Create usage plan with throttling
aws apigateway create-usage-plan \
  --name research-usage-plan \
  --throttle burstLimit=100,rateLimit=50 \
  --quota limit=10000,period=DAY

# Create API key
aws apigateway create-api-key \
  --name research-client-key \
  --description "API key for research client applications"
```

**Input Validation and Sanitization:**
```python
from marshmallow import Schema, fields, validate, ValidationError

class ResearchDataSchema(Schema):
    experiment_id = fields.Str(required=True, validate=validate.Length(min=1, max=50))
    temperature = fields.Float(required=True, validate=validate.Range(min=-273.15, max=1000))
    pressure = fields.Float(required=True, validate=validate.Range(min=0))
    timestamp = fields.DateTime(required=True)
    researcher_id = fields.Str(required=True, validate=validate.Regexp(r'^[a-zA-Z0-9_]+$'))

@app.route('/api/submit-data', methods=['POST'])
@require_auth
def submit_research_data():
    schema = ResearchDataSchema()
    try:
        # Validate input data
        data = schema.load(request.json)
        
        # Process validated data
        result = process_research_data(data)
        
        return jsonify({'status': 'success', 'id': result['id']})
        
    except ValidationError as err:
        return jsonify({'error': 'Invalid input', 'details': err.messages}), 400
```

---

## Day 2: Microservices Security Architecture

### Morning Session (3 hours)

#### Microservices vs. Monolithic Security

**HPC Monolithic Applications:**
- Single large application with all functionality
- Internal function calls and shared memory
- Centralized security controls
- Single point of failure and attack surface
- Uniform security policies

**Microservices Architecture:**
- Multiple small, independent services
- Network-based communication between services
- Distributed security controls
- Multiple attack surfaces to secure
- Service-specific security policies

#### Service Mesh Security

**Istio Service Mesh Configuration:**
```yaml
# Enable mTLS for all services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: research
spec:
  mtls:
    mode: STRICT

---
# Authorization policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: research-data-access
  namespace: research
spec:
  selector:
    matchLabels:
      app: research-data-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/research/sa/research-frontend"]
  - to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/data/*"]
```

### Afternoon Session (2 hours)

#### Zero Trust Networking

**Network Policies for Microservices:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: research-data-service-policy
spec:
  podSelector:
    matchLabels:
      app: research-data-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: research-frontend
    - podSelector:
        matchLabels:
          app: research-api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: research-database
    ports:
    - protocol: TCP
      port: 5432
  - to: []  # Allow DNS
    ports:
    - protocol: UDP
      port: 53
```

**Service-to-Service Authentication:**
```python
import requests
import jwt
from datetime import datetime, timedelta

class ServiceAuthenticator:
    def __init__(self, service_name, private_key):
        self.service_name = service_name
        self.private_key = private_key
    
    def generate_service_token(self, target_service):
        """Generate JWT token for service-to-service communication"""
        payload = {
            'iss': self.service_name,
            'aud': target_service,
            'exp': datetime.utcnow() + timedelta(minutes=5),
            'iat': datetime.utcnow(),
            'sub': self.service_name
        }
        
        token = jwt.encode(payload, self.private_key, algorithm='RS256')
        return token
    
    def call_service(self, service_url, endpoint, data=None):
        """Make authenticated service-to-service call"""
        token = self.generate_service_token('research-data-service')
        
        headers = {
            'Authorization': f'Bearer {token}',
            'Content-Type': 'application/json'
        }
        
        response = requests.post(f"{service_url}{endpoint}", 
                               json=data, headers=headers)
        return response

# Usage in microservice
auth = ServiceAuthenticator('research-compute-service', private_key)
response = auth.call_service('http://research-data-service:8080', 
                           '/api/store-results', 
                           {'experiment_id': '12345', 'results': results})
```

---

## Day 3: Application Security Testing

### Morning Session (3 hours)

#### SAST (Static Application Security Testing)

**SonarQube Integration:**
```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on: [push, pull_request]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run SonarQube Scan
      uses: sonarqube-quality-gate-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    
    - name: Run Bandit Security Scan
      run: |
        pip install bandit
        bandit -r src/ -f json -o bandit-report.json
    
    - name: Run Safety Check
      run: |
        pip install safety
        safety check --json --output safety-report.json
    
    - name: Upload Security Reports
      uses: actions/upload-artifact@v3
      with:
        name: security-reports
        path: |
          bandit-report.json
          safety-report.json
```

#### DAST (Dynamic Application Security Testing)

**OWASP ZAP Integration:**
```bash
#!/bin/bash
# Dynamic security testing script

# Start application
docker-compose up -d research-api

# Wait for application to be ready
sleep 30

# Run OWASP ZAP scan
docker run -v $(pwd):/zap/wrk/:rw \
  -t owasp/zap2docker-stable zap-baseline.py \
  -t http://host.docker.internal:8080/api \
  -g gen.conf \
  -J zap-report.json \
  -r zap-report.html

# Run Nikto scan
docker run --rm sullo/nikto \
  -h http://host.docker.internal:8080 \
  -output nikto-report.txt

# Cleanup
docker-compose down
```

### Afternoon Session (2 hours)

#### Dependency Scanning

**Automated Dependency Scanning:**
```yaml
# dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "security-team"
    assignees:
      - "dev-team"
    open-pull-requests-limit: 10
    
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Container Image Scanning:**
```bash
# Trivy container scanning
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy image research/api:latest

# Snyk container scanning
snyk container test research/api:latest \
  --severity-threshold=high \
  --json > container-scan-results.json

# Grype vulnerability scanning
grype research/api:latest -o json > grype-results.json
```

---

## Day 4: Runtime Protection

### Morning Session (3 hours)

#### Web Application Firewall (WAF)

**AWS WAF Configuration:**
```bash
# Create WAF Web ACL
aws wafv2 create-web-acl \
  --name research-api-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules file://waf-rules.json

# WAF rules configuration
cat > waf-rules.json << EOF
[
  {
    "Name": "SQLInjectionRule",
    "Priority": 1,
    "Statement": {
      "SqliMatchStatement": {
        "FieldToMatch": {
          "AllQueryArguments": {}
        },
        "TextTransformations": [
          {
            "Priority": 0,
            "Type": "URL_DECODE"
          }
        ]
      }
    },
    "Action": {
      "Block": {}
    }
  },
  {
    "Name": "XSSRule",
    "Priority": 2,
    "Statement": {
      "XssMatchStatement": {
        "FieldToMatch": {
          "Body": {}
        },
        "TextTransformations": [
          {
            "Priority": 0,
            "Type": "HTML_ENTITY_DECODE"
          }
        ]
      }
    },
    "Action": {
      "Block": {}
    }
  }
]
EOF
```

#### DDoS Protection

**CloudFlare DDoS Protection:**
```python
# Rate limiting with Redis
import redis
import time
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def rate_limit(max_requests=100, window=3600):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            client_ip = request.remote_addr
            key = f"rate_limit:{client_ip}"
            
            current_requests = redis_client.get(key)
            if current_requests is None:
                redis_client.setex(key, window, 1)
            else:
                current_requests = int(current_requests)
                if current_requests >= max_requests:
                    return jsonify({'error': 'Rate limit exceeded'}), 429
                redis_client.incr(key)
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/api/compute-intensive')
@rate_limit(max_requests=10, window=60)  # 10 requests per minute
def compute_intensive_endpoint():
    return jsonify({'result': 'computation complete'})
```

### Afternoon Session (2 hours)

#### Runtime Application Self-Protection (RASP)

**Application Security Monitoring:**
```python
import logging
import json
from datetime import datetime

class SecurityMonitor:
    def __init__(self):
        self.logger = logging.getLogger('security')
        handler = logging.StreamHandler()
        formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_security_event(self, event_type, details, severity='INFO'):
        event = {
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': event_type,
            'details': details,
            'severity': severity,
            'source_ip': request.remote_addr if request else 'unknown',
            'user_agent': request.headers.get('User-Agent') if request else 'unknown'
        }
        
        if severity == 'CRITICAL':
            self.logger.critical(json.dumps(event))
        elif severity == 'WARNING':
            self.logger.warning(json.dumps(event))
        else:
            self.logger.info(json.dumps(event))
    
    def detect_anomaly(self, user_id, action, resource):
        # Simple anomaly detection
        key = f"user_activity:{user_id}:{action}"
        recent_activity = redis_client.get(key)
        
        if recent_activity and int(recent_activity) > 50:  # Threshold
            self.log_security_event(
                'ANOMALOUS_ACTIVITY',
                {
                    'user_id': user_id,
                    'action': action,
                    'resource': resource,
                    'activity_count': int(recent_activity)
                },
                'WARNING'
            )
            return True
        
        redis_client.incr(key)
        redis_client.expire(key, 3600)  # 1 hour window
        return False

# Usage in application
security_monitor = SecurityMonitor()

@app.before_request
def security_check():
    # Log all API requests
    security_monitor.log_security_event(
        'API_REQUEST',
        {
            'method': request.method,
            'path': request.path,
            'args': dict(request.args)
        }
    )
    
    # Check for anomalous activity
    if hasattr(g, 'user_id'):
        if security_monitor.detect_anomaly(g.user_id, request.method, request.path):
            return jsonify({'error': 'Suspicious activity detected'}), 429
```

---

## Day 5: Container and Orchestration Security

### Morning Session (3 hours)

#### Container Runtime Security

**Falco Runtime Security:**
```yaml
# Falco rules for container security
- rule: Unexpected Network Connection
  desc: Detect unexpected network connections from containers
  condition: >
    spawned_process and container and
    (proc.name=nc or proc.name=ncat or proc.name=netcat) and
    not proc.pname in (sshd, systemd)
  output: >
    Unexpected network connection (user=%user.name command=%proc.cmdline 
    container=%container.name image=%container.image.repository)
  priority: WARNING

- rule: Sensitive File Access
  desc: Detect access to sensitive files
  condition: >
    open_read and container and
    (fd.name startswith /etc/passwd or 
     fd.name startswith /etc/shadow or
     fd.name startswith /etc/ssh/)
  output: >
    Sensitive file accessed (user=%user.name file=%fd.name 
    container=%container.name image=%container.image.repository)
  priority: WARNING
```

#### Kubernetes Security Policies

**Pod Security Standards:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: research-secure
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: v1
kind: Pod
metadata:
  name: secure-research-pod
  namespace: research-secure
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: research-app
    image: research/secure-app:v1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
      runAsNonRoot: true
      runAsUser: 1001
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      requests:
        memory: "2Gi"
        cpu: "1"
```

### Afternoon Session (2 hours)

#### Security Scanning and Compliance

**Automated Security Scanning:**
```bash
#!/bin/bash
# Comprehensive security scanning script

echo "Starting security scan..."

# Container image scanning
echo "Scanning container images..."
trivy image --format json --output image-scan.json research/api:latest

# Kubernetes configuration scanning
echo "Scanning Kubernetes configurations..."
kubesec scan k8s-manifests/*.yaml > kubesec-results.json

# Network policy validation
echo "Validating network policies..."
kubectl auth can-i --list --as=system:serviceaccount:research:default

# RBAC analysis
echo "Analyzing RBAC permissions..."
kubectl auth can-i --list --as=system:serviceaccount:research:research-app

# Compliance checking
echo "Running compliance checks..."
kube-bench run --targets node,policies,managedservices > compliance-report.txt

echo "Security scan completed. Check reports for details."
```

**Continuous Compliance Monitoring:**
```python
#!/usr/bin/env python3
import kubernetes
from kubernetes import client, config
import json

def check_pod_security_compliance():
    """Check pods for security compliance"""
    config.load_incluster_config()  # or load_kube_config() for local
    v1 = client.CoreV1Api()
    
    compliance_issues = []
    
    pods = v1.list_pod_for_all_namespaces()
    for pod in pods.items:
        issues = []
        
        # Check if running as root
        if pod.spec.security_context:
            if not pod.spec.security_context.run_as_non_root:
                issues.append("Pod may be running as root")
        
        # Check container security contexts
        for container in pod.spec.containers:
            if container.security_context:
                if container.security_context.privileged:
                    issues.append(f"Container {container.name} is privileged")
                
                if not container.security_context.read_only_root_filesystem:
                    issues.append(f"Container {container.name} has writable root filesystem")
        
        if issues:
            compliance_issues.append({
                'pod_name': pod.metadata.name,
                'namespace': pod.metadata.namespace,
                'issues': issues
            })
    
    return compliance_issues

if __name__ == "__main__":
    issues = check_pod_security_compliance()
    print(json.dumps(issues, indent=2))
```

---

## Week 7 Deliverable: Cloud-Native Security Architecture

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Cloud-native security approach vs. monolithic application security
- Key security challenges and solutions for microservices
- Implementation timeline and resource requirements
- Integration with existing security infrastructure

### API Security Implementation (3-4 pages)

**Authentication and Authorization:**
- OAuth 2.0 and JWT implementation strategy
- API gateway security configuration
- Rate limiting and throttling policies
- Input validation and sanitization procedures

**API Security Testing:**
- SAST/DAST integration in CI/CD pipelines
- Dependency scanning and vulnerability management
- Penetration testing procedures
- Security regression testing

### Microservices Security Architecture (3-4 pages)

**Service Mesh Security:**
- mTLS implementation for service-to-service communication
- Authorization policies and access controls
- Traffic encryption and network segmentation
- Service identity and certificate management

**Zero Trust Implementation:**
- Network policies and micro-segmentation
- Service-to-service authentication
- Continuous verification and monitoring
- Least privilege access controls

### Runtime Protection (2-3 pages)

**Web Application Firewall:**
- WAF rules and configuration
- DDoS protection and rate limiting
- Bot detection and mitigation
- Custom security rules for research applications

**Runtime Monitoring:**
- Application security monitoring
- Anomaly detection and behavioral analysis
- Security event logging and alerting
- Incident response automation

### Container and Orchestration Security (2-3 pages)

**Container Security:**
- Image scanning and vulnerability management
- Runtime security monitoring
- Container isolation and sandboxing
- Secrets management and configuration

**Kubernetes Security:**
- Pod security standards implementation
- RBAC and access control policies
- Network policies and segmentation
- Compliance monitoring and reporting

### Implementation Roadmap (1-2 pages)
- Phased implementation approach
- Security testing and validation procedures
- Training and awareness programs
- Continuous improvement processes

---

## Key Questions Answered This Week

By the end of week 7, you should be able to answer:

**About API Security:**
- [ ] How do you implement secure authentication and authorization for APIs?
- [ ] What are the key API security threats and how do you mitigate them?
- [ ] How do you implement rate limiting and input validation?
- [ ] What security testing should be performed on APIs?

**About Microservices Security:**
- [ ] How do you secure service-to-service communication?
- [ ] What is a service mesh and how does it improve security?
- [ ] How do you implement zero trust in microservices architectures?
- [ ] What are the security challenges of distributed applications?

**About Runtime Protection:**
- [ ] How do you implement WAF and DDoS protection?
- [ ] What runtime security monitoring should be implemented?
- [ ] How do you detect and respond to application security threats?
- [ ] What container and orchestration security controls are needed?

---

## Success Metrics for Week 7

You should be able to:
- [ ] Design and implement secure API architectures
- [ ] Configure service mesh security for microservices
- [ ] Implement comprehensive application security testing
- [ ] Deploy runtime protection and monitoring
- [ ] Secure container and orchestration platforms

---

## Week 7 Wrap-Up

By the end of week 7, you should understand how cloud-native application security requires a fundamentally different approach than traditional monolithic applications. The distributed nature of microservices creates new attack surfaces but also enables more granular security controls and better isolation.

The key insight: Cloud-native security is about securing the communication between services, implementing defense in depth at multiple layers, and using automation to manage security at scale.

**Next week**: We'll explore DevOps and CI/CD security, including how to build security into automated deployment pipelines and implement Infrastructure as Code securely.