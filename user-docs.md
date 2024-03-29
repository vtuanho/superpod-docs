Slurm Usage Guide
===
## Concept

SSH flow: Go by *hanoi* -> get into *login-sp.vinai-systems.com* 
```
ssh hanoi
ssh <username>@login-sp.vinai-systems.com
Ex: ssh tantnd@login-sp.vinai-systems.com
```

**HOME_FOLDER_ISILON <=> /home/your_username (on loginNode) <=> /vinai/your_username**

**SUPERPOD_STORAGE_DDN_FOLDER <=> /lustre/scratch/client (on all node)**

**PERSONAL_STORAGE_DDN_FOLDER <=> /lustre/scratch/client/vinai/user/your_username**

*You have to put your training data in DDN Storage, HOME ISILON will be used for data archive longterm*

## Introduction

Slurm is an open-source job scheduling system for Linux clusters, most frequently used for high-performance computing (HPC) applications. This guide will cover some of the basics to get started using slurm as a user. For more information, the [Slurm Docs](https://slurm.schedmd.com/documentation.html) are a good place to start.

After [slurm is deployed on a cluster](./README.md), a slurmd daemon should be running on each compute system. Users do not log directly into each compute system to do their work. Instead, they execute slurm commands (ex: srun, sinfo, scancel, scontrol, etc) from a slurm login node. These commands communicate with the slurmd daemons on each host to perform work.

## Simple Commands

### Cluster state with sinfo

To "see" the cluster, ssh to the slurm login node for your cluster and run the `sinfo` command:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST 
batch*       up 1-00:00:00      8   idle sdc2-hpc-dgx-a100-[001-008] 
batch*       up 1-00:00:00      2   down sdc2-hpc-dgx-a100-[013,015] 
```

There are 8 nodes available on this system, all in an `idle` state. If a node is busy, its state will change from `idle` to `alloc`.  If a node is down, its state will change from `idle` to `down`

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ sinfo -lN
Fri Jul 16 10:47:52 2021
NODELIST               NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT AVAIL_FE REASON               
sdc2-hpc-dgx-a100-001      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-002      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-003      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-004      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-005      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-006      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-007      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-008      1    batch*        idle 256    2:64:2 103100        0      1   (null) none                 
sdc2-hpc-dgx-a100-013      1    batch*        down 256    2:64:2 103100        0      1   (null) VinAI use            
sdc2-hpc-dgx-a100-015      1    batch*        down 256    2:64:2 103100        0      1   (null) VinAI use 
```

The `sinfo` command can be used to output a lot more information about the cluster. Check out the [sinfo doc for more](https://slurm.schedmd.com/sinfo.html).

### Running a job with srun

To run a job, use the `srun` command:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ srun --partition=batch hostname
sdc2-hpc-dgx-a100-001
```

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ srun --partition=batch --gres=gpu:8 env | grep CUDA
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
```

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ srun --partition=batch --ntasks 8 -l hostname
5: sdc2-hpc-dgx-a100-001
2: sdc2-hpc-dgx-a100-001
7: sdc2-hpc-dgx-a100-001
6: sdc2-hpc-dgx-a100-001
0: sdc2-hpc-dgx-a100-001
3: sdc2-hpc-dgx-a100-001
1: sdc2-hpc-dgx-a100-001
4: sdc2-hpc-dgx-a100-001
```

### Running an interactive job

Especially when developing and experimenting, it's helpful to run an interactive job, which requests a resource and provides a command prompt as an interface to it:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ srun --partition=batch --pty /bin/bash
dgxuser@sdc2-hpc-dgx-a100-001:~$ hostname
sdc2-hpc-dgx-a100-001
dgxuser@sdc2-hpc-dgx-a100-001:~$ exit
```

During interactive mode, the resource is being reserved for use until the prompt is exited (as shown above). Commands can be run in succession.

> Note: before starting an interactive session with srun it may be helpful to create a session on the login node with a tool like tmux or `screen`. This will prevent a user from losing interactive jobs if there is a network outage or the terminal is closed.

## More Advanced Use

### Run a batch job

While the `srun` command blocks any other execution in the terminal, `sbatch` can be run to queue a job for execution once resources are available in the cluster. Also, a batch job will let you queue up several jobs that run as nodes become available. It's therefore good practice to encapsulate everything that needs to be run into a script and then execute with `sbatch` vs with `srun`:

Example: run job python

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ cat demo.py
import time

a = 1
b = 2
print (a+b)

for i in range(15):
    print(i)
    time.sleep(0.5)
```

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ cat script.sh
#!/bin/bash
#SBATCH --job-name=demo            # create a short name for your job
#SBATCH --output=slurm_%j.out      # create a output file
#SBATCH --error=slurm_%j.err       # create a error file
#SBATCH --partition=batch          # choose partition
#SBATCH --gres=gpu:1               # gpu count
#SBATCH --ntasks=1                 # total number of tasks across all nodes
#SBATCH --nodes=1                  # node count
#SBATCH --mem-per-cpu=2G           # memory per cpu-core (4G is default)#SBATCH --mem=4G
#SBATCH --cpus-per-task=1          # cpu-cores per task (>1 if multi-threaded tasks)

python3 demo.py
echo   Date              = $(date)
echo   Hostname          = $(hostname -s)
echo   Working Directory = $(pwd)
echo   Number of Nodes Allocated      = $SLURM_JOB_NUM_NODES
echo   Number of Tasks Allocated      = $SLURM_NTASKS
echo   Number of Cores/Task Allocated = $SLURM_CPUS_PER_TASK
dgxuser@sdc2-hpc-login-mgmt001:~$ sbatch script.sh
```

### Resources can be requested several different ways:
| sbatch/srun Option | Description |
| --- | --- |
| `-N, --nodes=` | Specify the total number of nodes to request |
| `-n, --ntasks=` | Specify the total number of tasks to request |
| `--ntasks-per-node=` | Specify the number of tasks per node |
| `--gpus-per-node=` | Specify the number of GPUs to use Per node |
| `-G, --gpus=` | Total number of GPUs to allocate for the job |
| `--gpus-per-task=` | Number of gpus per task |
| `--gpus-per-node=` | Number of gpus to allocated per node |
| `--exclusive` | Guarantee that nodes are not shared amongst jobs|
### Observing running jobs with squeue

To see which jobs are running in the cluster, use the `squeue` command:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ squeue -a -l
Fri Jul 16 11:01:38 2021
JOBID PARTITION     NAME     USER    STATE       TIME TIME_LIMI   NODES NODELIST(REASON) 
125     batch       demo    dgxuser COMPLETI     0:09 1-00:00:00    1   sdc2-hpc-dgx-a100-001 
```

To see just the running jobs for a particular user `USERNAME`:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ squeue -l -u USERNAME
```

### Cancel a job with scancel

To cancel a job, use the `squeue` command to look up the JOBID and the `scancel` command to cancel it:

```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ squeue
dgxuser@sdc2-hpc-login-mgmt001:~$ scancel JOBID
```

### Running job with module

* List available modules
```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ module avail

----------------------------------------------------------------------------------------------------------------- /sw/modules/all ------------------------------------------------------------------------------------------------------------------
   mpi/3.0.6    python/2.7.18    python/3.6.10    python/3.8.10    python/miniconda3/miniconda3    python/pytorch/1.9.0+cu111    python/tensorflow/2.3.0

Use "module spider" to find all possible modules.
Use "module keyword key1 key2 ..." to search for all possible modules matching any of the "keys".
```

* Create your environment
```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ module load python/miniconda3/miniconda3
dgxuser@sdc2-hpc-login-mgmt001:~$ conda create -p /lustre/scratch/client/vinai/users/youruser/yourfolder python=yourversion
dgxuser@sdc2-hpc-login-mgmt001:~$ conda activate yourenv
````
* Installation your lib and packages you want
```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ cat conda.sh 
#!/bin/bash
#SBATCH --job-name=py-job        
#SBATCH --output=slurm_%A.out
#SBATCH --error=slurm_%A.err
#SBATCH --gres=gpu:1
#SBATCH --ntasks=4
#SBATCH --nodes=4
#SBATCH --tasks-per-node=1
#SBATCH --mem=4G
#SBATCH --cpus-per-task=1
#SBATCH --partition=batch

module purge
module load python/miniconda3/miniconda3
conda activate /lustre/scratch/client/vinai/users/youruser/yourfolder

command ...
```
Run sbatch
```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ sbatch conda.sh
````
### Running job with docker container
* List of available containers
```sh
tantnd/dc-pytorch                         1.6.0-python3.6-cuda11.0-arrow1.0-ubuntu18.04
quangbd/dc-tensorflow                     1.15.3-python3.6-cuda11.0-cudnn8-ubuntu18.04
quangbd/dc-miniconda                      3-cuda10.0-cudnn7-ubuntu18.04
quangbd/dc-miniconda                      3-cuda8.0-cudnn7-ubuntu16.04
quangbd/dc-tf-torch                       1.14.0-1.3.0-python3.6-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-python                         2.7-cuda9.0-cudnn7-ubuntu16.04
quangbd/dc-tensorflow                     1.15.0-python3.7-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-tensorflow                     1.14.0-python3.7-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-python                         3.6-cuda9.0-cudnn7-ubuntu16.04
quangbd/dc-python                         3.6-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-tf-torch                       1.15.0-1.4.0-python2.7-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-miniconda                      3-cuda10.1-cudnn7-ubuntu16.04
quangbd/dc-pytorch                        1.4.0-python3.7-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-miniconda                      3-cuda10.0-cudnn7-ubuntu16.04
quangbd/dc-tensorflow                     2.1.0-python3.7-cuda10.1-cudnn7-ubuntu16.04
quangbd/miniconda                         3-cuda10.0-cudnn7-ubuntu16.04
```
* You can run docker by example
```sh
dgxuser@sdc2-hpc-login-mgmt001:~$ cat container.sh
#!/bin/bash
#SBATCH --job-name=container-job        
#SBATCH --output=slurm_%A.out
#SBATCH --error=slurm_%A.err
#SBATCH --gres=gpu:2
#SBATCH --mem=48000MB
#SBATCH --cpus-per-gpu=8
#SBATCH --partition=batch

srun --partition=batch --gres=gpu:2 --cpus-per-gpu=8 --mem=48000MB \
                                        --container-image=yourimage \
                                        --container-mounts=host folder:container folder \
                                        command ...
```

* [ For more information ] (https://github.com/NVIDIA/pyxis/wiki/Usage)
### Running an MPI job

To run a deep learning job with multiple processes, use MPI:

```sh

```

## Additional Resources

* [SchedMD Slurm Quickstart Guide](https://slurm.schedmd.com/quickstart.html)
* [LLNL Slurm Quickstart Guide](https://hpc.llnl.gov/banks-jobs/running-jobs/slurm-quick-start-guide)
