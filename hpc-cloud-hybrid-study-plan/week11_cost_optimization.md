# Week 11: Cost Optimization & Resource Management

## Learning Objectives
- Implement cost optimization strategies for hybrid HPC-cloud environments
- Design intelligent resource allocation and scheduling policies
- Configure automated cost controls and budget management
- Optimize workload placement for cost-performance balance
- Implement chargeback and cost allocation models

## Overview
Cost optimization in hybrid environments requires balancing performance requirements with economic efficiency. This week covers cost modeling, optimization algorithms, and automated resource management strategies.

## Cost Optimization Strategies

### Cost Models
1. **Total Cost of Ownership (TCO)**: Complete cost analysis including hidden costs
2. **Pay-per-Use**: Dynamic pricing based on actual resource consumption
3. **Reserved Capacity**: Long-term commitments for predictable workloads
4. **Spot/Preemptible**: Discounted resources for fault-tolerant workloads

### Optimization Dimensions
- **Temporal**: When to run workloads (off-peak pricing)
- **Spatial**: Where to run workloads (geographic cost differences)
- **Resource Type**: Which resources to use (CPU vs GPU, instance types)
- **Commitment Level**: Reserved vs on-demand pricing

## Hands-On Lab: Cost Optimization Implementation

### Exercise 1: Cost-Aware Workload Scheduler (120 minutes)

#### Multi-Objective Optimization Engine
```python
import numpy as np
from scipy.optimize import minimize
from dataclasses import dataclass
from typing import Dict, List, Tuple
import boto3
from datetime import datetime, timedelta

@dataclass
class WorkloadRequirement:
    job_id: str
    cpu_cores: int
    memory_gb: int
    gpu_count: int
    storage_gb: int
    max_runtime_hours: int
    deadline: datetime
    priority: float
    fault_tolerance: bool  # Can use spot instances

@dataclass
class ResourceOption:
    provider: str  # 'hpc', 'aws', 'azure', 'gcp'
    instance_type: str
    cpu_cores: int
    memory_gb: int
    gpu_count: int
    cost_per_hour: float
    availability: float  # 0-1, probability of getting resource
    performance_factor: float  # relative performance multiplier
    spot_available: bool
    spot_discount: float  # discount percentage for spot instances

class CostOptimizer:
    def __init__(self):
        self.resource_options = self.load_resource_catalog()
        self.pricing_history = {}
        self.performance_models = {}
        
    def optimize_workload_placement(self, workloads: List[WorkloadRequirement], 
                                  budget_constraint: float = None) -> Dict:
        """Optimize workload placement across hybrid resources"""
        
        # Generate placement options for each workload
        placement_options = {}
        for workload in workloads:
            placement_options[workload.job_id] = self.generate_placement_options(workload)
        
        # Formulate optimization problem
        optimization_result = self.solve_placement_optimization(
            workloads, placement_options, budget_constraint
        )
        
        return optimization_result
    
    def generate_placement_options(self, workload: WorkloadRequirement) -> List[Dict]:
        """Generate viable placement options for a workload"""
        options = []
        
        for resource in self.resource_options:
            if self.resource_meets_requirements(resource, workload):
                # Calculate cost and performance for this option
                base_cost = resource.cost_per_hour * workload.max_runtime_hours
                
                # Apply spot discount if workload is fault-tolerant
                if workload.fault_tolerance and resource.spot_available:
                    cost = base_cost * (1 - resource.spot_discount)
                    availability = resource.availability * 0.7  # Spot has lower availability
                else:
                    cost = base_cost
                    availability = resource.availability
                
                # Estimate actual runtime based on performance factor
                estimated_runtime = workload.max_runtime_hours / resource.performance_factor
                actual_cost = cost * (estimated_runtime / workload.max_runtime_hours)
                
                # Check if can meet deadline
                completion_time = datetime.now() + timedelta(hours=estimated_runtime)
                if completion_time <= workload.deadline:
                    options.append({
                        'resource': resource,
                        'cost': actual_cost,
                        'estimated_runtime': estimated_runtime,
                        'availability': availability,
                        'performance_score': resource.performance_factor,
                        'meets_deadline': True
                    })
        
        # Sort by cost-performance ratio
        options.sort(key=lambda x: x['cost'] / x['performance_score'])
        return options
    
    def solve_placement_optimization(self, workloads: List[WorkloadRequirement],
                                   placement_options: Dict,
                                   budget_constraint: float) -> Dict:
        """Solve the workload placement optimization problem"""
        
        # Define objective function (minimize cost while maximizing performance)
        def objective_function(x):
            total_cost = 0
            total_performance = 0
            
            for i, workload in enumerate(workloads):
                options = placement_options[workload.job_id]
                
                # x[i] represents the selected option index for workload i
                selected_option_idx = int(x[i])
                if selected_option_idx < len(options):
                    option = options[selected_option_idx]
                    total_cost += option['cost']
                    total_performance += option['performance_score'] * workload.priority
            
            # Multi-objective: minimize cost, maximize performance
            # Normalize and combine objectives
            cost_weight = 0.6
            performance_weight = 0.4
            
            normalized_cost = total_cost / 10000  # Normalize to reasonable scale
            normalized_performance = total_performance / len(workloads)
            
            return cost_weight * normalized_cost - performance_weight * normalized_performance
        
        # Define constraints
        constraints = []
        
        # Budget constraint
        if budget_constraint:
            def budget_constraint_func(x):
                total_cost = 0
                for i, workload in enumerate(workloads):
                    options = placement_options[workload.job_id]
                    selected_option_idx = int(x[i])
                    if selected_option_idx < len(options):
                        total_cost += options[selected_option_idx]['cost']
                return budget_constraint - total_cost
            
            constraints.append({'type': 'ineq', 'fun': budget_constraint_func})
        
        # Bounds: each workload can select from available options
        bounds = []
        for workload in workloads:
            options_count = len(placement_options[workload.job_id])
            bounds.append((0, max(0, options_count - 1)))
        
        # Initial guess: select cheapest option for each workload
        x0 = [0] * len(workloads)
        
        # Solve optimization problem
        result = minimize(
            objective_function,
            x0,
            method='SLSQP',
            bounds=bounds,
            constraints=constraints
        )
        
        # Extract solution
        solution = {}
        total_cost = 0
        
        for i, workload in enumerate(workloads):
            selected_option_idx = int(result.x[i])
            options = placement_options[workload.job_id]
            
            if selected_option_idx < len(options):
                selected_option = options[selected_option_idx]
                solution[workload.job_id] = selected_option
                total_cost += selected_option['cost']
        
        return {
            'placements': solution,
            'total_cost': total_cost,
            'optimization_success': result.success,
            'optimization_message': result.message
        }
```

