# KAI Scheduler Demos

This repository contains demonstration examples for the [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler), showcasing its capabilities in both CPU and GPU resource management. These demos illustrate how the scheduler handles complex scheduling scenarios, including priority-based scheduling, workload consolidation, and gang scheduling.

## Table of Contents
- [Getting Started](#getting-started)
- [Demo Contents](#demo-contents)
  - [CPU Scheduling Example](#cpu-scheduling-example)
  - [GPU Scheduling Example](#gpu-scheduling-example)
- [Prerequisites](#prerequisites)
- [Documentation](#documentation)

## Getting Started

Each demo has its own README with detailed instructions. Please refer to:
- [CPU Scheduling Demo Guide](demo/cpu-scheduling-example/README.md)
- [GPU Scheduling Demo Guide](demo/gpu-scheduling-example/README.md)


## Demo Contents

### Scheduling Example for CPU & GPU each
- Demonstrates CPU/GPU resource allocation and management
- Shows priority-based scheduling
- Includes a distributed training workload
- Demonstrates gang scheduling capabilities
- Shows resource consolidation for CPU/GPU workloads


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


## Documentation 
For more detailed information about features and how the KAI scheduler works, please refer to the [documentation](https://github.com/NVIDIA/KAI-Scheduler/docs). We welcome any questions in Github issues! 
