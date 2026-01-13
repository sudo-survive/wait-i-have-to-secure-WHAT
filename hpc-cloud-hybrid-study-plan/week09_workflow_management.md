# Week 09: Workflow Management & Job Scheduling

## Learning Objectives
- Design complex workflows spanning HPC and cloud environments
- Implement workflow orchestration tools for hybrid deployments
- Configure intelligent job scheduling across multiple resource pools
- Optimize workflow performance and resource utilization
- Handle fault tolerance and recovery in distributed workflows

## Overview
Workflow management in hybrid environments requires orchestrating complex dependencies, data movement, and resource allocation across different platforms. This week covers workflow engines, scheduling strategies, and optimization techniques.

## Workflow Orchestration Tools

### Workflow Engines
- **Nextflow**: Scalable and portable workflow engine
- **Cromwell**: WDL workflow execution engine
- **Airflow**: Platform for developing and scheduling workflows
- **Argo Workflows**: Kubernetes-native workflow engine
- **Snakemake**: Python-based workflow management system

### Scheduling Integration
- **SLURM**: Traditional HPC job scheduling
- **Kubernetes Jobs**: Cloud-native job scheduling
- **HTCondor**: High-throughput computing scheduler
- **PBS/Torque**: Portable batch system

## Hands-On Lab: Hybrid Workflow Implementation

### Exercise 1: Nextflow Hybrid Pipeline (120 minutes)

#### Multi-Environment Nextflow Configuration
```groovy
// nextflow.config
profiles {
    hpc {
        process.executor = 'slurm'
        process.queue = 'compute'
        process.clusterOptions = '--account=research --qos=normal'
        
        executor {
            name = 'slurm'
            queueSize = 100
            pollInterval = '30 sec'
        }
        
        singularity {
            enabled = true
            cacheDir = '/shared/containers'
        }
    }
    
    cloud {
        process.executor = 'k8s'
        
        k8s {
            namespace = 'nextflow-workflows'
            serviceAccount = 'nextflow-service-account'
            storageClaimName = 'workflow-storage'
            storageMountPath = '/workspace'
        }
        
        docker {
            enabled = true
            registry = 'your-registry.com'
        }
    }
    
    hybrid {
        includeConfig 'conf/hpc.config'
        includeConfig 'conf/cloud.config'
        
        process {
            withLabel: 'compute_intensive' {
                executor = 'slurm'
                queue = 'gpu'
                clusterOptions = '--gres=gpu:4'
            }
            
            withLabel: 'data_processing' {
                executor = 'k8s'
                pod = [
                    [volumeClaim: 'data-pvc', mountPath: '/data'],
                    [secret: 'aws-credentials', mountPath: '/credentials']
                ]
            }
        }
    }
}
```

#### Hybrid Workflow Definition
```groovy
// main.nf
#!/usr/bin/env nextflow

nextflow.enable.dsl=2

// Define workflow parameters
params.input_data = '/shared/input/*.fastq'
params.reference_genome = '/shared/reference/genome.fa'
params.output_dir = '/shared/results'

// Import workflow modules
include { PREPROCESSING } from './modules/preprocessing'
include { ALIGNMENT } from './modules/alignment'
include { VARIANT_CALLING } from './modules/variant_calling'
include { ANNOTATION } from './modules/annotation'

workflow HYBRID_GENOMICS {
    take:
        input_files
        reference
    
    main:
        // Preprocessing on cloud (scalable, cost-effective)
        preprocessed = PREPROCESSING(input_files)
        
        // Alignment on HPC (compute-intensive, high memory)
        aligned = ALIGNMENT(preprocessed, reference)
        
        // Variant calling on HPC (CPU-intensive)
        variants = VARIANT_CALLING(aligned, reference)
        
        // Annotation on cloud (database access, web services)
        annotated = ANNOTATION(variants)
    
    emit:
        results = annotated
}

workflow {
    input_ch = Channel.fromPath(params.input_data)
    reference_ch = Channel.fromPath(params.reference_genome)
    
    HYBRID_GENOMICS(input_ch, reference_ch)
}
```

