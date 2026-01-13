# Week 05: Storage Tiering & Data Lifecycle Management

## Learning Objectives
- Implement automated storage tiering across HPC and cloud environments
- Design data lifecycle policies for cost optimization
- Configure intelligent data archival and retrieval systems
- Optimize storage performance through strategic data placement
- Implement compliance and retention policies for hybrid storage

## Overview
Effective storage tiering and lifecycle management are crucial for balancing performance, cost, and compliance in hybrid HPC-cloud environments. This week focuses on automated policies and intelligent data placement strategies.

## Storage Tier Architecture

### Tier Classification
1. **Hot Tier**: Frequently accessed data (NVMe, SSD)
2. **Warm Tier**: Occasionally accessed data (HDD, Standard cloud storage)
3. **Cold Tier**: Rarely accessed data (Tape, Glacier, Archive storage)
4. **Frozen Tier**: Compliance/backup data (Deep Archive, Tape libraries)

### Performance Characteristics
| Tier | Access Time | Throughput | Cost/GB | Use Cases |
|------|-------------|------------|---------|-----------|
| Hot | < 1ms | > 1 GB/s | High | Active datasets, scratch space |
| Warm | < 100ms | 100-500 MB/s | Medium | Reference data, recent results |
| Cold | Minutes | 10-50 MB/s | Low | Historical data, backups |
| Frozen | Hours | < 10 MB/s | Very Low | Long-term archive, compliance |

## Hands-On Lab: Automated Tiering Implementation

### Exercise 1: Policy-Based Data Movement (120 minutes)

#### Storage Policy Engine
```python
import os
import time
import boto3
from datetime import datetime, timedelta

class StorageTieringEngine:
    def __init__(self, config):
        self.config = config
        self.s3_client = boto3.client('s3')
        
    def analyze_file_access(self, file_path):
        """Analyze file access patterns"""
        stat = os.stat(file_path)
        
        return {
            'size': stat.st_size,
            'last_access': stat.st_atime,
            'last_modified': stat.st_mtime,
            'access_frequency': self.get_access_frequency(file_path)
        }
    
    def determine_optimal_tier(self, file_info):
        """Determine optimal storage tier based on access patterns"""
        days_since_access = (time.time() - file_info['last_access']) / 86400
        file_size = file_info['size']
        access_freq = file_info['access_frequency']
        
        if days_since_access < 7 and access_freq > 10:
            return 'hot'
        elif days_since_access < 30 and file_size < 1e9:
            return 'warm'
        elif days_since_access < 90:
            return 'cold'
        else:
            return 'frozen'
    
    def execute_tiering_policy(self, source_path):
        """Execute tiering policy for a directory"""
        for root, dirs, files in os.walk(source_path):
            for file in files:
                file_path = os.path.join(root, file)
                file_info = self.analyze_file_access(file_path)
                optimal_tier = self.determine_optimal_tier(file_info)
                
                self.move_to_tier(file_path, optimal_tier)
    
    def move_to_tier(self, file_path, tier):
        """Move file to appropriate storage tier"""
        tier_config = self.config['tiers'][tier]
        
        if tier == 'hot':
            # Keep on high-performance local storage
            pass
        elif tier == 'warm':
            # Move to standard cloud storage
            self.upload_to_s3(file_path, 'STANDARD')
        elif tier == 'cold':
            # Move to infrequent access storage
            self.upload_to_s3(file_path, 'STANDARD_IA')
        elif tier == 'frozen':
            # Move to archive storage
            self.upload_to_s3(file_path, 'GLACIER')
```

#### Configuration Management
```yaml
# storage-tiering-config.yaml
tiers:
  hot:
    storage_class: "local_nvme"
    max_age_days: 7
    min_access_frequency: 10
    
  warm:
    storage_class: "STANDARD"
    max_age_days: 30
    cost_per_gb: 0.023
    
  cold:
    storage_class: "STANDARD_IA"
    max_age_days: 90
    cost_per_gb: 0.0125
    
  frozen:
    storage_class: "GLACIER"
    cost_per_gb: 0.004
    retrieval_time_hours: 12

policies:
  research_data:
    hot_retention_days: 14
    warm_retention_days: 90
    auto_archive: true
    
  simulation_results:
    hot_retention_days: 7
    warm_retention_days: 30
    compression_enabled: true
```

### Exercise 2: Automated Lifecycle Management (90 minutes)

#### Lifecycle Policy Implementation
```bash
# AWS S3 Lifecycle Policy
aws s3api put-bucket-lifecycle-configuration \
    --bucket hpc-research-data \
    --lifecycle-configuration file://lifecycle-policy.json
```

```json
{
  "Rules": [
    {
      "ID": "HPCDataLifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "research-data/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555  // 7 years for compliance
      }
    }
  ]
}
```

#### Intelligent Retrieval System
```python
class DataRetrievalManager:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.retrieval_queue = []
        
    def request_data_retrieval(self, s3_key, priority='Standard'):
        """Request data retrieval from archive storage"""
        try:
            # Check current storage class
            response = self.s3_client.head_object(
                Bucket='hpc-research-data',
                Key=s3_key
            )
            
            storage_class = response.get('StorageClass', 'STANDARD')
            
            if storage_class in ['GLACIER', 'DEEP_ARCHIVE']:
                # Initiate restore request
                self.s3_client.restore_object(
                    Bucket='hpc-research-data',
                    Key=s3_key,
                    RestoreRequest={
                        'Days': 7,  # Keep restored for 7 days
                        'GlacierJobParameters': {
                            'Tier': priority  # Standard, Expedited, Bulk
                        }
                    }
                )
                
                return f"Restore initiated for {s3_key}"
            else:
                return f"Data {s3_key} is already available"
                
        except Exception as e:
            return f"Error retrieving {s3_key}: {str(e)}"
    
    def monitor_retrieval_status(self, s3_key):
        """Monitor the status of data retrieval"""
        response = self.s3_client.head_object(
            Bucket='hpc-research-data',
            Key=s3_key
        )
        
        restore_status = response.get('Restore')
        if restore_status:
            if 'ongoing-request="true"' in restore_status:
                return "Retrieval in progress"
            else:
                return "Data available for download"
        else:
            return "No active retrieval request"
```

