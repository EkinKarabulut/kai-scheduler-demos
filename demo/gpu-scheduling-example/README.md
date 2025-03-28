# Scheduling GPU Workloads with KAI Scheduler

## Table of Contents
- [Prerequisites](#prerequisites)
- [Queue Configuration](#queue-configuration)
- [Initial Cluster State](#initial-cluster-state)
- [Scheduling Actions after New Workload Submission](#scheduling-actions-after-new-workload-submission)
- [Demo Flow](#demo-flow)
    - [Step 1: Setup the Initial Cluster State](#step-1-setup-the-initial-cluster-state)
    - [Step 2: Submit New Workloads](#step-2-submit-new-workloads)
    - [Step 3: Monitor Scheduling](#step-2-monitor-scheduling)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Kubernetes cluster with KAI Scheduler installed (version 1.0 or later)
- 4 nodes with 8 GPUs each (32 total GPUs)
- NVIDIA device plugin installed and configured
- Kubeflow training operator installed (required for distributed training)

## Queue Configuration

### Department 1 (Higher Priority - 200)
- Project A (Priority: 150)
  - 8 GPU quota (25% of cluster)
- Project B (Priority: 100)
  - 8 GPU quota (25% of cluster)

### Department 2 (Lower Priority - 100)
- Project C (Priority: 150)
  - 8 GPU quota (25% of cluster)
- Project D (Priority: 100)
  - 8 GPU quota (25% of cluster)

## Initial Cluster State

- Node 1: Training Job 1 (4 GPUs) + Training Job 2 (2 GPUs)
- Node 2: Training Job 3 (6 GPUs) 
- Node 3: Interactive Job (5 GPUs) 
- Node 4: No Workloads Running

## Scheduling Actions after New Workload Submission

1. **Allocation Phase**
   - Training Job B (3 GPUs) is scheduled on Node 3
   - Training Job A remains pending

2. **Consolidation Phase**
   - Training Job 2 is relocated to Node 2
   - Training Job A (4 GPUs) is scheduled on Node 1

3. **After Consolidation**
   - Training Job 2 will be moved from Node 1 to Node 2
   - Training Job A will be scheduled on Node 1
   - Final state:
     - Node 1: Training Job 1 (4 GPUs) + Training Job A (4 GPUs)
     - Node 2: Training Job 3 (6 GPUs) + Training Job 2 (2 GPUs)
     - Node 3: Interactive Job (5 GPUs) + Training Job B (3 GPUs)
     - Node 4: Available (8 GPUs)

## Demo Flow

### Step 1: Setup Initial State
```bash
kubectl apply -f 1-queues.yaml
kubectl apply -f 2-initial-state.yaml
```

Monitor the pod status:
```bash
kubectl get pods -A -w
```

Expected Output:
```
Allocating 4 GPUs to training-job-1 on node node-1
Allocating 2 GPUs to training-job-2 on node node-1
Allocating 6 GPUs to training-job-3 on node node-2
Allocating 5 GPUs to interactive-job on node node-3
```

### Step 2: Submit New Workloads
```bash
kubectl apply -f 3-new-workloads.yaml
```

Expected Output:
```
New workload submitted: training-job-b (Project A, Priority 150)
New workload submitted: training-job-a (Project C, Priority 150)

Attempting to schedule training-job-b (3 GPUs)
Found suitable node: node-3 (3 GPUs available)
Allocating 3 GPUs to training-job-b on node node-3

Attempting to schedule training-job-a (4 GPUs)
Insufficient contiguous GPUs on any node
Initiating resource consolidation
```

### Step 3: Monitor Scheduling
```bash
kubectl get pods -w
```

Expected Output:
```
Starting allocation phase
Training Job B (3 GPUs) scheduled on Node 3
Training Job A (4 GPUs) pending - insufficient resources

Starting consolidation phase
Moving training-job-2 (2 GPUs) from node-1 to node-2
Creating 4 contiguous GPUs on node-1
Scheduling training-job-a (4 GPUs) on node-1
```


## Cleanup
```bash
kubectl delete -f 3-new-workloads.yaml
kubectl delete -f 2-initial-state.yaml
kubectl delete -f 1-queues.yaml
``` 
## Troubleshooting

If pods are not being scheduled, check:
1. Namespace exists and matches queue name
2. `runai/queue` label is correctly set
3. `schedulerName` is set to `kai-scheduler`
4. Resource requests are within queue quotas
5. Node resources are available
6. For distributed training make sure that sufficient resources are available on the nodes
