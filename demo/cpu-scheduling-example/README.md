# Scheduling GPU Workloads with KAI Scheduler

## Prerequisites

- Kubernetes cluster with KAI Scheduler installed (version 1.0 or later)
- 4 nodes with 8 CPUs each (32 total CPUs)
- Kubeflow training operator installed (required for distributed training)
- Namespaces created for each project:
   ```bash
   kubectl create namespace runai-project-a
   kubectl create namespace runai-project-c
   kubectl create namespace runai-project-d
   ```

## Queue Configuration

### Department 1 (Higher Priority - 200)
- Project A (Priority: 150)
  - 8 CPU quota (25% of cluster)
- Project B (Priority: 100)
  - 8 CPU quota (25% of cluster)

### Department 2 (Lower Priority - 100)
- Project C (Priority: 150)
  - 8 CPU quota (25% of cluster)
- Project D (Priority: 100)
  - 8 CPU quota (25% of cluster)

## Initial Cluster State

### Node 1 (8 CPUs)
- Training Job 1 (Project A): 4 CPUs
- Training Job 2 (Project C): 2 CPUs
- Available: 2 CPUs

### Node 2 (8 CPUs)
- Training Job 3 (Project C): 6 CPUs
- Available: 2 CPUs

### Node 3 (8 CPUs)
- Interactive Job (Project D): 5 CPUs
- Available: 3 CPUs

### Node 4 (8 CPUs)
- Available: 8 CPUs

## Demo Flow

### Step 1: Setup the Initial Cluster State
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
Allocating 4 CPUs to training-job-1 on node node-1
Allocating 2 CPUs to training-job-2 on node node-1
Allocating 6 CPUs to distributed-training (3 pods) on node node-2
Allocating 5 CPUs to interactive-job on node node-3
```

### Step 2: Submit New Workloads
```bash
kubectl apply -f 3-new-workloads.yaml
```

Expected Output:
```
New workload submitted: training-job-b (Project A, Priority 150)
New workload submitted: training-job-a (Project C, Priority 150)

Attempting to schedule training-job-b (3 CPUs)
Found suitable node: node-3 (3 CPUs available)
Allocating 3 CPUs to training-job-b on node node-3

Attempting to schedule training-job-a (4 CPUs)
Insufficient contiguous CPUs on any node
Initiating resource consolidation
```

### Step 3: Monitor Scheduling
```bash
kubectl get pods -w
```

Expected Output:
```
Starting allocation phase
Training Job B (3 CPUs) scheduled on Node 3
Training Job A (4 CPUs) pending - insufficient resources

Starting consolidation phase
Moving training-job-2 (2 CPUs) from node-1 to node-2
Creating 4 contiguous CPUs on node-1
Scheduling training-job-a (4 CPUs) on node-1

Gang scheduling check for distributed-training:
- All 3 pods must be scheduled together
- Current placement: node-2 (6 CPUs total)
- No preemption needed as resources are sufficient
```

## Scheduling Actions after New Workload Submission

1. **Allocation Phase**
   - Training Job B (3 CPUs) is scheduled on Node 3
   - Training Job A remains pending

2. **Consolidation Phase**
   - Training Job 2 is relocated to Node 2
   - Training Job A (4 CPUs) is scheduled on Node 1

3. **Final State**
   - Node 1: Training Job 1 (4 CPUs) + Training Job A (4 CPUs)
   - Node 2: Training Job 3 (6 CPUs) + Training Job 2 (2 CPUs)
   - Node 3: Interactive Job (5 CPUs) + Training Job B (3 CPUs)
   - Node 4: Available (8 CPUs)

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
