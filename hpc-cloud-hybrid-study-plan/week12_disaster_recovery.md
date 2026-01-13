# Week 12: Disaster Recovery & Business Continuity

## Learning Objectives
- Design comprehensive disaster recovery strategies for hybrid HPC-cloud environments
- Implement automated backup and recovery systems across platforms
- Configure high availability and failover mechanisms
- Develop business continuity plans for research and computational workflows
- Test and validate disaster recovery procedures

## Overview
Disaster recovery in hybrid environments requires coordinated protection across multiple platforms, data centers, and cloud regions. This week covers backup strategies, failover mechanisms, and business continuity planning for mission-critical HPC workloads.

## Disaster Recovery Architecture

### Recovery Objectives
- **Recovery Time Objective (RTO)**: Maximum acceptable downtime
- **Recovery Point Objective (RPO)**: Maximum acceptable data loss
- **Recovery Level Objective (RLO)**: Minimum acceptable service level during recovery

### DR Strategies
1. **Cold Standby**: Backup systems activated manually during disaster
2. **Warm Standby**: Backup systems running but not processing workloads
3. **Hot Standby**: Active-passive configuration with automatic failover
4. **Active-Active**: Workloads distributed across multiple sites

## Hands-On Lab: Comprehensive DR Implementation

### Exercise 1: Multi-Site Backup Strategy (120 minutes)

#### Automated Backup Orchestration
```python
#!/usr/bin/env python3
import boto3
import subprocess
import json
import logging
from datetime import datetime, timedelta
from concurrent.futures import ThreadPoolExecutor, as_completed
import hashlib

class HybridBackupOrchestrator:
    def __init__(self, config_file):
        with open(config_file, 'r') as f:
            self.config = json.load(f)
        
        self.s3_client = boto3.client('s3')
        self.glacier_client = boto3.client('glacier')
        self.backup_metadata = {}
        
        # Setup logging
        logging.basicConfig(level=logging.INFO)
        self.logger = logging.getLogger(__name__)
    
    def orchestrate_backup(self, backup_plan):
        """Orchestrate backup across multiple environments"""
        backup_results = {}
        
        with ThreadPoolExecutor(max_workers=4) as executor:
            # Submit backup tasks
            future_to_source = {}
            
            for source in backup_plan['sources']:
                future = executor.submit(self.backup_source, source)
                future_to_source[future] = source
            
            # Collect results
            for future in as_completed(future_to_source):
                source = future_to_source[future]
                try:
                    result = future.result()
                    backup_results[source['name']] = result
                    self.logger.info(f"Backup completed for {source['name']}")
                except Exception as e:
                    backup_results[source['name']] = {'error': str(e)}
                    self.logger.error(f"Backup failed for {source['name']}: {e}")
        
        # Generate backup report
        self.generate_backup_report(backup_results)
        return backup_results
    
    def backup_source(self, source):
        """Backup a specific data source"""
        source_type = source['type']
        
        if source_type == 'lustre':
            return self.backup_lustre_filesystem(source)
        elif source_type == 'slurm_config':
            return self.backup_slurm_configuration(source)
        elif source_type == 'application_data':
            return self.backup_application_data(source)
        elif source_type == 'user_home':
            return self.backup_user_directories(source)
        else:
            raise ValueError(f"Unknown source type: {source_type}")
    
    def backup_lustre_filesystem(self, source):
        """Backup Lustre filesystem with incremental support"""
        lustre_path = source['path']
        backup_destination = source['destination']
        
        # Create snapshot for consistent backup
        snapshot_name = f"backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        
        try:
            # Create LVM snapshot (assuming Lustre is on LVM)
            subprocess.run([
                'lvcreate', '-L', '10G', '-s', '-n', snapshot_name,
                source['lvm_path']
            ], check=True)
            
            # Mount snapshot
            snapshot_mount = f"/mnt/{snapshot_name}"
            subprocess.run(['mkdir', '-p', snapshot_mount], check=True)
            subprocess.run([
                'mount', f"/dev/{source['volume_group']}/{snapshot_name}",
                snapshot_mount
            ], check=True)
            
            # Perform incremental backup
            if backup_destination.startswith('s3://'):
                backup_result = self.backup_to_s3(snapshot_mount, backup_destination)
            else:
                backup_result = self.backup_to_filesystem(snapshot_mount, backup_destination)
            
            # Cleanup snapshot
            subprocess.run(['umount', snapshot_mount], check=True)
            subprocess.run(['lvremove', '-f', f"{source['volume_group']}/{snapshot_name}"], check=True)
            
            return backup_result
            
        except subprocess.CalledProcessError as e:
            self.logger.error(f"Lustre backup failed: {e}")
            raise
    
    def backup_to_s3(self, source_path, s3_destination):
        """Backup data to S3 with intelligent tiering"""
        bucket_name = s3_destination.replace('s3://', '').split('/')[0]
        s3_prefix = '/'.join(s3_destination.replace('s3://', '').split('/')[1:])
        
        # Use AWS CLI for efficient transfer
        sync_command = [
            'aws', 's3', 'sync', source_path, s3_destination,
            '--storage-class', 'STANDARD_IA',  # Use IA for backup data
            '--exclude', '*.tmp',
            '--exclude', '.snapshot/*'
        ]
        
        result = subprocess.run(sync_command, capture_output=True, text=True)
        
        if result.returncode == 0:
            # Calculate backup size and file count
            backup_stats = self.calculate_backup_stats(source_path)
            
            # Apply lifecycle policy for long-term archival
            self.apply_backup_lifecycle_policy(bucket_name, s3_prefix)
            
            return {
                'status': 'success',
                'destination': s3_destination,
                'size_bytes': backup_stats['total_size'],
                'file_count': backup_stats['file_count'],
                'duration_seconds': backup_stats['duration']
            }
        else:
            raise Exception(f"S3 sync failed: {result.stderr}")
    
    def backup_slurm_configuration(self, source):
        """Backup SLURM configuration and state"""
        config_files = [
            '/etc/slurm/slurm.conf',
            '/etc/slurm/slurmdbd.conf',
            '/etc/slurm/cgroup.conf',
            '/var/lib/slurm/slurm.state'
        ]
        
        backup_data = {}
        
        for config_file in config_files:
            try:
                with open(config_file, 'r') as f:
                    backup_data[config_file] = f.read()
            except FileNotFoundError:
                self.logger.warning(f"Config file not found: {config_file}")
        
        # Backup SLURM database
        db_backup = subprocess.run([
            'mysqldump', '--single-transaction', 'slurm_acct_db'
        ], capture_output=True, text=True)
        
        if db_backup.returncode == 0:
            backup_data['slurm_database'] = db_backup.stdout
        
        # Store backup data
        backup_file = f"slurm_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        
        with open(f"/backup/{backup_file}", 'w') as f:
            json.dump(backup_data, f, indent=2)
        
        # Upload to cloud storage
        self.s3_client.upload_file(
            f"/backup/{backup_file}",
            self.config['backup_bucket'],
            f"slurm_configs/{backup_file}"
        )
        
        return {
            'status': 'success',
            'backup_file': backup_file,
            'components_backed_up': list(backup_data.keys())
        }
```

