.. meta::
   :description: AMD Instinct accelerator compatibility with network cards.
   :keywords: network validation, DCGPU, PCIe, Infiniband, RoCE, card,
              compatibility

***********************
Hardware support matrix
***********************

The processes detailed in these guides are validated to run on the following
hardware in tandem with AMD Instinct™ accelerators:

Network cards for AMD Instinct MI300X
=====================================

+--------------------------+--------------+----------------------+
| Product name             | Speed (GB/s) | Interconnect         |
+==========================+==============+======================+
| Broadcom P2200G          | 400          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom P1400GD         | 400          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom N1400GD         | 400          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom N2200G          | 400          | RoCE v2              |
+--------------------------+--------------+----------------------+
| NVIDIA ConnectX-7 series | 400          | RoCE v2 / InfiniBand |
+--------------------------+--------------+----------------------+

Network cards for AMD Instinct MI200 and MI100
==============================================

+--------------------------+--------------+----------------------+
| Product name             | Speed (GB/s) | Interconnect         |
+==========================+==============+======================+
| Broadcom N2100G          | 200          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom N1200G          | 200          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom P2100G          | 200          | RoCE v2              |
+--------------------------+--------------+----------------------+
| Broadcom P1200G          | 200          | RoCE v2              |
+--------------------------+--------------+----------------------+
| NVIDIA ConnectX-6 series | 200          | RoCE v2 / InfiniBand |
+--------------------------+--------------+----------------------+

When deploying ROCm, consult the
:doc:`ROCm compatibility matrix <rocm:compatibility/compatibility-matrix>` to
ensure compatibility, and install the latest version appropriate for your
operating system and driver support.
