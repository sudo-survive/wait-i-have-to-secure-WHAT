# Week 07: Cloud Bursting & Dynamic Scaling

## Learning Objectives
- Implement automated cloud bursting for HPC workloads
- Design dynamic scaling policies based on resource utilization
- Configure hybrid job schedulers for seamless resource expansion
- Optimize cost and performance through intelligent workload placement
- Monitor and troubleshoot hybrid scaling operations

## Overview
Cloud bursting enables HPC systems to dynamically expand into cloud resources when local capacity is exhausted. This week covers the technologies, strategies, and best practices for implementing seamless hybrid scaling.

## Cloud Bursting Architecture

### Scaling Triggers
- **Queue Depth**: Number of pending jobs exceeds threshold
- **Resource Utilization**: CPU, memory, or storage utilization limits
- **Time-Based**: Predictable peak periods (deadlines, conferences)
- **Cost-Based**: Economic optimization triggers
- **SLA-Based**: Service level agreement requirements

### Scaling Strategies
1. **Reactive Scaling**: Scale after demand increases
2. **Predictive Scaling**: Scale before anticipated demand
3. **Scheduled Scaling**: Pre-planned scaling events
4. **Hybrid Scaling**: Combination of multiple strategies

## Hands-On Lab: Implementing Cloud Bursting

### Exercise 1: SLURM Cloud Integration (120 minutes)

#### SLURM Configuration for Cloud Bursting
```bash
# /etc/slurm/slurm.conf - Cloud partition configuration
PartitionName=cloud Nodes=cloud-[001-100] Default=NO MaxTime=INFINITE State=UP
NodeName=cloud-[001-100] CPUs=4 RealMemory=16000 State=CLOUD

# Cloud node configuration
NodeName=cloud-[001-100] \
    CPUs=4 \
    RealMemory=16000 \
    State=CLOUD \
    CloudAddr=aws-region \
    CloudCredentials=/etc/slurm/aws-credentials
```

#### Cloud Bursting Script
```python
#!/usr/bin/env python3
import boto3
import subprocess
import time
import json
from datetime import datetime

class SLURMCloudBurster:
    def __init__(self, config_file):
        with open(config_file, 'r') as f:
            self.config = json.load(f)
        
        self.ec2_client = boto3.client('ec2', 
            region_name=self.config['aws_region'])
        self.slurm_nodes = {}
        
    def check_queue_depth(self):
        """Check SLURM queue depth to determine scaling need"""
        result = subprocess.run(['squeue', '-h', '-t', 'PD'], 
                              capture_output=True, text=True)
        pending_jobs = len(result.stdout.strip().split('\n')) if result.stdout.strip() else 0
        
        return pending_jobs
    
    def launch_cloud_nodes(self, node_count):
        """Launch EC2 instances for SLURM compute nodes"""
        user_data_script = f"""#!/bin/bash
# Install SLURM and configure as compute node
yum update -y
yum install -y slurm slurm-slurmd munge

# Configure munge key
echo "{self.config['munge_key']}" > /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
chown munge:munge /etc/munge/munge.key

# Configure SLURM
cat > /etc/slurm/slurm.conf << 'EOF'
{self.generate_slurm_config()}
EOF

# Start services
systemctl enable munge slurmd
systemctl start munge slurmd

# Register with SLURM controller
scontrol update NodeName=$HOSTNAME State=IDLE
"""
        
        response = self.ec2_client.run_instances(
            ImageId=self.config['ami_id'],
            MinCount=node_count,
            MaxCount=node_count,
            InstanceType=self.config['instance_type'],
            KeyName=self.config['key_pair'],
            SecurityGroupIds=[self.config['security_group']],
            SubnetId=self.config['subnet_id'],
            UserData=user_data_script,
            TagSpecifications=[{
                'ResourceType': 'instance',
                'Tags': [
                    {'Key': 'Name', 'Value': 'SLURM-Cloud-Node'},
                    {'Key': 'Purpose', 'Value': 'HPC-Bursting'}
                ]
            }]
        )
        
        return [instance['InstanceId'] for instance in response['Instances']]
    
    def monitor_and_scale(self):
        """Main monitoring and scaling loop"""
        while True:
            pending_jobs = self.check_queue_depth()
            current_cloud_nodes = self.get_active_cloud_nodes()
            
            # Scaling decision logic
            if pending_jobs > self.config['scale_up_threshold']:
                nodes_needed = min(
                    (pending_jobs // self.config['jobs_per_node']) + 1,
                    self.config['max_cloud_nodes'] - len(current_cloud_nodes)
                )
                
                if nodes_needed > 0:
                    print(f"Scaling up: launching {nodes_needed} cloud nodes")
                    self.launch_cloud_nodes(nodes_needed)
            
            elif pending_jobs < self.config['scale_down_threshold']:
                # Scale down logic
                idle_nodes = self.get_idle_cloud_nodes()
                if idle_nodes:
                    print(f"Scaling down: terminating {len(idle_nodes)} idle nodes")
                    self.terminate_cloud_nodes(idle_nodes)
            
            time.sleep(self.config['monitoring_interval'])
```

