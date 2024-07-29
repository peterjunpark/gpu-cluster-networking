.. meta::
   :description: How to perform network validation testing on optimized hardware
   :keywords: network validation, DCGPU, PCIe, Infiniband, RoCE, ROCm, RCCL, machine learning, LLM, usage, tutorial

***********************************************************************
Cluster Networking Performance Guides for AMD Instinct GPU Accelerators
***********************************************************************

When leveraging AMD GPU accelerator products in a cluster network environment, it is imperative that you configure individual nodes for optimum data transfer rate and bandwidth usage, then validate GPU performance in single and multi node capacities with the appropriate network performance tests. 

This guide provides a high-level overview of the actions you must take to ensure you're getting the most out of your available network bandwidth.  

The single and multi node cluster networking guides provide a high-level summary of the necessary steps to properly test your network configuration with AMD Instinct™ accelerators by simulating conditions similar to a production environment. Each guide includes detailed instructions on system settings, device configuration, networking tools, and performance tests to help you verify your AMD Instinct™-powered GPU clusters are using optimal speeds and bandwidths during operation. 

- :doc:`Single node network configuration for AMD Instinct GPUs<how-to/single-node-config>`
- :doc:`Multi node network configuration for AMD Instinct GPUs<how-to/multi-node-config>`
