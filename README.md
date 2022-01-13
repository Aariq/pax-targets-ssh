# pax-targets-ssh

<!-- badges: start -->

<!-- badges: end -->

This is a minimal example of a [`targets`](https://docs.ropensci.org/targets/) workflow that can run on the [Tufts University HPC](https://it.tufts.edu/high-performance-computing) using the [`clustermq`](https://mschubert.github.io/clustermq/) backend for paralellization. This is intended to be run **locally** in RStudio. For an alternative workflow that is run *on the HPC* via the commandline, see [this repo](https://github.com/BrunaLab/hipergator-targets).

## Setup
0. All of this works best if you can SSH into the cluster without a password.  You'll need to be on the Tufts VPN to avoid 2FA, and set up an SSH key with `ssh-keygen`.
1. SSH into the cluster and start an interactive session with `srun -p interactive --pty bash`.  Launch R and install the `clustermq` package. 
2. On the cluster, edit your `~/.Rprofile` (e.g. with `nano ~/.Rprofile`) to include:

```r
options(
  clustermq.scheduler = "slurm",
  clustermq.template = "~/slurm_clustermq.tmpl"
)
```
3. Edit the `slurm_clustermq.tmpl` file if needed (but don't touch the wildcards in double curly braces) and then copy it to the cluster with `scp slurm_clustermq.tmpl username@xfer.cluster.tufts.edu:slurm_clustermq.tmpl`

[More detailed instructions on setting up `clustermq`](https://mschubert.github.io/clustermq/articles/userguide.html)

## Run `targets` workflow

To run this example workflow, you can either run `targets::tar_make_clustermq()` in the console, or use the "Jobs" feature of RStudio to run the `start_job.R` script as a local job---this keeps the console from being tied up waiting for the jobs to run on the cluster.  You can watch the progress of the pipeline with `tar_watch()` or by running `tar_visnetwork()` or `tar_progress()` occasionally.

## Notes

You will also need to be sure that all the packages your project uses are installed on the cluster.

If you only want certain targets to run on Hipergator, you can control this with the `deploy` argument to `tar_options_set()` and `tar_target()`.  Targets with `deploy = "main"` will  run locally and targets with `deploy = "worker"` will run on Hipergator. Set the default behavior with `tar_options_set()` inside of `_targets.R` and then adjust individual targets as needed.  See `_targets.R` for an example and see the [`targets` manual](https://books.ropensci.org/targets/hpc.html#advanced) for more detail. Note that you will still have to wait for remote targets to finish running for the pipeline to finish, so if any targets take a very long time to run, [this alternative approach](https://github.com/BrunaLab/hipergator-targets) might be better.

Currently (as of 10-18-2021) there is a bug with R version 4.1+ on HiperGator that affects this workflow.  To get around this, I've edited the `ssh_clustermq.tmpl` and `slurm_clustermq.tmpl` files to load R version 4.0.

## Troubleshooting:

Problems with Tufts cluster: 

- https://it.tufts.edu/high-performance-computing/hpc-welcome-page

Problems with `clustermq`:

- https://mschubert.github.io/clustermq/articles/userguide.html#ssh

Problems with `targets` (i.e. problem still exists with `tar_make()` instead of `tar_make_clustermq()`:

- https://docs.ropensci.org/targets/index.html#help

## How it works:

Using `tar_make_clustermq()` sends necessary data to the cluster via SSH and spawns worker jobs using the `slurm_clustermq.tmpl` file as a template for the SLURM submission scripts for each worker.

The parallelization happens at the level of targets.
In this example, a list of numeric vectors is stored as `many_vects`.
Then, independently, means and standard deviations are calculated for each vector in the list.
These two targets (the means and the sd's) should be able to run on separate workers in parallel if things are set up correctly.
Parallelizing code *within* a target (e.g. a function that does parallel computation) will require more setup.
