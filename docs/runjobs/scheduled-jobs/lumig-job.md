# LUMI-G example batch scripts

[gpu-binding]: ../../runjobs/scheduled-jobs/distribution-binding.md#gpu-binding

!!! warning "Only 63 cores available on LUMI-G"

    The LUMI-G compute nodes have the low-noise mode activated. This mode
    reserve 1 core to the operating system. As a consequence only 63 cores
    are available to the jobs. Jobs requesting 64 cores/node will never run.

## MPI-based job

!!! note "GPU Binding"

    It is recommended to read the section about [GPU Binding][gpu-binding] for
    more details about the motivation for the binding exemplified is this
    section.

Below, a job script to launch an application with one MPI rank per GPU (GCD). 

- A wrapper script is generated to set the `ROCR_VISIBLE_DEVICES` to the value
  of `SLURM_LOCALID`, i.e., the node local rank.
- CPU binding is done in such a way that the node local rank and GPU ID match
- The `MPICH_GPU_SUPPORT_ENABLED` environment variable is set to `1` to
  enable GPU-aware communication. If this is not required for your application, 
  i.e., your application is not linked against the GPU transfer library remove
  this export

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=standard-g  # Partition (queue) name
#SBATCH --nodes=16              # Total number of nodes 
#SBATCH --ntasks-per-node=8     # 8 MPI ranks per node, 128 total (16x8)
#SBATCH --gpus-per-node=8       # Allocate one gpu per MPI rank
#SBATCH --time=1-12:00:00       # Run time (d-hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_<id>  # Project for billing
#SBATCH --mail-user=username@domain.com

cat << EOF > select_gpu
#!/bin/bash

export ROCR_VISIBLE_DEVICES=\$SLURM_LOCALID
exec \$*
EOF

chmod +x ./select_gpu

CPU_BIND="map_cpu:48,56,16,24,1,8,32,40"

export MPICH_GPU_SUPPORT_ENABLED=1

srun --cpu-bind=${CPU_BIND} ./select_gpu <executable> <args>
rm -rf ./select_gpu
```

## Hybrid MPI+OpenMP job

!!! note "GPU Binding"

    It is recommended to read the section about [GPU Binding][gpu-binding] for
    more details about the motivation for the binding exemplified is this
    section.

Below, a job script to launch an application with one MPI ranks and 6 threads
per GPU (GCD).

- A wrapper script is generated to set the `ROCR_VISIBLE_DEVICES` to the value
  of `SLURM_LOCALID`, i.e., the node local rank.
- CPU binding is done in such a way that the node local rank and GPU ID match.
  Because the first core of the node is not available, we set a CPU mask to
  exclude the first and last core from each group 8 cores.
- The `MPICH_GPU_SUPPORT_ENABLED` environment variable is set to `1` to
  enable GPU-aware communication. If this is not required for your application, 
  i.e., your application is not linked against the GPU transfer library remove
  this export.

```bash
#!/bin/bash -l
#SBATCH --job-name=examplejob   # Job name
#SBATCH --output=examplejob.o%j # Name of stdout output file
#SBATCH --error=examplejob.e%j  # Name of stderr error file
#SBATCH --partition=standard-g  # Partition (queue) name
#SBATCH --nodes=16              # Total number of nodes 
#SBATCH --ntasks-per-node=8     # 8 MPI ranks per node, 128 total (16x8)
#SBATCH --gpus-per-node=8       # Allocate one gpu per MPI rank
#SBATCH --cpus-per-task=6       # 6 threads per ranks
#SBATCH --time=1-12:00:00       # Run time (d-hh:mm:ss)
#SBATCH --mail-type=all         # Send email at begin and end of job
#SBATCH --account=project_<id>  # Project for billing
#SBATCH --mail-user=username@domain.com

cat << EOF > select_gpu
#!/bin/bash

export ROCR_VISIBLE_DEVICES=\$SLURM_LOCALID
exec \$*
EOF

chmod +x ./select_gpu

CPU_BIND="mask_cpu:7e000000000000,7e00000000000000"
CPU_BIND="${CPU_BIND},7e0000,7e000000"
CPU_BIND="${CPU_BIND},7e,7e00"
CPU_BIND="${CPU_BIND},7e00000000,7e0000000000"

export OMP_NUM_THREADS=6
export MPICH_GPU_SUPPORT_ENABLED=1

srun --cpu-bind=${CPU_BIND} ./select_gpu <executable> <args>
rm -rf ./select_gpu
```