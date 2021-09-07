# AWS ParallelCluster, Slurm, Spack, Intel Compiler and WRF4.2.2 on China Region c5 instance
## 安装parallelcluster

按照如下修改配置文件.parallelcluster/config

```
[aws]
aws_region_name = cn-north-1

[cluster wrf4]
key_name = sean-beijing-key
base_os = alinux2
master_instance_type = c5.18xlarge #在master上编译相关依赖需要大量的CPU，内存资源
compute_instance_type = c5.18xlarge
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
cluster_template = wrf4
update_check = true
sanity_check = true
```


创建pcluster

```
pcluster create wrf4
```

## 安装WRF

`登录到master节点`

```
pcluster ssh wrf4 -i sean-beijing-key.pem
```

spack是为HPC的包管理器，spack可以自动下载包，依赖包和调优，以下是安装spack的步骤

```
`cd ``/``shared  ``# install to shared disk`
git clone https://github.com/spack/spack.git

`echo ``'export PATH=/shared/spack/bin:$PATH'`` ``>>`` ``~/.``bashrc  ``# to discover spack executable`
`source ``~/.``bashrc`
```

检查spack安装成功和版本

```
spack --version
0.16.1-2609-e7219db93d

```

查看目前环境的编译器

```
`spack compilers`
`==> Available compilers`
`-- gcc amzn2-x86_64 ---------------------------------------------`
`gcc@7.3.1`
```

安装Intel 编译器2021.2.0版本

```
spack install intel-oneapi-compilers
spack compiler add `spack location -i intel-oneapi-compilers`/compiler/latest/linux/bin/intel64
spack compiler add `spack location -i intel-oneapi-compilers`/compiler/latest/linux/bin # 增加新的intel编译器到spack管理
spack compilers
```



```
which sinfo  # AWS ParallelCluster自动安装到/opt/slurm/bin/sinfo
sinfo -V
slurm 20.02.4

vi ~/.spack/packages.yaml # 增加slurm到spack管理
packages:
   slurm:
       paths:
         slurm@20.02.4: /opt/slurm/
       buildable: False
```

安装intel-mpi 2019.10.317，

```
spack install intel-mpi%intel 
spack find -p intel-mpi
```


安装hdf5 1.10.7 ，netcdf-fortran 4.5.3，netcdf-c 4.8.0, parallel-netcdf 1.12.2

```
spack install parallel-netcdf%intel+fortran+shared ^intel-mpi
spack install hdf5%intel+fortran+hl ^intel-mpi
# 安装netcdf-fortran会自动安装netcdf-c
spack install netcdf-fortran ^hdf5%intel+fortran+hl ^intel-mpi 
```


在.bashrc增加环境变量和设置

```
#必要的执行文件路径
export PATH=$(spack location -i intel-oneapi-compilers)/compiler/latest/linux/bin:$PATH  # 最新安装的gcc路径
export PATH=$(spack location -i intel-oneapi-compilers)/compiler/2021.2.0/linux/bin/intel64:$PATH
export PATH=$(spack location -i intel-mpi%intel)/impi/2019.10.317/intel64/bin:$PATH
export PATH=$(spack location -i netcdf-c%intel)/bin:$PATH
export PATH=$(spack location -i netcdf-fortran%intel)/bin:$PATH
export PATH=$(spack location -i parallel-netcdf%intel)/bin:$PATH

#WRF编译需要的环境变量
export PNETCDF=$(spack location -i parallel-netcdf%intel)
export HDF5=$(spack location -i hdf5%intel)
export NETCDF=$(spack location -i netcdf-fortran%intel)
export PHDF5=$(spack location -i hdf5%intel)

#运行环境链接库
export LD_LIBRARY_PATH=$PNETCDF/lib:$HDF5/lib:$NETCDF/lib:$LD_LIBRARY_PATH

`export CC=mpiicc`
`export CXX=mpiicpc`
`export FC=mpiifort`
`export F77=mpiifort`
`export F90=mpiifort`

