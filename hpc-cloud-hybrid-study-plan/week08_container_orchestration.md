# Week 08: Container Orchestration Across Environments

## Learning Objectives
- Deploy containerized HPC workloads across hybrid environments
- Configure Kubernetes clusters spanning HPC and cloud resources
- Implement container-based job scheduling and resource management
- Optimize container performance for HPC applications
- Manage persistent storage and networking for hybrid containers

## Overview
Containers provide a consistent runtime environment that can span HPC clusters and cloud platforms. This week covers container orchestration strategies, performance optimization, and best practices for hybrid deployments.

## Container Technologies for HPC

### Container Runtimes
- **Docker**: Standard containerization platform
- **Singularity/Apptainer**: HPC-focused container runtime
- **Podman**: Daemonless container engine
- **containerd**: Industry-standard container runtime

### Orchestration Platforms
- **Kubernetes**: Cloud-native orchestration
- **Docker Swarm**: Simple container orchestration
- **Nomad**: Flexible workload orchestrator
- **SLURM**: HPC scheduler with container support

## Hands-On Lab: Hybrid Container Orchestration

### Exercise 1: Multi-Cluster Kubernetes Setup (120 minutes)

#### HPC Cluster Configuration
```yaml
# hpc-cluster-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hpc-cluster-config
  namespace: kube-system
data:
  cluster-type: "hpc"
  scheduler: "slurm-integration"
  storage-class: "lustre-storage"
  network-plugin: "infiniband-cni"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hpc-node-setup
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: hpc-node-setup
  template:
    metadata:
      labels:
        app: hpc-node-setup
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: hpc-setup
        image: hpc-node-configurator:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: host-root
          mountPath: /host
        env:
        - name: NODE_TYPE
          value: "hpc-compute"
      volumes:
      - name: host-root
        hostPath:
          path: /
```

#### Cloud Cluster Integration
```python
#!/usr/bin/env python3
import kubernetes
from kubernetes import client, config
import yaml

class HybridClusterManager:
    def __init__(self):
        self.clusters = {
            'hpc': self.load_cluster_config('hpc-cluster'),
            'cloud': self.load_cluster_config('cloud-cluster')
        }
        
    def deploy_workload(self, workload_spec, target_clusters):
        """Deploy workload across multiple clusters"""
        deployment_results = {}
        
        for cluster_name in target_clusters:
            try:
                # Switch to target cluster context
                config.load_kube_config(context=f"{cluster_name}-context")
                
                # Create cluster-specific deployment
                cluster_deployment = self.adapt_for_cluster(
                    workload_spec, cluster_name
                )
                
                # Deploy to cluster
                result = self.create_deployment(cluster_deployment)
                deployment_results[cluster_name] = result
                
            except Exception as e:
                deployment_results[cluster_name] = {'error': str(e)}
        
        return deployment_results
    
    def adapt_for_cluster(self, workload_spec, cluster_name):
        """Adapt workload specification for specific cluster"""
        adapted_spec = workload_spec.copy()
        
        if cluster_name == 'hpc':
            # HPC-specific adaptations
            adapted_spec['spec']['template']['spec']['nodeSelector'] = {
                'node-type': 'hpc-compute'
            }
            adapted_spec['spec']['template']['spec']['volumes'].append({
                'name': 'lustre-storage',
                'persistentVolumeClaim': {
                    'claimName': 'hpc-lustre-pvc'
                }
            })
        elif cluster_name == 'cloud':
            # Cloud-specific adaptations
            adapted_spec['spec']['template']['spec']['nodeSelector'] = {
                'node-type': 'cloud-compute'
            }
            adapted_spec['spec']['template']['spec']['volumes'].append({
                'name': 'cloud-storage',
                'persistentVolumeClaim': {
                    'claimName': 'cloud-efs-pvc'
                }
            })
        
        return adapted_spec
```

### Exercise 2: HPC Container Optimization (90 minutes)

#### Singularity Integration with SLURM
```bash
#!/bin/bash
# SLURM job script for containerized HPC workload

#SBATCH --job-name=hybrid-container-job
#SBATCH --partition=gpu
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=2
#SBATCH --mem=32G
#SBATCH --time=02:00:00
#SBATCH --output=hybrid-job-%j.out

# Load required modules
module load singularity/3.8.0
module load openmpi/4.1.0

# Define container image
CONTAINER_IMAGE="/shared/containers/hpc-workload.sif"

# Set up MPI environment for containers
export SINGULARITYENV_OMPI_MCA_btl_vader_single_copy_mechanism=none
export SINGULARITYENV_OMPI_MCA_btl_tcp_if_include=ib0

# Run MPI job across containers
mpirun -np $SLURM_NTASKS \
    singularity exec --bind /lustre:/data \
    $CONTAINER_IMAGE \
    /opt/application/run_simulation.sh
```