### Exercise 2: Automated Cost Controls (90 minutes)

#### Budget Management System
```python
import boto3
from datetime import datetime, timedelta
import json

class HybridBudgetManager:
    def __init__(self):
        self.budgets_client = boto3.client('budgets')
        self.ce_client = boto3.client('ce')  # Cost Explorer
        self.ec2_client = boto3.client('ec2')
        self.budget_alerts = []
        
    def create_cost_budget(self, budget_name: str, budget_amount: float,
                          time_period: str = 'MONTHLY') -> str:
        """Create a cost budget with automated controls"""
        
        budget_definition = {
            'BudgetName': budget_name,
            'BudgetLimit': {
                'Amount': str(budget_amount),
                'Unit': 'USD'
            },
            'TimeUnit': time_period,
            'TimePeriod': {
                'Start': datetime.now().replace(day=1),
                'End': (datetime.now().replace(day=1) + timedelta(days=32)).replace(day=1)
            },
            'CostFilters': {
                'Service': ['Amazon Elastic Compute Cloud - Compute']
            },
            'BudgetType': 'COST'
        }
        
        # Create budget alerts
        notifications = [
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80.0,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'admin@organization.edu'
                    }
                ]
            },
            {
                'Notification': {
                    'NotificationType': 'FORECASTED',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 100.0,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'admin@organization.edu'
                    }
                ]
            }
        ]
        
        response = self.budgets_client.create_budget(
            AccountId='123456789012',
            Budget=budget_definition,
            NotificationsWithSubscribers=notifications
        )
        
        return budget_definition['BudgetName']
    
    def implement_cost_controls(self, budget_name: str):
        """Implement automated cost control actions"""
        
        # Get current budget status
        budget_status = self.get_budget_status(budget_name)
        
        if budget_status['utilization_percentage'] > 90:
            # Emergency cost controls
            self.emergency_cost_reduction()
        elif budget_status['utilization_percentage'] > 75:
            # Proactive cost optimization
            self.optimize_running_resources()
    
    def emergency_cost_reduction(self):
        """Emergency cost reduction measures"""
        actions_taken = []
        
        # 1. Stop non-critical spot instances
        spot_instances = self.get_spot_instances()
        for instance in spot_instances:
            if instance['Tags'].get('Priority', 'normal') != 'critical':
                self.ec2_client.terminate_instances(InstanceIds=[instance['InstanceId']])
                actions_taken.append(f"Terminated spot instance {instance['InstanceId']}")
        
        # 2. Scale down auto-scaling groups
        autoscaling_groups = self.get_autoscaling_groups()
        for asg in autoscaling_groups:
            if asg['Tags'].get('CostOptimization', 'enabled') == 'enabled':
                self.scale_down_asg(asg['AutoScalingGroupName'], 0.5)
                actions_taken.append(f"Scaled down ASG {asg['AutoScalingGroupName']}")
        
        # 3. Pause non-critical workflows
        self.pause_non_critical_workflows()
        actions_taken.append("Paused non-critical workflows")
        
        return actions_taken
    
    def optimize_running_resources(self):
        """Optimize currently running resources for cost"""
        optimizations = []
        
        # 1. Right-size over-provisioned instances
        underutilized_instances = self.find_underutilized_instances()
        for instance in underutilized_instances:
            smaller_instance_type = self.recommend_smaller_instance_type(instance)
            if smaller_instance_type:
                self.resize_instance(instance['InstanceId'], smaller_instance_type)
                optimizations.append(f"Resized {instance['InstanceId']} to {smaller_instance_type}")
        
        # 2. Convert on-demand to spot where possible
        convertible_instances = self.find_convertible_instances()
        for instance in convertible_instances:
            self.convert_to_spot_instance(instance)
            optimizations.append(f"Converted {instance['InstanceId']} to spot")
        
        # 3. Optimize storage usage
        unused_volumes = self.find_unused_ebs_volumes()
        for volume in unused_volumes:
            self.delete_unused_volume(volume['VolumeId'])
            optimizations.append(f"Deleted unused volume {volume['VolumeId']}")
        
        return optimizations
```

