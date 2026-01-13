# Week 4: Cloud Storage and Data Management
## From Parallel Filesystems to Object Storage and Data Lakes

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how cloud storage services differ from parallel filesystems
- Design and implement cloud storage architectures for scientific data
- Configure data lifecycle management and archival strategies
- Implement data security, encryption, and access controls in cloud storage
- Design data lakes and analytics architectures for research data
- Plan migration strategies from HPC storage to cloud storage

---

## Day 1: Cloud Storage Models vs. Parallel Filesystems

### Morning Session (3 hours)

#### HPC Storage vs. Cloud Storage Paradigms

**Traditional HPC Storage (what you know):**
- Parallel filesystems (GPFS, Lustre, BeeGFS)
- POSIX-compliant file operations
- High-performance parallel I/O
- Shared namespace across all compute nodes
- Hierarchical directory structure
- Direct-attached storage with high-speed interconnects

**Cloud Storage Paradigm:**
- Multiple storage types for different use cases
- API-driven access (REST, not POSIX)
- Managed services with built-in redundancy
- Global accessibility and scalability
- Flat namespace with metadata
- Pay-per-use pricing model

#### Cloud Storage Types Deep Dive

**Object Storage (S3, Azure Blob, Google Cloud Storage):**

| Aspect | HPC Parallel Filesystem | Cloud Object Storage |
|--------|------------------------|---------------------|
| **Access Method** | POSIX file operations | REST API calls |
| **Namespace** | Hierarchical directories | Flat with key-value pairs |
| **Scalability** | Limited by hardware | Virtually unlimited |
| **Consistency** | Strong consistency | Eventually consistent (some services) |
| **Performance** | Optimized for parallel I/O | Optimized for throughput |
| **Cost Model** | Fixed infrastructure cost | Pay per GB stored and accessed |

**Object Storage Characteristics:**
- **Buckets/Containers**: Top-level storage containers
- **Objects**: Individual files with metadata
- **Keys**: Unique identifiers for objects
- **Metadata**: Custom attributes and system properties
- **Versioning**: Multiple versions of the same object
- **Lifecycle policies**: Automated data management

**Block Storage (EBS, Azure Disks, Persistent Disks):**
- **HPC equivalent**: Like dedicated disks attached to compute nodes
- **Use cases**: Database storage, file systems, high-IOPS applications
- **Performance**: Consistent, predictable performance
- **Durability**: Built-in replication and snapshots

**File Storage (EFS, Azure Files, Filestore):**
- **HPC equivalent**: Most similar to traditional NFS shares
- **Use cases**: Shared application data, content repositories
- **Access**: POSIX-compliant, mountable on multiple instances
- **Performance**: Good for concurrent access, but not HPC-level

### Afternoon Session (2 hours)

#### Hands-On Object Storage Exploration

**Basic Object Storage Operations (AWS S3 example):**
```bash
# Create bucket (like creating a filesystem)
aws s3 mb s3://research-data-bucket-unique-name

# Upload files (like copying to /scratch)
aws s3 cp dataset.tar.gz s3://research-data-bucket-unique-name/
aws s3 cp results/ s3://research-data-bucket-unique-name/results/ --recursive

# List objects (like ls command)
aws s3 ls s3://research-data-bucket-unique-name/
aws s3 ls s3://research-data-bucket-unique-name/results/ --recursive

# Download files (like copying from /scratch)
aws s3 cp s3://research-data-bucket-unique-name/dataset.tar.gz ./
aws s3 sync s3://research-data-bucket-unique-name/results/ ./local-results/

# Set metadata (additional information about objects)
aws s3api put-object \
  --bucket research-data-bucket-unique-name \
  --key experiment-1/data.csv \
  --body data.csv \
  --metadata experiment=climate-model,date=2024-01-15,researcher=jsmith
```

**Object Storage vs. Filesystem Commands:**
```bash
# HPC filesystem operations (what you know)
ls -la /scratch/project/
cp /home/user/data.txt /scratch/project/
chmod 644 /scratch/project/data.txt
find /scratch/project/ -name "*.csv" -mtime -7

# Cloud object storage equivalents
aws s3 ls s3://bucket/project/
aws s3 cp data.txt s3://bucket/project/
aws s3api put-object-acl --bucket bucket --key project/data.txt --acl public-read
aws s3api list-objects-v2 --bucket bucket --prefix project/ --query 'Contents[?LastModified>=`2024-01-08`]'
```

