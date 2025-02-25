<!-- ---
hide:
  - navigation
--- -->

[lmod]: ../../runjobs/lumi_env/Lmod_modules.md
[prgenv]: ../../development/compiling/prgenv.md
[batch]: ../../runjobs/scheduled-jobs/batch-job.md
[slurm]: ../../runjobs/scheduled-jobs/slurm-quickstart.md
[lumig]: ../../hardware/compute/lumig.md
[MI250X]: https://www.amd.com/en/products/server-accelerators/instinct-mi250x

# GPU Early Access Platform

!!! warning

    This page is a preliminary to the guide documenting the use of the GPU
    Early Access Platform (EAP). It contains information specific to the latter, 
    but does not stand on its own. Users of the EAP are also invited to read 
    other sections of the LUMI documentation. In particular, you are invited to
    read the section on the [module system][lmod] and the 
    [programming environment][prgenv].

The Early Access Platform consists of nodes with MI250x GPUs with the intended
use case being to give users access to the software stack so that they can work
on preparing their software for the [LUMI-G][lumig] hardware partition when it
reaches general availability.

!!! warning

    The GPU Early Access Platform (EAP) is highly experimental - your mileage
    may vary. Things may change without warning while we are testing LUMI-G.
    The EAP will be removed once LUMI-G enters general availability.

| Nodes | CPUs             |  CPU cores         | Memory | GPUs            | Disk | Network         |
| :---: | :--------------: | :----------------: | -----: | :-------------: | :--: | :-------------: |
| 4     | 1x AMD EPYC 7A53 | 64 cores           | 512GiB | 4x AMD MI250x   | none | 4x 200 Gb/s     |

!!! note

    Even if each nodes has 4 MI250x, 8 GPUs will be available throught Slurm
    as the MI250x card features 2 GPU dies (GCDs).

## About the programming environment

The programming environment of the EAP is still experimental and do not
entirely reflect the final environment or performance of the progrmming
environment that when LUMI-G will be available. Here is a list of the
characteristics of the current platform that will be available once LUMI-G is
available:

- OpenMP offload support will be available for C/C++ and Fortran with the Cray
  compilers (PrgEnv-cray). The same is true for the AMD compilers. Currently
  the AMD compilers are available by loading the `rocm` module. On the final
  system, they will be a `PrgEnv-amd` module.
- HIP code can be compiled with the Cray C++ compiler wrapper (`CC`) or with
  the AMD `hipcc` compiler wrappper.
- A GPU-aware MPI implementation is available (loading the `cray-mpich`
  module). You can use this MPI library with the Cray and AMD environment.

!!! failure

    Search path definition for the HIP libraries is incomplete at the moment
    in the `rocm` module. If you experience issue at link time or runtime with
    either missing HIP functions or libraries, please export:
    
    ```
    export LD_LIBRARY_PATH=$HIP_LIB_PATH:$LD_LIBRARY_PATH
    ```

## Compiling HIP code

You have two options to compile you HIP code: you can use the ROCm AMD compiler
or the Cray compiler.

=== "Using the Cray compilers"

    To compile HIP code with the Cray C/C++ compiler, load the following modules
    in your environment.

    ```
    module load CrayEnv
    module load PrgEnv-cray
    module load craype-accel-amd-gfx90a
    module load rocm
    ```

    The compilation flags to use to compile HIP code with the Cray C++ 
    compiler wrappers (`CC`) are the following

    ```
    -std=c++11 --rocm-path=${ROCM_PATH} --offload-arch=gfx90a -x hip
    ```

    In addition, at the linking step, you need to link your application with
    the HIP library using the flags

    ```
    --rocm-path=${ROCM_PATH} -L${ROCM_PATH}/lib -lamdhip64
    ```

=== "Using the hipcc wrapper"

    To compile HIP code with the `hipcc` compiler wrapper, load the following
    modules in your environment.

    ```
    module load CrayEnv
    module load rocm
    ```

    After that you can use the `hipcc` compiler wrapper to compile HIP code.

!!! warning

    Be careful and make sure to compile your code using the 
    `--offload-arch=gfx90a` flag in order to compile code optimized for the 
    MI250x GPU architecture. If you omit the flag, the compiler may optimize 
    your code for an older, less suitable architecture.


## Compiling OpenMP offload code

You have to options available OpenMP offload code compilation. The first option
is to use the Cray compilers and the second, to use the AMD compilers provided 
with ROCm.

=== "Using the Cray compilers"

    To compile an OpenMP offload code with the Cray compilers, you need to load the
    following modules:

    ```
    module load CrayEnv
    module load PrgEnv-cray
    module load craype-accel-amd-gfx90a
    module load rocm
    ``` 

    It is critical to load the `craype-accel-amd-gfx90a` module in order to make
    the compiler wrappers aware that you target the MI250x GPUs. To compile the 
    code, use the Cray compiler wrappers: `cc` (C), `CC` (C++) and `ftn`
    (Fortran) with the `-fopenmp` flag.

    === "C"

        ```
        cc -fopenmp -o <exec> <source>
        ```

    === "C++"

        ```
        CC -fopenmp -o <exec> <source>
        ```

    === "Fortran"

        ```
        ftn -fopenmp -o <exec> <source>
        ```

