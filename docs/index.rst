.. meta::
   :description: How to perform network validation testing on optimized hardware
   :keywords: network validation, DCGPU, PCIe, Infiniband, RoCE, ROCm, RCCL, machine learning, LLM, usage, tutorial

***********************************************************************
Cluster Networking Performance Guides for AMD Instinct GPU Accelerators
***********************************************************************

When leveraging AMD GPU accelerator products in a cluster network environment, it is imperative that you  configure individual nodes for optimum data transfer rate and bandwidth usage, then validate GPU performance in single and multinode capacities with the appropriate network performance tests. 

This guide provides a high-level overview of the actions you must take to ensure you're getting the most out of your available network bandwidth.  

Prerequisites
-------------

Before proceeding in this guide, ensure you have performed all necessary prequisites:

* Install system hardware.
* Install OS on each system.
* `Install RoCM <https://rocm.docs.amd.com/en/latest/deploy/linux/quick_start.html>`_.
* Install network drivers for NICs (add opensm if using InfiniBand).
* Configure network.
* `Compile MPI with GPU support <https://rocm.docs.amd.com/en/latest/how-to/gpu-enabled-mpi.html>`_.
* Build `RCCL tests <https://github.com/ROCm/rccl-tests>`_.
* Install `Slurm Workload Manager <https://slurm.schedmd.com/quickstart_admin.html>`_ (if applicable).
* Implement passwordless SSH.
* Run the :ref:`disable ACS script<disable-acs-script>` for all devices that support it (must be done on each reboot). 
* Add compute libraries (like OpenCL, HIP, and so on). Add headless graphics and multimedia permissions:

   .. code-block:: shell

       sudo usermod -a -G render $LOGNAME
       sudo usermod -a -G video $LOGNAME       

Best Practices
--------------

Applications must be the same on every system. There are two ways to accomplish this: 

1. Have an NFS mount available to all systems where the software is installed. 
2. Make a system image with all the software installed. Note that you must re-image anytime there is a software change. 

See each guide to learn more about:

- :doc:`Single node configuration <single-node-config>`
- :doc:`Multi node configuration <multi-node-config>`