#### Storage Performance Characteristics

**Performance Comparison:**

| Storage Type | Throughput | IOPS | Latency | Use Case |
|-------------|------------|------|---------|----------|
| HPC Parallel FS | 100+ GB/s | 1M+ | Microseconds | Parallel computing |
| Object Storage | 10-100 GB/s | N/A | Milliseconds | Bulk data, archives |
| Block Storage | 1-10 GB/s | 100K+ | Milliseconds | Databases, file systems |
| File Storage | 1-10 GB/s | 10K+ | Milliseconds | Shared file access |

**When to Use Each:**
- **Object Storage**: Data archival, backup, content distribution, data lakes
- **Block Storage**: Database storage, file systems, high-performance applications
- **File Storage**: Shared application data, content management, legacy applications

---

## Day 2: Data Security and Access Controls

### Morning Session (3 hours)

#### Cloud Storage Security Models

**HPC Storage Security (what you know):**
- Unix permissions (rwx for owner/group/other)
- ACLs for more granular control
- Physical security of storage hardware
- Network-based access controls
- Encryption often manual and optional

**Cloud Storage Security:**
- Identity-based access control (IAM)
- Resource-based policies (bucket policies)
- Encryption by default
- Audit logging for all access
- Network-based access controls (VPC endpoints)

#### Object Storage Access Control

**IAM Policies for Storage:**
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
      "Resource": "arn:aws:s3:::research-data-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