#### Process Definitions with Environment Selection
```groovy
// modules/alignment.nf
process ALIGNMENT {
    label 'compute_intensive'
    
    publishDir "${params.output_dir}/alignments", mode: 'copy'
    
    input:
    tuple val(sample_id), path(reads)
    path reference
    
    output:
    tuple val(sample_id), path("${sample_id}.bam")
    
    script:
    """
    # Use BWA-MEM for alignment
    bwa mem -t ${task.cpus} ${reference} ${reads} | \
    samtools sort -@ ${task.cpus} -o ${sample_id}.bam -
    
    # Index the BAM file
    samtools index ${sample_id}.bam
    """
}

process ANNOTATION {
    label 'data_processing'
    
    publishDir "${params.output_dir}/annotations", mode: 'copy'
    
    input:
    tuple val(sample_id), path(variants)
    
    output:
    tuple val(sample_id), path("${sample_id}.annotated.vcf")
    
    script:
    """
    # Use cloud-based annotation service
    vep --input_file ${variants} \
        --output_file ${sample_id}.annotated.vcf \
        --database \
        --cache_version 104 \
        --assembly GRCh38 \
        --format vcf
    """
}
```

### Exercise 2: Argo Workflows for Kubernetes (90 minutes)

#### Hybrid Workflow Template
```yaml
# hybrid-workflow-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hybrid-data-pipeline
spec:
  entrypoint: main-pipeline
  
  templates:
  - name: main-pipeline
    dag:
      tasks:
      - name: data-ingestion
        template: ingest-data
        arguments:
          parameters:
          - name: source-path
            value: "{{workflow.parameters.input-path}}"
      
      - name: preprocessing
        template: preprocess-data
        dependencies: [data-ingestion]
        arguments:
          parameters:
          - name: input-data
            value: "{{tasks.data-ingestion.outputs.parameters.output-path}}"
      
      - name: hpc-computation
        template: hpc-compute
        dependencies: [preprocessing]
        arguments:
          parameters:
          - name: preprocessed-data
            value: "{{tasks.preprocessing.outputs.parameters.output-path}}"
      
      - name: results-analysis
        template: analyze-results
        dependencies: [hpc-computation]
        arguments:
          parameters:
          - name: computation-results
            value: "{{tasks.hpc-computation.outputs.parameters.output-path}}"
  
  - name: hpc-compute
    inputs:
      parameters:
      - name: preprocessed-data
    outputs:
      parameters:
      - name: output-path
        valueFrom:
          path: /tmp/output-path
    nodeSelector:
      node-type: hpc-gateway
    container:
      image: hpc-job-submitter:latest
      command: ["/bin/bash"]
      args:
      - -c
      - |
        # Submit job to SLURM cluster
        sbatch --wait --output=/tmp/slurm-%j.out \
               --job-name=argo-hpc-job \
               --nodes=4 \
               --ntasks-per-node=16 \
               /scripts/run-computation.sh {{inputs.parameters.preprocessed-data}}
        
        # Get output path from SLURM job
        echo "/shared/results/$(date +%Y%m%d_%H%M%S)" > /tmp/output-path
      volumeMounts:
      - name: shared-storage
        mountPath: /shared
      - name: slurm-config
        mountPath: /etc/slurm
```

### Exercise 3: Intelligent Job Scheduling (75 minutes)

