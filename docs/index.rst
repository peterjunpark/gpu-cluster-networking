.. meta::
   :description: How to perform network validation testing on optimized hardware
   :keywords: network validation, DCGPU, PCIe, Infiniband, RoCE, ROCm, RCCL, machine learning, LLM, usage, tutorial

***********************************************************************
Cluster Networking Performance Guides for AMD Instinct GPU Accelerators
***********************************************************************

When running AI/HPC applications in a cluster network environment, performance is only as fast as the slowest individual node. It is therefore imperative that you configure each server for optimum data transfer rate and bandwidth usage, then validate host and device-based performance over your network in single and multi node capacities with the appropriate benchmarking tools. 

This guide provides a high-level overview of the actions you must take to ensure you're getting the most out of your available network bandwidth when running GPU accelerators from the AMD Instinct™ product line.  

The single and multi node cluster networking guides provide a high-level summary of the necessary steps to properly test your network configuration with AMD Instinct™ accelerators by running benchmarks that simulate an AI/HPC workload. Each guide includes detailed instructions on system settings, device configuration, networking tools, and performance tests to help you verify your AMD Instinct™-powered GPU clusters are using optimal speeds and bandwidths during operation.

AMD Instinct™ systems come in many shapes and sizes, and cluster design adds yet another dimension of configuration consider. These instructions are therefore written at a sufficiently high level to ensure they can be followed for as many environments as possible. While some scenarios do provide specific examples of hardware, always keep in mind that your configuration is likely to differ from what is being demonstrated in terms of GPUs and CPUs per server, firmware versions, and network interconnect fabric.

- :doc:`Single node network configuration for AMD Instinct GPUs<how-to/single-node-config>`
- :doc:`Multi node network configuration for AMD Instinct GPUs<how-to/multi-node-config>`
