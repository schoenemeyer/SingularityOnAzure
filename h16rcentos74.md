# Singularity running on H16r with CentOS HPC 7.4
This project describes how to install Singularity and run the Intel MPI pingpong benchmark within the Singularity container.

Create a VM of type H16r in the Azure Portal or with Azure CLI with CentOS HPC 7.4 

Connect to the machine and update the VM.
sudo yum -y update

# Install Singularity

Install required libraries
```
sudo yum -y install git
git clone https://github.com/singularityware/singularity.git
cd singularity
sudo yum -y install automake autoconf libtool
sudo yum -y install libarchive-devel
sudo yum -y install squashfs-tools
```
Configure and make
```
./configure --prefix=/usr/local
make
sudo make install
```
Full details can be found on the Singularity site .https://singularity.lbl.gov/install-linux

# Build an HPC container from a definition file
The container requires Intel MPI to be installed.  The installation is taken from the host system:
cd /opt
tar zcvf ~/intel.tgz intel
```
-rw-rw-r--.  1 thomas thomas 67891717 Jul  9 19:34 intel.tgz
```
cd

Create a file named centos.def with the following content:

```
Bootstrap: yum
OSVersion: 7
MirrorURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/os/$basearch/
Include: yum

%runscript
source /opt/intel/impi/5.1.3.223/bin64/mpivars.sh
exec "$@"

%files
intel.tgz /opt/intel.tgz
/etc/rdma/dat.conf /opt/dat.conf

%post
yum install -y tar gzip libmlx4 librdmacm libibverbs dapl rdma net-tools numactl
cd /opt
tar zxf intel.tgz
cp /opt/dat.conf /etc/rdma/dat.conf
rm /opt/dat.conf
```

The image requires root user to build:
```
sudo /usr/local/bin/singularity build centos7.simg centos.def

..
...
Complete!
+ cd /opt
+ tar zxf intel.tgz
+ cp /opt/dat.conf /etc/rdma/dat.conf
+ rm /opt/dat.conf
Adding runscript
Finalizing Singularity container
Calculating final size for metadata...
Skipping checks
Building Singularity image...
Singularity container built: centos7.simg
Cleaning up...
[thomas@h16r71Centos ~]$
```
After finishing the build you will get the new Singularity image as shown below “centos7.simg” Singularity image
```
total 883912
-rw-rw-r--.  1 thomas thomas  67891717 Jul  9 19:34 intel.tgz
-rw-rw-r--.  1 thomas thomas       429 Jul  9 19:39 centos.def
drwxrwxr-x. 17 thomas thomas      4096 Jul  9 19:46 singularity
-rwxr-xr-x.  1 root   root   300392479 Jul  9 19:52 centos7.simg
[thomas@h16r71Centos ~]$

```

# Testing the Singularity HPC container

source /opt/intel/impi/5.1.3.223/intel64/bin/mpivars.sh

Singularity containers can be run directly with mpirun.  The container that has been created is setup to execute the application specified in the first parameter.  We can test that the Infiniband is working with the following command: 
```
mpirun -np 2 \
-genv I_MPI_DEBUG 6 -genv I_MPI_FALLBACK 0 \
-genv I_MPI_FABRICS dapl -genv I_MPI_DAPL_PROVIDER ofa-v2-ib0 \
-genv I_MPI_DYNAMIC_CONNECTION 0 -genv I_MPI_DAPL_TRANSLATION_CACHE 0 \
./centos7.simg /opt/intel/impi/5.1.3.223/bin64/IMB-MPI1 PingPong
```

```
Benchmarking PingPong
processes = 2
#---------------------------------------------------
       #bytes #repetitions      t[usec]   Mbytes/sec
            0         1000         2.03         0.00
            1         1000         2.04         0.47
            2         1000         2.05         0.93
            4         1000         2.04         1.87
            8         1000         2.04         3.74
           16         1000         2.51         6.09
           32         1000         1.57        19.46
           64         1000         1.58        38.55
          128         1000         1.64        74.41
          256         1000         1.77       138.05
          512         1000         1.94       251.50
         1024         1000         2.30       424.50
         2048         1000         3.06       638.90
         4096         1000         4.53       862.79
         8192         1000         5.88      1327.98
        16384         1000         7.49      2086.54
        32768         1000        10.97      2847.91
        65536          640        17.24      3625.98
       131072          320        31.05      4026.35
       262144          160       447.98       558.05
       524288           80       563.45       887.39
      1048576           40       818.04      1222.44
      2097152           20      1263.03      1583.50
      4194304           10      2153.10      1857.79
      
```