### Exercise 2: Automated Failover Implementation (90 minutes)

#### Multi-Region Failover System
```python
import boto3
import time
import json
from kubernetes import client, config

class HybridFailoverManager:
    def __init__(self):
        self.primary_region = 'us-east-1'
        self.dr_region = 'us-west-2'
        
        # AWS clients for both regions
        self.primary_ec2 = boto3.client('ec2', region_name=self.primary_region)
        self.dr_ec2 = boto3.client('ec2', region_name=self.dr_region)
        
        # Route53 for DNS failover
        self.route53 = boto3.client('route53')
        
        # Kubernetes clients
        self.k8s_primary = self.setup_k8s_client('primary-cluster')
        self.k8s_dr = self.setup_k8s_client('dr-cluster')
        
        self.failover_state = 'primary'
        
    def monitor_primary_health(self):
        """Continuously monitor primary environment health"""
        health_checks = {
            'hpc_cluster': self.check_hpc_cluster_health(),
            'cloud_services': self.check_cloud_services_health(),
            'network_connectivity': self.check_network_connectivity(),
            'storage_systems': self.check_storage_health()
        }
        
        # Determine overall health
        failed_checks = [check for check, status in health_checks.items() if not status]
        
        if len(failed_checks) >= 2:  # Failover threshold
            self.logger.critical(f"Multiple health checks failed: {failed_checks}")
            return False
        
        return True
    
    def initiate_failover(self, failure_type='automatic'):
        """Initiate failover to DR environment"""
        self.logger.info(f"Initiating {failure_type} failover to DR region")
        
        failover_steps = [
            self.stop_primary_workloads,
            self.activate_dr_infrastructure,
            self.restore_data_to_dr,
            self.redirect_traffic_to_dr,
            self.start_dr_workloads,
            self.notify_stakeholders
        ]
        
        failover_results = {}
        
        for step in failover_steps:
            try:
                step_name = step.__name__
                self.logger.info(f"Executing failover step: {step_name}")
                
                result = step()
                failover_results[step_name] = result
                
                if not result.get('success', False):
                    self.logger.error(f"Failover step failed: {step_name}")
                    # Attempt rollback
                    self.rollback_failover(failover_results)
                    return False
                    
            except Exception as e:
                self.logger.error(f"Exception in failover step {step.__name__}: {e}")
                self.rollback_failover(failover_results)
                return False
        
        self.failover_state = 'dr'
        self.logger.info("Failover completed successfully")
        return True
    
    def activate_dr_infrastructure(self):
        """Activate DR infrastructure components"""
        
        # Start DR compute instances
        dr_instances = self.get_dr_instances()
        
        if dr_instances:
            self.dr_ec2.start_instances(InstanceIds=dr_instances)
            
            # Wait for instances to be running
            waiter = self.dr_ec2.get_waiter('instance_running')
            waiter.wait(InstanceIds=dr_instances)
        
        # Scale up Kubernetes cluster
        self.scale_k8s_cluster(self.k8s_dr, target_nodes=10)
        
        # Activate storage systems
        self.activate_dr_storage()
        
        return {'success': True, 'activated_instances': len(dr_instances)}
    
    def restore_data_to_dr(self):
        """Restore data from backups to DR environment"""
        
        # Get latest backup metadata
        latest_backups = self.get_latest_backup_metadata()
        
        restore_tasks = []
        
        for backup in latest_backups:
            if backup['type'] == 'lustre':
                restore_tasks.append(self.restore_lustre_data(backup))
            elif backup['type'] == 'application_data':
                restore_tasks.append(self.restore_application_data(backup))
            elif backup['type'] == 'user_data':
                restore_tasks.append(self.restore_user_data(backup))
        
        # Execute restore tasks in parallel
        with ThreadPoolExecutor(max_workers=3) as executor:
            restore_results = list(executor.map(lambda task: task(), restore_tasks))
        
        successful_restores = sum(1 for result in restore_results if result['success'])
        
        return {
            'success': successful_restores == len(restore_tasks),
            'restored_datasets': successful_restores,
            'total_datasets': len(restore_tasks)
        }
    
    def redirect_traffic_to_dr(self):
        """Redirect traffic to DR environment using Route53"""
        
        # Update DNS records to point to DR region
        hosted_zone_id = self.config['route53_zone_id']
        
        dns_updates = [
            {
                'name': 'hpc-gateway.organization.edu',
                'new_target': self.get_dr_gateway_ip()
            },
            {
                'name': 'api.hpc.organization.edu',
                'new_target': self.get_dr_api_endpoint()
            }
        ]
        
        for update in dns_updates:
            change_batch = {
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': update['name'],
                        'Type': 'A',
                        'TTL': 60,  # Short TTL for quick failover
                        'ResourceRecords': [{'Value': update['new_target']}]
                    }
                }]
            }
            
            self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
        
        return {'success': True, 'updated_records': len(dns_updates)}
```

