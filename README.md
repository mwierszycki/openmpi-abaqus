# OpenMPI & Abaqus

In the latest Abaqus releases the default MPI implementation is the IBM Platfrom MPI. The Intel MPI is stated as qualified so both of them can theoretically be used. The support policy for MPI depends on Abaqus version and even HotFix sometimes! Moreover, there are a lot of issues about MPI implementation and Abaqus releases, so be cautious with that. For more details, please see [Abaqus Program Directories](https://media.3ds.com/support/progdir/all/?pdir=simulia,ep623,update01&context=onpremises). However, there is one more option - the [OpenMPI](https://www.open-mpi.org/) can be used with Abaqus. Please find below the list of Abaqus version compatibile with OpenMPI:

| Abaqus | OpenMPI |
|--------|---------|
|2024    |[4.0.7](https://www.open-mpi.org/software/ompi/v4.0/)|
|2023    |[4.0.7](https://www.open-mpi.org/software/ompi/v4.0/)|
|2022    |[4.0.7](https://www.open-mpi.org/software/ompi/v4.0/)|

OpenMPI has a few more advantages in contrast to IBM Platform MPI. OpenMPI offers a very straightforward, flexible and efficient approach for process affinity. In general, it is a three stage approach where the process is first mapped to a specific slot, then ranked and finally bound to it. In the case of Abaqus, a user can control the last stage - binding. The Abaqus process and its threads can be bound to a slot, hardware thread (SMT), core, L1 cache, L2 cache, L3 cache, socket, NUMA node or explicitly defined list of cores. In the case of NUMA-based multi-socket machines or modern complex CPUs architecture like AMD EPYC family, the most interesting options are binding to last-level cache (L3 cache) or sockets/numa nodes.

Last but not least, the OpenMPI is seamlessly integrated with the Slurm. There are three different modes how Slurm launches MPI jobs:

1. Slurm directly launches the tasks and performs initialization of communications through the PMI APIs. It is supported by most modern MPI implementations but cannot be used with Abaqus, because Abaqus job is not a set of simple tasks from Slurm perspective.
2. Slurm creates a resource allocation for the job and then mpirun launches tasks using Slurm's infrastructure (srun). This mode is supported with Abaqus if OpenMPI is used.
3. Slurm creates a resource allocation for the job and then mpirun launches tasks using some mechanism other than Slurm, such as SSH or RSH. This mode is supported with Abaqus if Platform MPI is used. In this case the Abaqus job is initiated outside of Slurm's monitoring or control. To manage such tasks, an access to the nodes from the batch node is required (e.g. via SSH with Host-based Authentication).

Using second mode makes running Abauqs jobs using MPI on Slurm trivial. If you are interested in installing and configuring Slurm and Abaqus, feel free to reach me directly.

## Installing and configuring OpenMPI with Abaqus

Please find below a short instruction on how to install and configure OpenMPI 4.0.7 with Abaqus 2023 on Linux RHEL-like distribution.

First download OpenMPI from the project website:

https://www.open-mpi.org/software/ompi/v4.0/

rebuild RPM package and install it:

```
$ wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.7-1.src.rpm
$ sudo rpmbuild --rebuild openmpi-4.0.7-1.src.rpm
$ yum localinstall -y /root/rpmbuild/RPMS/x86_64/openmpi-4.0.7-1.el7.x86_64.rpm
```

Add to abaqus_v6.env (user level) or custom_v6.env (system level):

```
####################################################
# OpenMPI (4.0.7)
mp_mpi_implementation=OMPI
mp_mpirun_path={OMPI: '/usr/bin/mpirun'}
```

That's it! Now Abaqus will use OpenMPI to run jobs in MPI and hybrid parallel modes.

## Binding Abaqus jobs with OpenMPI

To configure Abaqus processes affinity with OpenMPI and to bind it to NUMA nodes, add the following line to your abaqus_v6.env (user level) or custom_v6.env files:

```
mp_mpirun_options="--bind-to numa -report-bindings"
```

The first option `--bind-to numa` is used to bind ranks to NUMA nodes. The second `-report-bindings` option can be used to save to *.log file the affinity mask which brings information how the Abaqus processes are bound to cores and sockets:
```
[budsoft20:24939] MCW rank 0 bound to socket 0[core 0[hwt 0-1]], socket 0[core 1[hwt 0-1]], socket 0[core 2[hwt 0-1]], socket 0[core 3[hwt 0-1]], socket 0[core 4[hwt 0-1]], socket 0[core 5[hwt 0-1]], socket 0[core 6[hwt 0-1]],
socket 0[core 7[hwt 0-1]]: [BB/BB/BB/BB/BB/BB/BB/BB][../../../../../../../..]
[budsoft20:24939] MCW rank 1 bound to socket 1[core 8[hwt 0-1]], socket 1[core 9[hwt 0-1]], socket 1[core 10[hwt 0-1]], socket 1[core 11[hwt 0-1]], socket 1[core 12[hwt 0-1]], socket 1[core 13[hwt 0-1]], socket 1[core 14[hwt 0
-1]], socket 1[core 15[hwt 0-1]]: [../../../../../../../..][BB/BB/BB/BB/BB/BB/BB/BB]
```
The process binding enforces usage of a memory block which is assigned to the socket (node in the NUMA terminology) where the process is run from the viewpoint of microarchitecture topology. It can be clearly shown with `numastat` command:
```
Per-node process memory usage (in MBs) for PID 25056 (standard)
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         5.74            0.00            5.74
Stack                        0.28            0.00            0.28
Private                   2299.59           34.72         2334.31
----------------  --------------- --------------- ---------------
Total                     2305.61           34.72         2340.33

Per-node process memory usage (in MBs) for PID 25057 (standard)
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.00            5.74            5.74
Stack                        0.00            0.27            0.27
Private                     54.18         2238.79         2292.97
----------------  --------------- --------------- ---------------
Total                       54.18         2244.80         2298.98
```
As we can see, the two Abaqus standard processes (PIDs 25056 and 25057) use Private memory on two different nodes. When the process binding is not used, both processes use Private memory on both nodes at the same time:
```
Per-node process memory usage (in MBs) for PID 31842 (standard)
                          Node 0      	Node 1       	Total
             	   --------------- --------------- ---------------
Huge                        0.00            0.00            0.00
Heap                        0.35            5.46            5.81
Stack                       0.20            0.07            0.27
Private                  1075.83         1223.94         2299.77
----------------  --------------- --------------- ---------------
Total                 	1076.39      	1229.47     	2305.86

Per-node process memory usage (in MBs) for PID 31843 (standard)
                          Node 0      	Node 1       	Total
             	   --------------- --------------- ---------------
Huge                        0.00            0.00            0.00
Heap                        5.09            0.74            5.83
Stack                       0.07            0.21            0.28
Private                   798.13         1294.96         2093.09
----------------  --------------- --------------- ---------------
Total                     803.29         1295.91         2099.20
```
The data bandwidth between socket 0 and memory in node 1 and vice versa is significantly slower than between sockets and memory in the same NUMA nodes whats decreases performance by ~10% in this case (Abaqus benchmark job s4d).

If you use OpenMPI with Abaqus, feel free to [share your experience](https://github.com/mwierszycki/openmpi-abaqus/discussions) with other Abaqus users.

Happy running Abaqus jobs in parallel with OpenMPI on Linux!

## Disclaimers

Please note that this is an unofficial and undocumented configuration. In the case of any problems DS SIMULIA will not support it.
