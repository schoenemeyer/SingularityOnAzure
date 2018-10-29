# Singularity on Azure HPC VMs

## Introduction

In addition to docker and shifter there is a third approach for providing container solutions for HPC.
Singularity is a container model suited to HPC and it was created by Gergory M. Kurtzer.  This has been tested on an H16r VM running the CentOS 7.1 HPC image in the Azure marketplace.
I recommend to read the Singularity documentation in https://www.sylabs.io/docs/ that contains complete user guide. 

## Install Singularity

Since the CentOS repository does not contain a Singularity package, it must be built from source.

Follow the quick installation steps described in https://www.sylabs.io/guides/3.0/user-guide/quick_start.html#quick-installation-steps
Verify that the installation is completed


    thomas@thschonm:~/go/src/github.com/sylabs/singularity$singularity --version
    singularity version 3.0.0-274.g224e2e9


## Build an HPC container from a definition file

The container requires Intel MPI to be installed.  The installation is taken from the host system:

    cd /opt
    tar zcvf ~/intel.tgz intel

This is the centos definition file used:

    Bootstrap: yum
    OSVersion: 7
    MirrorURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/os/$basearch/
    Include: yum

    %runscript
    source /opt/intel/impi/5.1.3.223/bin64/mpivars.sh
    exec "$@"

    %files
    intel.tgz /opt/intel.tgz

    %environment
    export I_MPI_FALLBACK=0
    export I_MPI_FABRICS=shm:dapl
    export I_MPI_DAPL_PROVIDER=ofa-v2-ib0
    export I_MPI_DYNAMIC_CONNECTION=0
    export I_MPI_DAPL_TRANSLATION_CACHE=0

    %post
    yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl
    cd /opt
    tar zxf intel.tgz
    rm -rf intel.tgz
    mkdir -p /etc/rdma
    touch /etc/rdma/dat.conf

The HPC image has an updated and node-specific version of “dat.conf” to the version in the “rdma” package. To make sure the correct version is used, the dat.conf will be used from the host node using a bindpath.

The image required root user to build:

    export PATH=/shared/bin/singularity/bin:$PATH
    sudo singularity build centos7.simg centos.def

This creates the “centos7.simg” Singularity image.


## Testing MPI on the image

Singularity containers can be run directly with mpirun.  The container that has been created is setup to execute the application specified in the first parameter.  We can test that the Infiniband is working with the following command: 
    
    mpirun -np 2 \
        -genv I_MPI_DEBUG 6 -genv I_MPI_FALLBACK 0 \
        -genv I_MPI_FABRICS dapl -genv I_MPI_DAPL_PROVIDER ofa-v2-ib0 \
        -genv I_MPI_DYNAMIC_CONNECTION 0 -genv I_MPI_DAPL_TRANSLATION_CACHE 0 \
        ./centos7.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong

Comments on the MPI options:

- I_MPI_FALLBACK=0 makes sure that the specified fabric is used and will not fallback to anything else (i.e. TCP)
- I_MPI_DEBUG=6 provides verbose information from MPI – useful when diagnosing issues.
- I_MPI_DAPL_PROVIDER, I_MPI_FABRICS and I_MPI_DYNAMIC_CONNECTION settings are both required when using Infiniband on Azure 
- We should typically use “I_MPI_FABRICS=shm:dapl” but this will make sure dapl is used for our test.
- I_MPI_DAPL_TRANSLATION_CACHE=0 works around an Intel MPI bug (random but infrequent crashes with certain applications) 

## Testing Singularity on H16r with CentOS 7.4

https://github.com/schoenemeyer/SingularityOnAzure/blob/master/h16rcentos74.md



