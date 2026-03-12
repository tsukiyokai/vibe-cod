# HCCL性能测试工具(HCCL Test)

> 来源：CANN商用版8.0.RC3 hiascend.com

## 工具介绍

### 适用场景

分布式训练场景下，开发者可以通过此工具测试HCCL(Huawei Collective Communication Library)集合通信的功能正确性以及性能。

### 工具源码包获取

安装完CANN Toolkit软件包后，HCCL性能测试工具源码存放${INSTALL_DIR}/tools/hccl_test路径下，${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。例如，若安装的Ascend-cann-toolkit软件包，则安装后文件存储路径为：$HOME/Ascend/ascend-toolkit/latest。

使用前需要参考下面的章节进行工具的编译。

### 约束说明

- 针对Atlas训练系列产品，当前版本HCCL性能测试工具最大支持集群组网包含4096张卡的场景。
- 针对Atlas A2 训练系列产品，当前版本HCCL性能测试工具最大支持集群组网包含32K张卡的场景。

### 背景知识

#### 集合通信带宽

集合通信带宽指的是"算法带宽"，即"执行某集合通信操作时的数据量/耗时"。

例如单机8卡做allreduce操作，则"数据量"除以 "做完allreduce的耗时" 就是allreduce算子执行的算法带宽。

使用HCCL性能测试工具进行测试时，带宽数据即指的"算法带宽"。

影响算法带宽的主要因素有：

- Server间的RDMA带宽（RoCE链路）。
- Server内卡间SDMA通信带宽（HCCS链路）。
- PCIe链路带宽。
- 通信算法自身编排实现。

#### 物理带宽

集群中的物理带宽包括HCCS链路物理带宽以及RoCE链路物理带宽，物理带宽是影响算法带宽的一个因素。

---

## MPI安装与配置

HCCL性能测试工具依赖MPI拉起多个进程，所以需要先安装MPI软件。

> 以下操作需要在每个参与集合通信的机器上执行。

- 如果通信网卡仅使用IPv4协议：
  - 针对Atlas训练系列产品，请安装[MPICH 3.2.1](https://www.mpich.org/static/downloads/)版本或者[Open MPI-4.1.5](https://www.open-mpi.org/software/ompi/v4.1/)版本。
  - 针对Atlas A2 训练系列产品，请安装[MPICH 3.2.1](https://www.mpich.org/static/downloads/)版本或者[Open MPI-4.1.5](https://www.open-mpi.org/software/ompi/v4.1/)版本。
- 如果通信网卡需要使用IPv6协议，请安装[Open MPI-4.1.5](https://www.open-mpi.org/software/ompi/v4.1/)版本。

下面分别介绍MPICH与Open MPI的安装配置流程，注意以下操作需要在每个参与集合通信的机器上执行。

### MPICH安装配置

1. 安装MPICH软件包。
  1. 下载并解压MPICH软件包。
      MPICH软件包下载地址：[Link](https://www.mpich.org/static/downloads/)。
- 针对Atlas训练系列产品，请下载MPICH 3.2.1版本。
- 针对Atlas A2 训练系列产品，请下载MPICH 3.2.1版本。

      获取到mpich-${version}.tar.gz后，执行如下命令解压缩软件包。
      ```
      tar -zxvf mpich-${version}.tar.gz
      ```
      ${version}为MPICH的版本号。

2. 进入MPICH解压后路径，并配置编译选项。
      ```
      cd mpich-${version}
      ./configure --disable-fortran  --prefix=/usr/local/mpich
      ```
  - `--disable-fortran`：禁用Fortran语言支持。
  - `--prefix`：配置的MPI安装路径，用户可自定义。

3. 编译并安装MPICH。
      ```
      make && make install
      ```
      以上命令执行完成后MPICH会安装在"/usr/local/mpich"路径下。

2. 配置网络节点信息。
   将运行环境的IP地址加入到"/etc/hosts"文件中，格式为"IP地址 主机名"，示例如下：
   ```
   172.16.0.100 node3
   ```
   其中"node3"为主机名。

   注意如果是Euler OS操作系统，需要执行如下命令使更新后的"/etc/hosts"文件生效：
   ```
   nmcli c reload
   ```

3. 配置当前操作节点到集群通信节点的SSH信任关系，以支持集群通信节点远程登录。
   以下仅为操作示例：

1. 在当前操作节点生成密钥信息（如若环境中存在，可不重复执行）：
      ```
      ssh-keygen -t rsa
      ```
      例如密钥信息生成后，存储在"/root/.ssh/id_rsa.pub"文件中。

2. 将操作节点的公钥文件复制到集群通信其他节点，实现SSH密钥登录远程主机。
      示例如下，其中${node_X_ip_address}是需要与操作节点通信的节点IP地址。
      ```
      ssh-copy-id -i /root/.ssh/id_rsa.pub ${node3_ip_address}
      ssh-copy-id -i /root/.ssh/id_rsa.pub ${node4_ip_address}
      ```

3. SSH远程登录上述node3与node4，确认是否可以直接登录。

### Open MPI安装配置

1. 下载并解压Open MPI软件包。
   参见[Open MPI-4.1.5](https://www.open-mpi.org/software/ompi/v4.1/)下载4.1.5版本的软件包，例如：openmpi-4.1.5.tar.gz，然后执行如下命令解压缩软件包。
   ```
   tar -zxvf openmpi-4.1.5.tar.gz
   ```
   解压缩后Open MPI源码存储在openmpi-4.1.5路径下。

2. 编辑Open MPI源码相关配置文件，修改Open MPI支持的最大Host数量。
  1. 进入Open MPI源码存储路径。
      ```
      cd openmpi-4.1.5
      ```

  2. 修改"orte/mca/routed/radix/routed_radix_component.c"配置文件。
      ```
      vi orte/mca/routed/radix/routed_radix_component.c
      ```
      修改配置参数"mca_routed_radix_component.radix"的值为"集群中总卡数/单Server中卡数"，例如：
      ```
      mca_routed_radix_component.radix = 1024;
      ```
      保存退出。

3. 修改"orte/mca/plm/rsh/plm_rsh_component.c"配置文件。
      ```
      vi orte/mca/plm/rsh/plm_rsh_component.c
      ```
      修改配置参数"mca_plm_rsh_component.num_concurrent"的值为"集群中总卡数/单Server中卡数"，例如：
      ```
      mca_plm_rsh_component.num_concurrent = 1024;
      ```
      保存退出。

3. 配置编译选项。
   ```
   ./configure --disable-fortran --enable-ipv6 --prefix=/usr/local/openmpi
   ```
  - `--disable-fortran`：禁用Fortran语言支持。
  - `--enable-ipv6`：启用IPv6支持。
  - `--prefix`：配置的Open MPI的安装路径，用户可自定义。

4. 编译并安装Open MPI。
   ```
   make && make install
   ```
   以上命令执行完成后Open MPI会安装在"/usr/local/openmpi"路径下。

5. 配置网络节点信息。
   将运行环境的IP地址加入到"/etc/hosts"文件中，格式为"IP地址 主机名"，示例如下：
   ```
   172.16.0.100 node1
   172.16.1.200 node2
   fec0::b6ef:69dc:337d:9a12 node3
   fec0::b6ef:998f:f3eb:4617 node4
   ```
   注意如果是Euler OS操作系统，需要执行如下命令使更新后的"/etc/hosts"文件生效：
   ```
   nmcli c reload
   ```

6. 配置当前操作节点到集群通信节点的SSH信任关系，以支持集群通信节点远程登录。
   以下仅为操作示例：

1. 在当前操作节点生成密钥信息（如若环境中存在，可不重复执行）：
      ```
      ssh-keygen -t rsa
      ```
      例如密钥信息生成后，存储在"/root/.ssh/id_rsa.pub"文件中。

2. 将操作节点公钥复制到集群通信其他节点，实现SSH密钥登录远程主机。
  - 如果通信网卡使用IPv4地址，公钥复制命令如下：
        ```
        ssh-copy-id -i /root/.ssh/id_rsa.pub ${node1_ipv4_address}
        ssh-copy-id -i /root/.ssh/id_rsa.pub ${node2_ipv4_address}
        ```
        例如：
        ```
        ssh-copy-id -i /root/.ssh/id_rsa.pub 172.16.0.100
        ```

- 如果通信网卡使用IPv6地址，公钥复制命令如下：
        ```
        ssh-copy-id -i /root/.ssh/id_rsa.pub ${node3_ipv6_address}%网卡名
        ssh-copy-id -i /root/.ssh/id_rsa.pub ${node4_ipv6_address}%网卡名
        ```
        例如：
        ```
        ssh-copy-id -i /root/.ssh/id_rsa.pub fec0::b6ef:998f:f3eb:4617%enp189s0f0
        ```

3. SSH远程登录到上述配置信任关系的节点，确认是否可以直接登录。

7. 配置MPICH启动参数，此步骤仅在通信网卡使用IPv6协议时进行，若使用IPv4协议，跳过即可。
   ```
   export HYDRA_LAUNCHER_EXTRA_ARGS="-B 本节点的IPv6网卡名"
   ```

---

## 工具编译

安装完MPI软件后，需要进行HCCL性能测试工具的编译。

### 1. 配置编译依赖环境变量

安装MPICH的场景：

```
export INSTALL_DIR=/usr/local/Ascend/ascend-toolkit/latest
export PATH=/usr/local/mpich/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/mpich/lib:${INSTALL_DIR}/lib64:$LD_LIBRARY_PATH
```

安装Open MPI的场景：

```
export INSTALL_DIR=/usr/local/Ascend/ascend-toolkit/latest
export PATH=/usr/local/openmpi/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/openmpi/lib:${INSTALL_DIR}/lib64:$LD_LIBRARY_PATH
```

"INSTALL_DIR"为CANN软件安装后的文件存储路径。"/usr/local/Ascend"系默认安装位置（root用户），普通用户或自定义路径安装需相应替换。"/usr/local/mpich"及"/usr/local/openmpi"为MPI安装路径，应根据实际情况调整。

### 2. 进入HCCL性能测试工具源码路径

```
cd ${INSTALL_DIR}/tools/hccl_test
```

### 3. 编译HCCL性能测试工具

安装MPICH的场景：

```
make MPI_HOME=/usr/local/mpich ASCEND_DIR=${INSTALL_DIR}
```

安装Open MPI的场景：

```
make MPI_HOME=/usr/local/openmpi ASCEND_DIR=${INSTALL_DIR}
```

编译成功后，在"${INSTALL_DIR}/tools/hccl_test/bin"目录下生成集合通信性能测试工具的可执行文件，如all_gather_test、all_reduce_test等，每一可执行文件对应一个集合通信算子。

---

## 工具执行

### 前提条件

- 运行环境已关闭防火墙。
- 由于Master节点允许处理的并发建链数受linux内核参数"somaxconn"与"tcp_max_syn_backlog"的限制，所以，针对大规模集群组网，若"somaxconn"与"tcp_max_syn_backlog"取值较小会导致部分客户端概率性提前异常退出，导致集群初始化失败。

大规模集群组网场景下，建议开发者根据集群数量在master节点适当调整"somaxconn"与"tcp_max_syn_backlog"参数的值，例如：

```
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

### 操作步骤

#### 1. 配置HCCL Test工具启动依赖的环境变量

安装MPICH的场景：

```
export INSTALL_DIR=/usr/local/Ascend/ascend-toolkit/latest
export PATH=/usr/local/mpich/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/mpich/lib:${INSTALL_DIR}/lib64:$LD_LIBRARY_PATH
```

安装Open MPI的场景：

```
export INSTALL_DIR=/usr/local/Ascend/ascend-toolkit/latest
export PATH=/usr/local/openmpi/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/openmpi/lib:${INSTALL_DIR}/lib64:$LD_LIBRARY_PATH
```

如果环境中已存在上述环境变量，无需再次配置。

#### 2. 配置HCCL集合通信相关环境变量

##### 在训练进程拉起节点配置初始化root通信网卡相关环境变量

配置HCCL的初始化root通信网卡使用的IP协议版本以及网卡名，HCCL可通过配置的网卡名获取Host IP，完成通信域创建。

```
# 配置HCCL的初始化root通信网卡使用的IP协议版本，AF_INET：IPv4；AF_INET6：IPv6
export HCCL_SOCKET_FAMILY=AF_INET

# 支持以下格式的网卡名配置（4种规格自行选择1种即可，环境变量中可配置多个网卡，多个网卡间使用英文逗号分隔，取最先匹配到的网卡作为root网卡）

# 精确匹配网卡
export HCCL_SOCKET_IFNAME==eth0,enp0     # 使用指定的eth0或enp0网卡
export HCCL_SOCKET_IFNAME=^=eth0,enp0    # 不使用eth0与enp0网卡

# 模糊匹配网卡
export HCCL_SOCKET_IFNAME=eth,enp        # 使用所有以eth或enp为前缀的网卡
export HCCL_SOCKET_IFNAME=^eth,enp       # 不使用任何以eth或enp为前缀的网卡
```

需要注意：MPI工具执行时，会将环境变量同步到所有节点，如果参与集合通信的不同节点的网卡名字不同，例如node1的网卡名为eth1，node2的网卡名为eth2，则需要使用模糊匹配网卡的方式进行环境变量配置。

##### 调整socket建链超时等待时间

集合通信场景中，设备间socket建链超时等待时间默认值为120s，当Master节点需要建链和处理的数据量较大时，默认值120秒无法满足建链需求，需要适当进行调整。

例如，若集群组网中卡数为3K，建议调整为240s；若集群组网中卡数为5K，建议调整为600s。

```
export HCCL_CONNECT_TIMEOUT=600
```

##### 调整NPU之间共享缓冲区的大小

两个NPU之间共享数据的缓存区大小默认为200M，可通过环境变量HCCL_BUFFSIZE进行调整，单位为M，取值需要大于等于1。

集合通信网络中，每一个HCCL通信域都会占用HCCL_BUFFSIZE大小的缓存区。若集群网络中存在较多的HCCL通信域，此缓存区占用量就会增多，可能存在影响模型数据正常存放的风险，此种场景下，可通过此环境变量减少通信域占用的缓存区大小；若业务的模型数据量较小，但通信数据量较大，则可通过此环境变量增大HCCL通信域占用的缓存区大小，提升数据通信效率。

使用hccl_test工具进行性能测试的场景下，往往通信数据量较大，此种场景下，可适当增大HCCL_BUFFSIZE的值，提升数据通信效率。针对集合通信算子，当测试数据量超过HCCL_BUFFSIZE的取值时，可能会出现性能下降的情况，建议HCCL_BUFFSIZE的取值大于测试数据量。

配置示例：

```
export HCCL_BUFFSIZE=2048
```

更多环境变量可参见《环境变量参考》中的集合通信部分。

#### 3. 配置Hostfile文件

Hostfile文件用于指定需要在哪些节点上启动通信进程，Hostfile文件是文本文件，需要用户自定义。

安装MPICH的场景，仅支持通信协议IPv4，内容格式如下：

```
节点ip:每节点的进程数
```

例如，定义Hostfile文件的名称为"hostfile"，内容如下：

```
10.78.130.22:8
10.78.130.21:8
```

安装Open MPI的场景，既支持通信协议IPv4，又支持通信协议IPv6，内容格式如下：

```
节点名 slots=每节点的进程数
```

例如，定义Hostfile文件的名称为"hostfile"，内容如下：

```
node3 slots=8
node4 slots=8
```

针对单机场景，Hostfile文件可不配置。

#### 4. 执行HCCL Test工具

开发者需要在hccl_test目录下执行HCCL Test工具。

安装MPICH的场景，命令格式如下：

```
mpirun [-f hostfile] [-n number] ./bin/<executable_file> [-p npus] [-b minbytes] [-e maxbytes] [-f stepfactor] [-o operator] [-r root] [-d datatype] [-n iters] [-w warmup_iters] [-c <0/1>]
```

命令示例如下：

```
mpirun -f hostfile -n 16 ./bin/all_reduce_test -p 8 -b 8K -e 64M -f 2 -d fp32 -o sum
```

- mpirun后跟随的是MPICH命令相关参数。
- ./bin/<executable_file>后跟随的是HCCL Test工具相关参数。

关于MPICH及集合通信测试命令相关参数的详细说明可参见参数说明部分。

安装Open MPI的场景，命令格式如下：

```
mpirun [-hostfile hostfile] [-n number] [-x environment_variable_name] [--allow-run-as-root] [--mca key value] ./bin/<executable_file> [-p npus] [-b minbytes] [-e maxbytes] [-f stepfactor] [-o operator] [-r root] [-d datatype] [-n iters] [-w warmup_iters] [-c <0/1>]
```

命令示例如下：

```
mpirun -hostfile hostfile -x LD_LIBRARY_PATH -x HCCL_SOCKET_FAMILY -x HCCL_SOCKET_IFNAME -x HCCL_CONNECT_TIMEOUT -x HCCL_BUFFSIZE --allow-run-as-root --mca btl_tcp_if_include eth0 --mca opal_set_max_sys_limits 1 -n 16 ./bin/all_reduce_test -p 16 -b 8K -e 64M -i 0 -o sum -d fp32 -w 3 -n 3
```

- mpirun后跟随的是Open MPI命令相关参数。
- ./bin/<executable_file>后跟随的是HCCL Test工具相关参数。

关于Open MPI及集合通信测试命令相关参数的详细说明可参见参数说明部分。

---

## 参数说明

本节介绍HCCL Test性能测试工具执行时的相关参数说明。

### 命令格式

#### 安装MPICH的场景

```
mpirun [-f hostfile] [-n number] ./bin/<executable_file> [-p npus] [-b minbytes] [-e maxbytes] [-f stepfactor] [-o operator] [-r root] [-d datatype] [-n iters] [-w warmup_iters] [-c <0/1>]
```

#### 安装Open MPI的场景

```
mpirun [-hostfile hostfile] [-n number] [-x environment_variable_name] [--allow-run-as-root] [--mca key value] ./bin/<executable_file> [-p npus] [-b minbytes] [-e maxbytes] [-f stepfactor] [-o operator] [-r root] [-d datatype] [-n iters] [-w warmup_iters] [-c <0/1>]
```

- mpirun后跟随的是MPI命令相关参数。
- ./bin/<executable_file>后跟随的是HCCL Test工具相关参数。

### MPICH命令参数

此处仅给出MPICH工具常见参数说明，更多参数介绍可执行"./mpirun --help"命令查看帮助信息。

| 参数名 | 可选/必选 | 描述                                                                         |
| ------ | --------- | ---------------------------------------------------------------------------- |
| -f     | 可选      | Hostfile节点列表文件。单机场景下无需配置此文件；多机场景下，需要配置此文件。 |
| -n     | 必选      | 需要启动的NPU总数。                                                          |

### Open MPI命令参数

此处仅给出Open MPI工具常见参数说明。

| 参数名              | 可选/必选 | 描述                                                                                                                                                                                                                                        |
| ------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -hostfile           | 可选      | 指定Hostfile节点列表文件。单机场景下无需配置此文件；多机场景下，需要配置此文件。                                                                                                                                                            |
| -n                  | 必选      | 设置需要启动的NPU总数。                                                                                                                                                                                                                     |
| -x                  | 必选      | 指定需要传递给远程节点的环境变量名称，环境变量为执行HCCL Test命令前配置的除PATH外的所有环境变量。开发者可以通过environment_variable_name=value的形式，将环境变量与设置的值一并传递给远程节点。                                              |
| --allow-run-as-root | 可选      | 允许mpirun使用root用户执行。                                                                                                                                                                                                                |
| --mca               | 可选      | 设置mca参数，openmpi的设计以组件架构(MPI Component Architecture, MCA)为中心。常用命令包括：`--mca btl_tcp_if_include <nic_name>`（使用指定网卡进行节点间通信）；`--mca opal_set_max_sys_limits 1`（设置系统限制，集群中卡较多时建议配置）。 |

### HCCL Test工具相关参数

| 参数名                  | 可选/必选 | 描述                                                                                                                                                                                                                                            |
| ----------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ./bin/<executable_file> | 必选      | 集合通信性能测试工具的执行命令。可选文件有：all_gather_test、all_reduce_test、alltoallv_test、alltoall_test、broadcast_test、reduce_scatter_test、reduce_test、scatter_test。针对Atlas 300I Duo推理卡，仅支持all_reduce_test与all_gather_test。 |

#### 集合通信性能测试命令支持的参数

| 参数名 | 可选/必选 | 描述                                                                                                                                                       |
| ------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -p     | 必选      | 单个计算节点上参与训练的NPU个数。默认为当前节点的NPU总数。                                                                                                 |
| -b     | 可选      | 定义执行集合通信操作所使用的测试数据大小的起始值（最小值）。默认值：64M，单位：K、M、G。                                                                   |
| -e     | 可选      | 测试数据大小的结束值（最大值）。默认值：64M，单位：K、M、G。                                                                                               |
| -i/-f  | 可选      | 数据增量类型。"-i"为增量步长方式，"-f"为乘法因子方式。默认开启"-i"增量步长方式，默认步长大小计算方式为：（测试数据大小的结束值-测试数据大小的起始值）/10。 |

说明：

- 当"-b"取值等于"-e"时，每次迭代按照固定的数据量大小进行测试。
- 当"-e"的取值大于"-b"时，需要设置数据增量类型，"-i"与"-f"二选一进行配置即可。
- 当"-i"取值为0时，会按照测试数据大小起始值（即"-b"定义的数据量大小）持续测试。

例如：

假设配置示例为：`-b 100M -e 400M -f 2`

即测试数据大小起始值为100M，结束值为400M，数据增量乘法因子为2，则每次迭代都会分别取大小为"100M"、"200M"、"400M"的数据进行测试。

#### 集合通信操作参数

| 参数名 | 可选/必选 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -o     | 可选      | Reduce相关执行命令的操作类型，包含：sum、prod、max、min，默认值为sum。Reduce相关的执行命令有：all_reduce_test、reduce_scatter_test、reduce_test。                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -r     | 可选      | 执行命令为broadcast_test、reduce_test、scatter_test时，需要通过此参数指定根节点的Device ID。取值范围：[0，实际Device数量-1]。默认值为：0。                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| -d     | 可选      | HCCL执行命令支持的数据类型，默认值为fp32。针对all_reduce_test、reduce_scatter_test、reduce_test：Atlas训练系列产品支持int8、int32、int64、fp16、fp32；Atlas 300I Duo推理卡支持int8、int16、int32、fp16、fp32（其中"prod"、"max"、"min"操作不支持int16）；Atlas A2训练系列产品支持int8、int16、int32、int64、fp16、fp32、bfp16（其中"prod"操作不支持int16、bfp16）。针对broadcast_test、all_gather_test、alltoallv_test、alltoall_test、scatter_test，支持的数据类型包括：int8、int16、int32、fp16、fp32、int64、uint64、uint8、uint16、uint32、fp64、bfp16（bfp16仅Atlas A2训练系列产品支持）。 |

#### 性能测试参数

| 参数名 | 可选/必选 | 描述                                                                                                                                                                                                         |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| -n     | 可选      | 迭代次数，默认值为20。                                                                                                                                                                                       |
| -w     | 可选      | 预热迭代次数，此参数不参与性能统计，仅影响HCCL Test工具的执行耗时，默认值：5。由于前几轮迭代可能存在影响性能测试的操作（例如，首轮迭代的socket建链操作等），建议将前几轮迭代设置为预热迭代，不进入性能统计。 |

#### 结果校验参数

| 参数名 | 可选/必选 | 描述                                                                                                                                         |
| ------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| -c     | 可选      | 是否开启集合通信操作结果正确性校验。0：不开启校验；1：开启校验。默认值：1。在大规模集群场景下，开启结果校验会使HCCL Test工具的执行耗时增加。 |

---

## 结果说明

### HCCL Test工具执行结果

HCCL Test工具执行完成后，回显各字段含义如下：

- data_size：单个NPU上参与集合通信的数据量，单位为Bytes。
- aveg_time：集合通信算子执行耗时，单位为us。
- alg_bandwidth：集合通信算子执行带宽，单位为GB/s。

  说明：此处的集合通信算子执行带宽指的是算法带宽，计算方式为："集合通信数据量/耗时"。

- check_result：集合通信算子执行结果校验标识，取值为：success、failed、NULL。
  - 若执行工具时"-c"参数配置为"0"，即未开启结果校验，check_result状态为NULL。
  - 当算子计算结果出现溢出或超出可精确表达的数值范围时，不会开启结果校验，check_result状态为NULL。

### 结果校验说明

HCCL Test工具通过将算子输入初始化为固定值，并检验算子输出是否符合预期来判断通信结果是否正确。由于计算机数值表达范围和表达精度有限，针对归约类算子的乘法与加法操作，如果卡数过多，可能会出现结果溢出或超出可精确表达的数值范围的情况，导致无法准确校验，此种情况check_result状态会显示为NULL。针对归约类算子，乘与加操作在不同的算子类型与数据类型下，结果校验所能支持的最大卡数如下表所示：

| 操作类型 | 算子类型                           | INT8 | INT16 | INT32 | INT64 | FP32 | FP16 | BF16 |
| -------- | ---------------------------------- | ---- | ----- | ----- | ----- | ---- | ---- | ---- |
| 乘(prod) | AllReduce / Reduce / ReduceScatter | 6    | 14    | 30    | 62    | 127  | 15   | 127  |
| 加(sum)  | AllReduce / Reduce                 | 63   | 16383 | ~1e9  | ~1e18 | ~1e6 | 511  | 63   |
| 加(sum)  | ReduceScatter                      | 11   | 181   | 46340 | ~1e9  | 2896 | 31   | 11   |

---

## 规格约束

HCCL性能测试工具会按照用户配置的单个计算节点参与训练的NPU个数来拉起Device，假设单个计算节点上参与训练的NPU个数为x（即-p后的参数取值为x），则会从Device ID为0的设备开始，连续拉起x个Device。

### 针对Atlas训练系列产品

- 若"-p"配置为1，拉起的Device ID为：0
- 若"-p"配置为2，拉起的Device ID为：0，1
- 若"-p"配置为4，拉起的Device ID为：0，1，2，3
- 若"-p"配置为8，拉起的Device ID为：0，1，2，3，4，5，6，7

### 针对Atlas A2 训练系列产品

- 若"-p"配置为1，拉起的Device ID为：0
- 若"-p"配置为2，拉起的Device ID为：0，1
- 若"-p"配置为3，拉起的Device ID为：0，1，2
- 若"-p"配置为4，拉起的Device ID为：0，1，2，3
- 若"-p"配置为5，拉起的Device ID为：0，1，2，3，4
- 若"-p"配置为6，拉起的Device ID为：0，1，2，3，4，5
- 若"-p"配置为7，拉起的Device ID为：0，1，2，3，4，5，6
- 若"-p"配置为8，拉起的Device ID为：0，1，2，3，4，5，6，7

---

## 常见问题及解决方法

- [gethostbyname failed](#gethostbyname-failed)
- [mpi库文件链接错误](#mpi库文件链接错误)
- ["bash:orted:未找到命令"错误](#bashorted未找到命令错误)
- [HCCL Test执行时返回"retcode: 7"错误](#hccl-test执行时返回retcode-7错误)

---

## gethostbyname failed

### 问题现象

执行mpirun命令时，出现"gethostbyname failed"错误。

### 解决方法

将当前节点的IP地址和对应的主机名信息添加到"/etc/hosts"文件中。例如，在错误环境中主机名为"HW-AI-LC-1-1"，假设IP地址为172.16.0.100，则需要在"/etc/hosts"文件中添加以下信息：

```
172.16.0.100 HW-AI-LC-1-1
```

---

## mpi库文件链接错误

### 问题现象

执行mpirun命令时，会报错："error while loading shared libraries: libmpi.so.12: cannot open shared object file: No such file or directory"

### 解决方法

在环境变量LD_LIBRARY_PATH中加入mpi的lib库路径。

例如：

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/mpich/lib
```

---

## "bash:orted:未找到命令"错误

### 问题现象

使用Open MPI-4.1.5执行性能测试时，系统出现"bash: orted: 未找到命令"错误。错误信息如下：

```
bash: orted: 未找到命令
--------------------------------------------------------------------------
A daemon (pid 8793) died unexpectedly with status 127 while attempting
to launch so we are aborting.

There may be more information reported by the environment (see above).

This may be because the daemon was unable to find all the needed shared
libraries on the remote node. You may set your LD_LIBRARY_PATH to have the
location of the shared libraries on the remote nodes and this will
automatically be forwarded to the remote nodes.
--------------------------------------------------------------------------
--------------------------------------------------------------------------
mpirun noticed that the job aborted, but has no info as to the process
that caused that situation.
```

### 可能原因

集群中残留有OpenMPI进程。

### 解决方法

通过mpirun终止残留的OpenMPI进程，步骤如下：

1. 创建自定义Hostfile（例如：hostfile_1），格式如下：

```
worker-0 slots=1
worker-1 slots=1
worker-2 slots=1
worker-3 slots=1
... ...
worker-510 slots=1
worker-511 slots=1
```

所有参与通信的节点都必须列在此文件中。

2. 执行以下命令终止多余的OpenMPI进程：

```
/usr/local/openmpi-4.1.5/bin/mpirun -hostfile hostfile_1 -n 512 pkill -9 -f openmpi
```

- `-n 512`：表示集群节点数
- `hostfile_1`：引用上面定义的Hostfile

---

## HCCL Test执行时返回"retcode: 7"错误

### 问题现象

执行HCCL Test命令时，HCCL Test工具启动成功但在打印数据量、时间和带宽的标题后失败。错误信息如下：

```
the minbytes is 8192, maxbytes is 2147483648, iters is 20, warmup_iters is 5
hccl interface return errreturn err ./common/src/hccl_test_common.cc:538, retcode: 7
This is an error in init_hcclComm.
```

此错误在不同进程中重复出现。

### 可能原因

集群中与当前节点通信的节点上存在残留的hccl_test进程。

### 解决方法

使用mpirun终止多余的hccl_test进程，步骤如下：

#### 步骤1：创建Hostfile

定义自定义Hostfile（例如：`hostfile_1`），格式如下：

```
worker-0 slots=1
worker-1 slots=1
worker-2 slots=1
worker-3 slots=1
... ...
worker-510 slots=1
worker-511 slots=1
```

包含所有参与集合通信的节点。

#### 步骤2：执行终止命令

```
/usr/local/openmpi-4.1.5/bin/mpirun -hostfile hostfile_1 -n 512 pkill -9 -f hccl_test
```

参数说明：
- `-n 512`：集群中的节点数
- `hostfile_1`：步骤1中定义的Hostfile