### Exercise 3: Business Continuity Testing (75 minutes)

#### DR Testing Framework
```python
class DRTestingFramework:
    def __init__(self):
        self.test_scenarios = [
            'complete_site_failure',
            'partial_infrastructure_failure',
            'data_corruption',
            'network_partition',
            'cyber_attack_simulation'
        ]
        
    def execute_dr_test(self, scenario_name, test_parameters=None):
        """Execute a specific DR test scenario"""
        
        if scenario_name not in self.test_scenarios:
            raise ValueError(f"Unknown test scenario: {scenario_name}")
        
        test_method = getattr(self, f"test_{scenario_name}")
        
        # Pre-test setup
        pre_test_state = self.capture_system_state()
        
        try:
            # Execute test
            test_results = test_method(test_parameters or {})
            
            # Validate recovery
            recovery_validation = self.validate_recovery()
            
            # Cleanup and restore
            self.restore_system_state(pre_test_state)
            
            return {
                'scenario': scenario_name,
                'test_results': test_results,
                'recovery_validation': recovery_validation,
                'success': test_results['success'] and recovery_validation['success']
            }
            
        except Exception as e:
            # Emergency cleanup
            self.emergency_cleanup(pre_test_state)
            raise e
    
    def test_complete_site_failure(self, parameters):
        """Test complete primary site failure scenario"""
        
        # Simulate site failure by blocking network access
        self.simulate_network_failure(parameters.get('duration', 300))
        
        # Measure failover time
        failover_start = time.time()
        
        # Trigger automated failover
        failover_success = self.failover_manager.initiate_failover('test')
        
        failover_duration = time.time() - failover_start
        
        # Test DR environment functionality
        dr_functionality_tests = [
            self.test_job_submission(),
            self.test_data_access(),
            self.test_user_authentication(),
            self.test_application_availability()
        ]
        
        functionality_results = {}
        for test in dr_functionality_tests:
            test_name = test.__name__
            try:
                functionality_results[test_name] = test()
            except Exception as e:
                functionality_results[test_name] = {'success': False, 'error': str(e)}
        
        return {
            'success': failover_success and all(
                result.get('success', False) 
                for result in functionality_results.values()
            ),
            'failover_duration': failover_duration,
            'rto_met': failover_duration < parameters.get('rto_target', 3600),
            'functionality_tests': functionality_results
        }
    
    def test_job_submission(self):
        """Test job submission in DR environment"""
        
        test_job_script = """#!/bin/bash
#SBATCH --job-name=dr-test
#SBATCH --nodes=1
#SBATCH --time=00:05:00
#SBATCH --output=dr-test-%j.out

echo "DR test job started at $(date)"
sleep 60
echo "DR test job completed at $(date)"
"""
        
        # Submit test job
        with open('/tmp/dr_test_job.sh', 'w') as f:
            f.write(test_job_script)
        
        result = subprocess.run([
            'sbatch', '/tmp/dr_test_job.sh'
        ], capture_output=True, text=True)
        
        if result.returncode == 0:
            # Extract job ID
            job_id = result.stdout.strip().split()[-1]
            
            # Monitor job completion
            job_completed = self.wait_for_job_completion(job_id, timeout=600)
            
            return {
                'success': job_completed,
                'job_id': job_id,
                'submission_time': time.time()
            }
        else:
            return {
                'success': False,
                'error': result.stderr
            }
    
    def generate_dr_test_report(self, test_results):
        """Generate comprehensive DR test report"""
        
        report = {
            'test_date': datetime.now().isoformat(),
            'test_summary': {
                'total_scenarios': len(test_results),
                'passed_scenarios': sum(1 for r in test_results if r['success']),
                'failed_scenarios': sum(1 for r in test_results if not r['success'])
            },
            'detailed_results': test_results,
            'recommendations': self.generate_recommendations(test_results),
            'next_test_date': (datetime.now() + timedelta(days=90)).isoformat()
        }
        
        return report
    
    def generate_recommendations(self, test_results):
        """Generate recommendations based on test results"""
        recommendations = []
        
        for result in test_results:
            if not result['success']:
                scenario = result['scenario']
                
                if scenario == 'complete_site_failure':
                    if not result['test_results'].get('rto_met', True):
                        recommendations.append(
                            "Failover time exceeds RTO target. Consider pre-warming DR resources."
                        )
                
                if 'functionality_tests' in result['test_results']:
                    failed_tests = [
                        test for test, res in result['test_results']['functionality_tests'].items()
                        if not res.get('success', False)
                    ]
                    
                    if failed_tests:
                        recommendations.append(
                            f"DR functionality issues in: {', '.join(failed_tests)}"
                        )
        
        return recommendations
```

