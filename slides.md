---
title: Notes on Scarf
subtitle: Dos and Don'ts
author: Alin M Elena
date: March 7, 2026
institute: scientific computing department
monofont: Hack
mainfont: NotoSans
mathfont: AsanaMath
theme: Montpellier
fontsize: 20pt
classoption:
- professionalofonts
- lualatex
toc: true
intro: false
---

# CPU

## CPU Specifications Overview


+-----------+--------------+----------+-----------+----------------+----------+
| **Type**  | **CPU Name** | **Cores**| **Phys/** | **Cache**      | **NUMA** |
|           |              | **(T/S)**| **Thrd**  | **(L1/L2/L3)** |          |
+===========+==============+==========+===========+================+==========+
| *scarf20* | EPYC 7502    | 32 / 32  | 32 / 64   | 2MB / 16MB /   | 1        |
|           |              |          |           | 128MB          |          |
+-----------+--------------+----------+-----------+----------------+----------+
| *scarf21* | EPYC 7502P   | 32 / 32  | 32 / 64   | 2MB / 16MB /   | 4        |
|           |              |          |           | 128MB          |          |
+-----------+--------------+----------+-----------+----------------+----------+
| *scarf22* | EPYC 7502P   | 32 / 32  | 32 / 64   | 2MB / 16MB /   | 4        |
|           |              |          |           | 128MB          |          |
+-----------+--------------+----------+-----------+----------------+----------+
| *scarf23* | EPYC 7502P   | 32 / 32  | 32 / 64   | 2MB / 16MB /   | 4        |
|           |              |          |           | 128MB          |          |
+-----------+--------------+----------+-----------+----------------+----------+
| *scarf24* | EPYC 9654    | 192 / 96 | 192 / 384 | 12MB / 192MB   | 2        |
|           |              |          |           | / 768MB        |          |
+-----------+--------------+----------+-----------+----------------+----------+

T/S - Total cores/cores per socket

## scarf21

\vspace{-5mm}
![EPYC 7502P High level schematic](figures/scarf21.pdf){width=100%}


## Cores vs. Threads: AMD SMT

- Simultaneous Multithreading (SMT)
- **Physical Core:** The actual hardware execution unit. Scarf nodes have 32 or 96 cores per socket.
- **Thread (Logical CPU):** A virtual core presented to the OS. AMD EPYC uses **2 threads per core**.
- **Slurm Mapping:** Slurm generally counts **threads** as "CPUs".
    - `scarf22`: 32 physical cores = 64 Slurm "CPUs".
    - `scarf24`: 192 physical cores = 384 Slurm "CPUs".

## Cores vs. Threads: Best Practice

- For most MPI applications, it is best to run **one task per physical core** to avoid resource contention within the core.
- **Why?** Threads on the same core share the FPU and L1/L2 caches. Heavy computation on both threads often leads to slowdowns.
- **Strategy:** Request `ntasks-per-node` equal to the number of physical cores and use binding.
- Use `srun --cpu-bind` option to control how tasks are pinned to physical hardware.


## NUMA Domains

- **Non-Uniform Memory Access (NUMA)**
- **Definition:** Memory is physically distributed across the processor. Accessing "local" memory is much faster than "remote" memory.
- **Scarf Configurations:**
    - `scarf20`: Single NUMA node (NPS1).
    - `scarf21-23`: 4 NUMA nodes (NPS4). Memory is split into 4 quarters.
    - `scarf24`: 2 NUMA nodes (1 per socket).

## NUMA performance


- **Latency:** Crossing NUMA boundaries increases memory latency.
- **Bandwidth:** Each NUMA domain has its own memory controller.
- **Strategy:** Align your MPI tasks with NUMA domains (e.g., on `scarf21`, use 8 tasks per NUMA node) to maximize performance.
- **Tooling:** Use `srun --cpu-bind=core` or `numactl` to enforce locality.

## Scarf20-23

- **Architecture:** AMD EPYC 7502 series (Rome).
- **NUMA Configuration:**
    - `scarf20`: Single NUMA node.
    - `scarf21-23`: 4 NUMA nodes (NPS4 configuration).
- **Consistency:** Identical core counts and cache sizes across the 7502/7502P models.

## Scarf24

- **Architecture:** Dual-socket AMD EPYC 9654 (Genoa).
- **Performance:** Significant leap in resources:
    - 192 physical cores (vs 32).
    - 768 MB L3 cache.
    - 2 NUMA nodes.
- **SMT:** 2 threads per core enabled (384 total logical CPUs).


# GPU

## GPU Hardware: NVIDIA A100 Variants

- **80GB High-Memory:** `gn3002`, `gn3003`.
- **40GB Standard-Memory:** `gn3000`, `gn3001` and `gn0001-gn0006`.
- **Form Factor:** All are SXM4 modules with high-speed **NVLink** peer-to-peer interconnect (NV4).

## gn3000-gn3002

![GPU 3000-3002 High level topology](figures/gn3002_topology_diagram.pdf){width=100%}

## gn0001-gn0006

![GPU 0000-0006 High level topology](figures/gn0004_topology_diagram.pdf){width=100%}

## GPU Host CPU: AMD EPYC 7302

- **CPU:** Dual AMD EPYC 7302 (Rome generation).
- **Cores:** 16 physical cores per socket (32 total physical cores).
- **Threads:** 2 threads per core (64 total Slurm "CPUs").
- **Cache:** 256 MiB L3 cache total.
- **Performance:** Designed to feed the A100s with low-latency data via PCIe 4.0 and high NUMA counts.

## GPU Node Networking

- **`gn3000-gn3003` (4 NICs):**
    - Equipped with **4 Mellanox NICs** (`mlx5_0` to `mlx5_3`).
    - Highly optimized: One dedicated NIC per GPU path.
- **`gn0001-gn0006` (2 NICs):**
    - Equipped with **2 Mellanox NICs** (`mlx5_0`, `mlx5_1`).
    - One NIC per pair of GPUs.

## GPU Topology: NVLink & Affinity

- **`gn3000-gn3003`:** 8 NUMA domains (NPS4/socket). Granular binding:
    - **GPU0:** CPU 12-15, 44-47 (NUMA 3).
    - **GPU1:** CPU 4-7, 36-39 (NUMA 1).
    - **GPU2:** CPU 28-31, 60-63 (NUMA 7).
    - **GPU3:** CPU 20-23, 52-55 (NUMA 5).
- **`gn0001-gn0006`:** 2 NUMA domains (NPS1/socket).
    - **GPU0/1:** CPU 0-15, 32-47 (NUMA 0).
    - **GPU2/3:** CPU 16-31, 48-63 (NUMA 1).

## GPU Topology: Performance Impact

- **Impact:** For maximum performance, your application should bind the process using a specific GPU to its corresponding local CPU cores.
- **`gn300x` Rule:** Bind to the specific 4 cores (8 threads) per GPU.
- **`gn000x` Rule:** Bind to the specific 16 cores (32 threads) per GPU pair.
- **Tooling:** Use `srun --accel-bind=g` or set `CUDA_VISIBLE_DEVICES` manually in alignment with task affinity.


# Scarf and slurm


## Partitions


+---------------+---------------+--------------------+----------------------------------+
| **Partition** | **Time Limit**| **Nodes (A/I/O/T)**| **Node List**                    |
+===============+===============+====================+==================================+
| `scarf*`      | 7-00:00:00    | 404/15/35/454      | cn[002-077, 1000-1167,           |
|               |               |                    | 2000-2031, 3020-3131,            |
|               |               |                    | 4000-4065]                       |
+---------------+---------------+--------------------+----------------------------------+
| `ibis`        | 7-00:00:00    | 2/16/2/20          | cn[3000-3019]                    |
+---------------+---------------+--------------------+----------------------------------+
| `devel`       | 12:00:00      | 404/16/36/456      | cn[002-079, 1000-1167,           |
|               |               |                    | 2000-2031, 3020-3131,            |
|               |               |                    | 4000-4065]                       |
+---------------+---------------+--------------------+----------------------------------+
| `gpu`         | 7-00:00:00    | 4/6/0/10           | gn[0001-0006, 3000-3003]         |
+---------------+---------------+--------------------+----------------------------------+
| `gpu-devel`   | 12:00:00      | 4/6/0/10           | gn[0001-0006, 3000-3003]         |
+---------------+---------------+--------------------+----------------------------------+
| `preemptable` | 7-00:00:00    | 406/32/38/476      | cn[002-079, 1000-1167,           |
|               |               |                    | 2000-2031, 3000-3131,            |
|               |               |                    | 4000-4065]                       |
+---------------+---------------+--------------------+----------------------------------+

- selectable with --partition, -p


## Resource Allocation (1/2)


+-----------------------+----------+------------+----------+----------------------------+
| **Node Range**        | **CPUs** | **Memory** | **RAM/** | **Full Features**          |
|                       |          |            | **Core** |                            |
+=======================+==========+============+==========+============================+
| `cn[002-077]`         | 64       | 257 GB     | ~8 GB    | `cpu,amd,scarf20,`         |
|                       |          |            |          | `scarf20a,vsim`            |
+-----------------------+----------+------------+----------+----------------------------+
| `cn[078-079]`         | 128      | 1 TB       | ~16 GB   | `cpu,amd,scarf20,`         |
|                       |          |            |          | `scarf20b,vsim`            |
+-----------------------+----------+------------+----------+----------------------------+
| `cn[1000-1051,1053,`  | 64       | 257 GB     | ~8 GB    | `cpu,amd,scarf21,`         |
| `1055-1107,1109-1167]`|          |            |          | `scarf21a,scarf2122,`      |
|                       |          |            |          | `scarf212223,vsim`         |
+-----------------------+----------+------------+----------+----------------------------+
| `cn[1052,1054,1108,`  | 64       | 257 GB     | ~8 GB    | `cpu,amd,interactive,`     |
| `1114,1134,1154]`     |          |            |          | `scarf21,scarf21a,`        |
|                       |          |            |          | `scarf2122,scarf212223,`   |
|                       |          |            |          | `vsim`                     |
+-----------------------+----------+------------+----------+----------------------------+

- scarf has a default limit set in slurm of 4G per CPU(logical)
- `--constraint, -C` can be used to select features

## Resource Allocation (2/2)


+-----------------------+----------+------------+----------+----------------------------+
| **Node Range**        | **CPUs** | **Memory** | **RAM/** | **Full Features**          |
|                       |          |            | **Core** |                            |
+=======================+==========+============+==========+============================+
| `cn[2000-2031]`       | 64       | 257 GB     | ~8 GB    | `cpu,amd,scarf22,`         |
|                       |          |            |          | `scarf2122,scarf212223,`   |
|                       |          |            |          | `vsim`                     |
+-----------------------+----------+------------+----------+----------------------------+
| `cn[3000-3131]`       | 64       | 257 GB     | ~8 GB    | `cpu,amd,scarf23,`         |
|                       |          |            |          | `scarf212223,scarf212223`  |
+-----------------------+----------+------------+----------+----------------------------+
| `cn[4000-4065]`       | 384      | 1.5 TB     | ~8 GB    | `cpu,amd,scarf24`          |
+-----------------------+----------+------------+----------+----------------------------+
| `gn[0001-0006]`       | 64       | 257 GB     | ~8 GB    | `gpu,amd,scarf21,`         |
|                       |          |            |          | `scarf21c,vsim`            |
+-----------------------+----------+------------+----------+----------------------------+
| `gn[3000-3003]`       | 64       | 257 GB     | ~8 GB    | `gpu,amd,scarf23,`         |
|                       |          |            |          | `scarf23b`                 |
+-----------------------+----------+------------+----------+----------------------------+

# Bad Practices

## Summary


- all these are real life examples from scarf
- all involve electronic structure or molecular dynamics codes
- all authors are from our theme(materials) but patterns apply across themes and departments.
- results:
    - longer run time, longer waiting time in queues, **less science**
    - more energy consumption, **more energy consumed, more money paid**

## Case 1: Fragmentation

+-------------+---------------+---------------+------------+-----------+---------------------------------+
| **JOBID**   | **PARTITION** | **USER**      | **TIME**   | **NODES** | **NODELIST**                    |
+=============+===============+===============+============+===========+=================================+
| xxx         | scarf         |               | 20:03      | 9         | cn[1057,1104,1119,1128,         |
|             |               |               |            |           | 1132,1143,3101,3115,3120]       |
+-------------+---------------+---------------+------------+-----------+---------------------------------+
| yyy         | scarf         |               | 20:04      | 21        | cn[1006,1009,1026-1027,         |
|             |               |               |            |           | 1036-1037,1040,1046-1047,       |
|             |               |               |            |           | 1084,1090,1158,1167,3025,       |
|             |               |               |            |           | 3046,3053-3056,3060,3062]       |
+-------------+---------------+---------------+------------+-----------+---------------------------------+


```bash
#SBATCH -p scarf
#SBATCH -n 96
#SBATCH -c 1
#SBATCH -t 10000:00
```

## Case 1

- **Extreme Time Limits:** Requesting `10000:00` minutes (almost 7 days) blocks scheduling.
- **Node Fragmentation:**
    - Job yyy is scattered across **21 fragmented nodes**.
    - **Impact:** High network latency and increased MPI communication overhead.
- **Missing Constraints:** No architecture constraints (e.g., `--constraint=scarf24`) leading to non-deterministic performance.

## Case 2: Misalignment

```bash
#SBATCH -n 169
#SBATCH -t 120:00:00
#SBATCH --exclusive
```

- **Architecture Agnostic:** Requesting 169 cores without a constraint means the job could span many older 64-core nodes or fewer newer 384-core nodes.
- **Task Alignment:** 169 is not a multiple of the common node core counts (32 or 96), leading to **inefficient bin-packing**.
- **Ambiguous Mapping:** Without `ntasks-per-node`, Slurm decides the distribution, splitting the job across more nodes than necessary.

## Case 3: Waste

```bash
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
```

- issue: **Missing `--constraint`!**
- **The "Scarf24" Trap:**
- **Scenario:** This job requests **64 logical CPUs** and the whole node (`--exclusive`).
- **If it lands on Scarf24:**
    - The job occupies a node with **384 logical CPUs**.
    - **Result:** 320 CPUs (83% of the node) sit idle and are blocked for other users.
- **Lesson:** Always use `--constraint` when using `--exclusive` on a cluster with varying node sizes.

## Case 4: mpirun, mpiexec

```bash
# Avoid this in Slurm:
mpirun -n 32 ~/hello-mpi/hello.x

# Prefer this:
srun ~/hello-mpi/hello.x
```

- **Resource Integration:** `srun` is natively integrated with Slurm, automatically using the job's allocated resources.
- **Process Management:** Slurm cannot properly track processes launched via `mpirun`.
- **Scalability & Binding:** `srun` handles process startup more efficiently and correctly respects Slurm's CPU affinity settings.

## Case 5: Under-utilization

+-------------+---------------+---------------+------------+-----------+---------------------------------+
| **JOBID**   | **PARTITION** | **USER**      | **TIME**   | **NODES** | **NODELIST**                    |
+=============+===============+===============+============+===========+=================================+
| xxx         | scarf         |               | 11:03      | 5         | cn[037-038,040-041,070]         |
+-------------+---------------+---------------+------------+-----------+---------------------------------+
| yyy         | scarf         |               | 16:33      | 12        | cn[1025,1146,2013,3025,         |
|             |               |               |            |           | 3027,3034-3035,3044,3054,       |
|             |               |               |            |           | 3060,3065,3101]                 |
+-------------+---------------+---------------+------------+-----------+---------------------------------+

```bash
#SBATCH -n 64
#SBATCH --cpus-per-task=1
#SBATCH --time=48:00:00
# Launching:
mpiexec -np 32 ./xxx
```

## Case 5

- **Resource Over-allocation:** Requesting 64 tasks (`-n 64`) but only utilizing 32 (`-np 32`) results in **50% CPU waste**.
- **Execution Mismatch:** `mpiexec` is being used instead of `srun`, and its `-np` flag overrides the SLURM allocation.
- **Extreme Fragmentation:** Job yyy is scattered across **12 nodes** to run a task that could fit on a single node.
- **MPI Overhead:** Fragmenting a 32-rank job across 12 nodes significantly increases inter-node communication latency compared to running on 1-2 nodes.

## Case 6: Non-deterministic performance

```bash
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=32
#SBATCH --exclusive
#SBATCH --constraint="scarf20|scarf21|scarf22|scarf23|scarf24"
#SBATCH --cpus-per-task=1
```
\Large
mix and match nodes with different specifications may affect performance

## Case 7: Bad Performance, Ugly Binding
```bash
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=36
#SBATCH --time=00:03:00

export OMP_NUM_THREADS=1

mpirun --cpu-bind=cores
```
- oversubscription, some MPI processes will run on both physical and logical cores...
- imbalance

## Case 7

![MPI Binding for above script](figures/bad_binding_diagram.pdf){width=100%}

# Good Practice

## Minimal Script

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE           # ignore any env variables active in current terminal*
#SBATCH --nodes=1               # how many nodes
#SBATCH --ntasks-per-node=32    # 32 tasks * 2 cores/task = 64
#SBATCH --cpus-per-task=2       # Useful for hybrid MPI+OpenMP
#SBATCH --constraint=scarf22    # Select specific architecture
#SBATCH --exclusive             # Get full node, even if not asking all resources
#SBATCH --time=01:00:00         # Use realistic time limit

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
module load foss/2025a         # Versioned software stack
srun --export=ALL ./my_app   # Native Slurm launch
```


## Minimal script notes

- **Predictable:** Landing on identical nodes with known core counts.
- **Fast:** Smaller nodes often have shorter queue times than large heterogeneous requests.
- **Efficient:** Full node utilization without wasting cores on large nodes.
- note --exclusive, will give you the full node regardless of the resources asked, and "bill" you for the full node, avoids interference from other users...
- --constraint, assures consistent performance
- srun will start $SLURM_NTASKS processes (`nodes*ntask-per-node`)
- you get 4G RAM per Slurm CPU, 64
- cpu partition(scarf) is default.
- default cpu binding on scarf is `--cpu-bind=cores`, check slurm output for exact placement and more information later.


## Minimal Script (Short Flags)

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH -N 1
#SBATCH --ntasks-per-node=32
#SBATCH -c 2
#SBATCH --exclusive
#SBATCH -t 01:00:00
#SBATCH -C scarf22

```
\Large
- less readable
- more compact
- not all long slurm options have short equivalents

## GPU Node Submission

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=8
#SBATCH --gres=gpu:1            # Request 1 GPU
#SBATCH --time=02:00:00

```

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=8
#SBATCH --gres=gpu:1                # Request 1 GPU
#SBATCH --nodelist=gn3002,gn3003    # Select A100-80GB nodes
#SBATCH --time=02:00:00

```

## preemptable

```bash

#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
#SBATCH --time=00:03:00
#SBATCH --constraint="scarf212223"
#SBATCH --partition=preemptable
#SBATCH --requeue

```

- shorten waiting times, especially on scarf were queues are unnaturally long
- gets killed when a scheduled job starts
- good for short jobs that can be restarted easily or automatically
- `--requeue` puts the job back in the queue if killed by scheduler use with care

## Job environments

\Large
three ways to control what environment a job sees
\large

+--------------------------+-------------------------------------+-----------------------------------+
| **Method**               | **Mechanism**                       | **Notes**                         |
+==========================+=====================================+===================================+
| `#!/bin/bash`            | Propagates current environment      | May results in unexpected         |
| `#!/usr/bin/env bash`    | to the job.                         | behaviour.                        |
+--------------------------+-------------------------------------+-----------------------------------+
| `#!/bin/bash --login`    | Sources `~/.bash_profile` similar   | Cleaner environment; depends on   |
|                          | to a ssh login.                     | a sanitised profile.              |
+--------------------------+-------------------------------------+-----------------------------------+
| `#SBATCH --export=NONE`  | Total isolation for maximum         | reproducibility; blocks variable  |
|                          | portability                         | propagation to `srun`; needs      |
|                          |                                     | `srun --export=ALL`.              |
+--------------------------+-------------------------------------+-----------------------------------+

## MPI+OpenMP Hybrid Jobs

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --constraint="scarf212223"
# Use #SBATCH --hint=nomultithread  # if you want to avoid logical cores (SMT)


export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PLACES=threads
export OMP_PROC_BIND=spread

srun --export=ALL --distribution=block:block--cpus-per-task=$SLURM_CPUS_PER_TASK \
--cpu-bind=cores,verbose ./my_app
```
## block:block
\vspace{-1.5cm}
![MPI Binding for 32 MPI processes, with block distribution](figures/block_32mpi_diagram.pdf){width=100%}

## cyclic:cyclic
\vspace{-1.5cm}
![MPI Binding for 32 MPI processes, with cyclic distribution](figures/cyclic_32mpi_diagram.pdf){width=100%}

## Default
\vspace{-1.5cm}
![MPI Binding for 32 MPI processes, with default distribution](figures/default_32mpi_diagram.pdf){width=100%}



## MPI8xOpenMP8

```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=8
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --constraint="scarf212223"

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PLACES=threads
export OMP_PROC_BIND=spread

srun --export=ALL --distribution=block:block--cpus-per-task=$SLURM_CPUS_PER_TASK \
--cpu-bind=cores,verbose ./my_app
```

## block:block
\vspace{-15mm}
![MPI Binding 8MPIx8 openmp, block distribution](figures/block_8mpi_fat_diagram.pdf){width=100%}

## cyclic:cyclic

\vspace{-15mm}
![MPI Binding 8MPIx8 openmp, cyclic distribution](figures/cyclic_8mpi_fat_diagram.pdf){width=100%}


## default
\vspace{-15mm}

![MPI Binding 8MPIx8 openmp, default distribution](figures/default_8mpi_nobind_diagram.pdf){width=100%}

## physical cores


```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=4
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --mem=0
#SBATCH --hint=nomultithread
#SBATCH --constraint="scarf212223"

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PLACES=threads
export OMP_PROC_BIND=spread

srun --export=ALL --distribution=block:block--cpus-per-task=$SLURM_CPUS_PER_TASK \
--cpu-bind=cores,verbose ./my_app
```

## no hint

\vspace{-15mm}
![MPI Binding with OpenMP with SMT enabled](figures/block_8mpi_nohint_diagram.pdf){width=100%}

## hint

\vspace{-15mm}
![MPI Binding with OpenMP with SMT disabled](figures/block_8mpi_hint_diagram.pdf){width=100%}

## hybrid notes

- non trivial if one wants performance
- settings tend to be application (class of applications) specific
- mixing nodes is a recipe for disaster
- the fine tuning buttons may be different compared with other machines so test yours for scarf
- if you use OpenMP pay attention to OMP placing and binding too, not touched here.
- check if SMT offer or not advantages for your case
- if you disable SMT, use `--mem=0` to get access to full memory node, in `--exclusive` or right amount of memory otherwise,
  `--mem-per-cpu` is another option

## Interactive Jobs

\Large
For testing, compilation, or interactive debugging, use `salloc`

Request 1 node and 32 cores interactively for 1 hour
\large
```bash
salloc --nodes=1 --ntasks-per-node=32 --cpus-per-task=1 \
--constraint=scarf23 --exclusive  --time=01:00:00
```
\Large
or, better avoided,  `srun` with the `--pty` flag to get a terminal on a compute node:
\large
```bash
srun --nodes=1 --ntasks-per-node=32 --exclusive --cpus-per-task=1 \
--constraint=scarf23 --time=01:00:00 --pty bash -i
```
\Large
- with salloc you get access to the node as in a job, eg you can run srun
- recommended to use `--exclusive`, especially if you debug
- srun sessions may be killed when commands err

## Task and Memory Management

- **Avoid using `-n`, `--ntasks` (total tasks) alone:**
- **Implicit Fragmentation:** `-n 64` could land on 1 node or 64 nodes. Slurm prioritizes throughput, not locality.
- **Performance Loss:** Spanning more nodes than necessary increases network latency and degrades MPI performance.
- **Recommendation:** Always specify `--nodes` AND `--ntasks-per-node` to force task locality.

## Memory Limits

- **Default Configuration:**
    - **DefMemPerCPU:** 4000 MB (~4 GB) per Slurm "CPU" (logical thread).
    - **Behavior:** If you request 1 core and 2 threads without specifying memory, Slurm allocates **8000 MB**.
- **Options:**
    - **`--mem=<size>`:** Total memory per node (e.g., `--mem=128G`).
    - **`--mem-per-cpu=<size>`:** Memory per requested Slurm CPU
- **Critical Considerations:**
    - **OOM Killing:** If your process exceeds the requested/default limit, Slurm will kill it immediately (Out Of Memory).
    - **Under-estimation:** Be realistic about memory needs. Requesting too little causes crashes; too much can lead to long queue times if you don't use `--exclusive`.
    - **Strategy:** Use `--mem=0` with `--exclusive` whenever your job needs to utilize the whole node or has unpredictable memory spikes.

## Advanced Constraints: **Mixing Scarf Features**

- **Logical AND (`&`):** Must have both features.
    - `--constraint="scarf21&vsim"`
- **Logical OR (`|`):** Can have either feature.
    - `--constraint="scarf21|scarf22"`
- **Logical XOR (`?`):** Must have exactly one of the features, remember NUMA.
    - `--constraint="scarf20?scarf22"`
- **Matching Multiple Groups:** Use the grouped feature flags.
    - `--constraint=scarf212223` (Equivalent to `scarf21|scarf22|scarf23`)

## Process Management: CPU Binding

- **Default Behavior:** Scarf is configured with `TaskPluginParam=verbose`. This means Slurm will log binding information to your `.out` file by default.
- **Importance:** Without binding, the OS scheduler can move processes between cores, flushing caches and increasing latency.
- **Explicit Control:** Use `srun` with binding flags:
    - `--cpu-bind=core`: Bind tasks to physical cores, this is default on scarf at writing.
    - `--cpu-bind=threads`: Bind tasks to logical threads.
    - `--cpu-bind=verbose`: Verify the mapping in your logs.
    - `--cpu-bind=none`: disable binding

## Process Management: Verification

\Large
Check your output file for messages like:
\normalsize
```
cpu-bind=MASK - cn3005, task  0  0 [1620807]: mask 0xffffffffffffffff set
cpu-bind=MASK - cn3006, task 32  0 [1593205]: mask 0x100000001 set
cpu-bind=MASK - cn3005, task  0  0 [1620843]: mask 0x100000001 set
cpu-bind=MASK - cn3006, task 33  1 [1593206]: mask 0x200000002 set
```
\Large
This confirms that Slurm has successfully restricted the task to the requested resources.

**Tip** Use `htop` or `top` (if logged into the node) to visually verify that your processes are pinned to specific cores and not hopping around.


## Throughput: Job Arrays

\Large
**Problem:** Launching 1000 single-node jobs manually.
\normalsize
```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --array=1-1000     # Run 1000 jobs, add %50 if you want 50 at a time
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --constraint="scarf212223"

# Use the array index to select input
./my_app --input=data_${SLURM_ARRAY_TASK_ID}.bin
```

## Why Job Arrays?

- **Scheduler Efficiency:** One job ID for 1000 tasks; much lower overhead for the Slurm controller.
- **Flow Control:** The `%50` limit prevents your jobs from saturating the queue and blocking others.
- **Scalability:** Perfect for parameter sweeps, Monte Carlo simulations, and data processing.
- **Management:** Cancel or monitor all 1000 tasks with a single command (`scancel <jobid>`).
- 1000 is maximum on scarf.
- each job will use 1 node, slurm will start as many as resources available.

## Flexible Node Counts
\Large
If your job can fit on multiple architectures but requires different node counts. **There is no good way.**

One solution is to use multiple jobs, 1 per architecture with a "lockfile"

\large
::: {.columns}
::: {.column width="50%"}
```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=6
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --constraint=scarf212223

```
:::

::: {.column width="50%"}
```bash
#!/usr/bin/env bash
#SBATCH --export=NONE
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=192
#SBATCH --cpus-per-task=2
#SBATCH --exclusive
#SBATCH --time=01:00:00
#SBATCH --constraint=scarf24

```
:::
:::

## ...

\large
::: {.columns}
::: {.column width="50%"}
```bash
lockfile="$PWD/myuniqueID"
if [ -f $lockfile ]; then
  echo "job run already"
else
  touch $lockfile
  module load foss/2025a
  srun ~/hello-mpi/hello.x
fi
```
:::

::: {.column width="50%"}
```bash
lockfile="$PWD/myuniqueID"
if [ -f $lockfile ]; then
  echo "job run already"
else
  touch $lockfile
  module load foss/2025a
  srun ~/hello-mpi/hello.x
fi
```
:::
:::


