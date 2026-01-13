# Week 6: Containers and Orchestration
## From Module Systems to Docker and Kubernetes

---

## Learning Objectives

By the end of this week, you should be able to:
- Understand how containers differ from HPC module systems and job isolation
- Build, deploy, and manage containerized applications
- Implement Kubernetes for container orchestration at scale
- Design microservices architectures for scientific applications
- Apply container security best practices
- Migrate HPC workloads to containerized environments

---

## Day 1: Container Fundamentals

### Morning Session (3 hours)

#### Containers vs. HPC Software Management

**HPC Software Management (what you know):**
- Module system (Environment Modules, Lmod)
- Shared software installations in /opt or /usr/local
- Environment variables for library paths
- Version conflicts and dependency hell
- System-wide installations requiring admin privileges

**Container Approach:**
- Package application with all dependencies
- Isolated runtime environment
- Immutable infrastructure
- Version control for entire software stack
- User-level container execution

#### Docker Basics

**Container Lifecycle:**
```bash
# Build container (like compiling software)
docker build -t research/climate-model:v1.0 .

# Run container (like submitting job)
docker run -it --rm research/climate-model:v1.0

# List running containers (like squeue)
docker ps

# Stop container (like scancel)
docker stop container-id

# Remove container (cleanup)
docker rm container-id
```

**Dockerfile for Scientific Application:**
```dockerfile
# Start with scientific computing base image
FROM continuumio/miniconda3:latest

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    gfortran \
    libopenmpi-dev \
    openmpi-bin \
    && rm -rf /var/lib/apt/lists/*

# Create conda environment
COPY environment.yml .
RUN conda env create -f environment.yml

# Activate environment
SHELL ["conda", "run", "-n", "research", "/bin/bash", "-c"]

# Copy application code
COPY src/ ./src/
COPY data/ ./data/
COPY scripts/ ./scripts/

# Set environment variables
ENV OMP_NUM_THREADS=4
ENV PYTHONPATH=/app/src

# Default command
CMD ["conda", "run", "-n", "research", "python", "src/main.py"]
```

### Afternoon Session (2 hours)

#### Container Registry and Image Management

**Container Registry Operations:**
```bash
# Login to registry
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag image for registry
docker tag research/climate-model:v1.0 \
123456789012.dkr.ecr.us-east-1.amazonaws.com/research/climate-model:v1.0

# Push image to registry
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/research/climate-model:v1.0

# Pull image from registry
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/research/climate-model:v1.0
```

**Multi-stage Builds for Optimization:**
```dockerfile
# Build stage
FROM continuumio/miniconda3:latest AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

COPY src/ ./src/
RUN python -m compileall src/

# Runtime stage
FROM continuumio/miniconda3:latest

# Copy only necessary files from builder
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app/src /app/src

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

WORKDIR /app
CMD ["python", "src/main.py"]
```

---

## Day 2: Container Orchestration with Kubernetes

### Morning Session (3 hours)

#### Kubernetes vs. Job Schedulers

**HPC Job Scheduler (Slurm/PBS):**
- Allocates compute nodes to jobs
- Manages job queues and priorities
- Handles resource limits and accounting
- Provides job lifecycle management
- Focuses on batch processing

**Kubernetes:**
- Orchestrates containers across cluster
- Manages service discovery and networking
- Handles scaling and self-healing
- Provides declarative configuration
- Focuses on long-running services

#### Kubernetes Core Concepts

**Kubernetes Resources:**
```yaml
# Pod (like a single job)
apiVersion: v1
kind: Pod
metadata:
  name: climate-simulation
  labels:
    app: climate-model
spec:
  containers:
  - name: simulation
    image: research/climate-model:v1.0
    resources:
      requests:
        memory: "4Gi"
        cpu: "2"
      limits:
        memory: "8Gi"
        cpu: "4"
    env:
    - name: OMP_NUM_THREADS
      value: "4"
    volumeMounts:
    - name: data-volume
      mountPath: /app/data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: research-data-pvc
```

**Deployment (like job array):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: research-workers
spec:
  replicas: 5
  selector:
    matchLabels:
      app: research-worker
  template:
    metadata:
      labels:
        app: research-worker
    spec:
      containers:
      - name: worker
        image: research/data-processor:v1.0
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
        env:
        - name: WORKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Afternoon Session (2 hours)

#### Kubernetes Services and Networking

**Service Discovery:**
```yaml
# Service (like MPI communication)
apiVersion: v1
kind: Service
metadata:
  name: research-api
spec:
  selector:
    app: research-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

**ConfigMaps and Secrets:**
```yaml
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: research-config
data:
  config.yaml: |
    simulation:
      timesteps: 1000
      output_frequency: 100
    parallel:
      mpi_processes: 4
      openmp_threads: 2

---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: research-secrets
type: Opaque
data:
  api_key: <base64-encoded-key>
  database_password: <base64-encoded-password>