`export I_MPI_CC=icc`
`export I_MPI_CXX=icpc`
`export I_MPI_F90=ifort`
`export I_MPI_F77=ifort`
#WRF特殊设置
export WRF_EM_CORE=1
export WRFIO_NCD_NO_LARGE_FILE_SUPPORT=0
```


netcdf需要把netcdf-c和netcdf-fortran合并到一起

```
source ~/.bashrc
NETCDF_C=$(spack location -i netcdf-c%intel)
ln -sf $NETCDF_C/include/* $NETCDF/include/
ln -sf $NETCDF_C/lib/*  $NETCDF/lib/
```


下载WRF并且编译

```
wget https://wrf-graviton2.s3.cn-north-1.amazonaws.com.cn/wrf/v4.2.2.zip
# wget https://github.com/wrf-model/WRF/archive/v4.2.2.zip
unzip v4.2.2.zip
cd WRF-4.2.2
./configure
Please select from among the following Linux x86_64 options:

  1. (serial)   2. (smpar)   3. (dmpar)   4. (dm+sm)   PGI (pgf90/gcc)
  5. (serial)   6. (smpar)   7. (dmpar)   8. (dm+sm)   PGI (pgf90/pgcc): SGI MPT
  9. (serial)  10. (smpar)  11. (dmpar)  12. (dm+sm)   PGI (pgf90/gcc): PGI accelerator
 13. (serial)  14. (smpar)  15. (dmpar)  16. (dm+sm)   INTEL (ifort/icc)
                                         17. (dm+sm)   INTEL (ifort/icc): Xeon Phi (MIC architecture)
 18. (serial)  19. (smpar)  20. (dmpar)  21. (dm+sm)   INTEL (ifort/icc): Xeon (SNB with AVX mods)
 22. (serial)  23. (smpar)  24. (dmpar)  25. (dm+sm)   INTEL (ifort/icc): SGI MPT
 26. (serial)  27. (smpar)  28. (dmpar)  29. (dm+sm)   INTEL (ifort/icc): IBM POE
 30. (serial)               31. (dmpar)                PATHSCALE (pathf90/pathcc)
 32. (serial)  33. (smpar)  34. (dmpar)  35. (dm+sm)   GNU (gfortran/gcc)
 36. (serial)  37. (smpar)  38. (dmpar)  39. (dm+sm)   IBM (xlf90_r/cc_r)
 40. (serial)  41. (smpar)  42. (dmpar)  43. (dm+sm)   PGI (ftn/gcc): Cray XC CLE
 44. (serial)  45. (smpar)  46. (dmpar)  47. (dm+sm)   CRAY CCE (ftn $(NOOMP)/cc): Cray XE and XC
 48. (serial)  49. (smpar)  50. (dmpar)  51. (dm+sm)   INTEL (ftn/icc): Cray XC
 52. (serial)  53. (smpar)  54. (dmpar)  55. (dm+sm)   PGI (pgf90/pgcc)
 56. (serial)  57. (smpar)  58. (dmpar)  59. (dm+sm)   PGI (pgf90/gcc): -f90=pgf90
 60. (serial)  61. (smpar)  62. (dmpar)  63. (dm+sm)   PGI (pgf90/pgcc): -f90=pgf90
 64. (serial)  65. (smpar)  66. (dmpar)  67. (dm+sm)   INTEL (ifort/icc): HSW/BDW
 68. (serial)  69. (smpar)  70. (dmpar)  71. (dm+sm)   INTEL (ifort/icc): KNL MIC
 72. (serial)  73. (smpar)  74. (dmpar)  75. (dm+sm)   FUJITSU (frtpx/fccpx): FX10/FX100 SPARC64 IXfx/Xlfx

Enter selection [1-75] : 16
选择16
------------------------------------------------------------------------
Compile for nesting? (1=basic, 2=preset moves, 3=vortex following) [default 1]: 1
选择1

./compile -j $(nproc) em_real 2>&1 | tee compile_wrf.out
```

## 安装WPS

安装Jasper

```
spack install jasper%intel
spack find jasper%intel
```

```
`cd ``/shared`
`wget `[`https``:``//github.com/wrf-model/WPS/archive/v4.2.tar.gz`](https://github.com/wrf-model/WPS/archive/v4.2.tar.gz)
`tar xvf v4``.``2.tar``.``gz`
`cd WPS``-``4.2``/`
# 在~/.bashrc中加入如下环境变量
export WRF_INSTALL=/shared
export JASPERLIB=$(spack location -i jasper%intel)/lib64
export JASPERINC=$(spack location -i jasper%intel)/include
export WRF_DIR=${WRF_INSTALL}/WRF-4.2.2-intel-compiler
export LD_LIBRARY_PATH=$(spack location -i jasper%intel)/lib64:$LD_LIBRARY_PATH
export PATH=$(spack location -i jasper%intel)/bin:$PATH

`./``configure`
```

输出：

```
Please select from among the following supported platforms.

   1.  Linux x86_64, gfortran    (serial)
   2.  Linux x86_64, gfortran    (serial_NO_GRIB2)
   3.  Linux x86_64, gfortran    (dmpar)
   4.  Linux x86_64, gfortran    (dmpar_NO_GRIB2)
   5.  Linux x86_64, PGI compiler   (serial)
   6.  Linux x86_64, PGI compiler   (serial_NO_GRIB2)
   7.  Linux x86_64, PGI compiler   (dmpar)
   8.  Linux x86_64, PGI compiler   (dmpar_NO_GRIB2)
   9.  Linux x86_64, PGI compiler, SGI MPT   (serial)
  10.  Linux x86_64, PGI compiler, SGI MPT   (serial_NO_GRIB2)
  11.  Linux x86_64, PGI compiler, SGI MPT   (dmpar)
  12.  Linux x86_64, PGI compiler, SGI MPT   (dmpar_NO_GRIB2)
  13.  Linux x86_64, IA64 and Opteron    (serial)
  14.  Linux x86_64, IA64 and Opteron    (serial_NO_GRIB2)
  15.  Linux x86_64, IA64 and Opteron    (dmpar)
  16.  Linux x86_64, IA64 and Opteron    (dmpar_NO_GRIB2)
  17.  Linux x86_64, Intel compiler    (serial)
  18.  Linux x86_64, Intel compiler    (serial_NO_GRIB2)
  19.  Linux x86_64, Intel compiler    (dmpar)
  20.  Linux x86_64, Intel compiler    (dmpar_NO_GRIB2)
  21.  Linux x86_64, Intel compiler, SGI MPT    (serial)
  22.  Linux x86_64, Intel compiler, SGI MPT    (serial_NO_GRIB2)
  23.  Linux x86_64, Intel compiler, SGI MPT    (dmpar)
  24.  Linux x86_64, Intel compiler, SGI MPT    (dmpar_NO_GRIB2)
  25.  Linux x86_64, Intel compiler, IBM POE    (serial)
  26.  Linux x86_64, Intel compiler, IBM POE    (serial_NO_GRIB2)
  27.  Linux x86_64, Intel compiler, IBM POE    (dmpar)
  28.  Linux x86_64, Intel compiler, IBM POE    (dmpar_NO_GRIB2)
  29.  Linux x86_64 g95 compiler     (serial)
  30.  Linux x86_64 g95 compiler     (serial_NO_GRIB2)
  31.  Linux x86_64 g95 compiler     (dmpar)
  32.  Linux x86_64 g95 compiler     (dmpar_NO_GRIB2)
  33.  Cray XE/XC CLE/Linux x86_64, Cray compiler   (serial)
  34.  Cray XE/XC CLE/Linux x86_64, Cray compiler   (serial_NO_GRIB2)
  35.  Cray XE/XC CLE/Linux x86_64, Cray compiler   (dmpar)
  36.  Cray XE/XC CLE/Linux x86_64, Cray compiler   (dmpar_NO_GRIB2)
  37.  Cray XC CLE/Linux x86_64, Intel compiler   (serial)
  38.  Cray XC CLE/Linux x86_64, Intel compiler   (serial_NO_GRIB2)
  39.  Cray XC CLE/Linux x86_64, Intel compiler   (dmpar)
  40.  Cray XC CLE/Linux x86_64, Intel compiler   (dmpar_NO_GRIB2)

Enter selection [1-40] : 19
```

编辑configure.wps,  加入对OpenMP的支持，找到"WRF_LIB"，在最后加入`-liomp5 -lpthread `

```
WRF_LIB         =       -L$(WRF_DIR)/external/io_grib1 -lio_grib1 \
                        -L$(WRF_DIR)/external/io_grib_share -lio_grib_share \
                        -L$(WRF_DIR)/external/io_int -lwrfio_int \
                        -L$(WRF_DIR)/external/io_netcdf -lwrfio_nf \
                        -L$(NETCDF)/lib -lnetcdff -lnetcdf -liomp5 -lpthread
```

编译

```
./compile 2>&1 | tee compile.log
```

验证是否编译成功：

```
`ls -rlt *.exe
lrwxrwxrwx 1 ec2-user ec2-user 23 May 13 23:54 geogrid.exe -> geogrid/src/geogrid.exe
lrwxrwxrwx 1 ec2-user ec2-user 21 May 13 23:54 ungrib.exe -> ungrib/src/ungrib.exe
lrwxrwxrwx 1 ec2-user ec2-user 23 May 13 23:54 metgrid.exe -> metgrid/src/metgrid.exe

`
```

## 计算测试

下载性能测试数据集

```
mkdir /shared/data
cd /shared/data

# 采用new conus 2.5km测试数据
aws s3 sync s3://wrf-graviton2/dataset/conus_2.5km_v4 /shared/data/

# 为方便执行wrf.exe测试，将wrf.exe软链接到数据目录下
ln -s /shared/WRF-4.2.2/main/wrf.exe /shared/data/conus_2.5km_v4/wrf.exe
```

编写PBS测试脚本job.sbatch,该脚本放置在数据集目录下

```
#!/bin/bash
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=72
#SBATCH --cpus-per-task=2

#ENV VARIABLES#

#---------------------Run-time env-----------------------------------------
#必要的执行文件路径
export PATH=$(spack location -i gcc)/bin:$PATH  # 最新安装的gcc路径
export PATH=$(spack location -i intel-mpi)/bin:$PATH
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

#设置OMP_NUM_THREADS环境变量，OpenMP线程数
if [ -n "$SLURM_CPUS_PER_TASK" ]; then
  omp_threads=$SLURM_CPUS_PER_TASK
else
  omp_threads=1
fi
export OMP_NUM_THREADS=$omp_threads
echo "OMP_NUM_THREADS=" $omp_threads

#WRF_NUM_TILES设置图快数，通常图块数等于OpenMP线程
export `WRF_NUM_TILES``=2`
#--------------------------------------------------------------------------


rm wrfout*

echo `date` Running WRF

resultdir=/shared/data/conus_2.5km_v4/result_c5-4N_72MPIx2OMP; if [ ! -d $resultdir ]; then mkdir -p $resultdir; fi

date -u +%Y-%m-%d_%H:%M:%S > wrf.times
mpirun --report-bindings ./wrf.exe &> wrf.out
echo nstasks=$SLURM_NTASKS
date -u +%Y-%m-%d_%H:%M:%S >> wrf.times
```


提交PBS脚本，进行计算

```
qsub job.sbatch

# 查看job列表和状态
qstat
```


实时查看运行输出rsl.out.0000, 运行完成后将rsl.out.0000文件转移到resultdir目录

```
cd /shared/data/conus_2.5km_v4
tail -f /shared/data/conus_2.5km_v4/rsl.out.0000

# 运行完成后，运行完成后将rsl.out.0000和其他rsl文件转移到resultdir目录
mv rsl.* /shared/data/conus_2.5km_v4/result_c5-4N_72MPIx2OMP
```


统计测试结果, 采用如下python3脚本，保存为/shared/scripts/wrf_stat.py

```
#!/usr/bin/env python3

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
cd /shared/data/conus_2.5km_v4/result_c5-4N_72MPIx2OMP
# 运行wrf_stat.py，输出统计结果
/shared/scripts/wrf_stat.py
|         File | Comp Steps |     Comp Total |       Comp Avg |     Init Total |       Init Avg |    Write Total |      Write Avg |  X  |  Y  |
|--------------+------------+----------------+----------------+----------------+----------------+----------------+----------------+-----+-----|
| rsl.out.0000 |        719 |    2467.711930 |       3.432145 |     274.544070 |     274.544070 |     474.392270 |     237.196135 |  16 |  18 |
```

