# KAI Scheduler Demos

This repository contains demonstration examples for the [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler), showcasing its capabilities in both CPU and GPU resource management. These demos illustrate how the scheduler handles complex scheduling scenarios, including priority-based scheduling, workload consolidation, and gang scheduling.

## Table of Contents
- [Repository Structure](#repository-structure)
- [Demo Contents](#demo-contents)
  - [CPU Scheduling Example](#cpu-scheduling-example)
  - [GPU Scheduling Example](#gpu-scheduling-example)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)

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

