This is a tutorial for submitting jobs to Gatsby cluster with SLURM

# How to submit jobs to the Gatsby cluster

The Gatsby cluster is essentially a bunch of computers (called "**nodes**") hooked up together in a network. This is useful because when you need to run many simulations, for example, you can parallelize and run each simulation on a different node in the cluster.

The first practical problem is that many different people may want to run different programs ("**jobs**") at the same time. To make sure two people don't try to run two different programs on the same "node", the cluster uses a **job scheduler**, which does exactly what it sounds like: it schedules different jobs from different users. This means making sure to allocate each job to a different node and, if all the nodes are being used, putting any extra jobs that can't currently be run in a **queue** so that they are sequentially allocated to different nodes in some useful way (e.g. run all the short jobs first, then run all the long ones).

The job scheduler that Gatsby uses is called SLURM. This is a pretty widely used scheduler that has extensive [documentation](https://slurm.schedmd.com/documentation.html). Like any other job scheduler, SLURM requires that you **submit** jobs to the queue in a particular way. Learning this is basically all there is to it.

**TABLE OF CONTENTS**
1. [Useful SLURM commands](#useful)
2. [Submitting jobs](#submit)
    * [Default settings](#default)
    * [Scripts with arguments](#arguments)
    * [Job arrays](#arrays)
3. [Details](#details)
    * [Python](#python)
    * [MATLAB](#matlab)
    * [Parallel computing with MATLAB](#parallel)


## Useful SLURM commands <a name="useful"></a>

* `squeue`: shows you the current queue, i.e. the jobs currently running and which nodes they are running on, and the jobs not yet running but on the queue
* `scancel`: remove a job from the queue, or cancel a job that is currently running. If the job number is `1234`, cancel it with `scancel 1234`. To cancel all the jobs submitted by user `myname`, use `scancel -u myname`.
* `sacct`: for job number `1234`, use `sacct -j 1234` to get some more information about it. See [here](https://slurm.schedmd.com/sacct.html) for other options to get additional information about the job that isn't given automatically. I like to use:
  ````
  sacct -j 1234 --format=JobName,ReqMem,MaxRSS,TimeLimit,Elapsed,End,NodeList;
  ````
  See the above link to get the meaning of each of these fields.


## Submitting jobs <a name="submit"></a>

Submitting a job has two steps:
1. Write a job submission script, which
    * specifies the settings for your job, using `#SBATCH` commands
    * specifies the script that you want this job to run
2. Submit the job using the `sbatch` command
Step 2 is straight-forward: given a job submission script called `submit_job.sbatch`, the job is submitted via
````
sbatch submit_job.sbatch
````
After submitting, run `squeue` to check that your job is now on the queue.

Step 1 is a little more involved. Job submission scripts have a particular structure, illustrated here in an example. Suppose you have a Python script called `script.py` that you want to run on the cluster, found in the directory `~/myfolder/`. Here is a script that would do this:
````
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=myjob_%A.out
#SBATCH --time=0-12:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=1000
#SBATCH --partition=cpu

cd ~/myfolder/
srun python -u script.py
````
Here's what it does, in order of each line of the code:
1. `job-name`: specifies the name under which the job will appear labelled in the queue (which can be viewed at any time using `squeue`)
2. `output`: specifies the name of the output file containing the text output from your program. SLURM will automatically save all text output from your program into a file called `myjob_%A.out` with `%A` replaced with the job number assigned to this job, which will be saved in the same folder in which this script was executed. For example, if `script.py` is iteratively optimizing some objective function and spitting out the value of the objective function every iteration, this text output from Python will be saved into the `.out` file SLURM saves for you. More importantly, if an error arises and your job stops because of it, you can look at the `.out` file to see what the error message was.
3. `time`: specifies maximum amount of time you want SLURM to allow your job to run for. In this example, I have chosen 0 days, 12 hours, and 0 minutes. The longer amount of time you put here, the lower down the queue your job is going to be placed, since SLURM gives priority to shorter jobs.
4. `nodes`: specifies how many nodes to allocate to this job. Usually this should be set to 1, but if you are running things in parallel (e.g. using a `parfor` loop in MATLAB), you will want more (see [below](#parallel)).
5. `cpus-per-task`: specifies how many CPUs in this node to use. It turns out each node has many CPUs. I usually set this to 1, but I'm not actually sure what the best way to set this is.
6. `mem`: specifies the maximum amount of memory to be allocated to this job, in MB. Again, the larger you set this, the lower down in the queue your job will go (SLURM gives priority to shorter and cheaper jobs). In this example, I have chosen 1000MB, which is equal to 1GB. In general, I believe there is no point in using less than this. You can also express this in units of GB by writing, `--mem=1G`.
7. `partition`: the Gatsby cluster currently has two "partitions": `cpu` and `gpu`, aptly named as the latter contains nodes with only GPUs on them. The old Gatsby cluster also had a partition called `wrkstn`, which consisted of the CPUs on every desktop computer at Gatsby. I believe this hasn't yet been incorporated to the new Gatsby cluster (formerly known as the SWC cluster). Make sure to set this appopriately so that your jobs get allocated to the right nodes (e.g. if you're using GPU code make sure you set `--partision=gpu`. You can see a list of all nodes and their partitions using `sinfo`.
8. The next line after all the SLURM specifications just changes directory to the folder in which your code is in.
9. The last line executes your script by using the SLURM command `srun` followed by the necessary code for running the script (see [below](#details)).

### Default settings <a name="default"></a>

These are the default settings I use, for no real reason other than empirical success:
````
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=3G
````

### Scripts with arguments <a name="arguments"></a>

Sometimes (or always) you might want to submit a job that requires running a script that takes some arguments. This can be accomodated as follows:
1. When using `sbatch`, use the `--export` option to define some variables, e.g. `ALPHA` and `BETA` via
````
sbatch --export=ALL,ALPHA=69,BETA=1.23 submit_job.sbatch
````
2. In your job submission script, include these variables as arguments to your script when using `srun`, e.g.
````
srun python -u script.py $ALPHA $BETA
````

A few notes: 
* In step 1, note that we start with `--export=ALL` before defining any variables. This is the default setting of the `--export` option, which forces all variables in your current workspace to be passed on to the environment in which the script is executed. You almost always want this to happen (e.g. if you are in a virtual environment), so don't forget to include this!
* In step 2, make sure to use the `$` sign at the start of each variable name. Otherwise, `ALPHA` and `BETA` will be treated as commands rather than variables.

### Job arrays <a name="arrays"></a>

Sometimes (or always) you want to run the same script 100s or 1000s of times. Nothing above prevents you from doing this, but SLURM has a built-in elegant way of doing this, called **job arrays**. Job arrays are run using `sbatch` the usual way, but with the additional use of the `--array` option, e.g.
````
sbatch --array=1-1000 submit_job.sbatch
````
will submit 1000 replications of this job.

A few notes:
* You can specify how each of these replications gets named by using the `--output` option as above, but now `%A` representing the number of the whole job *array* and `%a` representing the job number within the array, e.g.
```
#SBATCH --output=myjob_%A_%a.out
```
where `%a` would take on a value between 1 and 1000 in the example shown here.
* Sometimes, you might want to use the job number as an argument to your script. For example, you might want each of the 1000 job submissions to run the same function but with a different setting of a particular parameter. You might then provide the job number as an argument, and inside the function use this number to index the corresponding parameter value. This job number is saved as a variable called `SLURM_ARRAY_TASK_ID`, which can be passed as an argument to your script (along with any other arugments) as follows:
````
srun python -u script.py ${SLURM_ARRAY_TASK_ID}
````


## Details <a name="details"></a>

### Python <a name="python"></a>

* Note that above I always included the `-u` option to `python` when running a Python script. In my experience, I found that none of the text output from my script got saved into the `.out` file if I didn't include this option. This may have been fixed.
* Using **virtual environments**: this is easily accomodated by either activating the virtual environment before submitting the job, i.e. before running `sbatch`, or by activating it within the job submission script before the line executing `srun`.

### MATLAB <a name="matlab"></a>

I haven't used MATLAB in a while, but at least back when I did use it you had to specify some additional options for running a MATLAB script with `srun`. Here is an example:
```
srun /opt/matlab-R2014a/bin/matlab -nosplash -nodesktop -singleCompThread -r "script; exit"
```
A few notes:
* you need to identify which MATLAB to use and where it is located (the location in this example is probably not the right one anymore)
* the `-nosplash`, `-nodesktop`, `-singleCompThread`, and `-r` options are necessary to run any MATLAB code via `srun`
* note that the code inside the quotes `"` is raw MATALB code to be executed

### Parallel computing with MATLAB <a name="parallel"></a>

If you're using the Parralel Computing Toolbox in MATLAB (e.g. if you're using `parfor`), your job submission script will look slightly different. Namely, you will want to 
1. set the `--cpus-per-task` to the number of parallel "workers" you want to use (presumably the maximum possible, which I think is 20 on the Gatsby cluster)
2. remove the `-singleCompThread` option when calling MATLAB
Note that you need to make sure that in your MATLAB script you open a parpool with the same number of workers as the `--cpus-per-task` setting.
