#  AWS ParallelCluster, Slurm, Spack, and WRF4.2.2 on China Region c6g instance

作者：Sean Yang
邮件：[zhihay@amazon.com](mailto:zhihay@amazon.com)
欢迎指教，指错和探讨！

在跳板机上按照如下修改配置文件.parallelcluster/config

```
[aws]
aws_region_name = cn-north-1

[cluster wrf4-c6g]
key_name = sean-beijing-key
base_os = alinux2
# custom_ami= ami-0e7631acb5ae1dbda # parallelcluster2.10.3在中国区支持
master_instance_type = c6g.16xlarge # 在master上编译相关依赖需要大量的CPU，内存资源，建议配置大些
compute_instance_type = c6g.16xlarge
cluster_type = ondemand
initial_queue_size = 2
scheduler = slurm
placement_group = DYNAMIC
vpc_settings = defaultvpc
ebs_settings = ebsset

[vpc defaultvpc]
vpc_id = vpc-ba8fc4de
master_subnet_id = subnet-0ece5f9804f991e1d

[ebs ebsset]
shared_dir = shared
volume_type = gp2
volume_size = 2000

[global]
cluster_template = wrf4-c6g
update_check = true
sanity_check = true
```


创建pcluster

```
$ pcluster create wrf4-c6g
```


`登录到master节点`

```
$ pcluster ssh wrf4 -i sean-beijing-key.pem
```

spack是为HPC的包管理器，spack可以自动下载包，依赖包和调优，以下是安装spack的步骤

```
`$ cd /shared  # install to shared disk`
$ `wget https://wrf-graviton2.s3.cn-north-1.amazonaws.com.cn/wrf/spack-0.16.0.tar.gz`
`# 可选择海外github下载地址：https://github.com/spack/spack/releases/download/v0.16.1/spack-0.16.1.tar.gz`
`$ tar zxvf spack-0.16.1.tar.gz`
`$ mv spack-0.16.1 spack`
`$ echo 'export PATH=/shared/spack/bin:$PATH' >> ~/.bashrc  # to discover spack executable`
`$ source ~/.bashrc`
```

检查spack安装成功和版本

```
$ spack --version
0.16.1
```

查看目前环境的编译器

```
`$ spack compilers`
`==> Available compilers`
`-- gcc amzn2-x86_64 ---------------------------------------------`
`gcc@7.3.1`
```

安装新的gcc编译器，在这里安装gcc 10.2.0版本

```
$ spack install gcc@10.2.0  # 大约需要30分钟!
$ spack compiler add $(spack location -i gcc@10.2.0) # 增加新的gcc编译器到spack管理
$ spack compilers
==> Available compilers
-- gcc amzn2-x86_64 ---------------------------------------------
gcc@10.2.0  gcc@7.3.1
```

通过ParallelCluster部署的Cluster已经部署了调度器Slurm，在Spack包管理配置文件中注册，并且不再构建

```
$ which sinfo  # AWS ParallelCluster自动安装到/opt/slurm/bin/sinfo
$ sinfo -V
slurm 20.02.4

$ vi ~/.spack/packages.yaml # 增加slurm到spack管理
packages:
   slurm:
       paths:
         slurm@20.02.4: /opt/slurm/
       buildable: False
```

安装openmpi 3.1.6，

```
$ spack install openmpi+legacylaunchers+pmi schedulers=slurm  
# 使用legacylaunchers参数开启mpirun 
$ spack find -p openmpi
$ export PATH=$(spack location -i openmpi)/bin:$PATH
```


安装hdf5 1.12.0 ，netcdf-fortran 4.5.3，netcdf-c 4.7.4, parallel-netcdf 1.12.1

```
$ spack install parallel-netcdf+fortran+shared ^openmpi+legacylaunchers+pmi schedulers=slurm
$ spack install hdf5@1.12.0+fortran+hl ^openmpi+legacylaunchers+pmi schedulers=slurm
# 安装netcdf-fortran会自动安装netcdf-c
$ spack install netcdf-fortran ^hdf5@1.12.0+fortran+hl ^openmpi+legacylaunchers+pmi schedulers=slurm 
```


在.bashrc增加环境变量和设置

```
#必要的执行文件路径
export PATH=$(spack location -i gcc)/bin:$PATH  # 最新安装的gcc路径
export PATH=$(spack location -i openmpi)/bin:$PATH
export PATH=$(spack location -i netcdf-c)/bin:$PATH
export PATH=$(spack location -i netcdf-fortran)/bin:$PATH
export PATH=$(spack location -i parallel-netcdf)/bin:$PATH

