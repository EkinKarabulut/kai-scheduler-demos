# CPU Scheduling Example - Step by Step Guide

This guide demonstrates how the KAI Scheduler handles complex CPU allocation scenarios, focusing on priority-based scheduling, workload consolidation, and resource management. It includes a distributed training workload to demonstrate gang scheduling capabilities.

## Prerequisites

1. A Kubernetes cluster with KAI Scheduler installed (version 1.0 or later)
2. 4 nodes with 8 CPUs each (32 total CPUs)
3. `kubectl` configured to access the cluster
4. Kubeflow Training Operator is installed (required for distributed training)
5. Namespaces created for each project:
   ```bash
   kubectl create namespace runai-project-a
   kubectl create namespace runai-project-c
   kubectl create namespace runai-project-d
   ```

## Important Notes

1. **Namespace and Queue Association**
   - Each project must have its own namespace
   - The namespace name must be prefixed with 'runai-' followed by the queue name
   - The `runai/queue` label tells the scheduler which queue the pod belongs to
   - Pods must be created in the correct namespace

2. **Pod Requirements**
   - Every pod must have the `runai/queue` label matching its namespace
   - The scheduler name must be set to `kai-scheduler`
   - Resource requests and limits must be specified
   - For demo purposes, we use `sleep 3600` to simulate long-running workloads
   - Job priorities are specified using Kubernetes PriorityClass:
     - Use `priorityClassName: inference` (value 125) for preemptible inference jobs
     - Use `priorityClassName: build` (value 100) for non-preemptible interactive jobs
     - Use `priorityClassName: build-preemptible` (value 75) for preemptible interactive jobs
     - Use `priorityClassName: train` (value 50) for training jobs (default)
   - Users can create custom PriorityClasses with their own values
   - Any PriorityClass with value >= 100 will be treated as non-preemptible
   - The scheduler supports any PriorityClass deployed in the cluster

3. **Resource Quotas**
   - Each project has a quota of 8 CPUs (25% of cluster)
   - Projects can borrow resources from other projects when needed
   - Over-quota weights determine resource distribution when borrowing
   - The scheduler automatically manages resource allocation within quotas

4. **Gang Scheduling**
   - Distributed training workloads require all pods to be scheduled together
   - The scheduler ensures all required resources are available before scheduling
   - If resources become unavailable, all pods are preempted together
   - This ensures all workers can communicate effectively

5. **Job Priorities and Preemption**
   - Training jobs have priority 50 (lowest)
   - Preemptible interactive jobs have priority 75
   - Non-preemptible interactive jobs have priority 100
   - Inference jobs have priority 125
   - Higher priority jobs can preempt lower priority jobs
   - Jobs with priority >= 100 cannot be preempted
   - Jobs in the same queue can preempt each other based on priority
   - Jobs cannot preempt jobs in other queues

## Step 1: Set Up Queue Hierarchy

First, create the queue hierarchy with the following command:

```bash
kubectl apply -f 1-queues.yaml
```

This creates:
- Department 1 (Higher Priority - 200)
  - Project A (Priority: 150)
  - Project B (Priority: 100)
- Department 2 (Lower Priority - 100)
  - Project C (Priority: 150)
  - Project D (Priority: 100)

Priority numbers range from 0 to 1000, with higher numbers indicating higher priority. The scheduler uses these priorities to determine which workloads get resources first.

Verify the queues:
```bash
kubectl get queues
```

Expected output:
```
NAME         PRIORITY   WEIGHT   QUOTA
project-a    150        1.0      8
project-b    100        1.0      8
project-c    150        1.0      8
project-d    100        1.0      8
```

## Step 2: Create Initial Cluster State

Create the initial workloads to establish the cluster state:

```bash
kubectl apply -f 2-initial-state.yaml
```

This creates:
- Training Job 1 (Project A): 4 CPUs on Node 1
- Training Job 2 (Project C): 2 CPUs on Node 1
- Distributed Training Job (Project C): 6 CPUs total (2 CPUs per pod) on Node 2
- Interactive Job (Project D): 5 CPUs on Node 3

Note: The distributed training job is configured to run for 900 epochs to demonstrate long-running workload behavior.

Monitor the pod status:
```bash
kubectl get pods -A -w
```

Expected Log Output:
```
Queue priority: project-a = 150
Queue priority: project-c = 150
Queue priority: project-d = 100

Allocating 4 CPUs to training-job-1 on node node-1
Allocating 2 CPUs to training-job-2 on node node-1
Allocating 6 CPUs to distributed-training (3 pods) on node node-2
Allocating 5 CPUs to interactive-job on node node-3
```