=== "Using the AMD compilers"

    The AMD compilers are available by loading the `rocm` module.

    ```
    module load rocm
    ```

    It will give you access to the `amdclang` (C), `amdclang++` (C++) and 
    `amdflang` (Fortran) compilers. In order to compile OpenMP offload code, you 
    need to pass additional target flags to the compiler.

    === "C"

        ```
        amdclang -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa \
                -Xopenmp-target=amdgcn-amd-amdhsa -march=gfx90a \
                -o <exec> <source>
        ```

    === "C++"

        ```
        amdclang++ -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa \
                  -Xopenmp-target=amdgcn-amd-amdhsa -march=gfx90a \
                  -o <exec> <source>
        ```

    === "Fortran"

        ```
        amdflang -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa \
                -Xopenmp-target=amdgcn-amd-amdhsa -march=gfx90a \
                -o <exec> <source>
        ```

## Compiling a HIP+MPI code

The MPI implementation available on LUMI is GPU-aware. It means that you can
pass a pointer to memory allocated on the GPU to the MPI calls. This MPI 
implementation can be used by loading the `cray-mpich` module loaded.

```bash
module load CrayEnv
module load craype-accel-amd-gfx90a
module load cray-mpich
module load rocm

export MPICH_GPU_SUPPORT_ENABLED=1
```

=== "Using the Cray compilers"

    The Cray compiler wrappers will automatically link your application (acting 
    similarly to `mpicc`) to the MPI library and Cray GPU transfer library 
    (`libmpi_gtl`). Still, you need to use the flags presented in the 
    [previous section](#compiling-hip-code) in order to compile HIP code.

=== "Using the hipcc compiler wrapper"

    When using the `hipcc` compiler wrapper, you need the explicitly link your
    application with the MPI and GPU transfer libraries using the following 
    flags:

    ```
    -I${MPICH_DIR}/include \
    -L${MPICH_DIR}/lib -lmpi \
    -L${CRAY_MPICH_ROOTDIR}/gtl/lib -lmpi_gtl_hsa
    ```

!!! warning

    When running GPU-aware MPI code, **you need to enable GPU support** using 
    `MPICH_GPU_SUPPORT_ENABLED=1`. Failing to do so will lead to failure if 
    your application use GPU pointers in MPI calls. This usually manifest as
    a bus error. Note also that the Cray MPI do not support GPU-aware MPI for 
    multiple GPUs per rank, i.e. **you should only use one GPU per MPI rank**.

## Submitting jobs

LUMI use Slurm as a job scheduler. If you are not familiar with Slurm, please 
read the [Slurm quick start guide][slurm]

To submit jobs to the Early Access platform you need to select the `eap` 
partition and provide you project number. Below is an example job script to 
launch an application with 2 MPI ranks with 8 threads and 1 GPU per rank.

```bash title="eap.job"
#!/bin/bash

#SBATCH --partition=eap (1)
#SBATCH --account=<project_XXXXXXXXX> (2)
#SBATCH --time=10:00 (3)
#SBATCH --nodes=2 (4)
#SBATCH --ntasks-per-node=8 (5)
#SBATCH --cpus-per-task=8 (6)
#SBATCH --gpus-per-node=8 (7)

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK # (8)
export MPICH_GPU_SUPPORT_ENABLED=1 # (9)

srun <executable> # (9)
```

1. Select the Early Access Partition

2. Change this value to match your project number. If you don't know your
   project number use the `groups` command. You should see that you are a
   member of a group looking like this: `project_XXXXXXXXX`.

3. The format for time is `dd-hh:mm:ss`. In this case, the requested time is
   10 minutes.
   
4. Request 2 nodes. 
   
5. Request 8 tasks per node. A task in Slurm speaks is a process. If your application
   use MPI, it corresponds to the number of MPI ranks.

6. Request 8 threads per task. If your application is multithreaded (for
   example, using OpenMP) this is how you control the number of threads.

7. Request 8 GPUs for this job, one for each task. Most of the time the number
   of GPUs is the same as the number of tasks (MPI ranks).

8. If your application is multithreaded with OpenMP, set the value of
   `OMP_NUM_THREADS` to the value set with `--cpus-per-task`

9. If your code needs a GPU-aware MPI

10. Launch your application with `srun`. There are no `mpirun`/`mpiexec` on
   LUMI. You should always use `srun` to launch your application. If your
   application doesn't use MPI you can omit it.

Once your job script is ready, you can submit your job using the `sbatch`
command.

```bash
sbatch eap.job
```

More information about available batch script parameters is available
[here][batch]. The table below summarizes the GPU-specific options.

| Option             | Description                                        |
|--------------------|----------------------------------------------------|
| `--gpus`           | Specify the total number of GPUs across all nodes  |
| `--gpus-per-node`  | Specify the number of GPUs per node                |
