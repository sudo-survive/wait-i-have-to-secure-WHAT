# Week 9: Databases and Data Services
## From File-Based Storage to Managed Database Services

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how managed databases differ from file-based HPC data storage
- Design and implement secure database architectures for scientific data
- Configure and manage relational and NoSQL databases in the cloud
- Implement data warehousing and analytics solutions
- Apply database security best practices including encryption and access controls
- Design data migration strategies from HPC file systems to cloud databases

---

## Day 1: Relational Databases in the Cloud

### Morning Session (3 hours)

#### HPC Data Storage vs. Cloud Databases

**HPC Data Storage (what you know):**
- Files stored in parallel filesystems (GPFS, Lustre)
- CSV, HDF5, NetCDF formats for scientific data
- Custom file formats and binary data
- Direct file I/O from applications
- Manual data organization and indexing

**Cloud Database Services:**
- Managed database services (RDS, Aurora)
- Structured data with schemas and relationships
- SQL queries for data access and analysis
- Automatic backups, scaling, and maintenance
- Built-in security and compliance features

#### Amazon RDS Security Configuration

**Secure RDS Setup:**
```bash
# Create secure RDS instance
aws rds create-db-instance \
  --db-instance-identifier research-database \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username research_admin \
  --master-user-password $(aws secretsmanager get-random-password --password-length 32 --exclude-characters '"@/\' --query 'RandomPassword' --output text) \
  --allocated-storage 100 \
  --storage-type gp2 \
  --storage-encrypted \
  --kms-key-id alias/rds-encryption-key \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name research-db-subnet-group \
  --backup-retention-period 7 \
  --multi-az \
  --deletion-protection \
  --enable-performance-insights \
  --performance-insights-retention-period 7
```

**Database Security Groups:**
```bash
# Create database security group
aws ec2 create-security-group \
  --group-name research-database-sg \
  --description "Security group for research database"

# Allow access only from application servers
aws ec2 authorize-security-group-ingress \
  --group-id sg-database \
  --protocol tcp \
  --port 5432 \
  --source-group sg-application-servers
```

### Afternoon Session (2 hours)

#### Database Schema Design for Scientific Data

**Scientific Data Schema Example:**
```sql
-- Create research database schema
CREATE SCHEMA research_data;

-- Experiments table
CREATE TABLE research_data.experiments (
    experiment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    researcher_id VARCHAR(100) NOT NULL,
    start_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    end_date TIMESTAMP WITH TIME ZONE,
    status VARCHAR(50) DEFAULT 'active',
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Measurements table
CREATE TABLE research_data.measurements (
    measurement_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID REFERENCES research_data.experiments(experiment_id),
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    temperature DECIMAL(10,4),
    pressure DECIMAL(10,4),
    humidity DECIMAL(8,4),
    location POINT,
    quality_flag INTEGER DEFAULT 1,
    raw_data JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create indexes for performance
CREATE INDEX idx_measurements_experiment_timestamp 
ON research_data.measurements(experiment_id, timestamp);

CREATE INDEX idx_measurements_location 
ON research_data.measurements USING GIST(location);

-- Row Level Security
ALTER TABLE research_data.experiments ENABLE ROW LEVEL SECURITY;

CREATE POLICY experiment_access_policy ON research_data.experiments
    FOR ALL TO research_users
    USING (researcher_id = current_setting('app.current_user_id'));
```

---

## Day 2: NoSQL Databases

### Morning Session (3 hours)

#### NoSQL for Scientific Data

**DynamoDB for Scientific Applications:**
```python
import boto3
from decimal import Decimal
import json

dynamodb = boto3.resource('dynamodb')

# Create table for sensor data
table = dynamodb.create_table(
    TableName='SensorData',
    KeySchema=[
        {
            'AttributeName': 'sensor_id',
            'KeyType': 'HASH'  # Partition key
        },
        {
            'AttributeName': 'timestamp',
            'KeyType': 'RANGE'  # Sort key
        }
    ],
    AttributeDefinitions=[
        {
            'AttributeName': 'sensor_id',
            'AttributeType': 'S'
        },
        {
            'AttributeName': 'timestamp',
            'AttributeType': 'N'
        }
    ],
    BillingMode='PAY_PER_REQUEST',
    SSESpecification={
        'Enabled': True,
        'SSEType': 'KMS',
        'KMSMasterKeyId': 'alias/dynamodb-encryption-key'
    }
)

# Insert scientific data
def store_sensor_reading(sensor_id, timestamp, temperature, pressure, metadata):
    table.put_item(
        Item={
            'sensor_id': sensor_id,
            'timestamp': Decimal(str(timestamp)),
            'temperature': Decimal(str(temperature)),
            'pressure': Decimal(str(pressure)),
            'metadata': metadata,
            'ttl': int(timestamp) + 31536000  # 1 year TTL
        }
    )

# Query data
def get_sensor_data(sensor_id, start_time, end_time):
    response = table.query(
        KeyConditionExpression=Key('sensor_id').eq(sensor_id) & 
                             Key('timestamp').between(start_time, end_time)
    )
    return response['Items']
```

