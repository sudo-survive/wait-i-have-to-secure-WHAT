# Week 5: Cloud Compute Services
## From Job Schedulers to Elastic Compute and Serverless

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how cloud compute services differ from HPC job scheduling
- Configure and manage virtual machines, auto-scaling, and load balancing
- Implement serverless computing for event-driven workloads
- Design compute architectures for different workload patterns
- Apply security best practices to cloud compute services
- Optimize compute costs and performance in cloud environments

---

## Day 1: Virtual Machines vs. HPC Compute Nodes

### Morning Session (3 hours)

#### Compute Model Comparison

**HPC Compute Model (what you know):**
- Fixed cluster of compute nodes
- Job scheduler allocates resources
- Batch processing with defined start/end
- Shared nodes over time (sequential)
- Resource requests specify exact requirements
- Queue-based resource allocation

**Cloud Compute Model:**
- On-demand virtual machines
- Self-service resource provisioning
- Always-on services and applications
- Dedicated instances (concurrent isolation)
- Elastic scaling based on demand
- API-driven resource management

#### Virtual Machine Fundamentals

**Instance Types and Sizing:**
```bash
# HPC resource request (what you know)
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=32
#SBATCH --mem=128GB
#SBATCH --time=24:00:00

# Cloud equivalent (AWS EC2)
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --count 4 \
  --instance-type c5.9xlarge \  # 36 vCPUs, 72 GB RAM
  --key-name my-keypair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678
```

**Instance Lifecycle Management:**
```bash
# Launch instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name research-key \
  --security-group-ids sg-research \
  --user-data file://startup-script.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=ClimateModel}]'

# Monitor instance status
aws ec2 describe-instances --instance-ids i-1234567890abcdef0

# Stop/start instances (like job completion)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Terminate instances (permanent deletion)
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Afternoon Session (2 hours)

#### Hands-On Instance Management

**Create Research Computing Instance:**
```bash
#!/bin/bash
# startup-script.sh - Cloud equivalent of job script

# Update system
yum update -y

# Install research software (like module load)
yum install -y gcc openmpi python3 python3-pip

# Mount shared storage (like /scratch)
mkdir /shared-data
mount -t efs fs-12345678.efs.us-east-1.amazonaws.com:/ /shared-data

# Set up environment
export PATH=/opt/research-tools/bin:$PATH
export LD_LIBRARY_PATH=/opt/research-tools/lib:$LD_LIBRARY_PATH

# Run research application
cd /shared-data/project
python3 climate_simulation.py --config config.json

# Upload results (like job completion)
aws s3 cp results/ s3://research-results/experiment-001/ --recursive
```

**Instance Security Configuration:**
```bash
# Create security group for research instances
aws ec2 create-security-group \
  --group-name research-compute \
  --description "Security group for research compute instances"

# Allow SSH access from specific IP range
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/24

# Allow MPI communication between instances
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 1024-65535 \
  --source-group sg-12345678
```

---

## Day 2: Auto Scaling and Load Balancing

### Morning Session (3 hours)

#### Auto Scaling Concepts

**HPC vs. Cloud Scaling:**

| Aspect | HPC Scaling | Cloud Auto Scaling |
|--------|-------------|-------------------|
| **Capacity** | Fixed cluster size | Elastic, on-demand |
| **Scaling Trigger** | Queue length, manual | Metrics-based, automatic |
| **Scale Direction** | Job scheduling only | Scale up and down |
| **Resource Allocation** | Scheduler decides | Auto Scaling Group decides |
| **Cost Model** | Fixed cost | Pay for what you use |

**Auto Scaling Components:**
- **Launch Template**: Instance configuration (like job script template)
- **Auto Scaling Group**: Manages instance lifecycle
- **Scaling Policies**: Rules for when to scale
- **CloudWatch Metrics**: Monitoring data for scaling decisions

#### Auto Scaling Configuration

**Create Launch Template:**
```bash
# Create launch template (like standardized job script)
aws ec2 create-launch-template \
  --launch-template-name research-compute-template \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "c5.large",
    "KeyName": "research-key",
    "SecurityGroupIds": ["sg-12345678"],
    "UserData": "'$(base64 -w 0 startup-script.sh)'",
    "IamInstanceProfile": {
      "Name": "ResearchInstanceProfile"
    },
    "TagSpecifications": [
      {
        "ResourceType": "instance",
        "Tags": [
          {"Key": "Project", "Value": "Research"},
          {"Key": "Environment", "Value": "Compute"}
        ]
      }
    ]
  }'

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name research-compute-asg \
  --launch-template LaunchTemplateName=research-compute-template,Version=1 \
  --min-size 1 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-12345678,subnet-87654321"
