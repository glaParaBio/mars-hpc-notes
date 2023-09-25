<!-- vim-markdown-toc GFM -->

* [General](#general)
    * [Login to mars without typing password](#login-to-mars-without-typing-password)
    * [Transfer files between local and mars](#transfer-files-between-local-and-mars)
* [Writing and editing scripts](#writing-and-editing-scripts)
* [Setting up project environment](#setting-up-project-environment)
    * [Install conda/mamba package manager](#install-condamamba-package-manager)
    * [Configure bioconda](#configure-bioconda)
    * [Creating a project environment](#creating-a-project-environment)
* [Slurm scheduler](#slurm-scheduler)
    * [Submit jobs](#submit-jobs)
    * [Interactive jobs](#interactive-jobs)
    * [Check and cancel jobs](#check-and-cancel-jobs)
* [Where to write output](#where-to-write-output)
* [Job dependencies](#job-dependencies)

<!-- vim-markdown-toc -->

These are scattered notes for working on mvls-mars HPC and more in general for
managing bioinformatics projects.

NB: This notes are likely to change, become obsolete and definitely they are
incomplete. They assume your PC has a Unix system (MacOS or Linux).

# General

* [General information about mars](https://mars.ice.gla.ac.uk/)

* [Storage and data](https://mars.ice.gla.ac.uk/storage-and-data/)

## Login to mars without typing password

If you have not used ssh before on your local PC, execute (default options are ok):

```
ssh-keygen
```

Copy ssh key to mars (replace db291g with your mars username):

```
ssh-copy-id db291g@mars-login.ice.gla.ac.uk
```

From now on you shouldn't need to type your password anymore.

For convenience, add these lines to `~/.bashrc` of your PC (not mars):

```
mars=db291g@mars-login.ice.gla.ac.uk
alias mars="ssh -X $mars"
```

Now you can login to mars just by typing:

```
mars
```

## Transfer files between local and mars

```
rsync -arvP /some/local/file $mars:sharedscratch/
```

To *move* files from mars to local (I.e. remove from mars after transfer):

```
rsync -arvP --remove-source-files $mars:~/sharedscratch/somefile.txt ./
```

`rsync` also has useful `--dry-run\-n` optioni and options to include/exclude
file patterns, among many other things.

# Writing and editing scripts

* Option 1 (my choice): Use `vim` directly on mars 

* Option 2: Many text editors can connect to a remote server to edit files as
  if they were on your local PC. E.g. use [vscode](https://code.visualstudio.com/)
  with plugin
  [remote-ssh](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh).


# Setting up project environment

In my opinion, it is good practice to setup an isolated environment with all
the required programs for each project you work on. This way you have more
control and flexibility than asking IT to install stuff.

## Install [conda/mamba](https://mamba.readthedocs.io/en/latest/index.html) package manager

Login to mars and execute:

```
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-$(uname)-$(uname -m).sh"
bash Mambaforge-$(uname)-$(uname -m).sh
```

follow the instructions on the interactive terminal. Default options are fine
except when asked where to install mamba you may want to replace the default
`/users/db291g/mambaforge` with:

```
/users/db291g/sharedscratch
```

(replace db291g with your username) 

## Configure [bioconda](https://bioconda.github.io/)

See https://bioconda.github.io/ for details, but this should do:

```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

## Creating a project environment

When starting a new project, for example [apolloDemoData](https://github.com/glaParaBio/apolloDemoData):

* Create a dedicated environment with a name you like:

```
mamba create --yes --name apolloDemoData
```

* Write a text file listing all the dependencies one per line, call this file e.g. [requirements.txt](https://github.com/glaParaBio/apolloDemoData/blob/master/requirements.txt):

```
snakemake =7.32.3
samtools =1.17
hisat2 =2.2.1
deeptools =3.5.3
bedtools =2.31.0
blast
```

Specifying program versions is optional but recommended. Usually googling
*some-program conda* will lead you to the exact name you need, e.g. *ggplot
conda* should get https://anaconda.org/conda-forge/r-ggplot2 on the top hits.

If a program is not on (bio)conda you can always isntall it with traditional
methods in the dedicated environment.

* Install dependencies

```
mamba install -n apolloDemoData --yes --file requirements.txt
```

if you need to add more programs later on, just edit
`requirements.txt` and run `mamba install ...` again.

* Activate the environment: before working on a project activate its environment with:

```
mamba activate apolloDemoData
```

`requirements.txt` should be all you need to recreate an environment on a
different server. Consider creating a GitHub repository for your project(s) and
add `requirements.txt` to it together with your scripts.

# Slurm scheduler

## Submit jobs

Something like:

```
sbatch --cpus-per-task=10 --mem=10G --parsable --time=08:00:00 -o slurm/%x.%j.out -e slurm/%x.%j.err my-scripts.sh -J myjob
```

will run `my-scripts.sh` on a node with at least 10 CPUs available and 10 GB of
memory to. Standard output and errors will go to `slurm/%j.out` and
`slurm/%j.err` where `%j` will be replaced by the jobid and `%x` with the job
name (NB: directory `slurm` must exists before starting the job).

## Interactive jobs

Sometimes it is useful to login to a working node to experiment and
troubleshoot (the login in node has small memory and it should not be used for
running resource intensive programs). Execute this to spawn an interactive job
that will last up to 8 hours:

```
srun --nodes=1 --ntasks-per-node=1 --time=08:00:00 --pty bash -i
```

You may want to add this line to your `~/.bashrc` file on mars so you don't have to remember all that:

```
alias irun='srun --nodes=1 --ntasks-per-node=1 --time=08:00:00 --pty bash -i'
```

now you can start an interactive job just by executing:

```
irun
```

Unrelated, but once we are at it: add this line to `~/.bashrc` to enable
downloading from ftp site (e.g. from ENA archive):

```
export ftp_proxy=ftp://wwwcache.gla.ac.uk:8080
```

## Check and cancel jobs

To check jobs:

```
sacct 

# For lots of detail:
sacct -l

# For parsable, tab-delimited
sacct -P --delimiter $'\t'
```

To cancel jobs:

```
scancel 1234 1235 # Where 1234 is the job ID from sacct
```

# Where to write output

Most likely you want `/users/db291g/sharedscratch` since it is a large
partition and all nodes have access to it. The home directory has only 20GB of
space so don't use it for anything other than configuration files and small programs.

# Job dependencies

I.e. *start this job after another one has completed*. I use [snakemake](https://snakemake.github.io/) for this, regardless of using
an HPC. It has a learing curve but if you regularly work on bioinformatics
projects it is worth it. Ask for more information...
:
