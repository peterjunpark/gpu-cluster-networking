.. meta::
   :description: How to configure multiple nodes for testing
   :keywords: network validation, DCGPU, multi node, ROCm, RCCL, machine learning, LLM, usage, tutorial

******************************************************
Multi node network configuration for AMD Instinct GPUs
******************************************************

With single node configuration testing completed and verified, we can move on to validating network connections in node pairs. All the tests described in this guide must be run between two nodes in a client-server relationship. Both nodes must be configured verified per the :doc:`Single node configuration guide<single-node-config>` before running any node-to-node performance tests.

Prerequisites
=============

* Install required software on each node:
   * `Compile MPI with GPU support <https://rocm.docs.amd.com/en/latest/how-to/gpu-enabled-mpi.html>`_.
   * Build `RCCL tests <https://github.com/ROCm/rccl-tests>`_.
   * Install `Slurm Workload Manager <https://slurm.schedmd.com/quickstart_admin.html>`_ (if applicable).
* Implement passwordless SSH.

Evaluate platform-specific BIOS tunings
---------------------------------------

Check your BIOS settings to make sure they are optimized for AMD GPUs. See the `MI200 Tuning Guide <https://rocm.docs.amd.com/en/latest/how_to/tuning_guides/mi200.html>`_ for more details.

* Enable large bar addressing in the BIOS to support peer to peer GPU memory access.
* Verify SRIOV is enabled, if needed.
* Disable ACS (ACS forces P2P transactions through the PCIe root complex).

.. Note::
    If using virtual devices, AER and ACS should be enabled.

.. _OFED-Perftest-installation-and-benchmarking:

OFED Perftest installation and benchmarking
============================================

Install and run the `OFED performance tests <https://github.com/linux-rdma/perftest>`_ for host to host (H2H) testing. Loopback is implemented in the tests to remove the switch from benchmark results. Remember to install OFED Perfests on both nodes you plan to use in this section. Commands may require ``sudo`` depending on user privileges.

1. From the CLI of your host, run ``git clone https://github.com/linux-rdma/perftest.git``.

2. Navigate to the installation directory and build the tests.

    .. code-block:: shell

        $ cd perftest
        $ ./autogen.sh
        $ ./configure --prefix=$PWD/install --enable-rocm --with-rocm=/opt/rocm

3. Locate and open ``Makefile`` in your editor of choice, then append ``-D__HIP_PLATFORM_AMD__`` to ``CFLAGS`` and ``CXXFLAGS`` (lines 450 and 458 at the time of publication). This is required to compile the code correctly for this guide.

4. Run ``make && make install``.

5. Repeat these steps on a second node connected to the same switch.

Run Host to Host (H2H) Performance Tests (NIC-only)
---------------------------------------------------

Once installed, there are six main modules available with OFED Perftests:

* ib_write_bw - Test bandwidth with RDMA write transactions.
* ib_write_lat - Test latency with RDMA write transactions.
* ib_read_bw - Test bandwidth with RDMA read transactions.
* ib_read_lat - Test latency with RDMA read transactions.
* ib_send_bw - Test bandwidth with send transactions.
* ib_send_lat - Test latency with send transactions.

The examples in this section use **ib_send_bw**, but you may accomplish similar with any other test you require. The goal of the tests in this section is to verify high speed data transfer rates between nodes prior to including GPU traffic, therefore ``use_rocm`` flag is avoided in all commands.

1. Connect to the CLI of both nodes you installed the OFED perftests on.

2. Initiate a server connection on the first node:

    .. code-block:: shell
        
        $ cd perftest   #if not already in directory
        $ numactl -C 1 ./ib_send_bw -a -F -d <IB/RoCE interface>
        
        ************************************
        * Waiting for client to connect... *
        ************************************

3. Initiate a client connection on the second node:

    .. code-block:: shell

        $ cd perftest   #if not already in directory
        $ numactl -C 1 ./ib_send_bw <node1 IP> -a -F -d <IB/RoCE interface>

4. Test should run and complete in several moments.
      
.. note::
   The use of ``numactl`` or ``taskset`` commands makes sure NUMA domains are not crossed when communicating, which can create overhead and latency. When running tests you must ensure you use cores local to the network device.

Consult this table for an explanation of flags used in the ``numactl`` examples and other optional flags that may be useful for you.

.. raw:: html

   <style>
     #perftest-commands-table tr td:last-child {
       font-size: 0.9rem;
     }
   </style>

.. container::
   :name: perftest-commands-table

   .. list-table::
      :header-rows: 1
      :stub-columns: 1
      :widths: 2 5

      * - Flag
        - Description

      * - -d <IB/RoCE interface>
        - Specifies a NIC to use. Ensure you use a NIC that is both adjacent to a GPU and not crossing NUMA domains or otherwise needing pass traffic between CPUs before egressing from the host. Tools like ``rocm-smi --showtopo`` and ``lstopo`` can help define which NICs are adjacent to which GPUs.

      * - -p <port #>
        -  Assign a port number to the server/client, when running simultaneously you must use different ports.

      * - --report_gbits
        - Reports in Gb/s instead of Mb/s.

      * - -m <mtu>
        - Set MTU size.
    
      * - -b
        - Bidirectional runs.

      * - -a 
        - Runs messages in all sizes.

      * - -n <number> 
        - Provides the number of iterations.

      * - -F
        - Do not show warning if cpufreq_ondemand is loaded.

      * - --use_rocm=<rocm_device_number>
        - This is for device testing, allows you to specify which GPU to use. Zero-based numbering. 
     
      * - --perform_warm_up 
        - Runs several iterations before benchmarking to warm up memory cache.

As servers typically have one NIC per GPU, you must change the device location frequently as you iterate through tests. 

Run Multithreaded H2H Performance Tests
---------------------------------------

You can multithread an OFED perftest by running it simultaneously on each NIC in the server. Use ``taskset`` to select a CPU core on the same NUMA domain as the NICs. Although testing the XGMI/Infinity Fabric link between CPUs is not a goal at this point, it's an option if preferred.

Run Extended Multithreaded H2H Performance Tests
-------------------------------------------------

Run the previous test, but this time loop it and run it for a minimum of 8 hours. The goal is to stress the IO network on the fabric over a long period of time.

Run Device-based (GPU) OFED Performance Tests
=============================================

Once H2H performance is verified, you can run the OFED perftests again with GPU traffic included.

Device-to-Device (D2D) RDMA benchmark
-------------------------------------

Use this example to run an OFED perftest between GPUs in pairs (GPU0 to GPU1, GPU2 to GPU3, and so on). 

.. note::
   If you have Mellanox/Nvidia NIC, be aware that the default OFED perftest installation doesn't include ROCm support. Follow the :ref:`installation instructions<OFED-Perftest-installation-and-benchmarking>` if you haven't done so already.

In this example, localhost is used by the client to call the server. You may use a specific IP address to ensure the network is tested. 

.. code-block:: shell

   $ (ib_write_bw -b -a -d <RDMA-NIC-1> --report_gbits -F -use_rocm=0 >> /dev/null &); sleep 1; ib_write_bw -b -a -d <RDMA-NIC-2> --report_gbits -use_rocm=0 -F localhost
   ---------------------------------------------------------------------------------------
                    RDMA_Write Bidirectional BW Test
   Dual-port       : OFF          Device         : <RDMA-NIC-2>
   Number of qps   : 1            Transport type : IB
   Connection type : RC           Using SRQ      : OFF
   PCIe relax order: ON
   ibv_wr* API     : OFF
   TX depth        : 128
   CQ Moderation   : 100
   Mtu             : 4096[B]
   Link type       : Ethernet
   GID index       : 3
   Max inline data : 0[B]
   rdma_cm QPs     : OFF
   Data ex. method : Ethernet
   ---------------------------------------------------------------------------------------
   local address: LID 0000 QPN 0x0901 PSN 0x5e30c8 RKey 0x2000201 VAddr 0x007fe663d20000
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:01:01:101:45
   remote address: LID 0000 QPN 0x0901 PSN 0xf40c3c RKey 0x2000201 VAddr 0x007f282a06e000
   GID: 00:00:00:00:00:00:00:00:00:00:255:255:01:01:101:35
   ---------------------------------------------------------------------------------------
   #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
   2          5000           0.142947            0.012281            0.767588
   4          5000             0.28               0.26               8.255475
   8          5000             0.55               0.54               8.471791
   16         5000             1.16               1.16               9.025968
   32         5000             2.31               2.27               8.865877
   64         5000             4.49               4.43               8.647051
   128        5000             8.98               8.96               8.745890
   256        5000             17.57              16.32              7.969287
   512        5000             34.63              34.41              8.400441
   1024       5000             67.22              66.92              8.168969
   2048       5000             129.04             126.20             7.702863
   4096       5000             188.76             188.56             5.754307
   8192       5000             194.79             192.62             2.939080
   16384      5000             195.32             195.21             1.489355
   32768      5000             203.15             203.13             0.774887
   65536      5000             204.12             203.85             0.388818
   131072     5000             204.44             204.43             0.194964
   262144     5000             204.51             204.51             0.097517
   524288     5000             204.56             204.56             0.048770
   1048576    5000             204.57             204.57             0.024387
   2097152    5000             204.59             204.59             0.012194
   4194304    5000             204.59             204.59             0.006097
   8388608    5000             204.59             204.59             0.003049
   ---------------------------------------------------------------------------------------

.. note::
   If you run the test with different values for --use_rocm=# on the server and the client, the output will show results from whichever GPU is local to the node you're looking at. The tool is unable to show server and client simultaneously.

H2D and D2H RDMA Benchmark
--------------------------

This is similar to the D2D test, but also includes the CPU on either the server or client side of the test-case scenarios. 

for a 2-CPU/8-GPU node you would have have 32 test scenarios per pairs of server.

.. list-table:: H2D/D2H Benchmark with Server-Side CPUs
   :widths: 25 25 25 25 25 25 25 25 25
   :header-rows: 1

   * - Client
     - GPU 0
     - GPU 1
     - GPU 2
     - GPU 3
     - GPU 4
     - GPU 5
     - GPU 6
     - GPU 7 
   * - Server
     - CPU 0
     - CPU 1
     -
     -
     -
     -
     -
     -

.. list-table:: H2D/D2H Benchmark with Client-Side CPUs
   :widths: 25 25 25 25 25 25 25 25 25
   :header-rows: 1

   * - Server
     - GPU 0
     - GPU 1
     - GPU 2
     - GPU 3
     - GPU 4
     - GPU 5
     - GPU 6
     - GPU 7 
   * - Client
     - CPU 0
     - CPU 1
     -
     -
     -
     -
     -
     -

To run this test, use a command similar to the example in the D2D benchmark, but only add the ``--use_rocm`` flag on either the server or client side so that one node communicates with the GPUs while the other does so with CPUs. Then run the test a second time with the ``use_rocm`` flag on the other side. Continue to use the most adjacent NIC to the GPU or CPU being tested so that communication doesn't run across the Infinity Fabric between CPUs (testing this isn't a goal at this time). 

D2D RDMA Multithread Benchmark
------------------------------

For this test you must run the previous D2D benchmark simultaneously on all GPUs. Scripting is required to accomplish this, but the command input should resemble something like the following image with regard to your RDMA device naming scheme.

.. image:: ../data/D2D-perftest-multithread.png
   :alt: multithread perftest input

Important OFED perftest flags for this effort include:

* ``-p <port#>`` - Lets you assign specific ports for server/client combinations. Each pair needs an independent port number so you don't inadvertently use the wrong server. 
* ``-n <# of iterations>`` - Default is 1000, you can increase this to have the test run longer. 
* For bandwidth tests only:
   - ``-D <seconds>`` - Defines how long the test runs for. 
   - ``--run_infinitely`` - Requires user to break the runtime, otherwise runs indefinitely. 

D2D RDMA Multithread Extended Benchmark
---------------------------------------

Perform the D2D RDMA multithread benchmark again, but set the duration for a minimum of 8 hours.

Install and configure AI/HPC workload environment 
=================================================

This section guides you through setting up the tools necessary to simulate an AI workload on your GPU nodes after they have been sufficiently traffic-tested.

You must install the following:

* UCX & MPI (OpenMPI, MPICH, MVAPICH, CrayMPI)
* RCCL Collectives Test
* UCC Collectives test
* OSU Microbenchmarks (OMB) (with ROCM support)

Install RCCL
-------------

RCCL is likely already installed on your nodes, but you can build the latest version from source at https://github.com/ROCm/rccl
(RCCL does require ROCm to already be installed.)

Install UCC
-------------

