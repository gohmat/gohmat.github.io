---
layout: post
title: Building XMDS from source on the NCI
date: 2020-02-03 11:37:00
description: A guide to compiling XMDS with all the bells and whistles on the Gadi supercomputer at the National Computational Infrastructure (NCI).
tags: software HPC XMDS
categories: computing
---
Installing XMDS on a personal computer running Linux or Mac OS is simple (see [this guide](http://www.xmds.org/installation.html)). Unfortunately, getting it running on the NCI is a more involved task. This guide details the build/installation process for XMDS2 3.0.0 on the Gadi supercomputer at the NCI, but may be helpful for later versions or other clusters.

Begin by connecting to Gadi via SSH (substitute your own username):
{% highlight shell %}
ssh ab1234@gadi.nci.org.au
{% endhighlight %}
and enter your password. Download the source for XMDS:
{% highlight shell %}
wget https://sourceforge.net/projects/xmds/files/xmds-3.0.0.tar.gz
{% endhighlight %}
then extract and open the new directory:
{% highlight shell %}
tar -xvf xmds-3.0.0.tar.gz
cd xmds-3.0.0
{% endhighlight %}
Now build XMDS by running the following commands (substituting all uses of 123 with your own group ID, and all uses of ab1234 with your own username):
{% highlight shell %}
module load python3/3.7.4
module load hdf5/1.10.5
module load szip/2.1.1
module load fftw3/3.3.8
module load openmpi/3.1.4

export install_dir=/home/123/ab1234/xmds/3.0.0

mkdir -p ${install_dir}/lib/python3.7/site-packages 

export PYTHONPATH=/home/123/ab1234/xmds/3.0.0/lib/python3.7/site-packages:$PYTHONPATH

./setup.py develop --prefix=${install_dir}

export LD_LIBRARY_PATH=${MKL}/../compiler/lib/intel64/:$LD_LIBRARY_PATH
export LIBRARY_PATH=${MKL}/../compiler/lib/intel64/:$LIBRARY_PATH
{% endhighlight %}
You will now need to reconfigure a few things by editing the file (substitute 123 with your group ID and ab1234 with your username) `/home/123/ab1234/xmds-3.0.0/xpdeint/support/wscript`. If you are comfortable with command-line text editors (e.g. emacs), you should edit the file using one. Otherwise, download `wscript` using FileZilla (or your preferred SFTP method) and open it in your preferred text editor on your own machine. Locate the following lines in `wscript` and make the following changes (comment out the existing lines marked `#aab`, and for the `lib` definitions, add the line below):
{% highlight shell %}
# If we have a static library marker, try to link the simulation statically for performance.
        if conf.env['STLIB_MARKER']:
            #aab conf.env['FINAL_LINK_VAR'] = conf.env['STLIB_MARKER']
            result = conf.check_cxx(
            . . .
{% endhighlight %}
{% highlight shell %}
check_cxx(
            #aab lib=["iomp", "vml"],
            lib=['mkl_intel_lp64', 'mkl_intel_thread', 'mkl_core', 'iomp5', 'mkl_vml_avx'],
            header_name='mkl.h',
            uselib_store='mkl_vsl',
            msg = "Checking for Intel's Vector Math Library",
            . . .
{% endhighlight %}
{% highlight shell %}
# Find CBLAS
        cblas_options = [
            {# Intel MKL
                'defines': 'CBLAS_MKL',
                #aab 'lib': ['mkl_intel_lp64', 'mkl_intel_thread', 'mkl_core'],
                'lib': ['mkl_intel_lp64', 'mkl_intel_thread', 'mkl_core', 'iomp5'],
                'fragment': '''
                . . .
{% endhighlight %}
{% highlight shell %}
options = [
            #aab dict(stlib=lib, msg=KWs['msg'] + " (static library)", errmsg="no (will try dynamic library instead)", **extra_static_kws),
            dict(lib=lib,   msg=KWs['msg'] + " (dynamic library)", **extra_shared_kws),
        ]
{% endhighlight %}
If you're using a command-line text editor, save these changes and proceed - if you're editing it on your own machine, save it on your machine, then re-upload it to Gadi, replacing the old wscript file. Now reconfigure XMDS by running the following command on Gadi:
{% highlight shell %}
${install_dir}/bin/xmds2 --reconfigure
{% endhighlight %}
Read through the log to make sure everything's configured properly (if it reports being unable to find a required package, you have a problem). If all went well, you've successfully built XMDS from source for Gadi! Your binary is located at `/home/123/ab1234/xmds/3.0.0/bin/xmds2`, substituting 123 for your group ID and ab1234 for your own username.

# Using XMDS on Gadi
There are two parts to using XMDS: running the `xmds2` binary on your `.xmds` file to generate executable code, then running that executable code. 
The first part can be done via the command line on the Gadi head node. Connect to Gadi via SSH. You will need to set your PYTHONPATH, and depending on what features you are using, load some or all of the relevant modules. If in doubt, run all the following commands:
{% highlight shell %}
module load python3/3.7.4
module load hdf5/1.10.5
module load szip/2.1.1
module load fftw3/3.3.8
module load openmpi/3.1.4

export PYTHONPATH=/home/123/ab1234/xmds/3.0.0/lib/python3.7/site-packages:$PYTHONPATH

export LD_LIBRARY_PATH=${MKL}/../compiler/lib/intel64/:$LD_LIBRARY_PATH
export LIBRARY_PATH=${MKL}/../compiler/lib/intel64/:$LIBRARY_PATH
{% endhighlight %}
Now place the `xmds2` binary you built previously and your `.xmds` file in the same directory, and navigate to this directory. Then run XMDS on this file:
{% highlight shell %}
./xmds2 filename.xmds
{% endhighlight %}
If it runs successfully, XMDS should generate an executable (usually named `filename` unless you've specified otherwise in your `.xmds` file).

Running the executable differs from the usual process you would follow on your own computer, since Gadi will automatically kill any computationally expensive processes being run on the head node. You must queue it using `qsub` (see the [NCI documentation](https://opus.nci.org.au/display/Help/How+to+submit+a+job) if you are unfamiliar). Once again, your job scripts may require you to load modules and set PYTHONPATH - just include those lines in your job script if in doubt. The following is an example of a job script for an XMDS binary:
{% highlight shell %}
#!/bin/bash
#PBS -P kl43
#PBS -q normal
#PBS -l walltime=1:00:00,mem=8GB,ncpus=64,jobfs=2GB
#PBS -l wd
#PBS -M matt.goh@anu.edu.au
#PBS -m abe
module load python3/3.7.4
module load hdf5/1.10.5
module load szip/2.1.1
module load fftw3/3.3.8
module load openmpi/3.1.4
export PYTHONPATH=/home/123/ab1234/xmds/3.0.0/lib/python3.7/site-packages:$PYTHONPATH
mpirun -np 64 ./singleparticlemeasurement --k_ED 4 --c1 0 --c2 0 --switchtime 0 --r 0.1 --alpha 0.05 --simulationlength 50
{% endhighlight %}
which, if saved as `jobscript`, would be queued by running:
{% highlight shell %}
qsub jobscript
{% endhighlight %}
Note that this job script executes the binary using `mpirun`: only do this if you have enabled MPI in your `.xmds` script.