### Afternoon Session (2 hours)

#### Document Databases for Research

**MongoDB for Scientific Documents:**
```python
from pymongo import MongoClient
from datetime import datetime

# Connect to MongoDB
client = MongoClient('mongodb://username:password@cluster.mongodb.net/')
db = client.research_database

# Store research paper metadata
papers_collection = db.papers

paper_document = {
    "title": "Climate Change Impact on Arctic Ice",
    "authors": ["Dr. Smith", "Dr. Johnson"],
    "abstract": "This study examines...",
    "keywords": ["climate", "arctic", "ice", "temperature"],
    "publication_date": datetime(2024, 1, 15),
    "journal": "Nature Climate Change",
    "doi": "10.1038/s41558-024-01234-5",
    "data_files": [
        {
            "filename": "temperature_data.csv",
            "size": 1024000,
            "checksum": "sha256:abc123...",
            "location": "s3://research-data/climate/temperature_data.csv"
        }
    ],
    "methodology": {
        "data_collection": "Satellite observations",
        "analysis_tools": ["Python", "R", "MATLAB"],
        "statistical_methods": ["regression", "time_series"]
    }
}

# Insert document
result = papers_collection.insert_one(paper_document)

# Create text index for search
papers_collection.create_index([
    ("title", "text"),
    ("abstract", "text"),
    ("keywords", "text")
])

# Search papers
search_results = papers_collection.find(
    {"$text": {"$search": "climate arctic"}},
    {"score": {"$meta": "textScore"}}
).sort([("score", {"$meta": "textScore"})])
```

---

## Day 3: Data Warehousing and Analytics

### Morning Session (3 hours)

#### Scientific Data Warehousing

**Amazon Redshift for Research Analytics:**
```sql
-- Create research data warehouse schema
CREATE SCHEMA research_warehouse;

-- Fact table for experimental measurements
CREATE TABLE research_warehouse.fact_measurements (
    measurement_key BIGINT IDENTITY(1,1) PRIMARY KEY,
    experiment_key INTEGER NOT NULL,
    time_key INTEGER NOT NULL,
    location_key INTEGER NOT NULL,
    temperature DECIMAL(10,4),
    pressure DECIMAL(10,4),
    humidity DECIMAL(8,4),
    measurement_count INTEGER DEFAULT 1
)
DISTKEY(experiment_key)
SORTKEY(time_key, experiment_key);

-- Dimension tables
CREATE TABLE research_warehouse.dim_experiments (
    experiment_key INTEGER IDENTITY(1,1) PRIMARY KEY,
    experiment_id VARCHAR(100) NOT NULL,
    experiment_name VARCHAR(255),
    researcher_name VARCHAR(255),
    department VARCHAR(100),
    start_date DATE,
    end_date DATE
)
DISTSTYLE ALL;

CREATE TABLE research_warehouse.dim_time (
    time_key INTEGER PRIMARY KEY,
    full_date DATE,
    year INTEGER,
    month INTEGER,
    day INTEGER,
    hour INTEGER,
    minute INTEGER,
    day_of_week INTEGER,
    month_name VARCHAR(20),
    quarter INTEGER
)
DISTSTYLE ALL;

-- Analytical queries
-- Average temperature by month and experiment
SELECT 
    e.experiment_name,
    t.month_name,
    t.year,
    AVG(m.temperature) as avg_temperature,
    COUNT(*) as measurement_count
FROM research_warehouse.fact_measurements m
JOIN research_warehouse.dim_experiments e ON m.experiment_key = e.experiment_key
JOIN research_warehouse.dim_time t ON m.time_key = t.time_key
WHERE t.year = 2024
GROUP BY e.experiment_name, t.month_name, t.year
ORDER BY t.year, t.month_name;
```

### Afternoon Session (2 hours)

#### Real-time Analytics

**Amazon Kinesis for Streaming Data:**
```python
import boto3
import json
from datetime import datetime

kinesis = boto3.client('kinesis')

def send_sensor_data_to_stream(sensor_id, temperature, pressure, humidity):
    """Send real-time sensor data to Kinesis stream"""
    
    data = {
        'sensor_id': sensor_id,
        'timestamp': datetime.utcnow().isoformat(),
        'temperature': temperature,
        'pressure': pressure,
        'humidity': humidity,
        'location': {
            'latitude': 40.7128,
            'longitude': -74.0060
        }
    }
    
    response = kinesis.put_record(
        StreamName='research-sensor-stream',
        Data=json.dumps(data),
        PartitionKey=sensor_id
    )
    
    return response

# Lambda function for processing streaming data
def lambda_handler(event, context):
    """Process streaming sensor data"""
    
    for record in event['Records']:
        # Decode the data
        payload = json.loads(base64.b64decode(record['kinesis']['data']))
        
        # Validate data quality
        if validate_sensor_data(payload):
            # Store in database
            store_in_database(payload)
            
            # Check for anomalies
            if detect_anomaly(payload):
                send_alert(payload)
    
    return {'statusCode': 200}

def validate_sensor_data(data):
    """Validate sensor data quality"""
    required_fields = ['sensor_id', 'timestamp', 'temperature', 'pressure']
    
    for field in required_fields:
        if field not in data:
            return False
    
    # Check reasonable ranges
    if not (-50 <= data['temperature'] <= 100):
        return False
    
    if not (0 <= data['pressure'] <= 2000):
        return False
    
    return True
```

---

## Day 4: Database Security

### Morning Session (3 hours)

#### Database Encryption and Access Controls

**Encryption at Rest and in Transit:**
```bash
# Create encrypted RDS instance
aws rds create-db-instance \
  --db-instance-identifier secure-research-db \
  --storage-encrypted \
  --kms-key-id alias/research-db-key \
  --engine postgres \
  --engine-version 13.7 \
  --db-instance-class db.r5.large

# Enable SSL/TLS for connections
aws rds modify-db-instance \
  --db-instance-identifier secure-research-db \
  --ca-certificate-identifier rds-ca-2019 \
  --apply-immediately
```

**Database Access Controls:**
```sql
-- Create database roles
CREATE ROLE research_readers;
CREATE ROLE research_writers;
CREATE ROLE research_admins;

-- Grant permissions
GRANT SELECT ON ALL TABLES IN SCHEMA research_data TO research_readers;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA research_data TO research_writers;
GRANT ALL PRIVILEGES ON SCHEMA research_data TO research_admins;

-- Create application user
CREATE USER research_app WITH PASSWORD 'secure_password';
GRANT research_writers TO research_app;

-- Enable audit logging
ALTER SYSTEM SET log_statement = 'all';
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
```

### Afternoon Session (2 hours)

#### Database Monitoring and Compliance

**Database Activity Monitoring:**
```python
import boto3
import json

def monitor_database_activity():
    """Monitor database for suspicious activity"""
    
    rds = boto3.client('rds')
    logs = boto3.client('logs')
    
    # Get recent log events
    response = logs.filter_log_events(
        logGroupName='/aws/rds/instance/research-database/postgresql',
        startTime=int((datetime.now() - timedelta(hours=1)).timestamp() * 1000)
    )
    
    suspicious_patterns = [
        'DROP TABLE',
        'DELETE FROM',
        'TRUNCATE',
        'failed authentication',
        'connection limit exceeded'
    ]
    
    alerts = []
    for event in response['events']:
        message = event['message'].lower()
        for pattern in suspicious_patterns:
            if pattern.lower() in message:
                alerts.append({
                    'timestamp': event['timestamp'],
                    'message': event['message'],
                    'pattern': pattern
                })
    
    if alerts:
        send_security_alert(alerts)
    
    return alerts

def send_security_alert(alerts):
    """Send security alert for suspicious database activity"""
    sns = boto3.client('sns')
    
    message = f"Database security alert: {len(alerts)} suspicious activities detected"
    
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:database-security-alerts',
        Message=json.dumps(alerts, indent=2),
        Subject=message
    )
```

---

## Day 5: Data Migration and Best Practices

### Morning Session (3 hours)

#### Migrating from HPC File Storage to Databases

