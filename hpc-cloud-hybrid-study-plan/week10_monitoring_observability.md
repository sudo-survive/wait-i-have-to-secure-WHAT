# Week 10: Monitoring & Observability

## Learning Objectives
- Implement comprehensive monitoring across hybrid HPC-cloud environments
- Design observability strategies for distributed workloads
- Configure alerting and incident response for hybrid systems
- Create performance dashboards and analytics
- Implement distributed tracing for complex workflows

## Overview
Monitoring hybrid environments requires unified visibility across different platforms, protocols, and performance characteristics. This week covers monitoring tools, observability practices, and analytics for hybrid HPC-cloud systems.

## Monitoring Architecture

### Multi-Layer Monitoring
1. **Infrastructure Layer**: Hardware, network, storage metrics
2. **Platform Layer**: Kubernetes, SLURM, cloud services
3. **Application Layer**: Job performance, workflow execution
4. **Business Layer**: Cost, SLA compliance, research outcomes

### Key Metrics Categories
- **Performance**: Throughput, latency, resource utilization
- **Availability**: Uptime, error rates, service health
- **Cost**: Resource consumption, budget tracking
- **Security**: Access patterns, anomaly detection

## Hands-On Lab: Unified Monitoring Implementation

### Exercise 1: Prometheus-Based Monitoring (120 minutes)

#### Multi-Cluster Prometheus Setup
```yaml
# prometheus-federation.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
    # HPC cluster metrics
    - job_name: 'hpc-nodes'
      static_configs:
      - targets: ['hpc-node-01:9100', 'hpc-node-02:9100']
      scrape_interval: 30s
      metrics_path: /metrics
      
    # SLURM metrics
    - job_name: 'slurm-exporter'
      static_configs:
      - targets: ['slurm-controller:9341']
      
    # Cloud cluster federation
    - job_name: 'federate-cloud'
      scrape_interval: 15s
      honor_labels: true
      metrics_path: '/federate'
      params:
        'match[]':
          - '{job=~"kubernetes-.*"}'
          - '{job=~"node-exporter"}'
      static_configs:
      - targets: ['cloud-prometheus:9090']
    
    # Custom HPC application metrics
    - job_name: 'hpc-applications'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['hpc-workloads']
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: storage-volume
          mountPath: /prometheus
        args:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus'
        - '--web.console.libraries=/etc/prometheus/console_libraries'
        - '--web.console.templates=/etc/prometheus/consoles'
        - '--storage.tsdb.retention.time=30d'
        - '--web.enable-lifecycle'
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: storage-volume
        persistentVolumeClaim:
          claimName: prometheus-storage
```