```

---

## Day 3: Scientific Workloads in Kubernetes

### Morning Session (3 hours)

#### Batch Jobs in Kubernetes

**Kubernetes Job (like Slurm job):**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: climate-simulation-job
spec:
  parallelism: 4  # Like --ntasks
  completions: 4  # Total number of successful completions needed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: simulation
        image: research/climate-model:v1.0
        command: ["mpirun", "-np", "4", "python", "simulation.py"]
        resources:
          requests:
            memory: "8Gi"
            cpu: "4"
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: research-shared-pvc
```

**CronJob for Scheduled Tasks:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-data-processing
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: processor
            image: research/data-processor:v1.0
            command: ["python", "daily_processing.py"]
            env:
            - name: DATE
              value: "$(date +%Y-%m-%d)"
```

### Afternoon Session (2 hours)

#### High-Performance Computing in Kubernetes

**MPI Jobs in Kubernetes:**
```yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: mpi-climate-simulation
spec:
  slotsPerWorker: 4
  runPolicy:
    cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
          - image: research/mpi-climate:v1.0
            name: mpi-launcher
            command:
            - mpirun
            - -np
            - "16"
            - python
            - /app/climate_simulation.py
    Worker:
      replicas: 4
      template:
        spec:
          containers:
          - image: research/mpi-climate:v1.0
            name: mpi-worker
            resources:
              requests:
                cpu: 4
                memory: 8Gi
```

**GPU Workloads:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-ml-training
spec:
  containers:
  - name: ml-trainer
    image: research/ml-trainer:gpu
    resources:
      limits:
        nvidia.com/gpu: 2  # Request 2 GPUs
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0,1"
```

---

## Day 4: Container Security

### Morning Session (3 hours)

#### Container Security Best Practices

**Image Security:**
```dockerfile
# Use minimal base images
FROM alpine:3.18

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Install only necessary packages
RUN apk add --no-cache python3 py3-pip

# Copy application files
COPY --chown=appuser:appgroup src/ /app/src/

# Switch to non-root user
USER appuser

WORKDIR /app
CMD ["python3", "src/main.py"]
```

**Security Scanning:**
```bash
# Scan image for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy image research/climate-model:v1.0

# Scan with Snyk
snyk container test research/climate-model:v1.0

# Use Docker Scout
docker scout cves research/climate-model:v1.0
```

#### Kubernetes Security

**Pod Security Standards:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-research-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
  containers:
  - name: research-app
    image: research/secure-app:v1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      requests:
        memory: "2Gi"
        cpu: "1"
```

### Afternoon Session (2 hours)

#### Network Policies and RBAC

**Network Policies:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: research-network-policy
spec:
  podSelector:
    matchLabels:
      app: research-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: research-frontend
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
```

**RBAC Configuration:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: research-job-manager
rules:
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: research-job-manager-binding
subjects:
- kind: User
  name: research-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: research-job-manager
  apiGroup: rbac.authorization.k8s.io
```

---

## Day 5: Migration and Best Practices

### Morning Session (3 hours)

#### Migrating HPC Workloads to Containers

**Assessment and Planning:**
```python
#!/usr/bin/env python3
"""
HPC to Container Migration Assessment Tool
"""
import os
import subprocess
import json

def assess_hpc_application(app_path):
    """Assess HPC application for containerization"""
    assessment = {
        'application_path': app_path,
        'dependencies': [],
        'environment_variables': [],
        'file_dependencies': [],
        'recommendations': []
    }
    
    # Check for common HPC dependencies
    hpc_deps = ['mpi', 'openmp', 'cuda', 'mkl', 'fftw']
    for dep in hpc_deps:
        if check_dependency(dep):
            assessment['dependencies'].append(dep)
    
    # Check environment variables
    env_vars = ['OMP_NUM_THREADS', 'MPI_ROOT', 'CUDA_HOME', 'LD_LIBRARY_PATH']
    for var in env_vars:
        if var in os.environ:
            assessment['environment_variables'].append({
                'name': var,
                'value': os.environ[var]
            })
    
    # Generate recommendations
    if 'mpi' in assessment['dependencies']:
        assessment['recommendations'].append(
            "Consider using MPI operator for Kubernetes"
        )
    
    if 'cuda' in assessment['dependencies']:
        assessment['recommendations'].append(
            "Use NVIDIA GPU operator for GPU workloads"
        )
    
    return assessment

def check_dependency(dep_name):
    """Check if dependency is available"""
    try:
        result = subprocess.run(['which', dep_name], 
                              capture_output=True, text=True)
        return result.returncode == 0
    except:
        return False

