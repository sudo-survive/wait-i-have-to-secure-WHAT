# Week 06: Data Security & Compliance in Hybrid Environments

## Learning Objectives
- Implement end-to-end encryption for data in hybrid environments
- Design compliance frameworks spanning HPC and cloud systems
- Configure data loss prevention (DLP) and monitoring systems
- Implement zero-trust data security architectures
- Manage regulatory compliance across multiple jurisdictions

## Overview
Data security in hybrid HPC-cloud environments requires comprehensive protection strategies that work across different platforms, protocols, and regulatory frameworks. This week covers encryption, compliance, and monitoring best practices.

## Security Architecture Patterns

### Defense in Depth Strategy
1. **Data Classification**: Categorize data by sensitivity and regulatory requirements
2. **Encryption Layers**: At-rest, in-transit, and in-use protection
3. **Access Controls**: Identity-based and attribute-based access control
4. **Monitoring**: Continuous security monitoring and threat detection
5. **Compliance**: Automated compliance checking and reporting

### Zero-Trust Data Security
- **Never Trust, Always Verify**: Authenticate and authorize every data access
- **Least Privilege**: Minimal necessary access rights
- **Micro-Segmentation**: Isolate data based on classification
- **Continuous Monitoring**: Real-time security posture assessment

## Hands-On Lab: Comprehensive Data Security Implementation

### Exercise 1: Data Classification and Encryption (120 minutes)

#### Automated Data Classification
```python
import re
import hashlib
import boto3
from cryptography.fernet import Fernet

class DataClassifier:
    def __init__(self):
        self.patterns = {
            'PII': [
                r'\b\d{3}-\d{2}-\d{4}\b',  # SSN
                r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',  # Email
                r'\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b'  # Credit Card
            ],
            'PHI': [
                r'\b(patient|medical|diagnosis|treatment)\b',
                r'\b\d{2}/\d{2}/\d{4}\b.*\b(birth|dob)\b'
            ],
            'RESEARCH': [
                r'\b(experiment|trial|study|research)\b',
                r'\.(csv|xlsx|dat|hdf5)\b'
            ]
        }
    
    def classify_file(self, file_path):
        """Classify file based on content analysis"""
        classifications = []
        
        try:
            with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                content = f.read()
                
            for classification, patterns in self.patterns.items():
                for pattern in patterns:
                    if re.search(pattern, content, re.IGNORECASE):
                        classifications.append(classification)
                        break
                        
        except Exception as e:
            print(f"Error classifying {file_path}: {e}")
            
        return classifications if classifications else ['UNCLASSIFIED']
    
    def generate_encryption_key(self, classification):
        """Generate appropriate encryption key based on classification"""
        if classification in ['PII', 'PHI']:
            # Use stronger encryption for sensitive data
            return Fernet.generate_key()
        else:
            # Standard encryption for other data
            return Fernet.generate_key()

class HybridEncryptionManager:
    def __init__(self):
        self.kms_client = boto3.client('kms')
        self.local_keys = {}
        
    def encrypt_file(self, file_path, classification, destination='local'):
        """Encrypt file based on classification and destination"""
        if destination == 'cloud':
            return self.encrypt_for_cloud(file_path, classification)
        else:
            return self.encrypt_locally(file_path, classification)
    
    def encrypt_for_cloud(self, file_path, classification):
        """Encrypt file using cloud KMS"""
        # Generate data encryption key
        response = self.kms_client.generate_data_key(
            KeyId='arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012',
            KeySpec='AES_256'
        )
        
        plaintext_key = response['Plaintext']
        encrypted_key = response['CiphertextBlob']
        
        # Encrypt file with data key
        fernet = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
        
        with open(file_path, 'rb') as f:
            encrypted_data = fernet.encrypt(f.read())
        
        # Store encrypted file with encrypted key
        encrypted_file_path = f"{file_path}.encrypted"
        with open(encrypted_file_path, 'wb') as f:
            f.write(encrypted_key + b'|' + encrypted_data)
        
        return encrypted_file_path
```