#### Custom HPC Metrics Exporter
```python
#!/usr/bin/env python3
import time
import subprocess
import re
from prometheus_client import start_http_server, Gauge, Counter, Histogram
from prometheus_client.core import CollectorRegistry

class HPCMetricsExporter:
    def __init__(self, port=9342):
        self.port = port
        self.registry = CollectorRegistry()
        
        # Define metrics
        self.slurm_jobs_total = Counter(
            'slurm_jobs_total',
            'Total number of SLURM jobs',
            ['state', 'partition', 'user'],
            registry=self.registry
        )
        
        self.slurm_nodes_total = Gauge(
            'slurm_nodes_total',
            'Total number of SLURM nodes',
            ['state'],
            registry=self.registry
        )
        
        self.lustre_throughput = Gauge(
            'lustre_throughput_bytes_per_second',
            'Lustre filesystem throughput',
            ['operation', 'filesystem'],
            registry=self.registry
        )
        
        self.job_wait_time = Histogram(
            'slurm_job_wait_time_seconds',
            'Job wait time in queue',
            ['partition'],
            registry=self.registry
        )
        
        self.infiniband_throughput = Gauge(
            'infiniband_throughput_bytes_per_second',
            'InfiniBand network throughput',
            ['port', 'direction'],
            registry=self.registry
        )
    
    def collect_slurm_metrics(self):
        """Collect SLURM job and node metrics"""
        try:
            # Get job information
            result = subprocess.run(['squeue', '-h', '-o', '%T,%P,%u,%M'], 
                                  capture_output=True, text=True)
            
            job_counts = {}
            for line in result.stdout.strip().split('\n'):
                if line:
                    state, partition, user, time_str = line.split(',')
                    
                    # Count jobs by state/partition/user
                    key = (state, partition, user)
                    job_counts[key] = job_counts.get(key, 0) + 1
                    
                    # Track wait times for pending jobs
                    if state == 'PENDING':
                        wait_time = self.parse_time_string(time_str)
                        self.job_wait_time.labels(partition=partition).observe(wait_time)
            
            # Update job counters
            for (state, partition, user), count in job_counts.items():
                self.slurm_jobs_total.labels(
                    state=state, partition=partition, user=user
                )._value._value = count
            
            # Get node information
            result = subprocess.run(['sinfo', '-h', '-o', '%T,%D'], 
                                  capture_output=True, text=True)
            
            node_counts = {}
            for line in result.stdout.strip().split('\n'):
                if line:
                    state, count = line.split(',')
                    node_counts[state] = int(count)
            
            for state, count in node_counts.items():
                self.slurm_nodes_total.labels(state=state).set(count)
                
        except Exception as e:
            print(f"Error collecting SLURM metrics: {e}")
    
    def collect_lustre_metrics(self):
        """Collect Lustre filesystem metrics"""
        try:
            # Read Lustre stats
            with open('/proc/fs/lustre/llite/lustre-*/stats', 'r') as f:
                stats = f.read()
            
            # Parse throughput metrics
            read_bytes = re.search(r'read_bytes\s+\d+\s+samples.*?(\d+)\s+bytes', stats)
            write_bytes = re.search(r'write_bytes\s+\d+\s+samples.*?(\d+)\s+bytes', stats)
            
            if read_bytes:
                self.lustre_throughput.labels(
                    operation='read', filesystem='lustre'
                ).set(int(read_bytes.group(1)))
            
            if write_bytes:
                self.lustre_throughput.labels(
                    operation='write', filesystem='lustre'
                ).set(int(write_bytes.group(1)))
                
        except Exception as e:
            print(f"Error collecting Lustre metrics: {e}")
    
    def collect_infiniband_metrics(self):
        """Collect InfiniBand network metrics"""
        try:
            result = subprocess.run(['ibstat'], capture_output=True, text=True)
            
            # Parse InfiniBand port statistics
            port_pattern = r'Port (\d+):'
            data_pattern = r'Port\s+xmit\s+data:\s+(\d+)\s+bytes.*?Port\s+rcv\s+data:\s+(\d+)\s+bytes'
            
            ports = re.findall(port_pattern, result.stdout)
            data_matches = re.findall(data_pattern, result.stdout, re.DOTALL)
            
            for i, port in enumerate(ports):
                if i < len(data_matches):
                    xmit_bytes, rcv_bytes = data_matches[i]
                    
                    self.infiniband_throughput.labels(
                        port=port, direction='transmit'
                    ).set(int(xmit_bytes))
                    
                    self.infiniband_throughput.labels(
                        port=port, direction='receive'
                    ).set(int(rcv_bytes))
                    
        except Exception as e:
            print(f"Error collecting InfiniBand metrics: {e}")
    
    def start_server(self):
        """Start the metrics server"""
        start_http_server(self.port, registry=self.registry)
        
        while True:
            self.collect_slurm_metrics()
            self.collect_lustre_metrics()
            self.collect_infiniband_metrics()
            time.sleep(30)  # Collect metrics every 30 seconds

if __name__ == '__main__':
    exporter = HPCMetricsExporter()
    exporter.start_server()
```

### Exercise 2: Grafana Dashboard Creation (90 minutes)

#### HPC-Cloud Hybrid Dashboard
```json
{
  "dashboard": {
    "title": "HPC-Cloud Hybrid Overview",
    "panels": [
      {
        "title": "Resource Utilization",
        "type": "stat",
        "targets": [
          {
            "expr": "100 * (1 - avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])))",
            "legendFormat": "CPU Utilization %"
          },
          {
            "expr": "100 * (1 - avg(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))",
            "legendFormat": "Memory Utilization %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "green", "value": 0},
                {"color": "yellow", "value": 70},
                {"color": "red", "value": 90}
              ]
            }
          }
        }
      },
      {
        "title": "Job Queue Status",
        "type": "graph",
        "targets": [
          {
            "expr": "slurm_jobs_total{state=\"PENDING\"}",
            "legendFormat": "Pending Jobs"
          },
          {
            "expr": "slurm_jobs_total{state=\"RUNNING\"}",
            "legendFormat": "Running Jobs"
          },
          {
            "expr": "slurm_jobs_total{state=\"COMPLETED\"}",
            "legendFormat": "Completed Jobs"
          }
        ]
      },
      {
        "title": "Cross-Environment Data Transfer",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(node_network_transmit_bytes_total{device=\"eth0\"}[5m])",
            "legendFormat": "Network Out - {{instance}}"
          },
          {
            "expr": "rate(node_network_receive_bytes_total{device=\"eth0\"}[5m])",
            "legendFormat": "Network In - {{instance}}"
          }
        ]
      },
      {
        "title": "Cost Tracking",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(aws_ec2_instance_cost_per_hour * on(instance_id) group_left() aws_ec2_instance_running)",
            "legendFormat": "Hourly Cloud Cost"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "currencyUSD"
          }
        }
      }
    ]
  }
}
```

### Exercise 3: Distributed Tracing Implementation (75 minutes)