### Exercise 3: Chargeback and Cost Allocation (75 minutes)

#### Cost Allocation Engine
```python
class CostAllocationEngine:
    def __init__(self):
        self.cost_data = {}
        self.allocation_rules = {}
        
    def define_allocation_rules(self, rules: Dict):
        """Define cost allocation rules"""
        self.allocation_rules = {
            'departments': rules.get('departments', {}),
            'projects': rules.get('projects', {}),
            'users': rules.get('users', {}),
            'resource_types': rules.get('resource_types', {})
        }
    
    def collect_usage_data(self, time_period: Tuple[datetime, datetime]):
        """Collect resource usage data for allocation"""
        start_date, end_date = time_period
        
        usage_data = {
            'compute': self.get_compute_usage(start_date, end_date),
            'storage': self.get_storage_usage(start_date, end_date),
            'network': self.get_network_usage(start_date, end_date),
            'cloud_services': self.get_cloud_service_usage(start_date, end_date)
        }
        
        return usage_data
    
    def allocate_costs(self, usage_data: Dict, total_costs: Dict) -> Dict:
        """Allocate costs based on usage and rules"""
        allocation_result = {
            'departments': {},
            'projects': {},
            'users': {},
            'summary': {}
        }
        
        # Allocate compute costs
        compute_allocation = self.allocate_compute_costs(
            usage_data['compute'], 
            total_costs['compute']
        )
        
        # Allocate storage costs
        storage_allocation = self.allocate_storage_costs(
            usage_data['storage'],
            total_costs['storage']
        )
        
        # Allocate network costs
        network_allocation = self.allocate_network_costs(
            usage_data['network'],
            total_costs['network']
        )
        
        # Combine allocations
        for category in ['departments', 'projects', 'users']:
            allocation_result[category] = self.combine_allocations([
                compute_allocation.get(category, {}),
                storage_allocation.get(category, {}),
                network_allocation.get(category, {})
            ])
        
        # Generate summary
        allocation_result['summary'] = self.generate_allocation_summary(allocation_result)
        
        return allocation_result
    
    def generate_chargeback_reports(self, allocation_result: Dict) -> Dict:
        """Generate chargeback reports for different entities"""
        reports = {}
        
        # Department reports
        for dept, costs in allocation_result['departments'].items():
            reports[f"department_{dept}"] = {
                'entity': dept,
                'entity_type': 'department',
                'total_cost': sum(costs.values()),
                'cost_breakdown': costs,
                'cost_per_category': self.calculate_cost_per_category(costs),
                'recommendations': self.generate_cost_recommendations(dept, costs)
            }
        
        # Project reports
        for project, costs in allocation_result['projects'].items():
            reports[f"project_{project}"] = {
                'entity': project,
                'entity_type': 'project',
                'total_cost': sum(costs.values()),
                'cost_breakdown': costs,
                'budget_status': self.check_project_budget(project, sum(costs.values())),
                'recommendations': self.generate_cost_recommendations(project, costs)
            }
        
        return reports
    
    def generate_cost_recommendations(self, entity: str, costs: Dict) -> List[str]:
        """Generate cost optimization recommendations"""
        recommendations = []
        
        total_cost = sum(costs.values())
        
        # High compute costs
        if costs.get('compute', 0) > total_cost * 0.6:
            recommendations.append(
                "Consider using spot instances for fault-tolerant workloads"
            )
            recommendations.append(
                "Review instance sizing - may be over-provisioned"
            )
        
        # High storage costs
        if costs.get('storage', 0) > total_cost * 0.3:
            recommendations.append(
                "Implement data lifecycle policies for archival"
            )
            recommendations.append(
                "Review storage utilization and delete unused data"
            )
        
        # High network costs
        if costs.get('network', 0) > total_cost * 0.2:
            recommendations.append(
                "Optimize data transfer patterns between environments"
            )
            recommendations.append(
                "Consider data compression for large transfers"
            )
        
        return recommendations
```