#WRF编译需要的环境变量
export PNETCDF=$(spack location -i parallel-netcdf)
export HDF5=$(spack location -i hdf5)
export NETCDF=$(spack location -i netcdf-fortran)
export PHDF5=$(spack location -i hdf5)

#运行环境链接库
export LD_LIBRARY_PATH=$PNETCDF/lib:$HDF5/lib:$NETCDF/lib:$LD_LIBRARY_PATH

#WRF特殊设置
export WRF_EM_CORE=1
export WRFIO_NCD_NO_LARGE_FILE_SUPPORT=0
```

netcdf需要把netcdf-c和netcdf-fortran合并到一起

```
$ source ~/.bashrc
$ NETCDF_C=$(spack location -i netcdf-c)
$ ln -sf $NETCDF_C/include/* $NETCDF/include/
$ ln -sf $NETCDF_C/lib/*  $NETCDF/lib/
```


下载WRF并且编译

```
$ wget https://wrf-graviton2.s3.cn-north-1.amazonaws.com.cn/wrf/v4.2.2.zip
# 可选择海外github下载地址：https://github.com/wrf-model/WRF/archive/v4.2.2.zip
$ unzip v4.2.2.zip
$ cd WRF-4.2.2
$ ./configure
选择4和1
$ ./compile -j 36 em_real 2>&1 | tee wrf_compile.log
```


下载性能测试数据集

```
$ mkdir /shared/data
$ cd /shared/data

# 采用new conus 2.5km测试数据
$ aws s3 sync s3://wrf-graviton2/dataset/conus_2.5km_v4 /shared/data/

# 为方便执行wrf.exe测试，将wrf.exe软链接到数据目录下
$ ln -s /shared/WRF-4.2.2/main/wrf.exe /shared/data/conus_2.5km_v4/wrf.exe
```

编写PBS测试脚本job.sbatch,该脚本放置在数据集目录下

```
#!/bin/bash
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=32 #MPI任务数`(MPI processes),计算时使用的CPU核数`
#SBATCH --cpus-per-task=2 #每任务的线程数`(OMP threads)`设置SLURM_CPUS_PER_TASK环境变量
#ntasks-per-node*cpus-per-task=CPU总核数


#ENV VARIABLES#

#---------------------Run-time env-----------------------------------------
#必要的执行文件路径
export PATH=$(spack location -i gcc)/bin:$PATH  # 最新安装的gcc路径
export PATH=$(spack location -i openmpi)/bin:$PATH
export PATH=$(spack location -i netcdf-c)/bin:$PATH
export PATH=$(spack location -i netcdf-fortran)/bin:$PATH
export PATH=$(spack location -i parallel-netcdf)/bin:$PATH

#WRF编译和运行需要的环境变量
export PNETCDF=$(spack location -i parallel-netcdf)
export HDF5=$(spack location -i hdf5)
export NETCDF=$(spack location -i netcdf-fortran)
export PHDF5=$(spack location -i hdf5)

#运行环境链接库
export LD_LIBRARY_PATH=$PNETCDF/lib:$HDF5/lib:$NETCDF/lib:$LD_LIBRARY_PATH

#设置stack size，运行模型时避免分段错误
ulimit -s unlimited

#设置OMP_NUM_THREADS环境变量，OpenMP线程数，以下设置为读取SLURM_CPUS_PER_TASK环境变量，
#如果没有设置SLURM_CPUS_PER_TASK，则OpenMP线程数为1
if [ -n "$SLURM_CPUS_PER_TASK" ]; then
  omp_threads=$SLURM_CPUS_PER_TASK
else
  omp_threads=1
fi
export OMP_NUM_THREADS=$omp_threads
echo "OMP_NUM_THREADS=" $omp_threads
#export OMP_NUM_THREADS=2 #也可以单独设置OpenMP线程数

#计算任务运行时的图块数`num_tiles，首先会看WRF_NUM_TILES`，
#如果没有设置WRF_NUM_TILES，则会看OMP_NUM_THREADS
#最后如果没有OMP_NUM_THREADS，图块数默认为1
#export `WRF_NUM_TILES``=64`
#--------------------------------------------------------------------------


rm wrfout*

echo `date` Running WRF

resultdir=/shared/data/conus_2.5km_v4/result_c6g-4N_32MPIx2OMP; if [ ! -d $resultdir ]; then mkdir -p $resultdir; fi

date -u +%Y-%m-%d_%H:%M:%S > wrf.times
mpirun --report-bindings ./wrf.exe &> wrf.out
echo nstasks=$SLURM_NTASKS
date -u +%Y-%m-%d_%H:%M:%S >> wrf.times
```


提交PBS脚本，进行计算

```
$ qsub job.sbatch

# 查看job列表和状态
$ qstat
```


实时查看运行输出rsl.out.0000, 运行完成后将rsl.out.0000文件转移到resultdir目录

```
$ cd /shared/data/conus_2.5km_v4
$ tail -f /shared/data/conus_2.5km_v4/rsl.out.0000

# 运行完成后，运行完成后将rsl.out.0000和其他rsl文件转移到resultdir目录
$ mv rsl.* /shared/data/conus_2.5km_v4/result_c6g-4N_32MPIx2OMP
```


统计测试结果, 采用如下python3脚本，保存为/shared/scripts/wrf_stat.py

```
#!/usr/bin/env python3ls


import argparse, re

def main(args):
  files = args['files']
  if len(files) == 0: files = ["rsl.out.0000"]
  max_fname = max([len(f) for f in files]+[4])

  hdr_row1="File | Comp Steps |     Comp Total |       Comp Avg |     Init Total |       Init Avg |    Write Total |      Write Avg |  X  |  Y  |"
  hdr_row2="-----+------------+----------------+----------------+----------------+----------------+----------------+----------------+-----+-----|"
  row_fmt = "| {:"+str(max_fname)+"} | {:10d} | {:14.6f} | {:14.6f} | {:14.6f} | {:14.6f} | {:14.6f} | {:14.6f} | {:3d} | {:3d} |"
  print("| ", " "*(max_fname-4), hdr_row1, sep="")
  print("|-", "-"*(max_fname-4), hdr_row2, sep="")

  cpu_regex = r'Ntasks in X\s*(\d+)\s*,\s*ntasks in Y\s*(\d+)'
  main_regex = r'\s*Timing for main: time [0-9_:-]+ on domain'
  write_regex = r'Timing for Writing [a-zA-Z0-9_:-]+ for domain'
  secs_regex = r'\s*\d+:\s*(\d+\.\d+)\s*elapsed seconds'

  for fname in files:
    with open(fname, 'r') as f:
      txt = f.read()
      cpu_match = re.search(cpu_regex, txt)
      X, Y = int(cpu_match.group(1)), int(cpu_match.group(2))

      write = [float(t) for t in re.findall(write_regex+secs_regex, txt)]

      init = []; comp = []
      for m in re.finditer(r'('+main_regex+secs_regex+r')+', txt):
        secs = [float(t) for t in re.findall(main_regex+secs_regex, m.group())]
        init += [secs[0]]
        comp += secs[1:]

      print(row_fmt.format(
        fname, len(comp), sum(comp), sum(comp)/len(comp), sum(init), sum
        (init)/len(init), sum(write), sum(write)/len(write), X, Y))


if __name__ == '__main__':
  parser = argparse.ArgumentParser(
    description="Analyzes timing output info for WRF runs. If no rsl output "
    "files are specified then look for rsl.out.0000 in current directory")
  parser.add_argument("files", nargs="*", help= "rsl.out.0000 file")
  main(vars(parser.parse_args()))
```

运行wrf_stat.py，输出统计结果

```
# 跳转到运行结果目录
$ cd /shared/data/conus_2.5km_v4/result_c6g-4N_32MPIx2OMP
# 运行wrf_stat.py，输出统计结果
$ /shared/scripts/wrf_stat.py
|         File | Comp Steps |     Comp Total |       Comp Avg |     Init Total |       Init Avg |    Write Total |      Write Avg |  X  |  Y  |
|--------------+------------+----------------+----------------+----------------+----------------+----------------+----------------+-----+-----|
| rsl.out.0000 |        719 |    2212.317550 |       3.076937 |     143.276730 |     143.276730 |     264.042660 |     132.021330 |   8 |  16 |
```


接下来采用ncl工具从ARW WRF模型输出中提取数据，并进行诊断计算结果变量输出。关于ncl工具参考
http://www.ncl.ucar.edu/overview.shtml
通过conda来安装ncl工具：

```
`# 目前conda只支持x86平台，ARM暂不支持，可以参考https://docs.conda.io/en/latest/miniconda.html`
`$ wget `[`https``:``//repo.anaconda.com/miniconda/Miniconda2-latest-Linux-x86_64.sh`](https://repo.anaconda.com/miniconda/Miniconda2-latest-Linux-x86_64.sh)
`$ sh ``Miniconda2``-``latest``-``Linux``-``x86_64``.``sh`

# 安装ncl
$ conda create -n *ncl_stable* -c conda-forge ncl
$ source activate *ncl_stable*
```

使用ncl的examples参考http://www.ncl.ucar.edu/Training/Tutorials/WRF_Users_Workshop/WRF_NCL.shtml，在这里我们采用[wrf_demo_getvar_all.ncl](http://www.ncl.ucar.edu/Training/Tutorials/WRF_Users_Workshop/Scripts/wrf_demo_getvar_all.ncl)这个脚本，获得所有结果诊断。
在WRF结果输出wrfout_*文件目录下，运行该脚步

```
$ wget http://www.ncl.ucar.edu/Training/Tutorials/WRF_Users_Workshop/Scripts/wrf_demo_getvar_all.ncl
# 修改脚本的文件路径和名称
```

```
;----------------------------------------------------------------------
; This script opens a WRF-ARW NetCDF file and calculates every 
; available diagnostic known to "wrf_user_getvar" (as of NCL V6.4.0).
;----------------------------------------------------------------------
f = addfile("wrfout_d01_2008-09-29_16:30:00","r")

wrf_diagnostics = (/"avo","eth","cape_2d","cape_3d","cfrac","ctt","dbz","mdbz",\
                    "geopt","helicity","lat","lon","omg","p","pressure",\
                    "pvo","pw","rh2","rh","slp","ter","td2","td","tc",\
                    "th","tk","tv","twb","updraft_helicity","ua","va",\
                    "wa","uvmet10","uvmet","z"/)

print("======================================================================")
ndiag = dimsizes(wrf_diagnostics)
print("There are " + ndiag + " listed here.")

;---Loop through each diagnostic and calculate it. 
do n=0,ndiag-1
  print("======================================================================")
  print("Calculating diagnostic '" + wrf_diagnostics(n) + "'...")
  x := wrf_user_getvar(f,wrf_diagnostics(n),-1)
  printVarSummary(x)
  printMinMax(x,0)
end do
print("======================================================================")
```

```
# 用ncl工具运行脚本 ,结果输出到文本文件
$ ncl -f wrf_demo_getvar_all.ncl > wrf_demo_getvar_all.result
```




# WPS Build Instructions on Arm

* Download and Install Jasper Software
    * Download Jasper from here: https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/jasper-1.900.1.tar.gz
    * Download config.guess from here: 

`http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD`

    * Replace the config.guess in Jasper dir “**acaux/config.guess**” with above
    * Install Jasper as follows

```
#!/bin/bash

GCC_VERSION=10.2.0
#OPENMPI_VERSION=4.1.0
OPENMPI_VERSION=4.0.5

export PATH="/fsx/gcc-${GCC_VERSION}/bin:$PATH"
export PATH="/fsx/openmpi-${OPENMPI_VERSION}-arm-gcc10.2/bin:$PATH"

export LD_LIBRARY_PATH="/fsx/gcc-${GCC_VERSION}/lib64:$LD_LIBRARY_PATH"
export LD_LIBRARY_PATH="/fsx/openmpi-${OPENMPI_VERSION}-arm-gcc10.2/lib:$LD_LIBRARY_PATH"

which gcc; gcc --version
which mpirun;mpirun --version

export CC=mpicc
export CXX=mpicxx
export FC=mpifort
export F77=mpifort
export F90=mpifort


export CFLAGS="-g -O2 -fPIC"
export CXXFLAGS="-g -O2 -fPIC"
export FFLAGS="-g -fPIC"
export FCFLAGS="-g -fPIC"
export FLDFLAGS="-fPIC"
export F90LDFLAGS="-fPIC"
export LDFLAGS="-fPIC"

wrf_root=/fsx/wrf
wrf_install=${wrf_root}/wrf-build-arm-gcc10.2

./configure --prefix=${wrf_install}/jasper

make -j$(nproc) install
```



* Download and Install WPS
    * Latest from here: [https://github.com/wrf-model/WPS/releases/](https://github.com/wrf-model/WPS/releases/tag/v4.2)
    * Update **arch/configure.defaults** in WPS

```
########################################################################################################################
#ARCH Linux aarch64, Arm compiler OpenMPI # serial smpar dmpar dm+sm
#
COMPRESSION_LIBS    = CONFIGURE_COMP_L
COMPRESSION_INC     = CONFIGURE_COMP_I
FDEFS               = CONFIGURE_FDEFS
SFC                 = gfortran
SCC                 = gcc
DM_FC               = mpif90
DM_CC               = mpicc
FC                  = CONFIGURE_FC
CC                  = CONFIGURE_CC
LD                  = $(FC)
FFLAGS              = -ffree-form -O -fconvert=big-endian -frecord-marker=4 -ffixed-line-length-0 -fallow-argument-mismatch -fallow-invalid-boz
F77FLAGS            = -ffixed-form -O -fconvert=big-endian -frecord-marker=4 -ffree-line-length-0 -fallow-argument-mismatch -fallow-invalid-boz
FCSUFFIX            =
FNGFLAGS            = $(FFLAGS)
LDFLAGS             =
CFLAGS              =
CPP                 = /usr/bin/cpp -P -traditional
CPPFLAGS            = -D_UNDERSCORE -DBYTESWAP -DLINUX -DIO_NETCDF -DBIT32 -DNO_SIGNAL CONFIGURE_MPI
RANLIB              = ranlib
########################################################################################################################
```

* export env var

```
export PATH=/fsx/gcc-10.2.0/bin:$PATH
export LD_LIBRARY_PATH=/fsx/gcc-10.2.0/lib64:$LD_LIBRARY_PATH
export PATH=/fsx/openmpi-4.0.5-arm-gcc10.2/bin:$PATH
export LD_LIBRARY_PATH=/fsx/openmpi-4.0.5-arm-gcc10.2/lib:$LD_LIBRARY_PATH
export CC=mpicc
export CXX=mpicxx
export FC=mpifort
export F77=mpifort
export F90=mpifort
wrf_root=/fsx/wrf
wrf_install=${wrf_root}/wrf-build-arm-gcc10.2
export HDF5=${wrf_install}/hdf5
export PHDF5=${wrf_install}/hdf5
export NETCDF=${wrf_install}/netcdf
export PNETCDF=${wrf_install}/pnetcdf
export LD_LIBRARY_PATH=${wrf_install}/netcdf/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${wrf_install}/pnetcdf/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${wrf_install}/hdf5/lib:$LD_LIBRARY_PATH
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export NETCDF_classic=1
export JASPERLIB=${wrf_install}/jasper/lib
export JASPERINC=${wrf_install}/jasper/include
export WRF_DIR=${wrf_install}/WRF-4.2.1

```



    * Run configure
        * `./configure`
            * `Select 2. Linux aarch64, Arm compiler OpenMPI (dmpar)`
        * Update the `configure.wps` and update in the “WRF_LIB” section (change “-L$(NETCDF)/lib -lnetcdf” to “-L$(NETCDF)/lib -lnetcdff -lnetcdf -lgomp”)

`WRF_LIB = -L$(WRF_DIR)/external/io_grib1 -lio_grib1 \`
`-L$(WRF_DIR)/external/io_grib_share -lio_grib_share \`
`-L$(WRF_DIR)/external/io_int -lwrfio_int \`
`-L$(WRF_DIR)/external/io_netcdf -lwrfio_nf \`
`-L$(NETCDF)/lib -lnetcdff -lnetcdf -lgomp`

        * `./compile`
        * If compilation is successful, you should see 3 executables
            * geogrid.exe → geogrid/src/geogrid.exe
            * ungrib.exe → ungrib/src/ungrib.exe
            * metgrid.exe → metgrid/src/metgrid.exe



# 关于结算结果不一致

安装以上的编译方式,计算出来的结果会出现不一致的情况,这是因为intel在编译WRF的时候可以使用‘-fp-model precise’参数,这个可以保证每次的计算结果一致. 详见https://rc.dartmouth.edu/wp-content/uploads/2018/03/FP_Consistency_12.pdf
![Intel](/intel.png "Intel")
测试结果:
PS:以下所有测试结果均采用同一个数据集,同一个WRF版本(4.2.2)

1. 同一个单节点测试,前后测试了3次,结果一致
2. 2节点并行计算, 与1的测试存在差异
![result](/result.png "result")


C6G上安装conda，numpy，netcdf5等依赖包

```
wget https://github.com/Archiconda/build-tools/releases/download/0.2.3/Archiconda3-0.2.3-Linux-aarch64.sh
sh Archiconda3-0.2.3-Linux-aarch64.sh
source /home/ec2-user/.bashrc
sudo yum install python3-devel.aarch64
pip3 install Cython —user
pip3 install https://pypi.tuna.tsinghua.edu.cn/simple numpy --user
pip3 install wrf-python —user
pip3 install pystache --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple requests --user 


环境变量：
export WRF_INSTALL=/fsx/wrf-arm
export GCC_VERSION=10.2.0
export OPENMPI_VERSION=4.1.0
export PATH=${WRF_INSTALL}/gcc-${GCC_VERSION}/bin:$PATH
export LD_LIBRARY_PATH=${WRF_INSTALL}/gcc-${GCC_VERSION}/lib64:$LD_LIBRARY_PATH
export PATH=${WRF_INSTALL}/openmpi-${OPENMPI_VERSION}/bin:$PATH
export LD_LIBRARY_PATH=${WRF_INSTALL}/openmpi-${OPENMPI_VERSION}/lib:$LD_LIBRARY_PATH
export CC=mpicc
export CXX=mpicxx
export FC=mpif90
export F77=mpif90
export F90=mpif90
export HDF5=${WRF_INSTALL}/hdf5
export PHDF5=${WRF_INSTALL}/hdf5
export NETCDF=${WRF_INSTALL}/netcdf
export PNETCDF=${WRF_INSTALL}/pnetcdf
export PATH=${WRF_INSTALL}/netcdf/bin:${PATH}
export PATH=${WRF_INSTALL}/pnetcdf/bin:${PATH}
export PATH=${WRF_INSTALL}/hdf5/bin:${PATH}
export LD_LIBRARY_PATH=${WRF_INSTALL}/netcdf/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${WRF_INSTALL}/pnetcdf/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${WRF_INSTALL}/hdf5/lib:$LD_LIBRARY_PATH
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export NETCDF_classic=1


pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple hdf5 --user 

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple netcdf4 --user

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple python-dateutil --user


sudo pip3 install --upgrade --force-reinstall setuptools 
pip3 install wheel --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple matplotlib --user

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pandas --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pybind11 --user

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple scipy --user

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple envoy --user

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple h5py --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple mako --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pvlib --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple windrose --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pymysql --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pymongo --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple astral --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple boto --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple IPython --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple psutil --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple django --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple django-constance --user
sudo yum install  libxslt-devel
pip3 install -i https://pypi.douban.com/simple lxml --user
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pykml --user

在python 2.4以上的版本 commands模块被subprocess取代了（原来import commands是python内置的），
import subprocess即可


conda install -c conda-forge proj

安装cmake环境包（cmake --version查看一下版本，如低于3.10则需安装）
下载cmake包 ，按照安装说明安装
wget https://cmake.org/files/v3.19/cmake-3.19.0.tar.gz
tar zxvf cmake-3.19.0.tar.gz
sudo ln -s /home/ec2-user/archiconda3/lib/libstdc++.so.6.0.28 /lib64/libstdc++.so.6
./bootstrap && make && sudo make install
export CMAKE_ROOT=/usr/local/share/cmake-3.19

wget https://confluence.ecmwf.int/download/attachments/45757960/eccodes-2.21.0-Source.tar.gz
解压之后，进入解压后的文件夹：
mkdir build
cd build
hash -r
cmake ../../eccodes-2.21.0-Source  -DCMAKE_INSTALL_PREFIX=/usr/local

pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple eccodes --user



目前安装pygrib报错
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pygrib
Collecting pygrib
  Using cached https://pypi.tuna.tsinghua.edu.cn/packages/96/34/f76696b84d0249e41154f3bb3345bd05aa5385bddceb04e6ccbbde5b7ab0/pygrib-2.1.3.tar.gz
Requirement already satisfied: pyproj in /home/ec2-user/.local/lib/python3.7/site-packages (from pygrib)
Requirement already satisfied: numpy in /home/ec2-user/.local/lib/python3.7/site-packages (from pygrib)
Requirement already satisfied: certifi in /home/ec2-user/.local/lib/python3.7/site-packages (from pyproj->pygrib)
Building wheels for collected packages: pygrib
  Running setup.py bdist_wheel for pygrib ... error
  Complete output from command /usr/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-j48u6spw/pygrib/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" bdist_wheel -d /tmp/tmpuwgt0eompip-wheel- --python-tag cp37:
  eccodes not found, build may fail...
  running bdist_wheel
  running build
  running build_py
  creating build
  creating build/lib.linux-aarch64-3.7
  creating build/lib.linux-aarch64-3.7/pygrib
  copying pygrib/__init__.py -> build/lib.linux-aarch64-3.7/pygrib
  running build_ext
  cythoning pygrib/_pygrib.pyx to pygrib/_pygrib.c
  /home/ec2-user/.local/lib/python3.7/site-packages/Cython/Compiler/Main.py:369: FutureWarning: Cython directive 'language_level' not set, using 2 for now (Py2). This will change in a later release! File: /tmp/pip-build-j48u6spw/pygrib/pygrib/_pygrib.pyx
    tree = Parsing.p_module(s, pxd, full_module_name)
  building 'pygrib._pygrib' extension
  creating build/temp.linux-aarch64-3.7
  creating build/temp.linux-aarch64-3.7/pygrib
  mpicc -Wno-unused-result -Wsign-compare -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -moutline-atomics -D_GNU_SOURCE -fPIC -fwrapv -fPIC -I/usr/include/python3.7m -I/home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include -c pygrib/_pygrib.c -o build/temp.linux-aarch64-3.7/pygrib/_pygrib.o
  In file included from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/ndarraytypes.h:1944,
                   from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/ndarrayobject.h:12,
                   from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/arrayobject.h:4,
                   from pygrib/_pygrib.c:612:
  /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/npy_1_7_deprecated_api.h:17:2: warning: #warning "Using deprecated NumPy API, disable it with " "#define NPY_NO_DEPRECATED_API NPY_1_7_API_VERSION" [-Wcpp]
     17 | #warning "Using deprecated NumPy API, disable it with " \
        |  ^~~~~~~
  pygrib/_pygrib.c:622:10: fatal error: grib_api.h: No such file or directory
    622 | #include "grib_api.h"
        |          ^~~~~~~~~~~~
  compilation terminated.
  error: command 'mpicc' failed with exit status 1
  
  ----------------------------------------
  Failed building wheel for pygrib
  Running setup.py clean for pygrib
Failed to build pygrib
Installing collected packages: pygrib
  Running setup.py install for pygrib ... error
    Complete output from command /usr/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-j48u6spw/pygrib/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-x8u0sntp-record/install-record.txt --single-version-externally-managed --compile:
    eccodes not found, build may fail...
    running install
    running build
    running build_py
    creating build
    creating build/lib.linux-aarch64-3.7
    creating build/lib.linux-aarch64-3.7/pygrib
    copying pygrib/__init__.py -> build/lib.linux-aarch64-3.7/pygrib
    running build_ext
    skipping 'pygrib/_pygrib.c' Cython extension (up-to-date)
    building 'pygrib._pygrib' extension
    creating build/temp.linux-aarch64-3.7
    creating build/temp.linux-aarch64-3.7/pygrib
    mpicc -Wno-unused-result -Wsign-compare -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -moutline-atomics -D_GNU_SOURCE -fPIC -fwrapv -fPIC -I/usr/include/python3.7m -I/home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include -c pygrib/_pygrib.c -o build/temp.linux-aarch64-3.7/pygrib/_pygrib.o
    In file included from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/ndarraytypes.h:1944,
                     from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/ndarrayobject.h:12,
                     from /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/arrayobject.h:4,
                     from pygrib/_pygrib.c:612:
    /home/ec2-user/.local/lib/python3.7/site-packages/numpy/core/include/numpy/npy_1_7_deprecated_api.h:17:2: warning: #warning "Using deprecated NumPy API, disable it with " "#define NPY_NO_DEPRECATED_API NPY_1_7_API_VERSION" [-Wcpp]
       17 | #warning "Using deprecated NumPy API, disable it with " \
          |  ^~~~~~~
    pygrib/_pygrib.c:622:10: fatal error: grib_api.h: No such file or directory
      622 | #include "grib_api.h"
          |          ^~~~~~~~~~~~
    compilation terminated.
    error: command 'mpicc' failed with exit status 1
    
    ----------------------------------------
Command "/usr/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-j48u6spw/pygrib/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-x8u0sntp-record/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /tmp/pip-build-j48u6spw/pygrib/


```