UCC is used with MPI for communicating over different types of RDMA enabled interconnects like RoCE and InfiniBand. 

.. code-block:: shell

   $ git clone https://github.com/openucx/ucc ; cd ucc
    
   $ ./autogen.sh
   
   $ ./configure --prefix=/opt/ucx/ucc --with-rocm=/opt/rocm --with-ucx=/opt/ucx
   
   $ make -j 8
   
   $ sudo make install

Do not erase the source code folder after compiling and installing, as it's required to install the UCC collective tests in a later section.

To run with UCC you must also add additional parameters.

.. code-block:: shell

   mpirun --mca pml ucx --mca osc ucx \
   --mca coll_ucc_enable 1     \
   --mca coll_ucc_priority 100 -np 64 ./my_mpi_app

Install and compile OpenMPI with UCX and UCC
--------------------------------------------

.. code-block:: shell

   git clone --recursive -b v4.1.x  https://github.com/open-mpi/ompi.git ; cd ompi

   ./autogen.pl

   mkdir build ; cd build

  ../configure --prefix=/opt/ompi --with-ucx=/opt/ucx --with-ucc=/opt/ucx/ucc \ 
   --enable-mca-no-build=btl-uct

   make -j 8 & make install

Build RCCL collectives test
---------------------------

To more easily build and run the RCCL tests, review and implement the script provided in the drop-down. Otherwise, you can follow the steps to manually install at https://github.com/ROCm/rccl-tests. 

.. dropdown:: build-and-run_rccl-tests_sweep_multinode.sh

    .. code-block:: shell
      :linenos:

      #!/bin/bash -x
  
      ## change this if ROCm is installed in a non-standard path
      ROCM_PATH=/opt/rocm
      
      ## to use pre-installed MPI, change `build_mpi` to 0 and ensure that libmpi.so exists at `MPI_INSTALL_DIR/lib`.
      build_mpi=1
      MPI_INSTALL_DIR=/opt/ompi
      
      ## to use pre-installed RCCL, change `build_rccl` to 0 and ensure that librccl.so exists at `RCCL_INSTALL_DIR/lib`.
      build_rccl=1
      RCCL_INSTALL_DIR=${ROCM_PATH}
      
      
      WORKDIR=$PWD
      
      ## building mpich
      if [ ${build_mpi} -eq 1 ]
      then
          cd ${WORKDIR}
          if [ ! -d mpich ]
          then
              wget https://www.mpich.org/static/downloads/4.1.2/mpich-4.1.2.tar.gz
              mkdir -p mpich
              tar -zxf mpich-4.1.2.tar.gz -C mpich --strip-components=1
              cd mpich
              mkdir build
              cd build
              ../configure --prefix=${WORKDIR}/mpich/install --disable-fortran --with-ucx=embedded
              make -j 16
              make install
          fi
          MPI_INSTALL_DIR=${WORKDIR}/mpich/install
      fi
      
      
      ## building rccl (develop)
      if [ ${build_rccl} -eq 1 ]
      then
          cd ${WORKDIR}
          if [ ! -d rccl ]
          then
              git clone https://github.com/ROCm/rccl -b develop
              cd rccl
              ./install.sh -l
          fi
          RCCL_INSTALL_DIR=${WORKDIR}/rccl/build/release
      fi
      
      
      ## building rccl-tests (develop)
      cd ${WORKDIR}
      if [ ! -d rccl-tests ]
      then
          git clone https://github.com/ROCm/rccl-tests
          cd rccl-tests
          make MPI=1 MPI_HOME=${MPI_INSTALL_DIR} NCCL_HOME=${RCCL_INSTALL_DIR} -j
      fi
      
      
      ## running multi-node rccl-tests all_reduce_perf for 1GB
      cd ${WORKDIR}
      
      ## requires a hostfile named hostfile.txt for the multi-node setup in ${WORKDIR}/
      
      n=`wc --lines < hostfile.txt`   # count the numbers of nodes in hostfile.txt
      echo "No. of nodes: ${n}"       # print number of nodes
      m=8                             # assuming 8 GPUs per node
      echo "No. of GPUs/node: ${m}"   # print number of GPUs per node
      total=$((n * m))                # total number of MPI ranks (1 per GPU)
      echo "Total ranks: ${total}"    # print number of GPUs per node
      
      ### set these environment variables if using Infiniband interconnect
      ## export NCCL_IB_HCA=^mlx5_8
      
      ### set these environment variables if using RoCE interconnect
      ## export NCCL_IB_GID_INDEX=3
      
      for coll in all_reduce all_gather alltoall alltoallv broadcast gather reduce reduce_scatter scatter sendrecv
      do
          # using MPICH; comment next line if using OMPI
          mpirun -np ${total} --bind-to numa -env NCCL_DEBUG=VERSION -env PATH=${MPI_INSTALL_DIR}/bin:${ROCM_PATH}/bin:$PATH -env LD_LIBRARY_PATH=${RCCL_INSTALL_DIR}/lib:${MPI_INSTALL_DIR}/lib:$LD_LIBRARY_PATH ${WORKDIR}/rccl-tests/build/${coll}_perf -b 1 -e 16G -f 2 -g 1 2>&1 | tee ${WORKDIR}/stdout_rccl-tests_${coll}_1-16G_nodes${n}_gpus${total}.txt
      
          ## uncomment, if using OMPI
          ## mpirun -np ${total} --bind-to numa -x NCCL_DEBUG=VERSION -x PATH=${MPI_INSTALL_DIR}/bin:${ROCM_PATH}/bin:$PATH -x LD_LIBRARY_PATH=${RCCL_INSTALL_DIR}/lib:${MPI_INSTALL_DIR}/lib:$LD_LIBRARY_PATH --mca pml ucx --mca btl ^openib ${WORKDIR}/rccl-tests/build/${coll}_perf -b 1 -e 16G -f 2 -g 1 2>&1 | tee ${WORKDIR}/stdout_rccl-tests_${coll}_1-16G_nodes${n}_gpus${total}.txt
      
          sleep 10
      done