### Exercise 3: Cost Optimization Dashboard (60 minutes)

#### Storage Cost Analytics
```python
import matplotlib.pyplot as plt
import pandas as pd
from datetime import datetime, timedelta

class StorageCostAnalyzer:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        
    def get_storage_metrics(self, bucket_name, days=30):
        """Retrieve storage metrics from CloudWatch"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=days)
        
        metrics = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/S3',
            MetricName='BucketSizeBytes',
            Dimensions=[
                {'Name': 'BucketName', 'Value': bucket_name},
                {'Name': 'StorageType', 'Value': 'StandardStorage'}
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=86400,  # Daily
            Statistics=['Average']
        )
        
        return metrics['Datapoints']
    
    def calculate_tier_costs(self, storage_data):
        """Calculate costs for different storage tiers"""
        costs = {
            'STANDARD': storage_data['standard_gb'] * 0.023,
            'STANDARD_IA': storage_data['ia_gb'] * 0.0125,
            'GLACIER': storage_data['glacier_gb'] * 0.004,
            'DEEP_ARCHIVE': storage_data['deep_archive_gb'] * 0.00099
        }
        
        return costs
    
    def generate_cost_report(self):
        """Generate comprehensive cost analysis report"""
        report = {
            'total_storage_gb': 0,
            'monthly_cost': 0,
            'tier_breakdown': {},
            'optimization_recommendations': []
        }
        
        # Add analysis logic here
        return report
```

## Compliance and Governance

### Data Retention Policies
```python
class ComplianceManager:
    def __init__(self, retention_policies):
        self.policies = retention_policies
        
    def apply_retention_policy(self, data_classification):
        """Apply retention policy based on data classification"""
        policy = self.policies.get(data_classification)
        
        if not policy:
            raise ValueError(f"No policy found for {data_classification}")
        
        return {
            'retention_years': policy['retention_years'],
            'archive_after_days': policy['archive_after_days'],
            'deletion_allowed': policy['deletion_allowed'],
            'encryption_required': policy['encryption_required']
        }
    
    def audit_compliance(self, storage_inventory):
        """Audit storage for compliance violations"""
        violations = []
        
        for item in storage_inventory:
            policy = self.apply_retention_policy(item['classification'])
            
            # Check retention period
            if item['age_days'] > (policy['retention_years'] * 365):
                if policy['deletion_allowed']:
                    violations.append({
                        'item': item['path'],
                        'violation': 'Exceeds retention period',
                        'action': 'Schedule for deletion'
                    })
        
        return violations
```

### Audit Trail Implementation
```bash
# CloudTrail configuration for S3 data events
aws cloudtrail create-trail \
    --name hpc-storage-audit \
    --s3-bucket-name hpc-audit-logs \
    --include-global-service-events \
    --is-multi-region-trail \
    --enable-log-file-validation

# Configure data events for storage operations
aws cloudtrail put-event-selectors \
    --trail-name hpc-storage-audit \
    --event-selectors '[{
        "ReadWriteType": "All",
        "IncludeManagementEvents": true,
        "DataResources": [{
            "Type": "AWS::S3::Object",
            "Values": ["arn:aws:s3:::hpc-research-data/*"]
        }]
    }]'
```

## Performance Monitoring

### Storage Performance Metrics
```python
class StoragePerformanceMonitor:
    def __init__(self):
        self.metrics = {}
        
    def measure_tier_performance(self, tier_name, test_file_size=1024*1024*100):
        """Measure performance characteristics of storage tier"""
        import time
        import tempfile
        import os
        
        # Write performance test
        start_time = time.time()
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_file.write(b'0' * test_file_size)
            temp_file.flush()
            os.fsync(temp_file.fileno())
        write_time = time.time() - start_time
        
        # Read performance test
        start_time = time.time()
        with open(temp_file.name, 'rb') as f:
            data = f.read()
        read_time = time.time() - start_time
        
        # Cleanup
        os.unlink(temp_file.name)
        
        return {
            'tier': tier_name,
            'write_throughput_mbps': (test_file_size / (1024*1024)) / write_time,
            'read_throughput_mbps': (test_file_size / (1024*1024)) / read_time,
            'write_latency_ms': write_time * 1000,
            'read_latency_ms': read_time * 1000
        }
```

## Week 5 Deliverables

### Tiering Strategy Document
1. **Storage Architecture Design** (2-3 pages)
   - Tier definitions and characteristics
   - Data flow and movement policies
   - Performance requirements mapping

2. **Lifecycle Management Implementation** (2 pages)
   - Automated policy configuration
   - Monitoring and alerting setup
   - Cost optimization analysis

3. **Compliance Framework** (1-2 pages)
   - Retention policy documentation
   - Audit trail configuration
   - Risk assessment and mitigation

### Practical Implementation
- Configure automated tiering for lab environment
- Implement lifecycle policies with cost tracking
- Create performance monitoring dashboard
- Document troubleshooting procedures

---

*Intelligent storage tiering can reduce costs by 60-80% while maintaining performance. Invest in automation to make it sustainable.*