## Step 3: Submit New Workloads

Submit the new workloads that will demonstrate the scheduler's decision-making process:

```bash
kubectl apply -f 3-new-workloads.yaml
```

This creates:
- Training Job A (Project C): Requires 4 CPUs
- Training Job B (Project A): Requires 3 CPUs

Training Job B is scheduled first because Project A has higher priority (150) than Project C (150), and it was created first.

Expected Log Output:
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

## Step 4: Monitor Scheduling Process

Watch the scheduling process in real-time:

```bash
# Watch pod status changes
kubectl get pods -A -w

# Check node resource allocation
kubectl describe nodes

# Monitor queue resource usage
kubectl get queues
```

These commands help you understand:
- `kubectl get pods -A -w`: Shows pod status changes in real-time
- `kubectl describe nodes`: Shows detailed resource allocation on each node
- `kubectl get queues`: Shows current resource usage per queue

Expected Log Output:
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

## Expected Outcomes

1. **Initial State**
   - Node 1: Training Job 1 (4 CPUs) + Training Job 2 (2 CPUs)
   - Node 2: Distributed Training Job (6 CPUs total, 3 pods)
   - Node 3: Interactive Job (5 CPUs)
   - Node 4: Available (8 CPUs)

2. **After New Workload Submission**
   - Training Job B (3 CPUs) will be scheduled on Node 3
   - Training Job A (4 CPUs) will remain pending initially

3. **After Consolidation**
   - Training Job 2 will be moved from Node 1 to Node 2
   - Training Job A will be scheduled on Node 1
   - Distributed Training Job remains on Node 2 (gang scheduling)
   - Final state:
     - Node 1: Training Job 1 (4 CPUs) + Training Job A (4 CPUs)
     - Node 2: Distributed Training Job (6 CPUs) + Training Job 2 (2 CPUs)
     - Node 3: Interactive Job (5 CPUs) + Training Job B (3 CPUs)
     - Node 4: Available (8 CPUs)

## Key Observations

1. **Priority-based Scheduling**
   - Higher priority jobs are scheduled first (Training Job B: high-priority, Training Job A: medium-priority)
   - Queue priorities influence scheduling order
   - Within same priority, creation time determines order

2. **Resource Consolidation**
   - Workloads are moved to create contiguous CPU blocks
   - Maintains workload requirements while optimizing resource usage
   - Happens automatically when needed

3. **Fair Share Management**
   - All workloads stay within their queue quotas
   - Resource allocation respects priority levels
   - Over-quota weights influence resource distribution

4. **Gang Scheduling**
   - Distributed training pods are scheduled together
   - All pods remain on the same node
   - If preemption is needed, all pods are preempted together
   - Demonstrates the scheduler's ability to handle distributed workloads

5. **Node Placement**
   - The scheduler uses the same binpacking strategy for CPUs as it does for GPUs
   - Workloads are consolidated to minimize resource fragmentation
   - Node selection considers both available resources and workload requirements

## Troubleshooting

If pods are not being scheduled, check:
1. Namespace exists and matches queue name
2. `runai/queue` label is correctly set
3. `schedulerName` is set to `kai-scheduler`
4. Resource requests are within queue quotas
5. Node resources are available
6. For distributed training:
   - All pods have the same resource requirements
   - Gang scheduling is enabled
   - Sufficient resources are available on a single node

## Cleanup

To clean up all resources:

```bash
kubectl delete -f 3-new-workloads.yaml
kubectl delete -f 2-initial-state.yaml
kubectl delete -f 1-queues.yaml
```

## Comparison with GPU Scheduling

The CPU scheduling behavior mirrors the GPU scheduling behavior in several ways:

1. **Priority-based Allocation**
   - Both CPU and GPU workloads follow the same priority rules
   - Higher priority queues get resources first

2. **Resource Consolidation**
   - Both CPU and GPU workloads are consolidated to optimize resource usage
   - The scheduler moves workloads to create contiguous resource blocks

3. **Fair Share**
   - Both CPU and GPU resources are managed through the same quota system
   - Over-quota weights work the same way for both resource types

4. **Node Selection**
   - The scheduler uses similar strategies for both CPU and GPU placement
   - Both follow the binpacking strategy to minimize resource fragmentation

The main difference is that CPU workloads don't require contiguous CPU cores in the same way that GPU workloads require contiguous GPUs, but the scheduling principles remain the same. 