#### Key Management System
```python
class HybridKeyManager:
    def __init__(self):
        self.aws_kms = boto3.client('kms')
        self.local_keystore = '/secure/keystore/'
        
    def create_master_key(self, environment, classification):
        """Create master key for specific environment and classification"""
        if environment == 'cloud':
            response = self.aws_kms.create_key(
                Description=f'Master key for {classification} data',
                Usage='ENCRYPT_DECRYPT',
                KeySpec='SYMMETRIC_DEFAULT'
            )
            return response['KeyMetadata']['KeyId']
        else:
            # Local key generation for HPC environment
            key = Fernet.generate_key()
            key_file = f"{self.local_keystore}/{classification}_master.key"
            
            with open(key_file, 'wb') as f:
                f.write(key)
            
            # Set restrictive permissions
            os.chmod(key_file, 0o600)
            return key_file
    
    def rotate_keys(self, key_id, environment):
        """Implement key rotation policy"""
        if environment == 'cloud':
            self.aws_kms.enable_key_rotation(KeyId=key_id)
        else:
            # Implement local key rotation
            self.backup_old_key(key_id)
            new_key = self.create_master_key('local', 'rotated')
            self.update_key_references(key_id, new_key)
```

### Exercise 2: Compliance Framework Implementation (90 minutes)

#### GDPR Compliance Module
```python
class GDPRComplianceManager:
    def __init__(self):
        self.data_inventory = {}
        self.consent_records = {}
        
    def register_personal_data(self, data_id, data_type, location, purpose):
        """Register personal data for GDPR compliance"""
        self.data_inventory[data_id] = {
            'type': data_type,
            'location': location,
            'purpose': purpose,
            'created_date': datetime.now(),
            'retention_period': self.get_retention_period(data_type),
            'lawful_basis': self.determine_lawful_basis(purpose)
        }
    
    def process_data_subject_request(self, request_type, subject_id):
        """Handle GDPR data subject requests"""
        if request_type == 'access':
            return self.generate_data_export(subject_id)
        elif request_type == 'deletion':
            return self.delete_personal_data(subject_id)
        elif request_type == 'portability':
            return self.export_portable_data(subject_id)
        elif request_type == 'rectification':
            return self.update_personal_data(subject_id)
    
    def audit_data_processing(self):
        """Generate GDPR compliance audit report"""
        report = {
            'total_records': len(self.data_inventory),
            'retention_violations': [],
            'consent_issues': [],
            'security_measures': self.assess_security_measures()
        }
        
        for data_id, data_info in self.data_inventory.items():
            # Check retention period compliance
            age = (datetime.now() - data_info['created_date']).days
            if age > data_info['retention_period']:
                report['retention_violations'].append(data_id)
        
        return report
```

#### HIPAA Compliance Framework
```python
class HIPAAComplianceManager:
    def __init__(self):
        self.phi_locations = {}
        self.access_logs = []
        
    def classify_phi(self, data_path):
        """Identify and classify Protected Health Information"""
        phi_indicators = [
            'patient_id', 'medical_record', 'diagnosis',
            'treatment', 'prescription', 'lab_result'
        ]
        
        # Scan file for PHI indicators
        with open(data_path, 'r') as f:
            content = f.read().lower()
            
        phi_found = [indicator for indicator in phi_indicators 
                    if indicator in content]
        
        if phi_found:
            self.phi_locations[data_path] = {
                'phi_types': phi_found,
                'risk_level': self.assess_phi_risk(phi_found),
                'encryption_required': True,
                'access_restrictions': 'minimum_necessary'
            }
    
    def implement_minimum_necessary(self, user_role, requested_data):
        """Implement HIPAA minimum necessary standard"""
        role_permissions = {
            'researcher': ['aggregated_data', 'de_identified_data'],
            'clinician': ['patient_data', 'treatment_data'],
            'administrator': ['audit_logs', 'system_data']
        }
        
        allowed_data = role_permissions.get(user_role, [])
        filtered_data = [item for item in requested_data 
                        if any(allowed in item for allowed in allowed_data)]
        
        return filtered_data
```

### Exercise 3: Continuous Security Monitoring (75 minutes)