**Bucket Policies (Resource-Based):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowResearchTeamAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ResearchTeamRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::research-data-bucket/team-data/*"
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::research-data-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

#### Encryption in Cloud Storage

**Encryption Options:**
- **Server-side encryption**: Data encrypted by cloud provider
- **Client-side encryption**: Data encrypted before upload
- **Key management**: Cloud-managed vs. customer-managed keys
- **Encryption in transit**: HTTPS/TLS for all transfers

**Encryption Configuration:**
```bash
# Enable default encryption on bucket
aws s3api put-bucket-encryption \
  --bucket research-data-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }'

# Upload with specific encryption
aws s3 cp data.txt s3://research-data-bucket/ \
  --server-side-encryption AES256

# Upload with customer-managed key
aws s3 cp data.txt s3://research-data-bucket/ \
  --server-side-encryption aws:kms \
  --ssekms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
```

### Afternoon Session (2 hours)

#### Hands-On Security Implementation

**Implement Data Classification:**
```bash
# Create buckets for different data classifications
aws s3 mb s3://research-public-data
aws s3 mb s3://research-internal-data
aws s3 mb s3://research-restricted-data

# Set public access block (security best practice)
aws s3api put-public-access-block \
  --bucket research-internal-data \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Configure bucket logging
aws s3api put-bucket-logging \
  --bucket research-internal-data \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "research-access-logs",
      "TargetPrefix": "internal-data-access/"
    }
  }'
```

**Access Control Testing:**
```python
#!/usr/bin/env python3
import boto3
import json

def test_bucket_access():
    """Test bucket access controls"""
    s3 = boto3.client('s3')
    bucket_name = 'research-internal-data'
    
    try:
        # Test read access
        response = s3.list_objects_v2(Bucket=bucket_name, MaxKeys=1)
        print("✓ Read access granted")
    except Exception as e:
        print(f"✗ Read access denied: {e}")
    
    try:
        # Test write access
        s3.put_object(Bucket=bucket_name, Key='test-file.txt', Body=b'test data')
        print("✓ Write access granted")
        
        # Clean up test file
        s3.delete_object(Bucket=bucket_name, Key='test-file.txt')
    except Exception as e:
        print(f"✗ Write access denied: {e}")

def audit_bucket_permissions():
    """Audit bucket permissions and policies"""
    s3 = boto3.client('s3')
    
    # List all buckets
    buckets = s3.list_buckets()['Buckets']
    
    for bucket in buckets:
        bucket_name = bucket['Name']
        print(f"\nAuditing bucket: {bucket_name}")
        
        try:
            # Check bucket policy
            policy = s3.get_bucket_policy(Bucket=bucket_name)
            print("  Has bucket policy")
        except s3.exceptions.NoSuchBucketPolicy:
            print("  No bucket policy")
        
        try:
            # Check public access block
            pab = s3.get_public_access_block(Bucket=bucket_name)
            print("  Public access block configured")
        except Exception:
            print("  No public access block")
        
        try:
            # Check encryption
            encryption = s3.get_bucket_encryption(Bucket=bucket_name)
            print("  Encryption enabled")
        except Exception:
            print("  No default encryption")

if __name__ == "__main__":
    test_bucket_access()
    audit_bucket_permissions()
```

#### Data Loss Prevention

**Versioning and Backup:**
```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket research-data-bucket \
  --versioning-configuration Status=Enabled

# Enable MFA delete (additional protection)
aws s3api put-bucket-versioning \
  --bucket research-data-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/user 123456"

# Cross-region replication for disaster recovery
aws s3api put-bucket-replication \
  --bucket research-data-bucket \
  --replication-configuration file://replication-config.json
```

---

## Day 3: Data Lifecycle Management and Archival

### Morning Session (3 hours)

#### Cloud Storage Classes and Tiers

**Storage Classes Comparison:**

| Storage Class | HPC Equivalent | Use Case | Cost | Retrieval Time |
|---------------|----------------|----------|------|----------------|
| Standard | /scratch (active) | Frequently accessed | Highest | Immediate |
| Infrequent Access | /projects (occasional) | Monthly access | Medium | Immediate |
| Archive | Tape storage | Long-term retention | Low | Hours |
| Deep Archive | Offline tape | Compliance/backup | Lowest | 12+ hours |

**Lifecycle Policies:**
- **Automatic transitions**: Move data between storage classes
- **Expiration**: Delete data after specified time
- **Incomplete uploads**: Clean up failed uploads
- **Versioning**: Manage object versions

#### Lifecycle Policy Configuration

**Example Lifecycle Policy:**
```json
{
  "Rules": [
    {
      "ID": "ResearchDataLifecycle",
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
        "Days": 2555
      }
    },
    {
      "ID": "TempDataCleanup",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "temp/"
      },
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

**Implement Lifecycle Management:**
```bash
# Apply lifecycle policy
aws s3api put-bucket-lifecycle-configuration \
  --bucket research-data-bucket \
  --lifecycle-configuration file://lifecycle-policy.json

# Monitor lifecycle transitions
aws s3api get-bucket-lifecycle-configuration \
  --bucket research-data-bucket

# Check storage class of objects
aws s3api list-objects-v2 \
  --bucket research-data-bucket \
  --query 'Contents[*].[Key,StorageClass,LastModified]' \
  --output table
```

### Afternoon Session (2 hours)

#### Data Archival and Retrieval

**Archive Retrieval Options:**
- **Expedited**: 1-5 minutes (premium cost)
- **Standard**: 3-5 hours (standard cost)
- **Bulk**: 5-12 hours (lowest cost)

**Archive Operations:**
```bash
# Restore archived object
aws s3api restore-object \
  --bucket research-data-bucket \
  --key archived-dataset.tar.gz \
  --restore-request '{
    "Days": 7,
    "GlacierJobParameters": {
      "Tier": "Standard"
    }
  }'

# Check restore status
aws s3api head-object \
  --bucket research-data-bucket \
  --key archived-dataset.tar.gz

# Bulk restore multiple objects
aws s3api list-objects-v2 \
  --bucket research-data-bucket \
  --prefix "2023/experiments/" \
  --query 'Contents[?StorageClass==`GLACIER`].Key' \
  --output text | \
while read key; do
  aws s3api restore-object \
    --bucket research-data-bucket \
    --key "$key" \
    --restore-request '{"Days": 7, "GlacierJobParameters": {"Tier": "Bulk"}}'
done
```

#### Cost Optimization Strategies

**Storage Cost Analysis:**
```python
#!/usr/bin/env python3
import boto3
from datetime import datetime, timedelta

def analyze_storage_costs():
    """Analyze storage usage and costs"""
    s3 = boto3.client('s3')
    cloudwatch = boto3.client('cloudwatch')
    
    # Get storage metrics
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=30)
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/S3',
        MetricName='BucketSizeBytes',
        Dimensions=[
            {'Name': 'BucketName', 'Value': 'research-data-bucket'},
            {'Name': 'StorageType', 'Value': 'StandardStorage'}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=86400,  # Daily
        Statistics=['Average']
    )
    
    for datapoint in response['Datapoints']:
        size_gb = datapoint['Average'] / (1024**3)
        print(f"Date: {datapoint['Timestamp'].date()}, Size: {size_gb:.2f} GB")

def recommend_lifecycle_policies():
    """Recommend lifecycle policies based on access patterns"""
    s3 = boto3.client('s3')
    
    # Analyze object access patterns
    bucket_name = 'research-data-bucket'
    objects = s3.list_objects_v2(Bucket=bucket_name)
    
    for obj in objects.get('Contents', []):
        age_days = (datetime.now(obj['LastModified'].tzinfo) - obj['LastModified']).days
        size_mb = obj['Size'] / (1024**2)
        
        if age_days > 90 and size_mb > 100:
            print(f"Recommend archiving: {obj['Key']} (age: {age_days} days, size: {size_mb:.1f} MB)")
        elif age_days > 30:
            print(f"Consider IA storage: {obj['Key']} (age: {age_days} days)")

if __name__ == "__main__":
    analyze_storage_costs()
    recommend_lifecycle_policies()
```

**Cost Optimization Best Practices:**
- Use appropriate storage classes for access patterns
- Implement lifecycle policies for automatic transitions
- Monitor and analyze storage usage regularly
- Clean up incomplete multipart uploads
- Use data compression when appropriate
- Consider data deduplication strategies

---

## Day 4: Data Lakes and Analytics Architecture

### Morning Session (3 hours)

#### Data Lake Concepts

**Traditional HPC Data Management:**
- Structured data in specific formats
- Application-specific data processing
- Limited data sharing and discovery
- Manual data movement and transformation
- Siloed data repositories

**Cloud Data Lake Architecture:**
- Store raw data in native formats
- Schema-on-read approach
- Centralized data repository
- Automated data processing pipelines
- Integrated analytics and ML services

#### Data Lake Design Patterns

**Zone-Based Architecture:**
```
Data Lake Structure:
├── raw/                    # Ingested data in original format
│   ├── experiments/
│   ├── simulations/
│   └── observations/
├── processed/              # Cleaned and transformed data
│   ├── standardized/
│   ├── aggregated/
│   └── enriched/
├── curated/               # Business-ready datasets
│   ├── research-ready/
│   ├── publication/
│   └── collaboration/
└── archive/               # Long-term retention
    ├── compliance/
    └── historical/
```

**Data Lake Implementation:**
```bash
# Create data lake bucket structure
aws s3 mb s3://research-data-lake
aws s3api put-object --bucket research-data-lake --key raw/
aws s3api put-object --bucket research-data-lake --key processed/
aws s3api put-object --bucket research-data-lake --key curated/
aws s3api put-object --bucket research-data-lake --key archive/

# Set up data lake permissions
aws s3api put-bucket-policy \
  --bucket research-data-lake \
  --policy file://data-lake-policy.json

# Configure data lake catalog
aws glue create-database \
  --database-input '{
    "Name": "research_data_catalog",
    "Description": "Catalog for research data lake"
  }'
```

### Afternoon Session (2 hours)

#### Data Processing and Analytics

**Serverless Analytics:**
```bash
# Query data with Amazon Athena (serverless SQL)
aws athena start-query-execution \
  --query-string "SELECT * FROM research_data_catalog.experiments WHERE date >= '2024-01-01'" \
  --result-configuration OutputLocation=s3://query-results-bucket/

# Create external table for data lake
aws athena start-query-execution \
  --query-string "
    CREATE EXTERNAL TABLE research_data_catalog.climate_data (
      timestamp string,
      temperature double,
      humidity double,
      pressure double
    )
    STORED AS PARQUET
    LOCATION 's3://research-data-lake/curated/climate/'
  " \
  --result-configuration OutputLocation=s3://query-results-bucket/
```

**Data Processing Pipeline:**
```python
#!/usr/bin/env python3
import boto3
import pandas as pd
from io import StringIO

def process_research_data():
    """Process raw research data and store in curated zone"""
    s3 = boto3.client('s3')
    
    # Read raw data
    response = s3.get_object(Bucket='research-data-lake', Key='raw/experiment-001.csv')
    raw_data = pd.read_csv(StringIO(response['Body'].read().decode('utf-8')))
    
    # Process data
    processed_data = raw_data.dropna()  # Remove missing values
    processed_data['timestamp'] = pd.to_datetime(processed_data['timestamp'])
    processed_data = processed_data[processed_data['temperature'] > -50]  # Quality filter
    
    # Save processed data
    csv_buffer = StringIO()
    processed_data.to_csv(csv_buffer, index=False)
    
    s3.put_object(
        Bucket='research-data-lake',
        Key='processed/experiment-001-cleaned.csv',
        Body=csv_buffer.getvalue(),
        Metadata={
            'processing-date': '2024-01-15',
            'quality-checks': 'temperature-range,missing-values',
            'source': 'raw/experiment-001.csv'
        }
    )
    
    print("Data processing completed")

def create_data_catalog():
    """Create data catalog entries for processed data"""
    glue = boto3.client('glue')
    
    # Create table in data catalog
    glue.create_table(
        DatabaseName='research_data_catalog',
        TableInput={
            'Name': 'processed_experiments',
            'StorageDescriptor': {
                'Columns': [
                    {'Name': 'timestamp', 'Type': 'timestamp'},
                    {'Name': 'temperature', 'Type': 'double'},
                    {'Name': 'humidity', 'Type': 'double'},
                    {'Name': 'pressure', 'Type': 'double'}
                ],
                'Location': 's3://research-data-lake/processed/',
                'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat',
                'OutputFormat': 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
                'SerdeInfo': {
                    'SerializationLibrary': 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe',
                    'Parameters': {'field.delim': ','}
                }
            }
        }
    )

if __name__ == "__main__":
    process_research_data()
    create_data_catalog()
```

#### Machine Learning Integration

**ML Pipeline Setup:**
```bash
# Create SageMaker notebook for data analysis
aws sagemaker create-notebook-instance \
  --notebook-instance-name research-analysis \
  --instance-type ml.t3.medium \
  --role-arn arn:aws:iam::123456789012:role/SageMakerExecutionRole

# Set up automated ML pipeline
aws sagemaker create-pipeline \
  --pipeline-name research-data-pipeline \
  --pipeline-definition file://ml-pipeline-definition.json
```

---

## Day 5: Migration Strategies and Best Practices

### Morning Session (3 hours)

#### HPC to Cloud Storage Migration

**Migration Planning:**
1. **Data Assessment**: Inventory current HPC storage
2. **Classification**: Categorize data by access patterns
3. **Architecture Design**: Design target cloud storage architecture
4. **Migration Strategy**: Plan phased migration approach
5. **Validation**: Verify data integrity and accessibility

**Data Assessment Script:**
```python
#!/usr/bin/env python3
import os
import time
from pathlib import Path

def assess_hpc_storage():
    """Assess HPC storage for cloud migration planning"""
    storage_paths = ['/scratch', '/projects', '/home']
    
    for path in storage_paths:
        if os.path.exists(path):
            print(f"\nAnalyzing {path}:")
            analyze_directory(path)

def analyze_directory(path):
    """Analyze directory for migration planning"""
    total_size = 0
    file_count = 0
    access_patterns = {'recent': 0, 'old': 0, 'ancient': 0}
    
    current_time = time.time()
    
    for root, dirs, files in os.walk(path):
        for file in files:
            file_path = os.path.join(root, file)
            try:
                stat = os.stat(file_path)
                total_size += stat.st_size
                file_count += 1
                
                # Analyze access patterns
                days_since_access = (current_time - stat.st_atime) / 86400
                if days_since_access < 30:
                    access_patterns['recent'] += 1
                elif days_since_access < 365:
                    access_patterns['old'] += 1
                else:
                    access_patterns['ancient'] += 1
                    
            except (OSError, IOError):
                continue  # Skip files we can't access
    
    print(f"  Total size: {total_size / (1024**3):.2f} GB")
    print(f"  File count: {file_count}")
    print(f"  Access patterns:")
    print(f"    Recent (< 30 days): {access_patterns['recent']}")
    print(f"    Old (30-365 days): {access_patterns['old']}")
    print(f"    Ancient (> 365 days): {access_patterns['ancient']}")

if __name__ == "__main__":
    assess_hpc_storage()
```

#### Migration Execution

**Data Transfer Tools:**
```bash
# AWS DataSync for large-scale migration
aws datasync create-location-nfs \
  --server-hostname hpc-storage.example.com \
  --subdirectory /scratch/research-data \
  --on-prem-config AgentArns=arn:aws:datasync:us-east-1:123456789012:agent/agent-12345678

aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::research-data-migration \
  --s3-config BucketAccessRoleArn=arn:aws:iam::123456789012:role/DataSyncS3Role

aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-source \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-dest

# Alternative: AWS CLI sync for smaller datasets
aws s3 sync /scratch/research-data/ s3://research-data-migration/scratch/ \
  --storage-class STANDARD_IA \
  --exclude "*.tmp" \
  --exclude "core.*"

# Parallel transfer with GNU parallel
find /scratch/research-data -name "*.tar.gz" | \
parallel -j 8 aws s3 cp {} s3://research-data-migration/archives/
```

### Afternoon Session (2 hours)

#### Post-Migration Validation

**Data Integrity Verification:**
```python
#!/usr/bin/env python3
import boto3
import hashlib
import os

def verify_migration():
    """Verify data integrity after migration"""
    s3 = boto3.client('s3')
    bucket_name = 'research-data-migration'
    
    # List objects in bucket
    paginator = s3.get_paginator('list_objects_v2')
    
    for page in paginator.paginate(Bucket=bucket_name):
        for obj in page.get('Contents', []):
            key = obj['Key']
            
            # Get object metadata
            response = s3.head_object(Bucket=bucket_name, Key=key)
            cloud_etag = response['ETag'].strip('"')
            
            # Compare with local file if it exists
            local_path = f"/scratch/{key}"
            if os.path.exists(local_path):
                local_hash = calculate_file_hash(local_path)
                if local_hash != cloud_etag:
                    print(f"MISMATCH: {key}")
                else:
                    print(f"VERIFIED: {key}")

def calculate_file_hash(file_path):
    """Calculate MD5 hash of local file"""
    hash_md5 = hashlib.md5()
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            hash_md5.update(chunk)
    return hash_md5.hexdigest()

if __name__ == "__main__":
    verify_migration()
```

#### Best Practices and Lessons Learned

**Migration Best Practices:**
- **Start small**: Begin with non-critical data
- **Parallel transfers**: Use multiple transfer streams
- **Verify integrity**: Always validate transferred data
- **Monitor costs**: Track transfer and storage costs
- **Plan for rollback**: Have contingency plans
- **User communication**: Keep users informed of progress
- **Performance testing**: Validate application performance

**Common Challenges and Solutions:**
- **Large file transfers**: Use multipart uploads and resume capability
- **Network bandwidth**: Schedule transfers during off-peak hours
- **Cost management**: Use appropriate storage classes and lifecycle policies
- **Application compatibility**: Test applications with cloud storage APIs
- **Performance differences**: Optimize applications for cloud storage characteristics

---

## Week 4 Deliverable: Cloud Storage Architecture and Migration Plan

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Cloud storage approach vs. traditional HPC storage
- Key benefits and challenges of cloud storage adoption
- Migration timeline and resource requirements
- Cost analysis and optimization strategies

### Storage Architecture Design (4-5 pages)

**Storage Service Selection:**
- Object storage for different use cases
- Block storage for high-performance applications
- File storage for shared access requirements
- Archive storage for long-term retention

**Data Lake Architecture:**
- Zone-based data organization
- Data processing and analytics pipelines
- Metadata management and cataloging
- Integration with ML and analytics services

**Security and Access Control:**
- Data classification and handling
- Encryption and key management
- Access control policies and procedures
- Audit logging and monitoring

### Migration Strategy (4-5 pages)

**Current State Assessment:**
- HPC storage inventory and analysis
- Data classification and access patterns
- Application dependencies and requirements
- Performance and capacity requirements

**Migration Planning:**
- Phased migration approach
- Data transfer methods and tools
- Validation and testing procedures
- Rollback and contingency plans

**Post-Migration Optimization:**
- Performance tuning and optimization
- Cost optimization strategies
- Lifecycle management implementation
- Continuous monitoring and improvement

### Implementation Roadmap (2-3 pages)

**Phase 1: Foundation (0-3 months):**
- Basic cloud storage setup
- Initial data migration (non-critical)
- Security and access control implementation
- Basic monitoring and logging

**Phase 2: Scale (3-6 months):**
- Large-scale data migration
- Data lake implementation
- Advanced analytics capabilities
- Performance optimization

**Phase 3: Optimize (6-12 months):**
- Full lifecycle management
- Advanced security features
- Cost optimization
- Integration with organizational systems

### Operational Procedures (1-2 pages)
- Data management procedures
- Backup and recovery processes
- Security monitoring and response
- Cost monitoring and optimization

---

## Key Security Questions Answered This Week

By the end of week 4, you should be able to answer:

**About Cloud Storage:**
- [ ] How do cloud storage services differ from parallel filesystems?
- [ ] What are the appropriate use cases for different storage types?
- [ ] How should data be organized and managed in cloud storage?
- [ ] What are the performance and cost characteristics of cloud storage?

**About Security and Compliance:**
- [ ] How should data security be implemented in cloud storage?
- [ ] What encryption and access control options are available?
- [ ] How should data lifecycle management be implemented?
- [ ] What monitoring and auditing capabilities are needed?

**About Migration:**
- [ ] How should HPC storage be migrated to cloud storage?
- [ ] What are the key challenges and solutions for large-scale migration?
- [ ] How should data integrity be verified during migration?
- [ ] What post-migration optimization strategies should be implemented?

---

## Resources for Week 4

### Technical Documentation
- AWS S3 User Guide and best practices
- Azure Blob Storage documentation
- Google Cloud Storage documentation
- Data lake architecture patterns and guides

### Migration Tools
- AWS DataSync and Storage Gateway
- Azure Data Box and AzCopy
- Google Cloud Transfer Service
- Third-party migration tools and services

### Analytics and ML Services
- AWS Glue, Athena, and SageMaker
- Azure Data Factory and Machine Learning
- Google Cloud Dataflow and AI Platform
- Open source analytics frameworks

---

## Success Metrics for Week 4

You should be able to:
- [ ] Design cloud storage architectures for scientific data
- [ ] Implement data security and lifecycle management
- [ ] Plan and execute storage migration from HPC to cloud
- [ ] Set up data lakes and analytics pipelines
- [ ] Optimize storage costs and performance

---

## Common Week 4 Challenges and Solutions

**"Object storage APIs are very different from POSIX file operations"**
- Start with simple operations and build complexity gradually
- Use cloud provider SDKs and tools to simplify API interactions
- Consider file gateway solutions for POSIX compatibility
- Practice with small datasets before large-scale operations

**"Cloud storage performance is different from parallel filesystems"**
- Understand cloud storage performance characteristics
- Optimize applications for cloud storage patterns
- Use appropriate storage types for different workloads
- Implement caching and buffering strategies

**"Data migration seems too complex and risky"**
- Start with non-critical data for initial migration
- Use proven migration tools and services
- Implement comprehensive validation procedures
- Plan for rollback scenarios and contingencies

**"Storage costs in cloud can be unpredictable"**
- Implement lifecycle policies for automatic cost optimization
- Monitor storage usage and costs regularly
- Use appropriate storage classes for access patterns
- Consider data compression and deduplication strategies

---

## Week 4 Wrap-Up

By the end of week 4, you should understand how cloud storage differs from traditional HPC storage and how to implement secure, cost-effective storage architectures in the cloud. Cloud storage provides much more flexibility and scalability than traditional storage, but requires different design patterns and management approaches.

The key insight: Cloud storage is about API-driven, service-oriented data management rather than filesystem-based operations. It provides unlimited scalability and built-in durability, but requires understanding different access patterns and cost models.

**Next week**: We'll explore cloud compute services and containerization, including how to run scientific workloads on cloud compute services and how containers differ from traditional HPC job execution.