## Cost Optimization Algorithms

### Machine Learning-Based Cost Prediction
```python
import pandas as pd
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
import joblib

class CostPredictionModel:
    def __init__(self):
        self.model = RandomForestRegressor(n_estimators=100, random_state=42)
        self.feature_columns = [
            'cpu_hours', 'memory_gb_hours', 'gpu_hours', 'storage_gb_hours',
            'network_gb', 'instance_type_encoded', 'time_of_day', 'day_of_week'
        ]
        
    def train_model(self, historical_data: pd.DataFrame):
        """Train cost prediction model on historical data"""
        
        # Feature engineering
        features = self.engineer_features(historical_data)
        
        # Prepare training data
        X = features[self.feature_columns]
        y = historical_data['total_cost']
        
        # Split data
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42
        )
        
        # Train model
        self.model.fit(X_train, y_train)
        
        # Evaluate model
        train_score = self.model.score(X_train, y_train)
        test_score = self.model.score(X_test, y_test)
        
        return {
            'train_score': train_score,
            'test_score': test_score,
            'feature_importance': dict(zip(
                self.feature_columns, 
                self.model.feature_importances_
            ))
        }
    
    def predict_workload_cost(self, workload_specs: Dict) -> Dict:
        """Predict cost for a given workload specification"""
        
        # Convert workload specs to features
        features = self.workload_to_features(workload_specs)
        
        # Make prediction
        predicted_cost = self.model.predict([features])[0]
        
        # Get prediction confidence intervals
        predictions = []
        for estimator in self.model.estimators_:
            predictions.append(estimator.predict([features])[0])
        
        confidence_interval = {
            'lower': np.percentile(predictions, 5),
            'upper': np.percentile(predictions, 95)
        }
        
        return {
            'predicted_cost': predicted_cost,
            'confidence_interval': confidence_interval,
            'cost_breakdown': self.estimate_cost_breakdown(workload_specs)
        }
```

## Week 11 Deliverables

### Cost Optimization Implementation
1. **Cost Optimization Strategy** (2-3 pages)
   - Multi-objective optimization approach
   - Automated cost control mechanisms
   - Budget management and alerting

2. **Chargeback System Design** (2 pages)
   - Cost allocation methodology
   - Reporting and analytics framework
   - Integration with existing systems

3. **Cost Analysis Report** (1-2 pages)
   - Current cost baseline and trends
   - Optimization opportunities identified
   - ROI projections for implemented changes

---

*Cost optimization is an ongoing process, not a one-time activity. Build automation and continuous improvement into your approach.*