.. Add or link to the RCCL config script once it's cleared for publication.

Install OSU Microbenchmarks with ROCm support
---------------------------------------------

OSU Microbenchmarks (OMB) make use of MPI to communicate. There are several installation methods to choose here. Review `ROCm documentation <https://rocm.docs.amd.com/en/latest/how-to/gpu-enabled-mpi.html#rocm-enabled-osu-benchmarks>`_ for instructions on installing OMB with OpenMPI, or follow the separate install instructions in this section.

.. Note::
   When building OSU Benchmarks, there is a known issue where configuration will not work correctly with current versions of ROCm. As a workaround, use a configuration command (demonstrated below) that includes changes to the CFLAGS and CXXFLAGS. 

.. code-block:: shell

   wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-7.2.tar.gz
      
   tar zxvf osu-micro-benchmarks-7.2.tar.gz
      
   cd osu-micro-benchmarks-7.2/

   ./configure --enable-rocm --with-rocm=/opt/rocm --prefix=/opt/omb7.2 CC=/opt/ompi/bin/mpicc CXX=/opt/ompi/bin/mpicxx CFLAGS="-g -O2 -D__HIP_PLATFORM_AMD__" CXXFLAGS="-g -O2 -D__HIP_PLATFORM_AMD__ -std=c++11"

   make -j 4
      
   sudo make install

Build UCC Collective Test
-------------------------