#### Container Performance Tuning
```dockerfile
# Dockerfile for HPC-optimized container
FROM ubuntu:20.04

# Install HPC libraries and tools
RUN apt-get update && apt-get install -y \
    build-essential \
    gfortran \
    libopenmpi-dev \
    libhdf5-dev \
    libblas-dev \
    liblapack-dev \
    numactl \
    hwloc-nox

# Optimize for HPC performance
RUN echo 'kernel.shmmax = 68719476736' >> /etc/sysctl.conf
RUN echo 'kernel.shmall = 4294967296' >> /etc/sysctl.conf

# Set CPU affinity and memory binding
ENV OMP_PROC_BIND=true
ENV OMP_PLACES=cores
ENV MALLOC_ARENA_MAX=4

# Install application
COPY application/ /opt/application/
RUN cd /opt/application && make install

# Set up runtime environment
WORKDIR /opt/application
ENTRYPOINT ["/opt/application/entrypoint.sh"]
```

### Exercise 3: Cross-Cluster Service Mesh (75 minutes)

#### Istio Multi-Cluster Setup
```yaml
# istio-hpc-cloud-mesh.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: hpc-cloud-mesh
spec:
  values:
    pilot:
      env:
        EXTERNAL_ISTIOD: true
        CROSS_NETWORK_POLICY: true
    global:
      meshID: hpc-cloud-mesh
      clusterName: hpc-cluster
      network: hpc-network
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cross-cluster-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15021
      name: status-port
      protocol: TLS
    tls:
      mode: ISTIO_MUTUAL
    hosts:
    - cross-network-policy.local
```

## Storage and Networking for Containers

### Persistent Volume Management
```yaml
# hybrid-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hybrid-storage
provisioner: csi.hybrid-storage.io
parameters:
  type: "distributed"
  replication: "3"
  performance-tier: "high"
allowVolumeExpansion: true
reclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hpc-workload-storage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: hybrid-storage
  resources:
    requests:
      storage: 1Ti
```

### Network Performance Optimization
```python
class ContainerNetworkOptimizer:
    def __init__(self):
        self.network_configs = {}
        
    def optimize_for_hpc(self, pod_spec):
        """Optimize network configuration for HPC workloads"""
        # Enable SR-IOV for high-performance networking
        pod_spec['spec']['containers'][0]['resources']['requests'].update({
            'intel.com/sriov_netdevice': '1'
        })
        
        # Configure DPDK for userspace networking
        pod_spec['spec']['containers'][0]['env'].append({
            'name': 'DPDK_ENABLED',
            'value': 'true'
        })
        
        # Set CPU affinity for network interrupts
        pod_spec['spec']['containers'][0]['env'].append({
            'name': 'IRQ_AFFINITY',
            'value': '0-3'  # Dedicate cores 0-3 for network processing
        })
        
        return pod_spec
    
    def configure_infiniband(self, deployment_spec):
        """Configure InfiniBand networking for containers"""
        # Add InfiniBand device plugin
        deployment_spec['spec']['template']['spec']['containers'][0]['resources']['limits'].update({
            'rdma/hca': '1'
        })
        
        # Mount InfiniBand devices
        deployment_spec['spec']['template']['spec']['volumes'].append({
            'name': 'infiniband-devices',
            'hostPath': {
                'path': '/dev/infiniband'
            }
        })
        
        deployment_spec['spec']['template']['spec']['containers'][0]['volumeMounts'].append({
            'name': 'infiniband-devices',
            'mountPath': '/dev/infiniband'
        })
        
        return deployment_spec
```

## Week 8 Deliverables

### Container Orchestration Architecture
1. **Multi-Cluster Design** (2-3 pages)
   - Cluster topology and networking
   - Service mesh configuration
   - Storage integration strategy

2. **Performance Optimization Guide** (2 pages)
   - Container tuning parameters
   - Network optimization techniques
   - Storage performance considerations

3. **Implementation Results** (1-2 pages)
   - Deployment testing outcomes
   - Performance benchmarks
   - Troubleshooting documentation

---

*Containers provide consistency across hybrid environments. Optimize them properly for HPC performance requirements.*