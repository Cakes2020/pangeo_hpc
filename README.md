# Scripts and advice for running Pangeo with `dask-jobqueue` on NCI's Gadi, Pawsey's Zeus and CSIRO's Pearcey
These are the scripts that I (Dougie) use. They may be useful to you.

(Note that the NCI-recommended approach for using Pangeo on Gadi is outlined here: https://nci-data-training.readthedocs.io/en/latest/_notebook/prep/pangeo.html. The scripts and instructions in this repo describe an alternative approach where users can manage their own conda environment and scale clusters using `dask-jobqueue`)

## Prerequisites
Users will need to be able to log in to their system of interest. To use Gadi and Pawsey, users will need to be able to request resources under a project. 

New users to Gadi can sign up [here](https://my.nci.org.au/mancini/signup/0), but they will need to either join an existing project or propose a new project to be able to access NCI resources. Existing users can check their projects [here](https://my.nci.org.au/mancini/).

New users to Pawsey can apply [here](https://pawsey.org.au/supercomputing/).

Ideally, users will have a github account (it's free and easy to set up [here](https://github.com/join)), but this is not essential.

## Getting set up:
1. Log in to your system of choice:
	> Gadi: `ssh -Y <username>@gadi.nci.org.au`\
	> Zeus: `ssh -Y <username>@zeus.pawsey.org.au`\
	> Pearcey: `ssh -Y <username>@pearcey.hpc.csiro.au`

2. If you don't have conda installed or access to conda (try `which conda`), install it:  
	```
	wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
	chmod +x Miniconda3-latest-Linux-x86_64.sh
	./Miniconda3-latest-Linux-x86_64.sh
	```  
	You'll get prompted for where to install conda. The default is home, which may be quite limited for space. It may therefore be a good idea to instead use a different persistent location, e.g. `/g/data` on Gadi, `/group` on Zeus or Bowen storage on Pearcey.
	
	Note, to run the scripts in this repo `conda` will need to be initialised. When you first install conda you will be given the option to append some lines to your `.bashrc` that will initialise `conda` every time you log in. I recommend doing this. Otherwise, you'll have to initialise `conda` manually before progressing.
	
3. If there's any possibility you might edit the scripts in this repo and want to keep track of your edits using git, create a fork of this repo under your own github account by clicking on the `Fork` button on the top right of this page. Doing this will create a replica of this repo under your username at `https://github.com/<your_username>/pangeo_hpc.git`. If you don't have a github account and you don't want to create one, go to step 4.
	
4. Clone your fork of this repo to a location of your choice on Gadi, Zeus or Pearcey: go to the desired location and run 
	```
	git clone https://github.com/<your_username>/pangeo_hpc.git
	``` 
	(or if your using ssh keys: `git clone git@github.com:<your_username>/pangeo_hpc.git`)
	
	If you didn't create a fork, clone this repo directly: 
	```
	git clone https://github.com/csiro-dcfp/pangeo_hpc.git
	```

5. If you don't already have a pangeo-like conda environment (containing `jupyter`, `xarray`, `dask`...), create one using the `pangeo_environment.yml` file in this repo. This should only take a few minutes with a decent internet connection and and file system that supports lots of small files: 
	```
	conda env create -f pangeo_environment.yml
	```
	This will create a new conda environment called `pangeo`. If you wish to use a different name: 
	```
	conda env create --name <different_name> -f pangeo_environment.yml
	```

6. Activate your new `pangeo` environment and configure your Jupyter password (note, in a previous version of these instructions, you would have also installed and enabled a number of Jupyter labextensions at this point. This is no longer necessary with JupyterLab version 3): 
	```
	conda activate pangeo
	jupyter notebook --generate-config
	jupyter notebook password
	```
	and follow the prompts.
	
8. At this point, you're ready to submit a job to run your JupyterLab and Python instances. Once this job is running and you've accessed JupyterLab via your web browser (see below) you'll be able to request additional resources as a dask cluster (using `dask-jobqueue`). We can submit a job to run our JupyterLab instance using the relevant `start_jupyter_<system>.sh` script but it may require a little editing first:

	1. Open the relevant `start_jupyter_<system>.sh` file and edit the PBS/SLURM header information (the `#PBS`/`#SLURM` lines) to reflect your project (if relevant), required resources, etc. Remember these do not need to represent the total resources you require for the job you have planned because you will be able to request additional resources from within JupyterLab using `dask-jobqueue`. For interactive science work, I usually request few resources for a relatively long time, and then do compute-heavy reduction task(s) on shorter-term `dask-jobqueue` clusters. With this type of workflow, the resources you request in `start_jupyter_<system>.sh` need only reflect what is needed to handle the reduced data.

	You could now go ahead and submit your `start_jupyter_<system>.sh` script to the queue. However, for convenience I've also written a simple function for handling the submission of `start_jupyter_<system>.sh` and parsing instructions from the output file. This function receives some of the key job specifications as optional inputs so you don't have to edit the header on `start_jupyter_<system>.sh` every time you want to change any of these. It also receives the name of your pangeo-like conda environment as an input. You can append this function to your `.bashrc` by running: 
	```
	./instantiate_pangeo_function.sh
	```
	The `pangeo` function signature is:
	> Gadi: `pangeo walltime(02:00:00) ncpus(4) mem(16GB) project($PROJECT) pangeo_env_name(pangeo) notebook_directory(~)`\
	> Zeus: `pangeo time(02:00:00) cpus_per_task(4) mem-per-cpu(4GB) account($PAWSEY_PROJECT) pangeo_env_name(pangeo) notebook_directory(~)`\
	> Pearcey: `pangeo time(02:00:00) cpus_per_task(4) mem-per-cpu(6GB) pangeo_env_name(pangeo) notebook_directory(~)`
	
	where the defaults are given in brackets. For example, to run with the default settings, one would simply enter into their terminal:
	```
	pangeo
	```
	To specify a 2 hour job with 6 cpus, one would enter:
	```
	pangeo 02:00:00 6
	```

9. Run the `pangeo` function or submit `start_jupyter_<system>.sh` to the queue. For the former, instructions for setting up port forwarding to view your JupyterLab session and dask dashboard will be printed to your screen. For the latter, you'll have to parse them from the `jupyter_instructions.txt` file that will appear in the current directory. In both cases, the instructions will only appear once your jobs leaves the queue which may take a minute or so.

10. Follow the instructions to access your JupyterLab session via a web browser.

11. Do your science. As mentioned above, my typical workflow is to use `dask-jobqueue` to request and access resources for the "heavy-lifting" in my notebooks (e.g. reducing a large dataset down to a 1D or 2D field to plot). Examples of setting up a `dask-jobqueue` cluster are given in the [notebooks](https://github.com/csiro-dcfp/pangeo_hpc/tree/master/notebooks) directory of this repo. 

	Note that getting `dask-jobqueue` running on Gadi requires the manipulation of the default jobscripts submitted by dask's `PBSCluster` into a format that Gadi expects. An example of this hack is given in `notebooks/run_dask-jobqueue_Gadi.ipynb`. 

## Optional - add an R kernel and packages:
1. Create a new conda environment with some essential packages for working with R. You can also add any other packages you use that aren't included in `r-essentials` here. Deactivate your pangeo environment and then:
	```
	conda create -n r_env -c r r-essentials r-vars
	conda activate r_env
	conda install -c conda-forge r-raster r-matlab r-ncdf4 r-lmtest r-cowplot
	```

2. Register the R kernel with Jupyter:
	```
	Rscript -e 'IRkernel::installspec()'
	```

Now when you spin up JupyterLab you should be able to select and use your R kernel. 