Return to the location you cloned the source code for UCC previously. Now that MPI is installed, you can run the configure command and add `--with-mpi=/opt/ompi`` so it builds the MPI perftest (this is done out-of-sequence because MPI wth UCC support requires UCC to be already be built, and is in turn a dependency for the UCC collective test). 

.. code-block:: shell

   ./configure --prefix=/opt/ucx/ucc --with-rocm=/opt/rocm --with-ucx=/opt/ucx \
   --with-mpi=/opt/ompi/

   make -j 4
   
   sudo make install

Running AI/HPC workloads
========================

Once installed and on both systems, running OMB requires passwordless ssh between the servers and they must also be finger-printed, otherwise MPI will fail. 

OMB has two main types of benchmarks: point to point (pt2pt) and collectives. In a typical use case, you start with a pair of nodes and run the pt2pt workloads. 



Point to Point (pt2pt) OSU Benchmarks
-------------------------------------

Commands in the table below must run on 2 nodes with RoCE or Infiniband interconnect from Host to Host (CPU to CPU). You can invoke the command from either node, but directories must mirror one another or the tests will hang.

.. raw:: html

   <style>
     #osu-commands-table tr td:last-child {
       font-size: 0.9rem;
     }
   </style>

.. container::
   :name: osu-commands-table

   .. list-table::
      :header-rows: 1
      :stub-columns: 1
      :widths: 2 5

      * - Command
        - Usage

      * - osu_bw
        - /opt/ompi/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host <node1-IP>,<node2-IP> -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/omb7.2/libexec/osu-micro-benchmarks/mpi/pt2pt/osu_bw -d rocm

      * - osu_bibw
        - /opt/ompi/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host <node1-IP>,<node2-IP> -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/omb7.2/libexec/osu-micro-benchmarks/mpi/pt2pt/osu_bibw -d rocm 

      * - osu_mbw_mr
        - /opt/ompi/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host <node1-IP>,<node2-IP> -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/omb7.2/libexec/osu-micro-benchmarks/mpi/pt2pt/osu_mbw_mr -d rocm

      * - osu_latency
        - /opt/ompi/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host <node1-IP>,<node2-IP> -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/omb7.2/libexec/osu-micro-benchmarks/mpi/pt2pt/osu_latency -d rocm

      * - osu_multi_lat
        - /opt/ompi/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host <node1-IP>,<node2-IP> -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/omb7.2/libexec/osu-micro-benchmarks/mpi/pt2pt/osu_multi_lat -d rocm 

You can change the communication mode from H2D by appending ``D D`` to the end of command for D2D, or ``D H`` for D2H.

Collective OSU Benchmarks
-------------------------

The primary difference between the pt2pt and collective benchmarks is that collectives support the use of multiple (more than 2) devices.

.. raw:: html

   <style>
     #coll-commands-table tr td:last-child {
       font-size: 0.9rem;
     }
   </style>

.. container::
   :name: coll-commands-table

   .. list-table::
      :header-rows: 1
      :stub-columns: 1
      :widths: 2 5

      * - Command
        - Usage

      * - osu_allreduce
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host 10.1.10.110,10.1.10.72 -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_allreduce -d rocm D D
      
      * - osu_allreduce 2N 16Proc
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 16 -hostfile ./hostfile -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_allreduce -d rocm D D

      * - osu_alltoall
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host 10.1.10.110,10.1.10.72 -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoall -d rocm D D

      * - osu_alltoall 2N 16Proc
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 16 -hostfile ./hostfile -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoall -d rocm D D

      * - osu_allgather
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 2 -host 10.1.10.110,10.1.10.72 -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_allgather -d rocm D D

      * - osu_allgather 2N 16Proc
        - /opt/ompi-wIB/bin/mpirun --mca pml ucx --mca osc ucx --mca spml ucx --mca btl ^self,vader,openib --mca coll_hcoll_enable 0 --bind-to none -np 16 -hostfile ./hostfile -x UCX_TLS=all -x MV2_USE_ROCM=1 -x HIP_VISIBLE_DEVICES=1 numactl --localalloc /opt/osu-7.3/libexec/osu-micro-benchmarks/mpi/collective/osu_allgather -d rocm D D

RCCL Test
---------

RCCL is a collective communication library optimized for collective operations by multi-GPU and multi-node communication primitives that are in turn optimized for AMD GPUs. The RCCL Test is typically launched using MPI, but you can use MPICH or openMPI as well. 

.. Add output data when and if it's determined to be fit for public availability.

.. code-block:: shell

  /opt/ompi/bin/mpirun -mca oob_tcp_if_exclude docker,lo -mca btl_tcp_if_exclude docker,lo -host gt-pl1-u19-08:8,gt-pl1-u19-18:8 -np 16 -x LD_LIBRARY_PATH=/opt/rccl/build/rccl/install/lib:/opt/ompi/lib -x NCCL_IB_GID_INDEX=3 -x NCCL_DEBUG=VERSION -x NCCL_IB_HCA=bnxt_re0,bnxt_re1,bnxt_re2,bnxt_re3,bnxt_re4,bnxt_re5,bnxt_re6,bnxt_re7 -x NCCL_IGNORE_CPU_AFFINITY=1 -x NCCL_MAX_NCHANNELS=64 -x NCCL_MIN_NCHANNELS=64 /opt/rccl-tests/build/all_reduce_perf -b 8 -e 16G -f 2 -g 1