```

### Afternoon Session (2 hours)

#### Load Balancing for Distributed Applications

**Load Balancer Types:**
- **Application Load Balancer**: HTTP/HTTPS traffic (web applications)
- **Network Load Balancer**: TCP/UDP traffic (high performance)
- **Gateway Load Balancer**: Third-party appliances

**Create Load Balancer:**
```bash
# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name research-portal-alb \
  --subnets subnet-12345678 subnet-87654321 \
  --security-groups sg-12345678 \
  --scheme internet-facing \
  --type application

# Create target group
aws elbv2 create-target-group \
  --name research-compute-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345678 \
  --health-check-path /health \
  --health-check-interval-seconds 30

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/research-portal-alb/1234567890123456 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/research-compute-targets/1234567890123456
```

---

## Day 3: Serverless Computing

### Morning Session (3 hours)

#### Serverless vs. Traditional Computing

**HPC Job Model:**
- Submit job script to scheduler
- Wait in queue for resources
- Job runs on allocated nodes
- Resources freed when job completes
- Manual resource management

**Serverless Model:**
- Event triggers function execution
- Automatic resource allocation
- Function runs and terminates
- Pay only for execution time
- No server management required

#### AWS Lambda Functions

**Create Simple Lambda Function:**
```python
import json
import boto3
import pandas as pd
from io import StringIO

