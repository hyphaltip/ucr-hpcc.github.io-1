---
title: Managing Jobs
linkTitle: Managing Jobs
type: docs
weight: 3
aliases:
    - /manuals_linux-cluster_jobs.html
    - /manuals_linux-cluster_jobs
---

## What is a Job?
Submitting and managing jobs is at the heart of using the cluster.  A 'job' refers to the script, pipeline or experiment that you run on the nodes in the cluster.

## Partitions
In the past we used queues under the old Torque system, we now refer to these logically grouped nodes as partitions. There are several different partitions available for cluster users to send jobs to:

* intel
    * Default partition
    * Nodes: i01-02,i17-i40
    * Cores: Intel, 256 per user
    * RAM: 1 GB default
    * Time (walltime): 168 hours (7 days) default
* batch
    * Nodes: c01-c48
    * Cores: AMD, 256 per user
    * RAM: 1 GB default
    * Time (walltime): 168 hours (7 days) default
* highmem
    * Nodes: h01-h06
    * Cores: Intel, 32 per user
    * RAM: 100 GB min and 1000 GB max
    * Time (walltime): 48 hours (2 days) default
* gpu
    * Nodes: gpu01-gpu05
    * GPUs: 8 per group
    * RAM: 1 GB default
    * Time (walltime): 48 hours (2 days) default
* short
    * Nodes: Mixed set of nodes from batch, intel, and group partitions
    * Cores: AMD/Intel, 256 per user
    * RAM: 1 GB default
    * Time (walltime): 2 hours Maximum
* Group Partition
    * This partition is unique to the group, if your lab has purchased nodes then you will have a priority partition with the same name as your group (ie. girkelab).
In order to submit a job to different partitions add the optional '-p' parameter with the name of the partition you want to use:

```bash
sbatch -p batch SBATCH_SCRIPT.sh
sbatch -p highmem SBATCH_SCRIPT.sh
sbatch -p gpu SBATCH_SCRIPT.sh
sbatch -p intel SBATCH_SCRIPT.sh
sbatch -p mygroup SBATCH_SCRIPT.sh
```