#### Multi-Resource Scheduler
```python
import heapq
import time
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum

class ResourceType(Enum):
    HPC_CPU = "hpc_cpu"
    HPC_GPU = "hpc_gpu"
    CLOUD_CPU = "cloud_cpu"
    CLOUD_GPU = "cloud_gpu"

@dataclass
class Job:
    job_id: str
    resource_requirements: Dict[ResourceType, int]
    estimated_runtime: int  # minutes
    priority: int
    deadline: Optional[int] = None  # timestamp
    cost_budget: Optional[float] = None

@dataclass
class Resource:
    resource_type: ResourceType
    total_capacity: int
    available_capacity: int
    cost_per_hour: float
    performance_factor: float  # relative performance

class HybridJobScheduler:
    def __init__(self):
        self.job_queue = []
        self.resources = {}
        self.running_jobs = {}
        
    def add_resource_pool(self, resource_type: ResourceType, 
                         capacity: int, cost_per_hour: float, 
                         performance_factor: float = 1.0):
        """Add a resource pool to the scheduler"""
        self.resources[resource_type] = Resource(
            resource_type=resource_type,
            total_capacity=capacity,
            available_capacity=capacity,
            cost_per_hour=cost_per_hour,
            performance_factor=performance_factor
        )
    
    def submit_job(self, job: Job):
        """Submit a job to the scheduler"""
        # Calculate job priority score
        priority_score = self.calculate_priority_score(job)
        
        # Add to priority queue (negative score for min-heap)
        heapq.heappush(self.job_queue, (-priority_score, time.time(), job))
    
    def calculate_priority_score(self, job: Job) -> float:
        """Calculate job priority score based on multiple factors"""
        base_priority = job.priority
        
        # Deadline urgency factor
        urgency_factor = 1.0
        if job.deadline:
            time_to_deadline = job.deadline - time.time()
            urgency_factor = max(0.1, 1.0 / (time_to_deadline / 3600))  # hours
        
        # Resource availability factor
        availability_factor = self.calculate_availability_factor(job)
        
        return base_priority * urgency_factor * availability_factor
    
    def schedule_jobs(self):
        """Main scheduling loop"""
        scheduled_jobs = []
        
        while self.job_queue:
            _, submit_time, job = heapq.heappop(self.job_queue)
            
            # Find optimal resource allocation
            allocation = self.find_optimal_allocation(job)
            
            if allocation:
                # Reserve resources and start job
                self.allocate_resources(job, allocation)
                scheduled_jobs.append((job, allocation))
            else:
                # Put job back in queue if no resources available
                heapq.heappush(self.job_queue, 
                             (-self.calculate_priority_score(job), submit_time, job))
                break
        
        return scheduled_jobs
    
    def find_optimal_allocation(self, job: Job) -> Optional[Dict[ResourceType, int]]:
        """Find optimal resource allocation for a job"""
        best_allocation = None
        best_score = float('inf')
        
        # Generate possible allocations
        possible_allocations = self.generate_allocations(job)
        
        for allocation in possible_allocations:
            if self.can_allocate(allocation):
                score = self.calculate_allocation_score(job, allocation)
                
                if score < best_score:
                    best_score = score
                    best_allocation = allocation
        
        return best_allocation
    
    def calculate_allocation_score(self, job: Job, 
                                 allocation: Dict[ResourceType, int]) -> float:
        """Calculate score for a resource allocation"""
        total_cost = 0
        total_performance = 0
        
        for resource_type, count in allocation.items():
            resource = self.resources[resource_type]
            
            # Calculate cost
            runtime_hours = job.estimated_runtime / 60
            cost = count * resource.cost_per_hour * runtime_hours
            total_cost += cost
            
            # Calculate performance benefit
            performance = count * resource.performance_factor
            total_performance += performance
        
        # Score based on cost-performance ratio
        if total_performance == 0:
            return float('inf')
        
        score = total_cost / total_performance
        
        # Apply budget constraint penalty
        if job.cost_budget and total_cost > job.cost_budget:
            score *= 10  # Heavy penalty for budget violations
        
        return score
```

## Fault Tolerance and Recovery

### Workflow Checkpointing
```python
class WorkflowCheckpointer:
    def __init__(self, storage_backend):
        self.storage = storage_backend
        
    def create_checkpoint(self, workflow_id, state):
        """Create workflow checkpoint"""
        checkpoint_data = {
            'workflow_id': workflow_id,
            'timestamp': time.time(),
            'state': state,
            'completed_tasks': state.get('completed_tasks', []),
            'failed_tasks': state.get('failed_tasks', []),
            'pending_tasks': state.get('pending_tasks', [])
        }
        
        checkpoint_key = f"checkpoints/{workflow_id}/{int(time.time())}"
        self.storage.save(checkpoint_key, checkpoint_data)
        
        return checkpoint_key
    
    def restore_workflow(self, workflow_id, checkpoint_key=None):
        """Restore workflow from checkpoint"""
        if not checkpoint_key:
            # Find latest checkpoint
            checkpoint_key = self.find_latest_checkpoint(workflow_id)
        
        if checkpoint_key:
            checkpoint_data = self.storage.load(checkpoint_key)
            return self.rebuild_workflow_state(checkpoint_data)
        
        return None
    
    def implement_retry_logic(self, task, max_retries=3):
        """Implement intelligent retry logic"""
        retry_count = 0
        
        while retry_count < max_retries:
            try:
                result = self.execute_task(task)
                return result
            except Exception as e:
                retry_count += 1
                
                # Determine if error is retryable
                if self.is_retryable_error(e):
                    # Exponential backoff
                    wait_time = 2 ** retry_count
                    time.sleep(wait_time)
                    
                    # Potentially switch execution environment
                    if retry_count > 1:
                        task = self.adapt_task_for_retry(task, e)
                else:
                    # Non-retryable error, fail immediately
                    raise e
        
        raise Exception(f"Task failed after {max_retries} retries")
```

## Week 9 Deliverables

### Workflow Management System
1. **Workflow Architecture Design** (2-3 pages)
   - Multi-environment workflow topology
   - Scheduling strategy and policies
   - Fault tolerance and recovery mechanisms

2. **Implementation Documentation** (2 pages)
   - Workflow definitions and configurations
   - Scheduler integration setup
   - Performance optimization techniques

3. **Testing and Validation Report** (1-2 pages)
   - Workflow execution results
   - Performance benchmarks
   - Failure recovery testing

---

*Complex workflows require intelligent orchestration. Design for failure and optimize for the critical path.*