def lambda_handler(event, context):
    """Process research data triggered by S3 upload"""
    
    # Get S3 event information
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Initialize S3 client
    s3 = boto3.client('s3')
    
    try:
        # Read data from S3
        response = s3.get_object(Bucket=bucket, Key=key)
        data = pd.read_csv(StringIO(response['Body'].read().decode('utf-8')))
        
        # Process data (example: calculate statistics)
        stats = {
            'mean_temperature': float(data['temperature'].mean()),
            'max_temperature': float(data['temperature'].max()),
            'min_temperature': float(data['temperature'].min()),
            'record_count': len(data)
        }
        
        # Save results
        result_key = key.replace('.csv', '_stats.json')
        s3.put_object(
            Bucket=bucket,
            Key=f"processed/{result_key}",
            Body=json.dumps(stats),
            ContentType='application/json'
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Data processed successfully',
                'stats': stats
            })
        }
        
    except Exception as e:
        print(f"Error processing {key}: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

**Deploy Lambda Function:**
```bash
# Create deployment package
zip function.zip lambda_function.py

# Create Lambda function
aws lambda create-function \
  --function-name research-data-processor \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 300 \
  --memory-size 512

# Create S3 trigger
aws lambda add-permission \
  --function-name research-data-processor \
  --principal s3.amazonaws.com \
  --action lambda:InvokeFunction \
  --statement-id s3-trigger

aws s3api put-bucket-notification-configuration \
  --bucket research-data-bucket \
  --notification-configuration file://s3-notification.json
```

### Afternoon Session (2 hours)

#### Event-Driven Architectures

**Common Serverless Patterns:**
- **Data processing**: Process files uploaded to storage
- **API backends**: Handle HTTP requests without servers
- **Scheduled tasks**: Run functions on schedule (like cron jobs)
- **Stream processing**: Process real-time data streams
- **Workflow orchestration**: Coordinate multiple functions

**Step Functions for Workflows:**
```json
{
  "Comment": "Research data processing workflow",
  "StartAt": "ValidateData",
  "States": {
    "ValidateData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-research-data",
      "Next": "ProcessData"
    },
    "ProcessData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:process-research-data",
      "Next": "GenerateReport"
    },
    "GenerateReport": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:generate-report",
      "End": true
    }
  }
}
```

---

## Day 4: Container Services

### Morning Session (3 hours)

#### Containers vs. HPC Applications

**HPC Application Deployment:**
- Module system for software management
- Shared software installations
- Environment variables and paths
- Dependency management challenges
- Version conflicts and compatibility issues

**Container Approach:**
- Package application with all dependencies
- Consistent runtime environment
- Isolated from host system
- Version control for entire stack
- Portable across environments

#### Container Basics

**Docker Fundamentals:**
```dockerfile
# Dockerfile for research application
FROM ubuntu:20.04

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    gfortran \
    openmpi-bin \
    openmpi-common \
    libopenmpi-dev \
    python3 \
    python3-pip

# Install Python dependencies
COPY requirements.txt .
RUN pip3 install -r requirements.txt

# Copy application code
COPY src/ /app/
WORKDIR /app

# Set environment variables
ENV OMP_NUM_THREADS=4
ENV MPI_ROOT=/usr/lib/x86_64-linux-gnu/openmpi

# Default command
CMD ["python3", "climate_model.py"]
```

**Build and Run Container:**
```bash
# Build container image
docker build -t research/climate-model:v1.0 .

# Run container locally
docker run -it --rm \
  -v /data:/app/data \
  -e OMP_NUM_THREADS=8 \
  research/climate-model:v1.0

# Push to container registry
docker tag research/climate-model:v1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/research/climate-model:v1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/research/climate-model:v1.0
```

### Afternoon Session (2 hours)

#### Container Orchestration

**Amazon ECS (Elastic Container Service):**
```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name research-cluster

# Create task definition
aws ecs register-task-definition \
  --family research-task \
  --task-role-arn arn:aws:iam::123456789012:role/ECSTaskRole \
  --execution-role-arn arn:aws:iam::123456789012:role/ECSExecutionRole \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 1024 \
  --memory 2048 \
  --container-definitions file://container-definition.json

# Run task
aws ecs run-task \
  --cluster research-cluster \
  --task-definition research-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-12345678],securityGroups=[sg-12345678],assignPublicIp=ENABLED}"
```

**Container Security Best Practices:**
- Use minimal base images
- Scan images for vulnerabilities
- Run containers as non-root users
- Implement resource limits
- Use secrets management for sensitive data
- Enable logging and monitoring

---

## Day 5: Compute Security and Optimization

### Morning Session (3 hours)

#### Compute Security Best Practices

**Instance Security Hardening:**
```bash
#!/bin/bash
# Instance hardening script

# Update system packages
yum update -y

# Configure SSH security
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Install and configure fail2ban
yum install -y epel-release
yum install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# Configure firewall
firewall-cmd --permanent --remove-service=dhcpv6-client
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# Install CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U ./amazon-cloudwatch-agent.rpm

# Configure log forwarding
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << EOF
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/secure",
            "log_group_name": "/aws/ec2/security",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
EOF

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

### Afternoon Session (2 hours)

#### Cost Optimization Strategies

**Instance Right-Sizing:**
```python
#!/usr/bin/env python3
import boto3
from datetime import datetime, timedelta

def analyze_instance_utilization():
    """Analyze EC2 instance utilization for right-sizing"""
    ec2 = boto3.client('ec2')
    cloudwatch = boto3.client('cloudwatch')
    
    # Get all running instances
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            instance_type = instance['InstanceType']
            
            # Get CPU utilization
            end_time = datetime.utcnow()
            start_time = end_time - timedelta(days=7)
            
            cpu_metrics = cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=start_time,
                EndTime=end_time,
                Period=3600,
                Statistics=['Average']
            )
            
            if cpu_metrics['Datapoints']:
                avg_cpu = sum(dp['Average'] for dp in cpu_metrics['Datapoints']) / len(cpu_metrics['Datapoints'])
                print(f"Instance {instance_id} ({instance_type}): {avg_cpu:.1f}% CPU")
                
                if avg_cpu < 10:
                    print(f"  RECOMMENDATION: Consider downsizing or using Spot instances")
                elif avg_cpu > 80:
                    print(f"  RECOMMENDATION: Consider upsizing instance type")

