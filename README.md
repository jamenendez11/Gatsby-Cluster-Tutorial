This is a tutorial for submitting jobs to Gatsby cluster with SLURM

# How to submit jobs to the Gatsby cluster

The Gatsby cluster is essentially a bunch of computers (called "**nodes**") hooked up together in a network. This is useful because when you need to run many simulations, for example, you can parallelize and run each simulation on a different node in the cluster.

The first practical problem is that many different people may want to run different programs ("**jobs**") at the same time. To make sure two people don't try to run two different programs on the same "node", the cluster uses a **job scheduler**, which does exactly what it sounds like: it schedules different jobs from different users. This means making sure to allocate each job to a different node and, if all the nodes are being used, putting any extra jobs that can't currently be run in a **queue** so that they are sequentially allocated to different nodes in some useful way (e.g. run all the short jobs first, then run all the long ones).

The job scheduler that Gatsby uses is called SLURM. This is a pretty widely used scheduler that has extensive [documentation](https://slurm.schedmd.com/documentation.html). Like any other job scheduler, SLURM requires that you **submit** jobs to the queue in a particular way. Learning this is basically all there is to it.

## Useful SLURM commands

* `squeue`: shows you the current queue, i.e. the jobs currently running and which nodes they are running on, and the jobs not yet running but on the queue
* `scancel`: remove a job from the queue, or cancel a job that is currently running. If the job number is `1234`, cancel it with `scancel 1234`. To cancel all the jobs submitted by user `myname`, use `scancel -u myname`.
* `sacct`: for job number `1234`, use `sacct -j 1234` to get some more information about it. See [here](https://slurm.schedmd.com/sacct.html) for other options to get additional information about the job that isn't given automatically. I like to use:
  ````
  sacct -j 1234 --format=JobName,ReqMem,MaxRSS,TimeLimit,Elapsed,End,NodeList;
  ````
  See the above link to get the meaning of each of these fields.

## Submitting a single job

Suppose you have a MATLAB script called `script.m` that you want to run on the cluster, and it is found in the directory `~/myhome/myfolder/`. Here is a script that would do this:
````
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=myjob_%A.out
#SBATCH --time=0-12:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=1000
#SBATCH --partition=compute

cd /nfs/nhome/live/myhome/myfolder/
srun -n 1 -c 1 /opt/matlab-R2014a/bin/matlab -nosplash -nodesktop -singleCompThread -r "script; exit"
````
Here's what it does, in order of each line of the code:
1. `job-name`: specifies the name under which the job will appear labelled in the queue (which can be viewed at any time using `squeue`)
2. `output`: specifies the name of the output file containing the text output from your program. In other words, SLURM will automatically save all text output from your program into a file called `myjob_%A.out` with `%A` replaced with the job number assigned to this job, which will be saved in the same folder in which this script was executed. For example, if your MATLAB script `script.m` is iteratively optimizing some objective function and spitting out the value of the objective function every 10 iterations, this text output from MATLAB will be saved into the `.out` file SLURM saves for you. More importantly, if MATLAB encounters an error and your job stops because of it, you can look at the `.out` file to see what the error message from MATLAB was.
3. `time`: specifies maximum amount of time you want SLURM to allow your job to run for. In this example, I have chosen 0 days, 12 hours, and 0 minutes. The longer amount of time you put here, the lower down the queue your job is going to be placed, since SLURM gives priority to shorter jobs.
4. `nodes`: specifies how many nodes to allocate to this job. Usually this will be set to 1, but if you are running things in parallel (e.g. using a `parfor` loop in MATLAB), you will want more (see [below](#parallel)).
5. `cpus-per-task`: specifies how many CPUs in this node to use. It turns out each node has many CPUs. I don't think you would ever set this to anything other than 1 unless you were running things in parallel (see [below](#parallel)). I think the maximum you can set this to is 20 on the Gatsby cluster.
6. `mem`: specifies the maximum amount of memory to be allocated to this job, in MB. Again, the larger you set this, the lower down in the queue your job will go (SLURM gives priority to shorter and cheaper jobs). In this example, I have chosen 1000MB = 1GB. In general I think there is no point in using less than this.
7. `partition`: the Gatsby cluster is a collection of nodes that are separated into two different "partitions": `compute` and `wrkstn`. Partition `compute` consists of a bunch of CPUs on some server. The `wrkstn` partition, on the other hand, consists of the CPUs on every desktop computer at Gatsby - when these aren't being used, one can run jobs on them through SLURM by setting `partition=wrkstn`. SLURM will automatically detect when they are not being used and allocate jobs appropriately.
8. The next line after all the SLURM specifications just changes directory to the folder in which your code is in.
9. The last line executes your MATLAB script by using the SLURM command srun, followed by specifications of number of nodes `-n` and CPUs `-c` to use (these should match up with the `--nodes` and `--cpus-per-task` options above), the location of MATLAB on the Gatsby network (`/opt/matlab-version/bin/matlab`), a couple of options for how to open MATLAB (`-nosplash` `-nodesktop` `-singleCompThread`), and finally the MATLAB code to actually be run (`script; exit`) in quotations

To execute this script so that SLURM recognizes it and submits this job to the queue, save it as e.g. `submit_job.sbatch` and then use
````
sbatch submit_job.sbatch
````
to execute it. Then run `squeue` to check that your job appears in the queue. It should appear under some job number, with the name you specified in `job-name`.


## Job arrays

Sometimes you want to run 100s of jobs differing by a particular parameter. One way of doing this is using **job arrays**. Suppose job `i` consists of running a MATLAB function `myfun(i)`, with argument set to `i`, for `i = 1,...,100`. For example, `i` might index a particular file that `myfun()` loads to set the parameters of a neural network it simulates. You can do this with the following `.sbatch` script:
````
#!/bin/bash
#SBATCH --job-name=myjobarray
#SBATCH --output=myjobarray_%A_%a.out
#SBATCH --time=0-12:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=1000
#SBATCH --partition=compute

cd /nfs/nhome/live/myhome/myfolder/
srun -n 1 -c 1 /opt/matlab-R2014a/bin/matlab -nosplash -nodesktop -singleCompThread -r "myfun(${SLURM_ARRAY_TASK_ID}); exit"
````
where `/myhome/myfolder/` is the folder in which `myfun.m` is located. There are two differences from above to note here:
* The text output files for each job `i=1,...,100` are saved as `myjobarray_%A_%a.out`, where `%A` is replaced by the job number assigned to the whole job array, and `%a` is replaced by the index of each individual job `i`. 
* You can include the individual job index `i` in your MATLAB code using `${SLURM_ARRAY_TASK_ID}`.

Let the above script be saved as `jobarray.sbatch`. To submit jobs `i=1,...,100` as a job array, you would then execute this `.sbatch` script using:
````
sbatch --array=1-100 jobarray.sbatch

````

## Parallel computing with MATLAB <a name="parallel"></a>

If you're using the Parralel Computing Toolbox in MATLAB (e.g. if you're using `parfor`), your job submission script will look slightly different. Namely, you will want to 
1. set the `--cpus-per-task` to the number of parallel "workers" you want to use (presumably the maximum possible, which I think is 20 on the Gatsby cluster), and accordingly change the `-c` option when calling MATLAB
2. remove the `-singleCompThread` option when calling MATLAB
So it will look something like this:
````
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=myjob_%A.out
#SBATCH --time=0-12:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=20
#SBATCH --mem=1000
#SBATCH --partition=compute

cd /nfs/nhome/live/myhome/myfolder/
srun -n 1 -c 20 /opt/matlab-R2014a/bin/matlab -nosplash -nodesktop -r "script; exit"
````
for a MATLAB script `~/myhome/myfolder/script.m` that uses parallel computing.

You also need to make sure that in your MATLAB script you open a parpool with the same number of workers as the `--cpus-per-task` setting.

## Submitting to SWC cluster
Need to be careful about a [few things](https://wiki.ucl.ac.uk/display/SSC/Matlab) with MATLAB. Also, you need to specify the partition, and the location of MATLAB.

## Accessing the Gatsby cluster remotely
To access the Gatsby cluster from your personal computer, you need to get into the Gatsby network. You can do this by establishing a remote connection using a protocol called SSH. This requires a series of simple steps:
1. Open the ‘Terminal’ app on your mac and run the following command:
  ````
  ssh user@gatsby.ucl.ac.uk
  ````
  where `user` is you Gatsby username. The terminal window will then ask for a password.

2. Once you have done that, you will be in the Gatsby network. The next step is to connect into a desktop computer (you can't submit to the Gatsby cluster if you're not on one of the computers in the network), again using SSH by 
running
  ````
  ssh u123b
  ````
  where u123b is the ID of the desktop computer. Again you will be asked for a password, which should be the same as above.

3. This will take you to `user`'s home folder. Now all you need to do is go to the folder where your `.sbatch` script is to submit your jobs:
  ````
  cd ~/myfolder/
  ````
  where cd is the linux command to **c**hange **d**irectory. Then, submit your jobs with `sbatch` as described above. You can check the progress of any of your jobs by simply running the command
  ````
  squeue
  ````
  and looking for your job in the queue. The queue will tell you whether or not the job is running (the column headed `ST`, for state: `R` = running, `PD` = pending) and if it is how long it has been running for.

## Downloading your data
If a job has completed and saved some data on your Gatsby machine, you can download this onto your personal computer using the command
````
scp user@gatsby.ucl.ac.uk:~/myfolder/datafile.mat ~/localpath/
````
where `user` is your Gatsby username, `myfolder/` is the folder in which the data is in, `datafile.mat` is the name of the data file you want to download, and `~/localpath/` is the path to the folder on your personal computer to which you want to download the data file. You'll be asked for the password and that's that.

If you want to download a whole folder of data files, however, you have to use the "recursive" option `-r`:
````
scp -r user@gatsby.ucl.ac.uk:~/myfolderofdata/ ~/localpath/
````