### Exercise 2: Kubernetes-Based Hybrid Scaling (90 minutes)

#### Cluster Autoscaler Configuration
```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/hpc-hybrid-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: us-west-2
```

#### HPC Job Operator
```python
import kubernetes
from kubernetes import client, config
import yaml

class HPCJobOperator:
    def __init__(self):
        config.load_incluster_config()
        self.v1 = client.CoreV1Api()
        self.batch_v1 = client.BatchV1Api()
        
    def create_hpc_job(self, job_spec):
        """Create Kubernetes job for HPC workload"""
        job_manifest = {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {
                "name": f"hpc-job-{job_spec['job_id']}",
                "labels": {
                    "app": "hpc-workload",
                    "job-type": job_spec['type']
                }
            },
            "spec": {
                "parallelism": job_spec['parallel_tasks'],
                "completions": job_spec['parallel_tasks'],
                "template": {
                    "spec": {
                        "containers": [{
                            "name": "hpc-container",
                            "image": job_spec['container_image'],
                            "resources": {
                                "requests": {
                                    "cpu": job_spec['cpu_request'],
                                    "memory": job_spec['memory_request']
                                },
                                "limits": {
                                    "cpu": job_spec['cpu_limit'],
                                    "memory": job_spec['memory_limit']
                                }
                            },
                            "env": [
                                {"name": "JOB_ID", "value": str(job_spec['job_id'])},
                                {"name": "TASK_ID", "value": "$(JOB_COMPLETION_INDEX)"}
                            ],
                            "volumeMounts": [{
                                "name": "shared-storage",
                                "mountPath": "/shared"
                            }]
                        }],
                        "volumes": [{
                            "name": "shared-storage",
                            "persistentVolumeClaim": {
                                "claimName": "hpc-shared-pvc"
                            }
                        }],
                        "restartPolicy": "Never",
                        "nodeSelector": {
                            "node-type": "compute"
                        }
                    }
                }
            }
        }
        
        return self.batch_v1.create_namespaced_job(
            namespace="hpc-workloads",
            body=job_manifest
        )
```

### Exercise 3: Cost-Aware Scaling (75 minutes)

#### Cost Optimization Engine
```python
import boto3
from datetime import datetime, timedelta

class CostAwareScaler:
    def __init__(self):
        self.ec2_client = boto3.client('ec2')
        self.pricing_client = boto3.client('pricing', region_name='us-east-1')
        
    def get_spot_prices(self, instance_types, availability_zones):
        """Get current spot prices for instance types"""
        spot_prices = {}
        
        for instance_type in instance_types:
            response = self.ec2_client.describe_spot_price_history(
                InstanceTypes=[instance_type],
                ProductDescriptions=['Linux/UNIX'],
                MaxResults=1,
                AvailabilityZones=availability_zones
            )
            
            if response['SpotPriceHistory']:
                spot_prices[instance_type] = {
                    'price': float(response['SpotPriceHistory'][0]['SpotPrice']),
                    'timestamp': response['SpotPriceHistory'][0]['Timestamp']
                }
        
        return spot_prices
    
    def calculate_scaling_cost(self, scaling_plan):
        """Calculate cost of scaling plan"""
        total_cost = 0
        
        for node_type, count in scaling_plan.items():
            instance_info = self.get_instance_pricing(node_type)
            hourly_cost = instance_info['on_demand_price']
            
            # Consider spot pricing if available
            if instance_info['spot_available']:
                spot_price = instance_info['spot_price']
                if spot_price < hourly_cost * 0.7:  # Use spot if >30% savings
                    hourly_cost = spot_price
            
            total_cost += hourly_cost * count
        
        return total_cost
    
    def optimize_scaling_decision(self, workload_requirements, budget_constraints):
        """Optimize scaling decision based on cost and performance"""
        scaling_options = self.generate_scaling_options(workload_requirements)
        
        best_option = None
        best_score = 0
        
        for option in scaling_options:
            cost = self.calculate_scaling_cost(option['instances'])
            performance_score = self.estimate_performance(option, workload_requirements)
            
            if cost <= budget_constraints['max_hourly_cost']:
                # Score based on performance/cost ratio
                score = performance_score / cost
                
                if score > best_score:
                    best_score = score
                    best_option = option
        
        return best_option
```

