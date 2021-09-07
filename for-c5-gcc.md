#  AWS ParallelCluster, Slurm, Spack, and WRF4.2.2 on China Region c5 instance

作者：Sean Yang
邮件：[zhihay@amazon.com](mailto:zhihay@amazon.com)
欢迎指教，指错和探讨！

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
$ pcluster create wrf4
```


`登录到master节点`

```
$ pcluster ssh wrf4 -i sean-beijing-key.pem
```

spack是为HPC的包管理器，spack可以自动下载包，依赖包和调优，以下是安装spack的步骤

```
`$ cd ``/``shared  ``# install to shared disk`
$ `wget https``:``//wrf-graviton2.s3.cn-north-1.amazonaws.com.cn/wrf/spack-0.16.0.tar.gz`
`#wget https``:``//github.com/spack/spack/releases/download/v0.16.1/spack-0.16.1.tar.gz`
`$ tar zxvf spack``-``0.16``.``1.tar``.``gz`
`$ mv spack``-``0.16``.``1`` spack`
`$ echo ``'export PATH=/shared/spack/bin:$PATH'`` ``>>`` ``~/.``bashrc  ``# to discover spack executable`
`$ source ``~/.``bashrc`
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
$ spack install openmpi+legacylaunchers+pmi schedulers=slurm  # 使用legacylaunchers参数开启mpirun 
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
# wget https://github.com/wrf-model/WRF/archive/v4.2.2.zip
$ unzip v4.2.2.zip
$ cd WRF-4.2.2
$ ./configure
选择34和1
$ ./compile -j 32 em_real 2>&1 | tee wrf_compile.log
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
#SBATCH --ntasks-per-node=72
#SBATCH --cpus-per-task=2

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
$ qsub job.sbatch

# 查看job列表和状态
$ qstat
```


实时查看运行输出rsl.out.0000, 运行完成后将rsl.out.0000文件转移到resultdir目录

```
$ cd /shared/data/conus_2.5km_v4
$ tail -f /shared/data/conus_2.5km_v4/rsl.out.0000

# 运行完成后，运行完成后将rsl.out.0000和其他rsl文件转移到resultdir目录
$ mv rsl.* /shared/data/conus_2.5km_v4/result_c5-4N_72MPIx2OMP
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
$ cd /shared/data/conus_2.5km_v4/result_c5-4N_72MPIx2OMP
# 运行wrf_stat.py，输出统计结果
$ /shared/scripts/wrf_stat.py
|         File | Comp Steps |     Comp Total |       Comp Avg |     Init Total |       Init Avg |    Write Total |      Write Avg |  X  |  Y  |
|--------------+------------+----------------+----------------+----------------+----------------+----------------+----------------+-----+-----|
| rsl.out.0000 |        719 |    2467.711930 |       3.432145 |     274.544070 |     274.544070 |     474.392270 |     237.196135 |  16 |  18 |
```