#### Security Information and Event Management (SIEM)
```python
import json
from datetime import datetime
import elasticsearch

class HybridSIEM:
    def __init__(self):
        self.es_client = elasticsearch.Elasticsearch(['localhost:9200'])
        self.alert_rules = self.load_alert_rules()
        
    def ingest_security_event(self, event_data):
        """Ingest security event from HPC or cloud environment"""
        event = {
            'timestamp': datetime.now().isoformat(),
            'source': event_data.get('source', 'unknown'),
            'event_type': event_data.get('type', 'unknown'),
            'severity': event_data.get('severity', 'info'),
            'details': event_data.get('details', {}),
            'user': event_data.get('user', 'system'),
            'ip_address': event_data.get('ip', '0.0.0.0')
        }
        
        # Index event in Elasticsearch
        self.es_client.index(
            index=f"security-events-{datetime.now().strftime('%Y-%m')}",
            body=event
        )
        
        # Check for alert conditions
        self.evaluate_alerts(event)
    
    def evaluate_alerts(self, event):
        """Evaluate event against alert rules"""
        for rule in self.alert_rules:
            if self.matches_rule(event, rule):
                self.trigger_alert(event, rule)
    
    def detect_anomalies(self, time_window='1h'):
        """Detect anomalous patterns in security events"""
        query = {
            "query": {
                "range": {
                    "timestamp": {
                        "gte": f"now-{time_window}"
                    }
                }
            },
            "aggs": {
                "users": {
                    "terms": {"field": "user.keyword"},
                    "aggs": {
                        "event_types": {
                            "terms": {"field": "event_type.keyword"}
                        }
                    }
                }
            }
        }
        
        response = self.es_client.search(
            index="security-events-*",
            body=query
        )
        
        return self.analyze_patterns(response['aggregations'])
```

#### Data Loss Prevention (DLP)
```python
class HybridDLP:
    def __init__(self):
        self.sensitive_patterns = {
            'ssn': r'\b\d{3}-\d{2}-\d{4}\b',
            'credit_card': r'\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b',
            'api_key': r'[A-Za-z0-9]{32,}',
            'private_key': r'-----BEGIN PRIVATE KEY-----'
        }
        
    def scan_data_transfer(self, data_path, destination):
        """Scan data transfer for sensitive information"""
        violations = []
        
        with open(data_path, 'r', encoding='utf-8', errors='ignore') as f:
            content = f.read()
        
        for pattern_name, pattern in self.sensitive_patterns.items():
            matches = re.findall(pattern, content)
            if matches:
                violations.append({
                    'type': pattern_name,
                    'count': len(matches),
                    'destination': destination,
                    'action': self.get_policy_action(pattern_name, destination)
                })
        
        return violations
    
    def enforce_policy(self, violations):
        """Enforce DLP policy based on violations"""
        for violation in violations:
            if violation['action'] == 'block':
                raise SecurityError(f"Transfer blocked: {violation['type']} detected")
            elif violation['action'] == 'encrypt':
                self.encrypt_sensitive_data(violation)
            elif violation['action'] == 'alert':
                self.send_security_alert(violation)
```

## Regulatory Compliance Frameworks

### Multi-Jurisdiction Compliance
```python
class MultiJurisdictionCompliance:
    def __init__(self):
        self.regulations = {
            'US': ['HIPAA', 'SOX', 'FERPA'],
            'EU': ['GDPR', 'Medical Device Regulation'],
            'GLOBAL': ['ISO 27001', 'SOC 2']
        }
        
    def assess_compliance_requirements(self, data_location, data_type):
        """Assess compliance requirements based on data location and type"""
        requirements = []
        
        # Determine applicable jurisdictions
        jurisdictions = self.get_applicable_jurisdictions(data_location)
        
        for jurisdiction in jurisdictions:
            for regulation in self.regulations[jurisdiction]:
                if self.applies_to_data_type(regulation, data_type):
                    requirements.append({
                        'regulation': regulation,
                        'jurisdiction': jurisdiction,
                        'requirements': self.get_regulation_requirements(regulation)
                    })
        
        return requirements
    
    def generate_compliance_report(self, assessment_period='quarterly'):
        """Generate comprehensive compliance report"""
        report = {
            'period': assessment_period,
            'compliance_status': {},
            'violations': [],
            'remediation_actions': [],
            'risk_assessment': {}
        }
        
        # Assess each regulation
        for jurisdiction, regulations in self.regulations.items():
            for regulation in regulations:
                status = self.assess_regulation_compliance(regulation)
                report['compliance_status'][regulation] = status
        
        return report
```

## Week 6 Deliverables

### Security Architecture Document
1. **Data Security Framework** (3-4 pages)
   - Classification scheme and encryption strategy
   - Access control implementation
   - Monitoring and incident response procedures

2. **Compliance Implementation Plan** (2-3 pages)
   - Regulatory requirement mapping
   - Compliance monitoring setup
   - Audit trail configuration

3. **Security Assessment Report** (2 pages)
   - Vulnerability assessment results
   - Risk analysis and mitigation strategies
   - Security metrics and KPIs

### Practical Implementation
- Configure end-to-end encryption for lab environment
- Implement DLP policies and monitoring
- Create compliance dashboard and reporting
- Conduct security assessment and penetration testing

---

*Security and compliance are not optional in hybrid environments. Build them in from the beginning, not as an afterthought.*