## Performance Monitoring and Optimization

### Hybrid Performance Metrics
```python
class HybridPerformanceMonitor:
    def __init__(self):
        self.metrics = {
            'queue_times': [],
            'job_completion_times': [],
            'resource_utilization': {},
            'cost_per_job': [],
            'scaling_events': []
        }
    
    def collect_slurm_metrics(self):
        """Collect performance metrics from SLURM"""
        # Queue depth and wait times
        queue_info = subprocess.run(['squeue', '-h', '-o', '%i,%T,%M'], 
                                  capture_output=True, text=True)
        
        for line in queue_info.stdout.strip().split('\n'):
            if line:
                job_id, state, time_str = line.split(',')
                if state == 'PD':  # Pending
                    wait_time = self.parse_time_string(time_str)
                    self.metrics['queue_times'].append(wait_time)
    
    def analyze_scaling_efficiency(self):
        """Analyze efficiency of scaling operations"""
        analysis = {
            'average_scale_up_time': 0,
            'average_scale_down_time': 0,
            'resource_waste': 0,
            'cost_efficiency': 0
        }
        
        # Calculate metrics from collected data
        scaling_events = self.metrics['scaling_events']
        
        if scaling_events:
            scale_up_times = [event['duration'] for event in scaling_events 
                            if event['type'] == 'scale_up']
            scale_down_times = [event['duration'] for event in scaling_events 
                              if event['type'] == 'scale_down']
            
            analysis['average_scale_up_time'] = sum(scale_up_times) / len(scale_up_times)
            analysis['average_scale_down_time'] = sum(scale_down_times) / len(scale_down_times)
        
        return analysis
```

## Troubleshooting Common Issues

### Scaling Problems and Solutions
```python
class ScalingTroubleshooter:
    def __init__(self):
        self.common_issues = {
            'slow_scaling': self.diagnose_slow_scaling,
            'failed_launches': self.diagnose_launch_failures,
            'cost_overruns': self.diagnose_cost_issues,
            'performance_degradation': self.diagnose_performance_issues
        }
    
    def diagnose_slow_scaling(self):
        """Diagnose slow scaling issues"""
        checks = [
            self.check_api_rate_limits(),
            self.check_instance_availability(),
            self.check_network_connectivity(),
            self.check_image_launch_time()
        ]
        
        return {
            'issue': 'slow_scaling',
            'checks': checks,
            'recommendations': self.generate_scaling_recommendations(checks)
        }
    
    def diagnose_launch_failures(self):
        """Diagnose instance launch failures"""
        failure_reasons = self.analyze_launch_logs()
        
        common_fixes = {
            'InsufficientInstanceCapacity': 'Try different instance types or AZs',
            'InvalidAMI.ID.NotFound': 'Verify AMI ID and region',
            'UnauthorizedOperation': 'Check IAM permissions',
            'InvalidKeyPair.NotFound': 'Verify key pair exists'
        }
        
        return {
            'failures': failure_reasons,
            'fixes': [common_fixes.get(reason, 'Check AWS documentation') 
                     for reason in failure_reasons]
        }
```

## Week 7 Deliverables

### Cloud Bursting Implementation
1. **Scaling Architecture Design** (2-3 pages)
   - Scaling triggers and policies
   - Cost optimization strategy
   - Performance monitoring plan

2. **Implementation Documentation** (2 pages)
   - Configuration files and scripts
   - Testing results and validation
   - Troubleshooting procedures

3. **Performance Analysis Report** (1-2 pages)
   - Scaling efficiency metrics
   - Cost-benefit analysis
   - Optimization recommendations

### Practical Exercise
- Implement cloud bursting for lab environment
- Configure automated scaling policies
- Test scaling under various load conditions
- Create monitoring dashboard for hybrid resources

---

*Cloud bursting can provide 10x capacity expansion when needed. The key is making it seamless and cost-effective.*