#### Jaeger Tracing for Workflows
```python
import opentracing
from jaeger_client import Config
import time

class WorkflowTracer:
    def __init__(self, service_name):
        config = Config(
            config={
                'sampler': {'type': 'const', 'param': 1},
                'logging': True,
                'reporter_batch_size': 1,
            },
            service_name=service_name,
        )
        self.tracer = config.initialize_tracer()
        opentracing.set_global_tracer(self.tracer)
    
    def trace_workflow_execution(self, workflow_id, steps):
        """Trace complete workflow execution"""
        with self.tracer.start_span('workflow_execution') as workflow_span:
            workflow_span.set_tag('workflow.id', workflow_id)
            workflow_span.set_tag('workflow.steps', len(steps))
            
            for i, step in enumerate(steps):
                self.trace_workflow_step(step, workflow_span, i)
    
    def trace_workflow_step(self, step, parent_span, step_index):
        """Trace individual workflow step"""
        with self.tracer.start_span(
            f'step_{step["name"]}', 
            child_of=parent_span
        ) as step_span:
            
            step_span.set_tag('step.index', step_index)
            step_span.set_tag('step.environment', step['environment'])
            step_span.set_tag('step.resource_type', step['resource_type'])
            
            # Trace data movement if applicable
            if step.get('input_data'):
                self.trace_data_transfer(
                    step['input_data'], 
                    step['environment'], 
                    step_span
                )
            
            # Execute step and measure performance
            start_time = time.time()
            try:
                result = self.execute_step(step)
                step_span.set_tag('step.status', 'success')
                step_span.set_tag('step.result_size', len(str(result)))
            except Exception as e:
                step_span.set_tag('step.status', 'error')
                step_span.set_tag('step.error', str(e))
                step_span.log_kv({'event': 'error', 'message': str(e)})
            finally:
                execution_time = time.time() - start_time
                step_span.set_tag('step.execution_time', execution_time)
    
    def trace_data_transfer(self, data_info, target_environment, parent_span):
        """Trace data transfer between environments"""
        with self.tracer.start_span(
            'data_transfer', 
            child_of=parent_span
        ) as transfer_span:
            
            transfer_span.set_tag('data.size', data_info['size'])
            transfer_span.set_tag('data.source', data_info['source'])
            transfer_span.set_tag('data.destination', target_environment)
            transfer_span.set_tag('data.transfer_method', data_info['method'])
            
            # Measure transfer performance
            start_time = time.time()
            try:
                self.perform_data_transfer(data_info, target_environment)
                transfer_time = time.time() - start_time
                
                transfer_span.set_tag('transfer.duration', transfer_time)
                transfer_span.set_tag('transfer.throughput', 
                                    data_info['size'] / transfer_time)
                transfer_span.set_tag('transfer.status', 'success')
                
            except Exception as e:
                transfer_span.set_tag('transfer.status', 'error')
                transfer_span.log_kv({'event': 'transfer_error', 'message': str(e)})
```

## Alerting and Incident Response

### Alert Rules Configuration
```yaml
# prometheus-alerts.yaml
groups:
- name: hpc-hybrid-alerts
  rules:
  - alert: HighJobQueueDepth
    expr: slurm_jobs_total{state="PENDING"} > 100
    for: 5m
    labels:
      severity: warning
      environment: hpc
    annotations:
      summary: "High number of pending jobs in SLURM queue"
      description: "{{ $value }} jobs are pending in the queue for more than 5 minutes"
  
  - alert: CloudCostSpike
    expr: increase(aws_billing_estimated_charges[1h]) > 100
    for: 0m
    labels:
      severity: critical
      environment: cloud
    annotations:
      summary: "Unexpected cloud cost increase"
      description: "Cloud costs increased by ${{ $value }} in the last hour"
  
  - alert: CrossEnvironmentLatencyHigh
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="hybrid-gateway"}[5m])) > 1
    for: 2m
    labels:
      severity: warning
      environment: hybrid
    annotations:
      summary: "High latency between HPC and cloud environments"
      description: "95th percentile latency is {{ $value }}s"
  
  - alert: StorageCapacityLow
    expr: (node_filesystem_avail_bytes{mountpoint="/lustre"} / node_filesystem_size_bytes{mountpoint="/lustre"}) * 100 < 10
    for: 1m
    labels:
      severity: critical
      environment: hpc
    annotations:
      summary: "Lustre filesystem capacity critically low"
      description: "Only {{ $value }}% storage capacity remaining"
```

## Week 10 Deliverables

### Monitoring Implementation
1. **Observability Architecture** (2-3 pages)
   - Multi-environment monitoring design
   - Metrics collection strategy
   - Alerting and escalation procedures

2. **Dashboard and Analytics** (2 pages)
   - Performance dashboard screenshots
   - Key performance indicators (KPIs)
   - Trend analysis and insights

3. **Incident Response Plan** (1-2 pages)
   - Alert response procedures
   - Escalation matrix
   - Post-incident review process

---

*You can't manage what you can't measure. Comprehensive monitoring is essential for hybrid success.*