## Slurm
Slurm is now our default queuing system across all head nodes. [SSH directly into the cluster](#getting-started) and your connection will be automatically load balanced to a head node:

```bash
ssh -XY cluster.hpcc.ucr.edu
```

### Resources and Limits
To see your limits you can do the following:

```bash
slurm_limits
```

Check total number of cores used by your group in the all partitions:

```bash
group_cpus
```

However this does not tell you when your job will start, since it depends on the duration of each job.
The best way to do this is with the "--start" flag on the squeue command:

```bash
squeue --start -u $USER
```

### Submitting Jobs
There are 2 basic ways to submit jobs; non-interactive, interactive. Slurm will automatically start within the directory where you submitted the job from, so keep that in mind when you use relative file paths.
Non-interactive submission of a SBATCH script:

```bash
sbatch SBATCH_SCRIPT.sh
```

Here is an example of an SBATCH script:

```bash
#!/bin/bash -l

#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=10
#SBATCH --mem=10G
#SBATCH --time=1-00:15:00     # 1 day and 15 minutes
#SBATCH --mail-user=useremail@address.com
#SBATCH --mail-type=ALL
#SBATCH --job-name="just_a_test"
#SBATCH -p intel # This is the default partition, you can use any of the following; intel, batch, highmem, gpu

# Print current date
date

# Load samtools
module load samtools

# Concatenate BAMs
samtools cat -h header.sam -o out.bam in1.bam in2.bam

# Print name of node
hostname
```

The above job will request 1 node, 10 cores (parallel threads), 10GB of memory, for 1 day and 15 minutes. An email will be sent to the user when the status of the job changes (Start, Failed, Completed).
For more information regarding parallel/multi core jobs refer to [Parallelization](#parallelization).

Interactive submission:

```bash
srun --pty bash -l
```

If you do not specify a partition then the intel partition is used by default.

Here is a more complete example:

```bash
srun --x11 --mem=1gb --cpus-per-task 1 --ntasks 1 --time 10:00:00 --pty bash -l
```

The above example enables X11 forwarding and requests, 1GB of memory, 1 cores, for 10 hours within an interactive session.

### Monitoring Jobs
To check on your jobs states, run the following:

```bash
squeue -u $USER --start
```

To list all the details of a specific job, run the following:

```bash
scontrol show job JOBID
```

To view past jobs and their details, run the following:

```bash
sacct -u $USER -l
```

You can also adjust the start `-S` time and/or end `-E` time to view, using the YYYY-MM-DD format.
For example, the following command uses start and end times:

```bash
sacct -u $USER -S 2018-01-01 -E 2018-08-30 -l | less -S # Type 'q' to quit
```

### Canceling Jobs
In cancel/stop your job run the following:

```bash
scancel <JOBID>
```

You can also cancel multiple jobs:

```bash
scancel <JOBID1> <JOBID2> <JOBID3>
```

If you want to cancel/stop/kill ALL your jobs it is possible with the following:

```bash
# Be very careful when running this, it will kill all your jobs.
squeue --user $USER --noheader --format '%i' | xargs scancel
```

For more information please refer to [Slurm scancel documentation](https://slurm.schedmd.com/scancel.html "Slurm scancel doc").

### Advanced Jobs
There is a third way of submitting jobs by using steps.
Single Step submission:

```bash
srun <command>
```

Under a single step job your command will hang until appropriate resources are found and when the step command is finished the results will be sent back on STDOUT. This may take some time depending on the job load of the cluster.
Multi Step submission:

```bash
salloc -N 4 bash -l
srun <command>
...
srun <command>
exit
```

Under a multi step job the salloc command will request resources and then your parent shell will be running on the head node. This means that all commands will be executed on the head node unless preceeded by the srun command. You will also need to exit this shell in order to terminate your job.

### Highmem Jobs
The highmem partition does not have a default amount of memory set, however it does has a minimum limit of 100GB per job. This means that you need to explicity request at least 100GB or more of memory.

Non-Interactive:

```bash
sbatch -p highmem --mem=100g --time=24:00:00 SBATCH_SCRIPT.sh
```

Interactive

```bash
srun -p highmem --mem=100g --time=24:00:00 --pty bash -l
```

Of course you should adjust the time argument according to your job requirements.

### GPU Jobs
GPU nodes have multiple GPUs, and very in type (K80 or P100). This means you need to request how many GPUs and of what type that you would like to use.

To request a gpu of any type, only indicate how many GPUs you would like to use.

Non-Interactive:

```bash
sbatch -p gpu --gres=gpu:1 --mem=100g --time=1:00:00 SBATCH_SCRIPT.sh
```

Interactive

```bash
srun -p gpu --gres=gpu:4 --mem=100g --time=1:00:00 --pty bash -l
```

Since the HPCC Cluster has two types of GPUs installed (K80s and P100s), GPUs can be requested explicitly by type.

Non-Interactive:

```bash
sbatch -p gpu --gres=gpu:k80:1 --mem=100g --time=1:00:00 SBATCH_SCRIPT.sh
sbatch -p gpu --gres=gpu:p100:1 --mem=100g --time=1:00:00 SBATCH_SCRIPT.sh
```

Interactive

```bash
srun -p gpu --gres=gpu:k80:1 --mem=100g --time=1:00:00 --pty bash -l
srun -p gpu --gres=gpu:p100:1 --mem=100g --time=1:00:00 --pty bash -l
```

Of course you should adjust the time argument according to your job requirements.

Once your job starts your code must reference the environment variable "CUDA_VISIBLE_DEVICES" which will indicate which GPUs have been assigned to your job. Most CUDA enabled software, like MegaHIT, will check this environment variable and automatically limit accordingly.

For example, when reserving 4 GPUs for a NAMD2 job:

```bash
echo $CUDA_VISIBLE_DEVICES
0,1,2,3
namd2 +idlepoll +devices $CUDA_VISIBLE_DEVICES MD1.namd
```

Each group is limited to a maximum of 8 GPUs on the gpu partition. Please be respectful of others and keep in mind that the GPU nodes are a limited shared resource.
Since the CUDA libraries will only run with GPU hardward, development and compiling of code must be done within a job session on a GPU node.

Here are a few more examples of jobs that utilize more complex features (ie. array, dependency, MPI etc):
[Slurm Examples](https://github.com/ucr-hpcc/hpcc_slurm_examples)

### Web Browser Access

#### Ports
Some jobs require web browser access in order to utilize the software effectively.
These kinds of jobs typically use (bind) ports in order to provide a graphical user interface (GUI) through a web browser.
Users are able to run jobs that use (bind) ports on a compute node.
Any port can be used on any compute node, as long as the port number is greater than 1000 and it is not already in use (bound).


#### Tunneling
Once a job is running on a compute node and bound to a port, you may access this compute node via a web browser.
This is accomplished by using 2 chained SSH tunnels to route traffic through our firewall.
This acts much like 2 runners in a relay race, handing the baton to the next runer, to get past a security checkpoint.

We will create a tunnel that goes though a headnode and connect to a compute node on a particular port:

```bash
ssh -NL 8888:NodeName:8888 username@cluster.hpcc.ucr.edu
```

Port 8888 (first) is the local port you will be using on your laptop.
NodeName is the compute node where where job is running, which can be found by using the `squeue -u $USER` command.
Port 8888 (second) is the remote port on the compute node.
Again, the NodeName and ports will be different depending on where your job runs and what port your job uses.

At this point you may need to provide a password to make the SSH tunnel.
Once this has succeeded, the command will hang (this is normal).
Leave this session connected, if you close it your tunnel will be closed.

Then open a browser on your local computer (PC/laptop) and point it to:

```
http://localhost:8888
```

If your job uses TSL/SSL, so you may need to try https if the above does not work:

```
https://localhost:8888
```

#### Examples

A perfect example of this method is used for Jupyter Lab/Notebook.
For more details please refer to the following [Jupyter Example](https://github.com/ucr-hpcc/hpcc_slurm_examples/tree/master/jupyter).

### Desktop Environments

#### VNC Server (cluster)

**Start VNC Server**

Log into the cluster:

```bash
ssh username@cluster.hpcc.ucr.edu
```

The first time you run the vncserver it will need to be configured:

```bash
vncserver -fg
```

You should set a password for yourself, and the read-only password is optional.

Then configure X Startup with the following command:

```bash
echo '/usr/bin/ssh-agent /usr/bin/dbus-launch --exit-with-session /usr/bin/gnome-session --session=gnome-classic' > /rhome/$USER/.vnc/xstartup
```

After your vncserver is configured, submit a vncserver job to get it started:

```bash
sbatch -p short,batch --cpus-per-task=4 --mem=10g --time=2:00:00 --wrap='vncserver -fg' --output='vncserver-%j.out'
```

> Note: Appropriate job resources should be requested based on the processes you will be running from within the VNC session.


Check the contents of your job log to determine the `NodeName` and `Port` you were assigned:

```bash
cat vncserver-*.out
```

The contents of your slurm job log should be similar to the following:

```bash
vncserver

New 'i54:1' desktop is i54:1

Creating default startup script /rhome/username/.vnc/xstartup
Starting applications specified in /rhome/username/.vnc/xstartup
Log file is /rhome/username/.vnc/i54:1.log
```

The VNC `Port` used should be 5900+N, N being the display number mentioned above in the format `NodeName`:`DisplayNumber` (ie. `i54:1`).
In this example (default), the port is `5901`, if this `Port` were already in use then the vncserver will automatically increment the DisplayNumber and you might find something like `i54:2` or `i54:3` and so on.

**Stop VNC Server**

To stop the vncserver, you can click on the logout option from the upper right hand menu from within your VNC desktop environment.
If you want to kill your vncserver manually, then you will need to do the following:

```bash
ssh NodeName 'vncserver -kill :DisplayNumber'
```

You will need to replace `NodeName` with the node name of your where your job is running, and the `DisplayNumber` with the DisplayNumber from your slurm job log.

#### VNC Client (Desktop/Laptop)

After you know the `NodeName` and VNC `Port` you should be able to create an SSH tunnel to your vncserver, like so:

```bash
ssh -N -L Port:NodeName:Port cluster.hpcc.ucr.edu
```

Now let us create an SSH tunnel on your local machine (desktop/laptop) using the `NodeName` and VNC `Port` from above:

```bash
ssh -L 5901:i54:5901 cluster.hpcc.ucr.edu
```

After you have logged into the cluster with this shell, log into the node where your VNC server is running:

```bash
ssh NodeName
```

After you have logged into the correct `NodeName`, just let this terminal sit here, do not close it.

Then launch vncviewer on your local system (laptop/workstation), like so:

```bash
vncviewer localhost:5901
```

After launching the vncviewer, and providing your VNC password (not your cluster password), you should be able to see a Linux desktop environment.

For more information regarding tunnels and VNC in MS Windows, please refer [More VNC Info](https://docs.ycrc.yale.edu/clusters-at-yale/access/vnc/).

### Licenses
The cluster currently supports [Commercial Software](/about/software/commercial/). Since most of the licenses are campus wide there is no need to track individual jobs. One exception is the Intel Parallel Suite, which contains the Intel compilers.

The `--licenses` flag is used to request a license for Intel compilers, for example:

```bash
srun --license=intel:1 -p short --mem=10g --cpus-per-task=10 --time=2:00:00 --pty bash -l
module load intel
icc -help
```

The above interactive submission will request 1 Intel license, 10GB of RAM, 10 CPU cores for 2 hours on the short partition.
The short parititon can only be used for a maximum of 2 hours, however for compilation this could be sufficient.
It is recommended that you separate your compilation job from your computation/analysis job.
This way you will only have the license checked out for the duration of compilation, and not the during the execution of the analysis.

## Parallelization
There are 3 major ways to parallelize work on the cluster:

   1. Batch
   2. Thread
   3. MPI

### Parallel Methods
For __batch__ jobs, all that is required is that you have a way to split up the data and submit multiple jobs running with the different chunks.
Some data sets, for example a FASTA file is very easy to split up (ie. fasta-splitter). This can also be more easily achieved by submitting an array job. For more details please refer to [Advanced Jobs](#advanced-jobs).

For __threaded__ jobs, your software must have an option referring to "number of threads" or "number of processors". Once the thread/processor option is identified in the software, (ie. blastn flag `-num_threads 4`) you can use that as long as you also request the same number of CPU cores (ie. slurm flag `--cpus-per-task=4`).

For __MPI__ jobs, your software must be MPI enabled. This generally means that it was compiled with MPI libraries. Please refer to the user manual of the software you wish to use as well as our documentation regarding [MPI](#mpi). It is important that the number of cores used is equal to the number requested.

In Slurm you will need 2 different flags to request cores, which may seem similar, however they have different purposes:

   * The `--cpus-per-task=N` will provide N number of virtual cores with locality as a factor.
     Closer virtual cores can be faster, assuming there is a need for rapid communication between threads.
     Generally, this is good for threading, however not so good for independent subprocesses nor for MPI.
   * The `--ntasks=N` flag will provide N number of physical cores on a single or even multiple nodes.
     These cores can be further away, since the need for physical CPUs and dedicated memory is more important.
     Generally this is good for independent subprocesses, and MPI, however not so good for threading.

Here is a table to better explain when to use these Slurm options:

Slurm Flag        | Single Threaded                   | Multi Threaded (OpenMP) | MPI only | MPI + Multi Threaded (hybrid) 
----------------- | --------------------------------- | ----------------------- | -------- | -----------------------------
`--cpus-per-task` |                                   |                      X  |          |                             X 
`--ntasks`        |                                   |                         |        X |                             X 

As you can see:

   1. A single threaded job would use neither Slurm option, since Slurm already assumes at least a single core.
   2. A multi threaded OpenMP job would use `--cpus-per-task`.
   3. A MPI job would use `--ntasks`.
   4. A Hybrid job would use both.

For more details on how these Slurm options work please review [Slurm Multi-core/Multi-thread Support](https://slurm.schedmd.com/mc_support.html).

#### MPI

MPI stands for the Message Passing Interface. MPI is a standardized API typically used for parallel and/or distributed computing.
The HPCC cluster has a custom compiled versions of MPI that allows users to run MPI jobs across multiple nodes.
These types of jobs have the ability to take advantage of hundreds of CPU cores symultaniously, thus improving compute time.

Many implementations of MPI exists, however we only support the following:

   * [Open MPI](http://www.open-mpi.org/)
   * [MPICH](http://www.mpich.org/)
   * [IMPI](https://software.intel.com/en-us/mpi-developer-guide-linux)

For general information on MPI under Slurm look [here](https://slurm.schedmd.com/mpi_guide.html).
If you need to compile an MPI application then please email support@hpcc.ucr.edu for assistance.

When submitting MPI jobs it is best to ensure that the nodes are identical, since MPI is sensitive to differences in CPU and/or memory speeds.
The `batch` and `intel` partitions are designed to be homogeneous, however, the `short` partition is a mixed set of nodes.
When using the `short` partition for MPI append the constraint flag for Slurm.

__Short Example__

Here is an example that shows how to ensure that your job will only run on `intel` nodes from the `short` partition:

```bash
sbatch -p short --constraint=intel myJobScript.sh
```


__NAMD Example__

To run a NAMD2 process as an OpenMPI job on the cluster:

1. Log-in to the cluster
1. Create SBATCH script

   ```bash
   #!/bin/bash -l

   #SBATCH -J c3d_cr2_md
   #SBATCH -p batch
   #SBATCH --ntasks=32
   #SBATCH --mem=16gb
   #SBATCH --time=01:00:00

   # Load needed modules
   # You could also load frequently used modules from within your ~/.bashrc
   module load slurm # Should already be loaded
   module load openmpi # Should already be loaded
   module load namd

   # Run job utilizing all requested processors
   # Please visit the namd site for usage details: http://www.ks.uiuc.edu/Research/namd/
   mpirun --mca btl ^tcp namd2 run.conf &> run_namd.log
   ```

1. Submit SBATCH script to Slurm queuing system

   ```bash
   sbatch run_namd.sh
   ```

__Maker Example__

OpenMPI does not function properly with Maker, you must use MPICH.
Our version of MPICH does not use the mpirun/mpiexec wrappers, instead use srun:

```bash
#!/bin/bash -l

#SBATCH -p intel
#SBATCH --ntasks=32
#SBATCH --mem=16gb
#SBATCH --time=01:00:00

# Load maker
module load maker/2.31.11

mpirun maker # Provide appropriate maker options here

```

## More examples

The range of differing jobs and how to submit them is endless:

    1. Singularity containers
    2. Database services
    3. Graphical user interfaces
    4. Etc ...

For a growing list of examples please visit [HPCC Slurm Examples](https://github.com/ucr-hpcc/hpcc_slurm_examples).