def generate_dockerfile(assessment):
    """Generate Dockerfile based on assessment"""
    dockerfile_content = """
# Generated Dockerfile for HPC application
FROM ubuntu:20.04

# Install system dependencies
RUN apt-get update && apt-get install -y \\
"""
    
    if 'mpi' in assessment['dependencies']:
        dockerfile_content += "    openmpi-bin openmpi-common libopenmpi-dev \\\n"
    
    if 'cuda' in assessment['dependencies']:
        dockerfile_content += "    nvidia-cuda-toolkit \\\n"
    
    dockerfile_content += """    && rm -rf /var/lib/apt/lists/*

# Set environment variables
"""
    
    for env_var in assessment['environment_variables']:
        dockerfile_content += f"ENV {env_var['name']}={env_var['value']}\n"
    
    dockerfile_content += """
# Copy application
COPY . /app
WORKDIR /app

# Default command
CMD ["./run.sh"]
"""
    
    return dockerfile_content

if __name__ == "__main__":
    app_path = "/path/to/hpc/application"
    assessment = assess_hpc_application(app_path)
    
    print("HPC Application Assessment:")
    print(json.dumps(assessment, indent=2))
    
    dockerfile = generate_dockerfile(assessment)
    with open("Dockerfile", "w") as f:
        f.write(dockerfile)
    
    print("\nGenerated Dockerfile saved as 'Dockerfile'")
```

### Afternoon Session (2 hours)

#### Container Orchestration Best Practices

**Resource Management:**
```yaml
# Resource quotas for research namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: research-quota
  namespace: research
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "10"
    pods: "50"

---
# Limit ranges for individual pods
apiVersion: v1
kind: LimitRange
metadata:
  name: research-limits
  namespace: research
spec:
  limits:
  - default:
      cpu: "2"
      memory: "4Gi"
    defaultRequest:
      cpu: "1"
      memory: "2Gi"
    type: Container
```

**Monitoring and Observability:**
```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: research-app-metrics
spec:
  selector:
    matchLabels:
      app: research-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

---

## Week 6 Deliverable: Container Migration Strategy

Create a comprehensive document (12-18 pages) covering:

### Executive Summary (1-2 pages)
- Container adoption strategy for HPC workloads
- Benefits and challenges of containerization
- Migration timeline and resource requirements
- Integration with existing HPC infrastructure

### Container Architecture Design (4-5 pages)

**Containerization Strategy:**
- Application assessment and containerization approach
- Base image selection and optimization
- Multi-stage build strategies
- Container registry and image management

**Orchestration Platform:**
- Kubernetes cluster design and configuration
- Namespace and resource management
- Service mesh and networking architecture
- Storage and persistent volume strategy

### Migration Planning (4-5 pages)

**Current State Assessment:**
- HPC application inventory and dependencies
- Resource requirements and constraints
- Integration points and data flows
- User workflows and access patterns

**Migration Approach:**
- Phased migration strategy
- Pilot application selection
- Testing and validation procedures
- Rollback and contingency planning

### Security Implementation (2-3 pages)

**Container Security:**
- Image scanning and vulnerability management
- Runtime security and monitoring
- Network policies and micro-segmentation
- Secrets management and configuration

**Kubernetes Security:**
- RBAC and access control
- Pod security standards
- Network policies and isolation
- Audit logging and monitoring

### Operational Procedures (2-3 pages)

**Container Lifecycle Management:**
- Build, test, and deployment pipelines
- Image versioning and rollback procedures
- Monitoring and alerting configuration
- Troubleshooting and support processes

**Performance and Optimization:**
- Resource allocation and scaling policies
- Performance monitoring and tuning
- Cost optimization strategies
- Capacity planning and management

---

## Key Questions Answered This Week

By the end of week 6, you should be able to answer:

**About Containers:**
- [ ] How do containers differ from HPC module systems?
- [ ] When should you containerize HPC applications?
- [ ] How do you build secure, optimized container images?
- [ ] What are the performance implications of containerization?

**About Orchestration:**
- [ ] How does Kubernetes compare to HPC job schedulers?
- [ ] How do you run MPI and GPU workloads in Kubernetes?
- [ ] What are the security considerations for container orchestration?
- [ ] How do you monitor and troubleshoot containerized applications?

---

## Success Metrics for Week 6

You should be able to:
- [ ] Build and deploy containerized applications
- [ ] Design Kubernetes architectures for scientific workloads
- [ ] Implement container security best practices
- [ ] Plan migration from HPC to containerized environments
- [ ] Manage container orchestration at scale

---

## Week 6 Wrap-Up

By the end of week 6, you should understand how containers and orchestration provide a modern approach to application deployment and management that can complement or replace traditional HPC software management. Containers offer better reproducibility, portability, and scalability than traditional HPC approaches, but require different operational models.

The key insight: Containers are about packaging applications with their dependencies for consistent execution across environments, while orchestration platforms like Kubernetes provide the scheduling and management capabilities similar to HPC job schedulers but for containerized workloads.

**Next week**: We'll explore cloud-native application security, including how to secure microservices, APIs, and distributed applications in cloud environments.