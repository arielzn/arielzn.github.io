---
layout: post
title:  "Ubuntu server setup as standalone HPC machine"
date:   2017-01-10 01:01:01 +0200
categories: hpc server
---

Monako server
=============

Server setup
------------

-   \[\[Initial packages installation\]\]

<!-- -->

-   \[\[Environment modules\]\]

<!-- -->

-   \[\[Nvidia drivers and Cuda SDK\]\]

Packages built from source
--------------------------

-   \[\[GNU compilers\]\]

<!-- -->

-   \[\[OpenMpi libraries\]\]

<!-- -->

-   \[\[MVAPICH2\]\]

<!-- -->

-   \[\[FFTW3 libraries\]\]

<!-- -->

-   \[\[OpenBlas libraries\]\]

<!-- -->

-   \[\[GROMACS builds\]\]

<!-- -->

-   \[\[GSL library\]\]

<!-- -->

-   \[\[LAMMPS\]\]

<!-- -->

-   \[\[Slurm\]\]

Usage
-----

### Custom software with environment modules

Most of the compilers and different numerical libraries are configured
through the `module` command. This allow to easily choose different
versions of a given package or different mpi implementations for the
same compiler.

The main options of the module command are

    module avail               # to list all available modules you can load
    module load moduleName     # to load moduleName into your environment
    module unload moduleName   # to unload moduleName from your environment
    module purge               # to remove all loaded modules
    module list                # to list your currently loaded modules

### SLURM queuing system

To submit jobs

`sbatch scritp.sh`

To check queue status

`squeue`

To cancel a job

`scancel jobid`

To check the status of every node (only monako so far). The last line is
the CPUs info as Allocated / Idle / Other / Total

`sinfo`

### Example scripts to submit jobs

The time limit specified on the example scripts is not necessary and it
won’t have any effect on job’s priority. But if you can estimate roughly
how much time your job will take is useful to provide some value there
(at least the order of magnitude, hours, days, weeks…). That information
will be shown as a sharp ending time when checking the queue status.
Then, when the server is fully used, is handy to have an idea when the
enqueued jobs will start for sure.

**Script for a parallel mpi only job**

This script will work for an application build with OpenMPI gnu or intel
compilers.

    #!/bin/bash
    ##name of the job that will appear on the queue
    #SBATCH --job-name=mpi_job
    ## number of nodes don't change (will be always 1 since it's only monako we have)
    #SBATCH --nodes=1
    ## cores required (maximum 24)
    #SBATCH --ntasks-per-node=8
    ## exectuion time days-hours:minutes
    #SBATCH --time 7-0:00

    ### modules required ( the ones used to compile the code)
    module load mpi/....
    module load libs/....

    ## this is just to print information about running time afterwards
    echo "-----------------------------------------------------"
    echo "START_TIME           = `date +'%y-%m-%d %H:%M:%S %s'`"
    START_TIME=`date +%s`
    echo "-----------------------------------------------------"
    echo ""

    ### launch job

    mpirun  /path/to/binary

    ## calculation of running time
    echo "-----------------------------------------------------"
    echo "END_TIME (success)   = `date +'%y-%m-%d %H:%M:%S %s'`"
    END_TIME=`date +%s`
    echo "RUN_TIME (hours)     = "`echo "$START_TIME $END_TIME" | awk '{printf("%.4f",($2-$1)/60.0/60.0)}'`
    echo "-----------------------------------------------------"

\*

