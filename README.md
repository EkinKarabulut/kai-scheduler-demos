[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
# Kubernetes AI (KAI) Scheduler
The Kubernetes AI Scheduler is a robust, efficient, and scalable [Kubernetes scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) that optimizes GPU resource allocation for AI and machine learning workloads.

Designed to manage large-scale GPU clusters, including thousands of nodes, and high-throughput of workloads, makes the KAI Scheduler ideal for extensive and demanding environments.
The Kubernetes AI Scheduler allows administrators of Kubernetes clusters to dynamically allocate GPU resources to workloads. 

KAI Scheduler supports the entire AI lifecycle, from small, interactive jobs that require minimal resources to large training and inference, all within the same cluster. 
It ensures optimal resource allocation while maintaining resource fairness between the different consumers.
It can run alongside other schedulers installed on the cluster.

## Prerequisites
Before installing KAI Scheduler, ensure you have:

- A running Kubernetes cluster
- [Helm](https://helm.sh/docs/intro/install) CLI installed
- [NVIDIA GPU-Operator](https://github.com/NVIDIA/gpu-operator) installed in order to schedule workloads that request GPU resources

## Installation
KAI Scheduler will be installed in `kai-scheduler` namespace. When submitting workloads make sure to use a dedicated namespace.

### Installation Methods
KAI Scheduler can be installed:

- **From Production (Recommended)**
- **From Source (Build it Yourself)**

#### Install from Production
```sh
helm repo add nvidia-k8s https://helm.ngc.nvidia.com/nvidia/k8s
helm repo update
helm upgrade -i kai-scheduler nvidia-k8s/kai-scheduler -n kai-scheduler --create-namespace --set "global.registry=nvcr.io/nvidia/k8s"
```

#### Build from Source
Follow the instructions [here](docs/developer/building-from-source.md)

## Quick Start
To start scheduling workloads with KAI Scheduler, follow these steps:

### Configuration and Reference

#### 1. Queue Setup
A queue is an object that represents a job queue in the cluster. Queues are essential scheduling primitives that reflect different scheduling guarantees, such as resource quota and priority.

Here's a complete example of a queue configuration:

```yaml
apiVersion: scheduling.run.ai/v2
kind: Queue
metadata:
  name: my-queue
spec:
  # Optional display name for the queue
  displayName: "My Queue"
  # Optional parent queue for hierarchical structure
  parentQueue: "default"
  # Optional priority (default is 100)
  priority: 100
  resources:
    # CPU resources in millicpus (1000 = 1 cpu)
    cpu:
      quota: 1000  # Guaranteed CPU quota
      limit: 2000  # Maximum CPU limit
      overQuotaWeight: 1  # Weight for over-quota resource distribution
    # GPU resources in fractions (0.7 = 70% of a GPU)
    gpu:
      quota: 1  # Guaranteed GPU quota
      limit: 2  # Maximum GPU limit
      overQuotaWeight: 1
    # Memory resources in megabytes
    memory:
      quota: 1000  # Guaranteed memory quota
      limit: 2000  # Maximum memory limit
      overQuotaWeight: 1
```

Apply the queue configuration:
```bash
kubectl apply -f queue.yaml
```

#### 2. Workload Setup
Workloads can be configured with various KAI Scheduler-specific options. Here's a complete example of a pod configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-workload
  labels:
    # Required: Specify the queue this workload belongs to
    runai/queue: my-queue
    # Optional: Set workload priority
    priorityClassName: train  # Options: train (default), build, inference
spec:
  # Required: Use KAI Scheduler
  schedulerName: kai-scheduler
  containers:
    - name: main
      image: ubuntu
      args: ["sleep", "infinity"]
      resources:
        requests:
          cpu: 100m
          memory: 250M
        limits:
          # GPU resources
          nvidia.com/gpu: "1"
```

Apply the workload:
```bash
kubectl apply -f workload.yaml
```

### Priority Classes
KAI Scheduler supports the following priority classes:
- `train` (value: 50) - Default for training workloads
- `build` (value: 100) - For build/interactive workloads
- `inference` (value: 125) - For inference workloads
- `build-preemptible` (value: 75) - For preemptible build workloads

## Key Features
* [Batch Scheduling](docs/batch/README.md): Ensure all pods in a group are scheduled simultaneously or not at all.
* Bin Packing & Spread Scheduling: Optimize node usage either by minimizing fragmentation (bin-packing) or increasing resiliency and load balancing (spread scheduling).
* [Workload Priority](docs/priority/README.md): Prioritize workloads effectively within queues.
* [Hierarchical Queues](docs/queues/README.md): Manage workloads with two-level queue hierarchies for flexible organizational control.
* [Resource distribution](docs/fairness/README.md#resource-division-algorithm): Customize quotas, over-quota weights, limits, and priorities per queue.
* [Fairness Policies](docs/fairness/README.md#reclaim-strategies): Ensure equitable resource distribution using Dominant Resource Fairness (DRF) and resource reclamation across queues.
* Workload Consolidation: Reallocate running workloads intelligently to reduce fragmentation and increase cluster utilization.
* [Elastic Workloads](docs/elastic/README.md): Dynamically scale workloads within defined minimum and maximum pod counts.
* Dynamic Resource Allocation (DRA): Support vendor-specific hardware resources through Kubernetes ResourceClaims (e.g., GPUs from NVIDIA or AMD).
* [GPU Sharing](docs/gpu-sharing/README.md): Allow multiple workloads to efficiently share single or multiple GPUs, maximizing resource utilization.
* Cloud & On-premise Support: Fully compatible with dynamic cloud infrastructures (including auto-scalers like Karpenter) as well as static on-premise deployments.

## API
PLEACEHOLDER - How to use?

## Support and Getting Help
Please open [an issue on the GitHub project](https://github.com/NVIDIA/KAI-scheduler/issues/new) for any questions. Your feedback is appreciated.

# KAI Scheduler Demos

This repository contains demonstration examples for the [KAI Scheduler](https://github.com/NVIDIA/KAI-scheduler), showcasing its capabilities in both CPU and GPU resource management. These demos illustrate how the scheduler handles complex scheduling scenarios, including priority-based scheduling, workload consolidation, and gang scheduling.

## Repository Structure

```
.
├── demo/
│   ├── cpu-scheduling-example/     # CPU resource management demo
│   │   ├── README.md              # Detailed guide for CPU demo
│   │   ├── 1-queues.yaml         # Queue hierarchy configuration
│   │   ├── 2-initial-state.yaml  # Initial cluster state
│   │   └── 3-new-workloads.yaml  # New workloads to demonstrate scheduling
│   │
│   └── gpu-scheduling-example/     # GPU resource management demo
│       ├── README.md              # Detailed guide for GPU demo
│       ├── 1-queues.yaml         # Queue hierarchy configuration
│       ├── 2-initial-state.yaml  # Initial cluster state
│       └── 3-new-workloads.yaml  # New workloads to demonstrate scheduling
```

## Demo Contents

### CPU Scheduling Example
- Demonstrates CPU resource allocation and management
- Shows priority-based scheduling
- Includes a distributed training workload
- Demonstrates gang scheduling capabilities
- Shows resource consolidation for CPU workloads

### GPU Scheduling Example
- Demonstrates GPU resource allocation and management
- Shows priority-based scheduling
- Includes GPU-specific features like contiguous GPU allocation
- Demonstrates gang scheduling for GPU workloads
- Shows resource consolidation for GPU workloads

## Prerequisites

To run these demos, you need:
1. A Kubernetes cluster with KAI Scheduler installed (version 1.0 or later)
2. For CPU demo:
   - 4 nodes with 8 CPUs each (32 total CPUs)
   - Training operator installed
3. For GPU demo:
   - 4 nodes with 8 GPUs each (32 total GPUs)
   - NVIDIA device plugin installed and configured
   - Training operator installed

## Getting Started

Each demo has its own README with detailed instructions. Please refer to:
- [CPU Scheduling Demo Guide](demo/cpu-scheduling-example/README.md)
- [GPU Scheduling Demo Guide](demo/gpu-scheduling-example/README.md)

## About KAI Scheduler

The Kubernetes AI (KAI) Scheduler is a robust, efficient, and scalable Kubernetes scheduler that optimizes GPU resource allocation for AI and machine learning workloads. It supports:
- Priority-based scheduling
- Hierarchical queues
- Resource fairness
- Workload consolidation
- Gang scheduling
- GPU sharing
- Elastic workloads

For more information about KAI Scheduler, visit the [official repository](https://github.com/NVIDIA/KAI-scheduler).

## License

This repository is private and confidential. All rights reserved.