def implement_spot_instances():
    """Implement Spot instances for cost savings"""
    ec2 = boto3.client('ec2')
    
    # Create Spot fleet request
    spot_config = {
        'IamFleetRole': 'arn:aws:iam::123456789012:role/aws-ec2-spot-fleet-role',
        'AllocationStrategy': 'lowestPrice',
        'TargetCapacity': 4,
        'SpotPrice': '0.10',
        'LaunchSpecifications': [
            {
                'ImageId': 'ami-0abcdef1234567890',
                'InstanceType': 'c5.large',
                'KeyName': 'research-key',
                'SecurityGroups': [{'GroupId': 'sg-12345678'}],
                'SubnetId': 'subnet-12345678',
                'UserData': base64.b64encode(b'#!/bin/bash\necho "Spot instance started"').decode()
            }
        ]
    }
    
    response = ec2.request_spot_fleet(SpotFleetRequestConfig=spot_config)
    print(f"Spot fleet request created: {response['SpotFleetRequestId']}")

if __name__ == "__main__":
    analyze_instance_utilization()
    implement_spot_instances()
```

**Reserved Instances and Savings Plans:**
```bash
# Analyze Reserved Instance recommendations
aws ce get-reservation-purchase-recommendation \
  --service EC2-Instance \
  --account-scope PAYER \
  --lookback-period-in-days SIXTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT

# Purchase Reserved Instance
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id 12345678-1234-1234-1234-123456789012 \
  --instance-count 2
```

---

## Week 5 Deliverable: Cloud Compute Architecture Design

Create a comprehensive document (10-15 pages) covering:

### Executive Summary (1-2 pages)
- Cloud compute approach vs. HPC job scheduling
- Key benefits and challenges of cloud compute adoption
- Architecture recommendations and implementation plan
- Cost analysis and optimization strategies

### Compute Architecture Design (3-4 pages)

**Virtual Machine Strategy:**
- Instance types and sizing for different workloads
- Auto Scaling Group configuration and policies
- Load balancing and high availability design
- Security group and network configuration

**Serverless Integration:**
- Lambda functions for event-driven processing
- Step Functions for workflow orchestration
- API Gateway for service interfaces
- Event sources and triggers

**Container Strategy:**
- Container image design and management
- ECS/EKS cluster configuration
- Service mesh and networking
- CI/CD pipeline integration

### Security Implementation (3-4 pages)

**Compute Security Controls:**
- Instance hardening and configuration management
- Identity and access management for compute resources
- Network security and isolation
- Monitoring and logging configuration

**Container Security:**
- Image scanning and vulnerability management
- Runtime security and monitoring
- Secrets management and configuration
- Compliance and governance

### Cost Optimization (2-3 pages)

**Cost Management Strategy:**
- Right-sizing and utilization optimization
- Spot instances and Reserved Instance strategy
- Auto Scaling policies for cost efficiency
- Monitoring and alerting for cost control

**Performance Optimization:**
- Instance type selection and optimization
- Application performance tuning
- Network and storage optimization
- Monitoring and troubleshooting

### Implementation Roadmap (1-2 pages)
- Phased implementation approach
- Migration strategy from HPC to cloud compute
- Testing and validation procedures
- Success metrics and monitoring

---

## Key Questions Answered This Week

By the end of week 5, you should be able to answer:

**About Cloud Compute:**
- [ ] How do cloud compute services differ from HPC job scheduling?
- [ ] When should you use VMs vs. containers vs. serverless?
- [ ] How do you implement auto-scaling and load balancing?
- [ ] What are the security considerations for cloud compute?

**About Cost and Performance:**
- [ ] How do you optimize cloud compute costs?
- [ ] What are the performance characteristics of different compute options?
- [ ] How do you monitor and troubleshoot compute resources?
- [ ] What are the best practices for resource management?

---

## Success Metrics for Week 5

You should be able to:
- [ ] Design and implement cloud compute architectures
- [ ] Configure auto-scaling and load balancing
- [ ] Deploy and manage serverless functions
- [ ] Implement container-based applications
- [ ] Optimize compute costs and performance

---

## Week 5 Wrap-Up

By the end of week 5, you should understand how cloud compute services provide much more flexibility than traditional HPC job scheduling, but require different design patterns and management approaches. Cloud compute enables elastic scaling, event-driven processing, and pay-per-use models that can be more cost-effective than fixed HPC clusters.

The key insight: Cloud compute is about right-sizing resources for specific workloads and scaling dynamically based on demand, rather than maximizing utilization of fixed resources.

**Next week**: We'll explore containers and orchestration in depth, including Docker, Kubernetes, and how to run complex scientific workloads in containerized environments.