## Week 12 Deliverables

### Disaster Recovery Implementation
1. **DR Strategy Document** (3-4 pages)
   - Recovery objectives and strategies
   - Multi-site backup architecture
   - Failover procedures and automation

2. **Business Continuity Plan** (2-3 pages)
   - Critical process identification
   - Recovery prioritization matrix
   - Communication and escalation procedures

3. **DR Test Results Report** (2 pages)
   - Test scenario execution results
   - Performance against RTO/RPO targets
   - Recommendations for improvement

### Final Capstone Project
**Hybrid HPC-Cloud Implementation Portfolio**
- Complete hybrid architecture design
- Implementation documentation and code
- Performance analysis and optimization results
- Cost-benefit analysis
- Lessons learned and best practices

---

*Disaster recovery is insurance for your computational infrastructure. Test it regularly, or it won't work when you need it most.*

## Course Conclusion

Congratulations on completing the HPC ↔ Cloud Hybrid Integration Study Plan! You now have the knowledge and practical experience to:

- Design and implement hybrid architectures that leverage the best of both HPC and cloud computing
- Optimize performance, cost, and security across hybrid environments
- Manage complex workflows and data pipelines spanning multiple platforms
- Implement comprehensive monitoring, cost control, and disaster recovery systems

The future of computational infrastructure is hybrid. You're now prepared to lead that transformation in your organization.