**Data Migration Strategy:**
```python
#!/usr/bin/env python3
"""
Migrate HPC scientific data files to cloud database
"""
import pandas as pd
import psycopg2
import boto3
from pathlib import Path
import h5py
import netCDF4

class HPCDataMigrator:
    def __init__(self, db_connection_string):
        self.db_conn = psycopg2.connect(db_connection_string)
        self.s3_client = boto3.client('s3')
    
    def migrate_csv_files(self, file_path, table_name):
        """Migrate CSV files to PostgreSQL"""
        
        # Read CSV file
        df = pd.read_csv(file_path)
        
        # Data validation and cleaning
        df = self.validate_and_clean_data(df)
        
        # Insert into database
        df.to_sql(table_name, self.db_conn, if_exists='append', index=False)
        
        # Archive original file to S3
        self.archive_file_to_s3(file_path)
    
    def migrate_hdf5_files(self, file_path, experiment_id):
        """Migrate HDF5 files to database and S3"""
        
        with h5py.File(file_path, 'r') as hdf_file:
            # Extract metadata
            metadata = dict(hdf_file.attrs)
            
            # Store metadata in database
            self.store_experiment_metadata(experiment_id, metadata)
            
            # Extract datasets
            for dataset_name in hdf_file.keys():
                dataset = hdf_file[dataset_name]
                
                if dataset.size < 1000000:  # Small datasets go to database
                    self.store_dataset_in_db(experiment_id, dataset_name, dataset[:])
                else:  # Large datasets go to S3
                    s3_key = f"experiments/{experiment_id}/{dataset_name}.h5"
                    self.store_dataset_in_s3(dataset, s3_key)
                    self.store_dataset_reference(experiment_id, dataset_name, s3_key)
    
    def validate_and_clean_data(self, df):
        """Validate and clean scientific data"""
        
        # Remove invalid measurements
        df = df.dropna(subset=['temperature', 'pressure'])
        
        # Apply quality filters
        df = df[(df['temperature'] >= -50) & (df['temperature'] <= 100)]
        df = df[(df['pressure'] >= 0) & (df['pressure'] <= 2000)]
        
        # Standardize timestamps
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        
        return df
    
    def archive_file_to_s3(self, file_path):
        """Archive original file to S3"""
        key = f"archive/{Path(file_path).name}"
        self.s3_client.upload_file(str(file_path), 'research-data-archive', key)

if __name__ == "__main__":
    migrator = HPCDataMigrator("postgresql://user:pass@host:5432/research")
    
    # Migrate all CSV files in directory
    data_dir = Path("/scratch/research-data")
    for csv_file in data_dir.glob("*.csv"):
        migrator.migrate_csv_files(csv_file, "measurements")
```

### Afternoon Session (2 hours)

#### Database Performance Optimization

**Query Optimization for Scientific Data:**
```sql
-- Create materialized views for common queries
CREATE MATERIALIZED VIEW research_warehouse.monthly_temperature_summary AS
SELECT 
    experiment_id,
    DATE_TRUNC('month', timestamp) as month,
    AVG(temperature) as avg_temperature,
    MIN(temperature) as min_temperature,
    MAX(temperature) as max_temperature,
    STDDEV(temperature) as temp_stddev,
    COUNT(*) as measurement_count
FROM research_data.measurements
GROUP BY experiment_id, DATE_TRUNC('month', timestamp);

-- Create index on materialized view
CREATE INDEX idx_monthly_summary_experiment_month 
ON research_warehouse.monthly_temperature_summary(experiment_id, month);

-- Refresh materialized view (can be automated)
REFRESH MATERIALIZED VIEW research_warehouse.monthly_temperature_summary;

-- Partitioning for large tables
CREATE TABLE research_data.measurements_partitioned (
    measurement_id UUID DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    temperature DECIMAL(10,4),
    pressure DECIMAL(10,4),
    PRIMARY KEY (measurement_id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE research_data.measurements_2024_01 
PARTITION OF research_data.measurements_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE research_data.measurements_2024_02 
PARTITION OF research_data.measurements_partitioned
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

---

## Week 9 Deliverable: Database Architecture and Migration Plan

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Database strategy for scientific data management
- Benefits of managed database services vs. file-based storage
- Migration timeline and resource requirements
- Cost analysis and optimization strategies

### Database Architecture Design (4-5 pages)
- Relational database design for scientific data
- NoSQL database implementation for unstructured data
- Data warehousing and analytics architecture
- Real-time data processing and streaming analytics

### Security Implementation (3-4 pages)
- Database encryption and access controls
- Network security and VPC configuration
- Audit logging and compliance monitoring
- Backup and disaster recovery security

### Migration Strategy (3-4 pages)
- Current state assessment of HPC data storage
- Data migration planning and execution
- Validation and testing procedures
- Performance optimization and tuning

### Operational Procedures (2-3 pages)
- Database monitoring and maintenance
- Performance optimization and scaling
- Backup and recovery procedures
- Cost monitoring and optimization

---

## Success Metrics for Week 9

You should be able to:
- [ ] Design secure database architectures for scientific data
- [ ] Implement both relational and NoSQL databases
- [ ] Create data warehousing and analytics solutions
- [ ] Plan and execute data migration from HPC file systems
- [ ] Optimize database performance and costs

---

## Week 9 Wrap-Up

By the end of week 9, you should understand how cloud database services can provide better structure, security, and analytics capabilities than traditional file-based storage. Managed databases offer automatic scaling, backup, and maintenance while providing powerful query and analytics capabilities.

**Next week**: We'll explore serverless architectures and event-driven computing, including how to build scalable, cost-effective solutions using Function-as-a-Service platforms.