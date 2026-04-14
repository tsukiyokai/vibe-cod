# Ascend C算子开发指南

CANN社区版 8.5.0

文档版本 01 | 发布日期 2025-12-19

# 1 概述

## 1.1 Ascend C简介

Ascend C是CANN针对算子开发场景推出的编程语言，原生支持C和C++标准规范，兼具开发效率和运行性能。基于Ascend C编写的算子程序，通过编译器编译和运行时调度，运行在昇腾AI处理器上。使用Ascend C，开发者可以基于昇腾AI硬件，高效的实现自定义的创新算法。您可以通过Ascend C主页了解详细的内容。

Ascend C提供多层级API，满足多维场景算子开发诉求。

- 基础API：基于Tensor进行编程的C++类库API，实现单指令级抽象，为底层算子开发提供灵活控制能力。
- 高阶API：封装单核公共算法，涵盖一些常见的计算算法（如卷积、矩阵运算等），显著降低复杂算法开发门槛。
- 算子模板库：基于模板提供算子完整实现参考，简化Tiling（切分算法）开发，支撑用户自定义扩展。
- Python前端：PyAsc编程语言基于Python原生接口，提供芯片底层完备编程能力，支持基于Python接口开发高性能Ascend C算子。

[图: Ascend C软件架构层次图，从上到下分为：Ascend C Python(PyAsc)、Ascend C（包含算子模板库Cube类CATLASS/Vector类ATVOSS、高阶API、基础API、微指令API）、语言扩展层（Ascend C扩展的C API SIMD/SIMT）、毕昇编译器（基础数据类型、Builtin API）。右侧包含公共辅助函数、可视调优工具、算子工程编译脚本]

### 快速入门

从一个简单的样例出发，带您体验Ascend C算子开发的基本流程。

### 成长地图

- 环境准备
- 前置知识（编程模型）：异构并行编程模型、抽象硬件架构、核函数、编程范式
- 算子实现：矢量编程、矩阵编程、融合编程、基于高阶API
- 调试调优：功能调试（CPU域孪生调试、NPU域上板调试、CPU域调试、printf/DumpTensor、msSanitizer工具、msDebug工具）、性能调优（msProf工具）
- 算子部署：基于Kernel直调算子开发（Kernel直调 <<<...>>>）、AI框架算子适配（PyTorch框架）

### 概念原理

既涵盖基础概念供开发者快速查阅与参考，又深入剖析核心架构设计与关键技术原理，满足高阶用户的深度探索需求。

- 神经网络和算子（基础概念）
- 编程模型设计原理（技术原理）
- 内存访问原理（技术原理）
- 性能优化技术原理（技术原理）

### API参考

Ascend C提供一组类库API，开发者使用标准C++语法和类库API进行编程。

- 基础API
- 高阶API
- Utils API

### 最佳实践

结合经验总结的性能优化手段和性能优化案例：

- 性能优化建议
- FlashAttention算子性能调优案例
- Matmul算子性能调优案例
- MC2算子性能调优案例

Ascend C支持在如下产品型号使用：

- Atlas A3 训练系列产品/Atlas A3 推理系列产品
- Atlas A2 训练系列产品/Atlas A2 推理系列产品
- Atlas 200I/500 A2 推理产品
- Atlas 推理系列产品
- Atlas 训练系列产品

## 1.2 本文档组织结构

本指南作为Ascend C算子开发综合性技术文档，详细阐述编程模型、语言扩展特性、C++类库API、硬件实现、以及调试调优等。无论是入门开发者还是资深工程师，均能通过本指南充分掌握如何高效释放昇腾AI处理器的完整计算潜能。

本文档的组织结构和各章内容如下：

- 快速入门：从一个简单的样例出发，带您体验Ascend C算子开发的基本流程。
- 编程模型：介绍Ascend C编程模型和编程范式。
- 编译与运行：介绍如何编译Ascend C算子，以及算子运行需要的常用接口。
- 语言扩展层：介绍语言扩展层特性，包括内置变量和函数。
- C++类库API：介绍Ascend C多层级类库API，包括单指令封装抽象的基础API、基于单核公共算法抽象的高阶API等。
- 硬件实现：介绍Ascend C硬件实现，包括昇腾AI处理器的硬件架构、硬件规格和使用约束。
- 调试调优：介绍Ascend C算子如何进行功能调试和性能调优。

# 2 环境准备

- 进行Ascend C算子开发之前，需要安装驱动固件和CANN软件包，请参考《CANN 软件安装指南》完成环境准备。
  a. 安装驱动固件（仅昇腾设备需要），安装步骤请参见"安装NPU驱动和固件"章节。
  b. 安装CANN软件包，可参考"快速安装CANN"完成快速安装，可参考其他章节了解更多场景的安装步骤。

  > 说明：安装CANN软件后，使用CANN运行用户进行编译、运行时，需要以CANN运行用户登录环境，执行`source ${install_path}/set_env.sh`命令设置环境变量。其中${install_path}为CANN软件的安装目录，例如：/usr/local/Ascend/ascend-toolkit。

- 安装cmake。通过cmake编译Ascend C算子时，要求安装3.16及以上版本的cmake，如果版本不符合要求，可以参考如下示例安装满足要求的版本。

  示例：安装3.16.0版本的cmake（x86_64架构）。

  ```bash
  mkdir -p cmake-3.16 && wget -qO- "https://cmake.org/files/v3.16/cmake-3.16.0-Linux-x86_64.tar.gz" |
  tar --strip-components=1 -xz -C cmake-3.16
  export PATH=`pwd`/cmake-3.16/bin:$PATH
  ```

> 说明：对于Ascend C算子的开发，并非必须安装驱动固件。在非昇腾设备上，可以利用CPU仿真环境先行进行算子开发和测试，并在准备就绪后，利用昇腾设备进行加速计算。非昇腾设备的安装请参考《CANN 软件安装指南》中"附录B：常用操作 > 在非昇腾设备上安装CANN"章节。

# 3 快速入门

## 3.1 HelloWorld

本示例展示了如何使用Ascend C编写一个简单的"Hello World"程序，包括核函数（设备侧实现的入口函数）的实现、调用流程以及编译运行的完整步骤。通过本示例，您可以快速了解Ascend C的基本开发流程。完整样例请参考LINK。

代码文件hello_world.asc包括核函数实现和主函数实现。

- 核函数实现：该核函数的核心逻辑是输出"Hello World!!!"字符串。
- 主函数实现：在主函数中，进行初始化环境、申请资源、通过<<<>>>调用核函数以及释放资源等操作。完整的代码流程和逻辑可以通过代码注释查看。

```cpp
// Host侧应用程序需要包含的头文件
#include "acl/acl.h"
// Kernel侧需要包含的头文件
#include "kernel_operator.h"
__global__ __aicore__ void hello_world()
{
    // 设置Kernel类型，控制算子执行时仅启动Vector核
    KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIC_ONLY);
    AscendC::printf("Hello World!!!\n");
}

int32_t main(int argc, char const *argv[])
{
    // 初始化
    aclInit(nullptr);
    // 运行管理资源申请
    int32_t deviceId = 0;
    aclrtSetDevice(deviceId);
    aclrtStream stream = nullptr;
    aclrtCreateStream(&stream);

    // 设置参与计算的核数为1（核数可根据实际需求设置）
    constexpr uint32_t blockDim = 1;
    // 用内核调用符<<<>>>调用核函数
    hello_world<<<blockDim, nullptr, stream>>>();
    aclrtSynchronizeStream(stream);
    // 资源释放和去初始化
    aclrtDestroyStream(stream);
    aclrtResetDevice(deviceId);
    aclFinalize();
    return 0;
}
```

完成代码实现后，可以通过两种方式，对上述代码进行编译：

- 使用bisheng命令行进行编译

  ```bash
  bisheng hello_world.asc --npu-arch=dav-2201 -o demo
  ./demo
  ```

- 使用CMake进行编译

  CMake编译配置如下：

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  # find_package(ASC)是CMake中用于查找和配置Ascend C编译工具链的命令
  find_package(ASC REQUIRED)
  # 指定项目支持的语言包括ASC和CXX，ASC表示支持使用毕昇编译器对Ascend C编程语言进行编译
  project(kernel_samples LANGUAGES ASC CXX)
  add_executable(demo
      hello_world.asc
  )
  # 通过编译选项设置NPU架构
  target_compile_options(demo PRIVATE
      $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=dav-2201>
  )
  ```

  编译和运行步骤如下：

  ```bash
  mkdir -p build && cd build;
  cmake ..;make -j;
  ./demo
  ```

> 说明
> - 该样例仅支持如下型号：
>   - Atlas A3 训练系列产品/Atlas A3 推理系列产品
>   - Atlas A2 训练系列产品/Atlas A2 推理系列产品
> - --npu-arch用于指定NPU的架构版本，dav-后为架构版本号，各产品型号对应的架构版本号请通过表6-1进行查询。

运行结果如下，本样例共调度1个核，打印了核号和"Hello World!!!"等信息。

```
opType=hello_world, DumpHead: AIC-0, CoreType=AIC, block dim=1, total_block_num=1,
block_remain_len=1048416, block_initial_space=1048576, rsv=0, magic=5aa5bccd
CANN Version: xxxxxxxxx, TimeStamp: xxxxxxxxx
Hello World!!!
```

## 3.2 Add自定义算子开发

本入门教程，将会引导你完成以下任务，体验Ascend C算子开发基本流程。

1. 算子分析，明确数学表达式和计算逻辑等内容；
2. Add算子核函数开发；
3. 算子核函数运行验证。

在正式的开发之前，还需要先完成环境准备工作，开发Ascend C算子的基本流程如下图所示：

[图: 开发Ascend C算子的基本流程，包含四个步骤：环境准备 -> 算子分析 -> 核函数开发 -> 核函数运行验证]

> 说明
> - 请点击LINK获取样例代码。
> - 使用本教程只需要您具有一定的C/C++基础，在此基础上，如果您已经对Ascend C编程模型有一定的了解，您可以在实际操作的过程中加深对理论的理解；如果您还没有开始了解Ascend C编程模型，也无需担心，您可以先尝试跑通教程中的样例，参考教程最后的指引进行进一步的学习。

### 环境准备

- CANN软件安装
  开发算子前，需要先准备好开发环境和运行环境，开发环境和运行环境的介绍和具体的安装步骤可参见《CANN 软件安装指南》。
- 环境变量配置
  安装CANN软件后，使用CANN运行用户进行编译、运行时，需要以CANN运行用户登录环境，执行`source ${install_path}/set_env.sh`命令设置环境变量。其中${install_path}为CANN软件的安装目录，例如：/usr/local/Ascend/ascend-toolkit。

### 算子分析

主要分析算子的数学表达式、输入输出的数量、Shape范围以及计算逻辑的实现，明确需要调用的Ascend C接口。下文以Add算子为例，介绍具体的分析过程。

步骤1：明确算子的数学表达式及计算逻辑。

Add算子的数学表达式为：

z = x + y

计算逻辑是：从外部存储Global Memory搬运数据至内部存储Local Memory，然后使用Ascend C计算接口完成两个输入参数相加，得到最终结果，再搬运到Global Memory上。

步骤2：明确输入和输出。

- Add算子有两个输入：x与y，输出为z。
- 本样例中算子输入支持的数据类型为float，算子输出的数据类型与输入数据类型相同。
- 算子输入支持的shape为（8，2048），输出shape与输入shape相同。
- 算子输入支持的format为：ND。

步骤3：确定核函数名称和参数。

- 本样例中核函数命名为add_custom。
- 根据对算子输入输出的分析，确定核函数有3个参数x，y，z；x，y为输入参数，z为输出参数。

步骤4：确定算子实现所需接口。

- 实现涉及外部存储和内部存储间的数据搬运，查看Ascend C API参考中的数据搬运接口，需要使用DataCopy来实现数据搬移。
- 本样例只涉及矢量计算的加法操作，查看Ascend C API参考中的矢量计算接口矢量计算，初步分析可使用Add接口实现x+y。
- 计算中使用到的Tensor数据结构，使用AllocTensor、FreeTensor进行申请和释放。
- 并行流水任务之间使用Queue队列完成同步，会使用到EnQue、DeQue等接口。

----结束

通过以上分析，得到Ascend C Add算子的设计规格如下：

| 算子类型（OpType） | AddCustom |
|---|---|
| 算子输入 | name: x, shape: (8, 2048), data type: float, format: ND |
| | name: y, shape: (8, 2048), data type: float, format: ND |
| 算子输出 | name: z, shape: (8, 2048), data type: float, format: ND |
| 核函数名称 | add_custom |
| 使用的主要接口 | DataCopy：数据搬运接口 |
| | Add：矢量基础算术接口 |
| | AllocTensor、FreeTensor：内存管理接口 |
| | EnQue、DeQue接口：Queue队列管理接口 |
| 算子实现文件名称 | add_custom.asc |

### 核函数开发

完成环境准备和初步的算子分析后，即可开始Ascend C核函数的开发。开发之前请先从LINK获取样例代码，以下样例代码在add_custom.asc中实现。

本样例中使用多核并行计算，即把数据进行分片，分配到多个核上进行处理。Ascend C核函数是在一个核上的处理函数，所以只处理部分数据。分配方案是：假设共启用8个核，数据整体长度为8 * 2048个元素，平均分配到8个核上运行，每个核上处理的数据大小为2048个元素。对于单核上的处理数据，可以进行数据切块，实现对数据的流水并行处理。

步骤1：根据分配方案设计一个结构体AddCustomTilingData，用于保存并行数据切分相关的参数。AddCustomTilingData定义了两个参数：totalLength、tileNum。totalLength指待处理的数据总大小为（8 * 2048）个元素，tileNum指每个核需要计算的数据块个数。

```cpp
struct AddCustomTilingData
{
    uint32_t totalLength;
    uint32_t tileNum;
};
```

步骤2：根据核函数定义和调用中介绍的规则进行核函数的定义，并在核函数中调用算子类的Init和Process函数，算子类实现在后续步骤中介绍。

```cpp
__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIV_ONLY); // 设置Kernel类型为Vector核（用于矢量计算）
    KernelAdd op;
    op.Init(x, y, z, tiling.totalLength, tiling.tileNum);
    op.Process();
}
```

- 使用`__global__`函数类型限定符来标识它是一个核函数，可以被<<<>>>调用；使用`__aicore__`函数类型限定符来标识该核函数在设备端AI Core上执行。指针入参变量需要增加变量类型限定符`__gm__`，表明该指针变量指向Global Memory上某块内存地址。为了统一表达，使用GM_ADDR宏来修饰入参，GM_ADDR宏定义如下：
  `#define GM_ADDR __gm__ uint8_t*`
- 算子类的Init函数，完成内存初始化相关工作，Process函数完成算子实现的核心逻辑。

步骤3：根据矢量编程范式实现算子类，本样例中定义KernelAdd算子类，其具体成员如下：

```cpp
class KernelAdd {
public:
    __aicore__ inline KernelAdd(){}
    // 初始化函数，完成内存初始化相关操作
    __aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, uint32_t totalLength, uint32_t tileNum)
    // 核心处理函数，实现算子逻辑，调用私有成员函数CopyIn、Compute、CopyOut完成矢量算子的三级流水操作
    __aicore__ inline void Process(){}

private:
    // 搬入函数，从Global Memory搬运数据至Local Memory，被核心Process函数调用
    __aicore__ inline void CopyIn(int32_t progress){}
    // 计算函数，完成两个输入参数相加，得到最终结果，被核心Process函数调用
    __aicore__ inline void Compute(int32_t progress){}
    // 搬出函数，将最终结果从Local Memory搬运到Global Memory上，被核心Process函数调用
    __aicore__ inline void CopyOut(int32_t progress){}

private:
    AscendC::TPipe pipe; // TPipe内存管理对象
    AscendC::TQue<AscendC::TPosition::VECIN, BUFFER_NUM> inQueueX, inQueueY; // 输入数据Queue队列管理对象，TPosition为VECIN
    AscendC::TQue<AscendC::TPosition::VECOUT, BUFFER_NUM> outQueueZ; // 输出数据Queue队列管理对象，TPosition为VECOUT
    AscendC::GlobalTensor<float> xGm; // 管理输入输出Global Memory内存地址的对象，其中xGm, yGm为输入，zGm为输出
    AscendC::GlobalTensor<float> yGm;
    AscendC::GlobalTensor<float> zGm;
    uint32_t blockLength; // 每个核的计算数据长度
    uint32_t tileNum; // 每个核需要计算的数据块个数
    uint32_t tileLength; // 每个核内每个数据块的长度
};
```

[图: 核函数调用关系图。add_custom调用Init和Process，Process内部调用CopyIn、Compute、CopyOut]

由此可见除了Init函数完成初始化外，Process中完成了对流水任务"搬入、计算、搬出"的调用，开发者可以重点关注三个流水任务的实现。

步骤4：初始化函数Init主要完成以下内容：设置输入输出Global Tensor的Global Memory内存地址，通过TPipe内存管理对象为输入输出Queue分配内存。

上文我们介绍到，本样例将数据切分成8块，平均分配到8个核上运行，每个核上处理的数据大小为2048个元素。那么我们是如何实现这种切分的呢？

每个核上处理的数据地址需要在起始地址上增加GetBlockIdx() * blockLength（每个block处理的数据长度）的偏移来获取。这样也就实现了多核并行计算的数据切分。

以输入x为例，x + blockLength * GetBlockIdx()即为单核处理程序中x在Global Memory上的内存偏移地址，获取偏移地址后，使用GlobalTensor类的SetGlobalBuffer接口设定该核上Global Memory的起始地址以及长度。

[图: 多核并行处理示意图，totalLength = 8 * 2048，分成8个block，每个blockLength = 2048，blockIdx从0到7]

上面已经实现了多核数据的切分，那么单核上的处理数据如何进行切分？

对于单核上的处理数据，可以进行数据切块（Tiling），在本示例中，仅作为参考，将数据切分成8块（并不意味着8块就是性能最优）。切分后的每个数据块再次切分成2块，即可开启double buffer，实现流水线之间的并行。

这样单核上的数据（2048个数）被切分成16块，每块tileLength（128）个数据。TPipe为inQueueX分配了两块大小为tileLength * sizeof(float)个字节的内存块，每个内存块能容纳tileLength（128）个float类型数据。

[图: 单核数据切分示意图，blockLength=2048被切分为16个tile，每个tileLength=128]

具体的初始化函数代码如下：

```cpp
// Kernel侧所需头文件
#include "kernel_operator.h"

constexpr int32_t BUFFER_NUM = 2; // tensor num for each queue

__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, uint32_t totalLength, uint32_t tileNum)
{
    this->blockLength = totalLength / AscendC::GetBlockNum(); // length computed of each core
    this->tileNum = tileNum; // split data into 8 tiles for each core
    this->tileLength = this->blockLength / tileNum / BUFFER_NUM; // separate to 2 parts, due to double buffer
    // get start index for current core, core parallel
    xGm.SetGlobalBuffer((__gm__ float *)x + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    yGm.SetGlobalBuffer((__gm__ float *)y + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    zGm.SetGlobalBuffer((__gm__ float *)z + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    // pipe alloc memory to queue, the unit is Bytes
    pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(float));
    pipe.InitBuffer(inQueueY, BUFFER_NUM, this->tileLength * sizeof(float));
    pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(float));
}
```

步骤5：基于矢量编程范式，将核函数的实现分为3个基本任务：CopyIn，Compute，CopyOut。Process函数中通过如下方式调用这三个函数。

```cpp
__aicore__ inline void Process()
{
    // loop count need to be doubled, due to double buffer
    int32_t loopCount = this->tileNum * BUFFER_NUM;
    // tiling strategy, pipeline parallel
    for (int32_t i = 0; i < loopCount; i++) {
        CopyIn(i);
        Compute(i);
        CopyOut(i);
    }
}
```

1. CopyIn函数实现。
   a. 使用DataCopy接口将GlobalTensor数据拷贝到LocalTensor。
   b. 使用EnQue将LocalTensor放入VecIn的Queue中。

   ```cpp
   __aicore__ inline void CopyIn(int32_t progress)
   {
       AscendC::LocalTensor<float> xLocal = inQueueX.AllocTensor<float>();
       AscendC::LocalTensor<float> yLocal = inQueueY.AllocTensor<float>();
       // copy progress_th tile from global tensor to local tensor
       AscendC::DataCopy(xLocal, xGm[progress * this->tileLength], this->tileLength);
       AscendC::DataCopy(yLocal, yGm[progress * this->tileLength], this->tileLength);
       // enque input tensors to VECIN queue
       inQueueX.EnQue(xLocal);
       inQueueY.EnQue(yLocal);
   }
   ```

2. Compute函数实现。
   a. 使用DeQue从VecIn中取出LocalTensor。
   b. 使用Ascend C接口Add完成矢量计算。
   c. 使用EnQue将计算结果LocalTensor放入到VecOut的Queue中。
   d. 使用FreeTensor将释放不再使用的LocalTensor。

   ```cpp
   __aicore__ inline void Compute(int32_t progress)
   {
       // deque input tensors from VECIN queue
       AscendC::LocalTensor<float> xLocal = inQueueX.DeQue<float>();
       AscendC::LocalTensor<float> yLocal = inQueueY.DeQue<float>();
       AscendC::LocalTensor<float> zLocal = outQueueZ.AllocTensor<float>();
       // call Add instr for computation
       AscendC::Add(zLocal, xLocal, yLocal, this->tileLength);
       // enque the output tensor to VECOUT queue
       outQueueZ.EnQue<float>(zLocal);
       // free input tensors for reuse
       inQueueX.FreeTensor(xLocal);
       inQueueY.FreeTensor(yLocal);
   }
   ```

3. CopyOut函数实现。
   a. 使用DeQue接口从VecOut的Queue中取出LocalTensor。
   b. 使用DataCopy接口将LocalTensor拷贝到GlobalTensor上。
   c. 使用FreeTensor将不再使用的LocalTensor进行回收。

   ```cpp
   __aicore__ inline void CopyOut(int32_t progress)
   {
       // deque output tensor from VECOUT queue
       AscendC::LocalTensor<float> zLocal = outQueueZ.DeQue<float>();
       // copy progress_th tile from local tensor to global tensor
       AscendC::DataCopy(zGm[progress * this->tileLength], zLocal, this->tileLength);
       // free output tensor for reuse
       outQueueZ.FreeTensor(zLocal);
   }
   ```

----结束

### 核函数运行验证

完成Kernel侧核函数开发后，即可编写Host侧的核函数调用程序。实现从Host侧的APP程序调用算子，执行计算过程。

步骤1：Host侧应用程序框架的编写。

```cpp
// Host侧应用程序需要包含的头文件
#include "data_utils.h"
#include "acl/acl.h"
// Kernel侧需要包含的头文件
#include "kernel_operator.h"
// 核函数开发部分
...

__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIV_ONLY);
    KernelAdd op;
    op.Init(x, y, z, tiling.totalLength, tiling.tileNum);
    op.Process();
}

// 通过<<<...>>>内核调用符调用算子
std::vector<float> kernel_add(std::vector<float> &x, std::vector<float> &y)
{
    ...
}

// 计算结果比对
uint32_t VerifyResult(std::vector<float> &output, std::vector<float> &golden)
{
    auto printTensor = [](std::vector<float> &tensor, const char *name) {
        constexpr size_t maxPrintSize = 20;
        std::cout << name << ": ";
        std::copy(tensor.begin(), tensor.begin() + std::min(tensor.size(), maxPrintSize),
            std::ostream_iterator<float>(std::cout, " "));
        if (tensor.size() > maxPrintSize) {
            std::cout << "...";
        }
        std::cout << std::endl;
    };
    printTensor(output, "Output");
    printTensor(golden, "Golden");
    if (std::equal(output.begin(), output.end(), golden.begin())) {
        std::cout << "[Success] Case accuracy is verification passed." << std::endl;
        return 0;
    } else {
        std::cout << "[Failed] Case accuracy is verification failed!" << std::endl;
        return 1;
    }
    return 0;
}

// 算子验证主程序
int32_t main(int32_t argc, char *argv[])
{
    constexpr uint32_t totalLength = 8 * 2048;
    constexpr float valueX = 1.2f;
    constexpr float valueY = 2.3f;
    std::vector<float> x(totalLength, valueX);
    std::vector<float> y(totalLength, valueY);

    std::vector<float> output = kernel_add(x, y);

    std::vector<float> golden(totalLength, valueX + valueY);
    return VerifyResult(output, golden);
}
```

步骤2：编写通过<<<...>>>内核调用符调用算子的代码。

[图: 调用步骤流程图，包含：初始化 -> 运行管理资源申请 -> 分配Host内存并进行数据初始化 -> 分配Device内存并将数据从Host上拷贝到Device上 -> 用内核调用符<<<>>>调用核函数完成指定的运算并同步等待 -> 将Device上的运算结果拷贝回Host -> 释放申请的资源 -> 去初始化]

如下示例中的acl API使用方法请参考"acl API（C&C++）"章节。

```cpp
std::vector<float> kernel_add(std::vector<float> &x, std::vector<float> &y)
{
    constexpr uint32_t blockDim = 8;
    uint32_t totalLength = x.size();
    size_t totalByteSize = totalLength * sizeof(float);
    int32_t deviceId = 0;
    aclrtStream stream = nullptr;
    AddCustomTilingData tiling = {/*totalLength:*/totalLength, /*tileNum:*/8};
    uint8_t *xHost = reinterpret_cast<uint8_t *>(x.data());
    uint8_t *yHost = reinterpret_cast<uint8_t *>(y.data());
    uint8_t *zHost = nullptr;
    uint8_t *xDevice = nullptr;
    uint8_t *yDevice = nullptr;
    uint8_t *zDevice = nullptr;

    // 初始化
    aclInit(nullptr);
    // 运行管理资源申请
    aclrtSetDevice(deviceId);
    aclrtCreateStream(&stream);
    // 分配Host内存
    aclrtMallocHost((void **)(&zHost), totalByteSize);
    // 分配Device内存
    aclrtMalloc((void **)&xDevice, totalByteSize, ACL_MEM_MALLOC_HUGE_FIRST);
    aclrtMalloc((void **)&yDevice, totalByteSize, ACL_MEM_MALLOC_HUGE_FIRST);
    aclrtMalloc((void **)&zDevice, totalByteSize, ACL_MEM_MALLOC_HUGE_FIRST);
    // 将Host上的输入数据拷贝到Device侧
    aclrtMemcpy(xDevice, totalByteSize, xHost, totalByteSize, ACL_MEMCPY_HOST_TO_DEVICE);
    aclrtMemcpy(yDevice, totalByteSize, yHost, totalByteSize, ACL_MEMCPY_HOST_TO_DEVICE);
    // 用内核调用符<<<...>>>调用核函数完成指定的运算
    add_custom<<<blockDim, nullptr, stream>>>(xDevice, yDevice, zDevice, tiling);
    aclrtSynchronizeStream(stream);
    // 将Device上的运算结果拷贝回Host
    aclrtMemcpy(zHost, totalByteSize, zDevice, totalByteSize, ACL_MEMCPY_DEVICE_TO_HOST);
    std::vector<float> z((float *)zHost, (float *)(zHost + totalLength));
    // 释放申请的资源
    aclrtFree(xDevice);
    aclrtFree(yDevice);
    aclrtFree(zDevice);
    aclrtFreeHost(zHost);
    // 去初始化
    aclrtDestroyStream(stream);
    aclrtResetDevice(deviceId);
    aclFinalize();
    return z;
}
```

步骤3：CMake编译配置如下：

```cmake
cmake_minimum_required(VERSION 3.16)
# find_package(ASC)是CMake中用于查找和配置Ascend C编译工具链的命令
find_package(ASC REQUIRED)
# 指定项目支持的语言包括ASC和CXX，ASC表示支持使用毕昇编译器对Ascend C编程语言进行编译
project(kernel_samples LANGUAGES ASC CXX)

add_executable(demo
    add_custom.asc
)

# 通过编译选项设置NPU架构
target_compile_options(demo PRIVATE
    $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=dav-2201>
)
```

步骤4：编译和运行步骤如下

```bash
mkdir -p build && cd build;
cmake ..;make -j;
./demo
```

> 说明
> - 该样例仅支持如下型号：
>   - Atlas A3 训练系列产品/Atlas A3 推理系列产品
>   - Atlas A2 训练系列产品/Atlas A2 推理系列产品
> - --npu-arch用于指定NPU的架构版本，dav-后为架构版本号，各产品型号对应的架构版本号请通过表6-1进行查询。

----结束

### 接下来的引导

如果您对教程中的多核并行、流水编程等概念不了解，导致阅读过程有些吃力，可以参考4 编程模型学习基本概念，再来回顾本教程；如果您已经了解相关概念，并跑通了该样例，您可以参考10.1.2 矢量编程了解Ascend C矢量编程中的更多细节。

# 4 编程模型

## 4.1 异构并行编程模型

### Host-Device异构协同机制

Ascend C异构并行编程模型是为应对异构计算架构的挑战而设计的，旨在解决传统编程模型在处理复杂计算任务时的效率和可扩展性问题。

异构计算架构分为Host侧和Device侧（Device侧对应昇腾AI处理器），两者协同完成计算任务。Host侧主要负责运行时管理，包括存储管理、设备管理以及Stream管理等，确保任务的高效调度与资源的合理分配；Device侧，会执行开发者基于Ascend C语法编写的Kernel核函数，主要完成批量数据的矩阵运算、向量运算等计算密集型的任务，用于计算加速。

如下图所示，当一个Kernel下发到AI Core（昇腾AI处理器的计算核心）上执行时，运行时管理模块根据开发者设置的核数和任务类型启动对应的Task，该Task从Host加载到Device的Stream运行队列，调度单元会把就绪的Task分配到空闲AI Core上执行。这里将需要处理的数据拆分并同时在多个计算核心上运行的方式（也就是下文介绍的SPMD并行计算），可以获取更高的性能。

[图: Kernel调度示意图。Host加载Task到Device，Device中包含Stream队列、调度单元和多个AI Core（AI Core 0, AI Core 1, ... AI Core N）]

Host和Device拥有不同的内存空间，Host无法直接访问Device内存，反之亦然。所以，输入数据需要从Host侧拷贝至Device侧内存空间，供Device侧进行计算，输出结果需要从Device侧内存拷贝回Host侧，便于在Host侧继续使用。

> 说明：关于运行时管理的详细介绍请参考"运行时管理"章节。

### SPMD并行计算

Ascend C算子编程是SPMD（Single-Program Multiple-Data）编程，通俗来讲就是"一份代码，多处执行，处理的数据不同"。SPMD是一种常用的并行计算的方法，是提高计算速度的有效手段。

具体到Ascend C编程模型中的应用，是将需要处理的数据拆分并同时在多个计算核心（类比于上文介绍中的多个进程）上运行，从而获取更高的性能。多个AI Core共享相同的指令代码，每个核上的运行实例唯一的区别是block_idx不同，每个核通过不同的block_idx来识别自己的身份。block的概念类似于上文中进程的概念，block_idx就是标识进程唯一性的进程ID。并行计算过程如下图所示。

[图: SPMD并行计算示意图。Global Memory中的数据按block分配给AI Core 0(block_idx=0)、AI Core 1(block_idx=1)、...、AI Core N-1(block_idx=N-1)]

下面的代码片段取自于Ascend C Add算子的实现代码，算子被调用时，所有的计算核心都执行相同的实现代码，入口函数的入参也是相同的。每个核上处理的数据地址需要在起始地址上增加GetBlockIdx()*BLOCK_LENGTH（每个核处理的数据长度）的偏移来获取。这样也就实现了多核并行计算的数据切分。代码中GetBlockIdx接口用于获取每个核的block_idx。

```cpp
class KernelAdd {
public:
    __aicore__ inline KernelAdd() {}
    __aicore__ void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z)
    {
        // 不同核根据各自的block_idx设置数据地址
        xGm.SetGlobalBuffer((__gm__ half*)x + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
        yGm.SetGlobalBuffer((__gm__ half*)y + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
        zGm.SetGlobalBuffer((__gm__ half*)z + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
        // Queue初始化，单位为字节
        pipe.InitBuffer(inQueueX, BUFFER_NUM, TILE_LENGTH * sizeof(half));
        pipe.InitBuffer(inQueueY, BUFFER_NUM, TILE_LENGTH * sizeof(half));
        pipe.InitBuffer(outQueueZ, BUFFER_NUM, TILE_LENGTH * sizeof(half));
    }
    ...
}

// 实现核函数
__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z)
{
    // 初始化算子类，算子类提供算子初始化和核心处理等方法
    KernelAdd op;
    // 初始化函数，获取该核函数需要处理的输入输出地址，同时完成必要的内存初始化工作
    op.Init(x, y, z);
    // 核心处理函数，完成算子的数据搬运与计算等核心逻辑
    op.Process();
}
```

## 4.2 抽象硬件架构

AI Core是昇腾AI处理器的计算核心，昇腾AI处理器内部有多个AI Core。本章节将介绍AI Core的并行计算架构抽象，该抽象架构屏蔽了不同硬件之间的差异。使用Ascend C进行编程时，基于抽象硬件架构，可以简化硬件细节，显著降低开发门槛。如需了解更详细的硬件架构信息或者原理，请参考8 硬件实现。

[图: 抽象硬件架构图。AI Core内部包含：算子指令序列 -> Scalar计算单元（发出指令流、数据流、同步信号），下方有Vector计算单元、Cube计算单元、DMA搬运单元，各有指令队列。底部为Local Memory，更下方为Global Memory]

AI Core中包含计算单元、存储单元、搬运单元等核心组件。

- 计算单元包括了三种基础计算资源：Cube计算单元、Vector计算单元和Scalar计算单元。
- 存储单元包括内部存储和外部存储：
  - AI Core的内部存储，统称为Local Memory，对应的数据类型为LocalTensor。
  - AI Core能够访问的外部存储称之为Global Memory，对应的数据类型为GlobalTensor。
- DMA（Direct Memory Access）搬运单元：负责数据搬运，包括Global Memory和Local Memory之间的数据搬运，以及不同层级Local Memory之间的数据搬运。

AI Core内部核心组件及组件功能详细说明如下表。

| 组件分类 | 组件名称 | 组件功能 |
|---|---|---|
| 计算单元 | Scalar | 执行地址计算、循环控制等标量计算工作，并把向量计算、矩阵计算、数据搬运、同步指令发射给对应单元执行。 |
| | Vector | 负责执行向量运算。 |
| | Cube | 负责执行矩阵运算。 |
| 存储单元 | Local Memory | AI Core的内部存储。 |
| 搬运单元 | DMA（Direct Memory Access） | 负责数据搬运，包括Global Memory和Local Memory之间的数据搬运以及不同层级Local Memory之间的数据搬运。 |

开发者在理解硬件架构的抽象时，需要重点关注如下异步指令流、同步信号流、计算数据流三个过程：

- AI Core内部的异步并行计算过程：Scalar计算单元读取指令序列，并把向量计算、矩阵计算、数据搬运指令发射给对应单元的指令队列，向量计算单元、矩阵计算单元、数据搬运单元异步的并行执行接收到的指令。该过程可以参考图1中蓝色箭头所示的指令流。
- 不同的指令间可能存在依赖关系，为了保证不同指令队列间的指令按照正确的逻辑关系执行，Scalar计算单元也会给对应单元下发同步指令。各单元之间的同步过程可以参考图1中绿色箭头所示的同步信号流。
- AI Core内部数据处理的基本过程：DMA搬入单元将数据从Global Memory搬运到Local Memory，Vector/Cube计算单元完成数据计算，并把计算结果写回Local Memory，DMA搬出单元把处理好的数据从Local Memory搬运回Global Memory。该过程可以参考图1中红色箭头所示的数据流。

## 4.3 核函数

核函数（Kernel Function）是Ascend C算子设备侧实现的入口。Ascend C允许用户使用C/C++函数的语法扩展来编写设备端的运行代码，用户在核函数中进行数据访问和计算操作，由此实现该算子的所有功能。区别于普通的C++函数调用时仅执行一次，当核函数被调用时，多个核都执行相同的核函数代码，具有相同的函数入参，并行执行。

核函数定义时需要使用函数类型限定符`__global__`和`__aicore__`；其指针入参变量需要增加变量类型限定符`__gm__`，表明该指针变量指向Global Memory上某处内存地址；使用<<<...>>>内核调用符调用执行核函数，并指定调用时的执行核数。

以下是一个Add算子的核函数示例，完整样例请参考10.1.2 矢量编程中的Add算子示例。

```cpp
// 实现核函数
__global__ __aicore__ void add_custom(__gm__ uint8_t* x, __gm__ uint8_t* y, __gm__ uint8_t* z)
{
    // 初始化算子类，算子类提供算子初始化和核心处理等方法
    KernelAdd op;
    // 初始化函数，获取该核函数需要处理的输入输出地址，同时完成必要的内存初始化工作
    op.Init(x, y, z);
    // 核心处理函数，完成算子的数据搬运与计算等核心逻辑
    op.Process();
}

// 调用核函数
void add_custom_do(uint32_t blockDim, void* l2ctrl, void* stream, uint8_t* x, uint8_t* y, uint8_t* z)
{
    add_custom<<<blockDim, l2ctrl, stream>>>(x, y, z);
}
```

### 核函数定义和调用

定义核函数时需要遵循以下规则。

- 使用函数类型限定符
  除了需要按照C/C++函数声明的方式定义核函数之外，还要为核函数加上额外的函数类型限定符，包含`__global__`和`__aicore__`。

  使用`__global__`函数类型限定符来标识它是一个核函数，可以被<<<>>>调用；使用`__aicore__`函数类型限定符来标识该核函数在设备端AI Core上执行：

  ```cpp
  __global__ __aicore__ void kernel_name(argument list);
  ```

  编程中使用到的函数可以分为三类：核函数（device侧执行）、host侧执行函数、device侧执行函数（除核函数之外）。下图以Kernel直调算子开发方式为例描述三者的调用关系：

  - host侧执行函数可以调用同类的host执行函数，也就是通用C/C++编程中的函数调用；也可以通过<<<...>>>调用核函数。
  - device侧执行函数（除核函数之外）可以调用同类的device侧执行函数。
  - 核函数可以调用device侧执行函数（除核函数之外）。

  [图: 核函数、host侧执行函数、device侧执行函数调用关系图]

- 使用变量类型限定符
  指针入参变量需要增加变量类型限定符`__gm__`，表明该指针变量指向Global Memory上某处内存地址。

- 其他规则或建议
  a. 规则：核函数必须具有void返回类型。
  b. 规则：仅支持入参为指针或C/C++内置数据类型（Primitive data types），如：half* s0、float* s1、int32_t c。
  c. 建议：为了统一表达，建议使用GM_ADDR宏来修饰入参，GM_ADDR宏定义如下：
     `#define GM_ADDR __gm__ uint8_t*`
     使用GM_ADDR修饰入参的样例如下：
     `extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z)`
     这里统一使用uint8_t类型的指针，在后续的使用中需要将其转化为实际的指针类型。

常见的函数调用方式是如下的形式：

```cpp
function_name(argument list);
```

核函数使用内核调用符<<<...>>>这种语法形式，来规定核函数的执行配置：

```cpp
kernel_name<<<blockDim, l2ctrl, stream>>>(argument list);
```

内核调用符仅可在NPU侧编译时调用，CPU侧编译无法识别该符号。

执行配置由3个参数决定：

- blockDim，规定了核函数将会在几个核上执行。每个执行该核函数的核会被分配一个逻辑ID，即block_idx，可以在核函数的实现中调用GetBlockIdx来获取block_idx；

  > 说明
  > blockDim是逻辑核的概念，取值范围为[1,65535]。为了充分利用硬件资源，一般设置为物理核的核数或其倍数。
  > - 对于耦合模式和分离模式，blockDim在运行时的意义和设置规则有一些区别，具体说明如下：
  >   - 耦合模式：由于其Vector、Cube单元是集成在一起的，blockDim用于设置启动多个AI Core核实例执行，不区分Vector、Cube。AI Core的核数可以通过GetCoreNumAiv或者GetCoreNumAic获取。
  >   - 分离模式
  >     - 针对仅包含Vector计算的算子，blockDim用于设置启动多少个Vector（AIV）实例执行，比如某款AI处理器上有40个Vector核，建议设置为40。
  >     - 针对仅包含Cube计算的算子，blockDim用于设置启动多少个Cube（AIC）实例执行，比如某款AI处理器上有20个Cube核，建议设置为20。
  >     - 针对Vector/Cube融合计算的算子，启动时，按照AIV和AIC组合启动，比如某款AI处理器上有40个Vector核和20个Cube核，一个组合是2个Vector核和1个Cube核，建议设置为20，此时会启动20个组合，即40个vector核和20个Cube核。注意：该场景下，设置的blockDim逻辑核的核数不能超过物理核（2个Vector核和1个Cube核组合为1个物理核）的物理核数。
  >     - AIC/AIV的核数分别通过GetCoreNumAic和GetCoreNumAiv接口获取。
  > - 如果开发者使用了Device资源限制特性，那么算子设置的blockDim不应超过PlatformAscendC提供核数的API（GetCoreNum/GetCoreNumAic/GetCoreNumAiv等）返回的核数。例如，使用aclrtSetStreamResLimit设置Stream级别的Vector核数量为8，那么GetCoreNumAiv接口返回值为8，针对Vector算子设置的blockDim不应超过8，否则会抢占其他Stream的资源，导资源限制失效。

- l2ctrl，保留参数，暂时设置为固定值nullptr，开发者无需关注；

- stream，类型为aclrtStream，stream用于维护一些异步操作的执行顺序，确保按照应用程序中的代码调用顺序在device上执行。stream创建等管理接口请参考"Stream管理"章节。

如下名为add_custom的核函数，实现两个矢量的相加，调用示例如下：

```cpp
// blockDim设置为8表示在8个核上调用了add_custom核函数，每个核都会独立且并行地执行该核函数，该核函数的参数列表为x，y，z。
add_custom<<<8, nullptr, stream>>>(x, y, z);
```

核函数的调用是异步的，核函数的调用结束后，控制权立刻返回给主机端，可以调用以下aclrtSynchronizeStream函数来强制主机端程序等待所有核函数执行完毕。

```cpp
aclError aclrtSynchronizeStream(aclrtStream stream);
```

### 模板核函数定义和调用

支持开发者使用模板定义核函数，核函数定义示例如下，它有两个模板参数：a和T。a是一个非类型模板参数，T是一个类型模板参数。

```cpp
template<int a, typename T>
__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z)
{
    ...
    AscendC::printf("Print Template a: %d\n", a);
    ...
    xGm.SetGlobalBuffer((__gm__ T*)x + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
    yGm.SetGlobalBuffer((__gm__ T*)y + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
    zGm.SetGlobalBuffer((__gm__ T*)z + BLOCK_LENGTH * AscendC::GetBlockIdx(), BLOCK_LENGTH);
    ...
}
```

模板核函数的调用方式如下：add_custom<20, float>这部分代码调用了名为add_custom的核函数，并为其模板参数提供了具体值。

```cpp
add_custom<20, float><<<blockDim, nullptr, stream>>>(x, y, z);
```

## 4.4 编程范式

### 4.4.1 基于TPipe和TQue的编程范式

#### 4.4.1.1 典型算子的编程范式

编程范式描述了算子核函数实现的固定流程，基于编程范式进行编程，可以快速搭建算子实现的代码框架。

根据4.2 抽象硬件架构，AI Core内部的执行单元异步并行地执行接收到的指令，各执行单元配合，以一种流水线的方式完成完整的算子执行过程。

通过下图可以更直观地理解流水并行的概念。示意图中，从输入数据到输出数据需要经过3个阶段任务的处理（T1、T2、T3），多个执行单元并行处理，每个执行单元只会专注于一个任务的处理，会处理所有的数据分片；执行单元完成对某个数据分片的处理后，将其加入到通信队列，下一个执行单元空闲时就会从队列中取出数据继续处理；可以类比为生产流水线中的工人只完成某一项固定工序，完成后就交由下一项工序负责人继续处理。

[图: 流水线并行示意图，输入 -> 执行单元1处理T1 -> 通信队列 -> 执行单元2处理T2 -> 通信队列 -> 执行单元3处理T3 -> 输出]

Ascend C编程范式正是这样一种流水线式的编程范式，把算子核内的处理程序，分成多个流水任务，通过队列（TQue）完成任务间通信和同步，并通过统一的资源管理模块（TPipe）来统一管理内存、事件等资源。

下文将从三种典型类型的算子类型出发，对这种基于TPipe和TQue的编程范式进行详细介绍。

- 矢量编程范式
- 矩阵编程范式
- 融合算子编程范式

#### 矢量编程范式

[图: 矢量编程范式数据流图。输入 -> CopyIn(搬入单元/GM -> LocalMemory VECIN) -> 通信队列 -> Compute(计算单元) -> 通信队列 -> CopyOut(搬出单元/LocalMemory VECOUT -> GM) -> 输出]

如上图所示，矢量编程范式把算子的实现流程分为3个基本任务：CopyIn，Compute，CopyOut。

- CopyIn负责搬入操作：将输入数据从Global Memory搬运到Local Memory（VECIN用于表达矢量计算搬入数据的存放位置），完成搬运后执行入队列操作；
- Compute负责矢量指令计算操作：完成队列出队后，从Local Memory获取数据并计算，计算完成后执行入队操作；
- CopyOut负责搬出操作：完成队列出队后，将计算结果从Local Memory（VECOUT用于表达矢量计算搬出数据的存放位置）搬运到Global Memory。

上文中提到的VECIN/VECOUT是TPosition的概念。Ascend C管理不同层级的物理内存时，用一种抽象的逻辑位置（TPosition）来表达各级别的存储，代替了芯片上物理存储的概念，达到隐藏硬件架构的目的。除了VECIN/VECOUT，矢量编程中还会使用到VECCALC，一般在定义临时变量时使用此位置。TPosition与物理内存的映射关系请参考表1。

从编程的角度来讲，具体流程（如下文的伪代码）和流程图如下：

[图: 矢量编程范式详细流程图，展示Stage1:CopyIn、Stage2:Compute、Stage3:CopyOut三个阶段的AllocTensor、DataCopy、EnQue、DeQue、Compute、FreeTensor操作]

```cpp
AscendC::TPipe pipe;                     // 创建全局的资源管理
AscendC::TQue<AscendC::TPosition::VecIn, 1> queIn; // 创建CopyIn阶段的队列
AscendC::TQue<AscendC::TPosition::VecOut, 1> queOut;// 创建CopyOut阶段的队列
// Init 阶段
pipe.InitBuffer(queIn, 2, 1024);         // 开启DoubleBuffer，将待处理的数据一分为二,实现流水并行
pipe.InitBuffer(queOut, 2, 1024);
for-loop {
    // CopyIn 阶段
    {
        auto tensor = queIn.AllocTensor<half>();    // 从Que上申请资源, 长度1024
        AscendC::DataCopy(tensor, gm, 1024);       // 搬运数据从GM到VECIN
        queIn.EnQue(tensor);
    }
    // Compute阶段
    {
        auto tensor = queIn.DeQue<half>();
        auto tensorOut = queOut.AllocTensor<half>();
        AscendC::Abs(tensorOut, tensor, 1024);      // 计算
        queIn.FreeTensor(tensor);
        queOut.EnQue(tensorOut);
    }
    // CopyOut 阶段
    {
        auto tensor = queOut.DeQue<half>();
        AscendC::DataCopy(gmOut, tensor, 1024);     // 搬运数据从VECOUT到GM
        queOut.FreeTensor(tensor);                   // 释放资源
    }
}
```

任务间数据传递使用到的内存、事件等资源统一由管理模块Pipe进行管理。如下所示的内存管理示意图，TPipe通过InitBuffer接口对外提供队列内存初始化功能，开发者可以通过该接口为指定的队列分配内存。

队列内存初始化完成后，需要使用内存时，通过调用AllocTensor来为LocalTensor分配内存，当创建的LocalTensor完成相关计算无需再使用时，再调用FreeTensor来回收LocalTensor的内存。

[图: 内存管理示意图。AllocTensor从Buffer allocated for Queue中分配LocalTensor，FreeTensor回收LocalTensor。InitBuffer从Pipe: Chip Internal Storage Manager中为Queue分配Buffer]

编程过程中使用到的临时变量内存同样通过Pipe进行管理。临时变量可以使用TBuf数据结构来申请指定TPosition上的存储空间。使用TBuf申请的内存空间只能参与计算，无法执行队列的入队出队操作。具体的接口使用说明请参考TBuf。

按照上述编程范式进行编程即可实现单核上数据的并行处理。需要处理的数据被切分成n片，每个并行任务需要依次完成n个数据切片的处理。任务间的箭头表达数据间的依赖关系，比如CopyIn处理完第一个数据切片之后，Compute才能对该切片进行处理。

[图: 流水任务示意图。输入数据 -> CopyIn/Compute/CopyOut多个切片依次处理 -> 输出数据]

上图中的流水任务运行起来的示意图如下，从运行图中可以看出，对于同一片数据，CopyIn、Compute、CopyOut之间的处理具有依赖关系，需要串行处理；不同的数据切片，同一时间点，可以有多个任务在并行处理，由此达到任务并行、提升性能的目的。

[图: 流水任务运行示意图（时间轴）。CopyIn处理切片1 -> Compute处理切片1/CopyIn处理切片2 -> CopyOut处理切片1/Compute处理切片2/CopyIn处理切片3 ...]

#### 矩阵编程范式

Cube计算的典型数据流图如下所示：

[图: Cube计算数据流图。GM -> CopyIn(VECIN) -> Split(A1->A2, B1->B2, C1->C2) -> Compute(CO1) -> Aggregate(CO2) -> CopyOut -> GM。Local Memory包含BufPool1-5]

和矢量编程范式一样，同样也使用逻辑位置（TPosition）来表达数据流，Cube编程范式中主要使用的逻辑位置定义如下：

- A1：代表设备上用于矩阵计算的逻辑内存，用于存放左矩阵，物理存储对应AI Core的L1 Buffer。
- B1：代表设备上用于矩阵计算的逻辑内存，用于存放右矩阵，物理存储对应AI Core的L1 Buffer。
- C1：代表设备上用于矩阵计算的逻辑内存，用于存放Bias（偏置）数据，物理存储对应AI Core的L1 Buffer或Unified Buffer。
- A2：代表设备上用于矩阵计算的逻辑内存，用于存放小块左矩阵（如经过分割、适配L0A Buffer容量的分块），物理存储对应AI Core的L0A Buffer。
- B2：代表设备上用于矩阵计算的逻辑内存，用于存放小块右矩阵（如经过分割、适配L0B Buffer容量的分块），物理存储对应AI Core的L0B Buffer。
- C2：代表设备上用于矩阵计算的逻辑内存，用于存放小块Bias（偏置）数据（如经过分割、适配BT Buffer容量的分块），物理存储对应AI Core的BT Buffer或L0C Buffer。
- CO1：代表设备上用于矩阵计算的逻辑内存，用于存放小块矩阵计算结果（如经过分割的矩阵计算结果分块），物理存储对应AI Core的L0C Buffer。
- CO2：代表设备上用于矩阵计算的逻辑内存，用于存放矩阵计算结果（如原始矩阵的最终计算结果），物理存储对应Global Memory或AI Core的Unified Buffer。
- VECIN：代表设备上用于矢量计算的逻辑内存，用于存放矢量计算的输入数据，物理存储对应AI Core的Unified Buffer。
- VECCALC：代表设备上用于矢量计算的逻辑内存，用于存放临时变量，物理存储对应AI Core的Unified Buffer。
- VECOUT：代表设备上用于矢量计算的逻辑内存，用于存放矢量计算的输出数据，物理存储对应AI Core的Unified Buffer。

TPosition与物理内存的映射关系请参考表1。

Cube计算流程同样也可以理解为CopyIn、Compute、CopyOut这几个阶段，因为流程相对复杂，Matmul高阶API提供对此的高阶封装，简化了编程范式。

[图: Matmul高阶API数据流图。GM -> CopyIn(SetTensorA/SetTensorB/SetBias) -> Split -> Compute(Iterate) -> Aggregate -> CopyOut(GetTensorC) -> GM]

如上图所示：CopyIn阶段对应SetTensorA、SetTensorB、SetBias接口；Compute阶段对应Iterate接口；CopyOut阶段对应GetTensorC接口。具体流程可参考如下示例：

```cpp
// 创建Matmul对象 创建对象时需要传入A、B、C、Bias的参数类型信息，类型信息通过MatmulType来定义，
// 包括：内存逻辑位置、数据格式、数据类型。
typedef MatmulType<TPosition::GM, CubeFormat::ND, half> aType;
typedef MatmulType<TPosition::GM, CubeFormat::ND, half> bType;
typedef MatmulType<TPosition::GM, CubeFormat::ND, float> cType;
typedef MatmulType<TPosition::GM, CubeFormat::ND, float> biasType;
Matmul<aType, bType, cType, biasType> mm;

REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling); // 初始化
// CopyIn阶段：完成从GM到LocalMemory的搬运
mm.SetTensorA(gm_a);    // 设置左矩阵A
mm.SetTensorB(gm_b);    // 设置右矩阵B
mm.SetBias(gm_bias);    // 设置Bias
// Compute阶段：完成矩阵乘计算
while (mm.Iterate()) {
    // CopyOut阶段：完成从LocalMemory到GM的搬运
    mm.GetTensorC(gm_c);
}
// 结束矩阵乘操作
mm.End();
```

#### 融合算子编程范式

支持Vector与Cube混合计算的算子称之为融合算子。Ascend C提供融合算子的编程范式，方便开发者基于该范式表达融合算子的数据流，快速实现自己的融合算子。

融合算子数据流指融合算子的输入输出在各存储位置间的流向。以一个典型的Cube和Vector融合算子为例，逻辑位置间的数据流向如下图所示（为了简化描述，没有列出bias）：

- Cube的输出可以作为Vector的输入：CO2->VECIN
- Vector的输出可以作为Cube的输入：VECOUT->A1->A2、VECOUT->B1->B2

[图: Cube和Vector融合算子数据流图]

基于Matmul高阶API的融合算子编程范式，对上述数据流简化表达如下：

[图: 融合算子编程范式图。AIC侧包含MatMul，AIV侧包含Vec，通过VECIN/VECOUT连接，底部为Global Memory]

1. 初始化一个MatMul对象，将输入数据从Global Memory搬运到Cube核上。
2. 进行MatMul内部的计算。
3. 将MatMul的计算结果搬运到Vector核上。
4. 进行Vector矢量计算。
5. 将输出结果搬运到Global Memory上。

整个过程的示例代码如下（伪代码）：

```cpp
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::Process()
{
    // 步骤1：初始化一个MatMul对象，将输入数据从Global Memory搬运到Cube核上。
    uint32_t computeRound = 0;
    REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), matmulObj);
    matmulObj.Init(&tiling);
    matmulObj.SetTensorA(aGlobal);
    matmulObj.SetTensorB(bGlobal);
    matmulObj.SetBias(biasGlobal);

    while (matmulObj.template Iterate<true>()) { // 步骤2：进行MatMul内部的计算。
        // 步骤3：将MatMul的计算结果搬运到Vector核上。
        reluOutLocal = reluOutQueue_.AllocTensor<cType>();
        matmulObj.template GetTensorC<true>(reluOutLocal, false, true);
        // 步骤4：进行Vector矢量计算。
        AscendC::LeakyRelu(reluOutLocal, reluOutLocal, (cType)alpha, tiling.baseM * tiling.baseN);
        reluOutQueue_.EnQue(reluOutLocal);
        // 步骤5：将输出结果搬运到Global Memory上
        reluOutQueue_.DeQue<cType>();
        ...
        AscendC::DataCopy(cGlobal[startOffset], reluOutLocal, copyParam);
        reluOutQueue_.FreeTensor(reluOutLocal);

        computeRound++;
    }
    matmulObj.End();
}
```

### 4.4.2 静态Tensor编程范式

在基于Pipe进行算子开发的方式中，由Pipe（TPipe类）统一管理Device端内存等资源，开发者无需感知内存管理、DoubleBuffer流水、同步等处理，只需要按照计算流编写算子即可，但由此也带来了一些运行时开销（如TPipe创建、InitBuffer等）。

基于以上原因，Ascend C提供了静态Tensor编程方式，相比基于Pipe的编程方式，这种方式避免了TPipe内存管理初始化过程（约数百纳秒），从而减少了运行时开销，更有助于开发者实现极致性能。通过直接构造指定地址和存储位置的LocalTensor，并将其传递给计算、搬运等API进行编程，提供了更高的灵活性。然而，这种编程方式也带来了更高的开发复杂性，需要开发者自行管理DoubleBuffer和同步流水，并且只能使用Ascend C的基础API，而非全部功能。

两种编程方式的对比如下：

[图: 两种编程方式对比。左侧为基于TPipe和TQue自动管理内存与同步的代码，右侧为静态Tensor编程自主管理内存与同步的代码]

> 说明
> - 静态Tensor编程的使用约束和限制请参考使用约束和限制。
> - 本节涉及的完整样例请参考静态Tensor编程样例。

#### 编程范式

- AI Core包括多种内存单元，比如用于矢量计算的Unified Buffer和用于矩阵计算的L1 Buffer、L0A Buffer、L0B Buffer、L0C Buffer等内存资源。
- AI Core包括多条流水，比如Vector/Cube/Scalar计算流水、MTE1、MTE2、MTE3搬运流水等，每条流水并行执行。

静态Tensor编程方式下，开发者完全自主的管理AI Core上的所有内存资源，调用Ascend C提供的搬运或者计算类API编写算子，并根据数据依赖关系插入对应的同步事件，以达成最优性能。

下图是一个典型矢量算子的示意图，开发者首先根据业务计算量进行数据分块处理，之后根据核内的数据依赖关系完成同步事件的插入：

[图: 静态Tensor编程矢量算子示意图，展示多个Vector Core的数据分块处理和SetFlag/WaitFlag同步机制]

#### 内存管理

静态Tensor编程方式下，开发者可以使用两种方式创建Tensor：

- 通过LocalMemAllocator指定硬件位置进行Tensor分配。
  LocalMemAllocator是一种线性内存分配器，开发者可以调用Alloc方法进行内存分配，地址分配从0开始，根据调用次序依次向后进行线性分配。LocalMemAllocator只是一个简单的线性分配器，并不提供内存释放以及其它内存管理的能力。在不关注Bank冲突场景或者算子初始功能开发时，可以使用LocalMemAllocator简化算子编写，在后续性能优化时切换到使用LocalTensor进行地址分配的方式。

- 通过LocalTensor构造函数创建Tensor，极致性能场景推荐使用此方式。
  开发者可以使用LocalTensor构造函数直接指定内存地址，实现内存的完全自主管理（本质上无需申请和释放内存）。使用时，需根据需求合理指定地址（不超过物理存储上限），并在保证功能正确的前提下进行内存复用。如果需要通过规避Bank冲突或者复用内存来获得极致性能时，推荐使用该方式。

```cpp
// 方式1：使用LocalMemAllocator进行内存分配
AscendC::LocalMemAllocator<AscendC::Hardware::UB> ubAllocator;
AscendC::LocalTensor<float> xLocalPing = ubAllocator.Alloc<float, TILE_LENGTH>();
AscendC::LocalTensor<float> yLocalPing = ubAllocator.Alloc<float, TILE_LENGTH>();
AscendC::LocalTensor<float> zLocalPing = ubAllocator.Alloc<float, TILE_LENGTH>();

// 方式2：直接使用LocalTensor构造数据构造Tensor
AscendC::LocalTensor<float> xLocalPing(AscendC::TPosition::VECCALC, xAddrPing, TILE_LENGTH);
AscendC::LocalTensor<float> yLocalPing(AscendC::TPosition::VECCALC, yAddrPing, TILE_LENGTH);
AscendC::LocalTensor<float> zLocalPing(AscendC::TPosition::VECCALC, zAddrPing, TILE_LENGTH);
```

#### 同步管理

根据前文介绍的硬件架构，AI Core内部异步并行计算存在多条流水（包括矢量计算、矩阵计算、数据搬入、数据搬出等），多条流水之间存在数据依赖时，需要插入对应的同步事件。静态Tensor编程方式下，开发者使用SetFlag/WaitFlag(ISASI)和PipeBarrier(ISASI)手动插入同步，事件的类型和事件ID由开发者自行管理，但需要注意事件ID不能使用6和7（可能与内部使用的事件ID出现冲突，进而出现未定义行为）。另外由于需要使用SetFlag/WaitFlag/PipeBarrier底层同步接口（属于ISASI硬件体系结构相关的接口），无法保证跨硬件版本兼容。

在同步依赖中，根据数据依赖和指令执行关系，存在两种依赖关系，即正向同步（循环内依赖）与反向同步（循环间依赖）：

- 正向同步
  在本次数据搬入和计算之间，插入MTE2_V（矢量计算流水等待MT2搬运流水）同步事件，确保数据搬入之后再进行计算；在本次数据计算和搬出之间，插入V_MTE3（MTE3搬运流水等待矢量计算流水）同步事件，确保数据计算完成后再进行搬出。

- 反向同步
  在上一次的数据计算和本次数据搬入之间，插入V_MTE2（MT2搬运流水等待矢量计算流水）同步事件，确保上一次的数据计算完成后，本次的数据再进行搬入。防止本次的数据会覆盖掉上一次未计算完成的数据；在上一次的数据搬出和本次数据计算之间，插入MTE3_V（矢量计算流水等待MT3搬运流水）同步事件，确保上一次的数据搬出后，再进行本次数据的计算。防止本次的数据会覆盖掉上一次未搬出的数据。

上述的同步逻辑在使用Pipe编程框架时，框架会使用EnQue/DeQue/AllocTensor/FreeTensor进行封装。您可以通过11.3 编程模型设计原理来了解该如何在使用静态Tensor编程方式时手动进行同步控制。

```cpp
AscendC::LocalTensor<float> xLocal(AscendC::TPosition::VECCALC, xAddr, TILE_LENGTH);
AscendC::LocalTensor<float> yLocal(AscendC::TPosition::VECCALC, yAddr, TILE_LENGTH);
AscendC::LocalTensor<float> zLocal(AscendC::TPosition::VECCALC, zAddr, TILE_LENGTH);
for (int i = 0; i < loopCount; i++) {
    // dependency of PIPE_V & PIPE_MTE2 caused by xLocal/yLocal between 2 sequential loops
    if (i != 0) {
        AscendC::WaitFlag<AscendC::HardEvent::V_MTE2>(EVENT_ID0);
    }
    AscendC::DataCopy(xLocal, xGm[i * TILE_LENGTH], TILE_LENGTH);
    AscendC::DataCopy(yLocal, yGm[i * TILE_LENGTH], TILE_LENGTH);
    // dependency of PIPE_MTE2 & PIPE_V caused by xLocal/yLocal in one single loop
    AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(EVENT_ID0);
    AscendC::WaitFlag<AscendC::HardEvent::MTE2_V>(EVENT_ID0);
    if (i != 0) {
        // dependency of PIPE_MTE3 & PIPE_V caused by zLocal between 2 sequential loops
        AscendC::WaitFlag<AscendC::HardEvent::MTE3_V>(EVENT_ID0);
    }
    AscendC::Add(zLocal, xLocal, yLocal, TILE_LENGTH);
    if (i != (loopCount - 1)) {
        // dependency of PIPE_V & PIPE_MTE2 caused by xLocal/yLocal between 2 sequential loops
        AscendC::SetFlag<AscendC::HardEvent::V_MTE2>(EVENT_ID0);
    }
    // dependency of PIPE_V & PIPE_MTE3 caused by zLocal in one single loop
    AscendC::SetFlag<AscendC::HardEvent::V_MTE3>(EVENT_ID0);
    AscendC::WaitFlag<AscendC::HardEvent::V_MTE3>(EVENT_ID0);
    AscendC::DataCopy(zGm[i * TILE_LENGTH], zLocal, TILE_LENGTH);
    if (i != (loopCount - 1)) {
        // dependency of PIPE_MTE3 & PIPE_V caused by zLocal between 2 sequential loops
        AscendC::SetFlag<AscendC::HardEvent::MTE3_V>(EVENT_ID0);
    }
}
```

#### 流水优化

在基于TPipe的编程范式中，开发者只需要在InitBuffer时指定buffer数量为2，即可自动开启Double Buffer。但是静态Tensor编程方式下，开发者需要手动开启Double Buffer，具体示例如下，完整样例请参考静态Tensor编程样例中的Double Buffer示例。

```cpp
// ping
AscendC::LocalTensor<float> xLocalPing(AscendC::TPosition::VECCALC, xAddrPing, TILE_LENGTH);
AscendC::LocalTensor<float> yLocalPing(AscendC::TPosition::VECCALC, yAddrPing, TILE_LENGTH);
AscendC::LocalTensor<float> zLocalPing(AscendC::TPosition::VECCALC, zAddrPing, TILE_LENGTH);
// pong
AscendC::LocalTensor<float> xLocalPong(AscendC::TPosition::VECCALC, xAddrPong, TILE_LENGTH);
AscendC::LocalTensor<float> yLocalPong(AscendC::TPosition::VECCALC, yAddrPong, TILE_LENGTH);
AscendC::LocalTensor<float> zLocalPong(AscendC::TPosition::VECCALC, zAddrPong, TILE_LENGTH);

// double buffer
AscendC::SetFlag<AscendC::HardEvent::MTE3_MTE2>(EVENT_ID0);
AscendC::SetFlag<AscendC::HardEvent::MTE3_MTE2>(EVENT_ID1);
for (int i = 0; i < loopCount; i++) {
    int32_t eventID = (i % 2 == 0 ? EVENT_ID0 : EVENT_ID1);
    AscendC::LocalTensor<float> &xLocal = (i % 2 == 0 ? xLocalPing : xLocalPong);
    AscendC::LocalTensor<float> &yLocal = (i % 2 == 0 ? yLocalPing : yLocalPong);
    AscendC::LocalTensor<float> &zLocal = (i % 2 == 0 ? zLocalPing : zLocalPong);
    // dependency of PIPE_MTE3 & PIPE_MTE2 caused by xLocal/yLocal between 2 sequential loops
    AscendC::WaitFlag<AscendC::HardEvent::MTE3_MTE2>(eventID);
    AscendC::DataCopy(xLocal, xGm[i * TILE_LENGTH], TILE_LENGTH);
    AscendC::DataCopy(yLocal, yGm[i * TILE_LENGTH], TILE_LENGTH);

    // dependency of PIPE_MTE2 & PIPE_V caused by xLocal/yLocal in one single loop
    AscendC::SetFlag<AscendC::HardEvent::MTE2_V>(eventID);
    AscendC::WaitFlag<AscendC::HardEvent::MTE2_V>(eventID);
    AscendC::Add(zLocal, xLocal, yLocal, TILE_LENGTH);
    // dependency of PIPE_V & PIPE_MTE3 caused by zLocal in one single loop
    AscendC::SetFlag<AscendC::HardEvent::V_MTE3>(eventID);
    AscendC::WaitFlag<AscendC::HardEvent::V_MTE3>(eventID);
    AscendC::DataCopy(zGm[i * TILE_LENGTH], zLocal, TILE_LENGTH);
    // dependency of PIPE_MTE3 & PIPE_MTE2 caused by zLocal between 2 sequential loops
    AscendC::SetFlag<AscendC::HardEvent::MTE3_MTE2>(eventID);
}
AscendC::WaitFlag<AscendC::HardEvent::MTE3_MTE2>(EVENT_ID0);
AscendC::WaitFlag<AscendC::HardEvent::MTE3_MTE2>(EVENT_ID1);
```

以下为不使能DoubleBuffer和使能DoubleBuffer的流水示意图。多数情况下，采用DoubleBuffer能有效提升Vector的时间利用率，缩减算子执行时间，详细内容可参考11.5.1 DoubleBuffer。

[图: 未开启DoubleBuffer流水图：MTE2、Vector、MTE3三条流水串行处理loop 0/1/2]

[图: 使能DoubleBuffer流水图：MTE2、Vector、MTE3三条流水通过ping-pong交替实现并行处理]

#### 使用约束和限制

静态Tensor编程方式需要遵循以下约束和限制：

- 开发者不能使用TPipe/TQue/TQueBind/TBufPool等框架接口，和上述框架接口混用可能会出现未定义行为。
- 只能使用部分API。具体支持的API列表见支持的API范围。因为不在列表范围内的API内部依赖TPipe分配事件ID，可能会和开发者定义的事件ID产生冲突。
- 同步事件需要由开发者使用SetFlag/WaitFlag(ISASI)和PipeBarrier(ISASI)手动插入，事件的类型和事件ID由开发者自行管理，但需要注意事件ID不能使用6和7（可能与内部使用的事件ID出现冲突，进而出现未定义行为）。
- 由于需要使用SetFlag/WaitFlag/PipeBarrier底层同步接口（属于ISASI硬件体系结构相关的接口），无法保证算子跨硬件版本兼容。
- Kernel入口处需要开发者手动调用InitSocState接口用来初始化全局状态寄存器。因为全局状态寄存器处于不确定状态，如果不调用该接口，可能导致算子执行过程中出现未定义行为。在TPipe框架编程中，初始化过程由TPipe完成，无需开发者关注。

#### 支持的API范围

表 4-2 针对Atlas推理系列产品AI Core，支持的API范围

| 接口分类 | 接口名称 |
|---|---|
| 基础API > 标量计算 | ScalarGetCountOfValue、ScalarCountLeadingZero、ScalarCast、CountBitsCntSameAsSignBit、ScalarGetSFFValue |
| 基础API > 矢量计算 > 基础算术 | Exp、Ln、Abs、Reciprocal、Sqrt、Rsqrt、Relu、VectorPadding、Add、Sub、Mul、Div、Max、Min、BilinearInterpolation、Adds、Muls、Maxs、Mins、LeakyRelu |
| 基础API > 矢量计算 > 逻辑计算 | Not、And、Or |
| 基础API > 矢量计算 > 复合计算 | Axpy、CastDeq、AddRelu、AddReluCast、AddDeqRelu、SubRelu、SubReluCast、MulAddDst、FusedMulAdd、FusedMulAddRelu |
| 基础API > 矢量计算 > 比较与选择 | Compare、Compare（结果存入寄存器）、CompareScalar、GetCmpMask、SetCmpMask、Select、GatherMask |
| 基础API > 矢量计算 > 类型转换 | Cast |
| 基础API > 矢量计算 > 归约计算 | WholeReduceMax、WholeReduceMin、WholeReduceSum、BlockReduceMax、BlockReduceMin、BlockReduceSum、PairReduceSum、RepeatReduceSum、GetReduceMaxMinCount |
| 基础API > 矢量计算 > 数据转换 | Transpose、TransDataTo5HD |
| 基础API > 矢量计算 > 数据填充 | Duplicate |
| 基础API > 矢量计算 > 排序组合 | ProposalConcat、ProposalExtract、RpSort16、MrgSort4、GetMrgSortResult |
| 基础API > 矢量计算 > 离散与聚合 | Gather、Scatter |
| 基础API > 矢量计算 > 掩码操作 | SetMaskCount、SetMaskNorm、SetVectorMask、ResetMask |
| 基础API > 矢量计算 > 量化设置 | SetDeqScale |
| 基础API > 数据搬运 > DataCopy | 基础数据搬运 |
| 基础API > 同步控制 > 核内同步 | SetFlag/WaitFlag、PipeBarrier |
| 基础API > 缓存控制 | DataCachePreload、DataCacheCleanAndInvalid、ICachePreLoad |
| 基础API > 系统变量访问 | GetBlockNum、GetBlockIdx、GetDataBlockSizeInBytes、GetArchVersion、GetTaskRation、InitSocState、GetProgramCounter、CheckLocalMemoryIA |
| 基础API > 原子操作 | SetAtomicAdd、SetAtomicNone |
| 基础API > 矩阵计算 | InitConstValue、LoadData、SetAippFunctions、LoadImageToLocal、LoadUnzipIndex、LoadDataUnzip、SetLoadDataBoundary、SetLoadDataPaddingValue、Mmad |

表 4-3 针对Atlas A2训练系列产品/Atlas A2推理系列产品，支持的API范围

| 接口分类 | 接口名称 | 备注 |
|---|---|---|
| 基础API > 标量计算 | ScalarGetCountOfValue、ScalarCountLeadingZero、ScalarCast、CountBitsCntSameAsSignBit、ScalarGetSFFValue、ToBfloat16、ToFloat | - |
| 基础API > 矢量计算 > 基础算术 | Exp、Ln、Abs、Reciprocal、Sqrt、Rsqrt、Relu、Add、Sub、Mul、Div、Max、Min、BilinearInterpolation、Adds、Muls、Maxs、Mins、LeakyRelu | - |
| 基础API > 矢量计算 > 逻辑计算 | Not、And、Or、ShiftLeft、ShiftRight | - |
| 基础API > 矢量计算 > 复合计算 | Axpy、CastDeq、AddRelu、AddReluCast、AddDeqRelu、SubRelu、SubReluCast、MulAddDst、MulCast、FusedMulAdd、FusedMulAddRelu | - |
| 基础API > 矢量计算 > 比较与选择 | Compare、Compare（结果存入寄存器）、CompareScalar、GetCmpMask、SetCmpMask、Select、GatherMask | - |
| 基础API > 矢量计算 > 类型转换 | Cast | - |
| 基础API > 矢量计算 > 归约计算 | WholeReduceMax、WholeReduceMin、WholeReduceSum、BlockReduceMax、BlockReduceMin、BlockReduceSum、PairReduceSum、RepeatReduceSum、GetAccVal、GetReduceMaxMinCount | - |
| 基础API > 矢量计算 > 数据转换 | Transpose、TransDataTo5HD | - |
| 基础API > 矢量计算 > 数据填充 | Duplicate、Brcb | - |
| 基础API > 矢量计算 > 排序组合 | Sort32、MrgSort、GetMrgSortResult | - |
| 基础API > 矢量计算 > 离散与聚合 | Gather、Gatherb | - |
| 基础API > 矢量计算 > 掩码操作 | SetMaskCount、SetMaskNorm、SetVectorMask、ResetMask | - |
| 基础API > 矢量计算 > 量化设置 | SetDeqScale | - |
| 基础API > 数据搬运 > DataCopy | 基础数据搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| 基础API > 数据搬运 | 增强数据搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| | 切片数据搬运 | - |
| | 随路转换ND2NZ搬运、随路转换NZ2ND搬运、随路量化激活搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| 基础API > 数据搬运 | Copy、DataCopyPad、SetPadValue | - |
| 基础API > 同步控制 > 核内同步 | SetFlag/WaitFlag、PipeBarrier、DataSyncBarrier | - |
| 基础API > 同步控制 > 核间同步 | CrossCoreSetFlag、CrossCoreWaitFlag | - |
| 基础API > 缓存控制 | DataCachePreload、DataCacheCleanAndInvalid、ICachePreLoad、GetICachePreloadStatus | - |
| 基础API > 系统变量访问 | GetBlockNum、GetBlockIdx、GetDataBlockSizeInBytes、GetArchVersion、GetTaskRation、InitSocState、GetProgramCounter、GetSubBlockNum、GetSubBlockIdx、GetSystemCycle、CheckLocalMemoryIA | - |
| 基础API > 原子操作 | SetAtomicAdd、SetAtomicType、SetAtomicNone、SetAtomicMax、SetAtomicMin、SetStoreAtomicConfig、GetStoreAtomicConfig | - |
| 基础API > 矩阵计算 | Mmad、MmadWithSparse、SetHF32Mode、SetHF32TransMode、SetMMLayoutTransform、SetFixPipeConfig、SetFixpipeNz2ndFlag、SetFixpipePreQuantFlag、InitConstValue、LoadData、LoadDataWithTranspose、SetAippFunctions、LoadImageToLocal、LoadDataWithSparse、SetFmatrix、SetLoadDataBoundary、SetLoadDataRepeat、SetLoadDataPaddingValue、Fixpipe | - |
| Utils API > C++标准库 > 算法 | max、min、index_sequence | - |
| Utils API > C++标准库 > 容器函数 | tuple、get、make_tuple | - |
| Utils API > C++标准库 > 类型特性 | is_convertible、is_base_of、is_same、enable_if、conditional | - |

表 4-4 针对Atlas A3训练系列产品/Atlas A3推理系列产品，支持的API范围

| 接口分类 | 接口名称 | 备注 |
|---|---|---|
| 基础API > 标量计算 | ScalarGetCountOfValue、ScalarCountLeadingZero、ScalarCast、CountBitsCntSameAsSignBit、ScalarGetSFFValue、ToBfloat16、ToFloat | - |
| 基础API > 矢量计算 > 基础算术 | Exp、Ln、Abs、Reciprocal、Sqrt、Rsqrt、Relu、Add、Sub、Mul、Div、Max、Min、BilinearInterpolation、Adds、Muls、Maxs、Mins、LeakyRelu | - |
| 基础API > 矢量计算 > 逻辑计算 | Not、And、Or、ShiftLeft、ShiftRight | - |
| 基础API > 矢量计算 > 复合计算 | Axpy、CastDeq、AddRelu、AddReluCast、AddDeqRelu、SubRelu、SubReluCast、MulAddDst、MulCast、FusedMulAdd、FusedMulAddRelu | - |
| 基础API > 矢量计算 > 比较与选择 | Compare、Compare（结果存入寄存器）、CompareScalar、GetCmpMask、SetCmpMask、Select、GatherMask | - |
| 基础API > 矢量计算 > 类型转换 | Cast | - |
| 基础API > 矢量计算 > 归约计算 | WholeReduceMax、WholeReduceMin、WholeReduceSum、BlockReduceMax、BlockReduceMin、BlockReduceSum、PairReduceSum、RepeatReduceSum、GetAccVal、GetReduceMaxMinCount | - |
| 基础API > 矢量计算 > 数据转换 | Transpose、TransDataTo5HD | - |
| 基础API > 矢量计算 > 数据填充 | Duplicate、Brcb | - |
| 基础API > 矢量计算 > 排序组合 | Sort32、MrgSort、GetMrgSortResult | - |
| 基础API > 矢量计算 > 离散与聚合 | Gather、Gatherb | - |
| 基础API > 矢量计算 > 掩码操作 | SetMaskCount、SetMaskNorm、SetVectorMask、ResetMask | - |
| 基础API > 矢量计算 > 量化设置 | SetDeqScale | - |
| 基础API > 数据搬运 > DataCopy | 基础数据搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| 基础API > 数据搬运 | 增强数据搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| | 切片数据搬运 | - |
| | 随路转换ND2NZ搬运、随路转换NZ2ND搬运、随路量化激活搬运 | 不支持VECIN/VECCALC/VECOUT -> TSCM通路的数据搬运。 |
| | Copy、DataCopyPad、SetPadValue | - |
| 基础API > 同步控制 > 核内同步 | SetFlag/WaitFlag、PipeBarrier、DataSyncBarrier | - |
| 基础API > 同步控制 > 核间同步 | CrossCoreSetFlag、CrossCoreWaitFlag | - |
| 基础API > 缓存控制 | DataCachePreload、DataCacheCleanAndInvalid、ICachePreLoad、GetICachePreloadStatus | - |
| 基础API > 系统变量访问 | GetBlockNum、GetBlockIdx、GetDataBlockSizeInBytes、GetArchVersion、GetTaskRation、InitSocState、GetProgramCounter、GetSubBlockNum、GetSubBlockIdx、GetSystemCycle、CheckLocalMemoryIA | - |
| 基础API > 原子操作 | SetAtomicAdd、SetAtomicType、SetAtomicNone、SetAtomicMax、SetAtomicMin、SetStoreAtomicConfig、GetStoreAtomicConfig | - |
| 基础API > 矩阵计算 | Mmad、MmadWithSparse、SetHF32Mode、SetHF32TransMode、SetMMLayoutTransform、SetFixPipeConfig、SetFixpipeNz2ndFlag、SetFixpipePreQuantFlag、InitConstValue、LoadData、LoadDataWithTranspose、SetAippFunctions、LoadImageToLocal、LoadDataWithSparse、SetFmatrix、SetLoadDataBoundary、SetLoadDataRepeat、SetLoadDataPaddingValue、Fixpipe | - |
| Utils API > C++标准库 > 算法 | max、min、index_sequence | - |
| Utils API > C++标准库 > 容器函数 | tuple、get、make_tuple | - |
| Utils API > C++标准库 > 类型特性 | is_convertible、is_base_of、is_same、enable_if、conditional | - |
| 高阶API > C++标准库 > 类型特性 | is_convertible、is_base_of、is_same、enable_if、conditional | - |
| 高阶API > 模板库函数 > type_traits | is_convertible、is_base_of、is_same、enable_if、conditional | - |

### 4.4.3 AI CPU编程

AI CPU是位于Device侧ARM64架构的处理器，其具备与AI Core相同的内存访问能力，可直接访问Device侧内存资源；也可以与Host侧的CPU一样，进行类似的数据计算，通常作为AI Core的补充，主要承担非矩阵类、逻辑比较复杂的分支密集型计算。AI CPU的运行环境为基础的Linux环境，编程时可使用libc库，C++标准库，STL模板库等。其硬件架构图如下所示：

[图: AI CPU硬件架构图。Host服务器通过PCIE接口连接Device AI处理器，Device中包含AI CPU、AI Core、L2 Cache和Global Memory]

#### AI CPU核函数定义

在进行AI CPU编程时，与AI Core类似，同样需要定义设备侧数据入口（即核函数），该函数必须通过`__aicpu__`标识符进行声明，并且需与`__global__`标识符联合使用以表明其只能被Host侧调用。AI CPU的Device侧实现文件需要以.aicpu为后缀（或者在编译时增加-x aicpu选项）。该实现文件中包括上面介绍的核函数以及AI CPU普通函数定义，AI CPU普通函数无需添加执行空间标识符。

如下是一个AI CPU"Hello World"程序的示例，hello_world.aicpu文件内容如下：

```cpp
// 调用printf接口需要包含的头文件
#include "aicpu_api.h"

__global__ __aicpu__ uint32_t hello_world(void *args)
{
    AscendC::printf("Hello World!!!\n");
    return 0;
}
```

> 说明
> 编程时需要遵循如下规范：
> - `__aicpu__` `__global__`函数不能是void返回类型，并且入参只能是一个指针。
> - `__aicpu__` `__global__`函数不能是类的成员函数，也不能存在于匿名空间下。

#### AI CPU核函数调用

AI CPU核函数的调用需要在.asc文件中进行，和AI Core的算子调用类似，同样使用<<<>>>语法。

```cpp
hello_world<<<blockDim, nullptr, stream>>>(&args, sizeof(KernelArgs));
```

- blockDim：AI CPU Device侧暂不支持分核逻辑，因此Host侧调用多核无实际意义。建议设置为1。
- l2ctrl：保留参数，当前固定为nullptr，开发者无需关注。
- stream，类型为aclrtStream，stream用于维护一些异步操作的执行顺序，确保按照应用程序中的代码调用顺序在device上执行。stream创建等管理接口请参考"Stream管理"章节。

> 说明
> 在编写调用代码时需要遵循如下规范：
> - `__aicpu__` `__global__`函数不能在.asc文件中进行定义，只能声明，且需要使用extern。
> - Host侧调用`__aicpu__` `__global__`函数时必须使用<<<>>>异构调用语法，输入的函数入参在入参指针的基础上需要输入从指针中读取的数据大小。
> - 在Host侧使用内核调用符<<<...>>>调用AI Core与AI CPU算子时不能使用同一条stream。

加载和运行算子时，需要使用Runtime API，完成运行时管理和配置，详细内容请参考5.3 算子运行。AI CPU算子的编译请参考5.1 AI Core算子编译。

#### AI CPU模板核函数

若需要使用模板核函数，则需要在.aicpu文件中给出模板核函数的实例化声明，参考如下：

```cpp
template<typename T, int BUFF_SIZE>
__global__ __aicpu__ uint32_t hello_world(void *args)
{
    AscendC::printf("Hello World!!!\n");
    AscendC::printf("buffer_size is %d\n", BUFF_SIZE);
    return 0;
}
template __global__ __aicpu__ uint32_t hello_world<KernelArgs, 4096>(void *args);
```

并在.asc文件中新增模板核函数实例化的extern声明：

```cpp
template<typename T, int BUFF_SIZE>
extern __global__ __aicpu__ uint32_t hello_world(void *args);

template extern __global__ __aicpu__ uint32_t hello_world<KernelArgs, 4096>(void *args);
```

#### 更多进阶用法

> 说明：更多AI CPU API的使用方法请参考AI CPU API。

# 5 编译与运行

## 5.1 AI Core算子编译

### 5.1.1 算子编译简介

本章节介绍的算子编译方法支持开发者通过bisheng命令行和CMake进行手动配置编译选项，或编写CMake脚本来实现编译。开发者可以将Host侧main.cpp和Device侧Kernel核函数置于同一实现文件中，以实现异构编译。

- 目前，该编译方法仅支持如下型号：
  - Atlas A3 训练系列产品/Atlas A3 推理系列产品
  - Atlas A2 训练系列产品/Atlas A2 推理系列产品
  - Atlas 推理系列产品
- 当前版本暂不支持CPU孪生调试功能，NPU仿真调试功能。
- 暂不支持在同一个编译单元的两个核函数中，同时设定KERNEL_TYPE_MIX_AIC_1_1和KERNEL_TYPE_MIX_AIC_1_2两种Kernel类型。
- 针对Atlas推理系列产品，暂不支持设置Kernel类型为KERNEL_TYPE_MIX_VECTOR_CORE。

### 5.1.2 通过bisheng命令行编译

毕昇编译器是一款专为昇腾AI处理器设计的编译器，支持异构编程扩展，可以将用户编写的昇腾算子代码编译成二进制可执行文件和动态库等形式。毕昇编译器的可执行程序命名为bisheng，支持x86、aarch64等主机系统，并且原生支持设备侧AI Core架构指令集编译。通过使用毕昇编译器，用户可以更加高效地进行针对昇腾AI处理器的编程和开发工作。

#### 入门示例

以下是一个使用毕昇编译器编译静态Shape的add_custom算子入门示例。该示例展示了如何编写源文件add_custom.cpp以及具体的编译命令。通过这个示例，您可以了解如何使用毕昇编译器进行算子编译。

步骤1：包含头文件。

```cpp
// 头文件
#include "data_utils.h"
#include "acl/acl.h"
#include "kernel_operator.h"
```

步骤2：核函数实现。

- 核函数支持模板。
- 核函数入参支持传入用户自定义的结构体，比如示例中用户自定义的AddCustomTilingData结构体。

```cpp
// 用户自定义的TilingData结构体
struct AddCustomTilingData {
    uint32_t totalLength;
    uint32_t tileNum;
};

// Kernel核心实现逻辑，包括搬运、计算等
template <typename T>
class KernelAdd {
public:
    __aicore__ inline KernelAdd() {}
    // ...
};

// 核函数
// 核函数支持模板，核函数入参支持传入用户自定义的结构体
template <typename T>
__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIV_ONLY); // 该算子执行时仅启动AI Core上的Vector核
    KernelAdd<T> op;
    op.Init(x, y, z, tiling.totalLength, tiling.tileNum);
    op.Process();
}
```

步骤3：Host侧调用逻辑，包括内存申请和释放，初始化和去初始化，内核调用符调用核函数等。

（完整代码示例包含aclInit、aclrtSetDevice、内存分配、数据拷贝、<<<>>>核函数调用、结果回拷、资源释放、aclFinalize等流程）

步骤4：采用如下的编译命令进行编译。

```bash
bisheng -x asc add_custom.cpp -o add_custom --npu-arch=dav-2201
```

- `-x asc`：文件名为cpp时，指定输入文件的语言为Ascend C编程语言。
- `-o add_custom`：指定输出文件名为add_custom。
- `--npu-arch=dav-2201`：指定NPU的架构版本为dav-2201。dav-后为NPU架构版本号，各产品型号对应的架构版本号请通过表6-1进行查询。

步骤5：执行可执行文件。

```bash
./add_custom
```

----结束

#### 程序的编译与执行

通过毕昇编译器可以将算子源文件编译为当前平台的可执行文件或算子动态库、静态库。

- 编译生成可执行文件

  ```bash
  # 编译hello_world.cpp为当前平台可执行文件
  # bisheng [算子源文件] -o [输出产物名称] --npu-arch=[NPU架构版本号]，常见参数顺序与g++保持一致。
  bisheng -x asc hello_world.cpp -o hello --npu-arch=dav-xxxx
  ```

  生成的可执行文件可通过如下方式执行：`./hello`

- 编译生成算子动态库

  ```bash
  # 编译add_custom_base.cpp生成算子动态库
  # bisheng -shared [算子源文件] -o [输出产物名称] --npu-arch=[NPU架构版本号]
  bisheng -shared -x asc add_custom_base.cpp -o libadd.so --npu-arch=dav-xxxx
  ```

- 编译生成算子静态库

  ```bash
  # 编译add_custom_base.cpp生成算子静态库
  # bisheng -lib [算子源文件] -o [输出产物名称] --npu-arch=[NPU架构版本号]
  bisheng -lib -x asc add_custom_base.cpp -o libadd.a --npu-arch=dav-xxxx
  ```

在命令行编译场景下，可以按需链接需要的库文件，常见的库文件请参考常用的链接库。编译时会默认链接表5-3中列出的库文件。注意如下例外场景：在使用g++链接asc代码编译生成的静态库时，需要手动链接默认链接库。

### 5.1.3 常用的编译选项

常用的编译选项说明如下，全量的编译选项请参考毕昇编译器编译选项。

| 选项 | 是否必需 | 说明 |
|---|---|---|
| -help | 否 | 查看帮助。 |
| --npu-arch | 是 | 编译时指定的昇腾AI处理器架构，取值为dav-<arch-version>，其中<arch-version>为NPU架构版本号，各产品型号对应的架构版本号请通过表6-1进行查询。 |
| --npu-soc | 否 | 编译时指定的昇腾AI处理器型号，npu-soc和npu-arch同时配置时，优先使能npu-arch。昇腾AI处理器的型号请通过npu-smi info命令查询。 |
| -x | 否 | 指定编译语言。指定为asc时表示Ascend C编程语言。 |
| -o <file> | 否 | 指定输出文件的名称和位置。 |
| -c | 否 | 编译生成目标文件。 |
| -shared, --shared | 否 | 编译生成动态链接库。 |
| -lib, --cce-build-static-lib | 否 | 编译生成静态链接库。编译器会将Device侧的代码进行编译链接，生成Device侧二进制文件，随后将该文件作为Host侧编译的输入进行编译，最后链接生成静态链接库。 |
| -g | 否 | 编译时增加调试信息。 |
| --sanitizer | 否 | 编译时增加代码正确性校验信息。使用sanitizer选项时，需要同步添加-g选项。 |
| -fPIC | 否 | 告知编译器产生位置无关代码。 |
| -O | 否 | 用于指定编译器的优化级别，当前支持-O3，-O2，-O0。 |

### 5.1.4 通过CMake编译

项目中可以使用CMake来更简便地使用毕昇编译器编译Ascend C算子，生成可执行文件、动态库、静态库或二进制文件。

> 说明：使用前需要通过如下命令添加Ascend C CMake Module搜索路径至环境变量。${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：/usr/local/Ascend/ascend-toolkit/latest。
> ```bash
> export CMAKE_PREFIX_PATH=${INSTALL_DIR}/compiler/tikcpp/
> ascendc_kernel_cmake:$CMAKE_PREFIX_PATH
> ```

以下是CMake脚本的示例及其核心步骤说明：

```cmake
# 1、find_package(ASC)是CMake中用于查找和配置Ascend C编译工具链的命令
find_package(ASC)

# 2、指定项目支持的语言包括ASC和CXX，ASC表示支持使用毕昇编译器对Ascend C编程语言进行编译
project(kernel_samples LANGUAGES ASC CXX)

# 3、使用CMake接口编译可执行文件、动态库、静态库、二进制文件
add_executable(demo
    add_custom.asc
)
#.....
target_compile_options(demo PRIVATE
    # --npu-arch用于指定NPU的架构版本，dav-后为架构版本号，各产品型号对应的架构版本号请通过表6-1进行查询。
    # <COMPILE_LANGUAGE:ASC>:表明该编译选项仅对语言ASC生效
    $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=dav-2201>
)
```

以下是动态库、静态库编译示例，同时展示如何将源文件切换为用语言ASC编译：

- 编译.cpp文件生成动态库

  ```cmake
  # 将.cpp文件置为ASC属性，启用Ascend C语言进行编译
  set_source_files_properties(
      add_custom_base.cpp
      sub_custom_base.cpp
      PROPERTIES LANGUAGE ASC
  )

  add_library(kernel_lib SHARED
      add_custom_base.cpp
      sub_custom_base.cpp
  )
  ```

- 编译.asc文件生成静态库

  ```cmake
  # .asc文件会默认启用Ascend C语言进行编译，不需要通过set_source_files_properties进行设置
  add_library(kernel_lib STATIC
      add_custom_base.asc
      sub_custom_base.asc
  )
  ```

下文列出了使用CMake编译时常用的变量配置说明、常用的链接库、以及默认链接库。

表 5-1 常用的变量配置说明

| 变量 | 配置说明 |
|---|---|
| CMAKE_BUILD_TYPE | 编译模式选项，可配置为"Release"（Release版本，不包含调试信息，编译最终发布的版本）或"Debug"（Debug版本，包含调试信息，便于开发者开发和调试）。 |
| CMAKE_INSTALL_PREFIX | 用于指定CMake执行install时，安装的路径前缀，执行install后编译产物会安装在该路径下。默认路径为当前目录的out目录下。 |
| CMAKE_CXX_COMPILER_LAUNCHER | 用于配置C++语言编译器（如g++）、毕昇编译器的启动器程序为ccache，配置后即可开启cache缓存编译，加速重复编译并提高构建效率。使用该功能前需要安装ccache。 |

表 5-2 常用的链接库（在使用高阶API时，必须链接以下库，因为这些库是高阶API功能所依赖的）

| 名称 | 作用描述 | 使用场景 |
|---|---|---|
| libtiling_api.a | Tiling函数相关库。 | 使用高阶API相关的Tiling接口时需要链接。 |
| libregister.so | Tiling注册相关库。 | 使用高阶API相关的Tiling接口时需要链接。 |
| libgraph_base.so | 基础数据结构和接口库。 | 调用ge::Shape，ge::DataType等基础结构体时需要链接。 |
| libplatform.so | 硬件平台信息库。 | 使用PlatformAscendC相关硬件平台信息接口时需要链接。 |

表 5-3 默认链接库

| 名称 | 作用描述 |
|---|---|
| libascendc_runtime.a | Ascend C算子参数等组装库。 |
| libruntime.so | Runtime运行库。 |
| libprofapi.so | Ascend C算子运行性能数据采集库。 |
| libascentalog.so | CANN日志收集库。 |
| libmmpa.so | CANN系统接口库。 |
| libascend_dump.so | CANN维测信息库。 |
| libc_sec.so | CANN安全函数库。 |
| liberror_manager.so | CANN错误信息管理库。 |
| libascendcl.so | acl相关接口库。 |

### 5.1.5 RTC

RTC是Ascend C运行时编译库，通过aclrtc接口，在程序运行时，将中间代码动态编译成目标机器码，提升程序运行性能。

运行时编译库提供以下核心接口：

- aclrtcCreateProg：根据输入参数（字符串形式表达的Ascend C源代码等）创建aclrtcProg程序实例。
- aclrtcCompileProg：编译给定的程序，支持用户自定义编译选项，比如指定NPU架构版本号：--npu-arch=dav-2201。支持的编译选项可以参考毕昇编译器编译选项。
- aclrtcGetBinDataSize：获取编译后的Device侧二进制数据的大小。
- aclrtcGetBinData：获取编译后的Device侧二进制数据。
- aclrtcDestroyProg：在编译和执行过程结束后，销毁给定的程序。

编译完成后需要调用如下接口完成（仅列出核心接口）Kernel加载与执行。完整流程和详细接口说明请参考"Kernel加载与执行"章节。

1. 通过aclrtBinaryLoadFromData接口解析由aclrtcGetBinData接口获取的算子二进制数据。
2. 获取核函数句柄并根据核函数句柄操作其参数列表，相关接口包括aclrtBinaryGetFunction（获取核函数句柄）、aclrtKernelArgsInit（初始化参数列表）、aclrtKernelArgsAppend（添加拷贝用户设置的参数值如xDevice, zDevice）等。
3. 调用aclrtLaunchKernelWithConfig接口，启动对应算子的计算任务。

编译命令如下，编译时需要设置-I路径为${INSTALL_DIR}/include，用于找到aclrtc相关头文件，并需要链接alc_rtc动态库。

```bash
g++ add_custom.cpp -I${INSTALL_DIR}/include -L${INSTALL_DIR}/lib64 -lascendcl -lacl_rtc -o main
```

${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：/usr/local/Ascend/ascend-toolkit/latest。

## 5.2 AI CPU算子编译

### 通过bisheng命令行编译

下文基于一个Hello World打印样例来讲解如何通过bisheng命令行编译AI CPU算子。

hello_world.aicpu文件内容如下：

```cpp
#include "aicpu_api.h"

__global__ __aicpu__ uint32_t hello_world(void *args)
{
    AscendC::printf("Hello World!!!\n");
    return 0;
}
```

Host侧使用内核调用符<<<...>>>进行AI CPU算子的调用，main.asc示例代码如下：

```cpp
#include "acl/acl.h"

struct KernelArgs {
    int mode;
};

extern __global__ __aicpu__ uint32_t hello_world(void *args);

int32_t main(int argc, char const *argv[])
{
    aclInit(nullptr);
    int32_t deviceId = 0;
    aclrtSetDevice(deviceId);
    aclrtStream stream = nullptr;
    aclrtCreateStream(&stream);

    struct KernelArgs args = {0};
    constexpr uint32_t blockDim = 1;
    hello_world<<<blockDim, nullptr, stream>>>(&args, sizeof(KernelArgs));
    aclrtSynchronizeStream(stream);

    aclrtDestroyStream(stream);
    aclrtResetDevice(deviceId);
    aclFinalize();
    return 0;
}
```

开发者可以使用bisheng命令行将hello_world.aicpu与main.asc分别编译成.o，再链接成为可执行文件，编译命令如下：

```bash
$bisheng -O2 hello_world.aicpu --cce-aicpu-L${INSTALL_DIR}/toolkit/lib64/device/lib64 --cce-aicpu-laicpu_api -I${INSTALL_DIR}/include/ascendc/aicpu_api -c -o hello_world.aicpu.o
$bisheng --npu-arch=dav-2201 main.asc -c -o main.asc.o
$bisheng hello_world.aicpu.o main.asc.o -o demo
```

上文我们通过一个入门示例介绍了使用bisheng命令行编译生成可执行文件的示例。除此之外，使用bisheng命令行也支持编译生成AI CPU算子的动态库与静态库，用户可在asc代码中通过内核调用符<<<>>>调用AI CPU算子的核函数，并在编译asc代码源文件生成可执行文件的时候，链接AI CPU算子动态库或者静态库，注意：若单独编译AI CPU算子代码生成动态库、静态库时，需要手动链接表5-3。

- 编译生成算子动态库

  ```bash
  # bisheng -shared -x aicpu test_aicpu.cpp -o libtest_aicpu.so -lxxx ...
  ```

- 编译生成算子静态库

  ```bash
  # bisheng -lib -x aicpu test_aicpu.cpp -o libtest_aicpu.a -lxxx ...
  ```

### AI CPU算子常用编译选项

| 选项 | 是否必需 | 说明 |
|---|---|---|
| -help | 否 | 查看帮助。 |
| -x | 否 | 指定编译语言。指定为aicpu时表示AI CPU算子编程语言。 |
| -o <file> | 否 | 指定输出文件的名称和位置。 |
| -c | 否 | 编译生成目标文件。 |
| -shared, --shared | 否 | 编译生成动态链接库。 |
| -lib | 否 | 编译生成静态链接库。 |
| -g | 否 | 编译时增加调试信息。 |
| --sanitizer | 否 | 编译时增加代码正确性校验信息。使用sanitizer选项时，需要同步添加-g选项。 |
| -fPIC | 否 | 告知编译器产生位置无关代码。 |
| -O | 否 | 用于指定编译器的优化级别，当前支持-O3，-O2，-O0。 |
| --cce-aicpu-L | 否 | 指定AI CPU Device依赖的库路径。 |
| --cce-aicpu-l | 否 | 指定AI CPU Device依赖的库。 |

### 通过CMake编译

项目中可以使用CMake来更简便地使用毕昇编译器编译AI CPU算子，生成可执行文件、动态库、静态库或二进制文件。

> 说明：使用前需要通过如下命令添加Ascend C CMake Module搜索路径至环境变量。

仍以通过bisheng命令行编译中介绍的Hello World打印样例为例，除了代码实现文件，还需要在工程目录下准备一个CMakeLists.txt。

```cmake
cmake_minimum_required(VERSION 3.16)
# 1、find_package()是CMake中用于查找和配置Ascend C编译工具链的命令
find_package(ASC REQUIRED)
find_package(AICPU REQUIRED)

# 2、指定项目支持的语言包括ASC、AICPU和CXX，ASC表示支持使用毕昇编译器对Ascend C编程语言进行编译，AI CPU表示支持使用毕昇编译器对AI CPU算子进行编译
project(kernel_samples LANGUAGES ASC AICPU CXX)

# 3、使用CMake接口编译可执行文件
add_executable(demo
    hello_world.aicpu
    main.asc
)

#4、由于存在ASC与AI CPU语言，需要指定链接器
set_target_properties(demo PROPERTIES LINKER_LANGUAGE ASC) //指定链接使用语言

target_compile_options(demo PRIVATE
    $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=dav-2201>
)
```

## 5.3 算子运行

算子的计算算法实现通过Ascend C API来完成，而算子的加载调用则使用Runtime API来完成。本章节将结合核函数调用介绍CANN软件栈中Ascend C算子运行时常用的Runtime接口。Runtime接口更多信息与细节可以参考"acl API（C&C++）"章节。

### 加载和运行代码

加载和运行算子时，需要使用Runtime API，完成运行时管理和配置。主要流程和使用到的API如下：

1. 初始化：aclInit。
2. 运行时资源申请：通过aclrtSetDevice和aclrtCreateStream分别申请Device、Stream运行管理资源。
3. 分配Host内存，并进行数据初始化。
4. 分配Device内存，并将数据从Host上拷贝到Device上，参与核函数计算。
5. 使用<<<>>>调用算子核函数。
6. 执行核函数后，将Device上的运算结果拷贝回Host。
7. 异步等待核函数执行完成：aclrtSynchronizeStream。
8. 资源释放：通过aclrtDestroyStream和aclrtResetDevice分别释放Stream、Device运行管理资源。
9. 去初始化：aclFinalize。

[图: 加载和运行代码流程图，包含：初始化 -> 运行管理资源申请 -> 分配Host内存并进行数据初始化 -> 分配Device内存并将数据从Host上拷贝到Device上 -> 用内核调用符<<<>>>调用核函数完成指定的运算并同步等待 -> 将Device上的运算结果拷贝回Host -> 释放申请的资源 -> 去初始化]

### Kernel加载与执行的更多方式

Kernel的加载与执行也可以通过二进制加载方式实现，这是最底层的接口实现方式。内核调用符<<<...>>>为对底层接口的封装实现。使用时需要bisheng命令行编译将算子源文件编译为二进制制.o文件，再通过aclrtLaunchKernelWithConfig等Kernel加载与执行接口完成算子调用。

- Kernel加载与执行接口的具体说明请参考"Kernel加载与执行"章节。
- bisheng命令行编译选项的使用介绍请参考常用的编译选项。
  完整样例请参考Kernel加载与执行（加载二进制）样例。

> 说明：核函数的调用是异步的，核函数的调用结束后，控制权立刻返回给主机端，可以调用以下aclrtSynchronizeStream函数来强制主机端程序等待所有核函数执行完毕。
> `aclError aclrtSynchronizeStream(aclrtStream stream);`

# 6 语言扩展层

## 6.1 C++语言拓展

### 预处理符号拓展

- `__NPU_ARCH__`

  `__NPU_ARCH__`是Device侧AI Core代码中的预处理宏，用于标识AI处理器的架构版本。该宏由四位数字组成，其中前三位数字用于标识AI Core的IP核(Intellectual Property Core)类型，第四位数字标识该AI Core同一个IP核的配置版本。通过该宏，开发者可以针对不同AI处理器，差异化进行代码适配和优化。产品型号和`__NPU_ARCH__`对应关系如下表所示：

  表 6-1 产品型号和__NPU_ARCH__的对应关系

  | 产品型号 | __NPU_ARCH__ |
  |---|---|
  | Atlas A3 训练系列产品/Atlas A3 推理系列产品 | 2201 |
  | Atlas A2 训练系列产品/Atlas A2 推理系列产品 | 2201 |
  | Atlas 200I/500 A2 推理产品 | 3002 |
  | Atlas 推理系列产品 | 2002 |
  | Atlas 训练系列产品 | 1001 |

  以下为通过`__NPU_ARCH__`控制在不同AI处理器上算子输出值舍入模式的示例。

  ```cpp
  __aicore__ static inline void CopyOut(uint64_t mulLen)
  {
  #if __NPU_ARCH__ == 2002
      Cast(dstLocal, srcLocal, RoundMode::CAST_NONE, mulLen); // CAST_NONE表示舍入模式在转换有精度损失时不进行舍入
  #elif __NPU_ARCH__ == 2201
      Cast(dstLocal, srcLocal, RoundMode::CAST_RINT, mulLen); // CAST_RINT表示舍入模式为四舍六入五成双舍入
  #endif
      event_t eventVToMTE3 = static_cast<event_t>(GetTPipePtr()->FetchEventID(HardEvent::V_MTE3));
      SetFlag<HardEvent::V_MTE3>(eventVToMTE3);
      WaitFlag<HardEvent::V_MTE3>(eventVToMTE3);
      CommonCopyOut<float>(dstLocal, mulLen); // 拷贝LocalTensor至GlobalTensor
  }
  ```

- `ASCEND_IS_AIV`、`ASCEND_IS_AIC`

  ASCEND_IS_AIV和ASCEND_IS_AIC是通过C++实现的条件判断语句，用于在`__aicore__`修饰的函数中实现代码的条件编译。基于分离模式（AIC核和AIV核分离）开发融合算子时，算子逻辑中同时涉及AIV核和AIC核的处理逻辑，并需要进行核间同步，此时需要通过ASCEND_IS_AIV/ ASCEND_IS_AIC进行AIV和AIC核代码的隔离。

  > 说明：当使用高阶API Matmul时，其内部已通过REGIST_MATMUL_OBJ宏方式实现了AIV与AIC核代码的隔离处理。

  以MatmulNzCustom算子为例，该算子在分离模式下需要分别在AIV核和AIC核上实现不同的逻辑。具体而言，AIV核负责将矩阵数据搬入Unified Buffer，完成数据的重排（将矩阵数据转换为NZ格式），并将其写入Global Memory。而AIC核则直接从Global Memory读取已经重排好的NZ格式数据，并执行矩阵乘法（Matmul）计算。由于AIV核和AIC核的代码逻辑不同，需要通过ASCEND_IS_AIV和ASCEND_IS_AIC宏进行代码隔离，确保在编译时分别生成适用于AIV核和AIC核的代码。

  ```cpp
  template <typename AType, typename BType, typename CType, typename BiasType>
  __aicore__ inline void MatmulKernel<AType, BType, CType, BiasType>::Process(AscendC::TPipe *pipe)
  {
      // 利用AIV核的Vector计算单元实现ND2NZ格式转换
      if ASCEND_IS_AIV {
          pipe->InitBuffer(ubBuf, TOTAL_UB_SIZE);
          MatrixBtoNZ<typename B_TYPE::T>(tempGM,
              bGMNZ,
              tiling,
              isTransB,
              ubBuf,
              tiling.baseK,
              tiling.baseN); // Vector侧实现的ND2NZ函数
          SyncAll();
          // AIC核和AIV核同步
          AscendC::CrossCoreSetFlag<0x2, PIPE_MTE3>(0x4);
          return;
      }
      if ASCEND_IS_AIC {
          AscendC::CrossCoreWaitFlag(0x4); // 等待AIV核完成ND2NZ格式转换
      }
      // ... ...
      // 设置左矩阵A、右矩阵B、Bias。
      matmulObj.SetTail(tailM, tailN);
      matmulObj.SetTensorA(aGlobal, false);
      matmulObj.SetTensorB(bGlobal, false);
      if (tiling.isBias) {
          matmulObj.SetBias(biasGlobal);
      }
      // 完成矩阵乘操作
      matmulObj.IterateAll(cGlobal);
      // 结束矩阵乘操作
      matmulObj.End();
  }
  ```


### 6.3 Kernel Task类型

通过KERNEL_TASK_TYPE_DEFAULT宏设置核函数的任务类型，控制核函数在哪种核心上执行。

支持的Kernel Task类型：

| 类型 | 说明 |
|------|------|
| KERNEL_TYPE_AIV_ONLY | 核函数仅在AIV核上执行 |
| KERNEL_TYPE_AIC_ONLY | 核函数仅在AIC核上执行 |
| KERNEL_TYPE_MIX_AIC_1_1 | 核函数同时在AIC和AIV核上执行，配比1:1 |
| KERNEL_TYPE_MIX_AIC_1_2 | 核函数同时在AIC和AIV核上执行，配比1:2 |
| KERNEL_TYPE_MIX_VECTOR_CORE | 核函数在Vector Core上执行 |

### 6.4 函数修饰符

| 修饰符 | 执行位置 | 调用者 | 功能 |
|------|--------|--------|------|
| `__global__` | 设备端 | 主机端 | 标识核函数入口，必须返回void，该函数同时也需要使用`__aicore__`修饰 |
| `__aicore__` | 设备端 | 设备端 | 标识该函数在Device侧执行 |
| `__inline__` | 设备端 | 设备端 | 标识Device侧函数强制内联，可以减少函数频繁调用产生的指令压栈、出栈的开销，但可能会导致算子二进制增加。和C++函数修饰符inline的主要区别是Device侧`__inline__`是强制内联，C++的inline则是根据编译器优化选择性内联。AI Core对函数嵌套深度有限制，一般推荐嵌套深度不超过4层。使用强制内联可以减少调用层次 |
| `__cube__` | 设备端 | 设备端 | 标识该核函数仅Cube核执行。针对耦合模式的硬件架构，该修饰符不生效 |
| `__vector__` | 设备端 | 设备端 | 标识该核函数仅在Vector核执行。针对耦合模式的硬件架构，该修饰符不生效 |
| `__mix__(cube, vec)` | 设备端 | 设备端 | 标识该核函数同时在Cube核和Vector核上执行。(cube, vec)分别表示核函数启动的Cube核和Vector核的配比，支持的配比为(1, 0)，(0, 1)，(1, 1)，(1, 2)。针对耦合模式的硬件架构，该修饰符不生效 |

示例代码：

```cpp
class KernelAdd {
public:
  __aicore__ __inline__ KernelAdd() {}
  __aicore__ __inline__ void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, uint32_t totalLength, uint32_t tileNum)
  // __aicore__表示该函数在Device侧执行, __inline__强制该函数内联
  {
    this->blockLength = totalLength / AscendC::GetBlockNum();
    this->tileNum = tileNum;
    this->tileLength = this->blockLength / tileNum / BUFFER_NUM;

    xGm.SetGlobalBuffer((__gm__ DTYPE_X *)x + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    yGm.SetGlobalBuffer((__gm__ DTYPE_Y *)y + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    zGm.SetGlobalBuffer((__gm__ DTYPE_Z *)z + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);

    pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(DTYPE_X));
    pipe.InitBuffer(inQueueY, BUFFER_NUM, this->tileLength * sizeof(DTYPE_Y));
    pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(DTYPE_Z));
  }
  // ... ...
};

extern "C" __vector__ __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z,
    GM_ADDR workspace, GM_ADDR tiling) // 使用__global__表示该函数为核函数入口，__aicore__表示该函数在
    // Device侧执行, __vector__表示该核函数仅在Vector核上执行
{
    GET_TILING_DATA(tiling_data, tiling);
    KernelAdd op;
    op.Init(x, y, z, tiling_data.totalLength, tiling_data.tileNum);
    op.Process();
}
```

### 6.5 地址空间修饰符

| 修饰符 | 功能 |
|------|------|
| `__gm__` | 标识Global Memory，是GlobalTensor实际存储的物理位置 |

### 6.6 内置常量

| 常量名 | 取值 | 功能 |
|------|------|------|
| constexpr int32_t g_coreType | AscendC::AIC / AscendC::AIV | 常量值由框架自动设置，AIC核下配置为AscendC::AIC，AIV核下配置为AscendC::AIV。可以通过对该常量值的判断来实现AIV与AIC核代码的区分和隔离。功能等同于直接使用ASCEND_IS_AIV、ASCEND_IS_AIC |

### 6.7 内置变量

| 变量名 | 对应API | 功能 |
|------|--------|------|
| block_num | GetBlockNum | 当前任务配置的核数，用于代码内部的多核逻辑控制等 |
| block_idx | GetBlockIdx | 当前核的索引，用于代码内部的多核逻辑控制及多核偏移量计算等 |

通常建议用户使用内置变量对应的API获取所需值，不建议用户直接使用内置变量。因为内置变量反映的是单个硬件资源的配置信息，对于软件栈整合硬件资源、扩展硬件的功能，内置变量的值与实际语义可能不符。

例如，在Atlas推理系列产品中，当启用KERNEL_TYPE_MIX_VECTOR_CORE时，算子会同时运行在AI Core和Vector Core上。此时block_idx在这两种核心上都是从0开始计数，用户无法直接通过block_idx来切分数据和控制多核逻辑。而GetBlockIdx在Vector Core上对block_idx增加偏移量（AI Core的block_num），从而保证返回的值能够正确反映多核环境下的实际逻辑。

# 7 C++类库API

## 7.1 编程接口概述

本章提供编程API的概述。具体API参考见Ascend C API。

Ascend C提供一组类库API，开发者使用标准C++语法和类库API进行编程。Ascend C编程类库API示意图如下所示，分为：

- 基础数据结构：kernel API中使用到的基础数据结构，比如GlobalTensor和LocalTensor。
- 基础API：实现对硬件能力的抽象，开放芯片的能力，保证完备性和兼容性。标注为ISASI（Instruction Set Architecture Special Interface，硬件体系结构相关的接口）类别的API，不能保证跨硬件版本兼容。
- 高阶API：实现一些常用的计算算法，用于提高编程开发效率，通常会调用多种基础API实现。高阶API包括数学库、Matmul、Softmax等API。高阶API可以保证兼容性。
- Utils API（公共辅助函数）：丰富的通用工具类，涵盖标准库、平台信息获取、运行时编译及日志输出等功能，支持开发者高效实现算子开发与性能优化。

[图: Ascend C编程类库API层次结构图，包含算子模板库(CATLASS/ATVOSS)、高阶API(数学计算/矩阵计算/激活函数/池化计算/索引计算/通信编程)、基础API(矢量计算/矩阵计算/数据搬运/资源管理/同步控制)、SIMT API(开发中)、语言扩展层等]

Ascend C API所在头文件目录为：

- 基础API：`${INSTALL_DIR}/include/ascendc/basic_api/interface`
- 高阶API：（注意，如下目录头文件中包含的接口如果未在资料中声明，属于间接调用接口，开发者无需关注）
  - `${INSTALL_DIR}/include/ascendc/highlevel_api/lib`
  - `${INSTALL_DIR}/include/tiling`

`${INSTALL_DIR}`请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：`/usr/local/Ascend/ascend-toolkit/latest`。

## 7.2 基础API

### 7.2.1 概述

基础API实现对硬件能力的抽象，开放芯片的能力，保证完备性和兼容性。标注为ISASI（Instruction Set Architecture Special Interface，硬件体系结构相关的接口）类别的API，不能保证跨硬件版本兼容。

根据功能的不同，主要分为以下几类：

- 标量计算API，实现调用Scalar计算单元执行计算的功能。
- 矢量计算API，实现调用Vector计算单元执行计算的功能。
- 矩阵计算API，实现调用Cube计算单元执行计算的功能。
- 数据搬运API，计算API基于Local Memory数据进行计算。所以数据需要先从Global Memory搬运至Local Memory，再使用计算API完成计算，最后从Local Memory搬出至Global Memory。执行搬运过程的接口称之为数据搬运API，比如DataCopy接口。
- 资源管理API，用于分配管理内存，比如AllocTensor、FreeTensor接口；
- 同步控制API，完成任务间的通信和同步，比如EnQue、DeQue接口。不同的API指令间可能存在依赖关系，不同的指令异步并行执行，为了保证不同指令队列间的指令按照正确的逻辑关系执行，需要向不同的组件发送同步指令。同步控制API内部即完成这个发送同步指令的过程，开发者无需关注内部实现逻辑，使用简单的API接口即可完成。

根据对数据操作方法的不同，分为以下几类：

- 连续计算API：支持Tensor前n个数据计算。针对源操作数的连续n个数据进行计算并连续写入目的操作数，解决一维tensor的连续计算问题。`Add(dst, src1, src2, n);`
- 高维切分API：支持Repeat和Stride。功能灵活的计算API，提供与BuiltIn API完全对等编程能力，充分发挥硬件优势，支持对每个操作数的DataBlock Stride，Repeat Stride，Mask等参数的操作。

[图: 计算API几种计算方式的特点，展示连续计算API和高维切分API的示意图]

### 7.2.2 接口分类说明

#### 7.2.2.1 连续计算API

连续计算API，支持Tensor前n个数据计算。针对源操作数的连续n个数据进行计算并连续写入目的操作数，解决一维tensor的连续计算问题。`Add(dst, src1, src2, n);`

[图: 连续计算API示意图，src1 + src2 = dst，对tensor前n个数据计算]

#### 7.2.2.2 高维切分API

本章节对矢量计算基础API中的tensor高维切分计算接口做解释说明。如果您不需要使用此类接口，可略过该章节。

下文中的repeatTime、dataBlockStride、repeatStride、mask为通用描述，其命名不一定与具体指令中的参数命名完全对应。

使用tensor高维切分计算API可充分发挥硬件优势，支持开发者控制指令的迭代执行和操作数的地址间隔，功能更加灵活。

矢量计算通过Vector计算单元完成，矢量计算的源操作数和目的操作数均通过Unified Buffer（UB）来进行存储。Vector计算单元每个迭代会从UB中取出8个datablock（每个datablock数据块内部地址连续，长度32 Byte），进行计算，并写入对应的8个datablock中。

[图: 单次迭代内的8个datablock进行Exp计算的示意图]

- 矢量计算API支持开发者通过repeatTime来配置迭代次数，从而控制指令的多次迭代执行。假设repeatTime设置为2，矢量计算单元会进行2个迭代的计算，可计算2 * 8（每个迭代8个datablock）* 32Byte（每个datablock 32Byte）= 512Byte的结果。如果数据类型为half，则计算了256个元素。由于硬件限制，repeatTime不能超过255。

[图: 2次迭代Exp计算示意图]

- 针对同一个迭代中的数据，可以通过mask参数进行掩码操作来控制实际参与计算的个数。下图为进行Abs计算时通过mask逐比特模式按位控制哪些元素参与计算的示意图，1表示参与计算，0表示不参与计算。

[图: 通过mask参数进行掩码操作示意图（以float数据类型为例）]

- 矢量计算单元还支持带间隔的向量计算，通过dataBlockStride（单次迭代内不同datablock间地址步长）和repeatStride（相邻迭代间相同datablock的地址步长）来进行配置。

dataBlockStride：

如果需要控制单次迭代内，数据处理的步长，可以通过设置同一迭代内不同datablock的地址步长dataBlockStride来实现。

[图: 单次迭代内非连续场景的示意图]

repeatStride：

当repeatTime大于1，需要多次迭代完成矢量计算时，您可以根据不同的使用场景合理设置相邻迭代间相同datablock的地址步长repeatStride的值。

[图: 多次迭代间非连续场景的示意图]

[图: dataBlockStride不同取值举例]

repeatStride是指相邻迭代间相同datablock的地址步长。

- 连续计算场景：假设定义一个Tensor供目的操作数和源操作数同时使用（即地址重叠），repeatStride取值为8。此时，矢量计算单元第一次迭代读取连续8个datablock，第二轮迭代读取下一个连续的8个datablock，通过多次迭代即可完成所有输入数据的计算。

[图: repeatStride=8连续计算场景示意图]

- 非连续计算场景：repeatStride取值大于8（如取10）时，则相邻迭代间矢量计算单元读取的数据在地址上不连续，出现2个datablock的间隔。

[图: repeatStride=10非连续计算场景示意图]

- 反复计算场景：repeatStride取值为0时，矢量计算单元会对首个连续的8个datablock进行反复读取和计算。

[图: repeatStride=0反复计算场景示意图]

- 部分重复计算：repeatStride取值大于0且小于8时，相邻迭代间部分数据会被矢量计算单元重复读取和计算，此种情形一般场景不涉及。

mask参数

mask用于控制每次迭代内参与计算的元素。可通过连续模式和逐bit模式两种方式进行设置。

- 连续模式：表示前面连续的多少个元素参与计算。数据类型为uint64_t。取值范围和源操作数的数据类型有关，数据类型不同，每次迭代内能够处理的元素个数最大值不同（当前数据类型单次迭代时能处理的元素个数最大值为：256 / sizeof(数据类型)）。当操作数的数据类型占bit位16位时（如half/uint16_t），mask属于[1, 128]；当操作数为32位时（如float/int32_t），mask属于[1, 64]。

具体样例如下：

```cpp
// int16_t数据类型单次迭代能处理的元素个数最大值为256/sizeof(int16_t) = 128, mask = 64, mask∈[1, 128], 所以是合法输入
// repeatTime = 1, 共128个元素, 单次迭代能处理128个元素, 故repeatTime = 1
// dstBlkStride, src0BlkStride, src1BlkStride = 1, 单次迭代内连续读取和写入数据
// dstRepStride, src0RepStride, src1RepStride = 8, 迭代间的数据连续读取和写入
uint64_t mask = 64;
AscendC::Add(dstLocal, src0Local, src1Local, mask, 1, { 1, 1, 1, 8, 8, 8 });
```

- 逐bit模式：可以按位控制哪些元素参与计算，bit位的值为1表示参与计算，0表示不参与。mask为数组形式，数组长度和数组元素的取值范围和操作数的数据类型有关。当操作数为16位时，数组长度为2，mask[0]、mask[1]属于[0, 2^64-1]并且不同时为0；当操作数为32位时，数组长度为1，mask[0]属于(0, 2^64-1]；当操作数为64位时，数组长度为1，mask[0]属于(0, 2^32-1]。

### 7.2.3 常用操作速查指导

#### 7.2.3.1 如何使用掩码操作API

Mask用于控制矢量计算中参与计算的元素个数，支持以下工作模式及配置方式：

表 7-1 Mask工作模式

| 工作模式 | 说明 |
|--------|------|
| Normal模式 | 默认模式，支持单次迭代内的Mask能力，需要开发者配置迭代次数，额外进行尾块的计算。Normal模式下，Mask用来控制单次迭代内参与计算的元素个数。通过调用SetMaskNorm设置Normal模式。 |
| Counter模式 | 简化模式，直接传入计算数据量，自动推断迭代次数，不需要开发者去感知迭代次数、处理非对齐尾块的操作；但是不具备单次迭代内的Mask能力。Counter模式下，Mask表示整个矢量计算参与计算的元素个数。通过调用SetMaskCount设置Counter模式。 |

表 7-2 Mask配置方式

| 配置方式 | 说明 |
|--------|------|
| 接口传参（默认） | 通过矢量计算API的入参直接传递Mask值。矢量计算API的模板参数isSetMask（仅部分API支持）用于控制接口传参还是外部API配置，默认值为true，表示接口传参。Mask对应于高维切分计算API中的mask/mask[]参数或者tensor前n个数据计算API中的calCount参数。 |
| 外部API配置 | 调用SetVectorMask接口设置Mask值，矢量计算API的模板参数isSetMask设置为false，接口入参中的Mask参数（对应于高维切分计算API中的mask/mask[]参数或者tensor前n个数据计算API中的calCount参数）不生效。适用于Mask参数相同、多次重复使用的场景，无需在矢量计算API内部反复设置，会有一定的性能优势。 |

表 7-3 Mask操作的使用方式

| 配置方式 | 工作模式 | 前n个数据计算API | 高维切分计算API |
|--------|--------|--------------|------------|
| 接口传参 | Normal模式 | 不涉及。 | isSetMask模板参数设置为true，通过接口入参传入Mask，根据使用场景配置dataBlockStride、repeatStride、repeatTime参数。 |
| 接口传参 | Counter模式 | isSetMask模板参数设置为true，通过接口入参传入Mask。 | isSetMask模板参数设置为true，通过接口入参传入Mask。根据使用场景配置dataBlockStride、repeatStride参数。repeatTime传入固定值即可，建议统一设置为1，该值不生效。 |
| 外部API配置 | Normal模式 | 不涉及。 | 调用SetVectorMask设置Mask，之后调用高维切分计算API。isSetMask模板参数设置为false；接口入参中的mask值设置为占位符MASK_PLACEHOLDER，用于占位，无实际含义。根据使用场景配置repeatTime、dataBlockStride、repeatStride参数。 |
| 外部API配置 | Counter模式 | 调用SetVectorMask设置Mask，之后调用前n个数据计算API，isSetMask模板参数设置为false；接口入参中的calCount建议设置成1。 | 调用SetVectorMask设置Mask，之后调用高维切分计算API。isSetMask模板参数设置为false；接口入参中的mask值设置为MASK_PLACEHOLDER，用于占位，无实际含义。根据使用场景配置dataBlockStride、repeatStride参数。repeatTime传入固定值即可，建议统一设置为1，该值不生效。 |

典型场景的使用示例如下：

场景1：Normal模式 + 外部API配置 + 高维切分计算API

```cpp
AscendC::LocalTensor<half> dstLocal;
AscendC::LocalTensor<half> src0Local;
AscendC::LocalTensor<half> src1Local;

// 1、设置Normal模式
AscendC::SetMaskNorm();
// 2、设置Mask
AscendC::SetVectorMask<half, AscendC::MaskMode::NORMAL>(0xffffffffffffffff, 0xffffffffffffffff); // 逐bit模式
// SetVectorMask<half, MaskMode::NORMAL>(128); // 连续模式

// 3、多次调用矢量计算API, isSetMask模板参数设置为false, 接口入参中的mask值设置为占位符
// MASK_PLACEHOLDER, 用于占位, 无实际含义
// 根据使用场景配置repeatTime、dataBlockStride、repeatStride参数
AscendC::Add<half, false>(dstLocal, src0Local, src1Local, AscendC::MASK_PLACEHOLDER, 1, { 2, 2, 2, 8, 8, 8 });
AscendC::Sub<half, false>(src0Local, dstLocal, src1Local, AscendC::MASK_PLACEHOLDER, 1, { 2, 2, 2, 8, 8, 8 });
AscendC::Mul<half, false>(src1Local, dstLocal, src0Local, AscendC::MASK_PLACEHOLDER, 1, { 2, 2, 2, 8, 8, 8 });
// 4、恢复Mask值为默认值
AscendC::ResetMask();
```

场景2：Counter模式 + 外部API配置 + 高维切分计算API

```cpp
AscendC::LocalTensor<half> dstLocal;
AscendC::LocalTensor<half> src0Local;
AscendC::LocalTensor<half> src1Local;
int32_t len = 128; // 参与计算的元素个数
// 1、设置Counter模式
AscendC::SetMaskCount();
// 2、设置Mask
AscendC::SetVectorMask<half, AscendC::MaskMode::COUNTER>(len);
// 3、多次调用矢量计算API, isSetMask模板参数设置为false; 接口入参中的mask值设置为
// MASK_PLACEHOLDER, 用于占位, 无实际含义
// 根据使用场景正确配置dataBlockStride、repeatStride参数。repeatTime传入固定值即可, 建议统一设置
// 为1, 该值不生效
AscendC::Add<half, false>(dstLocal, src0Local, src1Local, AscendC::MASK_PLACEHOLDER, 1, { 1, 1, 1, 8, 8, 8 });
AscendC::Sub<half, false>(src0Local, dstLocal, src1Local, AscendC::MASK_PLACEHOLDER, 1, { 1, 1, 1, 8, 8, 8 });
AscendC::Mul<half, false>(src1Local, dstLocal, src0Local, AscendC::MASK_PLACEHOLDER, 1, { 1, 1, 1, 8, 8, 8 });
// 4、恢复工作模式
AscendC::SetMaskNorm();
// 5、恢复Mask值为默认值
AscendC::ResetMask();
```

场景3：Counter模式 + 外部API配置 + 前n个数据计算接口配合使用

```cpp
AscendC::LocalTensor<half> dstLocal;
AscendC::LocalTensor<half> src0Local;
half num = 2;
// 1、设置Mask
AscendC::SetVectorMask<half, AscendC::MaskMode::COUNTER>(128); // 参与计算的元素个数为128
// 2、调用前n个数据计算API, isSetMask模板参数设置为false; 接口入参中的calCount建议设置成1
AscendC::Adds<half, false>(dstLocal, src0Local, num, 1);
AscendC::Muls<half, false>(dstLocal, src0Local, num, 1);
// 3、恢复工作模式
AscendC::SetMaskNorm();
// 4、恢复Mask值为默认值
AscendC::ResetMask();
```

说明：

- 前n个数据计算API接口内部会设置工作模式为Counter模式，所以如果前n个数据计算API配合Counter模式使用时，无需手动调用SetMaskCount设置Counter模式。
- 所有手动使用Counter模式的场景，使用完毕后，需要调用SetMaskNorm恢复工作模式。
- 调用SetVectorMask设置Mask，使用完毕后，需要调用ResetMask恢复Mask值为默认值。
- 使用高维切分计算API配套Counter模式使用时，比前n个数据计算API增加了可间隔的计算，支持dataBlockStride、repeatStride参数。

#### 7.2.3.2 如何使用归约计算API

归约指令将数据集合简化为单一值或者更小的集合。按照归约操作的数据范围的不同，归约指令分为以下几种，可参考归约指令示意图：

- ReduceMax/ReduceMin/ReduceSum：对所有的输入数据做归约操作，得到最大值和最大值索引/最小值和最小值索引/数据总和。
- WholeReduceMax/WholeReduceMin/WholeReduceSum：对每个repeat内的输入数据做归约操作，得到每个repeat内的最大值和最大值索引/最小值和最小值索引/数据总和。返回索引时返回的是repeat内部索引。
- BlockReduceMax/BlockReduceMin/BlockReduceSum：对每个datablock内的输入数据做归约操作，得到每个datablock内的最大值/最小值/数据总和。
- PairReduce：相邻两个（奇偶）元素求和，例如（a1, a2, a3, a4, a5, a6...），归约后结果为（a1+a2, a3+a4, a5+a6, ......）。

[图: 归约指令示意图，展示ReduceMax/ReduceMin/ReduceSum、WholeReduceMax/WholeReduceMin/WholeReduceSum、BlockReduceMax/BlockReduceMin/BlockReduceSum、PairReduceSum的操作方式]

针对归约指令，和其他的基础API一样也提供了tensor高维切分计算接口，可充分发挥硬件优势，支持开发者控制指令的迭代执行和操作数的地址间隔，功能更加灵活。但具体参数的单位和约束与基础API略有不同，下文将对这些差异点进行介绍。

- repeatTime：迭代次数，开发者通过repeatTime来配置迭代次数，从而控制指令的多次迭代执行。
  - ReduceMax/ReduceMin/ReduceSum对于repeatTime超过255的情况，在API内部进行了处理，所以repeatTime支持更大的取值范围，保证不超过int32_t最大值的范围即可。
  - WholeReduceMax/WholeReduceMin/WholeReduceSum/BlockReduceMax/BlockReduceMin/BlockReduceSum/PairReduce和其他基础API一样，repeatTime要求不超过255。
- mask：用于控制每次迭代内参与计算的元素，mask参数的使用方法和基础API通用的使用方法一致。
- repeatStride：表示相邻迭代间的地址步长。
  - ReduceMax/ReduceMin/ReduceSum指令的目的操作数会归约成一个最大值/最小值/总和，因此其目的操作数不支持配置repeatStride。仅源操作数支持repeatStride，其含义、单位（datablock）和基础API通用说明一致。
  - WholeReduceMax/WholeReduceMin/WholeReduceSum/BlockReduceMax/BlockReduceMin/BlockReduceSum/PairReduce源操作数和目的操作数都支持配置repeatStride，源操作数repeatStride的含义、单位（datablock）和基础API通用说明一致。目的操作数repeatStride的含义、单位和基础API通用说明有差异，因为归约后每个repeat会合并为一个值，所以迭代之间的间隔不能在使用一个datablock为单位，而以一个repeat归约后的长度为单位。
- dataBlockStride：表示单次迭代内datablock的地址步长。
  - ReduceMax/ReduceMin/ReduceSum指令的目的操作数会归约成一个最大值/最小值/总和，所以其目的操作数不支持配置dataBlockStride。源操作数也不支持dataBlockStride。
  - WholeReduceMax/WholeReduceMin/WholeReduceSum/BlockReduceMax/BlockReduceMin/BlockReduceSum/PairReduce源操作数支持配置dataBlockStride，源操作数dataBlockStride的含义、单位（datablock）和基础API通用说明一致。目的操作数不支持dataBlockStride，因为归约后，目的操作数的长度会变短，比如WholeReduceSum归约后每个repeat会合并为一个值，不再有迭代内datablock和地址间隔的概念。

## 7.3 高阶API

### 7.3.1 概述

高阶API基于单核对常见算法进行抽象和封装，实现了一些常用的计算算法，旨在提高编程开发效率。高阶API一般通过调用多种基础API实现。高阶API包括数学计算、矩阵计算、激活函数等API。

[图: 高阶API与基础API的关系示意图，高阶API(Matmul: SetTensorA/SetTensorB/IterateAll/...)封装常用算法逻辑减少重复开发，基础API(CopyIn/Split/Compute/Aggregate/CopyOut等)]

### 7.3.2 常用操作速查指导

#### 7.3.2.1 如何使用Tiling依赖的头文件

由于AI处理器的Scalar计算单元执行能力有限，为减少算子Kernel侧的Scalar计算，部分计算在Host端执行，这需要编写Host端Tiling代码。注意，在程序中调用高阶API的Tiling接口或者使用高阶API的Tiling结构体参数时，需要引入依赖的头文件。在不同的Tiling实现方式下，具体为：

- 使用标准C++语法定义Tiling结构体

  这种方式需要引入依赖的头文件如下。所有高阶API的Tiling结构体定义在AscendC::tiling命名空间下，因此需要通过AscendC::tiling访问具体API的Tiling结构体。

  ```cpp
  #include "kernel_tiling/kernel_tiling.h"
  // ...
  AscendC::tiling::TCubeTiling cubeTilingData;
  ```

- 使用TILING_DATA_DEF宏定义Tiling结构体

  这种方式需要引入依赖的头文件如下。所有高阶API的Tiling结构体和Tiling函数定义在optiling命名空间下。

  ```cpp
  #include "tiling/tiling_api.h"
  namespace optiling {
  // ...
  }
  ```

#### 7.3.2.2 如何使用Kernel侧临时空间

Kernel侧接口的内部实现一般涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。除矩阵计算、HCCL通信类、卷积计算等，对于多数高阶API中临时空间的处理，开发者可以通过Kernel侧接口的入参sharedTmpBuffer传入提前申请的临时空间、通过接口框架申请临时空间两种方式。

- 通过sharedTmpBuffer入参传入，Kernel侧接口使用该传入的Tensor作为临时空间。该方式下，开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需在Kernel侧申请临时空间，但是需要预留临时空间的大小，即在分配内存空间时，应在可用空间大小中减去需预留的临时空间大小。

无论开发者采用上述哪种方式，在申请Tensor空间或预留临时空间时，都需要提前获取Kernel侧接口所需的临时空间大小/BufferSize，为此相应类别API中提供了GetxxxMaxMinTmpSize接口，用于获取所需预留空间的大小范围，其中xxx为对应的Kernel侧接口。开发者在Host侧调用GetxxxMaxMinTmpSize接口，获取预留/申请的临时空间的最大和最小临时空间大小，基于此范围选择合适的空间大小作为Tiling参数传递到Kernel侧使用。

- 为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；
- 在最小临时空间-最大临时空间范围内，随着临时空间增大，Kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。

以Asin接口为例：

```cpp
// 算子输入的数据类型T为half, isReuseSource传入默认值false
auto shape_input = context->GetInputTensor(0)->GetOriginShape();
std::vector<int64_t> srcDims = {shape_input.GetDim(0), shape_input.GetDim(1)};
uint32_t srcSize = 1;
for (auto dim : srcDims) {
  srcSize *= dim;
}
uint32_t typeSize = 2;
ge::Shape shape(srcDims);
uint32_t minValue = 0;
uint32_t maxValue = 0;
AscendC::GetAsinMaxMinTmpSize(shape, typeSize, false, maxValue, minValue);

auto platformInfo = context->GetPlatformInfo();
auto ascendcPlatform = platform_ascendc::PlatformAscendC(platformInfo);
uint64_t tailSize = 0; // UB剩余空间大小
ascendcPlatform.GetCoreMemSize(platform_ascendc::CoreMemType::UB, tailSize);
// 本样例中使用完整的ub空间, 实际情况下tailSize需要减掉用户已使用的UB空间
auto tmpSize = tailSize >= maxValue ? maxValue : tailSize;

AsinCustomTilingData tiling;
tiling.set_tmpBufferSize(tmpSize); // 将临时空间大小设置为Tiling参数
```

另外，多数高阶API中提供了GetxxxTmpBufferFactorSize接口，该接口用于获取maxLiveNodeCnt和extraBuf，maxLiveNodeCnt表示临时空间是单次计算数据量所占空间的多少倍；extraBuf表示Kernel侧接口所需的临时空间大小。在固定空间大小的情况下，通过maxLiveNodeCnt和extraBuf可以推算算子单次最大计算元素数量。

推算示例如下：

- 算子实现需要调用Mean接口，开发者为其预留currBuff大小的空间（即总可用空间），利用GetMeanTmpBufferFactorSize接口得到maxLiveNodeCnt、extraBuf输出值，可推导算子单次最大计算元素数量为：
  `currentShapeSize = (currBuff - extraBuf) / maxLiveNodeCnt / typeSize`

- 算子实现需要调用两个Kernel侧API KernelIntf1、KernelIntf2，利用两个GetXxxTmpBufferFactorSize（其中Xxx为需要调用的两个高阶API）接口的两组输出值(maxLiveNodeCnt, extraBuf)以及当前现有的临时空间currBuff，推导单次最大计算元素数量currentShapeSize为：
  `currentShapeSize1 = (currBuff - extraBuf1) / maxLiveNodeCnt1 / typeSize`
  `currentShapeSize2 = (currBuff - extraBuf2) / maxLiveNodeCnt2 / typeSize`
  `currentShapeSize = min(currentShapeSize1, currentShapeSize2)`

注意上文中的currBuff表示接口计算可用的空间，需要去除用户输入输出等空间。

以算子中需要同时调用Asin、Acos接口为例：

```cpp
// 算子输入的数据类型T为half
auto shape_input = context->GetInputTensor(0)->GetOriginShape();
std::vector<int64_t> srcDims = { shape_input.GetDim(0), shape_input.GetDim(1) };
uint32_t srcSize = 1;
uint32_t srcCurSize = 1;
for (auto dim : srcDims) {
  srcSize *= dim;
}
uint32_t typeSize = 2;

auto platformInfo = context->GetPlatformInfo();
auto ascendcPlatform = platform_ascendc::PlatformAscendC(platformInfo);
uint64_t tailSize = 0; // UB剩余空间大小
ascendcPlatform.GetCoreMemSize(platform_ascendc::CoreMemType::UB, tailSize);

uint32_t asinMaxLiveNodeCount = 0;
uint32_t asinExtraBuf = 0;

uint32_t acosMaxLiveNodeCount = 0;
uint32_t acosExtraBuf = 0;

AscendC::GetAsinTmpBufferFactorSize(typeSize, asinMaxLiveNodeCount, asinExtraBuf);
AscendC::GetAcosTmpBufferFactorSize(typeSize, acosMaxLiveNodeCount, acosExtraBuf);
// tmp的大小需要减去UB上调用api接口输入和输出占用的大小
// 该示例中包括Asin接口的输入输出, 以及Acos的输入输出, 其中Asin接口的输出作为Acos的输入, 因此一共需
// 要3份src的空间大小
auto tmpSize = tailSize - srcSize * typeSize * 3;
assert(tmpSize >= asinExtraBuf);
assert(tmpSize >= acosExtraBuf);
// 计算Asin算子单次最大计算元素数量
if (asinMaxLiveNodeCount != 0) {
  srcAsinCurSize = (tmpSize - asinExtraBuf) / asinMaxLiveNodeCount / typeSize;
} else {
  srcAsinCurSize = srcSize;
}
// 计算Acos算子单次最大计算元素数量
if (acosMaxLiveNodeCount != 0) {
  srcAcosCurSize = (tmpSize - acosExtraBuf) / acosMaxLiveNodeCount / typeSize;
} else {
  srcAcosCurSize = srcSize;
}
srcCurSize = std::min(srcAsinCurSize, srcAcosCurSize);

AsinCustomTilingData tiling;
tiling.set_srcCurSize(srcCurSize); // 将单次最大计算元素数量设置为Tiling参数
```

## 7.4 Utils API

Ascend C开发提供了丰富的通用工具类，涵盖标准库、平台信息获取、上下文构建、运行时编译及日志输出等功能，支持开发者高效实现算子开发与性能优化。

- C++标准库API：提供算法、数学函数、容器函数等C++标准库函数。
- 平台信息获取API：提供获取平台信息的功能，比如获取硬件平台的核数等信息。
- RTC API：Ascend C运行时编译库，通过aclrtc接口，在程序运行时，将中间代码动态编译成目标机器码，提升程序运行性能。
- log API：提供Host侧打印Log的功能。开发者可以在算子的TilingFunc代码中使用ASC_CPU_LOG_XXX接口来输出相关内容。

# 8 硬件实现

## 8.1 基本架构

基于Ascend C开发的算子运行在AI Core上。Ascend C编程模型基于AI Core硬件架构的抽象进行介绍，了解硬件架构能够帮助开发者更好的理解编程模型；对于需要完成高性能编程的深度开发者，更需要了解硬件架构相关知识，最佳实践中很多内容都以本章为基础进行介绍。

[图: AI Core基本架构图，包含Host服务器通过PCIE接口连接Device AI处理器，Device包含AI Core、AI CPU、L2 Cache、Global Memory]

AI Core负责执行矩阵、矢量计算密集的任务，其包括以下组成部分：

- 计算单元：包括Cube（矩阵）计算单元、Vector（矢量）计算单元和Scalar（标量）计算单元。
- 存储单元：包括L1 Buffer、L0A Buffer、L0B Buffer、L0C Buffer、Unified Buffer、BiasTable Buffer、Fixpipe Buffer等专为高效计算设计的存储单元。
- 搬运单元：包括MTE1、MTE2、MTE3和FixPipe，用于数据在不同存储单元之间的高效传输。

以Atlas A2训练系列产品/Atlas A2推理系列产品为例，硬件架构图如下：

[图: Atlas A2硬件架构详细图，展示AIC核(含MTE2/MTE1/L1 Buffer/Cube/L0A/L0B/L0C/BT/FP Buffer/Scalar/DCache/ICache)和AIV核(含MTE2/MTE3/Unified Buffer/Vector/Scalar/DCache/ICache)，以及Global Memory和L2 Cache]

关键概念和术语

- Core：拥有独立Scalar计算单元的计算核，Scalar计算单元承担了核内的指令发射等功能，也称之为核内的调度单元。
- AI Core：昇腾AI处理器的计算核，负责执行矩阵、矢量计算密集的任务。
- Cube Core：矩阵计算核，专注于矩阵计算。由Scalar调度单元、矩阵计算单元、搬运单元等组成，不包括矢量计算单元。
- Vector Core：矢量计算核，专注于矢量计算，由Scalar调度单元、矢量计算单元、搬运单元等组成，不包括矩阵计算单元。
- AIC：在AI Core分离模式下，一组Cube Core和Vector Core组合中的Cube Core。
- AIV：在AI Core分离模式下，一组Cube Core和Vector Core组合中的Vector Core。

AI Core的工作模式

- 分离模式：AI Core的一种工作模式，矩阵计算单元、矢量计算单元各自对应独立的Scalar调度单元，分离部署在Cube Core和Vector Core上。将Cube Core和Vector Core按照一定比例（1:N）进行组合，这样的组合视为一个AI Core，AI Core的核数以Cube Core为准。

  [图: 分离模式示意图，AI Core包含Cube Core(Cube+Scalar)和Vector Core(Vector+Scalar)，比例1:N]

- 耦合模式：AI Core的一种工作模式，矩阵计算单元、矢量计算单元对应同一个Scalar调度单元，部署在一个AI Core上。

  [图: 耦合模式示意图，AI Core包含Scalar、Vector、Cube在同一个核上]

说明：Ascend C编程中，不同产品的工作模式如下：

- Atlas推理系列产品：耦合模式
- Atlas训练系列产品：耦合模式
- Atlas A2训练系列产品/Atlas A2推理系列产品：分离模式
- Atlas A3训练系列产品/Atlas A3推理系列产品：分离模式
- Atlas 200I/500 A2推理产品：耦合模式

注意：针对Atlas 200I/500 A2推理产品，硬件的工作模式既可以支持耦合模式，又可以支持分离模式。耦合模式下，开发者仅需关注AI Core数量，无需关注Vector Core和Cube Core数量；分离模式下，需要关注AI Core、Vector Core、Cube Core的数量。Ascend C编程场景下，仅支持耦合模式。

计算单元

计算单元是AI Core中提供强大算力的核心单元，包括三种基础计算单元：Cube（矩阵）计算单元、Vector（矢量）计算单元和Scalar（标量）计算单元，完成AI Core中不同类型的数据计算。

- Cube：Cube计算单元负责执行矩阵运算，以float16数据类型为例，Cube每次执行可完成两个float16类型的16x16矩阵的乘法操作。

  [图: Cube计算单元数据访问图，高亮部分为Cube计算单元及其访问的存储单元L0A/L0B/L0C]

- Vector：Vector负责执行向量运算。向量计算单元执行向量指令，类似于传统的单指令多数据（Single Instruction Multiple Data，SIMD）指令，每个向量指令可以完成多个操作数的同一类型运算。向量计算单元可以快速完成两个float16类型的向量相加或者相乘。向量指令支持多次迭代执行，也支持对带有间隔的向量直接进行运算。Vector所有计算的源数据以及目标数据都要求存储在Unified Buffer中，Vector指令的首地址和操作长度有对齐要求，通常要求32B对齐，具体对齐要求参考API的约束描述。

  [图: Vector计算单元数据访问图，高亮部分为Unified Buffer和Vector计算单元]

- Scalar：Scalar负责各类型的标量数据运算和程序的流程控制。功能上可以看做一个小CPU，完成整个程序的循环控制、分支判断、Cube/Vector等指令的地址和参数计算以及基本的算术运算，并且可以通过在事件同步模块中插入同步符的方式来控制AI Core中其他执行单元的流水。相对于Host CPU，AI Core中的Scalar计算能力较弱，重点用于发射指令，所以在实际应用场景中应尽量减少Scalar计算，比如性能调优时尽量减少if/else等分支判断及变量运算。

  Scalar执行标量运算指令时，执行标准的ALU(Arithmetic Logic Unit)语句，ALU需要的代码段和数据段（栈空间）都来自于GM，ICache(Instruction Cache)用于缓存代码段，缓存大小和硬件规格相关，比如为16K或32K，以2K为单位加载；DCache(Data Cache)用于缓存数据段，大小也与硬件规格相关，比如为16K，以Cache Line（64Byte）为单位加载。考虑到核内访问效率最高，应尽量保证代码段和数据段被缓存在ICache和DCache，避免核外访问；同时根据数据加载单位不同，编程时可以考虑单次加载数据大小，来提升加载效率。例如在DCache加载数据时，当数据内存首地址与Cache Line（64Byte）对齐时，加载效率最高。

  [图: Scalar对指令和数据的访问图，高亮部分为DCache、ICache和Scalar]

说明：硬件提供L2Cache用于缓存访问GM的数据（包括代码段、数据段），以此加快访问速度，提高访问效率。核外L2Cache以Cache Line为单位加载数据，根据硬件规格不同，Cache Line大小不同（128/256/512Byte等）。

存储单元和搬运单元

AI处理器中的计算资源要想发挥强劲算力，必要条件是保证输入数据能够及时准确地出现在计算单元中，需要精心设计存储系统，保证计算单元所需的数据供应。

AI Core中包含多级内部存储，AI Core需要把外部存储中的数据加载到内部存储中，才能完成相应的计算。AI Core的主要内部存储包括：L1 Buffer（L1缓冲区），L0 Buffer（L0缓冲区），Unified Buffer（统一缓冲区）等。为了配合AI Core中的数据传输和搬运，AI Core中还包含MTE（Memory Transfer Engine，数据传递引擎）搬运单元，在搬运过程中可执行随路数据格式/类型转换。

表 8-1 存储单元介绍

| 存储单元 | 描述 |
|--------|------|
| L1 Buffer | L1缓冲区，通用内部存储，是AI Core内比较大的一块数据中转区，可暂存Cube计算单元需要反复使用的一些数据从而减少从总线读写的次数 |
| L0A Buffer / L0B Buffer | Cube指令的输入 |
| L0C Buffer | Cube指令的输出，但进行累加计算的时候，也是输入的一部分 |
| Unified Buffer | 统一缓冲区，向量和标量计算的输入和输出 |
| BT Buffer | BiasTable Buffer，存放矩阵计算中的Bias |
| FP Buffer | Fixpipe Buffer，存放量化参数、Relu参数等 |

表 8-2 搬运单元介绍

| 搬运单元 | 描述 |
|--------|------|
| MTE1 | 负责如下通路的数据搬运：L1->L0A/L0B、L1->BT Buffer |
| MTE2 | 负责如下通路的数据搬运：GM->{L1, L0A/B}（在该通路下，基于分形大小搬运，搬运时满足Cache Line大小对齐，性能更优）、GM->UB（基于Cache Line大小搬运性能更优） |
| MTE3 | 负责如下通路的数据搬运：UB->GM |
| FixPipe | 负责如下通路的数据搬运，搬运过程中可以完成随路数据格式/类型转换：L0C->{GM/L1}、L1->FP Buffer |

说明：

- 不同类型的AI处理器，存储单元大小不同，开发者可通过GetCoreMemSize接口获取。
- 所有通过搬运单元读写GM的数据都缺省被缓存在L2Cache，以此加快访问速度，提高访问效率。核外L2Cache以Cache Line为单位加载数据，根据硬件规格不同，Cache Line大小不同（128/256/512Byte等）。

典型的数据流

- Vector计算典型的数据流如下：GM -> UB -> Vector -> UB -> GM

  [图: Vector计算数据流示意图，展示Global Memory经L2 Cache、MTE2搬入Unified Buffer，Vector计算后由MTE3搬出]

- Cube计算典型的数据流如下：
  - GM -> L1 -> L0A/L0B -> Cube -> L0C -> FixPipe -> GM
  - GM -> L1 -> L0A/L0B -> Cube -> L0C -> FixPipe -> L1

  [图: Cube计算数据流示意图]

典型的指令流

多条指令从系统内存通过总线接口进入到ICache(Instruction Cache)中，后续的指令执行过程，根据指令的类型，有两种可能：

- 如果指令是Scalar指令，指令会被Scalar单元直接执行。
- 其他指令会被Scalar单元调度到独立的分类序列（Vector指令序列、Cube指令序列、MTE1/MTE2/MTE3指令序列等），然后再被对应的执行单元执行。

[图: 指令分类处理机制示意图，Scalar单元将指令分发到Cube指令序列、Vector指令序列、FixPipe指令序列、MTE1/MTE2/MTE3指令序列]

同一个指令序列中的指令是按照进入指令序列的顺序执行的，不同指令序列之间可以并行执行，通过多个指令序列的并行执行可以提升整体执行效率。对于并行执行过程中可能出现的数据依赖，通过事件同步模块插入同步指令来控制流水线的同步，提供PipeBarrier、SetFlag/WaitFlag两种API，保证序列内部以及序列之间按照逻辑关系执行。

- PipeBarrier本身是一条指令，用于在序列内部约束执行顺序（虽然指令是顺序执行，但并不意味着后一条指令开始执行时前一条指令会执行结束）。PipeBarrier指令可以保证前序指令中所有数据读写全部完成，后序指令才开始执行。
- SetFlag/WaitFlag为两条指令，在SetFlag/WaitFlag的指令中，可以指定一对指令序列的关系，表示两个序列之间完成一组"锁"机制，其作用方式为：
  - SetFlag：当前序指令的所有读写操作都完成之后，当前指令开始执行，并将硬件中的对应标志位设置为1。
  - WaitFlag：当执行到该指令时，如果发现对应标志位为0，该序列的后续指令将一直被阻塞；如果发现对应标志位为1，则将对应标志位设置为0，同时后续指令开始执行。

Ascend C提供同步控制API，开发者可以使用这类API来自行完成同步控制。需要注意的是，通常情况下，开发者基于编程模型中介绍的编程模型和范式进行编程时不需关注同步，编程模型帮助开发者完成了同步控制；使用编程模型和范式是我们推荐的编程方式，自行同步控制可能会带来一定的编程复杂度。

## 8.2 架构规格

### 8.2.1 NPU架构版本200x

本节介绍`__NPU_ARCH__`版本号为200x的硬件架构和其功能说明，其中200代表IP核编号，x表示同一个IP核的配置版本号。对应的产品型号为Atlas推理系列产品。

硬件架构图

[图: NPU架构版本200x硬件架构图，AI Core内Cube计算单元和Vector计算单元同核部署(耦合模式)]

计算单元

Cube计算单元和Vector计算单元同核部署，共享同一个Scalar计算单元。

Vector计算单元：
- Vector计算单元的数据来源来自于Unified Buffer，要求32字节对齐。
- 数据从L0C Buffer传输至Unified Buffer需要以Vector计算单元作为中转。

Cube计算单元：
- Cube计算单元可以访问的存储单元有L0A Buffer、L0B Buffer、L0C Buffer，其中L0A Buffer存储左矩阵，L0B Buffer存储右矩阵，L0C Buffer存储矩阵乘的结果和中间结果。

存储单元

获取存储单元的内存空间大小：开发者可以通过平台信息获取接口查询各存储单元的内存空间大小。

各存储单元的最小访问粒度（对齐要求）

| 存储单元 | 对齐要求 |
|--------|--------|
| Unified Buffer | 32Byte对齐 |
| L1 Buffer | 32Byte对齐 |
| L0A Buffer | 512Byte对齐 |
| L0B Buffer | 512Byte对齐 |
| L0C Buffer | 64Byte对齐 |

各存储单元推荐使用的数据排布格式

- L0A Buffer、L0B Buffer和L0C Buffer推荐分别采用以下分形格式：
  - L0A Buffer：FRACTAL_ZZ
  - L0B Buffer：FRACTAL_ZN
  - L0C Buffer：FRACTAL_NZ
  这些格式针对矩阵乘法等计算密集型任务进行优化，可显著提升计算效率。
- L1 Buffer缓存推荐使用FRACTAL_NZ格式。当L1采用NZ格式时，数据搬运到L0A/L0B Buffer（需分别转换为ZZ和ZN格式）时，可降低格式转换开销。
- Unified Buffer对数据格式没有要求。

解决存储单元的访问冲突，提升读写性能：当多个操作尝试同时访问Unified Buffer同一个bank或者bank group时，可能会发生bank冲突，包括读写冲突、写写冲突、读读冲突，这种冲突会导致访问排队，降低性能。可以通过优化bank分配的方式来提升读写性能，具体信息请参考避免Unified Buffer的bank冲突章节。

搬运单元

搬运时的对齐要求：由于搬运后的数据用于参与数据计算，因此对搬运数据大小有要求，搬运到Unified Buffer的数据大小需要按照DataBlock对齐，其余存储单元的数据搬运必须按分形要求进行搬运。例如，数据从L1 Buffer搬运到L0A Buffer时，数据格式需要从NZ转换为ZZ格式，搬运数据的大小要按分形大小对齐，如果L1 Buffer的剩余大小不足1个分形，则硬件执行中会出现异常。

同步控制

核内同步：由于AI Core内部的执行单元（如MTE2搬运单元、Vector计算单元等）以异步并行的方式运行，在读写Local Memory（如Unified Buffer）时可能存在数据依赖关系。为确保数据一致性及计算正确性，需通过同步控制协调操作时序。

以MTE2从GM搬运数据至UB，进行Vector计算单元的Abs计算，再搬运回GM的流程为例，需满足以下同步条件：

1. 数据搬运与计算顺序
   - GM->UB搬运完成后再启动Vector单元的Abs计算（避免计算时未完成搬运导致的数据缺失）；
   - Vector计算完成后再执行UB->GM的数据搬运（确保结果数据已就绪）。
2. 循环搬运计算场景的同步规则
   - 前序计算完成后再启动新搬运：上一次计算未完成时，不得触发新数据搬运（防止UB中旧数据被覆盖）；
   - 前序数据搬出完成后再启动新计算：上一次数据未完全从UB搬出时，不得触发新计算任务（避免目标内存区域的覆盖冲突）。

[图: 同步控制流程图，展示MTE2/Vector/MTE3三个指令序列之间通过SetFlag/WaitFlag进行同步的时序]

上图中，ID1、ID2、ID3、ID4、ID5、ID6表示事件ID（EventID），每个EventID对应一块存储数据的搬运状态，确保数据操作的正确性和一致性。

需要注意以下几点：

- 建议通过AllocEventID或者FetchEventID接口获取EventID，以确保其合法性和有效性。
- EventID的数量有限，使用后应立即调用ReleaseEventID释放资源，避免EventID耗尽，影响系统正常运行。
- SetFlag和WaitFlag必须成对使用，且SetFlag和WaitFlag的参数必须完全一致（包括模板参数和事件ID）。如果不匹配，可能导致当前核的计算异常，或影响下一个核的算子执行，引发timeout问题。例如，`SetFlag<HardEvent::S_MTE3>(1)`和`SetFlag<HardEvent::MTE3_MTE1>(1)`设置的不是同一个EventID，因为其模板参数不同。只有当模板参数和事件ID完全一致时，才表示同一个EventID。
- 不允许连续设置同一个EventID，因为这可能导致事件状态混乱或未被正确处理。

核间同步：该硬件架构不支持核间同步。

### 8.2.2 NPU架构版本220x

本节介绍`__NPU_ARCH__`版本号为220x的硬件架构和其功能说明，其中220代表IP核编号，x表示同一个IP核的配置版本号。对应的产品型号为：

- Atlas A3训练系列产品/Atlas A3推理系列产品
- Atlas A2训练系列产品/Atlas A2推理系列产品

硬件架构图

本架构中AI Core分为AIC和AIV两个独立的核，分别用于矩阵计算和向量计算。每个核都有自己的Scalar单元，能独立加载自己的代码段。AIV与AIC之间通过Global Memory进行数据传递。

[图: NPU架构版本220x硬件架构详细图，展示AIC核和AIV核分离部署]

计算单元

Cube计算单元和Vector计算单元分离部署，分别部署在AIC核和AIV核上，每个核都有自己的Scalar单元，能独立加载自己的代码段。

Vector计算单元：
- Vector计算单元的数据来源来自于Unified Buffer，要求32字节对齐。

Cube计算单元：
- Cube计算单元可以访问的存储单元有L0A Buffer、L0B Buffer、L0C Buffer，其中L0A Buffer存储左矩阵，L0B Buffer存储右矩阵，L0C Buffer存储矩阵乘的结果和中间结果。

存储单元

获取存储单元的内存空间大小：开发者可以通过平台信息获取接口查询各存储单元的内存空间大小。

各存储单元的最小访问粒度（对齐要求）

| 核 | 存储单元 | 对齐要求 |
|------|--------|--------|
| AIV | Unified Buffer | 32Byte对齐 |
| AIC | L1 Buffer | 32Byte对齐 |
| | L0A Buffer | 512Byte对齐 |
| | L0B Buffer | 512Byte对齐 |
| | L0C Buffer | 64Byte对齐 |
| | BiasTable Buffer | 64Byte对齐 |
| | Fixpipe Buffer | 64Byte对齐 |

各存储单元推荐使用的数据排布格式

- L0A Buffer、L0B Buffer和L0C Buffer推荐分别采用以下分形格式：
  - L0A Buffer：FRACTAL_ZZ
  - L0B Buffer：FRACTAL_ZN
  - L0C Buffer：FRACTAL_NZ
  这些格式针对矩阵乘法等计算密集型任务进行优化，可显著提升计算效率。
- L1 Buffer缓存推荐使用FRACTAL_NZ格式。当L1采用NZ格式时，数据搬运到L0A/L0B Buffer（需分别转换为ZZ和ZN格式）时，可降低格式转换开销。
- Unified Buffer对数据格式没有要求。

解决存储单元的访问冲突，提升读写性能：当多个操作尝试同时访问Unified Buffer同一个bank或者bank group时，可能会发生bank冲突，包括读写冲突、写写冲突、读读冲突，这种冲突会导致访问排队，降低性能。可以通过优化bank分配的方式来提升读写性能，具体信息请参考避免Unified Buffer的bank冲突章节。

搬运单元

搬运时的对齐要求：由于搬运后的数据用于参与数据计算，因此对搬运数据大小有要求，搬运到Unified Buffer的数据大小需要按照DataBlock对齐，其余存储单元的数据搬运必须按分形要求进行搬运。例如，数据从L1 Buffer搬运到L0A Buffer时，数据格式需要从NZ转换为ZZ格式，搬运数据的大小要按分形大小对齐，如果L1 Buffer的剩余大小不足1个分形，则硬件执行中会出现异常。

支持跨卡数据搬运（Hccs物理链路）：在跨卡通信算子开发场景，DataCopy类接口支持跨卡数据搬运，在Atlas A2训练系列产品/Atlas A2推理系列产品设备上，仅支持Hccs物理链路，不支持其他通路；开发者开发过程中，请关注涉及卡间通信的物理通路；通过npu-smi info -t topo指令查询Hccs物理通路。

支持Fixpipe硬件化加速：Fixpipe是NPU将典型操作进行硬化的加速模块，位于AIC内部，配合Cube计算单元完成随路计算，主要功能如下：

- 量化反量化：包括S322FP16、S322S32、S322S4、S322S8、S322S16、FP322FP16、FP322BF16、FP322S8、FP322S4、FP322FP32。
- Relu功能，包括ReLu、PReLu和Leaky ReLu等类型的激活函数。
- 数据格式转换，包括：
  - 通过Channel merge、Channel split可以实现分形大小的转换，保证输出到L1 Buffer/GM的分形满足诉求。
  - NZ2ND数据格式转换。

[图: Fixpipe硬件化加速流程图，展示L0C Buffer -> Quant_PRE -> ReLU_PRE -> Channel merge/split/NZ2ND -> L1 Buffer/GM，以及Fixpipe Buffer提供参数]

Channel merge（S8和U8数据类型）：对于转换为S8或U8的目标数据类型，分形矩阵通过硬件从16x16转换为16x32，如果输出通道数N是16的偶数倍，则N方向上每2个相邻的16x16分形矩阵将合并为1个16x32分形矩阵。如果N是16的倍数的奇数，则将通道1到通道（N-16）合并，最后16个通道保持未合并。

Channel merge（S4和U4数据类型）：对于转换为S4或U4的目标数据类型，分形矩阵通过硬件从16x16转换为16x64，如果输出通道数N是64的倍数，则N方向上每4个相邻的16x16分形矩阵将合并为1个16x64分形矩阵。在这种情况下，N的配置必须是64的倍数。

FP32 Channel split：对于目标类型为FP32，分形矩阵可以通过硬件从16x16转换为16x8，如果使能Channel split，则每个16x16分形矩阵将被分裂为2个16x8分形矩阵。

同步控制

核内同步：由于AI Core内部的执行单元（如MTE2搬运单元、Vector计算单元等）以异步并行的方式运行，在读写Local Memory（如Unified Buffer）时可能存在数据依赖关系。为确保数据一致性及计算正确性，需通过同步控制协调操作时序。

以MTE2从GM搬运数据至UB，进行Vector计算单元的Abs计算，再搬运回GM的流程为例，需满足以下同步条件：

a. 数据搬运与计算顺序
   - GM->UB搬运完成后再启动Vector单元的Abs计算（避免计算时未完成搬运导致的数据缺失）；
   - Vector计算完成后再执行UB->GM的数据搬运（确保结果数据已就绪）。
b. 循环搬运计算场景的同步规则
   - 前序计算完成后再启动新搬运：上一次计算未完成时，不得触发新数据搬运（防止UB中旧数据被覆盖）；
   - 前序数据搬出完成后再启动新计算：上一次数据未完全从UB搬出时，不得触发新计算任务（避免目标内存区域的覆盖冲突）。

[图: 220x架构同步控制流程图]

需要注意以下几点：
- 建议通过AllocEventID或者FetchEventID接口获取EventID，以确保其合法性和有效性。
- EventID的数量有限，使用后应立即调用ReleaseEventID释放资源，避免EventID耗尽，影响系统正常运行。
- SetFlag和WaitFlag必须成对使用，且参数必须完全一致（包括模板参数和事件ID）。如果不匹配，可能导致当前核的计算异常，或影响下一个核的算子执行，引发timeout问题。
- 不允许连续设置同一个EventID，因为这可能导致事件状态混乱或未被正确处理。

核间同步

当不同核之间操作同一块全局内存时，可能存在读后写、写后读以及写后写等数据依赖问题，需要进行核间同步控制。

核间同步控制分为以下几种模式：

- 模式0：AI Core核间的同步控制。对于AIC场景，同步所有的AIC核，直到所有的AIC核都执行到CrossCoreSetFlag时，CrossCoreWaitFlag后续的指令才会执行；对于AIV场景，同步所有的AIV核，直到所有的AIV核都执行到CrossCoreSetFlag时，CrossCoreWaitFlag后续的指令才会执行。
- 模式1：AI Core内部，AIV核之间的同步控制。如果两个AIV核都运行了CrossCoreSetFlag，CrossCoreWaitFlag后续的指令才会执行。
- 模式2：AI Core内部，AIC与AIV之间的同步控制。在AIC核执行CrossCoreSetFlag之后，两个AIV上CrossCoreWaitFlag后续的指令才会继续执行；两个AIV都执行CrossCoreSetFlag后，AIC上CrossCoreWaitFlag后续的指令才能执行。

[图: 核间同步模式示意图，展示AIC 0到AIC N-1以及对应的AIV核之间的同步关系]

例如，在AIC中将L0C的计算结果搬运到GM后，AIV需要将GM的数据搬运到UB。此时，可以使用CrossCoreSetFlag和CrossCoreWaitFlag命令，确保数据从L0C成功搬运到GM后，再从GM搬运到UB。

[图: AIC和AIV之间通过CrossCoreSetFlag/CrossCoreWaitFlag进行核间同步的流程图]

CrossCoreSetFlag和CrossCoreWaitFlag接口配合使用。使用时需传入核间同步的标记ID(flagId)，每个ID对应一个初始值为0的计数器。执行CrossCoreSetFlag后ID对应的计数器增加1；执行CrossCoreWaitFlag时如果对应的计数器数值为0则阻塞不执行；如果对应的计数器大于0，则计数器减一，同时后续指令开始执行。flagId取值范围是0-10。

需要注意以下几点：

- 成对使用：CrossCoreSetFlag和CrossCoreWaitFlag必须成对使用，否则可能导致算子超时问题。
- 一致性要求：CrossCoreSetFlag的模板参数和flagId必须与CrossCoreWaitFlag完全一致，否则视为不同的flagId。例如，`CrossCoreSetFlag<0x0, PIPE_MTE3>(0x8)`和`CrossCoreSetFlag<0x2, PIPE_FIX>(0x8)`设置的不是同一个flagId。
- 避免连续设置：不允许连续设置同一个flagId，以防止计数器状态混乱。
- 与高阶API的使用冲突：Matmul高阶API内部实现中使用了本接口进行核间同步控制，所以不建议开发者同时使用该接口和Matmul高阶API，否则会有flagId冲突的风险。
- 计数器限制：同一flagId的计数器最多可以设置15次。
- 默认流水类型：CrossCoreWaitFlag不需要显式设置指令所在的流水类型，默认使用PIPE_S。

### 8.2.3 NPU架构版本300x

本节介绍`__NPU_ARCH__`版本号为300x的硬件架构和其功能说明，其中300代表IP核编号，x表示同一个IP核的配置版本号。对应的产品型号为：

- Atlas 200I/500 A2推理产品

硬件架构图

[图: NPU架构版本300x硬件架构图，AI Core内包含完整的Cube/Vector/Scalar计算单元和存储单元]

存储单元

获取存储单元的内存空间大小：开发者可以通过平台信息获取接口查询各存储单元的内存空间大小。

各存储单元的最小访问粒度（对齐要求）

| 核 | 存储单元 | 对齐要求 |
|------|--------|--------|
| AIV | Unified Buffer | 32Byte对齐 |
| AIC | L1 Buffer | 32Byte对齐 |
| | L0A Buffer | 512Byte对齐 |
| | L0B Buffer | 512Byte对齐 |
| | L0C Buffer | 64Byte对齐 |
| | BiasTable Buffer | 64Byte对齐 |
| | Fixpipe Buffer | 64Byte对齐 |

各存储单元推荐使用的数据排布格式

- L0A Buffer、L0B Buffer和L0C Buffer推荐分别采用以下分形格式：
  - L0A Buffer：FRACTAL_ZZ
  - L0B Buffer：FRACTAL_ZN
  - L0C Buffer：FRACTAL_NZ
- L1 Buffer缓存推荐使用FRACTAL_NZ格式。
- Unified Buffer对数据格式没有要求。

解决存储单元的访问冲突，提升读写性能：当多个操作尝试同时访问Unified Buffer同一个bank或者bank group时，可能会发生bank冲突。可以通过优化bank分配的方式来提升读写性能。

搬运单元

搬运时的对齐要求与其他架构一致。

支持Fixpipe硬件化加速：Fixpipe位于AIC内部，配合Cube计算单元完成随路计算，主要功能包括量化反量化、Relu功能、数据格式转换（Channel merge/split、NZ2ND）。

同步控制

- 核内同步：与220x架构核内同步机制相同，使用SetFlag/WaitFlag进行同步控制。
- 核间同步：该硬件架构不支持核间同步。

## 8.3 硬件约束

本节介绍硬件约束以及解决方案建议。对应的产品型号为：

- Atlas A3训练系列产品/Atlas A3推理系列产品
- Atlas A2训练系列产品/Atlas A2推理系列产品

表 8-3 硬件约束以及解决方案建议

| 分类 | 硬件约束描述 | 解决方案建议 |
|------|----------|----------|
| 内存访问(L0 Buffer/L1 Buffer/UB等) | 各存储单元的最小访问粒度/地址对齐要求：Unified Buffer 32Byte对齐、L1 Buffer 32Byte对齐、L0A/L0B Buffer 512Byte对齐、L0C Buffer 64Byte对齐、BiasTable Buffer 64Byte对齐、Fixpipe Buffer 128Byte对齐 | 进行数据搬运时，需要感知对齐约束。针对UB，遇到一些非对齐的场景，可以使用非对齐搬运的接口或者通过一些技巧（比如搬入时包含冗余数据，搬出时去除冗余数据）来解决 |
| 内存访问(UB) | UB bank访问冲突（Vector计算访问/搬运访问） | 需要按照芯片要求，在软件实现时错开处理的地址，从而解决bank冲突 |
| 内存访问(GM) | 多核并行同地址访问GM，会被硬件串行化 | 多核访问通过错峰访问（调整数据访问顺序和修改切分策略等），使得第一次加载数据到L2 Cache，后续访问性能反而提升。性能为排队时间，大约下降10%-20% |
| 内存访问(GM) | 单次搬运数据长度16KB以上时，可发挥带宽的最佳性能 | 根据实测经验，单次搬运数据长度16KB以上时，通常能较好地发挥出带宽的最佳性能。因此对于单次搬运，应考虑尽可能的搬运较大的数据块 |
| 内存访问(GM-->L1) | DataCopy中源操作数相邻连续数据块的间隔（前面一个数据块的尾与后面数据块的头的间隔）不要超出65535，单位为DataBlock（32字节） | 前面一个数据块的尾与后面数据块的头的间隔超出65535时，需要拆分成多条指令来实现 |
| 内存访问(GM) | 数据搬运会被拆成128B/256B/512B不同长度进行搬运，非对齐会向上取整 | Tiling尽量让搬运的内轴128B、256B、512B对齐 |
| Cube | MTE1和MMAD指令队列的深度为32 | 对应指令队列容易满，会阻塞其他指令下发，引起流水断流。Load2D从L1 Buffer搬运到L0 Buffer，需要发射32条指令，Load3D只需要一条指令，建议使用Load3D来实现 |
| ICache | ICache硬件规格限制32KB | 拆分Tiling_key或使用模板函数来减少代码段 |
| ICache | 多核并行同地址访问ICache，会被硬件串行化 | 小Shape场景尽量减少启动核数，减少多核同地址访问问题 |
| DCache | DCache硬件规格限制32KB | 无 |
| Scalar | 标量写GM时，数据会被缓存在DCache。硬件不保证DCache和GM一致性，需要用户保证 | 使用DataCacheCleanAndInvalid来保证一致性 |
| Cube | L0C Buffer容量128KB | 无 |
| Cube | BiasTable Buffer为1KB | 无 |
| Cube | Cube计算场景float算力为half算力的1/4 | 无 |
| Cube | Cube输出随路量化场景，不支持int32_t到float>bfloat16_t类型量化 | AIV上存在int32_t->float、float->bfloat16_t的转换。AIC支持float->bfloat16_t的随路量化 |
| Vector | Reduce接口half比float性能差 | half写回UB时存在非32Byte对齐，导致性能劣化，建议把half转float计算，此场景建议使用float数据类型 |
| Vector | Exp/Ln接口处理同样数量的half/float的耗时是一样的 | 内部对float做了优化，故两者性能相当，开发者可以根据实际情况选择合适的精度类型 |
| 流水同步(核内) | set/wait同步不匹配，状态会残留，影响后续算子 | 通过孪生调试/mssanitizer工具，提前识别此类问题 |
| 流水同步(核间) | CrossCoreSetFlag计数器存在限制，超出15次需要做一次反向同步，否则会出现卡死 | 通过孪生调试/mssanitizer工具，超出场景提前报错 |
| API通用约束 | 使用Ascend C API时，源操作数和目的操作数的地址重叠通用约束 | 使用基础API的Tensor高维切分计算接口时，为了节省地址空间，开发者可以定义一个Tensor，供源操作数与目的操作数同时使用（即地址重叠）。使用时需要注意以下约束：单次迭代内源操作数与目的操作数须100%完全重叠，不支持部分重叠。多次迭代间：不支持前序迭代的目的操作数与后序迭代的源操作数重叠。特别地，对于部分双目计算类的API（Add、Sub、Mul、Max、Min、AddRelu、SubRelu），当数据类型为half、int32_t、float时，支持前序迭代的目的操作数与后序迭代的源操作数重叠：仅针对目的操作数和第二个源操作数重叠的情况，且src1RepStride或者dstRepStride必须为0 |

# 9 调试调优

## 9.1 概述

Ascend C算子调试的整体方案如下：开发者通过调用Ascend C类库编写Ascend C算子kernel侧源码，kernel侧源码通过通用的GCC编译器进行编译，编译生成通用的CPU域的二进制，可以通过gdb通用调试工具等调试手段进行调试；kernel侧源码通过毕昇编译器进行编译，编译生成NPU域的二进制文件，可以通过仿真打点图或者Profiling工具上板数据采集等方式进行调试。

[图: Ascend C算子调试整体方案图，展示CPU域调试(GCC编译器->CPU bin->GDB/printf)和NPU域调试(毕昇编译器->NPU bin->msprof工具/数据打印)]

表 9-1 调试调优方法和使用的工具列表

| 分类 | 子分类 | 方法 |
|------|------|------|
| 功能调试 | CPU域孪生调试 | 孪生调试：相同的算子代码可以在CPU域调试精度，NPU域调试性能。在CPU域可以进行gdb调试、使用printf命令打印 |
| | NPU域上板调试 | printf/assert：printf主要用于打印标量和字符串信息；assert主要用于在代码中设置检查点，当某个条件不满足时，程序会立即终止并报错。DumpTensor：使用DumpTensor接口打印指定Tensor的数据。上板调试工具：使用msDebug工具调试NPU侧运行的算子程序，在真实的硬件环境中，对算子的输入输出进行测试，以验证算子的功能是否正确。具体功能包括断点设置、打印变量和内存、单步调试、中断运行等。内存检测工具：使用msSanitizer工具进行内存检测，可以检测并报告算子运行中对外部存储（Global Memory）和内部存储（Local Memory）的越界及未对齐等内存访问异常 |
| 性能调优 | - | msProf工具：用于采集和分析运行在昇腾AI处理器上算子的关键性能指标，用户可根据输出的性能数据，快速定位算子的软、硬件性能瓶颈，提升算子性能的分析效率。当前支持基于不同运行模式（上板或仿真）和不同文件形式（可执行文件或算子二进制.o文件）进行性能数据的采集和自动解析 |

## 9.2 功能调试

### 9.2.1 CPU域孪生调试

本节介绍CPU域调试的方法：CPU侧验证核函数，gdb调试、使用printf命令打印。

说明：

- 当前您可以基于Kernel直调样例工程实现CPU域调试功能。异构编译场景，开发者使用命令行或者编写Cmake文件进行编译的情况，暂不支持CPU孪生调试并将在后续的版本中逐步支持。
- CPU调测过程中，配置日志相关环境变量，可以记录程序的运行过程及异常信息，有助于开发者进行功能调测。

CPU侧验证核函数

在非昇腾设备上，开发者可以利用CPU仿真环境先行进行算子开发和测试，并在准备就绪后，利用昇腾设备进行加速计算。Kernel程序NPU域的编译运行。相比于NPU域的算子运行逻辑，CPU域调试，实际上是通过标准的GCC编译器编译算子Kernel程序。此时算子Kernel程序链接CPU调测库，执行编译生成的可执行文件，可以完成算子CPU域的运行验证。CPU侧的运行程序，通过GDB通用调试工具进行单步调试，可以精准验证程序执行流程是否符合预期。

[图: CPU域和NPU域的核函数运行逻辑对比图]

基于Kernel直调样例工程，通过ACLRT_LAUNCH_KERNEL接口调用核函数时，可实现CPU与NPU域的代码的统一，且该方式仅支持以下型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品、Atlas推理系列产品。

如果通过<<<>>>调用核函数，则需要使用单独的调测接口进行内存分配等操作，并对CPU域和NPU域的代码进行宏隔离。

下面代码以add_custom算子为例，介绍算子核函数在CPU侧验证时，算子调用的应用程序如何编写（通过ACLRT_LAUNCH_KERNEL接口调用核函数的方式）。

步骤1 按需包含头文件。

```cpp
#include "data_utils.h"
#include "acl/acl.h"
#include "aclrtlaunch_add_custom.h"
```

步骤2 应用程序框架编写。

```cpp
int32_t main(int32_t argc, char* argv[])
{
    uint32_t blockDim = 8;
    size_t inputByteSize = 8 * 2048 * sizeof(uint16_t);
    size_t outputByteSize = 8 * 2048 * sizeof(uint16_t);
    // 运行算子的调用程序
    return 0;
}
```

步骤3 运行验证。

```cpp
// 初始化
CHECK_ACL(aclInit(nullptr));
// 运行管理资源申请
int32_t deviceId = 0;
CHECK_ACL(aclrtSetDevice(deviceId));
aclrtStream stream = nullptr;
CHECK_ACL(aclrtCreateStream(&stream));
// 分配Host内存
uint8_t *xHost, *yHost, *zHost;
uint8_t *xDevice, *yDevice, *zDevice;

CHECK_ACL(aclrtMallocHost((void**)(&xHost), inputByteSize));
CHECK_ACL(aclrtMallocHost((void**)(&yHost), inputByteSize));
CHECK_ACL(aclrtMallocHost((void**)(&zHost), outputByteSize));
// 分配Device内存
CHECK_ACL(aclrtMalloc((void**)&xDevice, inputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
CHECK_ACL(aclrtMalloc((void**)&yDevice, inputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
CHECK_ACL(aclrtMalloc((void**)&zDevice, outputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
// Host内存初始化
ReadFile("./input/input_x.bin", inputByteSize, xHost, inputByteSize);
ReadFile("./input/input_y.bin", inputByteSize, yHost, inputByteSize);
// 将数据从Host上拷贝到Device上
CHECK_ACL(aclrtMemcpy(xDevice, inputByteSize, xHost, inputByteSize, ACL_MEMCPY_HOST_TO_DEVICE));
CHECK_ACL(aclrtMemcpy(yDevice, inputByteSize, yHost, inputByteSize, ACL_MEMCPY_HOST_TO_DEVICE));
// 用ACLRT_LAUNCH_KERNEL接口调用核函数完成指定的运算
ACLRT_LAUNCH_KERNEL(add_custom)(blockDim, stream, xDevice, yDevice, zDevice);
CHECK_ACL(aclrtSynchronizeStream(stream));
// 将Device上的运算结果拷贝回Host
CHECK_ACL(aclrtMemcpy(zHost, outputByteSize, zDevice, outputByteSize, ACL_MEMCPY_DEVICE_TO_HOST));
WriteFile("./output/output_z.bin", zHost, outputByteSize);
// 释放申请的资源
CHECK_ACL(aclrtFree(xDevice));
CHECK_ACL(aclrtFree(yDevice));
CHECK_ACL(aclrtFree(zDevice));
CHECK_ACL(aclrtFreeHost(xHost));
CHECK_ACL(aclrtFreeHost(yHost));
CHECK_ACL(aclrtFreeHost(zHost));
// 去初始化
CHECK_ACL(aclrtDestroyStream(stream));
CHECK_ACL(aclrtResetDevice(deviceId));
CHECK_ACL(aclFinalize());
```

说明：为了实现CPU域与NPU域代码归一，仅对部分acl接口进行适配，开发者在使用CPU域调测功能时，仅支持使用如下acl接口：

- 有实际功能接口，支持CPU域调用：aclDataTypeSize、aclFloat16ToFloat、aclFloatToFloat16、aclrtMalloc、aclrtFree、aclrtMallocHost、aclrtFreeHost、aclrtMemset、aclrtMemsetAsync、aclrtMemcpy、aclrtMemcpyAsync、aclrtMemcpy2d、aclrtMemcpy2dAsync、aclrtCreateContext、aclrtDestroyContext。
- 无实际功能接口，打桩实现：Profiling数据采集、系统配置、运行时管理。

gdb调试

可使用gdb单步调试算子计算精度。由于cpu调测已转为多进程调试，每个核都会拉起独立的子进程，故gdb需要转换成子进程调试的方式。针对Atlas推理系列产品、Atlas训练系列产品，每个核会拉起1个子进程。针对Atlas A2训练系列产品/Atlas A2推理系列产品，每个核会拉起3个子进程，1个Cube，2个Vector。

- 调试单独一个子进程：启动gdb，示例中的add_custom_cpu为CPU域的算子可执行文件。gdb启动后，首先设置跟踪子进程，之后再打断点。但是这种方式只会停留在遇到断点的第一个子进程中，其余子进程和主进程会继续执行直到退出。涉及到核间同步的算子无法使用这种方法进行调试。

- 调试多个子进程：如果涉及到核间同步，需要能同时调试多个子进程。在gdb启动后，首先设置调试模式为只调试一个进程，挂起其他进程。使用`set detach-on-fork off`和`catch fork`命令。通过切换inferior来调试不同的子进程。

使用printf打印命令打印

在代码中直接编写printf(...)来观察数值的输出。

```cpp
printf("xLocal size: %d\n", xLocal.GetSize());
printf("tileLength: %d\n", tileLength);
```

### 9.2.2 NPU域上板调试

通过DumpTensor、printf打印进行调试

NPU域上板数据打印功能包括DumpTensor、printf两种，其中DumpTensor用于打印指定Tensor的数据，printf主要用于打印标量和字符串信息。

该功能仅在如下场景支持：通过Kernel直调方式调用算子、通过单算子API调用方式调用算子、间接调用单算子API(aclnnxxx)接口（Pytorch框架单算子直调的场景）。

在算子kernel侧实现代码中需要输出日志信息的地方调用DumpTensor接口或者printf接口打印相关内容。

说明：DumpTensor、printf接口打印功能会对算子实际运行的性能带来一定影响，通常在调测阶段使用。开发者可以按需关闭打印功能。

使用msSanitizer工具进行异常检测

msSanitizer工具是基于昇腾AI处理器的异常检测工具，包含了单算子开发场景下的内存检测、竞争检测、未初始化检测和同步检测四个子功能。

- 内存检测：工具可以在用户开发算子的过程中，协助定位非法读写、多核踩踏、非对齐访问、内存泄漏以及非法释放等内存问题。同时工具也支持对CANN软件栈的内存检测，帮助用户定界软件栈内存异常发生的模块。
- 竞争检测：工具可以协助用户定位由于竞争风险可能导致的数据竞争问题，包含核内竞争和核间竞争问题。其中，核内竞争包含流水间竞争和流水内竞争。
- 未初始化检测：工具可以协助用户定位由于内存未初始化可能导致的脏数据读取问题。
- 同步检测：工具可以协助用户定位由于前序算子中的未配对同步指令导致的后续算子同步失败的问题。

使用msDebug工具进行算子调试

msDebug是一款面向昇腾设备的算子测试工具，用于调试NPU侧运行的算子程序，为算子开发人员提供调试手段。msDebug工具支持调试所有的昇腾算子，包含Ascend C算子（Vector、Cube以及融合算子）程序。具体功能包括断点设置、打印变量和内存、单步调试、中断运行、核切换、检查程序状态、调试信息展示、解析Core dump文件，用户可根据实际情况进行选择。

## 9.3 性能调优

NPU模式下的性能采集与分析

基于NPU域算子的调用接口编写的算子程序，通过毕昇编译器编译后生成可执行程序，使用算子调优工具运行NPU模式下生成的可执行文件从而采集Ascend C算子在AI处理器上执行的性能数据，进行性能精细调优。

- Profiling性能数据采集：使用msProf工具采集Ascend C算子在AI处理器上执行的性能数据。
- Roofline瓶颈分析：通过msprof op生成的visualize_data.bin文件可通过MindStudio Insight进行可视化呈现，Roofline瓶颈分析图可构建出处理器的性能模型，然后利用该性能模型快速评估出算子的理论性能极限，协助开发者快速识别瓶颈类型。
- 指令流水图：通过msprof op simulator生成的visualize_data.bin文件或trace.json文件，并进行可视化呈现。指令流水图以指令维度展示时序关系，并关联调用栈快速定位瓶颈位置。

# 10 算子实践参考

## 10.1 SIMD算子实现

### 10.1.1 概述

Ascend C的算子实现主要包含两个部分：

- Host侧Tiling实现：由于NPU中AI Core内部存储无法完全容纳算子输入输出的所有数据，需要每次搬运一部分输入数据进行计算然后搬出，再搬运下一部分输入数据进行计算，这个过程就称之为Tiling。切分数据的算法称为Tiling算法或者Tiling策略。根据算子的shape等信息来确定数据切分算法相关参数（比如每次搬运的块大小，以及总共循环多少次）的计算程序，称之为Tiling实现，也叫Tiling函数（Tiling Function）。由于Tiling实现中完成的均为标量计算，AI Core并不擅长，所以我们将其独立出来放在Host侧CPU上执行。

- Device侧Kernel实现：Kernel实现即算子核函数实现，在Kernel函数内部通过解析Host侧传入的Tiling结构体获取Tiling信息，根据Tiling信息控制数据搬入搬出Local Memory的流程；通过调用计算、数据搬运、内存管理、任务同步API，实现算子逻辑。其核心逻辑基本上都为计算密集型任务，需要在NPU上执行。

本章介绍矢量编程、矩阵编程、融合算子编程三种典型场景下的算子实现，是对上文中三种典型编程范式的具体应用，同时也介绍了编程的更多细节、API的使用方法等。

### 10.1.2 矢量编程

#### 10.1.2.1 概述

本节将以Add算子为例，带您快速构建Ascend C矢量算子程序，并学习矢量算子开发的典型场景以及处理方式。涉及的场景包括：

- 基础矢量算子：开发一个简单的Add矢量算子。
- TBuf的使用：在算子计算过程中使用临时空间存储运算的中间结果。
- 多核Tiling：算子在AI处理器的多个核上运行，所有核的计算数据量相等且32字节对齐。
- 尾块Tiling：算子在AI处理器的多个核上运行，所有核的计算数据量相等，每个核上除最后一个数据块（尾块）外，其余数据块的数据量相等，每个核都需要处理尾块数据的计算。
- 尾核Tiling：算子在AI处理器的多个核上运行，数据无法平均分配到每个核。将所有核分为多个整核和多个尾核，整核的计算数据量相等，尾核的计算数据量相等。
- 尾核&尾块：算子在AI处理器的多个核上运行，数据无法平均分配到每个核，同时每个核内的数据无法均分，除最后一个数据块（尾块）外，其余数据块的数据量相等，每个核都需要单独处理尾块数据的计算。
- DoubleBuffer场景：使能double buffer，算子中的多条流水并行执行。
- Broadcast场景：算子中两个输入的shape（形状）不相等，需要将一个输入的shape进行Broadcast（广播）后，再执行计算。
- 非对齐场景：更多数据非32字节对齐场景的处理方案。

须知：进行数据搬运和Vector计算时，对于搬运的数据长度和操作数的起始地址有如下的对齐要求：

- 使用DataCopy接口进行数据搬运，搬运的数据长度和操作数的起始地址（Unified Buffer上）必须保证32字节对齐。
- 通常情况下，进行Vector计算时，操作数的起始地址必须保证32字节对齐，执行计算的基本单位为32字节。

#### 10.1.2.2 基础矢量算子

基于Ascend C方式实现基础矢量算子核函数的流程如下图所示。

[图: 矢量算子核函数实现流程图，包括算子分析->核函数定义->根据矢量编程范式实现算子类(CopyIn/Compute/CopyOut)]

- 算子分析：分析算子的数学表达式、输入、输出以及计算逻辑的实现，明确需要调用的Ascend C接口。

下文以输入为half数据类型且shape的最后一维为32Bytes对齐、在单核上运行的、一次完成计算的Add算子为例，对上述步骤进行详细介绍。

算子分析

步骤1 明确算子的数学表达式及计算逻辑。Add算子的数学表达式为：`z = x + y`

计算逻辑是：Ascend C提供的矢量计算接口的操作元素都为LocalTensor，输入数据需要先从外部存储（Global Memory）搬运进片上存储（Unified Buffer），然后使用计算接口完成两个输入参数相加，得到最终结果，再搬出到外部存储上。

[图: 算子计算逻辑图，x和y经CopyIn搬运到Local内存，Compute进行计算，CopyOut搬出到z]

步骤2 明确输入和输出。Add算子有两个输入x与y；输出为z。本样例中算子的输入支持的数据类型为half（float16），算子输出的数据类型与输入的数据类型相同。算子输入支持的shape为（1, 2048），输出shape与输入shape相同。算子输入支持的format为ND。

步骤3 确定核函数名称和参数。核函数命名为add_custom。根据对算子输入输出的分析，确定核函数有3个参数x, y, z；x, y为输入在Global Memory上的内存地址，z为输出在Global Memory上的内存地址。

步骤4 确定算子实现所需接口。实现涉及外部存储和内部存储间的数据搬运，需要使用DataCopy；本样例只涉及矢量计算的加法操作，初步分析可使用基础算术Add接口实现x+y；使用Queue队列管理计算中使用的Tensor数据结构，具体使用EnQue、DeQue等接口。

表 10-1 Ascend C Add算子设计规格

| 算子类型（OpType） | Add |
|---|---|
| 算子输入输出 | x（输入）(1,2048) half ND、y（输入）(1,2048) half ND、z（输出）(1,2048) half ND |
| 核函数名称 | add_custom |
| 使用的主要接口 | DataCopy、Add、EnQue/DeQue等 |
| 算子实现文件名称 | add_custom.cpp |

核函数定义

步骤1 函数原型定义：使用`__global__`函数类型限定符标识它是一个核函数，可以被<<<>>>调用；使用`__aicore__`函数类型限定符标识该核函数在设备端aicore上执行；为方便起见，统一使用GM_ADDR宏修饰入参。

```cpp
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z)
{
    KernelAdd op;
    op.Init(x, y, z);
    op.Process();
}
```

步骤2 调用算子类的Init和Process函数。

步骤3 对核函数的调用进行封装，得到add_custom_do函数，便于主程序调用。

```cpp
#ifndef ASCENDC_CPU_DEBUG
// call of kernel function
void add_custom_do(uint32_t blockDim, void* l2ctrl, void* stream, uint8_t* x, uint8_t* y, uint8_t* z)
{
    add_custom<<<blockDim, l2ctrl, stream>>>(x, y, z);
}
#endif
```

算子类实现

根据矢量编程范式对Add算子的实现流程进行设计，Add算子的实现流程分为3个基本任务：CopyIn, Compute, CopyOut。

[图: Add算子实现流程图，展示Stage1:CopyIn(xGm/yGm通过DataCopy到xLocal/yLocal，经inQueueX/Y EnQue)、Stage2:Compute(DeQue取出后Add计算，经outQueueZ EnQue)、Stage3:CopyOut(DeQue取出后DataCopy到zGm)]

算子类KernelAdd具体成员如下：

```cpp
class KernelAdd {
public:
  __aicore__ inline KernelAdd() {}
  __aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z){}
  __aicore__ inline void Process(){}
private:
  __aicore__ inline void CopyIn(){}
  __aicore__ inline void Compute(){}
  __aicore__ inline void CopyOut(){}
private:
  AscendC::TPipe pipe;
  AscendC::TQue<AscendC::TPosition::VECIN, 1> inQueueX;
  AscendC::TQue<AscendC::TPosition::VECIN, 1> inQueueY;
  AscendC::TQue<AscendC::TPosition::VECOUT, 1> outQueueZ;
  AscendC::GlobalTensor<half> xGm;
  AscendC::GlobalTensor<half> yGm;
  AscendC::GlobalTensor<half> zGm;
};
```

初始化函数主要完成以下内容：设置输入输出Global Tensor的Global Memory内存地址；通过Pipe内存管理对象为输入输出Queue分配内存。

```cpp
constexpr int32_t TOTAL_LENGTH = 1 * 2048;
__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z)
{
    xGm.SetGlobalBuffer((__gm__ half *)x, TOTAL_LENGTH);
    yGm.SetGlobalBuffer((__gm__ half *)y, TOTAL_LENGTH);
    zGm.SetGlobalBuffer((__gm__ half *)z, TOTAL_LENGTH);

    pipe.InitBuffer(inQueueX, 1, TOTAL_LENGTH * sizeof(half));
    pipe.InitBuffer(inQueueY, 1, TOTAL_LENGTH * sizeof(half));
    pipe.InitBuffer(outQueueZ, 1, TOTAL_LENGTH * sizeof(half));
}
```

基于矢量编程范式，将核函数的实现分为3个基本任务：CopyIn, Compute, CopyOut。Process函数中通过调用这三个函数：

```cpp
__aicore__ inline void Process()
{
    CopyIn();
    Compute();
    CopyOut();
}
```

Stage1 CopyIn函数实现：使用DataCopy接口将GlobalTensor数据拷贝到LocalTensor，使用EnQue将LocalTensor放入VECIN的Queue中。

```cpp
__aicore__ inline void CopyIn()
{
    AscendC::LocalTensor<half> xLocal = inQueueX.AllocTensor<half>();
    AscendC::LocalTensor<half> yLocal = inQueueY.AllocTensor<half>();
    AscendC::DataCopy(xLocal, xGm, TOTAL_LENGTH);
    AscendC::DataCopy(yLocal, yGm, TOTAL_LENGTH);
    inQueueX.EnQue(xLocal);
    inQueueY.EnQue(yLocal);
}
```

Stage2 Compute函数实现：使用DeQue从VECIN中取出LocalTensor，使用Add完成矢量计算，使用EnQue将计算结果LocalTensor放入到VECOUT的Queue中，使用FreeTensor释放不再使用的LocalTensor。

```cpp
__aicore__ inline void Compute()
{
    AscendC::LocalTensor<half> xLocal = inQueueX.DeQue<half>();
    AscendC::LocalTensor<half> yLocal = inQueueY.DeQue<half>();
    AscendC::LocalTensor<half> zLocal = outQueueZ.AllocTensor<half>();
    AscendC::Add(zLocal, xLocal, yLocal, TOTAL_LENGTH);
    outQueueZ.EnQue<half>(zLocal);
    inQueueX.FreeTensor(xLocal);
    inQueueY.FreeTensor(yLocal);
}
```

Stage3 CopyOut函数实现：使用DeQue接口从VECOUT的Queue中取出LocalTensor，使用DataCopy将LocalTensor拷贝到GlobalTensor上，使用FreeTensor将不再使用的LocalTensor进行回收。

```cpp
__aicore__ inline void CopyOut()
{
    AscendC::LocalTensor<half> zLocal = outQueueZ.DeQue<half>();
    AscendC::DataCopy(zGm, zLocal, TOTAL_LENGTH);
    outQueueZ.FreeTensor(zLocal);
}
```

#### 10.1.2.3 TBuf的使用

在大多数算子开发时，核函数计算过程需要使用临时内存来存储运算的中间结果，这些中间结果以临时变量表示，临时变量占用的内存可以使用TBuf数据结构来管理。

下文将以输入的数据类型为bfloat16_t、在单核上运行的Add算子为例，介绍TBuf的使用方式。

在Atlas A2训练系列产品/Atlas 800I A2推理产品上，Add接口不支持对数据类型bfloat16_t的源操作数进行求和计算。因此，需要先将算子输入的数据类型转换成Add接口支持的数据类型，再进行计算。为保证计算精度，调用Cast接口将输入bfloat16_t类型转换为float类型，再进行Add计算，并在计算结束后将float类型转换回bfloat16_t类型。

[图: 输入为bfloat16_t类型的Add计算流程图，展示Cast(bfloat16_t->float) -> Add -> Cast(float->bfloat16_t)]

与基础矢量算子类相比，本样例新增三个TBuf类型的成员变量tmpBuf0、tmpBuf1、tmpBuf2，用于管理计算过程中使用的临时内存。

Compute函数的实现步骤如下：
1. 使用DeQue从VECIN的Queue中取出LocalTensor。
2. 使用TBuf.Get从TBuf上获取全部长度的Tensor作为临时内存。
3. 使用Cast接口将LocalTensor转换为float类型，并存入临时内存。
4. 使用Add接口完成矢量计算，将计算结果存入临时内存。
5. 使用Cast接口将临时内存中的计算结果转换为bfloat16_t类型。
6. 使用EnQue将bfloat16_t类型的结果LocalTensor放入VECOUT的Queue中。
7. 使用FreeTensor释放不再使用的LocalTensor。

```cpp
__aicore__ inline void Compute()
{
    AscendC::LocalTensor<bfloat16_t> xLocal = inQueueX.DeQue<bfloat16_t>();
    AscendC::LocalTensor<bfloat16_t> yLocal = inQueueY.DeQue<bfloat16_t>();
    AscendC::LocalTensor<bfloat16_t> zLocal = outQueueZ.AllocTensor<bfloat16_t>();

    AscendC::LocalTensor<float> tmpTensor0 = tmpBuf0.Get<float>();
    AscendC::LocalTensor<float> tmpTensor1 = tmpBuf1.Get<float>();
    AscendC::LocalTensor<float> tmpTensor2 = tmpBuf2.Get<float>();
    AscendC::Cast(tmpTensor0, xLocal, AscendC::RoundMode::CAST_NONE, TOTAL_LENGTH);
    AscendC::Cast(tmpTensor1, yLocal, AscendC::RoundMode::CAST_NONE, TOTAL_LENGTH);

    AscendC::Add(tmpTensor2, tmpTensor0, tmpTensor1, TOTAL_LENGTH);
    AscendC::Cast(zLocal, tmpTensor2, AscendC::RoundMode::CAST_RINT, TOTAL_LENGTH);

    outQueueZ.EnQue<bfloat16_t>(zLocal);
    inQueueX.FreeTensor(xLocal);
    inQueueY.FreeTensor(yLocal);
}
```

#### 10.1.2.4 多核&Tiling切分

##### 10.1.2.4.1 概述

Ascend C核函数是运行在一个核上的处理函数，上述介绍的基础矢量算子与TBuf的使用样例均为在单核上运行的算子，不涉及Host侧Tiling实现。

[图: 算子实现组成图，包含Kernel侧算子实现和Host侧Tiling实现（可选）]

为了提高算子的执行效率，通常在算子中实现多核并行计算，即对输入数据进行切分，并将不同的数据块分配到不同的核上处理。此外，由于单个核上内部存储Local Memory大小有限，存在无法一次完整地容纳算子的输入和输出数据的场景，因此需要每次搬运一部分输入进行计算然后搬出，再搬运下一部分输入进行计算，直到获得最终的完整结果，这个数据切分、分块计算的过程称之为Tiling。

由于硬件限制，在对输入数据进行数据切分时应遵循以下几个原则：

1. 由于AI Core中Unified Buffer上的物理限制，要求Unified Buffer上的数据存储空间必须保持32字节对齐。
2. 尽可能最大利用Unified Buffer空间，减少从Global Memory上搬运数据的次数。
3. 昇腾AI处理器包含多个AI Core，应该充分均衡利用多核计算能力，将计算均衡分配到多个AI Core上。

[图: 多核及Tiling示意图，展示TOTAL_LENGTH被切分为多个BLOCK_LENGTH，每个BLOCK_LENGTH再切分为TILE_NUM个TILE_LENGTH]

根据每个核计算的数据量是否相同、核内每个数据块的数据量是否相同，切分策略可能存在以下几种场景：

1. 核间均分，核内均分：通过多核Tiling将数据均匀分配到各个核上执行。
2. 核间均分，核内不均分：此场景基于多核Tiling，核内数据不能切分为多个数据量相同且32字节对齐的数据块，需要通过尾块Tiling处理尾块数据的计算。
3. 核间不均分，核内均分：通过尾核Tiling的处理解决数据无法在各核间均匀分配的问题。
4. 核间不均分，核内不均分：该场景下需要同时考虑尾核&尾块，处理多核间及核内数据的合理切分。

##### 10.1.2.4.2 多核Tiling

基于Ascend C方式实现带有Tiling的算子的开发流程：算子分析 -> Tiling实现（可选）-> Kernel侧算子实现。

[图: 算子开发流程图]

算子分析：本样例为输入数据在核间均分、核内均分场景。TOTAL_LENGTH为8*2048，数据平均分配到8个核上运行，每个核上计算的数据长度BLOCK_LENGTH为2048，将单核上的数据切分成16块，每块数据的长度TILE_LENGTH为128。

Tiling实现

基于本节的切分策略，Tiling需要定义如下参数：blockLength、tileNum、tileLength。

```cpp
struct AddCustomTilingData {
    uint32_t blockLength;
    uint32_t tileNum;
    uint32_t tileLength;
};
```

在Host侧调用程序中，调用Tiling参数计算函数，计算出相关参数，然后传递到Kernel侧核函数。

算子类实现

- 设置输入输出Global Tensor的Global Memory内存地址：由于本样例中将数据分配到了多个核上进行处理，每个核处理不同的数据，不同核要处理的数据在Global Memory上的地址不同，在初始化函数Init中，需要获取单核所需处理的输入输出在Global Memory上的内存偏移地址，并将该偏移地址设置到GlobalTensor中。

```cpp
xGm.SetGlobalBuffer((__gm__ half *)x + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
```

- 通过Pipe内存管理对象为输入输出Queue分配内存：使用单核内每个数据块的长度tileLength作为分配内存的长度。

- Process函数中每个核需要对tileNum个数据块分别进行搬入、计算、搬出处理，因此将tileNum作为循环上限。

```cpp
__aicore__ inline void Process()
{
    int32_t loopCount = this->tileNum;
    for (int32_t i = 0; i < loopCount; i++) {
        CopyIn(i);
        Compute(i);
        CopyOut(i);
    }
}
```

CopyIn和CopyOut函数中使用DataCopy接口时，需增加每个数据块的地址偏移。

##### 10.1.2.4.3 尾块Tiling

针对一些shape，比如算子的输入shape为（1, 1904），支持的数据类型为half类型，输入数据可以对齐到一个datablock的大小（32字节），可以平均分配到每个核上（假设使用8个核），每个核上处理238个数，238个数无法均匀分到datablock上，分满14个datablock后，剩余14个数（28字节），多核切分后需要进行尾块处理。

针对此种场景，在Tiling参数中增加变量lastTileLength，用来表示最后一个分块，即尾块的大小。Tiling结构体包含以下四个成员：

- blockLength：每个核上计算的数据长度；
- tileNum：每个核上切分的数据块的个数；
- tileLength：每个核上除尾块外，每个数据块的长度；
- lastTileLength：每个核上尾块的长度。其中，当lastTileLength等于tileLength时，为核间均分同时核内均分场景，因此这两种场景可以做代码归一化处理。

[图: 多核Tiling尾块示意图]

由于尾块长度为lastTileLength，与其它数据块的长度不同，因此CopyIn函数、CopyOut函数需要对尾块单独处理。

##### 10.1.2.4.4 尾核Tiling

对于不同shape的输入进行数据切分时，可能会发生数据无法平均分配到多个核的情况。例如当算子的输入shape为[1, 1999]，使用核数为8，数据类型为half时，需要计算的数据总量为1 * 1999 * sizeof(half) = 3998字节，3998字节既不满足32字节对齐，也无法平均分配到8个核上。因此该场景下，对数据进行多核切分后，每个核的计算数据量不同。这种情况下，应该尽可能均匀的分配数据，所有核上的计算数据量有两种情况，将计算量较多的核称为整核，计算量较少的核称为尾核。

[图: 数据无法均分到每个核上的示例图]

基于上文，设计如下的算子Tiling结构体成员：

- formerNum：分配到数据量较多的核数，即整核的核数。
- tailNum：分配到数据量较少的核数，即尾核的核数。
- formerLength：整核计算的数据长度。
- tailLength：尾核计算的数据长度。

在Kernel侧的Init函数中，计算输入在Global Memory上的内存偏移地址时，应对整核和尾核加以区分。

```cpp
__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    if (AscendC::GetBlockIdx() < formerNum) {
        this->tileLength = formerLength;
        xGm.SetGlobalBuffer((__gm__ half *)x + formerLength * AscendC::GetBlockIdx(), formerLength);
        // ... y, z同理
    } else {
        this->tileLength = tailLength;
        xGm.SetGlobalBuffer((__gm__ half *)x + formerLength * formerNum + tailLength *
            (AscendC::GetBlockIdx() - formerNum), tailLength);
        // ... y, z同理
    }
    pipe.InitBuffer(inQueueX, 1, this->tileLength * sizeof(half));
    pipe.InitBuffer(inQueueY, 1, this->tileLength * sizeof(half));
    pipe.InitBuffer(outQueueZ, 1, this->tileLength * sizeof(half));
}
```

##### 10.1.2.4.5 尾核&尾块

对于不同shape的输入进行数据切分时，可能会发生数据无法平均分配到多个核、同时每个核内的数据无法均分的情况。参考核间均分场景下的尾块处理与核间不均分场景下的尾核处理的处理方式，将两者结合起来考虑整核的尾块、尾核的尾块的处理方式。

Tiling实现

由于本场景中核间、核内的数据均无法均分，在核间不均分场景下的尾核处理定义的Tiling结构体的基础上增加两个成员变量：

- formerLastTileLength：数据量多的核最后一个分块大小，即整核的尾块大小。计算时，先按10.1.2.4.4尾核Tiling中提到的分核策略，切分数据量多的核。

```cpp
// shape需要对齐到的datablock
uint32_t totalLengthAligned = ((totalLength + ALIGN_NUM - 1) / ALIGN_NUM) * ALIGN_NUM;
// 计算整核数量
uint32_t formerNum = (totalLengthAligned / ALIGN_NUM) % BLOCK_DIM;
// 计算整核的数据量
uint32_t formerLength = ((totalLengthAligned / BLOCK_DIM + ALIGN_NUM - 1) / ALIGN_NUM) * ALIGN_NUM;
```

再按10.1.2.4.3尾块Tiling中的切分策略，计算尾块长度。

```cpp
uint32_t formerTileNum = formerLength / ALIGN_NUM / UB_BLOCK_NUM;
if ((formerLength / alignNum) % UB_BLOCK_NUM == 0 || formerTileNum == 0) {
    if (formerTileNum == 0) {
        formerTileNum = 1;
    }
    if (formerLength < UB_BLOCK_NUM * ALIGN_NUM) {
        formerTileLength = formerLength / ALIGN_NUM * ALIGN_NUM;
        formerLastTileLength = formerTileLength;
    } else {
        formerTileLength = UB_BLOCK_NUM * ALIGN_NUM;
        formerLastTileLength = formerTileLength;
    }
} else {
    formerTileNum = formerTileNum + 1;
    formerTileLength = UB_BLOCK_NUM * ALIGN_NUM;
    formerLastTileLength = (formerLength - (formerTileNum - 1) * formerTileLength);
}
```

- tailLastTileLength：数据量少的核最后一个分块大小，即尾核的尾块大小。计算时，先按10.1.2.4.4尾核Tiling中提到的分核策略，切分数据量少的核。

```cpp
// 计算尾核数量
uint32_t tailNum = BLOCK_DIM - formerNum;
// 计算尾核的数据量
uint32_t tailLength = (totalLengthAligned / BLOCK_DIM / ALIGN_NUM) * ALIGN_NUM;
```

再按10.1.2.4.3尾块Tiling中的切分策略，计算尾块长度。

```cpp
uint32_t tailTileNum = tailLength / ALIGN_NUM / UB_BLOCK_NUM;
if ((tailLength / alignNum) % UB_BLOCK_NUM == 0 || tailTileNum == 0) {
    if (tailTileNum == 0) {
        tailTileNum = 1;
    }
    if (tailLength < UB_BLOCK_NUM * ALIGN_NUM) {
        tailTileLength = tailLength / ALIGN_NUM * ALIGN_NUM;
        tailLastTileLength = tailTileLength;
    } else {
        tailTileLength = UB_BLOCK_NUM * ALIGN_NUM;
        tailLastTileLength = tailTileLength;
    }
} else {
    tailTileNum = tailTileNum + 1;
    tailTileLength = UB_BLOCK_NUM * ALIGN_NUM;
    tailLastTileLength = (tailLength - (tailTileNum - 1) * tailTileLength);
}
```

算子类实现

Kernel侧Init函数和Process函数的实现需将核间均分场景下的尾块处理与核间不均分场景下的尾核处理的实现结合起来。

Init函数中由于整核和尾核对应的tileLength和lastTileLength不同，因此需按照核间不均分场景下的尾核处理中提到的分别处理整核和尾核。

Init函数实现代码如下：

```cpp
__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    if (AscendC::GetBlockIdx() < formerNum) {
        this->tileNum = tiling.formerTileNum;
        this->tileLength = tiling.formerTileLength;
        this->lastTileLength = tiling.formerLastTileLength;

        xGm.SetGlobalBuffer((__gm__ half *)x + tiling.formerLength * AscendC::GetBlockIdx(),
            tiling.formerLength);
        yGm.SetGlobalBuffer((__gm__ half *)y + tiling.formerLength * AscendC::GetBlockIdx(),
            tiling.formerLength);
        zGm.SetGlobalBuffer((__gm__ half *)z + tiling.formerLength * AscendC::GetBlockIdx(),
            tiling.formerLength);
    } else {
        this->tileNum = tiling.tailTileNum;
        this->tileLength = tiling.tailTileLength;
        this->lastTileLength = tiling.tailLastTileLength;

        xGm.SetGlobalBuffer((__gm__ half *)x + tiling.formerLength * tiling.formerNum +
            tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
        yGm.SetGlobalBuffer((__gm__ half *)y + tiling.formerLength * tiling.formerNum +
            tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
        zGm.SetGlobalBuffer((__gm__ half *)z + tiling.formerLength * tiling.formerNum +
            tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
    }

    pipe.InitBuffer(inQueueX, 1, this->tileLength * sizeof(half));
    pipe.InitBuffer(inQueueY, 1, this->tileLength * sizeof(half));
    pipe.InitBuffer(outQueueZ, 1, this->tileLength * sizeof(half));
}
```

CopyIn函数、CopyOut函数的整块和尾块处理按照核间均分场景下的尾块处理方式，尾块场景单独处理。

CopyIn函数实现代码如下：

```cpp
__aicore__ inline void CopyIn(int32_t progress)
{
    AscendC::LocalTensor<dataType> xLocal = inQueueX.AllocTensor<dataType>();
    AscendC::LocalTensor<dataType> yLocal = inQueueY.AllocTensor<dataType>();
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopy(xLocal, xGm[progress * this->tileLength],
            this->lastTileLength);
        AscendC::DataCopy(yLocal, yGm[progress * this->tileLength],
            this->lastTileLength);
    } else {
        AscendC::DataCopy(xLocal, xGm[progress * this->tileLength], this->tileLength);
        AscendC::DataCopy(yLocal, yGm[progress * this->tileLength], this->tileLength);
    }
    inQueueX.EnQue(xLocal);
    inQueueY.EnQue(yLocal);
}
```

CopyOut函数实现代码如下：

```cpp
__aicore__ inline void CopyOut(int32_t progress)
{
    AscendC::LocalTensor<dataType> zLocal = outQueueZ.DeQue<dataType>();
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopy(zGm[progress * this->tileLength], zLocal,
            this->lastTileLength);
    } else {
        AscendC::DataCopy(zGm[progress * this->tileLength], zLocal, this->tileLength);
    }
    outQueueZ.FreeTensor(zLocal);
}
```

#### 10.1.2.5 DoubleBuffer场景

因存在算子中多次搬入搬出数据的场景，为充分利用硬件资源，实现多流水并行，引入DoubleBuffer机制。DoubleBuffer是通过将输入数据分成大小相等的两块，充分利用AI Core的硬件资源，实现数据搬入、计算、数据搬出的并行执行方式。下面以"核间不均，核内不均分"的样例为例，介绍算子中DoubleBuffer的实现，完整样例代码请参见使用DoubleBuffer的Add算子样例。

[图: DoubleBuffer数据切分示意图，展示TOTAL_LENGTH被切分为多个BLOCK_LENGTH，每个BLOCK_LENGTH经Tiling后再经DoubleBuffer分成两块]

Tiling实现

使能DoubleBuffer后，每一个数据块会分成大小相等的两块，因此，若要使能DoubleBuffer，要求数据总量应该能够均分。为了简化处理，将可用的Unified Buffer空间以32字节为粒度，分成n块dataBlock，如果n不是偶数，则减1，这样就可以保证一套代码兼容开启或不开启DoubleBuffer功能。对应步骤如下：

步骤1 判断数据总长度totalLength是否满足32字节对齐，如不满足，则计算totalLength向上32字节对齐后的长度totalLengthAligned。

```cpp
constexpr uint32_t BLOCK_SIZE = 32;
// 为方便计算，这里根据数据类型定义变量alignNum作为对齐数
uint32_t alignNum = BLOCK_SIZE / dataTypeSize;
// totalLength为数据总量
uint32_t totalLengthAligned = (totalLength % alignNum == 0) ?
    totalLength : ((totalLength + alignNum - 1) / alignNum) * alignNum;
```

步骤2 根据totalLengthAligned，计算每个核的计算数据长度blockLength，分核策略可参考10.1.2.4.4尾核Tiling。

步骤3 计算其余Tiling参数。

对当前Unified Buffer可用空间以32字节为粒度，进行切分，计算出数据块个数UB_BLOCK_NUM。根据是否开启DoubleBuffer计算出当前可用的最大数据块个数，记作MAX_AVAILABLE_UB_BLOCK_NUM。最后，以MAX_AVAILABLE_UB_BLOCK_NUM为粒度，对blockLength进行切分。为方便演示，如下代码直接给出UB_BLOCK_NUM，作为当前Unified Buffer可用空间包含的block（32字节）数。

```cpp
constexpr uint32_t BUFFER_NUM = 2;
constexpr uint32_t UB_BLOCK_NUM = 21; // UB最大可以使用的block数量
constexpr uint32_t MAX_AVAILABLE_UB_BLOCK_NUM = UB_BLOCK_NUM / BUFFER_NUM * BUFFER_NUM;

tileNum = blockLength / (alignNum * MAX_AVAILABLE_UB_BLOCK_NUM);
if ((blockLength / alignNum) % MAX_AVAILABLE_UB_BLOCK_NUM == 0 || tileNum == 0) {
    if (tileNum == 0) {
        tileNum = 1;
    }
    if (blockLength < MAX_AVAILABLE_UB_BLOCK_NUM * alignNum) {
        tileLength = ((blockLength / alignNum) + 1) / BUFFER_NUM * BUFFER_NUM * alignNum;
        lastTileLength = tileLength;
    } else {
        tileLength = MAX_AVAILABLE_UB_BLOCK_NUM * alignNum;
        lastTileLength = (blockLength - (tileNum - 1) * tileLength);
    }
} else {
    tileNum = tileNum + 1;
    tileLength = MAX_AVAILABLE_UB_BLOCK_NUM * alignNum;
    lastTileLength = (blockLength - (tileNum - 1) * tileLength);
}
```

算子类实现

不开启DoubleBuffer时，只需要对每个核上最后一个分块的起始地址做处理；开启DoubleBuffer后，需要处理的数据块长度变成原来的一半，所以需要对最后两个数据块的起始地址做处理。

开启DoubleBuffer，参考InitBuffer接口函数原型，将num参数配置成2。

```cpp
pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(dataType));
pipe.InitBuffer(inQueueY, BUFFER_NUM, this->tileLength * sizeof(dataType));
pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(dataType));
```

同时在计算核内每个数据块的长度时，考虑DoubleBuffer场景，需要将Buffer数量，即BUFFER_NUM=2带入计算。

```cpp
this->tileLength = tiling.tileLength / BUFFER_NUM;
```

由于无法保证尾块满足DoubleBuffer的条件，因此不对尾块进行切分。

```cpp
this->lastTileLength = tiling.lastTileLength;
```

Init函数实现代码如下：

```cpp
__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    if (tiling.isEvenCore) {
        this->blockLength = tiling.blockLength;
        this->tileNum = tiling.tileNum;
        this->tileLength = tiling.tileLength / BUFFER_NUM;
        this->lastTileLength = tiling.lastTileLength;

        xGm.SetGlobalBuffer((__gm__ dataType *)x + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
        yGm.SetGlobalBuffer((__gm__ dataType *)y + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
        zGm.SetGlobalBuffer((__gm__ dataType *)z + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    } else {
        if (AscendC::GetBlockIdx() < tiling.formerNum) {
            this->tileNum = tiling.formerTileNum;
            this->tileLength = tiling.formerTileLength / BUFFER_NUM;
            this->lastTileLength = tiling.formerLastTileLength;

            xGm.SetGlobalBuffer((__gm__ dataType *)x + tiling.formerLength * AscendC::GetBlockIdx(),
                tiling.formerLength);
            yGm.SetGlobalBuffer((__gm__ dataType *)y + tiling.formerLength * AscendC::GetBlockIdx(),
                tiling.formerLength);
            zGm.SetGlobalBuffer((__gm__ dataType *)z + tiling.formerLength * AscendC::GetBlockIdx(),
                tiling.formerLength);
        } else {
            this->tileNum = tiling.tailTileNum;
            this->tileLength = tiling.tailTileLength / BUFFER_NUM;
            this->lastTileLength = tiling.tailLastTileLength;

            xGm.SetGlobalBuffer((__gm__ dataType *)x + tiling.formerLength * tiling.formerNum +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
            yGm.SetGlobalBuffer((__gm__ dataType *)y + tiling.formerLength * tiling.formerNum +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
            zGm.SetGlobalBuffer((__gm__ dataType *)z + tiling.formerLength * tiling.formerNum +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
        }
    }
    pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(dataType));
    pipe.InitBuffer(inQueueY, BUFFER_NUM, this->tileLength * sizeof(dataType));
    pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(dataType));
}
```

由于开启DoubleBuffer后，切分后的数据块个数翻倍，在Process函数中，需要将BUFFER_NUM带入计算。

```cpp
__aicore__ inline void Process()
{
    // loop count need to be doubled, due to DoubleBuffer
    constexpr int32_t loopCount = TILE_NUM * BUFFER_NUM;
    for (int32_t i = 0; i < loopCount; i++) {
        CopyIn(i);
        Compute(i);
        CopyOut(i);
    }
}
```

CopyIn函数、CopyOut函数需要对尾块进行单独处理。对于最后一个数据块，先向前偏移tileLength + lastTileLength索引，再使用tileLength作为单次计算量（仅作为参考，可能并非最佳）。

CopyIn函数实现代码如下：

```cpp
__aicore__ inline void CopyIn(int32_t progress)
{
    AscendC::LocalTensor<dataType> xLocal = inQueueX.AllocTensor<dataType>();
    AscendC::LocalTensor<dataType> yLocal = inQueueY.AllocTensor<dataType>();
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopy(xLocal, xGm[(progress - 2) * this->tileLength + this->lastTileLength],
            this->tileLength);
        AscendC::DataCopy(yLocal, yGm[(progress - 2) * this->tileLength + this->lastTileLength],
            this->tileLength);
    } else {
        AscendC::DataCopy(xLocal, xGm[progress * this->tileLength], this->tileLength);
        AscendC::DataCopy(yLocal, yGm[progress * this->tileLength], this->tileLength);
    }
    inQueueX.EnQue(xLocal);
    inQueueY.EnQue(yLocal);
}
```

CopyOut函数实现代码如下：

```cpp
__aicore__ inline void CopyOut(int32_t progress)
{
    AscendC::LocalTensor<dataType> zLocal = outQueueZ.DeQue<dataType>();
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopy(zGm[(progress - 2) * this->tileLength + this->lastTileLength], zLocal,
            this->tileLength);
    } else {
        AscendC::DataCopy(zGm[progress * this->tileLength], zLocal, this->tileLength);
    }
    outQueueZ.FreeTensor(zLocal);
}
```

#### 10.1.2.6 Broadcast场景

在某些场景下，可能会存在两个输入shape不相同的情况。由于Add接口只支持对shape相同的输入进行计算，因此需要先对输入进行shape变换，再进行Add计算。本节将对满足Broadcast条件的输入在算子实现中的Broadcast处理进行介绍，其他场景可以参考本章节中提供的思路。

须知：Broadcast机制通过扩展较小维度的数据，使得不同shape的输入能够进行运算，从而避免了显式的复制操作，提高了计算效率。数据进行Broadcast需满足：两个输入的维度个数相同，并且仅在某一个维度上的长度不同，某一个输入在此维度的长度为1。比如：shape为(32, 8)和(32, 1)的两个输入可以进行Broadcast，因为它们都是二维，且第一个维度大小相等，而不相等的维度中第二个输入的维度为1，满足条件。

本节中将使用Broadcast接口，因此输入需满足该API相关约束。同时，由于硬件限制，该API的输入地址需满足32字节对齐。本节以输入维度为2、第二个轴（axis = 1）需要Broadcast为例进行说明。完整的样例代码请参见输入Broadcast的Add算子样例。

Tiling实现

与输入shape相同的场景相比，在Tiling结构体中增加相应的成员变量，表示是否需要对输入进行Broadcast、需要对哪个维度进行Broadcast、Broadcast的轴需要扩充的倍数。因此新增四个Tiling结构体成员：

- xLen和yLen：表示两个输入的数据长度。
- axis：表示对输入的哪个维度进行Broadcast。
- coef：表示Broadcast的输入需要扩维的倍数。例如，x shape为(m, n)，y shape为(1, n)，则coef = m。

[图: axis=1时coef示意图，展示shape=(1,n)扩展为shape=(m,n)的过程]

Tiling结构体定义代码如下所示：

```cpp
struct AddCustomTilingData {
    uint32_t xLen;
    uint32_t yLen;
    uint32_t coef;
    uint32_t axis;
    ...
};
```

设需要进行Broadcast的输入长度为shorterAxisLen；不需要进行Broadcast的输入长度为totalLength。

```cpp
constexpr uint32_t BLOCK_SIZE = 32;
... // 读入数据
uint32_t totalLength = (xLen > yLen) ? xLen : yLen;
uint32_t shorterAxisLen = (xLen < yLen) ? xLen : yLen;
```

使用shorterAxisLen进行分核计算，并使用分核后的长度与coef相乘作为totalLength的分核长度。

```cpp
constexpr uint32_t BLOCK_SIZE = 32;
if (shorterAxisLen % (BLOCK_DIM * BUFFER_NUM) == 0) {
    uint32_t blockLength = shorterAxisLen / BLOCK_DIM * coef;
    ...
} else {
    uint32_t formerNum = (shorterAxisLen / BUFFER_NUM) % BLOCK_DIM;
    uint32_t tailNum = BLOCK_DIM - formerNum;

    uint32_t formerLength = ((shorterAxisLen / BUFFER_NUM) / BLOCK_DIM + 1) * BUFFER_NUM * coef;
    uint32_t tailLength = ((shorterAxisLen / BUFFER_NUM) / BLOCK_DIM) * BUFFER_NUM * coef;
    ....
}
```

进行核内数据切分时，需要计算Unified Buffer数据块的数量向coef和BUFFER_NUM对齐之后的数量ubBlockAligned。

```cpp
ubBlockAligned =
    (UB_BLOCK_NUM * alignNum / (coef * BUFFER_NUM) * (coef * BUFFER_NUM) == 0) ?
    UB_BLOCK_NUM : UB_BLOCK_NUM * alignNum / (coef * BUFFER_NUM) * (coef * BUFFER_NUM);
...
tileNum = length / ubBlockAligned;
if (length % ubBlockAligned == 0 || tileNum == 0) {
    if (tileNum == 0) {
        tileNum = 1;
    }
    if (length < ubBlockAligned) {
        tileLength = length;
        lastTileLength = tileLength;
    } else {
        tileLength = ubBlockAligned;
        lastTileLength = tileLength;
    }
} else {
    tileNum = tileNum + 1;
    tileLength = ubBlockAligned;
    lastTileLength = (length - (tileNum - 1) * tileLength);
}
```

算子类实现

在核函数初始化阶段，根据Tiling结构体传入的参数确定对哪个输入进行Broadcast。由于针对输入的第二个轴（axis = 1）进行Broadcast，可以计算出，对于需要进行Broadcast的输入，每个核搬入数据长度为blockLength / coef。

初始化函数代码如下：

```cpp
__aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, AddCustomTilingData tiling)
{
    GM_ADDR longerInputPtr;
    GM_ADDR shorterInputPtr;
    if (tiling.xLen > tiling.yLen) {
        longerInputPtr = x;
        shorterInputPtr = y;
    } else {
        longerInputPtr = y;
        shorterInputPtr = x;
    }
    this->coef = tiling.coef;
    if (tiling.isEvenCore) {
        this->tileNum = tiling.tileNum;
        this->tileLength = tiling.tileLength / BUFFER_NUM;
        this->lastTileLength = tiling.lastTileLength;
        xGm.SetGlobalBuffer((__gm__ dataType *)longerInputPtr + tiling.blockLength *
            AscendC::GetBlockIdx(), tiling.blockLength);
        yGm.SetGlobalBuffer((__gm__ dataType *)shorterInputPtr + tiling.blockLength *
            AscendC::GetBlockIdx() / this->coef, tiling.blockLength / this->coef);
        zGm.SetGlobalBuffer((__gm__ dataType *)z + tiling.blockLength * AscendC::GetBlockIdx(),
            tiling.blockLength);
    } else {
        if (AscendC::GetBlockIdx() < tiling.formerNum) {
            this->tileNum = tiling.formerTileNum;
            this->tileLength = tiling.formerTileLength / BUFFER_NUM;
            this->lastTileLength = tiling.formerLastTileLength;
            xGm.SetGlobalBuffer((__gm__ dataType *)longerInputPtr + tiling.formerLength *
                AscendC::GetBlockIdx(), tiling.formerLength);
            yGm.SetGlobalBuffer((__gm__ dataType *)shorterInputPtr + tiling.formerLength *
                AscendC::GetBlockIdx() / this->coef, tiling.formerLength / this->coef);
            zGm.SetGlobalBuffer((__gm__ dataType *)z + tiling.formerLength * AscendC::GetBlockIdx(),
                tiling.formerLength);
        } else {
            this->tileNum = tiling.tailTileNum;
            this->tileLength = tiling.tailTileLength / BUFFER_NUM;
            this->lastTileLength = tiling.tailLastTileLength;
            xGm.SetGlobalBuffer((__gm__ dataType *)longerInputPtr + tiling.formerLength * tiling.formerNum +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
            yGm.SetGlobalBuffer((__gm__ dataType *)shorterInputPtr + tiling.formerLength *
                tiling.formerNum / this->coef +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum) / this->coef,
                tiling.tailLength / this->coef);
            zGm.SetGlobalBuffer((__gm__ dataType *)z + tiling.formerLength * tiling.formerNum +
                tiling.tailLength * (AscendC::GetBlockIdx() - tiling.formerNum), tiling.tailLength);
        }
    }
    pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(dataType));
    pipe.InitBuffer(inQueueY, BUFFER_NUM, this->coef * sizeof(dataType));
    pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(dataType));
    pipe.InitBuffer(tmpBuf2, this->tileLength * sizeof(dataType));
}
```

由于数据是向coef对齐的，在数据拷贝的过程中可能会出现地址不满足32字节对齐的场景，因此CopyIn函数、CopyOut函数中使用DataCopyPad进行数据拷贝。

CopyIn函数实现代码如下：

```cpp
__aicore__ inline void CopyIn(int32_t progress)
{
    AscendC::LocalTensor<dataType> xLocal = inQueueX.AllocTensor<dataType>();
    AscendC::LocalTensor<dataType> yLocal = inQueueY.AllocTensor<dataType>();
    AscendC::DataCopyExtParams copyXParams = {1, (uint32_t)(this->tileLength * sizeof(dataType)), 0, 0, 0};
    AscendC::DataCopyExtParams copyYParams = {1, (uint32_t)(this->tileLength * sizeof(dataType) / this->coef), 0, 0, 0};
    AscendC::DataCopyPadExtParams<dataType> padParams = {false, 0, 0, 0};
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopyPad<dataType>(xLocal, xGm[(progress - LAST_TWO_TILE) * this->tileLength + this->lastTileLength],
            copyXParams, padParams);
        AscendC::DataCopyPad<dataType>(yLocal, yGm[((progress - LAST_TWO_TILE) * this->tileLength + this->lastTileLength) / this->coef],
            copyYParams, padParams);
    } else {
        AscendC::DataCopyPad<dataType>(xLocal, xGm[progress * this->tileLength], copyXParams, padParams);
        AscendC::DataCopyPad<dataType>(yLocal, yGm[progress * this->tileLength / this->coef],
            copyYParams, padParams);
    }
    inQueueX.EnQue(xLocal);
    inQueueY.EnQue(yLocal);
}
```

CopyOut函数实现代码如下：

```cpp
__aicore__ inline void CopyOut(int32_t progress)
{
    AscendC::LocalTensor<dataType> zLocal = outQueueZ.DeQue<dataType>();
    AscendC::DataCopyExtParams copyParams = {1, (uint32_t)(this->tileLength * sizeof(dataType)), 0, 0, 0};
    if (progress == (this->tileNum * BUFFER_NUM - 1)) {
        AscendC::DataCopyPad<dataType>(zGm[(progress - LAST_TWO_TILE) * this->tileLength + this->lastTileLength], zLocal, copyParams);
    } else {
        AscendC::DataCopyPad<dataType>(zGm[progress * this->tileLength], zLocal, copyParams);
    }
    outQueueZ.FreeTensor(zLocal);
}
```

在Compute函数中，调用Add接口前需要先对输入进行Broadcast。这里需要计算Broadcast前后的shape。基于前文提到的数据关系，可以计算得出Broadcast前后的shape分别为{tileLength / broadcastCoef, 1}和{tileLength / broadcastCoef, broadcastCoef}。在此基础上对输入进行Broadcast，并将计算结果存入临时空间中，然后进行Add计算。实现代码示例如下所示：

```cpp
__aicore__ inline void Compute(int32_t progress)
{
    AscendC::LocalTensor<dataType> xLocal = inQueueX.DeQue<dataType>();
    AscendC::LocalTensor<dataType> yLocal = inQueueY.DeQue<dataType>();
    AscendC::LocalTensor<dataType> zLocal = outQueueZ.AllocTensor<dataType>();
    AscendC::LocalTensor<dataType> broadcastTmpTensor = broadcastTmpBuf.Get<dataType>();
    uint32_t dstShape[] = {this->tileLength / this->coef, this->coef};
    int32_t srcShape[] = {this->tileLength / this->coef, 1};
    AscendC::BroadCast<dataType, 2, 1>(broadcastTmpTensor, yLocal, dstShape, srcShape);
    ...
}
```

#### 10.1.2.7 非对齐场景

本节介绍非32字节对齐数据的更多处理方式，包括数据搬入、计算和搬出的处理。用户在实际算子开发中，可以参考如下方案介绍和算子样例灵活地处理非对齐场景。

数据搬运和Vector计算的对齐要求

须知：进行数据搬运和Vector计算时，对于搬运的数据长度和操作数的起始地址有如下的对齐要求：

- 使用DataCopy接口进行数据搬运，搬运的数据长度和操作数的起始地址（UB上）必须保证32字节对齐。
- 通常情况下，进行Vector计算时，操作数的起始地址必须保证32字节对齐。具体对齐要求需要查阅对应的API参考进行确认。

下文描述中的Global指Global Memory上的tensor，Local指Local Memory上的tensor。

下面是一些非对齐搬运和计算的例子。

- 非对齐搬入：当需要从Global拷贝11个half数值到Local时，为保证搬运的数据长度32字节对齐，使用DataCopy拷贝16个half（32B）数据到Local上，Local[11]~Local[15]被写成无效数据-1。

[图: 非对齐搬入示意图，Global中0-10的有效数据加上-1填充，DataCopy到Local中保持32B对齐]

- 非对齐搬出：当需要从Local拷贝11个half数值到Global时，为保证搬运的数据长度32字节对齐，使用DataCopy拷贝16个half（32B）数据到Global上，Global[11]~Global[15]被写成无效数据-1。

[图: 非对齐搬出示意图，Local中数据通过DataCopy搬到Global，多余位置被填充-1]

- 矢量计算起始地址非32字节对齐的错误示例：矢量计算时需要保证起始地址32字节对齐，如下的示例中，从Local1[7]，即LocalTensor的第8个数开始计算，起始地址不满足32字节对齐，是错误示例。

[图: 矢量计算起始地址非32字节对齐的错误示例，展示Abs(Local2, Local1[7], 9)导致错误]

非对齐处理方案

DataCopyPad接口提供非对齐搬运的功能，如果基于该接口支持的产品开发算子（参见产品支持情况），则可以直接使用该接口解决非对齐场景下的搬运问题。使用DataCopyPad的完整示例请参考DataCopyPad样例（工程化算子开发）和DataCopyPad样例（kernel直调）。

部分型号不支持DataCopyPad接口，需要参考如下的方案处理。

[图: 非对齐处理方案示意图，展示CopyIn阶段(搬入后LocalTensor存在冗余数据)、Compute阶段(多种冗余数据处理方式)、CopyOut阶段(多种搬出方式)]

由于搬入时搬运的数据长度必须保证32字节对齐。数据长度非对齐的情况下，从Global逐行搬运Tensor数据到Local中，Local中每行都存在冗余数据。

搬入后，进行矢量计算时对冗余数据的处理方式有以下几种：

- 冗余数据参与计算。一般用于elewise计算场景。
- 通过mask参数掩掉冗余数据。一般用于轴归约计算等场景。
- 通过Duplicate逐行清零。计算前，针对每一行数据，调用基础API Duplicate对冗余数据位置填充0值。
- 通过Pad一次性清零。计算前，针对多行数据，可以采用高阶API Pad接口对冗余数据一次性清零。

由于搬出时搬运的数据长度和操作数的起始地址（UB上）必须保证32字节对齐，搬出时可以选择去除冗余数据或者带着冗余数据搬出的方式。

- 使用UnPad接口去除冗余数据后搬出。待搬出的有效数据总长度满足32字节对齐时，可使用高阶API UnPad接口去除冗余数据并完整搬出。
- 使用GatherMask收集有效数据后搬出。待搬出的有效数据长度大于等于32字节时，可使用GatherMask重新收集有效数据，保证搬出的有效数据起始地址和数据长度32字节对齐。
- 带冗余数据搬出。注意多核处理时开启原子加（使用SetAtomicAdd接口），防止数据踩踏。

下面分别对上述几种处理方案做详细说明。

- 冗余数据参与计算：如下图所示，对前11个half数据进行Abs计算，冗余数据可以参与计算，不影响最终结果。步骤为：a. 使用DataCopy从Global搬运16个half数据到Local1中，包含冗余数据-11~-15；b. 直接使用Abs做整块计算，不用计算尾块大小，冗余数据参与计算。

[图: 冗余数据参与计算示意图]

- 使用mask掩掉冗余数据：如下图所示，假设输入数据的shape为16*4，将输入数据搬入到UB后每行数据前4个half数据为有效数据，其余为冗余数据。为只对前4个half数据进行ReduceMin计算，可以通过设置mask参数的方法掩掉冗余数据。针对每行数据的处理步骤为：a. 使用DataCopy从Global搬运16个half数据到Local1中；b. 对归约计算的目的操作数Local2清零，如使用Duplicate等；c. 进行归约操作，将ReduceMin的mask模式设置为前4个数据有效，从而掩掉冗余数据。

[图: 使用mask掩掉脏数据示意图]

- 通过Duplicate逐行清零：如下图所示，对于搬入后的非对齐数据，逐行进行Duplicate清零处理，步骤为：a. 使用DataCopy从Global搬运16个half数据到Local中；b. 使用基础API Duplicate，按照如下方式设置mask值，控制仅后5个元素位置有效，将冗余数据填充为0。

```cpp
uint64_t mask0 = ((uint64_t)1 << 16) - ((uint64_t)1 << 11);
uint64_t mask[2] = {mask0, 0};
```

[图: 通过Duplicate逐行清零示意图]

- 通过Pad一次性清零：如下图所示，假设输入数据的shape为16*6，搬入Local后大小为16*16，每行都包含冗余数据，逐行清零性能较差，可以使用Pad一次性清零，步骤为：a. 将16*6的数据从GM上逐行搬入UB后，每行有6个有效数据；b. 使用Pad接口将冗余数据位置填充为0。（对应Pad接口使用场景为：tensor的width已32B对齐，但是有部分冗余数据）。

[图: 通过Pad一次性清零示意图，展示Global(16*6)数据搬入Local1(16*16)后使用Pad清零冗余数据]

- 使用UnPad接口去除冗余数据后搬出：如下图所示，Local内存大小为16*16，每行中只有前6个数为有效数据，要搬出的有效数据16*6满足32B对齐，可以使用UnPad接口去除冗余数据并完整搬出。步骤如下：a. 使用UnPad高阶API去除冗余值；b. 使用DataCopy搬运出连续的16*6个half数据到Global中。

[图: 使用UnPad接口去除冗余数据后搬出示意图]

- 使用GatherMask收集有效数据后搬出：如下图所示，为搬出19个half数据到Global中，有16-18这3个数据的搬运无法满足对齐要求，使用GatherMask对有效数据进行重新收集，收集3-18这16个数据并搬出。步骤如下：a. 完整拷贝前16个half（32B）数据到Global中；b. 使用GatherMask接口，将Local1[3]~[18]的数Gather到Local2中，Local2从对齐地址开始；c. 从Local2中搬运Gather的数据（32B整数倍）到Global中。

[图: 使用GatherMask收集有效数据后搬出示意图]

- 带冗余数据搬出：如下图所示，有4个核参与计算，每个核拷贝出4个数，每个核上拷贝的数据长度不满足32字节对齐，采用将冗余数据一起搬出的方式，步骤如下：a. 将目标Global完整清零，可以通过在host清零或者在kernel侧用UB覆盖的方式处理；b. 将本核内的Local数据，除了要搬出的4个有效数据，其余冗余部分清零（使用Duplicate接口）；c. 使用原子累加的方式拷贝到Global，原子累加结合冗余数据清零，确保不会出现数据踩踏。

[图: 带冗余数据搬出示意图，展示4个核各自搬出数据到Global，使用SetAtomicAdd保证正确性]

样例介绍

- 样例一：冗余数据参与计算+使用GatherMask收集有效数据后搬出。本样例中展示了shape为128*18的tensor进行Abs计算的算子实现。针对每行数据的处理方案如下：搬入后，每行数据的后14个数为冗余数据。Abs接口入参BLOCKLEN_CEIL为32个数，是18个数进行32字节对齐后的结果，有14个冗余数据参与计算。

```cpp
AscendC::Abs(outputLocal, inputLocal, BLOCKLEN_CEIL); // main calculation
```

计算完成后，通过GatherMask的bufPattern入参控制收集18个数中的前16个数。

```cpp
uint16_t tmpValue = 0;
AscendC::Duplicate<uint16_t>(bufPattern, tmpValue, 16);
bufPattern.SetValue(0, 0b1111111111111100); // select the last 14 elements of the first 16 positions
bufPattern.SetValue(1, 0b0000000000000011); // select the first 2 elements of the next 16 positions
uint32_t mask = 32;
uint64_t rsvdCnt = 0;
AscendC::LocalTensor<half> tailLocal = outQueueTail.AllocTensor<half>();
AscendC::GatherMask(tailLocal, outputLocal, bufPattern, true, mask, {1, 1, 8, 8}, rsvdCnt);
```

首先使用DataCopy搬运前16个数，然后搬运后16个数，中间的14个数存在重复搬运。注意：因为DataCopy的目的地址存在重叠所以需要通过PipeBarrier添加流水同步。

```cpp
uint32_t copyLenMain = TILE_LENGTH * sizeof(half) / 32 * 32 / sizeof(half);
uint32_t offsetMain = progress * TILE_LENGTH;
AscendC::DataCopy(dstGlobal[offsetMain], outputLocal, copyLenMain);
AscendC::PipeBarrier<PIPE_MTE3>();
uint32_t tailLen = 32 / sizeof(half);
uint32_t offsetTail = offsetMain + (TILE_LENGTH - tailLen);
AscendC::DataCopy(dstGlobal[offsetTail], tailLocal, tailLen);
```

搬入时要保证32字节对齐，需要将输入的最后一行补齐到32字节对齐，防止访问非法数据。搬出时带冗余数据搬出，输出的最后一行也需要补齐到32字节对齐。main.cpp中对GM上输入输出的长度的定义如下：

```cpp
size_t inputByteSize = 2318 * sizeof(int16_t); // 2318 = 2304 + 32 - 18
size_t outputByteSize = 2304 * sizeof(int16_t);
```

- 样例二：通过Duplicate逐行清零+带冗余数据搬出。本样例中展示了shape为64*11的tensor进行Abs计算的算子实现。共使用4个核，每个核处理16*11个数据。搬入后，每行数据的后5个数为冗余数据。通过Duplicate接口对每行数据中的后5个数据进行清零。

```cpp
// mask mode controls only the last 5 elements doing Duplicate
uint64_t mask0 = (1ul << 16) - (1ul << BLOCK_ELEMENT_NUM);
uint64_t mask[2] = {mask0, 0};
for (int32_t i = 0; i < BLOCK_GROUP_NUM; i++) {
    AscendC::Duplicate<half>(inputLocal[i * BLOCKLEN_CEIL], 0, mask, 1, 1, 1); // clear dummy data on inputLocal
}
AscendC::Abs(outputLocal, inputLocal, BLOCK_GROUP_NUM * BLOCKLEN_CEIL);
```

搬出时，带冗余数据搬出并开启原子累加，BLOCKLEN_CEIL中包含冗余数据。

```cpp
AscendC::SetAtomicAdd<half>();
for (int32_t i = 0; i < BLOCK_GROUP_NUM; i++) {
    AscendC::DataCopy<half>(dstGlobal[i * BLOCK_ELEMENT_NUM], outputLocal[i * BLOCKLEN_CEIL],
        BLOCKLEN_CEIL);
}
AscendC::SetAtomicNone();
```

所以在初始化时，需要对GM数据进行清零，清零代码如下，示例中多个核都调用Fill接口进行清零，需要调用SyncAll进行核间同步。

```cpp
AscendC::Fill<half>(dstGlobal, blockLength, 0);

pipe.InitBuffer(inQueue, BUFFER_NUM, BLOCK_GROUP_NUM * BLOCKLEN_CEIL * sizeof(half));
pipe.InitBuffer(outQueue, BUFFER_NUM, BLOCK_GROUP_NUM * BLOCKLEN_CEIL * sizeof(half));
pipe.InitBuffer(syncLocalTBuf, USE_CORE_NUM * DEFAULT_SYNCALL_NEED_SIZE * sizeof(int32_t));
AscendC::LocalTensor<int32_t> SyncLocal = syncLocalTBuf.Get<int32_t>();
AscendC::SyncAll(syncGlobal, SyncLocal, USE_CORE_NUM);
```

搬入时要保证32字节对齐，需要将输入的最后一行补齐到32字节对齐，防止访问非法数据；搬出时带冗余数据搬出，输出的最后一行也需要补齐到32字节对齐。main.cpp中对GM上输入输出的长度的定义如下：

```cpp
// copy in borrow the next (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM) elements of srcGM
size_t inputByteSize = 709 * sizeof(int16_t);
// copy out atomic add extra (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM) zeros to dstGM
size_t outputByteSize = 709 * sizeof(int16_t);
```

- 样例三：冗余数据参与计算+使用UnPad接口去除冗余数据后搬出。本样例中展示了shape为2048*14的tensor进行Abs计算的算子实现。共使用8个核，每个核处理256*14个数据。搬入后，每行数据的后2个数为冗余数据。Abs接口入参BLOCK_GROUP_NUM * BLOCKLEN_CEIL为连续的16行数据，每行16个数，每行的冗余数据参与计算。

```cpp
AscendC::Abs(inputLocal, inputLocal, BLOCK_GROUP_NUM * BLOCKLEN_CEIL); // main calculation
```

计算后，使用UnPad接口去除冗余数据后再搬出，通过unPadParams.rightPad参数控制去除每行最后的2个冗余数据。

```cpp
unPadParams.rightPad = BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM; // delete 2 dummy half each row
AscendC::UnPad<half>(outputLocal, inputLocal, unPadParams, this->tiling);
```

注意：UnPad接口需要传入tiling参数。abs_unpad_tiling.cpp中关键计算过程如下：

```cpp
AscendC::GetUnPadMaxMinTmpSize(*ascendcPlatform, srcShape, sizeof(int16_t), tmpMaxSize, tmpMinSize);
optiling::UnPadTiling tilingData;
AscendC::UnPadTilingFunc(srcShape, tmpMaxSize, sizeof(int16_t), tilingData);
```

main.cpp中tiling参数需要通过核函数的入参传入到kernel侧，供UnPad高阶API使用。

```cpp
ACLRT_LAUNCH_KERNEL(abs_unpad_custom)(blockDim, stream, xDevice, zDevice, workspaceDevice, tilingDevice);
```

搬入时要保证32字节对齐，需要将输入的最后一行补齐到32字节对齐，防止访问非法数据。main.cpp中对GM上输入的长度的定义如下：

```cpp
// 28674 is TOTAL_LENGTH + (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM)
// 28672 is TOTAL_LENGTH
// copy in borrow the next (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM) elements of srcGM
uint32_t oriLength = 28672;
uint32_t colNum = 14;
uint32_t maxColNum = 32 / sizeof(uint16_t);
uint32_t padLength = oriLength + maxColNum - colNum;
size_t inputByteSize = padLength * sizeof(int16_t);
size_t outputByteSize = oriLength * sizeof(int16_t);
```

- 示例四：通过Pad一次性清零+带冗余数据搬出。本样例中展示了shape为2048*7的tensor进行Abs计算的算子实现。共使用8个核，每个核处理256*7个数据。搬入后，每行数据的后9个数为冗余数据。每个核上通过Pad接口将256*9的冗余数据块整体清零后参与Abs计算。

```cpp
AscendC::PadParams padParams = {0, BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM, 0};
AscendC::Pad(outputLocal, inputLocal, padParams, this->tiling);
AscendC::Abs(outputLocal, outputLocal, BLOCK_GROUP_NUM * BLOCKLEN_CEIL); // main calculation
```

计算后带冗余数据搬出的代码解释和样例二一致。

注意：Pad接口需要传入tiling参数。abs_pad_tiling.cpp中关键计算过程如下：

```cpp
AscendC::GetPadMaxMinTmpSize(srcShape, sizeof(int16_t), tmpMaxSize, tmpMinSize);
optiling::PadTiling tilingData;
AscendC::PadTilingFunc(srcShape, oriSrcShape, tmpMaxSize, sizeof(int16_t), tilingData);
```

main.cpp中tiling参数需要通过核函数的入参传入到kernel侧，供Pad高阶API使用。

```cpp
ACLRT_LAUNCH_KERNEL(abs_pad_custom)(blockDim, stream, xDevice, zDevice, workspaceDevice, tilingDevice);
```

搬入时要保证32字节对齐，需要将输入的最后一行补齐到32字节对齐，防止访问非法数据；搬出时带冗余数据搬出，输出的最后一行也需要补齐到32字节对齐。main.cpp中对GM上输入输出的长度的定义如下：

```cpp
// 14336 is the length of input data
uint32_t oriLength = 14336;
// we must allocate more space to prevent invalid address access
uint32_t padLength = oriLength + shapePad[1] - shapeUsed[1];
size_t inputByteSize = padLength * sizeof(int16_t);
size_t outputByteSize = padLength * sizeof(int16_t);
// however, original length must be used when output to file
size_t outputFileSize = oriLength * sizeof(int16_t);
```

- 样例五：使用mask掩掉冗余数据+带冗余数据搬出。本样例中展示了shape为16*4的tensor每行数据进行ReduceMin计算的算子实现。共使用4个核，每个核处理4*4个数据。搬入后，每行数据的后12个为冗余数据。通过ReduceMin的入参Mask控制只有前4个数参与计算。

```cpp
uint64_t Mask0 = ((uint64_t)1 << BLOCK_ELEMENT_NUM) - 1; // mask mode controls only the first 4 elements do ReduceMin calculation
uint64_t Mask[2] = {Mask0, 0};
// main calculation
for (int i = 0; i < BLOCK_GROUP_NUM; i++) {
    AscendC::ReduceMin<half>(outputLocal[i * BLOCKLEN_CEIL], inputLocal[i * BLOCKLEN_CEIL],
        workLocal, Mask, 1, 8, false);
}
```

计算后带冗余数据搬出的代码解释和样例二一致。

搬入时要保证32字节对齐，需要将输入的最后一行补齐到32字节对齐，防止访问非法数据；搬出时带冗余数据搬出，输出的最后一行也需要补齐到32字节对齐。main.cpp中对GM上输入输出的长度的定义如下：

```cpp
// copy in borrow the next (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM) elements of srcGM
size_t inputByteSize = 76 * sizeof(int16_t);
// copy out atomic add extra (BLOCKLEN_CEIL - BLOCK_ELEMENT_NUM) zeros to dstGM
size_t outputByteSize = 76 * sizeof(int16_t);
```

### 10.1.3 矩阵编程（高阶API）

#### 10.1.3.1 基础知识

说明：本节内容为使用高阶API进行矩阵乘法的编程指导。使用高阶API进行实际的矩阵编程时，需要通过API参考确认接口支持的产品型号。

矩阵乘法概述

Matmul的计算公式为：C = A * B + bias，其示意图如下。

- A、B为源操作数，A为左矩阵，形状为[M, K]；B为右矩阵，形状为[K, N]。
- C为目的操作数，存放矩阵乘结果的矩阵，形状为[M, N]。
- bias为矩阵乘偏置，形状为[1, N]。对A*B结果矩阵的每一行都采用该bias进行偏置。

[图: Matmul矩阵乘示意图，展示A[M,K] x B[K,N] + bias[1,N] = C[M,N]]

矩阵乘法数据流

在了解矩阵乘法数据流之前，需要先回顾一下几个重要的存储逻辑位置的概念：

- 搬入数据的存放位置：A1，用于存放整块A矩阵，可类比CPU多级缓存中的二级缓存；
- 搬入数据的存放位置：B1，用于存放整块B矩阵，可类比CPU多级缓存中的二级缓存；
- 搬入数据的存放位置：C1，用于存放整块的矩阵乘偏置Bias矩阵，可类比CPU多级缓存中的二级缓存；
- 搬入数据的存放位置：A2，用于存放切分后的小块A矩阵，可类比CPU多级缓存中的一级缓存；
- 搬入数据的存放位置：B2，用于存放切分后的小块B矩阵，可类比CPU多级缓存中的一级缓存；
- 搬入数据的存放位置：C2，用于存放切分后的小块矩阵乘偏置Bias矩阵，可类比CPU多级缓存中的一级缓存；
- 结果数据的存放位置：CO1，用于存放小块结果C矩阵，可理解为Cube Out；
- 结果数据的存放位置：CO2，用于存放整块结果C矩阵，可理解为Cube Out；
- 搬入数据的存放位置：VECCALC，一般在计算需要临时变量时使用此位置。

矩阵乘法数据流指矩阵乘的输入输出在各存储位置间的流向。逻辑位置的数据流如下图所示（为了简化描述，没有列出bias）：

- A矩阵从输入位置到A2的数据流如下（输入位置可以是GM或者VECOUT）：GM->A2，GM->A1->A2；VECOUT->A1->A2。由于A1比A2的空间更大，数据从GM或VECOUT可以先搬入A1进行缓存，待该数据执行Cube计算前，数据直接从A1搬入A2，这样在搬运大量数据时可以减少计算前的等待时间，提升性能，只有在搬入数据较少的场景才可能使用GM->A2的数据流。
- B矩阵从输入位置到B2的数据流如下（输入位置可以是GM或者VECOUT）：GM->B2，GM->B1->B2；VECOUT->B1->B2。由于B1比B2的空间更大，数据从GM或VECOUT可以先搬入B1进行缓存，待该数据执行Cube计算前，数据直接从B1搬入B2，这样在搬运大量数据时可以减少计算前的等待时间，提升性能，只有在搬入数据较少的场景才可能使用GM->B2的数据流。
- 完成A2*B2=CO1计算。
- CO1数据汇聚到CO2：CO1->CO2。
- 从CO2到输出位置（输出位置可以是GM或者VECIN）：CO2->GM/CO2->VECIN。

[图: 矩阵乘法数据流示意图，展示GM到A1/B1/C1(BufPool1)到A2/B2/C2(BufPool2)到CO1(BufPool3)到CO2/VECIN/VECOUT(BufPool4/5)的数据流向]

数据格式

在完成Matmul矩阵乘法时，主要涉及到两种分形格式ND和NZ。

- ND：普通格式，N维张量。
- NZ：为满足AICore中Cube计算单元高性能计算的需要，引入该特殊格式。

ND -> NZ的变换过程为：(..., N, H, W)->pad->(..., N, H1*H0, W1*W0)->reshape->(..., N, H1, H0, W1, W0)->transpose->(..., N, W1, H1, H0, W0)

如下图所示（W, H）大小的矩阵被分为（H1*W1）个分形；每个分形内部有（H0*W0）个元素，按照行优先排布，形状如Z字形。所以这种数据格式称为NZ（大N小Z）格式。

[图: ND到NZ格式转换示意图，展示矩阵按列优先分块、块内行优先排布]

下面我们再通过一个具体的例子来深入理解ND和NZ格式的数据排布区别。假设分形格式为2*2，如下图所示4*4的矩阵，ND和NZ格式存储两种情况下，数据在内存中的排布格式分别为：

ND: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15
NZ: 0, 1, 4, 5, 8, 9, 12, 13, 2, 3, 6, 7, 10, 11, 14, 15

关于矩阵ND到NZ格式转换的样例请参考Matmul输入矩阵ND到NZ格式转换的算子样例。

数据分块（Tiling）

- 多核切分

为了实现多核并行，需要将矩阵数据进行切分，分配到不同的核上进行处理。切分策略如下图所示：

对于A矩阵，沿着M轴进行切分，切分成多份的singleCoreM，单核上处理SingleCoreM * K大小的数据。

对于B矩阵，沿着N轴进行切分，切分成多份的singleCoreN，单核上处理K * SingleCoreN大小的数据。

对于C矩阵，SingleCoreM * K大小的A矩阵和K * SingleCoreN大小的B矩阵相乘得到SingleCoreM * SingleCoreN大小的C矩阵，即为单核上输出的C矩阵大小。

比如，下图中共有8个核参与计算，将A矩阵沿着M轴划分为4块，将B矩阵沿着N轴切分为两块，单核上仅处理某一分块（比如图中绿色部分为core3上参与计算的数据）：SingleCoreM * K大小的A矩阵分块和SingleCoreN * K大小的B矩阵分块相乘得到SingleCoreM * SingleCoreN大小的C矩阵分块。

[图: 多核切分示意图，展示8个核的矩阵切分方案，A矩阵M轴切4块、B矩阵N轴切2块]

- 核内切分

大多数情况下，Local Memory的存储，无法完整的容纳算子的输入与输出，需要每次搬运一部分输入进行计算然后搬出，再搬运下一部分输入进行计算，直到得到完整的最终结果，也就是需要做核内的输入切分。切分的策略如下所示：

对于A矩阵，沿M轴进行切分，将singleCoreM切分成多份的baseM，切分成的份数对应图示的mIter；沿K轴进行切分，切分成多份的baseK。

对于B矩阵，沿N轴进行切分，将singleCoreN切分成多份的baseN，切分成的份数对应图示的nIter；沿K轴进行切分，切分成多份的baseK。

对于C矩阵，A矩阵中baseM*baseK大小的分块和B矩阵中baseK*baseN大小的分块相乘并累加，得到C矩阵中对应位置baseM*baseN大小的分块。比如，图中结果矩阵中的绿色矩阵块5是通过如下的累加过程得到的：a*a+b*b+c*c+d*d+e*e+f*f。

[图: 核内切分示意图，展示singleCoreM和singleCoreK被切分为baseM和baseK的过程，以及A1缓存的depthA1和stepKa参数]

除了基本块形状baseM, baseN, baseK外，还有一些常用的tiling参数，其含义如下：

- iterateOrder：一次Iterate迭代计算出[baseM, baseN]大小的C矩阵分片。Iterate完成后，Matmul会自动偏移下一次Iterate输出的C矩阵位置，iterateOrder表示自动偏移的顺序。0代表先往M轴方向偏移再往N轴方向偏移；1代表先往N轴方向偏移再往M轴方向偏移。在上图的示例中，iterateOrder取值为0。
- depthA1，depthB1：A1、B1上存储的矩阵全载A2/B2的份数，A2、B2存储大小分别是baseM * baseK，baseN * baseK，即depthA1是A1矩阵切片含有baseM * baseK块的个数，depthB1是B1矩阵切片含有baseN * baseK块的个数。
- stepM，stepN：stepM为左矩阵在A1中缓存的buffer M方向上baseM的倍数，stepN为右矩阵在B1中缓存的buffer N方向上baseN的倍数。
- stepKa，stepKb：stepKa为左矩阵在A1中缓存的buffer K方向上baseK的倍数，stepKb为右矩阵在B1中缓存的buffer K方向上baseK的倍数。

#### 10.1.3.2 算子实现

实现流程

上文介绍了Matmul矩阵乘的数据切分方案和数据流。Ascend C提供一组Matmul高阶API，封装了这些常用的切分和数据搬运、计算的算法逻辑，方便用户快速实现Matmul矩阵乘法的运算操作。开发者在host侧通过调用API自动获取Tiling参数，该参数传递到kernel侧后，在初始化操作时传入，通过几个简单的API即可完成矩阵乘操作。完整样例请参考LINK。

[图: 矩阵编程流程示意图，展示host侧(创建Tiling对象->设置类型格式->设置shape->设置空间大小->获取Tiling参数)和kernel侧(创建Matmul对象->初始化->设置矩阵->完成矩阵乘操作->结束)]

host侧自动获取Tiling参数的关键步骤介绍如下：

步骤1 创建Tiling对象。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MatmulApiTiling cubeTiling(ascendcPlatform);
```

传入硬件平台信息创建PlatformAscendC对象，然后创建Tiling对象，硬件平台信息可以通过GetPlatformInfo获取。

步骤2 设置A、B、Bias的内存逻辑位置、格式和数据类型。

```cpp
cubeTiling.SetAType(AscendC::TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetBType(AscendC::TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetCType(AscendC::TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT);
cubeTiling.SetBiasType(AscendC::TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT);
```

步骤3 设置矩阵shape信息。

```cpp
cubeTiling.SetShape(M, N, K);
cubeTiling.SetOrgShape(M, N, K); // 设置原始完整的形状M、N、K
```

步骤4 设置可用空间大小信息。

设置Matmul计算时可用的L1 Buffer/L0C Buffer/Unified Buffer空间大小，-1表示AI处理器对应Buffer的大小。

```cpp
cubeTiling.SetBufferSpace(-1, -1, -1);
```

步骤5 按需设置其他参数，比如设置bias参与计算。

```cpp
cubeTiling.EnableBias(true);
```

步骤6 获取Tiling参数。

```cpp
MatmulCustomTilingData tiling;
if (cubeTiling.GetTiling(tiling.cubeTilingData) == -1) {
    return ge::GRAPH_FAILED;
}
```

步骤7 Tiling参数的序列化保存等其他操作。

kernel侧使用Matmul API矩阵乘运算的具体步骤如下：

步骤1 创建Matmul对象

创建Matmul对象的示例如下：

- 纯Cube模式（只有矩阵计算）场景下，需要在代码中定义ASCENDC_CUBE_ONLY宏，本节内容以纯Cube模式举例。
- 默认为MIX模式（包含矩阵计算和矢量计算），该场景下，不能定义ASCENDC_CUBE_ONLY宏，更多内容请参考10.1.5融合算子编程。

```cpp
// 纯Cube模式（只有矩阵计算）场景下，需要设置该代码宏，并且必须在#include "lib/matmul_intf.h"之前设置
#define ASCENDC_CUBE_ONLY
#include "lib/matmul_intf.h"
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> aType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> bType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> cType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> biasType;
AscendC::Matmul<aType, bType, cType, biasType> mm;
```

创建对象时需要传入A、B、C、Bias的参数类型信息，类型信息通过MatmulType来定义，包括：内存逻辑位置、数据格式、数据类型。

步骤2 初始化操作。

```cpp
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling); // 初始化
```

说明：Matmul高阶API内部实现时需要使用系统workspace（即对应本步骤中的GetSysWorkSpacePtr接口），开发者需要自行申请系统workspace的空间：

- 在host侧Tiling实现时，设置总的workspace的数值大小（包含用户workspace和系统workspace），workspace空间由框架来申请并管理。系统workspace的空间大小通过GetLibApiWorkSpaceSize获取。

```cpp
size_t userWorkspaceSize = 0;
size_t systemWorkspaceSize = static_cast<size_t>(ascendcPlatform.GetLibApiWorkSpaceSize());
size_t *currentWorkspace = context->GetWorkspaceSizes(1);
currentWorkspace[0] = userWorkspaceSize + systemWorkspaceSize;
```

- 若算子工程不是自定义算子工程，也不是带有HAVE_WORKSPACE编译宏的Kernel直调算子工程，框架不会自动设置workspace，需要在kernel侧的Matmul初始化前，通过SetSysWorkSpace设置系统workspace。

```cpp
// 使用Matmul时必须设置workspace空间
SetSysWorkspace(workspace);
if (GetSysWorkSpacePtr() == nullptr) {
    return;
}
```

步骤3 设置左矩阵A、右矩阵B、Bias。

```cpp
mm.SetTensorA(gm_a);   // 设置左矩阵A
mm.SetTensorB(gm_b);   // 设置右矩阵B
mm.SetBias(gm_bias);   // 设置Bias
```

步骤4 完成矩阵乘操作。

- 调用Iterate完成单次迭代计算，叠加while循环完成单核全量数据的计算。Iterate方式，可以自行控制迭代次数，完成所需数据量的计算，方式比较灵活。

```cpp
while (mm.Iterate()) {
    mm.GetTensorC(gm_c);
}
```

- 调用IterateAll完成单核上所有数据的计算。IterateAll方式，无需循环迭代，使用比较简单。

```cpp
mm.IterateAll(gm_c);
```

步骤5 结束矩阵乘操作。

```cpp
mm.End();
```

设置Shape信息

在实现Host Tiling时可以设置Shape信息，用于Tiling计算；kernel侧运行时也可以修改部分Shape信息，用于尾块设置、Matmul复用（多个Matmul计算复用一个Matmul对象）等场景。本节对涉及到的Shape概念进行介绍，并给出host侧和kernel侧设置Tiling信息的指导。

- orgShape：M、N、K
- singleCoreShape：singleCoreM、singleCoreN、singleCoreK
- singleShape：singleM、singleN、singleK
- baseShape：baseM、baseN、baseK

通过数据分块（Tiling）的介绍我们已经了解了orgShape(M、N、K)，singleCoreShape(singleCoreM、singleCoreN、singleCoreK)，baseShape(baseM、baseN、baseK)的概念。

[图: Shape层次关系示意图，展示orgShape到singleCoreShape到singleShape到baseShape的切分关系]

除此之外，单核的Matmul Tiling时，实际参与Matmul计算的shape可以是原始shape中的一部分，singleM, singleN, singleK用于表达实际参与Matmul计算的shape，如下图所示。在单核的情况下，singleM, singleN, singleK会透传给singleCoreM, singleCoreN, singleCoreK。

[图: singleShape与singleCoreShape关系示意图]

- Kernel运行时设置
  - SetTail、SetSingleShape都是运行时修改singleCoreM、singleCoreN、singleCoreK，处理尾块时使用SetTail，Matmul复用（多个Matmul计算复用一个Matmul对象）的场景可以使用SetSingleShape重新设置。
  - SetOrgShape是运行时修改M、N、K，Matmul复用的场景可以使用SetOrgShape重新设置。

- 单核Tiling时设置
  - SetOrgShape（必选）：设置M、N、K
  - SetShape（非必选）：设置singleM、singleN、singleK，等同于设置singleCoreM、singleCoreN、singleCoreK
  - SetFixSplit（非必选）：设置baseM、baseN、baseK

- 多核Tiling时设置
  - SetOrgShape（必选）：设置M、N、K
  - SetShape（非必选）：设置singleM、singleN、singleK
  - SetFixSplit（非必选）：设置baseM、baseN、baseK
  - SetSingleShape(非必选)：设置singleCoreM、singleCoreN、singleCoreK
  - SetSingleRange(非必选)：设置singleCoreM、singleCoreN、singleCoreK的范围

设置format格式

创建Matmul对象时需要传入A、B、C、Bias的参数类型信息，类型信息通过MatmulType来定义，包括：内存逻辑位置、数据格式、数据类型。示例如下：

```cpp
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> aType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> bType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> cType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> biasType;
AscendC::Matmul<aType, bType, cType, biasType> mm;
```

针对数据格式，包括CubeFormat::ND, CubeFormat::NZ, CubeFormat::ND_ALIGN三种，ND和NZ格式在数据格式章节已经介绍，ND_ALIGN格式的介绍请参考数据排布格式。

#### 10.1.3.3 特性场景

##### 10.1.3.3.1 Matmul特性介绍

除了前述基础知识和算子实现中介绍的基本计算能力外，Matmul矩阵编程还提供了适用于不同场景的处理能力及多种功能，具体场景和功能列于下表中，详细内容请见后续章节。

表 10-4 Matmul特性表

| 特性分类 | 特性描述 | 功能简介 |
|------|------|------|
| 功能实现 | 多核对齐切分 | 在多核场景中，支持将矩阵数据沿M、N、K轴切分，满足M能被singleCoreM整除、N能被singleCoreN整除、K能被singleCoreK整除的对齐场景时的处理方式，从而实现多核并行计算矩阵乘。 |
| | 多核非对齐切分 | 在多核场景中，支持将矩阵数据沿M、N、K轴切分。当出现M不能被singleCoreM整除、或N不能被singleCoreN整除、或K不能被singleCoreK整除的非对齐场景（即尾块场景）时的处理方式。 |
| | 异步场景处理 | MIX场景（包含矩阵计算和矢量计算）下不需要等待矩阵乘计算完成，先执行其它计算。 |
| | 自定义数据搬入搬出 | 自定义矩阵乘计算前后的数据搬运函数。本功能支持用户实现左矩阵A、右矩阵B从Global Memory分别自定义搬入到A1、B1的过程，输出C矩阵从CO1自定义搬出到Global Memory的过程。 |
| | 矩阵乘输出的量化/反量化 | 将矩阵乘的计算结果从CO1搬出到Global Memory时，对矩阵元素执行数据量化或反量化操作。 |
| | 矩阵乘输出的Channel拆分 | 矩阵乘输出的Channel拆分，又称ChannelSplit。指float数据类型、NZ数据格式的输出C矩阵按照16*8的分形大小存储。 |
| | 矩阵向量乘 | 矩阵向量乘即GEMV，指矩阵乘计算中M=1，K>1的场景，即对形状为(1, K)的左矩阵A实现矩阵乘计算。 |
| | 4:2稀疏矩阵乘 | 4:2稀疏矩阵乘，又称Sparse Matmul。指对稀疏左矩阵A和4:2稠密化的右矩阵B实现矩阵乘计算。 |
| | 上/下三角矩阵乘 | 忽略位于矩阵中下三角或上三角位置的元素的计算，实现矩阵中上三角或下三角位置的元素的矩阵乘计算。 |
| | TSCM输入的矩阵乘 | 对内存逻辑位置为TSCM的左矩阵A或右矩阵B实现矩阵乘计算。 |
| | 矩阵乘输出的N方向对齐 | 矩阵乘输出的N方向对齐，又称ND_ALIGN格式输出。指对数据格式为ND_ALIGN的输出C矩阵实现N方向的自动补齐及输出。 |
| | 单次矩阵乘局部输出 | 单次矩阵乘局部输出，又称Partial Output，指矩阵乘计算时不对单核K方向的计算结果做累加，直接输出计算结果。 |
| | AIC和AIV独立运行机制 | AIC和AIV独立运行机制，又称双主模式。MIX场景（包含矩阵计算和矢量计算）下AIC核和AIV核独立运行代码，不依赖消息驱动。 |

表 10-5 BatchMatmul特性表

| 特性分类 | 特性描述 | 功能简介 |
|------|------|------|
| 功能实现 | Batch Matmul基础功能 | Batch Matmul基础功能，支持批量处理Matmul，调用一次IterateBatch接口，计算出多个singleCoreM * singleCoreN大小的C矩阵。 |
| | Batch Matmul复用Bias矩阵 | 每个Batch的Matmul计算复用同一个不带Batch轴的Bias矩阵。 |

##### 10.1.3.3.2 多核对齐切分

为了实现多核并行，提升计算效率，需要将矩阵数据进行切分，分配到不同的核上进行处理。主要的切分策略有切分K轴和不切分K轴两种。

不切分K轴、仅切分M、N轴的策略如下：

- 对于A矩阵，沿着M轴进行切分，切分成多份的singleCoreM，单核上处理SingleCoreM * K大小的数据。
- 对于B矩阵，沿着N轴进行切分，切分成多份的singleCoreN，单核上处理K * SingleCoreN大小的数据。
- 对于C矩阵，SingleCoreM * K大小的A矩阵和K * SingleCoreN大小的B矩阵相乘得到SingleCoreM * SingleCoreN大小的C矩阵，即为单核上输出的C矩阵大小。

比如，下图共有8个核参与计算，将A矩阵沿着M轴划分为4块，将B矩阵沿着N轴切分为两块，单核上仅处理某一分块（比如图中绿色部分为core5上参与计算的数据）：SingleCoreM * K大小的A矩阵分块和SingleCoreN * K大小的B矩阵分块相乘得到SingleCoreM * SingleCoreN大小的C矩阵分块。

[图: 多核对齐切分(不切K轴)示意图]

切分M、N、K轴的策略如下图所示：

- 对于A矩阵，沿着M轴进行切分，切分成多份的singleCoreM，沿着K轴切分，切分成多份的singleCoreK，单核上处理singleCoreM * singleCoreK大小的数据。
- 对于B矩阵，沿着K轴进行切分，切分成多份的singleCoreK，沿着N轴进行切分，切分成多份的singleCoreN，单核上处理singleCoreK * singleCoreN大小的数据。
- 对于C矩阵，singleCoreM * singleCoreK大小的A矩阵与singleCoreK * singleCoreN大小的B矩阵相乘并累加得到singleCoreM * singleCoreN大小的C矩阵分块。

比如下图中，C矩阵中的R矩阵块，是通过A1*B1+A2*B2+A3*B3累加得到的，其中，A1*B1、A2*B2、A3*B3可在多个核上并行计算。

[图: 多核对齐切分(切K轴)示意图]

上述的切分策略会在Tiling参数中体现，比如SingleCoreM、SingleCoreN、SingleCoreK，开发者在host侧通过调用API自动获取Tiling参数，与单核场景的不同是，多核Tiling需要使用MultiCoreMatmulTiling构造多核Tiling对象，并通过SetDim接口设置Matmul计算所用的核数。注意：这里设置的核数为Matmul计算可用的核数，仅在多核场景下设置，用于计算tiling参数；SetBlockDim为整个算子计算所用核数，是实际加载的核数，是必须设置的。SetBlockDim的设置规则请参考blockDim的说明。SetDim的设置规则如下：

- 纯Cube模式（只有矩阵计算）场景，本节内容以纯Cube模式举例。SetDim设置当前AI处理器可用的核数，通过Tiling计算得到执行Matmul计算实际使用的核数。SetBlockDim按照实际使用的核数由用户进行配置。

- MIX模式（包含矩阵计算和矢量计算）的设置规则请参考MIX场景核数设置规则。

使用场景：多核处理Matmul矩阵计算场景。

约束说明：无

调用示例

该场景的关键代码示例如下。Matmul多核对齐场景的完整样例请参考：多核切M、N的样例：Matmul多核Kernel直调样例；多核切K的样例：多核切K场景的算子样例。

```cpp
// 构造多核Tiling对象
auto ascendcPlatform = platform_ascendc::PlatformAscendCManager::GetInstance(socVersion);
matmul_tiling::MultiCoreMatmulTiling cubeTiling(*ascendcPlatform);
// 仅包含Cube计算的算子，设置可参与矩阵乘运算的核数为当前AI处理器上的Cube核数
cubeTiling.SetDim(ascendcPlatform.GetCoreNumAic());

cubeTiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
cubeTiling.SetBiasType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
cubeTiling.SetOrgShape(M, N, K);
cubeTiling.SetShape(M, N, K);
cubeTiling.EnableBias(isBias);
optiling::TCubeTiling tilingData;
// 获取Tiling参数
int ret = cubeTiling.GetTiling(tilingData);   // if ret = -1, gen tiling failed
```

##### 10.1.3.3.3 多核非对齐切分

功能介绍

多核场景，对矩阵进行切分时，若M、N、K无法整除singleCoreM、singleCoreN、singleCoreK时，就会出现尾块，即多核非对齐场景。如下图矩阵A、B的最后一行和最后一列的矩阵块：

[图: 多核非对齐切分示意图，展示矩阵A和B的tailM、tailK、tailN尾块]

此时，C矩阵中的R矩阵块，依然是通过A1*B1+A2*B2+A3*B3+A4*B4累加得到的，处理A1*B1、A2*B2、A3*B3、A4*B4等尾块时，需要在kernel侧设置尾块大小，在不改变原有tiling的情况下，调用SetTail接口重新设置本次计算的singleCoreM/singleCoreN/singleCoreK，在处理尾块的时候按照设置的值也就是tailM/tailN/tailK进行搬运和计算。

使用场景：多核处理Matmul矩阵计算，存在尾块的场景。

约束说明：处理尾块调用的SetTail接口，需要在Iterate/IterateAll之前调用。

调用示例

Matmul多核非对齐场景的完整样例请参考Matmul多核非对齐切分算子样例。该场景的关键代码示例如下：

```cpp
// 处理尾块
int tailM = tiling.M - mCoreIndex * tiling.singleCoreM;
tailM = tailM < tiling.singleCoreM ? tailM : tiling.singleCoreM;
int tailN = tiling.N - nCoreIndex * tiling.singleCoreN;
tailN = tailN < tiling.singleCoreN ? tailN : tiling.singleCoreN;
// 当tailM < singleCoreM 或 tailN < singleCoreN时被认为需要处理尾块，此时可以调用SetTail进行设置
if (tailM < tiling.singleCoreM || tailN < tiling.singleCoreN) {
    matmulObj.SetTail(tailM, tailN);
}
```

##### 10.1.3.3.4 异步场景处理

功能介绍

Matmul的Iterate和IterateAll接口在MIX场景（包含矩阵计算和矢量计算）下提供了同步和异步两种模式，纯Cube场景（只有矩阵计算）下，只支持同步模式。

同步模式指的是程序执行时，需要等待某个操作完成后才能继续执行下一步操作。异步模式指的是程序执行时，不需要等待某个操作完成就可以继续执行下一步操作。

- Iterate&GetTensorC的同步和异步
  - 同步：执行完一次Iterate迭代计算后，执行GetTensorC搬运矩阵C分片，搬运完成后，才能进行下一次计算。如下图所示，C矩阵中，矩阵块1搬走后，才能计算矩阵块2，矩阵块2搬运完成后，才能计算矩阵块3。

Iterate&GetTensorC同步模式的关键代码示例如下：

```cpp
while (mm.Iterate()) {
    mm.GetTensorC(gm_c);
}
```

  - 异步：通过设置Iterate接口的模板参数开启异步模式。调用Iterate后，无需立即调用GetTensorC同步等待矩阵C分块搬运完成，可以先执行其它操作，待需要获取结果时再调用GetTensorC。异步模式可以减少同步等待，提高并行度，开发者对计算性能要求较高时，可以选用该方式。异步场景时，需要使用一块临时空间来缓存Iterate计算结果，否则会覆盖计算结果，调用GetTensorC时会在该临时空间中获取C的矩阵分片。临时空间通过SetWorkspace接口进行设置。SetWorkspace接口需要在Iterate接口之前调用。

Iterate&GetTensorC异步模式的关键代码示例如下：

```cpp
mm.SetWorkspace(workspace, size); // 其中，workspace为临时空间的物理地址，size为singleCoreM * singleCoreN的矩阵C大小
// 异步模式
mm.template Iterate<false>();
...... // 执行其他操作
auto mIter = Ceil(singleCoreM, baseM);
auto nIter = Ceil(singleCoreN, baseN);
for (int i = 0; i < mIter * nIter; ++i) {
    mm.GetTensorC<false>(gm_c);
}
```

- IterateAll的同步和异步
  - 同步：后续操作需要同步等待IterateAll执行结束。

IterateAll同步模式的关键代码示例如下：

```cpp
mm.SetTensorA(gm_a);   // 设置左矩阵A
mm.SetTensorB(gm_b);   // 设置右矩阵B
mm.SetBias(gm_bias);   // 设置Bias
mm.IterateAll(gm_c);
// 后续操作
...
```

  - 异步：后续操作不需要同步等待IterateAll执行结束，需要IterateAll的结果时，调用WaitIterateAll等待IterateAll异步接口返回。

IterateAll异步模式的关键代码示例如下：

```cpp
AscendC::Matmul<aType, bType, cType, biasType> mm;
mm.SetTensorA(queryGm[tensorACoreOffset]);
mm.SetTensorB(keyGm[tensorBCoreOffset + sInnerStart * singleProcessSInnerSize *
    tilingData->attentionScoreOffsetStrideParams.matmulHead], true);
mm.SetTail(singleProcessSOuuteSize, mmNNum);
mm.template IterateAll<false>(workspaceGm[tmp_block_idx * mmResUbSize *
    sInnerLoopTimes],0, false, true);
// 执行其他操作
mm.WaitIterateAll(); // 等待IterateAll完成
DataCopy(dstUB, GM);  // 进行GM到UB的拷贝
```

使用场景

- Iterate&GetTensorC的同步：MIX场景（包含矩阵计算和矢量计算）、纯Cube场景（只有矩阵计算）。
- Iterate&GetTensorC的异步：仅MIX场景（包含矩阵计算和矢量计算）。
- IterateAll的同步：MIX场景（包含矩阵计算和矢量计算）、纯Cube场景（只有矩阵计算）。
- IterateAll的异步：仅MIX场景（包含矩阵计算和矢量计算）。

约束说明

- Iterate&GetTensorC的异步场景：
  - 传入的C矩阵地址空间大小需要保证不小于baseM * baseN。
  - SetWorkspace接口需要在Iterate接口之前调用。
  - 支持只输出到VECIN、只输出到Global Memory，同时输出到Global Memory和VECIN三种输出方式。
  - 取出C矩阵到VECIN时，数据格式仅支持NZ；取出C矩阵到GM时，数据格式支持ND或NZ。

- IterateAll的异步场景：
  - 传入的C矩阵地址空间大小需要保证不小于singleCoreM * singleCoreN。
  - 仅支持连续输出至Global Memory。

调用示例

- Iterate&GetTensorC的异步场景的完整样例请参考异步场景样例、Iterate异步场景矩阵乘法。
- IterateAll的异步场景的完整样例请参考IterateAll异步场景矩阵乘法。

##### 10.1.3.3.5 矩阵乘输出的量化/反量化

功能介绍

对于特定输入输出数据类型，Matmul支持将计算结果从CO1搬出到Global Memory时，对输出C矩阵元素执行数据量化或反量化操作。

- Matmul量化场景：Matmul计算时左矩阵A、右矩阵B为half或bfloat16_t数据类型，输出C矩阵为int8_t数据类型。该场景下，C矩阵的数据从CO1搬出到Global Memory时，会执行量化操作，将最终结果量化为int8_t类型。

[图: Matmul量化场景示意图，展示half输入经MTE2搬入A1/B1，MTE1搬入A2/B2，Cube计算得CO1(float输出)，经FixPipe随路量化得int8_t输出到GM]

- Matmul反量化场景：Matmul计算时左矩阵A、右矩阵B为int8_t或int4b_t数据类型，输出C矩阵为half数据类型，或者左矩阵A、右矩阵B为int8_t数据类型，输出C矩阵为int8_t数据类型。该场景下，C矩阵的数据从CO1搬出到Global Memory时，会执行反量化操作，将最终结果反量化为对应的half类型或int8_t类型。

[图: Matmul反量化场景示意图，展示int8_t输入经搬运和Cube计算得CO1(int32_t输出)，经FixPipe随路反量化得half输出到GM]

Matmul量化/反量化包含两种模式：同一系数的量化/反量化模式、向量的量化/反量化模式，开发者在算子Tiling侧调用SetDequantType接口设置量化或反量化模式，这两种模式的具体区别为：

- 同一系数的量化/反量化模式（PER_TENSOR模式）：整个C矩阵对应一个量化参数，量化参数的shape为[1]。开发者在算子Kernel侧调用接口SetQuantScalar设置量化参数。
- 向量的量化/反量化模式（PER_CHANNEL模式）：C矩阵的shape为[m, n]，每个channel维度即C矩阵的每一列，对应一个量化参数，量化参数的shape为[n]。开发者在算子Kernel侧调用接口SetQuantVector设置量化参数。

表 10-6 量化/反量化模式对应的接口配置

| 模式 | Tiling侧接口 | Kernel侧接口 |
|------|------|------|
| 同一系数的量化/反量化 | SetDequantType(DequantType::SCALAR) | SetQuantScalar(gmScalar) |
| 向量的量化/反量化 | SetDequantType(DequantType::TENSOR) | SetQuantVector(gmTensor) |

使用场景

需要对矩阵计算结果进行量化/反量化操作的场景下，Matmul输入输出矩阵支持的数据类型如下表所示。

表 10-7 Matmul量化/反量化支持的数据类型

| A矩阵 | B矩阵 | C矩阵 | 支持平台 |
|------|------|------|------|
| half | half | int8_t | Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品 |
| bfloat16_t | bfloat16_t | int8_t | Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品 |
| int8_t | int8_t | half | Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品 |
| int4b_t | int4b_t | half | Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品 |
| int8_t | int8_t | int8_t | Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品 |

约束说明

- SetQuantScalar和SetQuantVector接口必须在Iterate或者IterateAll接口前调用。
- 在Kernel侧与Tiling侧设置的量化/反量化模式需要保持一致：Kernel侧调用SetQuantScalar接口设置同一系数的量化/反量化模式，对应Tiling侧调用SetDequantType接口配置模式为DequantType::SCALAR。Kernel侧调用SetQuantVector接口设置向量的量化/反量化模式，对应Tiling侧调用SetDequantType接口配置模式为DequantType::TENSOR。
- 当A、B矩阵为int8_t或int4b_t类型，C矩阵为half时，本特性的输出结果不支持INF_NAN模式。若结果需要以INF_NAN输出，建议在调用Matmul API时将结果输出到TPosition::VECIN，同时将输出的数据类型设置为int32_t，再基于AIV核使用高阶API AscendDequant将该结果反量化为half类型。

调用示例

完整的算子样例请参考matmul_quant算子样例。

- Tiling实现

调用SetDequantType接口设置量化或反量化模式，其他实现内容与基础场景相同。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MatmulApiTiling tiling(ascendcPlatform);
tiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT8);
tiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT8);
tiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT32);
tiling.SetBiasType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT32);
tiling.SetShape(M, N, K);
tiling.SetOrgShape(M, N, K);
tiling.EnableBias(true);
tiling.SetDequantType(DequantType::SCALAR); // 设置同一系数的量化/反量化模式
// tiling.SetDequantType(DequantType::TENSOR); // 设置向量的量化/反量化模式
... // 执行其他配置
```

- Kernel实现

根据具体量化模式场景，调用SetQuantScalar或SetQuantVector接口设置量化参数。其他实现内容与基础场景相同。

  - 同一系数的量化/反量化模式

```cpp
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling);
float tmp = 0.1; // 输出gm时会乘以0.1
uint64_t ans = static_cast<uint64_t>(*reinterpret_cast<int32_t*>(&tmp)); // 浮点值量化系数转换为uint64_t类型进行设置
mm.SetQuantScalar(ans);
mm.SetTensorA(gm_a);
mm.SetTensorB(gm_b);
mm.SetBias(gm_bias);
mm.IterateAll(gm_c);
```

  - 向量的量化/反量化模式

```cpp
GlobalTensor gmQuant;
...
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling);
mm.SetQuantVector(gmQuant);
mm.SetTensorA(gm_a);
mm.SetTensorB(gm_b);
mm.SetBias(gm_bias);
mm.IterateAll(gm_c);
```

##### 10.1.3.3.6 矩阵乘输出的Channel拆分

功能介绍

矩阵乘输出的Channel拆分，又称ChannelSplit。指当Matmul计算结果C矩阵的格式为NZ时，C矩阵采用分形存储，关于NZ格式的详细内容请参考数据格式。当C矩阵的物理排布格式为NZ、数据类型为float时，默认情况下，每个分形内部包含16*16个元素，即分形的大小为16*16。ChannelSplit的功能为将此场景下C矩阵的每个16*16的分形切分为16*8的分形，使得C矩阵按照16*8的分形进行存储。

由于1个float类型数据的大小为4字节，16*8的分形在内轴满足32字节对齐，内轴上的数据量与一条NPU矢量计算指令处理的数据单元一致，这便于后续的其它计算。ChannelSplit功能默认不启用，用户需通过设置MatmulConfig中的isEnableChannelSplit参数为true来开启此功能。

[图: ChannelSplit功能示意图，展示非ChannelSplit时矩阵C为NZ(16,16)分形，ChannelSplit后为NZ(16,8)分形]

使用场景：对于NZ格式、float类型的C矩阵，需要按16*8的分形存储时，使用该功能。

约束说明：开启ChannelSplit功能需满足：C矩阵的数据排布格式为CubeFormat::NZ。C矩阵的数据类型为float。C矩阵的内存逻辑位置为Global Memory。

调用示例

完整的算子样例请参考matmul_channelsplit算子样例。

```cpp
// 指定获取和修改的MatmulConfig模板
constexpr static MatmulConfigMode configMode = MatmulConfig::CONFIG_NORM;
// 修改模板参数isEnableChannelSplit=true，开启该MatmulConfig模板的ChannelSplit功能
constexpr static MatmulFuncParams funcParamsChannelSplit{
    false, false, false, false, 0, IterateOrder::ORDER_M, ScheduleType::INNER_PRODUCT, true, false, false,
    false, true/*isEnableChannelSplit*/
};
constexpr static MatmulConfig MM_CFG = GetMMConfig<configMode>(funcParamsChannelSplit);
Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, MM_CFG> mm;

// 常规Matmul计算，最后输出分形为16*8
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm);
mm.SetTensorA(gm_a);
mm.SetTensorB(gm_b);
mm.SetBias(gm_bias);
mm.IterateAll(gm_c);
```

##### 10.1.3.3.7 矩阵向量乘

功能介绍

矩阵向量乘（General Matrix-Vector multiplication），即GEMV，是指Matmul计算中M=1，形状为(1, K)的左矩阵A与形状为(K, N)的右矩阵B进行矩阵乘运算的场景。Matmul支持在Tiling侧与Kernel侧通过配置A矩阵的数据格式为VECTOR来开启GEMV模式，从而高效处理M=1的计算场景。若在M=1时未开启GEMV模式，Matmul计算则将M方向作为非对齐场景进行处理。GEMV模式相较于非对齐处理方式，搬运数据量更少，性能更好。

以M=1，K=256，N=32，左右矩阵数据类型为half的Matmul为具体示例，说明GEMV模式的Matmul API内部处理过程。

- GEMV模式：将A矩阵从A1搬运到A2时，1*256的向量被当作16*16的矩阵进行处理，调用LoadData接口一次完成16*16分形大小的矩阵搬运。B矩阵的搬运以及矩阵乘计算跟基础场景相同。

[图: GEMV模式M=1的矩阵乘计算示意图]

- 非GEMV模式：将A矩阵从A1搬运到A2时，1*256的向量被当作非对齐矩阵数据进行处理，将M方向对齐到32字节后进行搬运。调用LoadData接口每次搬运16*16分形大小的矩阵，一共搬运K/16=16次，导致搬运数据量增加，性能相较于GEMV模式差。

[图: 非GEMV模式M=1的矩阵乘计算示意图]

使用场景：形状为(1, K)的A矩阵（M=1，K>1）做矩阵乘计算，即输入A矩阵的数据是向量数据。

约束说明：

- 在Matmul计算中，若要开启GEMV模式，A矩阵的原始输入形状M必须等于1。
- GEMV场景下，左矩阵A不支持转置。

调用示例

完整的算子样例请参考matmul_gemv算子样例。

- Tiling实现

调用SetAType接口，设置A矩阵的数据格式为CubeFormat::VECTOR，其它Tiling实现与基础场景相同。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MatmulApiTiling tiling(ascendcPlatform);
// 调用设置A矩阵的格式为CubeFormat::VECTOR
tiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::VECTOR,
    matmul_tiling::DataType::DT_FLOAT16);
tiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
tiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
tiling.SetBiasType(AscendC::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
... // 其他实现内容
optiling::TCubeTiling tilingData;
int ret = tiling.GetTiling(tilingData);
```

- Kernel实现

相较于基础场景，GEMV场景在创建Matmul对象时，设置模板参数A_TYPE的数据格式为CubeFormat::VECTOR。

```cpp
#include "lib/matmul_intf.h"

using A_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::VECTOR, half>;
using B_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half>;
using C_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float>;
using BIAS_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float>;
AscendC::Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE> mm;
```

##### 10.1.3.3.8 4:2稀疏矩阵乘

功能介绍

4:2稀疏矩阵乘，又称Sparse Matmul。该场景下输入的原始左矩阵A、右矩阵B为稀疏矩阵，稀疏矩阵B中每4个元素中至少有2个为零元素；在进行Matmul计算前，用户需要自行对B矩阵进行4:2稠密化，即基于原始稀疏矩阵B在每4个元素中过滤掉2个零元素，使B矩阵稠密化为稠密矩阵；Sparse Matmul场景调用Matmul API完成A矩阵与4:2稠密化后的B矩阵的矩阵乘计算。Sparse Matmul可以跳过稀疏矩阵B中的零元素，仅对非零元素进行数据搬运存储和计算，从而减少矩阵乘计算时的内存占用和计算量，提升性能。

实现流程

步骤1 数据预处理

在计算前的数据准备阶段，用户自行对原始为稀疏的B矩阵完成稠密化，稠密过程请参考稠密算法说明。稠密化过程结束后，得到4:2稠密化后的右矩阵B和索引矩阵index，稠密化后的右矩阵B和索引矩阵index将作为Sparse Matmul场景的计算输入。

[图: 对原始稀疏矩阵B进行4:2稠密化过程示意图]

稠密化过程对于稀疏矩阵B的每4个元素，在索引矩阵index中生成2个2位索引，每个索引分别指向对应非零元素的相对位置，具体规则可参考稠密算法说明。稠密化过程生成的索引矩阵的数据类型为int2，索引矩阵在加载入Matmul前，需要拼成int8的数据类型。索引矩阵在一个int8的地址中的排布是逆序排布的，例如：索引矩阵1 0 2 1 0 2 1 0，其中1 0 2 1（对应索引矩阵前四位1201）为一个int8，0 1 2 0（对应索引矩阵后四位0210）为一个int8。

步骤2 使能Sparse Matmul场景

在Host侧，获取Tiling前需要通过SetSparse接口设置使能Sparse Matmul场景。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MatmulApiTiling tiling(ascendcPlatform);
tiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT8);
tiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT8);
tiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT32);
tiling.SetBiasType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_INT32);
// 设置使能Sparse Matmul场景
tiling.SetSparse(true);
... // 其他实现内容
optiling::TCubeTiling tilingData;
int ret = tiling.GetTiling(tilingData);
```

步骤3 创建Matmul对象

在Kernel侧创建Matmul对象时，通过MatmulType定义A、C、Bias的参数类型信息，包括：内存逻辑位置、数据格式、数据类型。通过SparseMatmulType类型定义B矩阵的参数类型，包括：B矩阵的内存逻辑位置、索引矩阵的内存逻辑位置、数据格式、数据类型等。

```cpp
#include "lib/matmul_intf.h"

using A_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, ATYPE, false>;
// 使用SparseMatmulType定义B矩阵的参数类型信息
using B_TYPE = AscendC::SparseMatmulType<AscendC::TPosition::GM, AscendC::TPosition::GM,
    CubeFormat::ND, BType, true>;
using C_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, CType>;
using BIAS_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, BiasType>;
AscendC::Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, CFG_MDL> mm;
```

步骤4 设置索引矩阵

通过SetSparseIndex接口传入稠密化过程中生成的索引矩阵。

```cpp
mm.SetTensorA(gm_a);   // 设置左矩阵A
mm.SetTensorB(gm_b);   // 设置右矩阵B
mm.SetSparseIndex(gm_index); // 传入稠密化过程中生成的索引矩阵
mm.SetBias(gm_bias);   // 设置Bias
```

步骤5 完成矩阵乘操作

在Kernel侧，基于步骤4加载的索引矩阵，完成矩阵乘操作。Matmul API内部完成对A矩阵的稠密化，即根据索引矩阵从A矩阵的每4个元素中，选择2个对应位置元素参与计算。

```cpp
// 调用Iterate和GetTensorC或IterateAll接口完成矩阵乘计算
while (mm.Iterate()) {
    mm.GetTensorC(gm_c);
}
// mm.IterateAll(gm_c);
mm.End();
```

参数说明

表 10-8 SparseMatmulType类型参数说明

| 参数 | 说明 |
|------|------|
| POSITION | 内存逻辑位置。B矩阵仅支持设置为TPosition::GM。 |
| INDEX_POSITION | 索引矩阵内存逻辑位置。仅支持设置为TPosition::GM。 |
| CubeFormat | 数据的物理排布格式。B矩阵支持设置为CubeFormat::ND，CubeFormat::NZ。 |
| TYPE | B矩阵仅支持设置为int8_t数据类型。 |
| ISTRANS | 是否开启使能矩阵转置的功能。当前只支持取值为true，表示开启使能矩阵转置的功能。 |
| LAYOUT | 表征数据的排布。Sparse Matmul场景仅支持取值为LAYOUT::NONE。NONE：默认值，表示不使用BatchMatmul。 |
| IBSHARE | 是否使能IBShare（IntraBlock Share）。IBShare的功能是能够复用L1 Buffer上相同的A矩阵或B矩阵数据。当A矩阵和B矩阵同时使能IBShare时，表示L1 Buffer上的A矩阵和B矩阵同时复用。Sparse Matmul场景当前仅支持该参数取值为false，表示不使能IBShare。 |

使用场景：左矩阵A为稀疏矩阵、右矩阵B为4:2稠密化后的矩阵的Matmul计算场景。

约束说明：

- 该场景仅支持MDL模板下的纯Cube模式（只有矩阵计算）。
- 通过SetSparseIndex接口传入的索引矩阵只支持int8数据类型和NZ数据排布格式。
- 原始稀疏矩阵B中每4个元素中应保证最多2个非零元素（即最少2个零元素），如果存在3个或更多非零元素，则仅使用前2个非零元素。
- M、K、N中的任意一个值不能为0。

调用示例：Sparse Matmul场景的完整样例请参考Sparse Matmul场景的算子样例。

##### 10.1.3.3.9 TSCM输入的矩阵乘

功能介绍

TSCM表示L1 Buffer空间对应的逻辑内存，L1 Buffer相关内容见存储单元，开发者可以自行管理TSCM以高效利用硬件资源。比如，开发者可缓存一份TSCM数据，在不同使用场景中灵活配置为Matmul操作的A矩阵、B矩阵或Bias偏置矩阵，实现内存复用与计算效率优化。在TSCM输入场景，用户管理整块TSCM内存空间，Matmul直接使用传入的TSCM内存地址，不进行Global Memory到TSCM的数据搬入。

使用场景：用户需要自定义数据搬入到TSCM及自定义管理的场景，即需要自定义实现数据搬入功能，如非连续搬入或对搬入数据进行预处理等。用户通过自定义管理TSCM可灵活配置MTE2流水，实现跨Matmul对象的全局DoubleBuffer，MTE2相关内容见搬运单元。

约束说明：设置为TSCM输入的矩阵必须在TSCM中全载，全载即全部的矩阵数据同时搬入及保持在TSCM中。

调用示例

完整的算子样例请参考Matmul自定义TSCM输入的算子样例、BatchMatmul自定义TSCM输入的算子样例。

```cpp
TQue<TPosition::A1, 1> scm; // 队列逻辑位置A1，队列深度为1
pipe->InitBuffer(scm, 1, tiling.M * tiling.Ka * sizeof(A_T));
// A_TYPE的TPosition为TSCM，B_TYPE的TPosition为GM
Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE> mm1;
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm1);
mm1.Init(&tiling);
// 自定义A矩阵GM到TSCM的搬运
auto scmTensor = scm.AllocTensor<A_T>();
DataCopy(scmTensor, gm_a, tiling.M * tiling.Ka);
scm.EnQue(scmTensor);
LocalTensor<A_T> scmLocal = scm.DeQue<A_T>();
// A矩阵设置为TSCM输入，B矩阵为GM输入
mm1.SetTensorA(scmLocal);
mm1.SetTensorB(gm_b);
mm1.SetBias(gm_bias);
mm1.IterateAll(gm_c);
scm.FreeTensor(scmLocal);
```

##### 10.1.3.3.10 矩阵乘输出的N方向对齐

功能介绍

矩阵乘输出的N方向对齐，即矩阵乘结果C矩阵按ND_ALIGN格式输出。在Matmul矩阵乘法中，常用的矩阵数据格式有ND、NZ，相关介绍可参考数据格式章节。ND_ALIGN是矩阵的另一种数据格式，该格式一般用于N方向非32字节对齐的矩阵乘计算中，配置结果C矩阵为ND_ALIGN格式后，将按照N方向32字节对齐的补齐规则输出C矩阵，详细内容请见ND_ALIGN。

以M=16，K=16，N=14，A、B矩阵数据类型为half的Matmul为具体示例，说明ND_ALIGN输出功能。当配置C矩阵为ND格式并输出到Global Memory时，按照原始N方向大小非32字节对齐输出。当配置C矩阵为ND格式时，按照N方向32字节对齐输出，C矩阵的N方向最后两列由下一行的实际数据进行填充补齐，以实现N方向对齐到32字节并输出。当配置C矩阵为ND_ALIGN格式时，Matmul API会在C矩阵的N方向上通过添加无效数据来填充最后两列，以确保N方向对齐至32字节并输出。

[图: ND格式C矩阵N方向非32字节对齐示意图]

[图: ND格式C矩阵N方向32字节对齐示意图，最后两列由下一行数据填充]

[图: ND_ALIGN格式C矩阵N方向32字节对齐示意图，最后两列填充无效数据]

使用场景：Matmul计算中N方向非32字节对齐，输出C矩阵的N方向要求32字节对齐的场景。

约束说明：若配置C矩阵为ND_ALIGN格式输出，则为C矩阵申请的Buffer空间为N向上32字节对齐后的空间大小。

调用示例

完整的算子样例请参考matmul_nd_align算子样例。

- Tiling实现

调用SetCType接口，设置C矩阵的数据格式为CubeFormat::ND_ALIGN，其它Tiling实现与基础场景相同。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MatmulApiTiling tiling(ascendcPlatform);
tiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
tiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
// 设置C矩阵，buffer位置为GM，数据格式为ND_ALIGN
tiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND_ALIGN,
    matmul_tiling::DataType::DT_FLOAT);
tiling.SetBiasType(AscendC::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
... // 其他内容
optiling::TCubeTiling tilingData;
int ret = tiling.GetTiling(tilingData);
```

- Kernel实现

相较于基础场景，ND_ALIGN输出功能要求在创建Matmul对象时，设置模板参数cType的数据格式为CubeFormat::ND_ALIGN。

```cpp
#include "lib/matmul_intf.h"

typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> aType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half> bType;
// 设置模板参数cType的数据格式为ND_ALIGN
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND_ALIGN, float> cType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> biasType;
AscendC::Matmul<aType, bType, cType, biasType> mm;
```

##### 10.1.3.3.11 单次矩阵乘局部输出

功能介绍

单次矩阵乘局部输出，又称Partial Output。如基础知识中所述，一次Iterate计算过程中，会按K方向进行一次或多次基本块计算，其中的一次基本块计算为baseM*baseK和baseK*baseN大小的输入数据进行计算得到baseM*baseN大小的结果；每次基本块计算的结果进行累加后，便得到baseM*singleCoreK和singleCoreK*baseN大小的输入数据计算得到的结果baseM*baseN，并将其作为一次Iterate的最终结果输出。

开启Partial Output功能后，调用Iterate接口不会进行K轴累加，只进行单次基本块计算。用户可以通过GetTensorC接口获取对应的单片数据，最后自行进行K轴上的累加。

[图: 未开启Partial Output功能计算示意图，展示K方向累加后输出]

[图: 开启Partial Output功能计算示意图，展示单次计算后直接输出]

使用场景：矩阵乘计算结果不需要累加，只需要输出baseM*baseK和baseK*baseN的计算结果baseM*baseN。例如需要先获取单次基本块计算的数据进行反量化，再累加得到最终结果。

约束说明：

- 该功能仅支持MDL模板。
- 获取矩阵乘计算结果时，仅支持调用Iterate和GetTensorC接口的连续写模式，不支持非连续写模式以及IterateAll接口获取计算结果，连续写模式的介绍请参考GetTensorC。
- 该功能不支持带有Bias矩阵的Matmul计算，即不支持输入Bias矩阵。

调用示例

完整的算子样例请参考开启Partial Output功能的算子样例。

```cpp
// 配置MDL模板，使能Partial Output
constexpr static MatmulConfigMode configMode = MatmulConfigMode::CONFIG_MDL;
constexpr static MatmulFuncParams funcParams = {
    false, false, false, false, 0, IterateOrder::UNDEF, ScheduleType::INNER_PRODUCT, true, true,
    true /* isPartialOutput */
};
constexpr static MatmulConfig CFG_PARTIAL = GetMMConfig<configMode>(funcParams);
Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, CFG_PARTIAL> mm;
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm);
mm.Init(&tiling);
mm.SetTensorA(gmA, isTransposeA);
mm.SetTensorB(gmB, isTransposeB);
while (mm.Iterate()) {
    mm.GetTensorC(tmpGmC[dstOffset], false, true);
    dstOffset += baseM * baseN;
    // 其他操作
}
```

##### 10.1.3.3.12 AIC和AIV独立运行机制

功能介绍

AIC和AIV独立运行机制，又称双主模式。在分离模式下，区别于MIX模式（包含矩阵计算和矢量计算）通过消息机制驱动AIC运行，双主模式为AIC和AIV独立运行代码，不依赖消息驱动，使能双主模式能够提高Matmul计算性能。默认情况下，双主模式不使能，需要通过MatmulConfig中的enableMixDualMaster参数开启。

使用场景：算子中的矩阵计算和矢量计算相关代码独立运行，不依赖消息驱动时，可以开启双主模式，以提高Matmul计算性能。

约束说明：

- 该功能仅支持Norm模板和MDL模板。
- 算子核函数的类型为MIX，同时AIC核数：AIV核数为1:1。
- 算子核函数的类型为MIX，同时AIC核数：AIV核数为1:2，且A矩阵和B矩阵同时使能IBSHARE参数。
- 同一算子中所有Matmul对象的该参数取值必须保持一致。
- A、B、Bias矩阵只支持从Global Memory输入。
- 获取矩阵计算结果只支持调用IterateAll接口输出到GlobalTensor，即计算结果放置于Global Memory的地址，不能调用GetTensorC等接口获取结果。

调用示例

完整的算子样例请参考使能双主模式的算子样例。

```cpp
// 修改模板参数enableMixDualMaster=true，Norm模板开启双主模式，MDL模板使用GetMDLConfig接口获取模板参数。
constexpr static MatmulConfig MM_CFG = GetNormalConfig(false, false, false,
    BatchMode::BATCH_LESS_THAN_L1, true, IterateOrder::ORDER_M, ScheduleType::OUTER_PRODUCT, false,
    true/*enableMixDualMaster*/);
Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, MM_CFG> mm;

// 常规Matmul计算
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm);
mm.SetTensorA(gm_a);
mm.SetTensorB(gm_b);
mm.SetBias(gm_bias);
mm.IterateAll(gm_c);
```

##### 10.1.3.3.13 Batch Matmul基础功能

功能介绍

Batch Matmul是指批量处理Matmul计算的场景。该场景对外提供了IterateBatch的调用接口，可以计算出多个singleCoreM * singleCoreN大小的C矩阵。

Matmul单次计算的过程需要搬入和搬出数据，当进行多次Matmul计算且单次Matmul计算的输入shape较小时，搬运开销在整体耗时中占比较大。通过IterateBatch接口批量处理Matmul，可以有效提升带宽利用率。

Batch Matmul当前支持4种Layout类型：BSNGD、SBNGD、BNGS1S2、NORMAL（BMNK的数据排布格式），相关数据排布格式请参考IterateBatch。

下图为NORMAL数据排布格式的Batch Matmul计算。整个Matmul计算一共包含4个矩阵乘操作：mat_a1*mat_b1、mat_a2*mat_b2、mat_a3*mat_b3、mat_a4*mat_b4，需要单核上计算四个singleCoreM * singleCoreN。在该场景下，如果shape较小，可以将其视为Batch Matmul场景进行批量处理，以提升性能。一次IterateBatch可同时计算出mat_c1 = mat_a1 * mat_b1、mat_c2 = mat_a2 * mat_b2、mat_c3 = mat_a3 * mat_b3、mat_c4 = mat_a4 * mat_b4。

[图: NORMAL数据排布格式的Batch Matmul示意图]

使用场景：Matmul计算需要计算出多个singleCoreM * singleCoreN大小的C矩阵，且单次Matmul计算处理的shape较小。

约束说明：

- 只支持Norm模板。
- 对于BSNGD、SBNGD、BNGS1S2 Layout格式，输入A、B矩阵按分形对齐后的多Batch数据总和应小于L1 Buffer的大小；对于NORMAL Layout格式没有这种限制，但需通过MatmulConfig配置batchMode参数，即输入A、B矩阵多Batch数据大小与L1 Buffer的大小关系。
- 对于BSNGD、SBNGD、BNGS1S2 Layout格式，称左矩阵、右矩阵的G轴分别为ALayoutInfoG、BLayoutInfoG，则ALayoutInfoG / batchA = BLayoutInfoG / batchB；对于NORMAL Layout格式，batchA、batchB必须满足倍数关系。Bias的shape(batch, n)中的batch必须与C矩阵的batch相等。
- 如果接口输出到Unified Buffer上，输出C矩阵大小BaseM*BaseN应小于分配的Unified Buffer内存大小。
- 对于BSNGD、SBNGD Layout格式，输入输出只支持ND格式数据。对于BNGS1S2、NORMAL Layout格式，输入支持ND/NZ格式数据。
- Batch Matmul不支持量化/反量化模式，即不支持SetQuantScalar、SetQuantVector接口。
- BSNGD场景，不支持一次计算多行SD，需要算子程序中循环计算。
- 异步模式不支持IterateBatch搬运到Unified Buffer上。
- 模板参数enableMixDualMaster（默认取值为false）设置为true，即使能MixDualMaster（双主模式）场景时，不支持Batch Matmul。
- 在batch场景，A矩阵、B矩阵支持half/float/bfloat16_t/int8_t/int4b_t数据类型，不支持int4b_t数据类型。

调用示例

以下是NORMAL数据排布格式的Batch Matmul调用示例。BSNDG数据排布格式的Batch Matmul完整示例请参考BatchMatmul样例。

- Tiling实现

使用SetBatchInfoForNormal设置A/B/C的M/N/K轴信息和A/B矩阵的BatchNum。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MultiCoreMatmulTiling tiling(ascendcPlatform);
int32_t M = 32;
int32_t N = 256;
int32_t K = 64;
tiling->SetDim(1);
tiling->SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
tiling->SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
tiling->SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
tiling->SetBiasType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
tiling->SetShape(M, N, K);
tiling->SetOrgShape(M, N, K);
tiling->EnableBias(true);
tiling->SetBufferSpace(-1, -1, -1);

constexpr int32_t BATCH_NUM = 3;
tiling->SetBatchInfoForNormal(BATCH_NUM, BATCH_NUM, M, N, K); // 设置矩阵排布
tiling->SetBufferSpace(-1, -1, -1);

optiling::TCubeTiling tilingData;
int ret = tiling.GetTiling(tilingData);
```

- Kernel实现
  - 创建Matmul对象。通过MatmulType设置输入输出的Layout格式为NORMAL。

```cpp
#include "lib/matmul_intf.h"

typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half, false,
    LayoutMode::NORMAL> aType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half, true,
    LayoutMode::NORMAL> bType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float, false,
    LayoutMode::NORMAL> cType;
typedef AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND, float> biasType;
constexpr MatmulConfig MM_CFG = GetNormalConfig(false, false, false,
    BatchMode::BATCH_LESS_THAN_L1);
AscendC::Matmul<aType, bType, cType, biasType, MM_CFG> mm;
```

  - 初始化操作。

```cpp
REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling); // 初始化matmul对象
```

  - 设置左矩阵A、右矩阵B、Bias。

```cpp
mm.SetTensorA(gm_a);   // 设置左矩阵A
mm.SetTensorB(gm_b);   // 设置右矩阵B
mm.SetBias(gm_bias);   // 设置Bias
```

  - 完成矩阵乘操作。左矩阵每次计算batchA个MK数据，右矩阵每次计算batchB个KN数据。

```cpp
mm.IterateBatch(gm_c, batchA, batchB, false);
```

  - 结束矩阵乘操作。

```cpp
mm.End();
```

##### 10.1.3.3.14 Batch Matmul复用Bias矩阵

功能介绍

在Batch Matmul场景中，Matmul API可以一次性计算出多个大小为singleCoreM * singleCoreN的C矩阵。当Batch Matmul场景有Bias输入时，默认的Bias输入矩阵包含Batch轴，即Bias的大小为Batch * N。通过开启Bias复用功能，当每个Batch计算使用的Bias数据相同时，只需输入一个不带Batch轴的Bias矩阵。Batch Matmul的Bias矩阵复用功能默认不启用，用户需要设置MatmulConfig中的isBiasBatch参数为false来开启此功能。

[图: 带有Batch轴的Bias计算示意图]

[图: 复用Bias计算示意图，所有batch共用一个bias1]

使用场景：Batch Matmul中每个Batch的Matmul计算可以使用相同的Bias矩阵。

约束说明：A、B、C矩阵的Layout类型都为NORMAL时，不支持batchMode参数设为SINGLE_LARGE_THAN_L1，即Bias复用场景下，单Batch的A、B矩阵数据总和不得超过L1 Buffer的大小。

调用示例

完整的算子样例请参考BatchMatmul复用Bias算子样例。

```cpp
// 自定义MatmulConfig参数，将其中的isBiasBatch参数设置为false，使能BatchMatmul的Bias复用功能。
constexpr MatmulConfigMode configMode = MatmulConfigMode::CONFIG_NORM;
constexpr MatmulBatchParams batchParams = {
    false, BatchMode::BATCH_LESS_THAN_L1, false /* isBiasBatch */
};
constexpr MatmulConfig CFG_MM = GetMMConfig<configMode>(batchParams);
AscendC::Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, CFG_MM> mm;

REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), mm, &tiling); // 初始化matmul对象
mm.SetTensorA(gm_a);   // 设置左矩阵A
mm.SetTensorB(gm_b);   // 设置右矩阵B
mm.SetBias(gm_bias);   // 设置Bias，矩阵大小为1 * singleCoreN
mm.IterateBatch(gm_c, batchA, batchB, false);
mm.End();
```

### 10.1.4 矩阵编程（基础API）

#### 10.1.4.1 耦合模式

说明：本节内容为针对耦合模式，使用基础API进行矩阵乘法的编程指导。

编程范式

Cube编程范式把算子的实现流程分为5个基本任务：CopyIn，Split，Compute，Aggregate，CopyOut。CopyIn负责搬入操作，Split负责数据切分操作，Compute负责矩阵指令计算操作，Aggregate负责数据汇聚操作，CopyOut负责搬出操作。

[图: 矩阵编程基本任务设计图，展示5个Stage：CopyIn->Split->Compute->Aggregate->CopyOut]

具体任务之间的交互流程和流程图如下。

步骤1 Stage1：CopyIn任务。
1. 使用DataCopy接口将GlobalTensor数据拷贝到LocalTensor。
2. 使用EnQue将LocalTensor放入A1/B1的Queue中。

步骤2 Stage2：Split任务。
1. 使用DeQue从A1/B1中取出LocalTensor。
2. 使用LoadData接口将LocalTensor从A1/B1中搬运到A2/B2。
3. 使用EnQue将计算结果LocalTensor放入到A2/B2的Queue中。

步骤3 Stage3：Compute任务。
1. 使用DeQue从A2/B2中取出LocalTensor。
2. 使用Mmad接口完成矩阵计算。
3. 使用EnQue将计算结果LocalTensor放入到CO1的Queue中。

步骤4 Stage4：Aggregate任务。
1. 使用DeQue从CO1中取出LocalTensor。
2. 使用Ascend C接口拷贝结果矩阵到CO2。
3. 使用EnQue将计算结果LocalTensor放入到CO2的Queue中。

步骤5 Stage5：CopyOut任务。
1. 使用DeQue接口从CO2的Queue中取出LocalTensor。
2. 使用DataCopy接口将LocalTensor拷贝到GlobalTensor上。

[图: 矩阵编程Queue队列示意图，展示各Stage之间通过A1/B1、A2/B2、CO1、CO2 Queue进行数据传递]

开发流程

基于Ascend C方式实现矩阵算子的流程如下图所示。

[图: 矩阵算子实现流程图，包括算子分析->核函数定义->根据矩阵编程范式实现算子类(CopyIn/Split/Compute/Aggregate/CopyOut)]

- 算子分析：分析算子的数学表达式、输入、输出以及计算逻辑的实现，明确需要调用的Ascend C接口。
- 核函数定义：定义Ascend C算子入口函数。
- 根据矩阵编程范式实现算子类：完成核函数的内部实现，调用私有成员函数CopyIn、SplitA、SplitB、Compute、Aggregate、CopyOut完成矩阵算子的五级流水操作。

下文将以Matmul算子为例对上述步骤进行详细介绍，Matmul算子的代码框架如下，完整代码请参见Mmad样例。

```cpp
#include "kernel_operator.h"

// 根据编程范式实现算子类
class KernelMatmul {
public:
    __aicore__ inline void Init(GM_ADDR a, GM_ADDR b, GM_ADDR c)
    {
        // ...
    }
    __aicore__ inline void Process()
    {
        CopyIn();
        SplitA();
        AscendC::LocalTensor<half> b1Local = inQueueB1.DeQue<half>();
        AscendC::LocalTensor<half> a2Local = inQueueA2.DeQue<half>();
        AscendC::LocalTensor<float> c2Local = outQueueCO2.AllocTensor<float>();
        // split matrix b into 2 parts, [32, 16] and [32, 16]
        for (int i = 0; i < 2; ++i) {
            SplitB(b1Local, i);
            Compute(a2Local, i);
            Aggregate(c2Local, i);
        }
        inQueueB1.FreeTensor(b1Local);
        inQueueA2.FreeTensor(a2Local);
        outQueueCO2.EnQue<float>(c2Local);
        CopyOut();
    }
private:
    __aicore__ inline void CopyIn()
    {
        // ...
    }
    __aicore__ inline void SplitA()
    {
        // ...
    }
    __aicore__ inline void SplitB(const LocalTensor<half>& b1Local, const int bSplitIdx)
    {
        // ...
    }
    __aicore__ inline void Compute(const LocalTensor<half>& a2Local)
    {
        // ...
    }
    __aicore__ inline void Aggregate(const LocalTensor<float>& c2Local, int bSplitIdx)
    {
        // ...
    }
    __aicore__ inline void CopyOut()
    {
        // ...
    }
private:
    // ...
};

//核函数定义
extern "C" __global__ __aicore__ void matmul_custom(GM_ADDR a, GM_ADDR b, GM_ADDR c)
{
    KernelMatmul op;
    op.Init(a, b, c);
    op.Process();
}
```

算子分析

在开发算子代码之前需要分析算子的数学表达式、输入、输出以及计算逻辑的实现，明确需要调用的Ascend C接口。

步骤1 明确算子的数学表达式及计算逻辑。

Matmul算子完成矩阵乘操作，其数学表达式如下，形状为[m, k]的矩阵a和形状为[k, n]的矩阵b相乘，得到形状为[m, n]的矩阵c。为了方便，令m=k=n=32。c = a * b

注意需要处理的数据过大时，需要对数据进行切分并分块搬运到A2、B2，分别计算后再进行汇聚。下文的计算逻辑为了展示Split和Aggregate阶段的样例，请您根据实际需要处理的数据大小决定是否需要切分和汇聚。

计算逻辑如下：
1. 分别搬运输入数据矩阵a、b至Local Memory A1、B1。
2. 将a矩阵从A1搬运至A2。为实现部分并行，将b矩阵切分为part1和part2，形状均为[k, n / 2]，切分后再分块搬运至B2。
3. a矩阵和b矩阵part1、part2分别做矩阵乘运算，获得矩阵c的part1和part2，形状均为[m, n / 2]。计算结果在CO1存储。
4. 将矩阵c的part1和part2分别拷贝到CO2进行合并。
5. 将合并后的输出数据从CO2搬出。

步骤2 明确输入和输出。
- Matmul算子有两个输入：a与b，输出为c。
- 本样例中算子输入支持的数据类型为half（float16），算子输出的数据类型为float32。
- 矩阵a、b、c的形状均为[32, 32]。
- 算子输入输出支持的数据格式为：ND。

步骤3 确定核函数名称和参数。
- 您可以自定义核函数名称，本样例中核函数命名为matmul_custom。
- 根据对算子输入输出的分析，确定核函数有3个参数a, b, c；a, b为输入在Global Memory上的内存地址，c为输出在Global Memory上的内存地址。

步骤4 约束分析。

由于硬件架构对矩阵乘计算的输入输出有格式约束，需要在算子实现中增加格式转换的流程。
- 搬运矩阵a、b至A1、B1时，将ND格式的矩阵a、b转换为NZ格式。
- 从A1搬运矩阵a至A2时，将NZ格式的a矩阵转换为ZZ格式；从B1搬运矩阵b到B2时将NZ格式的b矩阵转换为ZN格式。
- 将计算结果从CO2搬出时，将NZ格式的c矩阵转换为ND格式。
- 数据排布格式的相关介绍详见数据排布格式。

步骤5 确定算子实现所需接口。
- 实现外部存储和内部存储间的数据搬运，查看Ascend C API参考中的数据搬移接口，具体参考DataCopy。
- 实现矩阵数据格式转换，查看Ascend C API参考中的数据转换接口，具体参考LoadData。
- 矩阵计算过程涉及矩阵乘法，查看Ascend C API参考中的矩阵计算接口，具体参考Mmad。
- 计算中使用到的Tensor数据结构，使用Queue队列进行管理，会使用到EnQue、DeQue等接口。
- 计算中使用到的Tensor数据结构，使用Queue队列进行管理，会使用到EnQue、DeQue等接口。

----结束

通过以上分析，得到Ascend C Matmul算子的计算流程图和设计规格如下：

[图: Matmul算子的计算流程图 - CopyIn/Split/Compute/Aggregate/CopyOut五阶段]

表 10-9 Ascend C Matmul算子设计规格

| 算子类型（OpType） | Matmul |
| --- | --- |
| 算子输入 | name: a, shape: (m,k)=(32,32), data type: half, format: ND |
|  | name: b, shape: (k,n)=(32,32), data type: half, format: ND |
| 算子输出 | name: c, shape: (m,n)=(32,32), data type: float32, format: ND |
| 核函数名称 | matmul_custom |
| 使用的主要接口 | DataCopy: 数据搬移接口 |
|  | LoadData: 矩阵数据格式转换接口 |
|  | Mmad: 矩阵乘计算接口 |
|  | EnQue、DeQue等接口: Queue队列管理接口 |
| 算子实现文件名称 | matmul_custom.cpp |

## 核函数定义

根据4.3 核函数中介绍的规则进行核函数的定义。

步骤1 函数原型定义。

本样例中，函数名为matmul_custom（核函数名称可自定义）；根据算子分析中对算子输入输出的分析，确定有3个参数a，b，c，其中a，b都为输入内存，c为输出内存。根据4.3 核函数中核函数的规则介绍，函数原型定义如下所示：使用__global__函数类型限定符来标识它是一个核函数，可以被<<<>>>调用；使用__aicore__函数类型限定符来标识该核函数在设备端aicore上执行；为方便起见，统一使用GM_ADDR宏来修饰入参，GM_ADDR宏定义请参考4.3 核函数。

```cpp
extern "C" __global__ __aicore__ void matmul_custom(GM_ADDR a, GM_ADDR b, GM_ADDR c)
{
}
```

步骤2 调用算子类的Init和Process函数。

算子类的Init函数，完成内存初始化相关工作，Process函数完成算子实现的核心逻辑，具体介绍参见算子类实现。

```cpp
extern "C" __global__ __aicore__ void matmul_custom(GM_ADDR a, GM_ADDR b, GM_ADDR c)
{
    KernelMatmul op;
    op.Init(a, b, c);
    op.Process();
}
```

步骤3 对核函数进行封装，得到matmul_custom_do函数，便于主程序调用。#ifndef ASCENDC_CPU_DEBUG表示该封装函数仅在编译运行NPU侧的算子时会用到，编译运行CPU侧的算子时，可以直接调用matmul_custom函数。根据核函数定义和调用章节，调用核函数时，除了需要传入参数a，b，c，还需要传入blockDim（核函数执行的核数），l2ctrl（保留参数，设置为nullptr），stream（应用程序中维护异步操作执行顺序的stream）来规定核函数的执行配置。

```cpp
#ifndef ASCENDC_CPU_DEBUG
// call of kernel function
void matmul_custom_do(uint32_t blockDim, void* l2ctrl, void* stream, uint8_t* a, uint8_t* b, uint8_t* c)
{
    matmul_custom<<<blockDim, l2ctrl, stream>>>(a, b, c);
}
#endif
```

----结束

## 算子类实现

根据上一章节介绍，核函数中会调用算子类的Init和Process函数，本章具体讲解基于编程范式实现算子类。矩阵编程范式请参考编程范式。

算子类中主要包含对外开放的初始化Init函数和核心处理函数Process以及一些实现中会用到的私有成员。KernelMatmul算子类的定义如下：

```cpp
class KernelMatmul {
public:
    __aicore__ inline KernelMatmul(){}
    // 初始化函数，完成内存初始化相关操作
    __aicore__ inline void Init(GM_ADDR a, GM_ADDR b, GM_ADDR c){}
    // 核心处理函数，实现算子逻辑
    // 调用私有成员函数CopyIn、SplitA、SplitB、Compute、Aggregate、CopyOut完成矩阵算子的五级流水操作
    __aicore__ inline void Process(){}

private:
    __aicore__ inline void CopyND2NZ(const LocalTensor<half>& dst, const GlobalTensor<half>& src, const uint16_t height, const uint16_t width){}
    // 搬进函数，完成编程范式中的CopyIn阶段的处理，由Process函数调用
    __aicore__ inline void CopyIn(){}
    // 搬进函数，完成编程范式中的Split阶段的处理，由Process函数调用
    __aicore__ inline void SplitA(){}
    // 搬进函数，完成编程范式中的Split阶段的处理，由Process函数循环调用两次，分别搬运b矩阵的两个part
    __aicore__ inline void SplitB(const LocalTensor<half>& b1Local, const int bSplitIdx){}
    // 计算函数，完成编程范式中的Compute阶段的处理，由Process函数循环调用两次，分别计算出矩阵c的两个part
    __aicore__ inline void Compute(const LocalTensor<half>& a2Local){}
    // 搬出函数，完成编程范式中的Aggregate阶段的处理，由Process函数循环调用两次，分别搬运矩阵c的两个part
    __aicore__ inline void Aggregate(const LocalTensor<float>& c2Local, const int bSplitIdx){}
    // 搬出函数，完成编程范式中的CopyOut阶段的处理，由Process函数调用
    __aicore__ inline void CopyOut(){}

private:
    AscendC::TPipe pipe; // Pipe内存管理对象，管理Queue队列的内存
    AscendC::TQue<AscendC::TPosition::A1, 1> inQueueA1; // 输入数据的队列，TPosition为A1
    AscendC::TQue<AscendC::TPosition::A2, 1> inQueueA2; // 输入数据的队列，TPosition为A2
    AscendC::TQue<AscendC::TPosition::B1, 1> inQueueB1; // 输入数据的队列，TPosition为B1
    AscendC::TQue<AscendC::TPosition::B2, 2> inQueueB2; // 输入数据的队列，TPosition为B2
    AscendC::TQue<AscendC::TPosition::CO1, 2> outQueueCO1; // 输出数据的队列，TPosition为CO1
    AscendC::TQue<AscendC::TPosition::CO2, 1> outQueueCO2; // 输出数据的队列，TPosition为CO2
    // 管理输入输出Global Memory内存地址的对象，其中aGM、bGM为输入，cGM为输出
    AscendC::GlobalTensor<half> aGM;
    AscendC::GlobalTensor<half> bGM;
    AscendC::GlobalTensor<float> cGM;

    uint16_t m = 32;
    uint16_t n = 32;
    uint16_t k = 32;
    uint16_t aSize, bSize, cSize, mBlocks, nBlocks, kBlocks;
};
```

### KernelMatmul构造函数实现

构造函数中对私有成员变量进行初始化，具体代码如下：

```cpp
__aicore__ inline KernelMatmul()
{
    aSize = m * k;
    bSize = k * n;
    cSize = m * n;
    mBlocks = m / 16;
    nBlocks = n / 16;
    kBlocks = k / 16;
}
```

矩阵a的形状为[m, k]，矩阵b的形状为[k, n]，矩阵c的形状为[m,n]，此样例中m、n、k均设置为32。

aSize、bSize、cSize分别为矩阵a、b、c的数值个数。

mBlocks、nBlocks、kBlocks为m、n、k所占分形数量，half类型一个分形Shape为16 * 16，blocks计算公式为：

- mBlocks = m / 16
- nBlocks = n / 16
- kBlocks = k / 16

分形具体介绍可参考11.2.2 数据排布格式。

### Init函数实现

Init函数主要完成以下内容：

- 设置输入输出Global Tensor的Global Memory内存地址。

以设置输入a在Global Memory上的内存偏移地址为例：

```cpp
aGM.SetGlobalBuffer((__gm__ half*)a);
```

注意，因为本样例中Init函数的入参统一设置为uint8_t*，这里需要强转成具体的数据类型(__gm__ half*)，再进行偏移。

- 通过Pipe内存管理对象为输入输出Queue分配内存。

比如，为输入队列inQueueB2分配内存，可以通过如下代码段实现：

```cpp
pipe.InitBuffer(inQueueB2, 2, bSize * sizeof(half) / 2);
```

此样例中将b矩阵切分为两个part，为inQueueB2分配内存时需要申请两块内存空间，每一块的大小为b矩阵大小的一半，outQueueCO1的内存初始化同理。

具体的初始化代码如下：

```cpp
__aicore__ inline void Init(GM_ADDR a, GM_ADDR b, GM_ADDR c)
{
    aGM.SetGlobalBuffer((__gm__ half*)a);
    bGM.SetGlobalBuffer((__gm__ half*)b);
    cGM.SetGlobalBuffer((__gm__ float*)c);
    pipe.InitBuffer(inQueueA1, 1, aSize * sizeof(half));
    pipe.InitBuffer(inQueueA2, 1, aSize * sizeof(half));
    pipe.InitBuffer(inQueueB1, 1, bSize * sizeof(half));
    pipe.InitBuffer(inQueueB2, 2, bSize * sizeof(half) / 2);
    pipe.InitBuffer(outQueueCO1, 2, cSize * sizeof(float) / 2);
    pipe.InitBuffer(outQueueCO2, 1, cSize * sizeof(float));
}
```

### Process函数实现

基于矩阵编程范式，将核函数的实现分为5个基本阶段：CopyIn，Split，Compute，Aggregate，CopyOut。Split，Compute，Aggregate阶段需要区分a、b矩阵。Process函数中通过如下方式调用这几个函数。

```cpp
__aicore__ inline void Process()
{
    CopyIn();
    SplitA();
    AscendC::LocalTensor<half> b1Local = inQueueB1.DeQue<half>();
    AscendC::LocalTensor<half> a2Local = inQueueA2.DeQue<half>();
    AscendC::LocalTensor<float> c2Local = outQueueCO2.AllocTensor<float>();
    // split matrix b into 2 parts, [32, 16] and [32, 16]
    for (int i = 0; i < 2; ++i) {
        SplitB(b1Local, i);
        Compute(a2Local);
        Aggregate(c2Local, i);
    }
    inQueueB1.FreeTensor(b1Local);
    inQueueA2.FreeTensor(a2Local);
    outQueueCO2.EnQue<float>(c2Local);
    CopyOut();
}
```

两次循环内，SplitB需要从inQueueB1中分别搬运两个part的b矩阵，Compute需要分别计算a矩阵和两个part b矩阵的乘法，Aggregate要分别搬运两个part的c矩阵，具体五个阶段数据流通示意图如下：

[图: 数据流通示意图 - CopyIn/SplitA/SplitB/Compute/Aggregate/CopyOut]

[图: 并行示意图 - CopyIn, SplitA, SplitB iter1/iter2, Compute iter1/iter2, Aggregate iter1/iter2, CopyOut]

步骤1 Stage1: CopyIn函数实现。

1. 使用AllocTensor从A1、B1的Queue中申请a1Local和b1Local。
2. 使用DataCopy接口将矩阵a、b搬运到Local Memory，同时将其数据格式从ND转换为NZ。

一次DataCopy指令搬运height*16个数，循环执行width/16次。DataCopy的参数设置如下：
- blockCount设置为height，共搬运height次。
- blockLen设置为1，一次搬运16个类型为half的数。
- srcStride设置为width/16 - 1，源矩阵每搬运一个block需要跳跃一行。
- dstStride设置为0，目的矩阵每个block在内存中连续存储。
- 每次循环迭代，源矩阵首地址移动16个数，目的矩阵首地址移动16*height个数。

[图: ND to NZ 转换示意图]

注意：上述ND到NZ的格式转换仅作为举例说明，开发者可根据实际情况选择合适的转换方式。

3. 使用EnQue将a1Local、b1Local分别放入A1、B1的Queue中。

具体代码如下：

```cpp
__aicore__ inline void CopyND2NZ(const LocalTensor<half>& dst, const GlobalTensor<half>& src, const uint16_t height, const uint16_t width)
{
    for (int i = 0; i < width / 16; ++i) {
        int srcOffset = i * 16;
        int dstOffset = i * 16 * height;
        AscendC::DataCopy(dst[dstOffset], src[srcOffset], { height, 1, uint16_t(width / 16 - 1), 0 });
    }
}
__aicore__ inline void CopyIn()
{
    AscendC::LocalTensor<half> a1Local = inQueueA1.AllocTensor<half>();
    AscendC::LocalTensor<half> b1Local = inQueueB1.AllocTensor<half>();
    CopyND2NZ(a1Local, aGM, m, k);
    CopyND2NZ(b1Local, bGM, k, n);
    inQueueA1.EnQue(a1Local);
    inQueueB1.EnQue(b1Local);
}
```

步骤2 Stage2: SplitA函数实现。

1. 使用DeQue从A1的Queue中取出a1Local。
2. 使用AllocTensor从A2的Queue中申请a2Local。
3. 使用LoadData将矩阵a搬运到A2，同时将a矩阵从NZ格式转换为ZZ格式。

搬运及格式转换示意图如下：图中k为32，占kBlocks（k/16=2）个分形，m为32，占mBlocks（m/16=2）个分形，一共搬运4个16*16分形。本示例中，调用一次LoadData接口完成两个16*16分形的搬运，循环调用两次LoadData。第一次循环搬运蓝色部分两个分形，第二次循环搬运绿色部分两个分形。

[图: NZ to ZZ 格式转换示意图]

单次循环中LoadData（本样例中要完成2个分形的搬运，蓝色部分或者绿色部分）的参数设置如下：
- repeatTimes表示数据处理的迭代次数，因为LoadData每个迭代处理一个分形，所以也可以理解为待搬运分形的个数。本样例中即为k轴方向的分形个数，设置为kBlocks，表示搬运kBlocks个分形。
- srcStride表示，相邻迭代间源操作数分形首地址之间的间隔，以搬运蓝色部分分形为例：下图中左侧源操作数矩阵，第一个蓝色分形和第二个蓝色分形起始地址之间的间隔为mBlocks个分形，此处设置为mBlocks。
- dstGap使用默认值，目的矩阵两个分形连续存储。
- ifTranspose设置为false，每块分形搬运前搬运后都是Z格式，不使能转置。
- 每次循环迭代源矩阵首地址偏移16*16，目的矩阵首地址偏移16*k。

4. 使用EnQue将计算结果a2Local放入到A2的Queue中。

具体代码如下：

```cpp
__aicore__ inline void SplitA()
{
    int srcOffset = 0;
    int dstOffset = 0;
    AscendC::LocalTensor<half> a1Local = inQueueA1.DeQue<half>();
    AscendC::LocalTensor<half> a2Local = inQueueA2.AllocTensor<half>();

    // transform nz to zz
    for (int i = 0; i < mBlocks; ++i) {
        AscendC::LoadData2DParams loadDataParams;
        loadDataParams.repeatTimes = kBlocks;
        loadDataParams.srcStride = mBlocks;
        loadDataParams.ifTranspose = false;

        AscendC::LoadData(a2Local[dstOffset], a1Local[srcOffset], loadDataParams);

        srcOffset += 16 * 16;
        dstOffset += k * 16;
    }

    inQueueA2.EnQue<half>(a2Local);
    inQueueA1.FreeTensor(a1Local);
}
```

步骤3 Stage2: SplitB函数实现。

1. SplitB需要传入两个参数：使用DeQue从B1的Queue中取出的b1Local和循环迭代变量index。
2. 使用AllocTensor从B2的Queue中申请b2Local。
3. 使用LoadData将b矩阵搬运到B2，同时从NZ格式转换为ZN格式。

搬运及格式转换示意图如下：图中k为32，占kBlocks（k/16=2）个分形，n为32，占nBlocks（n/16=2）个分形，一共搬运4个16*16分形。本示例中，调用一次LoadData接口完成两个16*16分形的搬运，循环调用两次LoadData。第一次循环搬运蓝色部分两个分形，第二次循环搬运绿色部分两个分形。

[图: NZ to ZN 格式转换示意图]

单次循环中LoadData（本样例中要完成2个分形的搬运，蓝色部分或者绿色部分）的参数设置如下：
- repeatTimes表示数据处理的迭代次数，因为LoadData每个迭代处理一个分形，所以也可以理解为待搬运分形的个数。本样例中即为k轴方向的分形个数，设置为kBlocks，表示搬运kBlocks个分形。
- srcStride设置为1，源矩阵两个分形之间的间隔为1个分形，此处设置为1。
- dstGap使用默认值0，目的矩阵两个分形连续存储。
- ifTranspose设置为true，每块分形搬运前为Z格式，搬运后需要为N格式，需要使能转置。
- 每次循环迭代，源矩阵首地址需要偏移k*n/2。

4. 使用EnQue将计算结果b2Local放入到B2的Queue中。

具体代码如下：

```cpp
__aicore__ inline void SplitB(const AscendC::LocalTensor<half>& b1Local, const int bSplitIdx)
{
    AscendC::LocalTensor<half> b2Local = inQueueB2.AllocTensor<half>();

    // transform nz to zn
    AscendC::LoadData2DParams loadDataParams;
    loadDataParams.repeatTimes = kBlocks;
    loadDataParams.srcStride = 1;
    loadDataParams.ifTranspose = true;

    AscendC::LoadData(b2Local, b1Local[bSplitIdx * bSize / 2], loadDataParams);

    inQueueB2.EnQue<half>(b2Local);
}
```

步骤4 Stage3: Compute函数实现，完成核心的矩阵计算功能。

1. Compute函数需要传入参数a2Local，a2Local从A2的Queue中使用DeQue取出。
2. 使用AllocTensor从CO1的Queue中申请c1Local。
3. 使用DeQue从B2中取出b2Local。
4. 使用Mmad完成矩阵乘计算。
5. 使用EnQue将计算结果c1Local放入到CO1的Queue中。

具体代码如下：

```cpp
__aicore__ inline void Compute(const AscendC::LocalTensor<half>& a2Local)
{
    AscendC::LocalTensor<half> b2Local = inQueueB2.DeQue<half>();
    AscendC::LocalTensor<float> c1Local = outQueueCO1.AllocTensor<float>();

    AscendC::MmadParams mmadParams;
    mmadParams.m = m;
    mmadParams.n = n / 2;
    mmadParams.k = k;
    AscendC::Mmad(c1Local, a2Local, b2Local, mmadParams);

    outQueueCO1.EnQue<float>(c1Local);
    inQueueB2.FreeTensor(b2Local);
}
```

步骤5 Stage4: Aggregate函数实现，完成数据汇聚操作。

1. Aggregate需要传入两个参数：使用AllocTensor从CO2的Queue中申请的c2Local和循环迭代变量index。
2. 使用DeQue从CO1中取出c1Local。
3. 使用DataCopy将结果矩阵从CO1搬运到CO2。

DataCopy参数设置如下：
- blockCount设置为1，blockLen设置为2，连续搬运两个分形，无需格式转换。
- blockMode设置为BlockMode::BLOCK_MODE_MATRIX，表示需要按分形搬运。
- c2Local首地址偏移量设置为index * cSize / 2。

具体代码如下：

```cpp
__aicore__ inline void Aggregate(const AscendC::LocalTensor<float>& c2Local, const int bSplitIdx)
{
    AscendC::LocalTensor<float> c1Local = outQueueCO1.DeQue<float>();

    AscendC::DataCopyParams dataCopyParams;
    dataCopyParams.blockCount = 1;
    dataCopyParams.blockLen = 2;
    AscendC::DataCopyEnhancedParams enhancedParams;
    enhancedParams.blockMode = AscendC::BlockMode::BLOCK_MODE_MATRIX;
    AscendC::DataCopy(c2Local[bSplitIdx * cSize / 2], c1Local, dataCopyParams, enhancedParams);

    outQueueCO1.FreeTensor(c1Local);
}
```

步骤6 Stage5: CopyOut函数实现。

1. 使用DeQue从CO2中取出c2Local。
2. 使用DataCopy将结果矩阵从CO2搬运到Global Memory，同时需要将格式从NZ转换为ND。

每次循环移动一个分形，搬运m*16个数。DataCopy参数说明如下：
- blockCount设置为m，共搬运m次。
- blockLen设置为2，DataCopy指令一次搬运2个block，每个block16个数。
- srcStride设置为0，每两次搬运间没有间隙。
- dstStride设置为(nBlocks - 1) * 2，每两次搬运间隔2个block。
- 每次循环迭代，目的矩阵偏移16，源矩阵偏移m*16。

[图: NZ to ND 格式转换示意图]

具体代码如下：

```cpp
__aicore__ inline void CopyOut()
{
    AscendC::LocalTensor<float> c2Local = outQueueCO2.DeQue<float>();

    // transform nz to nd
    for (int i = 0; i < nBlocks; ++i) {
        AscendC::DataCopy(cGM[i * 16], c2Local[i * m * 16], { m, 2, 0, uint16_t((nBlocks - 1) * 2) });
    }

    outQueueCO2.FreeTensor(c2Local);
}
```

----结束

## 10.1.4.2 分离模式

本节内容为针对分离模式，使用基础API进行矩阵乘法的编程指导。

针对分离模式，使用基础API进行矩阵乘法算子实现的编程范式和10.1.4.1 耦合模式一致，由于硬件架构不同，具体实现有一些差异，本节仅提供差异点说明。完整代码请参见Mmad样例。

- CopyIn阶段差异
  - 耦合模式：在CopyIn阶段，即从GM->A1/B1（L1 Buffer）的阶段，耦合模式下可以使用DataCopy接口直接将数据从GM搬入L1 Buffer，也可以将数据从GM搬入UB，再搬入L1 Buffer。若需要ND2NZ的格式转换，需要开发者自行完成；或使用DataCopy接口提供的随路格式转换功能，但该功能会使用UB临时空间。
  - 分离模式：分离模式下数据无法经过VECIN/VECCALC/VECOUT（UB）直接搬运到A1/B1（L1 Buffer），但是使用DataCopy接口提供的随路格式转换功能一条指令即可完成格式转换，无需UB作为临时空间。

- Aggregate及CopyOut阶段差异
  - 耦合模式：耦合模式中，完成矩阵乘计算后数据存放于CO1（L0C Buffer），最终搬入GM需要通过CO2（UB），且NZ2ND的格式转换需要在CO1->CO2->GM阶段中手动完成。
  - 分离模式：分离模式下，矩阵乘的计算结果从CO1（L0C Buffer）可以通过Fixpipe通路直接写入GM，而且Fixpipe提供了随路NZ2ND的功能，方便用户做格式转换。

## 10.1.5 融合算子编程

### 10.1.5.1 CV融合

#### 10.1.5.1.1 基础知识

CV融合算子

融合算子由多个独立的小算子融合而成，其功能与多个小算子的功能等价，性能方面通常优于独立的小算子。用户可以根据实际业务场景诉求，按照具体算法自由融合向量（Vector）、矩阵（Cube）算子以达到性能上的收益。融合了Cube计算、Vector计算的算子统称为CV融合算子。

比如对于LLM大模型中最核心的一个融合算子Flash Attention，其核心实现如下图。图中的MatMul算子（Cube）、Scale算子（Vector）、Mask算子（Vector）、SoftMax算子（Vector）融合为一个大的Flash Attention。

[图: Flash Attention 核心实现]

使用场景和优势

针对有数据依赖的矢量算子和矩阵算子，可以通过融合算子编程的方式，将矢量算子和矩阵算子进行融合，通过一个算子Kernel函数来承载，由此来获得性能上的收益。

[图: 独立矢量算子和矩阵算子、Mix融合算子的执行耗时对比]

- 独立的矢量算子和矩阵算子实现：矩阵计算后的结果需要搬运到Global Memory上，然后由Global Memory搬运到Local Memory，再进行矢量算子的计算，计算和搬运都是串行执行，另外多个算子的调度执行，会增加算子的调度耗时。
- 融合算子的实现方法：可以对数据进行切片，再通过流水的设计，使得矢量计算单元和矩阵计算单元实现并行计算；另外相比于不融合的单算子，减少了算子的调度耗时。

除了有效提升算子性能，充分发挥AI处理器的算力，融合算子还有如下优势：
- 减少计算量：融合算子可以将多个算子合并为一个，简化计算过程，减少计算量，提高计算效率。
- 减少内存占用：融合算子可以将多个算子的中间结果合并为一个，从而减少内存占用，提高内存利用率。
- 优化数据流：融合算子可以优化数据流，减少数据在不同算子之间的传输，从而提高数据处理效率。
- 简化代码实现：融合算子可以简化代码实现，减少代码量，提高代码可读性和可维护性。

总之，融合算子是一种优化计算的有效手段，可以提高计算效率和内存利用率，优化数据流，简化代码实现。

编程范式

Ascend C提供融合算子的编程范式，方便开发者基于该范式表达融合算子的数据流，快速实现自定义的融合算子。

融合算子数据流指融合算子的输入输出在各存储位置间的流向。以一个典型的Cube和Vector融合算子为例，逻辑位置间的数据流向如下图所示：

- Cube的输出可以作为Vector的输入：CO2->VECIN
- Vector的输出可以作为Cube的输入：VECOUT->A1->A2、VECOUT->B1->B2

[图: 融合算子编程范式 - AIC(MatMul) -> AIV(Vec)]

基于Matmul高阶API的融合算子编程范式，对上述数据流简化表达如下：

1. 初始化一个MatMul对象，将输入数据从Global Memory搬运到Cube核上。
2. 进行MatMul内部的计算。
3. 将MatMul的计算结果搬运到Vector核上。
4. 进行Vector矢量计算。
5. 将输出结果搬运到Global Memory上。

整个过程的示例代码如下（伪代码）。完整样例请参考MatmulLeakyRelu。

```cpp
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::Process()
{
    // 步骤1：初始化一个MatMul对象，将输入数据从Global Memory搬运到Cube核上。
    uint32_t computeRound = 0;
    REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), matmulObj);
    matmulObj.Init(&tiling);
    matmulObj.SetTensorA(aGlobal);
    matmulObj.SetTensorB(bGlobal);
    matmulObj.SetBias(biasGlobal);

    while (matmulObj.template Iterate<true>()) { // 步骤2：进行MatMul内部的计算。
        // 步骤3：将MatMul的计算结果搬运到Vector核上。
        reluOutLocal = reluOutQueue_.AllocTensor<cType>();
        matmulObj.template GetTensorC<true>(reluOutLocal, false, true);
        // 步骤4：进行Vector矢量计算。
        AscendC::LeakyRelu(reluOutLocal, reluOutLocal, (cType)alpha, tiling.baseM * tiling.baseN);
        reluOutQueue_.EnQue(reluOutLocal);
        // 步骤5：将输出结果搬运到Global Memory上
        reluOutLocal = reluOutQueue_.DeQue<cType>();
        ...
        AscendC::DataCopy(cGlobal[startOffset], reluOutLocal, copyParam);
        reluOutQueue_.FreeTensor(reluOutLocal);

        computeRound++;
    }
    matmulObj.End();
}
```

#### 10.1.5.1.2 算子实现

下文将以Matmul+LeakyRelu融合算子的实现为例，介绍Mix融合算子的设计和实现流程。该样例仅支持在Atlas A2 训练系列产品/Atlas A2 推理系列产品上运行。

算子的设计过程分为算子分析、数据流分析、Tiling策略设计三部分。

算子分析

算子分析是指明确算子的数学表达式、输入、输出，核函数的名称等信息。

步骤1 明确算子的数学表达式及计算逻辑。该算子的计算逻辑为，先进行一个矩阵乘操作，然后将矩阵乘的结果与一个alpha参数进行LeakyRelu操作。数学表达式如下：
c = LeakyRelu(a * b + bias, alpha);

步骤2 明确输入和输出。
- MatMul+LeakyRelu算子输入为a、b、bias，输出为c。alpha作为激活函数LeakyRelu的系数，为固定值，可以在算子实现中直接使用常数值参与计算。
- 本样例中算子输入a、b支持的数据类型为half（float16），算子输入bias支持的数据类型为float32，算子输出c的数据类型为float32。
- 输入矩阵a的形状为[M, K]，输入矩阵b的形状为[K, N]，输出矩阵c的形状为[M, N]，输入bias的形状为[1, N]。
- 算子输入输出支持的数据格式为：ND。

步骤3 确定核函数名称和参数。
- 您可以自定义核函数名称，本样例中核函数命名为matmul_leakyrelu_custom。
- 根据对算子输入输出的分析，确定核函数的参数a，b，bias，c；a，b，bias为输入在Global Memory上的内存地址，c为输出在Global Memory上的内存地址。

----结束

表 10-10 Ascend C MatMul+LeakyRelu 算子设计规格

| 算子类型（OpType） | MATMUL_LEAKYRELU |
| --- | --- |
| 算子输入 | name: a, shape: [M, K], data type: half, format: ND |
|  | name: b, shape: [K, N], data type: half, format: ND |
|  | name: bias, shape: [1, N], data type: float32, format: - |
| 算子输出 | name: c, shape: [M, N], data type: float32, format: ND |
| 核函数名称 | matmul_leakyrelu_custom |

数据流分析

进行算子的数据流分析：数据流向为在Cube核上完成Matmul计算后将数据搬运至Vector核进行LeakyRelu计算。根据上述数据流并结合融合算子的编程范式，规划并行的流水任务：如下图所示：

[图: 数据流分析 - Stage1~Stage5流水]

步骤1 将输入数据从Global Memory搬运到Cube核。
步骤2 进行MatMul内部的计算，计算公式和计算示意图如下：
注：bias的shape为[1, N]，对A*B结果矩阵的每一行都采用该bias进行偏置。

[图: Matmul矩阵乘示意图]

步骤3 将MatMul的计算结果搬运到Vector核。
步骤4 进行Vector矢量计算，该样例中进行LeakyReLU计算。

Leaky ReLU（带泄露线性整流函数）激活函数，是人工神经网络中一种常用的激活函数，其数学表达式和函数图像如下所示：

f(x_i) = x_i  if x_i >= 0
f(x_i) = a*x_i  if x_i < 0

[图: Leaky ReLU函数图像]

步骤5 将输出结果搬运到Global Memory。

----结束

前三步的内容都封装在Matmul高阶API内，本样例中可以简化为3个stage。如下图所示：

[图: MatMul+LeakyRelu简化为3个stage - Stage1: MatMul Compute, Stage2: LeakyRelu Compute, Stage3: CopyOut]

根据上述分析，明确实现过程中会使用到Matmul高阶API接口，LeakyRelu Vector计算接口、DataCopy、EnQue、DeQue接口。

Tiling策略设计

Tiling策略的设计主要包括多核切分和核内切分策略。

- 多核切分: 根据当前核数，对输入shape的M, K, N进行多核切分，得到单核内shape大小singleCoreM, singleCoreK, singleCoreN。
- 核内切分: 根据Local Memory的大小约束，对单核内的shape大小进一步切分，得到A、B、C矩阵参与一次矩阵乘指令的shape大小baseM, baseN, baseK。切分时需要注意：GetTensorC的结果如果放在LocalMemory（UB）上，需要注意，baseM * baseN的大小不能超出UB的限制。

[图: Tiling切分策略示意图 - TensorA(singleCoreK x M with baseK, baseM), TensorB(singleCoreN x K with baseN, baseK), TensorC(N x M)]

算子实现

在矩阵编程章节，我们得知Ascend C提供一组Matmul高阶API，封装了常用的切分和数据搬运、计算的算法逻辑，方便用户快速实现Matmul矩阵乘法的运算操作。融合算子中矩阵编程部分的实现与之类似，开发者在host侧通过调用API自动获取Tiling参数，该参数传递到kernel侧后，在初始化操作时传入，通过几个简单的API即可完成矩阵乘操作。再结合上文的融合算子的编程范式，融合算子实现的步骤如下。完整样例请参考MatmulLeakyRelu。

[图: 融合算子实现流程图 - host侧(创建Tiling对象→设置A/B/C/Bias数据类型格式→设置矩阵shape信息→设置可用空间大小信息→按需设置其他信息→获取Tiling参数) / kernel侧(Stage1:MatMul Compute→Stage2:LeakyRelu Compute→Stage3:CopyOut) / 详细流程(将输入数据从GM搬运到Cube核→Matmul计算→将Matmul计算结果搬运到Vector核上)]

kernel侧实现的代码框架如下，在完成Matmul对象的初始化、左矩阵A、右矩阵B、Bias的设置后，通过单次Iterate叠加while循环的方式完成后续的MatMul计算、LeakyRelu计算、CopyOut流程。

```cpp
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::Process(){
    uint32_t computeRound = 0;
    // Matmul对象初始化
    REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), matmulObj);
    // 设置Matmul的输入（包括左矩阵、右矩阵、bias）
    matmulObj.Init(&tiling);
    matmulObj.SetTensorA(aGlobal);
    matmulObj.SetTensorB(bGlobal);
    matmulObj.SetBias(biasGlobal);
    // 调用matmul iterate获取一块[baseM, baseN]的计算结果
    while (matmulObj.template Iterate<true>())
    {
        MatmulCompute();
        LeakyReluCompute();
        CopyOut(computeRound);
        computeRound++;
    }
    matmulObj.End();
}
```

Matmul计算、LeakyRelu计算、CopyOut的具体实现代码如下：

步骤1 Matmul计算：
1. 将输入数据从Global Memory搬运到Cube核。
2. 进行MatMul内部的计算。
3. 将MatMul的计算结果搬运到Vector核。

```cpp
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::Process(){
    uint32_t computeRound = 0;
    // ...
    // 调用matmul iterate获取一块[baseM, baseN]的计算结果
    while (matmulObj.template Iterate<true>())
    {
        MatmulCompute();
        // ...
        computeRound++;
    }
    matmulObj.End();
}

template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::MatmulCompute(){
    reluOutLocal = reluOutQueue_AllocTensor<cType>();
    // 调用GetTensorC将Matmul的计算结果搬运到Vector核。
    matmulObj.template GetTensorC<true>(reluOutLocal, false, true);
}
```

步骤2 LeakyRelu计算。

```cpp
// 调用LeakyRule接口进行计算
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::LeakyReluCompute(){
    AscendC::LeakyRelu(reluOutLocal, reluOutLocal, (cType)alpha, tiling.baseM * tiling.baseN);
    reluOutQueue_EnQue(reluOutLocal);
}
```

步骤3 CopyOut，将输出结果搬运到Global Memory。

```cpp
// 将结果搬出到GM
template<typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::CopyOut(uint32_t count){
    reluOutQueue_DeQue<cType>();
    const uint32_t roundM = tiling.singleCoreM / tiling.baseM;
    const uint32_t roundN = tiling.singleCoreN / tiling.baseN;
    uint32_t startOffset = (count % roundM * tiling.baseM * tiling.N + count / roundM * tiling.baseN);
    AscendC::DataCopyParams copyParam = {(uint16_t)tiling.baseM,
        (uint16_t)(tiling.baseN * sizeof(cType) / DEFAULT_C0_SIZE), 0,
        (uint16_t)((tiling.N - tiling.baseN) * sizeof(cType) / DEFAULT_C0_SIZE)};
    AscendC::DataCopy(cGlobal[startOffset], reluOutLocal, copyParam);
    reluOutQueue_FreeTensor(reluOutLocal);
}
```

----结束

host侧自动获取Tiling参数的关键步骤介绍如下：

步骤1 创建Tiling对象。

```cpp
auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
matmul_tiling::MultiCoreMatmulTiling cubeTiling(ascendcPlatform);
```

创建对象时需要传入硬件平台信息，硬件平台信息可以通过GetPlatformInfo获取。

步骤2 设置A、B、Bias的数据类型和格式。

设置示例如下，其中TPosition::LCM是Unified Buffer上的逻辑位置，等同于TPosition::VECCALC，关于TPosition的详细内容请参考TPosition。

```cpp
cubeTiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT16);
cubeTiling.SetCType(matmul_tiling::TPosition::LCM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
cubeTiling.SetBiasType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
    matmul_tiling::DataType::DT_FLOAT);
```

步骤3 设置矩阵shape信息。

```cpp
cubeTiling.SetShape(M, N, K);
cubeTiling.SetOrgShape(M, N, K);
```

步骤4 设置可用空间大小信息。

设置Matmul计算时可用的L1 Buffer/L0C Buffer/Unified Buffer空间大小，-1表示AI处理器对应Buffer的大小。

```cpp
cubeTiling.SetBufferSpace(-1, -1, -1);
```

说明

- 特别的对于多核场景，需要通过SetDim接口设置Matmul计算所用的核数，MIX模式（包含矩阵计算和矢量计算）的设置规则如下：
  - 分离模式：Matmul API都是从AIV侧发起的，调用Iterate计算时在AIV侧只会起到通知的作用，通知AIC去做矩阵计算，计算完成后AIC告知AIV计算完成。这个架构下，SetBlockDim设置为实际计算会用到的AI Core（AIC、AIV组合）的数量，SetDim设置为实际计算会用到的AIV的数量。例如，SetBlockDim时可以设置为20，启动20个AI Core（AIC AIV的组合），SetDim设置为40，表示按照40个AIV进行切分。
  - 耦合模式：SetBlockDim加载的核数就是Matmul API实际计算会用到的核数，SetDim和SetBlockDim设置的值是一样的。
- Matmul高阶API内部实现时需要使用系统workspace，开发者需要：
  - 在host侧Tiling实现时，设置总的workspace的数值大小（包含用户workspace和系统workspace），workspace空间由框架来申请并管理。系统workspace的空间大小通过GetLibApiWorkSpaceSize获取。

```cpp
size_t userWorkspaceSize = 0;
size_t systemWorkspaceSize = ascendcPlatform.GetLibApiWorkSpaceSize();
size_t *currentWorkspace = context->GetWorkspaceSizes(1);
currentWorkspace[0] = userWorkspaceSize + systemWorkspaceSize;
```

  - 若算子工程不是自定义算子工程，也不是带有HAVE_WORKSPACE编译宏的Kernel直调算子工程，kernel侧需要在Matmul初始化前，通过SetSysWorkSpace设置系统workspace。

```cpp
// 使用Matmul时必须设置workspace空间
SetSysWorkspace(workspace);
if (GetSysWorkSpacePtr() == nullptr) {
    return;
}
```

- 上文介绍的实现方法，AIC侧和AIV侧的代码隔离和核间同步由框架来完成，开发者无需关心。除该方法外，开发者也可以选择底层编码的方式在分离模式下实现融合算子，这种方式将更加灵活。采用底层编码方式时，需要注意以下几点：
  - 通过ASCEND_IS_AIV和ASCEND_IS_AIC实现AIV和AIC代码之间的隔离。
  - 自行实现AIC和AIV核之间的同步：比如Matmul + LeakyRelu算子样例中，需要确保在AIC完成矩阵计算后，AIV再进行LeakyRelu的计算。
  - 使用高阶API Matmul时需要设置ASCENDC_CUBE_ONLY，表示仅在AIC侧调用Matmul API。
  - 使用设置Kernel类型接口设置Kernel类型为KERNEL_TYPE_MIX_xxx，同时启用AIV核和AIC核。

```cpp
#define ASCENDC_CUBE_ONLY // 指定Matmul运行在AIC上
KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_AIC_1_2); // 设置Kernel类型为KERNEL_TYPE_MIX_xxx
if ASCEND_IS_AIC {
    // ...
    // AIC核进行Matmul计算
    // AIC核完成计算后，通过AscendC::CrossCoreSetFlag<modelId, pipe>(flagId)发送同步flag
}
if ASCEND_IS_AIV {
    // ...
    // AIV核通过AscendC::CrossCoreWaitFlag(flagId)接收同步flag
    // AIV核进行LeakyRelu计算
}
```

完整样例请参考BareMix样例。

步骤5 按需设置其他参数，比如设置bias参与计算。

```cpp
cubeTiling.SetBias(true);
```

步骤6 获取Tiling参数。

```cpp
MatmulLeakyreluCustomTilingData tiling;
if (cubeTiling.GetTiling(tiling.cubeTilingData) == -1){
    return ge::GRAPH_FAILED;
}
```

步骤7 Tiling参数的序列化保存等其他操作。

----结束

### 10.1.5.2 通算融合

#### 10.1.5.2.1 基础知识

说明

本节内容为通算融合算子的理论背景和开发指导，学习本节内容之前，请确保已经掌握矩阵编程和《HCCL集合通信库使用指南》中的相关知识。

通算融合算子一般支持如下产品型号：
- Atlas A3 训练系列产品/Atlas A3 推理系列产品
- Atlas A2 训练系列产品/Atlas A2 推理系列产品

通算融合算子

相比于一般的计算或搬运类算子，通算融合算子将原本串行的通信和计算操作融合在一起，通过在算子内部进行数据切分，实现了计算和通信任务在算子内的并行执行，从而提升算子性能。通算融合算子统称为MC²算子，即Matrix Computation & Communication。

如下图所示，串行的通信算子和计算算子的理想执行耗时为两个算子执行时间的加和，而在融合通信和计算任务得到的通算融合算子内，将需要通信和计算的数据进行切分，一次通信和计算的数据量减少，整个通信和计算任务分多次进行，使得计算与通信流水并行，理论执行耗时大大缩短，从而带来性能收益。

[图: 通信计算融合前后的理论执行耗时对比示意图 - 串行执行(通信100us + 计算80us) vs 并行执行(4次通信25us与4次计算20us流水并行，减少60us)]

使用场景和优势

随着模型规模的增长，单设备上的训练和推理在计算能力、内存容量和能效等方面面临瓶颈，因此分布式并行计算成为必选技术路径。对于大模型分布式训练和推理过程中的通信和计算任务，可根据通信和计算的依赖关系分为两类：

- 弱依赖计算通信任务
  通信或计算的结果不会立即被对方使用，两者虽有依赖，但在两者中间可以调度执行其他无依赖的计算或通信任务。弱依赖计算通信任务不适用通算融合场景。

[图: 弱依赖计算通信任务示意图]

[图: 弱依赖计算通信任务的调度模拟示意图]

- 强依赖计算通信任务
  通信或计算的结果立即被对方使用，两者间存在紧密依赖关系。计算通信任务必须串行执行，在通信执行过程中，硬件计算资源闲置，将导致算力利用率低，通信成为主要性能瓶颈。强依赖计算通信任务适合融合为通算融合算子，利用通算融合技术提升性能。

[图: 强依赖计算通信任务示意图]

[图: 强依赖计算通信任务的调度模拟示意图]

通算融合技术与网络模型结构密切相关，一般而言，符合上述强依赖计算通信任务都有可能通过通算融合算子实现性能提升。

#### 10.1.5.2.2 算子实现

在通算融合类算子的实现中，通信操作使用Hccl高阶API，矩阵乘计算操作使用Matmul高阶API。关于更多集合通信的内容和相关概念请参考《HCCL集合通信库使用指南》。通算融合算子的开发过程与一般算子相同，但请注意，当前通算融合算子暂不支持Kernel直调和入图（GE图）开发，仅支持单算子API调用。

下文将以AllGatherMatmulCustom算子（简称AllGatherMatmul）的实现为例，从算子分析、数据流分析、创建算子工程、原型定义、Tiling实现、Kernel实现、编译与运行等方面介绍通算融合算子的设计和实现流程。本样例中算子的完整代码请参见AllGatherMatmul样例。该样例仅支持在Atlas A2 训练系列产品/Atlas A2 推理系列产品上运行。

算子分析

算子分析是指明确算子的数学表达式、输入、输出，核函数的名称等信息。

步骤1 明确算子的数学表达式及通信计算逻辑。

AllGatherMatmul算子实现了AllGather通信和Matmul矩阵乘法的融合。算子逻辑为：对输入的通信矩阵a做AllGather通信得到Matmul计算的左矩阵，即通信结果gather_out，将gather_out和右矩阵b做Matmul运算得到输出c。对应的数学表达式为：

```
gather_out = AllGather(a)
c = gather_out * b
```

步骤2 明确输入输出和属性。

- a、b为源操作数，a为通信的输入矩阵，形状为[M, K]；b为Matmul的右矩阵，形状为[K, N]。在样例中，M、K、N分别固定为512、5120、640。
- gather_out为目的操作数，存放AllGather通信结果，形状为[M * rankDim, K]，其中，rankDim为通信域内的卡数，在样例中固定为8。
- c为目的操作数，存放Matmul运算结果，形状为[M * rankDim, N]。
- 算子输入输出的数据类型为float16，format为：ND。
- group为算子属性，表示通信域名称，明确算子运行时所在的通信域。

步骤3 确定核函数名称和参数。

- 本样例中核函数命名为all_gather_matmul_custom。
- 根据对算子输入输出的分析，确定核函数的参数aGM, bGM, cGM, gatherOutGM; aGM, bGM为输入在Global Memory上的内存地址，cGM, gatherOutGM为输出在Global Memory上的内存地址。注意，核函数的参数和单算子API调用的参数在命名上存在区别，原因是核函数的参数是输入输出在Global Memory上的内存地址，而单算子API调用时输入输出的类型是aclTensor，两者并不完全一致。

步骤4 确定算子实现所需接口。

- 算子涉及AllGather通信，查看Ascend C API参考中的通信相关接口，需要使用Hccl高阶API来实现AllGather通信。
- 算子涉及Matmul左右矩阵在外部存储和内部存储间的数据搬运，查看Ascend C API参考中的数据搬运接口，需要使用DataCopy来实现数据搬运。
- 计算过程涉及矩阵计算操作，查看Ascend C API参考中的矩阵计算相关接口，需要使用Matmul高阶API实现矩阵乘计算。

----结束

表 10-11 AllGatherMatmulCustom算子规格

| 算子类型（OpType） | AllGatherMatmulCustom |
| --- | --- |
| 算子输入输出 | name: a, shape: [512, 5120], data type: float16, format: ND |
|  | name: b, shape: [5120, 640], data type: float16, format: ND |
|  | name: c, shape: [4096, 640], data type: float16, format: ND |
|  | name: gather_out, shape: [4096, 5120], data type: float16, format: ND |
| 算子属性 | group (char*)，Host侧标识通信域的字符串，表示通信域名称。 |
| 核函数名称 | all_gather_matmul_custom |

数据流分析

AllGatherMatmul算子的数据在卡间进行AllGather通信，在卡内进行Matmul计算，通信和计算按照数据切分后的主块、尾块分多次进行，流水互相掩盖。分析过程中，假定通信矩阵的切分策略为按M轴进行切分，切分后主块数（tileCnt）为2，尾块数（tailCnt）为1，则可得到通信计算掩盖示意图如下。

[图: AllGatherMatmul通信计算掩盖示意图]

AllGather的功能为将通信域内所有卡的输入按照卡id重新排序，然后拼接起来，再将结果发送到所有卡。因此，AllGather的结果中包含本卡输入的通信矩阵a，即本卡输入的通信矩阵a，算子无需等待这部分数据的通信完成，也无需对数据进行切分，可直接基于完整的通信矩阵a进行Matmul计算。AllGatherMatmul算子首先做本卡数据的Matmul计算，这样做的好处在于主块1的通信能与Matmul计算互相掩盖，同时，主块1、主块2、尾块1的计算无需再包括本卡数据的Matmul计算，可以减少后续主尾块的计算量，增加通信计算的掩盖率，从而提高性能。注意，不是所有的通算融合算子都适合首先进行本卡数据的Matmul计算。因为AllGatherMatmul算子的通信在计算之前，所以先进行本卡数据计算，可以实现本卡数据计算和第一次通信之间的互相掩盖。如果是计算在通信前的算子，如MatmulAllReduce，建议将本卡数据的计算放在最后，与最后一次通信互相掩盖。如下图所示。

[图: MatmulAllReduce通信计算掩盖示意图]

AllGatherMatmul算子逻辑分析：

步骤1 AI Core将要执行的通信信息写入Global Memory中的消息区，实现任务下发。消息区是特定地址的Global Memory，AI Core和AI CPU通过向其写入和轮询读取来实现消息在两者间的传递，这些操作统一封装于Hccl高阶API中。

[图: 通算融合算子通信流程示意图 - Rank0和Rank1之间通过HCCS/RoCE通信]

步骤2 AI CPU从消息区读取到所有通信任务信息，开始基于HCCS（华为缓存一致性系统，用于CPU/NPU之间的高速互联）或RoCE（承载在融合以太网上的RDMA技术，即跨越以太网的RDMA通信方式）等链路执行第一轮AllGather集合通信任务。与此同时，AI Core开始对本卡数据进行Matmul计算。

下图是通信卡数为4时，第一轮通信与本卡计算的示意图。tile 1表示图示为第一轮通信和与其相互掩盖的矩阵乘计算的处理流程。图中切分后的小矩阵中形如X-Y的数字表示它所在的数据块对应于第X张卡第Y块数据。

[图: AllGatherMatmul第一轮通信与rank0上的本卡数据矩阵乘示意图]

步骤3 AI CPU完成第一轮通信任务后，向消息区写入第一轮通信任务已完成的消息，并开始执行第二轮通信任务。同时，AI Core在完成本卡数据的Matmul计算后，通过轮询消息区等到第一轮通信任务已完成的消息，开始进行第一轮通信结果即主块1的Matmul计算。

下图是通信卡数为4时，第二轮通信与rank0计算的示意图。tile 2表示图示为第二轮通信和与其相互掩盖的矩阵乘计算的处理流程。

[图: AllGatherMatmul第二轮通信与rank0上主块1的矩阵乘示意图]

步骤4 类似步骤3，逐步完成剩余所有数据块的通信和计算。

----结束

创建算子工程

创建通算融合算子的算子工程与一般算子相同，具体请参考13.2.2 创建算子工程章节。本样例基于如下原型定义json文件，使用自定义算子工程生成工具msOpGen，为AllGatherMatmul算子创建算子工程。

```json
[
    {
        "op": "AllGatherMatmulCustom",
        "input_desc": [
            {
                "name": "a",
                "param_type": "required",
                "format": ["ND"],
                "type": ["float16"]
            },
            {
                "name": "b",
                "param_type": "required",
                "format": ["ND"],
                "type": ["float16"]
            }
        ],
        "output_desc": [
            {
                "name": "c",
                "param_type": "required",
                "format": ["ND"],
                "type": ["float16"]
            },
            {
                "name": "gather_out",
                "param_type": "required",
                "format": ["ND"],
                "type": ["float16"]
            }
        ],
        "attr": [
            {
                "name": "group",
                "type": "string",
                "default_value": "",
                "param_type": "required"
            }
        ]
    }
]
```

算子原型定义

相比于一般算子，通算融合算子在实现算子原型定义时，有如下约束：

- 必须定义一个表示算子通信域名称的属性。通信域是集合通信执行的上下文，管理对应的通信实体（例如一个NPU就是一个通信实体）和通信所需的资源。
- 必须通过原型注册中的MC2接口注册该算子为通算融合算子，并通过HcclGroup接口配置该算子的通信域名称。

AllGatherMatmul算子使用"group"属性表示该算子的通信域名称，其在算子原型中定义如下：

```cpp
this->Attr("group").AttrType(REQUIRED).String(); // "group"为通算融合算子的属性，表示通信域名称，原型定义中的String类型对应单算子API中的char*类型
...
this->MC2().HcclGroup("group"); // 将"group"属性配置为该算子的通信域名称
```

AllGatherMatmul算子的完整原型定义如下：

```cpp
namespace ops {
class AllGatherMatmulCustom : public OpDef {
public:
    explicit AllGatherMatmulCustom(const char *name) : OpDef(name)
    {
        this->Input("a")
            .ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16})
            .Format({ge::FORMAT_ND})
            .UnknownShapeFormat({ge::FORMAT_ND});
        this->Input("b")
            .ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16})
            .Format({ge::FORMAT_ND})
            .UnknownShapeFormat({ge::FORMAT_ND})
            .IgnoreContiguous();
        this->Output("c")
            .ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16})
            .Format({ge::FORMAT_ND})
            .UnknownShapeFormat({ge::FORMAT_ND});
        this->Output("gather_out")
            .ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16})
            .Format({ge::FORMAT_ND})
            .UnknownShapeFormat({ge::FORMAT_ND});

        this->Attr("group").AttrType(REQUIRED).String();

        this->AICore().SetTiling(AllGatherMatmulCustomTilingFunc); // 注册AllGatherMatmulCustomTilingFunc为Tiling入口函数
            this->AICore().AddConfig("ascendxxx"); // ascendxxx请修改为对应的昇腾AI处理器型号。
            this->MC2().HcclGroup("group");
    }
};
OP_ADD(AllGatherMatmulCustom);
}
```

Tiling实现

通算融合算子Tiling策略的设计主要包括通信切分策略、Matmul多核切分和核内切分策略。

- 通信切分策略：每轮通信数据块的大小，对通算融合算子的性能有较大影响。样例中按照主块M轴长度448对通信矩阵A的M轴进行切分。具体场景中如何确定切分策略请参考《Ascend C最佳实践》中的优秀实践 > MC²算子性能调优案例。
- Matmul多核切分和核内切分:
  - 多核切分: 根据当前核数，对输入shape的M、K、N进行多核切分，得到单核内shape大小singleCoreM、singleCoreK、singleCoreN。
  - 核内切分: 根据Local Memory的大小约束，对单核内的shape大小进一步切分，得到A、B、C矩阵参与一次矩阵乘指令的shape大小baseM、baseN、baseK。

如上所述，通信矩阵被切分为主块、尾块，主块、尾块的通信结果以及本卡数据需要分别进行Matmul计算。如下图，主块、尾块和本卡数据在M轴的长度分别为tileM、tailM和rankM，即Matmul计算时的左矩阵存在三种不同的形状，因此需要分别以通信矩阵主块、尾块和本卡数据块的大小为矩阵乘原始的输入形状，调用Matmul高阶API提供的Tiling接口，得到对应这三种形状的多核切分和核内切分策略。这里，singleCoreM、baseM等概念和相关原理的介绍请参考10.1.3.1 基础知识。

[图: AllGatherMatmul算子在rank0的矩阵乘示意图 - 本卡数据rankM, 主块tileM, 尾块tailM]

下面给出Tiling实现的关键步骤：

步骤1 定义AllGatherMatmul算子的Tiling结构体。

通信和Matmul融合得到的通算融合算子的Tiling结构体一般包括如下三个部分：

- Hccl高阶API的Tiling结构体。定义Mc2InitTiling和Mc2CcTiling参数。Mc2InitTiling参数用于初始化通信任务配置，必须定义为算子Tiling结构体的第一个参数。Mc2CcTiling为具体每个通信任务的参数配置，由于AllGatherMatmul算子中只有AllGather一个通信任务，因此仅需定义一个Mc2CcTiling参数。
- Matmul高阶API的Tiling结构体TCubeTiling。一般而言，主块、尾块和本卡数据的shape是不同的，由于TCubeTiling只能存储对一个输入形状进行Tiling计算得到的结果，因此需要分别定义主块、尾块和本卡数据块的Tiling结构体，来存放它们的多核切分和核内切分策略。
- AllGatherMatmul算子额外需要的自定义结构体AllGatherMatmulTiling。

AllGatherMatmul算子的完整Tiling结构体定义如下：

```cpp
struct AllGatherMatmulTiling {
    uint32_t rankM;      // A矩阵M轴的长度
    uint32_t rankN;      // B矩阵N轴的长度
    uint32_t rankK;      // A、B矩阵K轴的长度
    uint32_t tileNum;    // 主块个数
    uint32_t tailM;      // 尾块的M轴长度
    uint32_t tailNum;    // 尾块个数（0或1）
};

class AllGatherMatmulCustomTilingData {
public:
    Mc2InitTiling mc2InitTiling;
    Mc2CcTiling mc2CcTiling;
    TCubeTiling localTiling;
    TCubeTiling tileTiling;
    TCubeTiling tailTiling;
    AllGatherMatmulTiling cfg;
};
```

步骤2 获取AllGatherMatmul算子的Tiling结构体对象指针。

```cpp
AllGatherMatmulCustomTilingData *tiling = context->GetTilingData<AllGatherMatmulCustomTilingData>();
```

context为TilingContext的对象指针，该指针由框架自动从注册的Tiling入口函数AllGatherMatmulCustomTilingFunc传入，用于保存算子Tiling计算的上下文。在AllGatherMatmul算子的Tiling实现中，通过该上下文context获取计算Tiling所需要的输入输出shape、输入属性等参数，然后将Tiling结果（例如TilingKey、TilingData）保存至上下文中，供后续算子执行时使用。

步骤3 设置算子自定义的Tiling结构体参数。

```cpp
tiling->cfg.tileNum = rankM / TILE_M; // TILE_M在样例中为常量448，表示通信数据块切分后的主块在M轴的长度
tiling->cfg.tailM = rankM % TILE_M;
tiling->cfg.tailNum = (rankM % TILE_M == 0) ? 0 : 1;
tiling->cfg.rankM = rankM;
tiling->cfg.rankN = rankN;
tiling->cfg.rankK = rankK;
```

步骤4 设置Matmul高阶API Tiling结构体。

通过matmul_tiling::MultiCoreMatmulTiling获取TCubeTiling结构体，首先创建多核Tiling对象mmTiling，然后设置A、B、C的参数类型信息，M、N、K形状信息等，最后调用GetTiling接口，获取Tiling信息，具体方法可详见Matmul Tiling类。AllGatherMatmul算子中将上述逻辑封装为matmulTilingFunc函数，再分别根据主块、尾块和本卡数据的形状大小，调用matmulTilingFunc函数，得到对应的TCubeTiling参数。

```cpp
// 封装设置TCubeTiling结构体的函数为matmulTilingFunc
auto matmulTilingFunc = [&](int64_t m, int64_t n, int64_t k, TCubeTiling &cubeTiling) -> bool {
    matmul_tiling::MultiCoreMatmulTiling mmTiling;
    mmTiling.SetAType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
        matmul_tiling::DataType::DT_FLOAT16);
    mmTiling.SetBType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
        matmul_tiling::DataType::DT_FLOAT16);
    mmTiling.SetCType(matmul_tiling::TPosition::GM, matmul_tiling::CubeFormat::ND,
        matmul_tiling::DataType::DT_FLOAT16);
    mmTiling.SetBias(false);
    mmTiling.SetDim(aicCoreNum);
    mmTiling.SetShape(m, n, k);
    mmTiling.SetOrgShape(m, n, k);
    mmTiling.SetBufferSpace(L1_BUFFER_SIZE);
    if (mmTiling.GetTiling(cubeTiling) != 0) {
        return false;
    }
    return true;
};
// 设置本卡数据的Matmul TCubeTiling结构体
if (!matmulTilingFunc(rankM, rankN, rankK, tiling->localTiling)) {
    ERROR_LOG("Get local matmul tiling failed");
    return ge::GRAPH_FAILED;
}
// 设置主块的Matmul TCubeTiling结构体
if (!matmulTilingFunc(TILE_M, rankN, rankK, tiling->tileTiling)) {
    ERROR_LOG("Get tile matmul tiling failed");
    return ge::GRAPH_FAILED;
}
// 设置尾块的Matmul TCubeTiling结构体
if (!matmulTilingFunc(rankM % TILE_M, rankN, rankK, tiling->tailTiling)) {
    ERROR_LOG("Get tail matmul tiling failed");
    return ge::GRAPH_FAILED;
}
```

步骤5 设置Hccl高阶API Tiling结构体。

根据通信任务类型、算法配置等，创建一个Mc2CcTilingConfig类对象，通过向GetTiling方法传入算子Tiling结构体中mc2InitTiling和mc2CcTiling成员的引用，完成需要传递给Kernel侧的Mc2InitTiling参数和Mc2CcTiling参数的获取。Hccl高阶API Tiling结构体的具体使用方法详见Hccl Tiling使用说明。

```cpp
uint32_t opType = HCCL_CMD_ALLGATHER; // 设置通信任务类型
std::string algConfig = "AllGather=level0:doubling"; // 设置通信算法，该参数为预留字段，配置后不生效
uint32_t reduceType = HCCL_REDUCE_SUM; // 设置Reduce操作类型，仅对有归约的操作的通信任务有效，作为AllGather通信，可以直接配置默认值HCCL_REDUCE_SUM
AscendC::Mc2CcTilingConfig mc2CcTilingConfig(group, opType, algConfig, reduceType);
mc2CcTilingConfig.GetTiling(tiling->mc2InitTiling);
mc2CcTilingConfig.SetSkipLocalRankCopy(0); // 输出gatherOut需带有本卡的A矩阵，因此设置为0
mc2CcTilingConfig.GetTiling(tiling->mc2CcTiling);
```

----结束

Kernel实现

在AllGatherMatmul算子的Kernel实现中，需要对本卡数据、通信主块、通信尾块共三种形状的左矩阵进行Matmul运算，为避免重复代码，有必要抽象出一个通用的适用于不同输入形状的Matmul计算函数。设计该Matmul计算函数前，需要考虑Matmul计算需要的基本信息，罗列如下：

- 输入A、B矩阵和输出C矩阵的地址。
- TCubeTiling结构体：包含矩阵A、B、C的形状、数据类型等信息，以及A、B矩阵进行Matmul运算时在核间和核内的切分策略。

除了上述Matmul运算所需的信息外，为了快速实现Matmul矩阵乘法，可以使用Matmul高阶API中的Matmul对象来执行计算。如果Matmul对象在Matmul计算函数中定义，每次调用该函数时都会实例化Matmul对象并释放资源，这将导致较大的运行时开销。因此，将该对象也作为Matmul计算函数的参数，以实现对象的复用。

综上所述，在Kernel实现中定义的适用于不同输入形状的Matmul计算函数如下。其中Matmul计算函数名定义为MatmulKernel，入参aGM、bGM、cGM表示需要运算的原始输入输出矩阵的地址，入参tiling表示TCubeTiling结构体，入参mm对应Matmul高阶API的实现类。MATMUL_TYPE是特化了MatmulType模板的类型别名。

```cpp
using MATMUL_TYPE = MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half>;

__aicore__ inline void MatmulKernel(GM_ADDR aGM, GM_ADDR bGM, GM_ADDR cGM, TCubeTiling &tiling,
    Matmul<MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE> &mm)
```

MatmulKernel函数的实现步骤如下。

步骤1 TCubeTiling结构体存储了Matmul计算所需的核数，在无需计算的核上直接返回，结束计算。

```cpp
if (GetBlockIdx() >= tiling.usedCoreNum) {
    return;
}
```

步骤2 Matmul高阶API要求使用GlobalTensor作为输入输出矩阵，因此，根据函数输入的A、B、C矩阵在Global Memory的地址，分别定义aGlobal、bGlobal、cGlobal三个GlobalTensor。

```cpp
GlobalTensor<half> aGlobal, bGlobal, cGlobal;
aGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(aGM), tiling.M * tiling.Ka);
bGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(bGM), tiling.Ka * tiling.N);
cGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(cGM), tiling.M * tiling.N);
```

步骤3 为了实现多核并行，提升计算效率，将矩阵数据进行切分，切分后的数据分配到不同的核上进行处理。这里采用了不切分K轴、仅切分M、N轴的切分策略，示意图如下。在这种场景下，每个核需要计算待处理的矩阵数据相对于原始矩阵的偏移量，并将偏移后的矩阵作为传入A、B、C矩阵时的入参。同时，为支持分核后尾块数据的处理，每个核需要计算实际处理的singleCoreM、singleCoreN大小，并在下一步中通过调用Matmul高阶API进行设置。

[图: Matmul计算分核示意图]

```cpp
int mSingleBlocks = (tiling.M + tiling.singleCoreM - 1) / tiling.singleCoreM;
int mCoreIndex = GetBlockIdx() % mSingleBlocks;
int nCoreIndex = GetBlockIdx() / mSingleBlocks;
// 计算当前核需要处理的矩阵数据相对于原始矩阵的偏移
int offsetA = mCoreIndex * tiling.Ka * tiling.singleCoreM;
int offsetB = nCoreIndex * tiling.singleCoreN;
int offsetC = mCoreIndex * tiling.N * tiling.singleCoreM + nCoreIndex * tiling.singleCoreN;
// 计算当前核的singleCoreM/singleCoreN，作为后续SetTail接口的入参
int tailM = Std::min(tiling.M - mCoreIndex * tiling.singleCoreM, tiling.singleCoreM);
int tailN = Std::min(tiling.N - nCoreIndex * tiling.singleCoreN, tiling.singleCoreN);
```

步骤4 调用Matmul高阶API设置Matmul计算的原始完整的形状、当前核处理的输入输出矩阵的地址和计算的实际singleCoreM、singleCoreN的大小，并完成矩阵乘运算。

```cpp
mm.SetOrgShape(tiling.M, tiling.N, tiling.Ka, tiling.Kb);
mm.SetTensorA(aGlobal[offsetA]);
mm.SetTensorB(bGlobal[offsetB]);
mm.SetTail(tailM, tailN);
mm.IterateAll(cGlobal[offsetC]);
```

----结束

AllGatherMatmul算子的核函数定义如下，aGM、bGM、cGM、gatherOutGM参数含义如算子分析中所述，workspaceGM和tilingGM分别表示workspace空间和tiling数据在Global Memory的地址。

```cpp
extern "C" __global__ __aicore__ void all_gather_matmul_custom(GM_ADDR aGM, GM_ADDR bGM,
    GM_ADDR cGM, GM_ADDR gatherOutGM, GM_ADDR workspaceGM, GM_ADDR tilingGM)
```

下面介绍AllGatherMatmul算子主流程实现的具体步骤。

步骤1 Matmul计算依赖AIC核，因此控制算子逻辑仅运行于AIC中。通过ASCEND_IS_AIV宏，判断如果当前核为AIV核，直接返回，结束当前核的运行。

```cpp
if ASCEND_IS_AIV {
    return;
}
```

步骤2 注册算子Tiling结构体、获取Tiling，并初始化TPipe。

```cpp
REGISTER_TILING_DEFAULT(AllGatherMatmulCustomTilingData);
GET_TILING_DATA(tilingData, tilingGM);
TPipe pipe;
```

步骤3 定义并赋值后续计算所需变量。

```cpp
auto &&localTiling = tilingData.localTiling;
auto &&tileTiling = tilingData.tileTiling;
auto &&tailTiling = tilingData.tailTiling;
const auto tileNum = tilingData.cfg.tileNum;       // 主块数量
const auto tailNum = tilingData.cfg.tailNum;        // 尾块数量
const auto aTileEleCnt = tileTiling.M * tileTiling.Ka;       // 通信矩阵主块元素数
const auto aTileSize = tileTiling.M * tileTiling.Ka * sizeof(half);  // 通信矩阵主块字节数
const auto cTileSize = tileTiling.M * tileTiling.N * sizeof(half);   // 输出矩阵主块对应在输出矩阵的字节数
const auto aTailEleCnt = tailTiling.M * tailTiling.Ka;       // 通信矩阵尾块元素数
const auto aRankEleCnt = localTiling.M * localTiling.Ka;     // 单卡通信矩阵元素数
const auto aRankSize = localTiling.M * localTiling.Ka * sizeof(half); // 单卡通信矩阵字节数
const auto cRankSize = localTiling.M * localTiling.N * sizeof(half); // 单卡输出矩阵字节数
```

步骤4 初始化hccl对象并下发AllGather通信任务。

```cpp
Hccl hccl;
GM_ADDR contextGM = GetHcclContext<HCCL_GROUP_ID_0>();
hccl.InitV2(contextGM, &tilingData);
hccl.SetCcTilingV2(offsetof(AllGatherMatmulCustomTilingData, mc2CcTiling));
auto handleId = hccl.AllGather<true>(aGM, gatherOutGM, aTileEleCnt,
    HcclDataType::HCCL_DATA_TYPE_FP16, aRankEleCnt, tileNum);
auto tailHandleId = hccl.AllGather<true>(aGM + tileNum * aTileSize, gatherOutGM + tileNum * aTileSize,
    aTailEleCnt, HcclDataType::HCCL_DATA_TYPE_FP16, aRankEleCnt, tailNum);
```

步骤5 初始化Matmul对象，对本卡数据进行Matmul计算。

```cpp
Matmul<MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE> mm;
REGIST_MATMUL_OBJ(GetTPipePtr(), GetSysWorkSpacePtr(), mm);
mm.Init(&localTiling);
MatmulKernel(aGM, bGM, cGM + hccl.GetRankId() * cRankSize, localTiling, mm);
```

步骤6 逐轮等待主块的通信完成，并对其进行Matmul计算。

```cpp
auto aAddr = gatherOutGM;
auto cAddr = cGM;
mm.Init(&tileTiling);
for (uint32_t i = 0; i < tileNum; i++) {
    hccl.Wait(handleId);
    for (uint32_t rankId = 0; rankId < hccl.GetRankDim(); rankId++) {
        if (rankId == hccl.GetRankId())
            continue;
        MatmulKernel(aAddr + rankId * aRankSize, bGM, cAddr + rankId * cRankSize, tileTiling, mm);
    }
    aAddr += aTileSize;
    cAddr += cTileSize;
}
```

步骤7 等待尾块的通信完成，并对其进行Matmul计算。

```cpp
aAddr = gatherOutGM + tileNum * aTileSize;
cAddr = cGM + tileNum * cTileSize;
if (tailNum > 0) {
    mm.Init(&tailTiling);
    hccl.Wait(tailHandleId);
    for (uint32_t rankId = 0; rankId < hccl.GetRankDim(); rankId++) {
        if (rankId == hccl.GetRankId())
            continue;
        MatmulKernel(aAddr + rankId * aRankSize, bGM, cAddr + rankId * cRankSize, tailTiling, mm);
    }
}
```

步骤8 释放资源。

```cpp
mm.End();
hccl.Finalize();
```

----结束

整合前述代码，完整Kernel代码如下：

```cpp
#define ASCENDC_CUBE_ONLY
#include "kernel_operator.h"
#include "lib/matmul_intf.h"
#include "all_gather_matmul_custom_tiling.h"
using namespace AscendC;
using MATMUL_TYPE = MatmulType<AscendC::TPosition::GM, CubeFormat::ND, half>;

__aicore__ inline void MatmulKernel(GM_ADDR aGM, GM_ADDR bGM, GM_ADDR cGM, TCubeTiling &tiling,
                    Matmul<MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE> &mm)
{
    if (GetBlockIdx() >= tiling.usedCoreNum) {
        return;
    }

    GlobalTensor<half> aGlobal, bGlobal, cGlobal;
    aGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(aGM), tiling.M * tiling.Ka);
    bGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(bGM), tiling.Ka * tiling.N);
    cGlobal.SetGlobalBuffer(reinterpret_cast<__gm__ half *>(cGM), tiling.M * tiling.N);

    int mSingleBlocks = (tiling.M + tiling.singleCoreM - 1) / tiling.singleCoreM;
    int mCoreIndx = GetBlockIdx() % mSingleBlocks;
    int nCoreIndx = GetBlockIdx() / mSingleBlocks;
    int offsetA = mCoreIndx * tiling.Ka * tiling.singleCoreM;
    int offsetB = nCoreIndx * tiling.singleCoreN;
    int offsetC = mCoreIndx * tiling.N * tiling.singleCoreM + nCoreIndx * tiling.singleCoreN;
    int tailM = Std::min(tiling.M - mCoreIndx * tiling.singleCoreM, tiling.singleCoreM);
    int tailN = Std::min(tiling.N - nCoreIndx * tiling.singleCoreN, tiling.singleCoreN);

    mm.SetOrgShape(tiling.M, tiling.N, tiling.Ka, tiling.Kb);
    mm.SetTensorA(aGlobal[offsetA]);
    mm.SetTensorB(bGlobal[offsetB]);
    mm.SetTail(tailM, tailN);
    mm.IterateAll(cGlobal[offsetC]);
}

extern "C" __global__ __aicore__ void all_gather_matmul_custom(GM_ADDR aGM, GM_ADDR bGM,
    GM_ADDR cGM, GM_ADDR gatherOutGM, GM_ADDR workspaceGM, GM_ADDR tilingGM)
{
    if ASCEND_IS_AIV {
        return;
    }
    REGISTER_TILING_DEFAULT(AllGatherMatmulCustomTilingData);
    GET_TILING_DATA(tilingData, tilingGM);
    TPipe pipe;

    auto &&localTiling = tilingData.localTiling;
    auto &&tileTiling = tilingData.tileTiling;
    auto &&tailTiling = tilingData.tailTiling;
    const auto tileNum = tilingData.cfg.tileNum;
    const auto tailNum = tilingData.cfg.tailNum;
    const auto aTileEleCnt = tileTiling.M * tileTiling.Ka;
    const auto aTileSize = tileTiling.M * tileTiling.Ka * sizeof(half);
    const auto cTileSize = tileTiling.M * tileTiling.N * sizeof(half);
    const auto aTailEleCnt = tailTiling.M * tailTiling.Ka;
    const auto aRankEleCnt = localTiling.M * localTiling.Ka;
    const auto aRankSize = localTiling.M * localTiling.Ka * sizeof(half);
    const auto cRankSize = localTiling.M * localTiling.N * sizeof(half);

    Hccl hccl;
    GM_ADDR contextGM = GetHcclContext<HCCL_GROUP_ID_0>();
    hccl.InitV2(contextGM, &tilingData);
    hccl.SetCcTilingV2(offsetof(AllGatherMatmulCustomTilingData, mc2CcTiling));
    auto handleId = hccl.AllGather<true>(aGM, gatherOutGM, aTileEleCnt,
        HcclDataType::HCCL_DATA_TYPE_FP16, aRankEleCnt, tileNum);
    auto tailHandleId = hccl.AllGather<true>(aGM + tileNum * aTileSize, gatherOutGM + tileNum * aTileSize,
        aTailEleCnt, HcclDataType::HCCL_DATA_TYPE_FP16, aRankEleCnt, tailNum);

    Matmul<MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE, MATMUL_TYPE> mm;
    REGIST_MATMUL_OBJ(GetTPipePtr(), GetSysWorkSpacePtr(), mm);
    mm.Init(&localTiling);
    MatmulKernel(aGM, bGM, cGM + hccl.GetRankId() * cRankSize, localTiling, mm);

    auto aAddr = gatherOutGM;
    auto cAddr = cGM;
    mm.Init(&tileTiling);
    for (uint32_t i = 0; i < tileNum; i++) {
        hccl.Wait(handleId);
        for (uint32_t rankId = 0; rankId < hccl.GetRankDim(); rankId++) {
            if (rankId == hccl.GetRankId())
                continue;
            MatmulKernel(aAddr + rankId * aRankSize, bGM, cAddr + rankId * cRankSize, tileTiling, mm);
        }
        aAddr += aTileSize;
        cAddr += cTileSize;
    }

    aAddr = gatherOutGM + tileNum * aTileSize;
    cAddr = cGM + tileNum * cTileSize;
    if (tailNum > 0) {
        mm.Init(&tailTiling);
        hccl.Wait(tailHandleId);
        for (uint32_t rankId = 0; rankId < hccl.GetRankDim(); rankId++) {
            if (rankId == hccl.GetRankId())
                continue;
            MatmulKernel(aAddr + rankId * aRankSize, bGM, cAddr + rankId * cRankSize, tailTiling, mm);
        }
    }

    mm.End();
    hccl.Finalize();
}
```

编译和运行

下面从编译、安装、运行三个步骤对AllGatherMatmul样例作简要介绍。

步骤1 编译

参考AllGatherMatmul样例中生成自定义算子工程、编译算子的命令，运行install.sh脚本完成编译。

样例目录结构如下，AllGatherMatmulCustom目录为必要的算子实现，install.sh脚本使用msOpGen在21_all_gather_matmul_custom目录下创建一个CustomOp目录，并将算子实现文件复制到对应目录下，再调用msOpGen生成的编译入口脚本build.sh编译算子。

```
21_all_gather_matmul_custom
├── AclNNInvocation       // 通过aclnn调用的方式调用AllGatherMatmulCustom算子
├── AllGatherMatmulCustom // AllGatherMatmulCustom算子工程
├── all_gather_matmul_custom.json  // AllGatherMatmulCustom算子的原型定义json文件
├── all_gather_matmul_demo_def.h   // AllGatherMatmulCustom算子参数配置
└── install.sh            // 脚本，调用msOpGen生成自定义算子工程，并编译
```

msOpGen生成的CustomOp目录结构如下。

```
CustomOp              // msOpGen生成的AllGatherMatmul自定义算子工程
├── cmake
├── op_host           // host侧实现文件
├── op_kernel         // kernel侧实现文件
├── scripts           // 自定义算子工程打包相关脚本所在目录
├── build.sh          // 编译入口脚本
├── CMakeLists.txt    // 算子工程的CMakeLists.txt
└── CMakePresets.json // 编译配置项
```

步骤2 安装

部署自定义算子包前，请确保环境中存在自定义算子包默认部署路径的环境变量ASCEND_OPP_PATH。

```bash
echo $ASCEND_OPP_PATH
# 输出示例 /usr/local/Ascend/ascend-toolkit/latest/opp

# 若无输出，则需设置环境变量，ASCEND_INSTALL_PATH为CANN软件包安装路径
source [ASCEND_INSTALL_PATH]/bin/setenv.bash
# 例如 source /usr/local/Ascend/ascend-toolkit/latest/bin/setenv.bash
```

然后执行如下命令，切换目录为编译出的自定义算子安装包所在目录，并安装自定义算子包。

```bash
cd CustomOp/build_out
./custom_opp_<target os>_<target architecture>.run
```

命令执行成功后，自定义算子包中的相关文件将部署至环境变量ASCEND_OPP_PATH指向的vendors/customize目录中。

步骤3 运行

切换目录为AclNNInvocation目录，执行run.sh脚本运行单算子样例。

```bash
cd ../../AclNNInvocation
bash run.sh
```

样例中的AclNNInvocation目录提供了完整的单算子API调用的示例代码。完成前两个步骤自定义算子的编译部署后，会自动生成单算子API，该API可以直接在应用程序中调用。算子API的形式一般为"两段式接口"，形如：

```cpp
// 获取算子使用的workspace空间大小
aclnnStatus aclnnAllGatherMatmulCustomGetWorkspaceSize(
    const aclTensor *a,
    const aclTensor *b,
    char *group,
    const aclTensor *cOut,
    const aclTensor *gatherOut,
    uint64_t *workspaceSize,
    aclOpExecutor **executor);
// 执行算子
aclnnStatus aclnnAllGatherMatmulCustom(
    void *workspace,
    uint64_t workspaceSize,
    aclOpExecutor *executor,
    const aclrtStream stream);
```

其中aclnnAllGatherMatmulCustomGetWorkspaceSize为第一段接口，主要用于计算本次API调用计算过程中需要的workspace内存大小。按照该workspaceSize大小申请Device侧内存，然后调用第二段接口aclnnAllGatherMatmulCustom执行计算。详细内容请参考单算子API调用章节。

在通算融合场景，单算子API调用的程序中需要调用HCCL接口参考中的接口创建通信域，并在多线程上执行AllGatherMatmul算子。以下给出main函数和线程调用函数中关键步骤的代码示例，仅供参考。

```cpp
int main(int argc, char **argv)
{
    // 1.AscendCL初始化
    if (aclInit(NULL) != ACL_SUCCESS) {
        ERROR_LOG("aclInit failed");
        return FAILED;
    }
    // 2.通信域创建
    HcclComm comms[RANK_DIM]; // RANK_DIM为卡数，示例中为8
    int32_t devices[RANK_DIM];
    for (int32_t i = 0; i < RANK_DIM; i++) {
        devices[i] = i;
    }
    if (HcclCommInitAll(RANK_DIM, devices, comms) != HCCL_SUCCESS) {
        ERROR_LOG("Hccl comm init failed.");
        (void)aclFinalize();
        return FAILED;
    }
    // 3.创建多线程以在通信域的所有卡上都调用AllGatherMatmul算子
    std::vector<std::unique_ptr<std::thread>> threads(RANK_DIM);
    for (uint32_t rankId = 0; rankId < RANK_DIM; rankId++) {
        threads[rankId].reset(new(std::nothrow) std::thread(&RunOp, rankId, std::ref(comms[rankId])));
    }
    for (uint32_t rankId = 0; rankId < RANK_DIM; rankId++) {
        threads[rankId]->join();
    }
    // 4.AscendCL去初始化
    (void)aclFinalize();
    return SUCCESS;
}
```

在main函数中，通过HcclCommInitAll接口在当前进程统一创建了RANK_DIM张卡的通信域，一张卡对应后续创建的一个线程。每个线程都调用RunOp函数，该函数负责卡上运行时资源申请和单算子API的二阶段接口调用。RunOp函数的代码示例如下。

```cpp
bool RunOp(uint32_t rankId, HcclComm &comm)
{
    // 1.申请当前线程的context、stream等资源
    aclrtContext context;
    aclrtCreateContext(&context, rankId);
    aclrtStream stream;
    aclrtCreateStream(&stream);
    aclrtSetCurrentContext(context);

    // 2.获取当前线程对应卡的通信域名称
    char group[128] = {0};
    HcclGetCommName(comm, group);

    // 3.申请device侧内存存放算子的输入输出
    // ......

    // 4.计算workspace大小并申请内存
    size_t workspaceSize = 0;
    aclOpExecutor *handle = nullptr;
    auto ret = aclnnAllGatherMatmulCustomGetWorkspaceSize(a, b, group, c, gatherOut, &workspaceSize,
        &handle);
    void *workspace = nullptr;
    if (workspaceSize != 0) {
        aclrtMalloc(&workspace, workspaceSize);
    }

    // 5.执行算子
    ret = aclnnAllGatherMatmulCustom(workspace, workspaceSize, handle, stream);

    // 6.同步等待
    ret = aclrtSynchronizeStreamWithTimeout(stream, 10000); // 10000ms 流同步超时

    // 7.释放算子输入、输出和workspace等device侧内存
    // ......

    // 8.释放通信域、context、stream等资源
    (void)HcclCommDestroy(comm);
    (void)aclrtDestroyStream(stream);
    (void)aclrtDestroyContext(context);
    (void)aclrtResetDevice(rankId);
```
    return true;
}
```

----结束

#### 10.1.5.2.3 特性场景

重执行

为避免执行通信任务的环境中硬件闪断导致发生通信中断，通算融合算子可通过配置编译宏与环境变量，开启重执行能力。通算融合算子开启重执行后，AI CPU在检测到环境异常时，通过下图示意的机制，通知AI Core重新下发通信任务，避免由于硬件闪断造成的通信中断，提升通信稳定性。

当前，该能力的支持情况如下：

- Atlas A2 训练系列产品/Atlas A2 推理系列产品不支持通算融合算子的重执行。
- Atlas A3 训练系列产品/Atlas A3 推理系列产品支持通算融合算子的重执行。

[图: 通信任务重执行机制 - AI Core与AI CPU之间通过GM通信，AI CPU检测异常后通知AI Core重新下发任务]

开启重执行的条件如下：

- 通算融合算子的输出内存地址和输入内存地址不相同。
- 算子编译时，配置编译宏AICORE_EXCEPTION_RESTART，如下所示。具体的编译宏配置阶段和方式请参考支持自定义编译选项。

```
add_ops_compile_options(ALL OPTIONS -DAICORE_EXCEPTION_RESTART)
```

- 配置HCCL重执行环境变量HCCL_OP_RETRY_ENABLE，开启重执行的检测和上报能力，该环境变量的说明请参考《环境变量参考》"集合通信 > HCCL_OP_RETRY_ENABLE"。请在算子执行前设置该环境变量，具体配置如下：

```bash
# server内L0和server间L1均需配置为1，不支持跨超节点，L2配置为0。
export HCCL_OP_RETRY_ENABLE="L0:1, L1:1, L2:0"
```

注意，开启重执行后，若AI Core第一次下发通信任务后通信中断，默认只重执行一次。若需修改重执行次数或重传间隔时间，请参考《环境变量参考》"集合通信 > HCCL_OP_RETRY_PARAMS"。

## 11 概念原理和术语

### 11.1 术语表

表 11-1 术语表

| 术语/缩略语 | 含义 |
| --- | --- |
| A1 | AscendC::TPosition::A1代表设备上用于矩阵计算的逻辑内存，用于存放左矩阵，物理存储对应AI Core的L1 Buffer。 |
| A2 | AscendC::TPosition::A2代表设备上用于矩阵计算的逻辑内存，用于存放小块左矩阵（如经过分割、适配L0A Buffer容量的分块），物理存储对应AI Core的L0A Buffer。 |
| AI Core | 昇腾AI处理器的计算核，负责执行矩阵、矢量计算密集的任务。 |
| AIC | 在AI Core分离模式下，一组Cube Core和Vector Core组合中的Cube Core。 |
| AIV | 在AI Core分离模式下，一组Cube Core和Vector Core组合中的Vector Core。 |
| Ascend IR | Ascend Intermediate Representation，昇腾AI处理器专用的、用于表达计算流程的抽象数据结构。在本文档中，若无特殊说明，IR默认指代Ascend IR。 |
| B1 | AscendC::TPosition::B1代表设备上用于矩阵计算的逻辑内存，用于存放右矩阵，物理存储对应AI Core的L1 Buffer。 |
| B2 | AscendC::TPosition::B2代表设备上用于矩阵计算的逻辑内存，用于存放小块右矩阵（如经过分割、适配L0B Buffer容量的分块），物理存储对应AI Core的L0B Buffer。 |
| Block | Block在不同场景下具有多种含义，通常情况下指AI Core的逻辑核。典型场景有：AI Core逻辑核（一个Block表示一个AI Core的逻辑核，其BlockID是以0为起始的逻辑编号）；DataBlock（一个DataBlock表示一条NPU矢量计算指令处理的数据单元，大小通常为32字节，一条指令可同时处理多个DataBlock）；基本块（表示一次计算需要的典型数据块大小）。 |
| BlockID | 以0为起始的AI Core逻辑编号，可以比实际硬件核数大。 |
| BlockDim | 参与计算的逻辑AI Core核数，在调用核函数时由开发者指定，其值一般等于或大于实际物理核数。 |
| BiasTable Buffer | 偏置存储，AI Core内部物理存储单元，通常用于存储矩阵计算所需的Bias（偏置）数据，与逻辑内存AscendC::TPosition::C2相对应。 |
| Broadcast | 广播，一种张量操作机制。通过广播，较小的张量可以自动扩展以匹配较大的张量的形状。 |
| C1 | AscendC::TPosition::C1代表设备上用于矩阵计算的逻辑内存，用于存放Bias（偏置）数据，物理存储对应AI Core的L1 Buffer或Unified Buffer。 |
| C2 | AscendC::TPosition::C2代表设备上用于矩阵计算的逻辑内存，用于存放小块Bias（偏置）数据（如经过分割、适配BT Buffer容量的分块），物理存储对应AI Core的BT Buffer或L0C Buffer。 |
| C2PIPE2GM | AscendC::TPosition::C2PIPE2GM代表设备上用于矩阵计算的逻辑内存，用于存放量化参数，物理存储对应AI Core的Fixpipe Buffer。 |
| Cache Line | 缓存（DCache、ICache、L2 Cache）中的最小数据单位。 |
| Core | 拥有独立Scalar计算单元的计算核，Scalar计算单元承担了核内的指令发射等功能，也称之为核内的调度单元。 |
| CO1 | AscendC::TPosition::CO1代表设备上用于矩阵计算的逻辑内存，用于存放小块矩阵计算结果（如经过分割的矩阵计算结果分块），物理存储对应AI Core的L0C Buffer。 |
| CO2 | AscendC::TPosition::CO2代表设备上用于矩阵计算的逻辑内存，用于存放矩阵计算结果（如原始矩阵的最终计算结果），物理存储对应Global Memory或AI Core的Unified Buffer。 |
| Compute | Ascend C算子编程范式中典型的三个阶段之一，负责完成计算任务。 |
| CopyIn | Ascend C算子编程范式中典型的三个阶段之一，负责将待计算数据从Global Memory搬运到Local Memory。 |
| CopyOut | Ascend C算子编程范式中典型的三个阶段之一，负责将计算结果从Local Memory搬运到Global Memory。 |
| Core ID | AI Core核的物理编号，与实际硬件核数一一对应。 |
| Cube | AI Core上的Cube计算单元，负责执行矩阵运算。以float16数据类型为例，Cube每次执行可完成两个float16类型的16x16矩阵的乘法操作。 |
| Cube Core | 矩阵计算核，专注于矩阵计算。由Scalar调度单元、矩阵计算单元、搬运单元等组成，不包括矢量计算单元。 |
| DataBlock | 矢量计算指令处理的数据单元，大小通常为32字节，矢量计算指令执行一次，可同时处理多个DataBlock。 |
| DataBlock Stride | 矢量计算指令单次Repeat内DataBlock的间隔大小，即下次处理的起始数据地址与本次处理的起始数据地址之间的DataBlock个数。 |
| DCache | Data Cache，数据缓存。用于缓存Scalar计算单元近期可能被重复访问的数据段，以提升访问效率。 |
| Device | Device指安装了昇腾AI处理器的硬件设备，利用PCIe接口与主机Host侧连接，为Host提供神经网络计算能力。若存在多个Device，多个Device之间的内存资源不能共享。 |
| DMA | Direct Memory Access，直接内存访问单元。负责数据搬运，包括Global Memory和Local Memory之间的数据搬运以及不同层级Local Memory之间的数据搬运，包含搬运单元MTE2、MTE3等。 |
| DoubleBuff/DB | 双缓冲，并行域常用的优化方式，通过创建多个持有数据的缓冲区（Buffer）提高数据处理的并行性。 |
| Elementwise | 元素级操作是对张量的每个元素独立进行的操作。每个元素的结果仅依赖于对应的输入元素。 |
| Fixpipe | AI Core中负责将矩阵计算结果从L0C Buffer搬运到Global Memory或L1 Buffer的单元，搬运过程中随路完成量化、激活等操作。 |
| Fixpipe Buffer | AI Core内部物理存储单元，通常用于存储Fixpipe搬运过程中所需的量化参数等数据，与逻辑内存AscendC::TPosition::C2PIPE2GM相对应。 |
| Global Memory/GM | 设备端的主内存，AI Core的外部存储，用于存储大规模数据，但需要优化访问模式以提升性能。 |
| GlobalTensor | 存放Global Memory全局数据的Tensor。 |
| Host | 指与设备端Device相连接的X86服务器、ARM服务器，会利用Device提供的NN（Neural-Network）计算能力，完成业务。 |
| ICache | Instruction Cache，指令缓存。用于缓存最近或频繁使用的指令。极致性能优化时，需要关注如何降低ICache Miss（指令缓存未命中）。 |
| InferShape | 算子shape推导，仅在GE图模式时才使用。实际的网络模型生成过程中，会先进行Tensor shape以及datatype的推导。这样可以在图执行之前，就知道各Tensor的数据类型和形状，提前校验其正确性；同时提前推理出算子的输出张量描述，包括张量的形状、数据类型及数据排布格式等信息，算子构图准备阶段就可以为所有的张量静态分配内存，避免动态内存分配带来的开销。 |
| Kernel | 核函数，是Device设备上执行的并行函数。核函数通过__global__修饰，多个核并行执行相同的核函数，主要区别是不同核函数运行时具有不同的BlockID。 |
| Kernel Launch | 将kernel程序提交至硬件进行启动执行的过程。 |
| L0A Buffer | AI Core内部物理存储单元，通常用于存储矩阵计算的左矩阵，与逻辑内存AscendC::TPosition::A2相对应。 |
| L0B Buffer | AI Core内部物理存储单元，通常用于存储矩阵计算的右矩阵，与逻辑内存AscendC::TPosition::B2相对应。 |
| L0C Buffer | AI Core内部物理存储单元，通常用于存储矩阵计算的结果，与逻辑内存AscendC::TPosition::CO1相对应。 |
| L1 Buffer | AI Core内部物理存储单元，空间相对较大，通常用于缓存矩阵计算的输入数据。矩阵计算的输入一般需要从GM搬运到L1 Buffer，然后分别搬运到L0A Buffer和L0B Buffer。L1 Buffer与逻辑内存AscendC::TPosition::A1、AscendC::TPosition::B1相对应。 |
| L2 Cache | 二级缓存，专门用于存储频繁访问的数据，以便减少对Global Memory的读写。 |
| LCM | Local Cache Memory，AscendC::TPosition::LCM代表临时共享的Unified Buffer空间，与VECCALC实现同样的功能。 |
| Local Memory | AI Core的内部存储，包括L1 Buffer、L0A Buffer、L0B Buffer、L0C Buffer、Unified Buffer等存储单元。 |
| LocalTensor | 存放AI Core中Local Memory本地数据的Tensor。 |
| Mask | 用于控制矢量计算指令每次Repeat内参与计算的元素，可通过连续模式和逐比特模式两种方式进行设置。 |
| MTE1 | Memory Transfer Engine 1，AI Core的数据传递引擎，负责将数据从L1 Buffer搬运到L0A Buffer或L0B Buffer等。注意：不同硬件能力可能有差异。 |
| MTE2 | Memory Transfer Engine 2，AI Core的数据传递引擎，负责将数据从GM搬运到L1 Buffer、L0A Buffer、L0B Buffer、Unified Buffer等。注意：不同硬件能力可能有差异。 |
| MTE3 | Memory Transfer Engine 3，AI Core的数据传递引擎，负责将数据从Unified Buffer搬运到Global Memory、L1 Buffer等。注意：不同硬件能力可能有差异。 |
| NC1HWC0 | 一种五维数据格式，其中C0与硬件架构强相关，采用该格式可提升矩阵乘法的计算效率。 |
| NCHW | 按照[Batch, Channels, Height, Width]的排列顺序存储特征图数据。 |
| ND | 普通格式，N维张量。 |
| NHWC | 按照[Batch, Height, Width, Channels]的排列顺序存储特征图数据。 |
| NPU | Neural-Network Processing Unit，神经网络处理器单元。采用"数据驱动并行计算"的架构，专门用于处理人工智能应用中的大量计算任务。 |
| OP | 算子（Operator，简称OP），是深度学习算法中执行特定数学运算或操作的基础单元，例如激活函数（如ReLU）、卷积（Conv）、池化（Pooling）以及归一化（如Softmax）。通过组合这些算子，可以构建神经网络模型。 |
| OpType | 算子类型，一类算子的统称。例如，在网络中可能会出现多个Add算子，名称分别为Add1、Add2，但这类算子的OpType均为Add。 |
| Pipe | Ascend C编程范式核心概念之一，用于统一管理Device端内存等资源，一个Kernel函数必须且只能初始化一个Pipe对象。 |
| Preload | 在计算任务开始前，预先将必要的指令或数据加载到缓存中，用于减少指令或数据访问间的延迟，提高计算效率。 |
| Reduce | 减维操作，用于减少多维张量的维度。常见的减维操作包括求和、求最大值、求最小值等。 |
| Repeat | 矢量计算指令执行一次，读取8个DataBlock数据进行计算，称之为一个迭代（Repeat）。通常情况下，需要循环执行多次才能完成所有数据的读取与计算。 |
| Repeat Stride | 矢量计算指令循环执行时，下一次Repeat起始数据地址与当前Repeat起始数据地址之间的DataBlock个数。 |
| Repeat Times | 矢量计算指令循环执行的次数。 |
| Scalar | AI Core上的标量计算单元，主要负责标量数据运算和对其他单元（如MTE数据搬运单元、Vector矢量计算单元、Cube矩阵计算单元）的指令发射。 |
| SPMD | Single-Program Multiple-Data，一种并行程序设计模型，其主要思想是使用同一个程序在多个核上并行执行，但每个核处理不同数据。 |
| Tensor | Tensor张量是算子计算数据的容器，是N维数据结构，最常见的是标量、矢量或矩阵。张量的元素可以包含整数值、浮点值或字符串值。 |
| Tiling | Tiling指数据的切分和分块。计算数据量较大时，需要将数据进行多核切分、每个核也需要分块多次计算。 |
| TilingData | TilingData指数据切分和分块的相关参数（如每次搬运的块大小、循环次数）。鉴于设备端Scalar计算能力限制，一般Tiling参数在Host侧计算完成，然后传输到设备侧供Kernel函数使用。 |
| TilingFunc | 算子工程提供的在Host侧计算Tiling的默认函数。 |
| TilingKey | 用来区分Kernel函数不同版本的特例实现，不同的TilingKey会编译生成不同二进制。 |
| TPosition | Ascend C管理不同层级的物理内存时，用一种抽象的逻辑位置（TPosition）来表达各级别的存储，代替了片上物理存储的概念，达到隐藏硬件架构的目的。TPosition类型包括：VECIN、VECOUT、VECCALC、A1、A2、B1、B2、CO1、CO2等，其中VECIN、VECCALC、VECOUT主要用于矢量编程，A1、A2、B1、B2、CO1、CO2用于矩阵编程。 |
| TSCM | AscendC::TPosition::TSCM表示L1 Buffer空间对应的逻辑内存，需开发者自行管理以高效利用硬件资源，主要用于Matmul计算。比如，开发者可缓存一份TSCM数据，在不同使用场景中灵活配置为Matmul操作的A矩阵、B矩阵或Bias偏置矩阵，实现内存复用与计算效率优化。 |
| Unified Buffer/UB | AI Core内部存储单元，主要用于矢量计算，与逻辑内存AscendC::TPosition::VECIN、AscendC::TPosition::VECOUT、AscendC::TPosition::VECCALC相对应。 |
| VECCALC | Vector Calculation，AscendC::TPosition::VECCALC代表设备上用于矢量计算的逻辑内存，用于存放临时变量，物理存储对应AI Core的Unified Buffer。 |
| VECIN | Vector Input，AscendC::TPosition::VECIN代表设备上用于矢量计算的逻辑内存，用于存放矢量计算的输入数据，物理存储对应AI Core的Unified Buffer。 |
| VECOUT | Vector Output，AscendC::TPosition::VECOUT代表设备上用于矢量计算的逻辑内存，用于存放矢量计算的输出数据，物理存储对应AI Core的Unified Buffer。 |
| Vector | AI Core上的Vector计算单元，负责执行矢量运算。其算力低于Cube，但灵活度高于Cube（如支持数学中的求倒数，求平方根等）。 |
| Vector Core | 矢量计算核，专注于矢量计算，由Scalar调度单元、矢量计算单元、搬运单元等组成，不包括矩阵计算单元。 |
| Workspace | 通常情况下指一个预分配的、临时使用的Global Memory内存，用于存储中间结果或临时数据。 |
| CPU域调试 | Ascend C提供的一种孪生调试方法，在CPU上模拟设备侧Kernel函数的执行和调试，仅调试算子功能和精度。 |
| 基本块 | 一次计算需要的典型数据块大小。 |
| Kernel直调 | 一种简单直接的Kernel调用方式。完成Kernel侧算子实现和Host侧Tiling实现后，即可通过运行时接口，完成算子Kernel直调。该方式下Tiling开发不受CANN框架的限制，简单直接，多用于算子功能的快速验证。 |
| NPU域调试 | Ascend C提供的一种孪生调试方法，指基于NPU仿真软件或NPU硬件的执行和调试，仅调试算子精度/性能。 |
| Tiling下沉 | Tiling下沉是指将Tiling计算下沉至Device侧的AI CPU上执行，从而实现计算全程在Device侧高效完成。 |
| 分离模式 | AI Core的一种工作模式，其矩阵计算单元和矢量计算单元分别由独立的Scalar调度单元进行调度，并分离部署在Cube Core和Vector Core上。将Cube Core和Vector Core按照一定比例（1:N）进行组合，这样的组合视为一个AI Core，AI Core的核数以Cube Core为准。 |
| 孪生调试 | Ascend C提供的算子调试方法，支持在CPU域调试算子功能和精度和NPU域调试精度/性能。 |
| 流水任务 | Ascend C编程范式是一种流水线式的编程范式，把算子核内的处理程序，分成多个流水任务。流水任务是指单核处理程序中主程序调度的并行任务。在核函数内部，可以通过流水任务实现数据的并行处理，进一步提升性能。 |
| 连续模式 | 使用Mask控制矢量计算每次Repeat内参与计算的元素时，可选择的模式之一，表示前面连续的多少个元素参与计算。 |
| 耦合模式 | AI Core的一种工作模式，采用同一个Scalar调度单元同时调度矩阵计算单元、矢量计算单元，所有的单元部署在一个AI Core上。 |
| 融合算子 | 融合算子由多个独立的小算子融合而成，其功能与多个小算子的功能等价，性能方面通常优于独立的小算子。用户可以根据实际业务场景诉求，按照具体算法自由融合矢量（Vector）、矩阵（Cube）算子以达到性能上的收益。 |
| 算子入图 | 算子入图指通过GE图模式运行算子，在图模式下首先将所有算子构造成一张图，然后通过GE将图下发到昇腾AI处理器执行。 |
| 算子原型 | 算子原型是算子的抽象描述，定义了算子的输入、输出、属性等信息。 |
| 通算融合算子 | 通算融合算子是融合集合通信任务和计算任务的算子，在执行过程中，计算和通信任务可以实现部分流水并行，从而提升性能。 |
| 逐比特模式 | 使用Mask控制矢量计算每次Repeat内参与计算的元素时，可选择的模式之一，可以按位控制哪些元素参与计算，bit位的值为1表示参与计算，0表示不参与。 |
| 自定义算子工程 | Ascend C提供的基于msOpGen工具生成的算子工程。 |

### 11.2 神经网络和算子

#### 11.2.1 算子基本概念

算子（Operator，简称OP），是深度学习算法中执行特定数学运算或操作的基础单元，例如激活函数（如ReLU）、卷积（Conv）、池化（Pooling）以及归一化（如Softmax）。通过组合这些算子，可以构建神经网络模型。

本章节介绍算子中常用的基本概念。

算子名称（Op Name）

算子的名称，用于标识网络中的某个算子，同一网络中算子的名称需要保持唯一。如下图所示Conv1、Pool1、Conv2都是此网络中的算子名称，其中Conv1与Conv2算子的类型为Convolution，表示分别做一次卷积运算。

[图: 网络拓扑示例 - Data→Conv1→Pool1→Conv2]

算子类型（Op Type）

网络中每一个算子根据算子类型进行算子实现的匹配，相同类型算子的实现逻辑相同。在一个网络中同一类型的算子可能存在多个，例如上图中的Conv1算子与Conv2算子的类型都为Convolution。

张量（Tensor）

Tensor是算子计算数据的容器，包含如下属性信息。

表 11-2 Tensor属性信息

| 属性 | 定义 |
| --- | --- |
| 形状 | Tensor的形状，比如(10,)或者(1024, 1024)或者(2, 3, 4)等。如形状(3, 4)表示第一维有3个元素，第二维有4个元素，(3, 4)表示一个3行4列的矩阵数组。形式：(i1, i2, ..., in)，其中i1到in均为正整数。 |
| 数据类型 | 指定Tensor对象的数据类型。取值范围：float16, float32, int8, int16, int32, uint8, uint16, bfloat16, bool等。 |
| 数据排布格式 | 数据的物理排布格式，详细请参见11.2.2 数据排布格式。 |

形状（Shape）

张量的形状，以(D0, D1, ..., Dn-1)的形式表示，D0到Dn是任意的正整数。

如形状(3,4)表示第一维有3个元素，第二维有4个元素，(3,4)表示一个3行4列的矩阵数组。

形状的第一个元素对应张量最外层中括号中的元素个数，形状的第二个元素对应张量中从左边开始数第二个中括号中的元素个数，依此类推。例如：

表 11-3 张量的形状举例

| 张量 | 形状 | 描述 |
| --- | --- | --- |
| 1 | (0,) | 0维张量，也是一个标量 |
| [1,2,3] | (3,) | 1维张量 |
| [[1,2],[3,4]] | (2, 2) | 2维张量 |
| [[[1,2],[3,4]], [[5,6],[7,8]]] | (2, 2, 2) | 3维张量 |

物理含义我们应该怎么理解呢？假设我们有这样一个shape=(4, 20, 20, 3)。

假设有一些照片，每个像素点都由红/绿/蓝3色组成，即shape里面3的含义，照片的宽和高都是20，也就是20*20=400个像素，总共有4张照片，这就是shape=(4, 20, 20, 3)的物理含义。

[图: shape=(4,20,20,3)的示意图 - 4张20x20的3通道照片]

如果体现在编程上，可以简单把shape理解为操作Tensor的各层循环，比如我们要对shape=(4, 20, 20, 3)的A tensor进行操作，循环语句如下：

```
produce A {
  for (i, 0, 4) {
    for (j, 0, 20) {
      for (p, 0, 20) {
        for (q, 0, 3) {
          A[(((((i*20) + j)*20) + p)*3) + q)] = a_tensor[(((((i*20) + j)*20) + p)*3) + q)]
        }
      }
    }
  }
}
```

轴（axis）

轴是相对shape来说的，轴代表张量的shape的下标，比如张量a是一个5行6列的二维数组，即shape是(5,6)，则axis=0表示是张量中的第一维，即行。axis=1表示是张量中的第二维，即列。

例如张量数据[[[1,2],[3,4]], [[5,6],[7,8]]]，Shape为(2,2,2)，则轴0代表第一个维度的数据即[[1,2],[3,4]]与[[5,6],[7,8]]这两个矩阵，轴1代表第二个维度的数据即[1,2]、[3,4]、[5,6]、[7,8]这四个数组，轴2代表第三个维度的数据即1、2、3、4、5、6、7、8这八个数。

轴axis可以为负数，此时表示是倒数第axis个维度。

N维Tensor的轴有：0, 1, 2, ......, N-1。

[图: 轴示意图 - Shape:(4,20,20,3)对应Axis: 0 1 2 3 / -4 -3 -2 -1]

#### 11.2.2 数据排布格式

数据排布格式（Data Layout Format）是深度学习中对多维Tensor在内存中存储方式的描述。

常见的数据格式包括ND、NHWC和NCHW等，为Tensor的每个轴赋予了特定的业务语义。

除了上述NHWC和NCHW格式外，还存在一些特殊的私有数据格式，如FRACTAL_NZ（也简称NZ）、NC1HWC0、FRACTAL_Z、NDC1HWC0、FRACTAL_Z_3D等。这些格式的引入是为了满足AI Core中Cube计算单元的高性能计算需求，通过优化内存布局，这些格式能够提升计算效率。在使用矩阵乘、卷积API开发相关算子的过程中，您可以看到这些格式的具体应用。

普通格式

- ND、NHWC和NCHW
  数据排布格式最初用于表示图像在内存中的存储方式，其中常见的包括ND、NHWC和NCHW。在一般情况下，所有的Tensor都是N维的（ND），而NHWC和NCHW则是为四维Tensor中的每个轴赋予了特定的业务语义，例如高度（Height）、宽度（Width）和通道数（Channels）。

  NHWC和NCHW的主要区别在于通道（Channel）维度的位置：
  - NHWC格式中，通道维度位于最后一个位置。
  - NCHW格式中，通道维度位于高度和宽度之前。

  具体解释每个轴的含义：
  - N：Batch数量，表示图像的数目。
  - H：Height，图像的高度，即垂直方向的像素个数。
  - W：Width，图像的宽度，即水平方向的像素个数。
  - C：Channels，图像的通道数，例如彩色RGB图像的Channels为3。

  如图11-4所示，以一张格式为RGB的图片为例，NCHW中，C排列在外层，实际存储的是"RRRRRRGGGGGBBBBBB"，即同一通道的所有像素值顺序存储在一起；而NHWC中C排列在最内层，实际存储的则是"RGBRGBRGBRGBRGBRGB"，即多个通道的同一位置的像素值顺序存储在一起。

  [图: NCHW和NHWC存储示例]

  尽管存储的数据相同，但不同的存储顺序会导致数据的访问特性不一致，因此即便进行同样的运算，相应的计算性能也会不同。

- NDHWC和NCDHW
  NDHWC和NCDHW是五维Tensor，较NHWC和NCHW多了一个D的维度，D代表特征深度（Depth），表示数据在深度方向上的扩展，如视频的时间步或医学图像的深度层，因此该类格式便于在时间维度上进行卷积操作。以NDHWC为例，其数据格式如下图所示：

[图: NDHWC数据格式示意图 - N组, D=3, C, H, W维度]

矩阵乘相关特殊格式

使用Mmad基础API进行矩阵乘计算时，对矩阵输入输出的数据排布格式有一定的要求，如下图所示，要求A矩阵（位于L0A Buffer）为FRACTAL_ZZ，B矩阵（位于L0B Buffer）为FRACTAL_ZN，C矩阵（位于L0C Buffer）为FRACTAL_NZ。这些格式将矩阵划分成了一些分形（Fractal Matrix），适配Cube计算单元每次读取(16, 16) x (16, 16)的数据进行计算的硬件特点（以half数据类型为例），从而提高矩阵计算的效率。分形的大小和数据类型有关，也和所在的存储位置有关，具体可参见下文的详细介绍。

[图: 矩阵乘分形格式示意 - MatrixA x MatrixB = MatrixC]

- FRACTAL_NZ/NZ

  FRACTAL_NZ格式，简称NZ格式，是对一个Tensor最低两维（一个Tensor的所有维度，右侧为低维，左侧为高维）进行填充（pad）、拆分（reshape）和转置（transpose）操作后得到的格式。具体的转换过程如下：

  (M, N)大小的矩阵被分为M1 * N1个分形，按照column major（列优先）排布，形状如N字形；每个分形内部有M0 * N0个元素，按照row major（行优先）排布，形状如Z字形，所以这种数据格式称为NZ格式。其中，(M0, N0)表示一个分形的大小。

  通过公式表达为：
  `(..., B, M, N)->pad->(..., B, M1 * M0, N1 * N0)->reshape->(..., B, M1, M0, N1, N0)->transpose->(..., B, N1, M1, M0, N0)`

  说明

  通常情况下，NZ格式在L0C Buffer和L1 Buffer中分别用于不同的场景：
  - 在L0C Buffer中，NZ格式用于存储矩阵乘法的结果。其分形形状为16x16，包含256个元素，这种结构非常适合Cube计算单元进行高效的矩阵乘法运算。
  - 在L1 Buffer中，NZ格式被采用以便于将数据搬运到L0A Buffer和L0B Buffer时，能够方便地转换为对应的ZZ和ZN格式。此时，分形形状为16 x (32B / sizeof(Datatype))，大小为512字节。

  因此，当数据从L0C Buffer搬运到L1 Buffer时，其分形大小可能会发生变化。

  下面通过一个具体的例子来了解ND格式转换为NZ格式的过程。

  原始Tensor的Shape为(20, 28)：

  ```python
  data = [x for x in range(20 * 28)]
  data_a = data * np.ones((20 * 28), dtype="float16")
  tensor_a = data_a.reshape((20, 28))
  print(tensor_a)
  ```

  转换过程伪代码表达如下：

  ```python
  N0 = 16
  N1 = (28 + N0 - 1) // N0
  pad_n = N1 * N0 - 28
  M0 = 16
  M1 = (20 + M0 - 1) // M0
  pad_m = M1 * M0 - 20
  tensor_b = np.pad(tensor_a, [[0, pad_m], [0, pad_n]])
  tensor_b = tensor_b.reshape((M1, M0, N1, N0))
  tensor_b = tensor_b.transpose((2, 0, 1, 3))
  print(tensor_b)
  ```

  [图: ND到NZ格式转换过程示意图 - 20x28矩阵经过pad、reshape、transpose后变为NZ格式]

- FRACTAL_ZZ/ZZ

  FRACTAL_ZZ格式，简称ZZ格式，是对一个Tensor最低两维（一个Tensor的所有维度，右侧为低维，左侧为高维）进行填充（pad）、拆分（reshape）和转置（transpose）操作后得到的格式。

  (M, K)大小的矩阵被分为M1 * K1个分形，按照row major排布，形状如Z字形；每个分形内部有M0 * K0个元素，按照row major排布，形状如Z字形，所以这种数据格式称为ZZ格式。其中，(M0, K0)表示一个分形的大小，分形Shape为16 x (32B / sizeof(Datatype))，大小为512字节。

  [图: ZZ格式分形排布示意图]

  通过公式表达转换过程如下：
  `(..., B, M, K)->pad->(..., B, M1 * M0, K1 * K0)->reshape->(..., B, M1, M0, K1, K0)->transpose->(..., B, M1, K1, M0, K0)`

  对于不同的数据类型，M0和K0的大小不同：
  - 位宽为4的数据类型：M0=16，K0=64。
  - 位宽为8的数据类型：M0=16，K0=32。
  - 位宽为16的数据类型：M0=16，K0=16。
  - 位宽为32的数据类型：M0=16，K0=8。

- FRACTAL_ZN/ZN

  FRACTAL_ZN格式，简称ZN格式，是对一个Tensor最低两维（一个Tensor的所有维度，右侧为低维，左侧为高维）进行填充（pad）、拆分（reshape）和转置（transpose）操作后得到的格式。具体转换过程如下：

  (K, N)大小的矩阵被分为K1 * N1个分形，按照row major排布，形状如Z字形；每个分形内部有K0 * N0个元素，按照column major排布，形状如N字形，所以这种数据格式称为ZN格式。其中，(K0, N0)表示一个分形的大小，分形shape为(32B / sizeof(Datatype)) x 16，大小为512字节。

  [图: ZN格式分形排布示意图]

  通过公式表达转换过程如下：
  `(..., B, K, N)->pad->(..., B, K1 * K0, N1 * N0)->reshape->(..., B, K1, K0, N1, N0)->transpose->(..., B, K1, N1, K0, N0)`

  对于不同的数据类型，K0和N0的大小不同：
  - 位宽为4的数据类型：K0=64，N0=16；
  - 位宽为8的数据类型：K0=32，N0=16；
  - 位宽为16的数据类型：K0=16，N0=16；
  - 位宽为32的数据类型：K0=8，N0=16。

卷积相关特殊格式

- NC1HWC0

  昇腾AI处理器中，为了提高通用矩阵乘法（GEMM）运算数据块的访问效率，所有张量数据统一采用NC1HWC0的五维数据格式。其中C0与微架构强相关，等于AI Core中矩阵计算单元的大小，对于float16_t类型为16，对于int8_t类型则为32，这部分数据需要连续存储。

  C1=(C+C0-1)/C0。如果结果不整除，向下取整。

  NHWC/NCHW -> NC1HWC0的转换过程为：将数据在C维度进行分割，变成C1份NHWC0/NC0HW，再将C1份NHWC0/NC0HW在内存中连续排列成NC1HWC0，其格式转换示意图如下图所示。

  [图: NHWC/NCHW到NC1HWC0格式转换示意图]

  - NHWC -> NC1HWC0的转换公式如下：
    `Tensor.reshape( [N, H, W, C1, C0]).transpose( [0, 3, 1, 2, 4] )`
  - NCHW -> NC1HWC0的转换公式如下：
    `Tensor.reshape( [N, C1, C0, H, W]).transpose( [0, 1, 3, 4, 2] )`

- FRACTAL_Z

  FRACTAL_Z是用于定义卷积权重的数据格式，由FT Matrix（FT: Filter，卷积核）变换得到。FRACTAL_Z是送往Cube的最终数据格式，采用"C1HW,N1,N0,C0"的4维数据排布。

  数据有两层Tiling，如下图所示：

  [图: FRACTAL_Z数据格式示意图 - 第一层与Cube的Size相关，第二层与矩阵的Size相关]

  第一层与Cube的Size相关，数据按照列的方向连续（小n）；第二层与矩阵的Size相关，数据按照行的方向连续（大Z）。

  例如：HWCN = (2, 2, 32, 32)，将其变成FRACTAL_Z(C1HW, N1, N0, C0) = (8, 2, 16, 16)。

  HWCN变换FRACTAL_Z的过程为：
  `Tensor.padding([ [0,0], [0,0], [0,(C0-C%C0)%C0], [0,(N0-N%N0)%N0] ]).reshape( [H, W, C1, C0, N1, N0]).transpose( [2, 0, 1, 4, 5, 3] ).reshape( [C1*H*W, N1, N0, C0])`

  NCHW变换FRACTAL_Z的过程为：
  `Tensor.padding([ [0,(N0-N%N0)%N0], [0,(C0-C%C0)%C0], [0,0], [0,0] ]).reshape( [N1, N0, C1, C0, H, W,]).transpose( [2, 4, 5, 0, 1, 3] ).reshape( [C1*H*W, N1, N0, C0])`

- NDC1HWC0

  为了提高矩阵乘法运算数据块的访问效率，将NDHWC转换为NDC1HWC0格式。其中C0与微架构强相关，等于AI Core中矩阵计算单元的大小，对于float16_t类型为16，对于int8_t类型则为32，这部分数据需要连续存储。

  C1=(C+C0-1)/C0。如果结果不整除，向下取整。

  NDHWC -> NDC1HWC0的转换过程为：将数据在C维度进行分割，变成C1份NDHWC0，再将C1份NDHWC0在内存中连续排列成NDC1HWC0，其格式转换示意图如下图所示。

  [图: NDHWC到NDC1HWC0格式转换示意图]

- FRACTAL_Z_3D

  FRACTAL_Z_3D是3D卷积权重格式，例如Conv3D算子都会涉及到用这种格式来表达3D卷积的权重。

  NDHWC -> FRACTAL_Z_3D的变换过程通过公式表达如下：
  `(..., N, D, H, W, C)->pad->(..., N1 * N0, D, H, W, C1 * C0)->reshape->(..., N1, N0, D, H, W, C1, C0)->transpose->(D, C1, H, W, N1, N0, C0)->reshape->(..., D * C1 * H * W, N1, N0, C0)`

  对于不同的数据类型，C0和N0的大小不同：
  - 位宽为4的数据类型：C0=64，N0=16；
  - 位宽为8的数据类型：C0=32，N0=16；
  - 位宽为16的数据类型：C0=16，N0=16；
  - 位宽为32的数据类型：C0=8，N0=16。

  输入一个NDHWC格式的Tensor，Shape大小为(48, 2, 2, 2, 32)：

  [图: NDHWC格式Tensor示意 - N=48, D=2, C=32, W=2, H=2]

  转换后，得到FRACTAL_Z_3D格式如下所示：

  [图: FRACTAL_Z_3D格式数据排列 - N1*N0列, C0行, 按C1*H*W*C0和D*C1*H*W*C0分组]

Matmul高阶API相关格式

- BSH/SBH：B: Batch，批处理的大小；S: sequence length，序列长度；H = N * D，其中，N为head的数量，D为head的大小，此格式通常用于Matmul矩阵乘。数据排布格式如下图所示：

  [图: BSH格式(B=2, S=2, H=12)和SBH格式(S=2, B=2, H=12)的数据排布]

- BMNK：通用数据格式；B: Batch，批处理的大小；M、N、K为矩阵乘[M, K]*[K, N]的矩阵维度；其数据排布格式如下：

  [图: BMNK格式(B=2, M=3, N=4, K=4)的矩阵A和矩阵B数据排布]

- BSNGD：为原始BSH shape做reshape后的shape，S和D为单Batch的矩阵乘的M轴（或N轴）和K轴，一个SD为一个batch的计算数据，此格式通常用于Matmul矩阵乘，数据排布格式如下图所示：

  [图: BSNGD格式(B=2, S=3, N=3, G=2, D=2)的数据排布]

- SBNGD：为原始SBH shape做reshape后的shape，S和D为单Batch的矩阵乘的M轴（或N轴）和K轴，一个SD为一个Batch的计算数据，此格式通常用于Matmul矩阵乘，数据排布格式如下图所示：

  [图: SBNGD格式(S=3, B=2, N=3, G=2, D=2)的数据排布]

- BNGS1S2：一般为前两种数据排布进行矩阵乘的输出，S1S2数据连续存放，一个S1S2为一个Batch的计算数据，此格式通常用于Matmul矩阵乘，数据排布格式如下图所示：

  [图: BNGS1S2格式(B=2, N=3, G=2, S1=3, S2=2)的数据排布]

- ND_ALIGN：ND_ALIGN是ND数据格式的一种变换数据格式。输出矩阵乘的结果矩阵C时，用于配置C矩阵按照N方向32字节对齐的规则进行输出。

  ND->ND_ALIGN变换过程如下图所示，假设矩阵乘结果矩阵C的数据类型是int32_t，输出到VECOUT，原矩阵N方向没有32字节对齐，设置ND_ALIGN后则在其后补0，将其对齐到32字节。

  [图: ND到ND_ALIGN变换示意图]

- VECTOR：VECTOR是GEMV（矩阵向量乘，General Matrix-Vector Multiply）场景使用的一种数据格式，配置矩阵为VECTOR数据排布格式即代表输入数据是一个向量。

  [图: GEMV场景输入Vector格式的A矩阵示意图，M=1, K=256]

## 11.3 编程模型设计原理

Ascend C编程模型中，并行编程范式核心要素是：一组并行计算任务、通过队列实现任务之间的同步、开发者自主表达对并行计算任务和资源的调度。本节介绍编程模型的设计原理，作为扩展阅读，便于开发者更好的理解编程模型的设计思路和优势，对于后续的深度开发也会有所帮助。

每个并行任务Stage的编程范式如下：

1. 获取Local Memory的内存：调用AllocTensor申请内存，或者从上游队列DeQue一块内存数据。
2. 完成计算或者数据搬运。
3. 把上一步处理好的数据调用EnQue入队。
4. 调用FreeTensor释放不再需要的内存。

以最简单的矢量编程范式为例，在调用上述接口时，实际上会给各执行单元下发一些指令，如下图所示：

[图: Vector编程范式指令队列示例，展示搬入单元指令队列、计算单元指令队列、搬出单元指令队列的协作流程]

- EnQue/DeQue的具体处理流程：
  a. 标量执行单元读取算子指令序列
  b. 把这些指令发射到对应的执行单元的指令队列
  c. 各个执行单元并行执行这些指令序列
  d. EnQue/DeQue解决对内存的写后读问题
    - EnQue调用会发射同步指令set，发送信号激活wait
    - DeQue调用会发射同步指令wait，等待数据写入完成
    - wait需要等到set指令执行完成后才能执行否则阻塞

[图: 算子指令序列经Scalar计算单元分发到搬入单元指令队列和计算单元指令队列的详细流程图，展示SetFlag/WaitFlag同步机制]

由此可以看出，EnQue/DeQue主要解决了存在数据依赖时，并行执行单元的写后读同步控制问题。

[图: 搬入单元、计算单元、搬出单元之间的EnQue:SetFlag和Deque:WaitFlag时序图]

- AllocTensor/FreeTensor的具体处理流程
  a. 标量执行单元读取算子指令序列
  b. 把这些指令发射到对应的执行单元的指令队列
  c. 各个执行单元并行执行这些指令序列
  d. AllocTensor/FreeTensor，解决对内存的读后写问题
    - AllocTensor调用会发射同步指令wait等待内存被读完成
    - FreeTensor调用会发射同步指令set，通知内存释放，可以重复写
    - wait需要等到set指令执行完成后才能执行否则阻塞

[图: 算子指令序列经Scalar计算单元分发，展示AllocTensor的WaitFlag和FreeTensor的SetFlag同步机制]

由此可以看出，AllocTensor/FreeTensor主要解决了存在数据依赖时，并行执行单元的读后写同步控制问题。

[图: 循环第2个迭代再次写入xl时，搬入单元、计算单元、搬出单元之间AllocTensor:WaitFlag和FreeTensor:SetFlag的时序图]

通过上文的详细说明，可以看出异步并行程序需要考虑复杂的同步控制，而Ascend C编程模型将这些流程进行了封装，通过EnQue/DeQue/AllocTensor/FreeTensor这种开发者熟悉的资源控制方式来体现，达到简化编程和易于理解的目的。

## 11.4 内存访问原理

### 11.4.1 Scalar读写数据

AI Core中Scalar计算单元负责各类型的标量数据运算和程序的流程控制。根据硬件架构设计，Scalar仅支持对Global Memory和Unified Buffer的读写操作，而不支持对L1 Buffer、L0A Buffer、L0B Buffer和L0C Buffer等其他类型存储的访问。下文分别介绍了Scalar读写Global Memory和Unified Buffer的方式和Scalar读写数据时的同步机制。

#### Scalar读写Global Memory

如上图所示，Scalar读写GM数据时会经过DataCache，DataCache主要用于提高标量访存指令的执行效率，每一个AIC/AIV核内均有一个独立的DataCache。下面通过一个具体示例来讲解DataCache的具体工作机制。

[图: AI Core架构图，展示Scalar通过Cacheable路径访问DataCache(Cache Line 64B)，再经L2 Cache到Global Memory的层次结构]

globalTensor1是位于GM上的Tensor：

- 执行完GetValue(0)后，globalTensor1的前8个元素会进入DataCache，后续GetValue(1)~GetValue(7)不需要再访问GM，而可以直接从DataCache的Cache Line中读取数据，提高了标量连续访问的效率。
- 执行完SetValue(8, val)后，globalTensor1的index为8~15的元素会进入DataCache，SetValue只会修改DataCache中的Cache Line数据，同时将Cache Line的状态设置为Dirty，表明Cache Line中的数据与GM中的数据不一致。

```cpp
AscendC::GlobalTensor<int64_t> globalTensor1;
globalTensor1.SetGlobalBuffer((__gm__ int64_t *)input);
// 从0-7共计8个uint64_t类型，DataCache的Cache Line长度为64字节
// 执行完GetValue(0)后，GetValue(1)~GetValue(7)可以直接从Cache Line中读取，不需要再访问GM
globalTensor1.GetValue(0);
globalTensor1.GetValue(1);
globalTensor1.GetValue(2);
globalTensor1.GetValue(3);
globalTensor1.GetValue(4);
globalTensor1.GetValue(5);
globalTensor1.GetValue(6);
globalTensor1.GetValue(7);

// 执行完SetValue(8)后，不会修改GM上的数据，只会修改DataCache中Cache Line数据
// 同时Cache Line的状态置为dirty，dirty表示DataCache中Cache Line数据与GM中的数据不一致
int64_t val = 32;
globalTensor1.SetValue(8, val);
globalTensor1.GetValue(8);
```

根据上文的工作机制，多核间访问globalTensor1会出现数据不一致的情况，如果其余核需要获取GM数据的变化，则需要开发者手动调用DataCacheCleanAndInvalid来保证数据的一致性。

[图: globalTensor1的DataCache示意图，展示两个Cache Line(各64B)中valid/clean和valid/dirty状态]

#### Scalar读写Unified Buffer

Scalar读写Unified Buffer时，可以使用LocalTensor的SetValue和GetValue接口。示例如下：

```cpp
for (int32_t i = 0; i < 16; ++i) {
  inputLocal.SetValue(i, i); // 对inputLocal中第i个位置进行赋值为i
}

for (int32_t i = 0; i < srcLen; ++i) {
  auto element = inputLocal.GetValue(i); // 获取inputLocal中第i个位置的数值
}
```

#### Scalar读写数据时的同步

Scalar读写Global Memory/Unified Buffer时属于PIPE_S（Scalar流水）操作，当用户使用SetValue或者GetValue接口，且算子工程使能自动同步时，不需要手动插入同步事件。

如果用户关闭算子工程的自动同步功能时，则需要手动插入同步事件：

```cpp
// GetValue为Scalar操作，与后续的Duplicate存在数据依赖
// 因此Vector流水需要等待Scalar操作结束
float inputVal = srcLocal.GetValue(0);
SetFlag<HardEvent::S_V>(eventID1);
WaitFlag<HardEvent::S_V>(eventID1);
AscendC::Duplicate(dstLocal, inputVal, srcDataSize);

// SetValue为Scalar操作，与后续的数据搬运操作存在数据依赖
// 因此MTE3流水需要等待Scalar操作结束
srcLocal.SetValue(0, value);
SetFlag<HardEvent::S_MTE3>(eventID2);
WaitFlag<HardEvent::S_MTE3>(eventID2);
AscendC::DataCopy(dstGlobal, srcLocal, srcDataSize);
```

## 11.5 性能优化技术原理

### 11.5.1 DoubleBuffer

执行于AI Core上的指令队列主要包括如下几类，即Vector指令队列、Cube指令队列和MTE指令队列。不同指令队列间的相互独立性和可并行执行性，是DoubleBuffer优化机制的基石。

矢量计算CopyIn、CopyOut过程使用MTE指令队列（MTE2、MTE3），Compute过程使用Vector指令队列（V），意味着CopyIn、CopyOut过程和Compute过程是可以并行的。

如图11-7所示，考虑一个完整的数据搬运和计算过程，CopyIn过程将数据从Global Memory搬运到Local Memory，Vector计算单元完成计算后，经过CopyOut过程将计算结果搬回Global Memory。

[图: 数据搬运与Vector计算过程示意图，CopyIn/Copyout与Tensor/Compute之间的串行流程]

在此过程中，数据搬运与Vector计算串行执行，Vector计算单元无可避免存在资源闲置问题。举例而言，若CopyIn、Compute、CopyOut三阶段分别耗时t，则Vector的时间利用率仅为1/3，等待时间过长，Vector利用率严重不足。

为减少Vector等待时间，DoubleBuffer机制将待处理的数据一分为二，比如Tensor1、Tensor2。如图11-8所示，当Vector对Tensor1中数据进行Compute时，Tensor2可以执行CopyIn的过程；而当Vector切换到计算Tensor2时，Tensor1可以执行CopyOut的过程。由此，数据的进出搬运和Vector计算实现并行执行，Vector闲置问题得以有效缓解。

总体来说，DoubleBuffer是基于MTE指令队列与Vector指令队列的独立性和可并行性，通过将数据搬运与Vector计算并行执行以隐藏数据搬运时间并降低Vector指令的等待时间，最终提高Vector单元的利用效率，您可以通过为队列申请内存时设置内存块的个数来实现数据并行，简单代码示例如下：

```cpp
pipe.InitBuffer(inQueueX, 2, 256);
```

[图: DoubleBuffer机制示意图，Tensor1和Tensor2交替进行CopyIn/CopyOut与Compute]

需要注意：

多数情况下，采用DoubleBuffer能有效提升Vector的时间利用率，缩减算子执行时间。然而，DoubleBuffer机制缓解Vector闲置问题并不代表它总能带来整体的性能提升。例如：

- 当数据搬运时间较短，而Vector计算时间显著较长时，由于数据搬运在整个计算过程中的时间占比较低，DoubleBuffer机制带来的性能收益会偏小。
- 又如，当原始数据较小且Vector可一次性完成所有计算时，强行使用DoubleBuffer会降低Vector计算资源的利用率，最终效果可能适得其反。

因此，DoubleBuffer的性能收益需综合考虑Vector算力、数据量大小、搬运与计算时间占比等多种因素。

# 12 常用操作

12.1 如何开发动态输入算子

12.2 如何在矢量编程时使能Vector Core

12.3 如何使用workspace

12.4 如何进行Tiling调测

12.5 如何使用Tensor原地操作提升算子性能

## 12.1 如何开发动态输入算子

动态输入算子是指算子的输入个数是动态的，例如AddN，将N个输入tensor累加到一起，输出一个tensor，输入tensor的个数是不固定的。动态输入算子的开发在构造和解析输入数据方面有差异：核函数的入参采用ListTensorDesc的结构存储输入数据信息，对应的，调用时需构造TensorList结构保存参数信息。下面基于kernel直调和工程化算子开发两种开发方式分别介绍具体开发流程。

> 说明：下文仅列出代码片段，完整样例请参考动态输入算子样例（工程化算子开发）和动态输入算子样例（kernel直调）。

- kernel直调
  - 参考ListTensorDesc数据结构自行定义ListTensorDesc和TensorDesc结构体，并将实际的输入数据保存至ListTensorDesc结构中。示例如下：

    ptrOffset传入为ListTensorDesc首地址和数据指针首地址dataPtr之间的偏移量，tensorDesc中保存两个输入的tensor描述信息，dataPtr传入为保存输入数据的地址指针。

    ```cpp
    constexpr uint32_t SHAPE_DIM = 2;
    struct TensorDesc {
      uint32_t dim[SHAPE_DIM];
      uint32_t index;
      uint64_t shape[SHAPE_DIM] = {8, 2048};
    };

    TensorDesc xDesc;
    xDesc.index = 0;
    TensorDesc yDesc;
    yDesc.index = 1;
    ```

    ```cpp
    constexpr uint32_t TENSOR_DESC_NUM = 2;
    struct ListTensorDesc {
      uint64_t ptrOffset;
      TensorDesc tensorDesc[TENSOR_DESC_NUM];
      uintptr_t dataPtr[TENSOR_DESC_NUM];
    } inputDesc;
    ...
    inputDesc = {(1 + (1 + SHAPE_DIM) * TENSOR_DESC_NUM) * sizeof(uint64_t), {xDesc, yDesc},
    {(uintptr_t)xDevice, (uintptr_t)yDevice}};
    ```

  - kernel侧调用时，直接传入ListTensorDesc表达的输入信息。示例如下：

    ```cpp
    void *inputDescInDevice = nullptr;
    CHECK_ACL(aclrtMalloc((void **)&inputDescInDevice, sizeof(ListTensorDesc),
    ACL_MEM_MALLOC_HUGE_FIRST));
    CHECK_ACL(aclrtMemcpy(inputDescInDevice, sizeof(ListTensorDesc), &inputDesc,
    sizeof(ListTensorDesc),
                ACL_MEMCPY_HOST_TO_DEVICE));

    ACLRT_LAUNCH_KERNEL(addn_custom)(blockDim, stream, inputDescInDevice, zDevice);
    ```

  - kernel侧算子实现，通过ListTensorDesc和TensorDesc提供的接口解析ListTensorDesc输入信息，并处理。示例如下：

    ```cpp
    uint64_t buf[SHAPE_DIM] = {0};
    AscendC::TensorDesc<int32_t> tensorDesc;
    tensorDesc.SetShapeAddr(buf);
    listTensorDesc.GetDesc(tensorDesc, 0);
    uint64_t totalLength = tensorDesc.GetShape(0) * tensorDesc.GetShape(1);
    __gm__ uint8_t *x = listTensorDesc.GetDataPtr<__gm__ uint8_t>(0);
    __gm__ uint8_t *y = listTensorDesc.GetDataPtr<__gm__ uint8_t>(1);
    ```

- 工程化算子开发
  - 单算子调用时，构造List类型tensor并传入。
    使用aclCreateTensor创建tensor后，需调用aclCreateTensorList，将创建好的tensor组成List形式，如下所示。

    ```cpp
    inputTensorList = aclCreateTensorList(inputTensor_.data(), inputTensor_.size());
    ```

    获取算子使用的workspace空间大小接口的入参，也需使用aclTensorList结构参数，用来计算workspace的大小，调用示例如下。

    ```cpp
    // 获取算子使用的workspace空间大小
    aclnnStatus aclnnAddNCustomGetWorkspaceSize(const aclTensorList *srcList, const aclTensor
    *out, uint64_t *workspaceSize, aclOpExecutor **executor);
    ```

  - 算子原型定义中，输入数据的参数类型设置为动态，示例如下。

    ```cpp
    this->Input("srcList")
      .ParamType(DYNAMIC)
      .DataType({ge::DT_FLOAT16})
      .Format({ge::FORMAT_ND});
    ```

  - host侧算子实现，获取动态输入信息的接口，需使用对应的动态接口。

    例如，Tiling函数和InferShape函数中，GetDynamicInputShape接口用于获取动态输入的shape信息，InferDataType函数中，GetDynamicInputDataType接口用于获取动态输入的数据类型，示例如下。

    ```cpp
    namespace ge {
    static graphStatus InferShape(gert::InferShapeContext *context)
    {
      const gert::Shape *x1_shape = context->GetDynamicInputShape(0, 0);
      gert::Shape *y_shape = context->GetOutputShape(0);
      *y_shape = *x1_shape;
      return GRAPH_SUCCESS;
    }

    static graphStatus InferDataType(gert::InferDataTypeContext *context)
    {
      const auto inputDataType = context->GetDynamicInputDataType(0, 0);
      context->SetOutputDataType(0, inputDataType);
      return ge::GRAPH_SUCCESS;
    }
    ```

  - kernel侧算子实现，入参需传入动态结构的数据，并使用AscendC::ListTensorDesc结构做解析。

    核函数入参需传入动态结构的数据，例如GM_ADDR srcList，示例如下。

    ```cpp
    extern "C" __global__ __aicore__ void addn_custom(GM_ADDR srcList, GM_ADDR z, GM_ADDR
    workspace, GM_ADDR tiling)
    ```

    对传入的参数srcList，需使用AscendC::ListTensorDesc结构做解析，得到每个tensor的具体信息，示例如下。

    ```cpp
    AscendC::ListTensorDesc keyListTensorDescInit((__gm__ void*)srcList);
    GM_ADDR x = (__gm__ uint8_t*)keyListTensorDescInit.GetDataPtr<__gm__ uint8_t>(0);
    GM_ADDR y = (__gm__ uint8_t*)keyListTensorDescInit.GetDataPtr<__gm__ uint8_t>(1);
    ```

## 12.2 如何在矢量编程时使能Vector Core

针对Atlas推理系列产品，其硬件架构除了AI Core外，还额外设置了单独的Vector Core，作为AI Core中Vector计算单元的补充，从而缓解Vector计算瓶颈。Vector Core只包括了两种基础计算资源：向量计算单元（Vector Unit）和标量计算单元（Scalar Unit），分别用于完成向量与标量的数据计算。矢量算子开发时，使能Vector Core，算子执行时会同时启动AI Core和Vector Core，这些核并行执行相同的核函数代码。

本节将重点介绍如何使能Atlas推理系列产品中的Vector Core。学习本节内容之前，建议您先熟悉算子实现、13.1基于样例工程完成Kernel直调、13.2工程化算子开发的相关内容，掌握基于AI Core的算子端到端开发流程。在此基础上本章将重点阐述使能Vector Core时的差异点。具体如下：

1. 完成算子kernel侧开发时，需要通过宏KERNEL_TASK_TYPE_DEFAULT使能Vector Core，算子执行时会同时启动AI Core和Vector Core，此时AI Core会当成Vector Core使用。如下的代码样例展示了使能Vector Core的方法：

```cpp
extern "C" __global__ __aicore__ void add_custom(__gm__ uint8_t *x, __gm__ uint8_t *y, __gm__
uint8_t *z, __gm__ uint8_t *workspace, __gm__ uint8_t *tiling)
{
  GET_TILING_DATA(tilingData, tiling);
  if (workspace == nullptr) {
    return;
  }
  GM_ADDR usr = AscendC::GetUserWorkspace(workspace);
  KernelAdd op;
  op.Init(x, y, z, tilingData.blockDim, tilingData.totalLength, tilingData.tileNum);
  KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_VECTOR_CORE); // 使能VectorCore
  if (TILING_KEY_IS(1)) {
    op.Process1();
  } else if (TILING_KEY_IS(2)) {
    op.Process2();
  }
  // ...
}
```

2. 完成host侧tiling开发时，设置的blockDim代表的是AI Core和Vector Core的总数，比如用户在host侧设置blockDim为10，则会启动总数为10的AI Core和Vector Core；为保证启动Vector Core，设置数值应大于AI Core的核数。您可以通过GetCoreNumAic接口获取AI Core的核数，GetCoreNumVector接口获取Vector Core的核数。如下代码片段，分别为使用kernel直调工程和自定义算子工程时的设置样例，此处设置为AI Core和Vector Core的总和，表示所有AI Core和Vector Core都启动。

  - kernel直调工程

    ```cpp
    auto ascendcPlatform = platform_ascendc::PlatformAscendCManager::GetInstance();
    auto totalCoreNum = ascendcPlatform.GetCoreNumAic();
    // ASCENDXXX请替换为实际的版本型号
    if (ascendcPlatform.GetSocVersion() == platform_ascendc::SocVersion::ASCENDXXX) {
      totalCoreNum = totalCoreNum + ascendcPlatform.GetCoreNumVector();
    }
    ...
    kernel_name<<<totalCoreNum, l2ctrl, stream>>>(argument list);
    ```

  - 自定义算子工程

    ```cpp
    // 配套的host侧tiling函数示例：
    ge::graphStatus TilingFunc(gert::TilingContext* context)
    {
      // 使能VectorCore，将blockDim置为AI Core中vector核数 + Vector Core中的vector核数
      auto ascendcPlatform = platform_ascendc::PlatformAscendC(platformInfo);
      auto totalCoreNum = ascendcPlatform.GetCoreNumAic();
      // ASCENDXXX请替换为实际的版本型号
      if (ascendcPlatform.GetSocVersion() == platform_ascendc::SocVersion::ASCENDXXX) {
        totalCoreNum = totalCoreNum + ascendcPlatform.GetCoreNumVector();
      }
      context->SetBlockDim(totalCoreNum);
    }
    ```

> 说明
> - 请参考Ascend C API中具体API支持的型号，来判断API接口是否支持Atlas推理系列产品Vector Core。
> - 支持Vector Core后，因为AI Core和Vector Core会分别执行，通过不同的任务进行调度，所以不支持核间同步指令，如IBSet、IBWait、SyncAll等。
> - 算子计算溢出（输入inf/nan或计算结果超出范围）时，需注意AI Core和Vector Core结果表现不一致，AI Core仅支持饱和模式，Vector Core仅支持inf/nan模式。

## 12.3 如何使用workspace

workspace是设备侧Global Memory上的一块内存。workspace内存分为两部分：系统workspace和用户workspace。

- 系统workspace：Ascend C API需要预留的workspace内存
  API在计算过程需要一些workspace内存作为缓存，因此算子需要为API预留workspace内存，预留内存大小通过GetLibApiWorkSpaceSize接口获取。

- 用户workspace：算子实现使用到的workspace内存
  算子内部需要通过额外的device内存进行数据交换或者缓存的时候才需要分配，根据实际情况自行分配。使用场景如下：
  - 需要使用Unified Buffer和L1 Buffer上的空间且空间不够用时，可以将数据暂存至workspace上。
  - 调用SyncAll等API接口时，需要workspace作为入参。
  - 其他需要使用Global Memory上内存空间的场景。

不同开发方式下，具体的使用方法如下：

- 工程化算子开发方式

  在tiling函数中先通过GetWorkspaceSizes接口获取workspace大小的存放位置，再设置workspace的大小，框架侧会为其申请对应大小的设备侧Global Memory，在对应的算子kernel侧实现时可以使用这块workspace内存。在使用Matmul Kernel侧接口等需要系统workspace的高阶API时，设置的workspace空间大小为系统workspace和用户workspace之和。

  ```cpp
  // 用户自定义的tiling函数
  static ge::graphStatus TilingFunc(gert::TilingContext* context)
  {
    AddApiTiling tiling;
    ...
    size_t usrSize = 256; // 设置用户需要使用的workspace大小为256字节。
    // 如需要使用系统workspace需要调用GetLibApiWorkSpaceSize获取系统workspace的大小。
    auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
    uint32_t sysWorkspaceSize = ascendcPlatform.GetLibApiWorkSpaceSize();
    size_t *currentWorkspace = context->GetWorkspaceSizes(1); // 通过框架获取workspace的指针，
  GetWorkspaceSizes入参为所需workspace的块数。当前限制使用一块。
    currentWorkspace[0] = usrSize + sysWorkspaceSize; // 设置总的workspace的数值大小，总的
  workspace空间由框架来申请并管理。
    ...
  }
  ```

  在device侧kernel入口处的workspace为用户的workspace指针：

  ```cpp
  // 用户写的Kernel函数，核函数必须包括GM_ADDR workspace入参，位置需要放在tiling之前
  extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
  workspace, GM_ADDR tiling)
  {
    ...
  }
  ```

- Kernel直调算子开发场景

  需要使用workspace空间时，建议开启编译选项HAVE_WORKSPACE。host侧开发者仍需要自行申请workspace的空间，并传入。在使用Matmul Kernel侧接口等需要系统workspace的高阶API时，设置的workspace空间大小为系统workspace和用户workspace之和。系统workspace大小可以通过PlatformAscendCManager的GetLibApiWorkSpaceSize接口获取。开启HAVE_WORKSPACE后，开发者在kernel侧入参处获取的workspace为偏移了系统workspace后的用户workspace。

## 12.4 如何进行Tiling调测

在工程化算子开发过程中，开发者需实现Tiling函数，该函数原型是固定的，接受TilingContext作为输入。框架负责构造TilingContext并调用Tiling函数。若需单独进行Tiling调测，开发者可通过OpTilingRegistry加载编译后的Tiling动态库，获取Tiling函数的指针并进行调用，调用时Tiling函数的TilingContext入参使用ContextBuilder构建。

以下是具体步骤：

步骤1 参考工程化算子开发的开发步骤，完成算子实现，并通过算子包编译或算子动态库编译获取对应的Tiling动态库文件。

- 算子包编译：Tiling实现对应的动态库为算子包部署目录下的liboptiling.so。具体路径可参考13.2.6.2 算子包部署。
- 动态库编译：Tiling实现集成在算子动态库libcust_opapi.so中。具体路径可参考13.2.7 算子动态库和静态库编译。

步骤2 编写测试代码。

- 使用ContextBuilder配置输入输出Tensor的形状、数据类型、格式及平台信息等，构建TilingContext。
- 通过OpTilingRegistry的LoadTilingLibrary接口加载Tiling动态库；使用GetTilingFunc接口获取Tiling函数指针。
- 执行Tiling函数，验证其正确性。

```cpp
// test.cpp
#include <iostream>
#include "exe_graph/runtime/storage_shape.h"
#include "tiling/context/context_builder.h"

int main()
{
  gert::StorageShape x_shape = {{2, 32}, {2, 32}};
  gert::StorageShape y_shape = {{2, 32}, {2, 32}};
  gert::StorageShape z_shape = {{2, 32}, {2, 32}};

  auto param = gert::TilingData::CreateCap(4096);
  auto workspace_size_holder = gert::ContinuousVector::Create<size_t>(4096);
  auto ws_size = reinterpret_cast<gert::ContinuousVector *>(workspace_size_holder.get());

  auto holder = context_ascendc::ContextBuilder()
              .NodeIoNum(2, 1)
              .IrInstanceNum({1, 1})
              .AddInputTd(0, ge::DT_FLOAT, ge::FORMAT_ND, ge::FORMAT_ND, x_shape)
              .AddInputTd(1, ge::DT_FLOAT, ge::FORMAT_ND, ge::FORMAT_ND, y_shape)
              .AddOutputTd(0, ge::DT_FLOAT, ge::FORMAT_ND, ge::FORMAT_ND, z_shape)
              .TilingData(param.get())
              .Workspace(ws_size)
              .AddPlatformInfo("Ascendxxxyy")
              .BuildTilingContext();
  auto tilingContext = holder->GetContext<gert::TilingContext>();
  context_ascendc::OpTilingRegistry tmpIns;
  bool flag = tmpIns.LoadTilingLibrary("/your/path/to/so_path/liboptiling.so"); // 加载对应的Tiling动态库文件
  if (flag == false) {
    std::cout << "Failed to load tiling so" << std::endl;
    return -1;
  }
  context_ascendc::TilingFunc tilingFunc = tmpIns.GetTilingFunc("AddCustom"); // 获取AddCustom算子对应的Tiling函数，此处入参为OpType
  if (tilingFunc != nullptr) {
    ge::graphStatus ret = tilingFunc(tilingContext); // 执行Tiling函数
    if (ret != ge::GRAPH_SUCCESS) {
      std::cout << "Exec tiling func failed." << std::endl;
      return -1;
    }
  } else {
    std::cout << "Get tiling func failed." << std::endl;
    return -1;
  }
  return 0;
}
```

步骤3 编译测试代码。

```bash
g++ test.cpp -I${INSTALL_DIR}/include -L${INSTALL_DIR}/lib64 -Wl,-rpath,${INSTALL_DIR}/lib64 -ltiling_api -
lc_sec -lgraph_base -lregister -lascendalog -lplatform -o test
```

- ${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：/usr/local/Ascend/ascend-toolkit/latest。
- 开发者根据需要链接依赖的动态库，必需链接的动态库有：
  - libtiling_api.so：Tiling功能相关的动态库，包含ContextBuilder类、OpTilingRegistry类等。
  - libc_sec.so：安全函数库，libtiling_api.so依赖该库。
  - libgraph_base.so：基础数据结构与接口库，libtiling_api.so依赖该库。
  - libregister.so：业务函数注册相关库（例如Tiling函数注册，算子原型注册等）。
  - libascendalog.so：log库，libtiling_api.so依赖该库。
  - libplatform.so：平台信息库，libtiling_api.so依赖该库；Tiling函数中使用硬件平台信息时，需要依赖该库。

步骤4 执行可执行文件。

```bash
./test
```

----结束

## 12.5 如何使用Tensor原地操作提升算子性能

Tensor原地操作（inplace接口）是一种优化技术，全局申请、保留LocalTensor内存，避免了频繁创建和销毁LocalTensor对象。AllocTensor、FreeTensor、EnQue、DeQue接口不产生新的LocalTensor，而是在该全局LocalTensor上反复申请、释放、入队、出队。其实现原理如下图所示：

[图: Tensor原地操作实现原理对比图。左侧non-inplace流程：TPipe.InitBuffer初始化正反向同步事件 -> LOOP中AllocTensor新建LocalTensor/EnQue将Buffer地址Push入队列/DeQue从队列Pop新建LocalTensor/FreeTensor设置反向同步事件。右侧inplace流程：TPipe.InitBuffer分配正反向同步事件并创建全局LocalTensor -> LOOP中AllocTensor使用原LocalTensor/EnQue仅设置正向同步事件/DeQue返回原LocalTensor/FreeTensor设置反向同步事件。inplace减少了额外分配、释放事件的开销和新建LocalTensor的开销]

Tensor原地操作的优势

- 减少栈变换：相比构造新Tensor的方式，inplace接口减少了LocalTensor的栈变换，允许Tensor被反复使用。
- 减少入队/出队操作：在调用EnQue、DeQue的过程中，TQue对象没有存储该Tensor对应的Buffer地址，实际没有真正入队、出队，减少了反复入队、出队的Scalar指令。

保留EnQue和DeQue的原因

既然Tensor原地操作没有执行真正的入队出队操作，为什么还需要保留EnQue和DeQue接口呢？

- 编程兼容性：为了保持编程接口的一致性，inplace接口仍然需要调用EnQue和DeQue，确保代码结构的统一性和可维护性。
- 内存同步功能：EnQue和DeQue操作中实现了内存读写同步功能，确保数据的一致性和正确性，即使没有实际的队列操作，这些同步机制仍然需要保留。

适用场景

适合计算循环次数多的场景：如图12-1所示，inplace接口虽然增加了TQue对象InitBuffer的初始化开销，但显著减少了每次循环中AllocTensor、EnQue、DeQue和FreeTensor内部对LocalTensor和事件的操作次数，特别适合需要多次循环来完成计算的场景。

使用方法

- 配置TQue对象：在创建TQue对象时，设置深度（depth）为0，启用inplace操作模式。
- 调用原地操作接口：使用inplace接口直接操作LocalTensor。
  - AllocTensor和DeQue区分non-inplace和inplace接口，详情请参考AllocTensor、DeQue。
  - FreeTensor和EnQue不区分non-inplace和inplace接口。

示例代码

```cpp
// ...
namespace AscendC {
class MyKernel {
public:
  __aicore__ inline MyKernel() {}
  __aicore__ inline void Init(__gm__ uint8_t* src0Gm, __gm__ uint8_t* src1Gm, __gm__ uint8_t* dstGm)
  {
    src0Global.SetGlobalBuffer((__gm__ half*)src0Gm);
    src1Global.SetGlobalBuffer((__gm__ half*)src1Gm);
    dstGlobal.SetGlobalBuffer((__gm__ half*)dstGm);
    pipe.InitBuffer(srcQue0, 1, BLOCK_SIZE * sizeof(half));
    pipe.InitBuffer(srcQue1, 1, BLOCK_SIZE * sizeof(half));
    pipe.InitBuffer(dstQue0, 1, BLOCK_SIZE * sizeof(half));
  }

  __aicore__ inline void Process()
  {
    for (int i = 0; i < REPTIMES; i++) {
      CopyIn(i);
      Compute(i);
      CopyOut(i);
    }
  }

private:
  __aicore__ inline void CopyIn(int32_t i)
  {
    srcQue0.AllocTensor<half>(src0Local);
    srcQue1.AllocTensor<half>(src1Local);
    DataCopy(src0Local, src0Global[i*BLOCK_SIZE], BLOCK_SIZE);
    DataCopy(src1Local, src1Global[i*BLOCK_SIZE], BLOCK_SIZE);
    srcQue0.EnQue(src0Local);
    srcQue1.EnQue(src1Local);
  }
  __aicore__ inline void Compute(int32_t i)
  {
    srcQue0.DeQue<half>(src0Local);
    srcQue1.DeQue<half>(src1Local);
    dstQue0.AllocTensor<half>(dstLocal);
    Add(dstLocal, src0Local, src1Local, BLOCK_SIZE);
    dstQue0.EnQue<half>(dstLocal);
    srcQue0.FreeTensor(src0Local);
    srcQue1.FreeTensor(src1Local);
  }
  __aicore__ inline void CopyOut(int32_t i)
  {
    dstQue0.DeQue<half>(dstLocal);
    DataCopy(dstGlobal[i*BLOCK_SIZE], dstLocal, BLOCK_SIZE);
    dstQue0.FreeTensor(dstLocal);
  }

private:
  TPipe pipe;
  TQue<QuePosition::VECIN, 0> srcQue0, srcQue1;
  TQue<QuePosition::VECOUT, 0> dstQue0;
  GlobalTensor<half> src0Global, src1Global, dstGlobal;
  LocalTensor<half> src0Local;
  LocalTensor<half> src1Local;
  LocalTensor<half> dstLocal;
};
} // namespace AscendC

// ...
```

# 13 附录

13.1 基于样例工程完成Kernel直调

13.2 工程化算子开发

13.3 算子入图（GE图）开发

13.4 AI框架算子适配

13.5 show_kernel_debug_data工具

13.6 msobjdump工具

13.7 简易自定义算子工程

13.8 FAQ

## 13.1 基于样例工程完成Kernel直调

> 说明：本章节介绍的基于样例工程完成Kernel直调的方式，后续不再演进。推荐开发者直接使用命令行或者编写Cmake文件进行编译，详细内容请参考5.1 AI Core算子编译。

下文将以Add矢量算子为例对Kernel直调算子开发流程进行详细介绍。

更多算子样例工程请通过如下链接获取：

- 矢量算子样例
- 支持Tiling的矢量算子样例
- 矩阵算子样例
- 矢量+矩阵融合算子样例

### 环境准备

- 使用Kernel Launch算子工程之前，需要参考2环境准备章节安装驱动固件和CANN软件包，完成开发环境和运行环境的准备。
- 使用该算子工程要求cmake版本为3.16及以上版本，如不符合要求，请参考如下的命令示例更新cmake版本，如下示例以更新到3.16.0版本为例。

```bash
wget https://cmake.org/files/v3.16/cmake-3.16.0.tar.gz
tar -zxvf cmake-3.16.0.tar.gz
cd cmake-3.16.0
./bootstrap --prefix=/usr
sudo make
sudo make install
```

### 工程目录

您可以单击矢量算子样例，获取核函数开发和运行验证的完整样例。样例目录结构如下所示：

```
AddKernelInvocationNeo
|-- cmake                    // CMake编译文件
|-- scripts
|   |-- gen_data.py          // 输入数据和真值数据生成脚本文件
|   |-- verify_result.py     // 验证输出数据和真值数据是否一致的验证脚本
|-- CMakeLists.txt            // CMake编译配置文件
|-- add_custom.cpp            // 矢量算子kernel实现
|-- data_utils.h              // 数据读入写出函数
|-- main.cpp                  // 主函数，调用算子的应用程序，含CPU域及NPU域调用
|-- run.sh                    // 编译运行算子的脚本
```

基于该算子工程，开发者进行算子开发的步骤如下：

- 完成算子kernel侧实现。
- 编写算子调用应用程序main.cpp。
- 编写CMake编译配置文件CMakeLists.txt。
- 根据实际需要修改输入数据和真值数据生成脚本文件gen_data.py；验证输出数据和真值数据是否一致的验证脚本verify_result.py。
- 根据实际需要修改编译运行算子的脚本run.sh并执行该脚本，完成算子的编译运行和结果验证。

### 算子Kernel侧实现

请参考10.1.2矢量编程和工程目录中的矩阵算子、融合算子的Kernel实现完成Ascend C算子实现文件的编写。

> 说明：一个算子Kernel实现文件中只支持定义一个核函数。

### 算子调用应用程序

下面代码以固定shape的add_custom算子为例，介绍算子核函数调用的应用程序main.cpp如何编写。您在实现自己的应用程序时，需要关注由于算子核函数不同带来的修改，包括算子核函数名，入参出参的不同等，合理安排相应的内存分配、内存拷贝和文件读写等，相关API的调用方式直接复用即可。

步骤1 按需包含头文件，通过ASCENDC_CPU_DEBUG宏区分CPU/NPU侧需要包含的头文件。需要注意的是，NPU侧需要包含对应的核函数调用接口声明所在的头文件aclrtlaunch_{kernel_name}.h（该头文件为工程框架自动生成），kernel_name为算子核函数的名称。

```cpp
#include "data_utils.h"
#ifndef ASCENDC_CPU_DEBUG
#include "acl/acl.h"
#include "aclrtlaunch_add_custom.h"
#else
#include "tikicpulib.h"
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z);
#endif
```

步骤2 应用程序框架编写。该应用程序通过ASCENDC_CPU_DEBUG宏区分代码逻辑运行于CPU侧还是NPU侧。

```cpp
int32_t main(int32_t argc, char* argv[])
{
  uint32_t blockDim = 8;
  size_t inputByteSize = 8 * 2048 * sizeof(uint16_t);
  size_t outputByteSize = 8 * 2048 * sizeof(uint16_t);

#ifdef ASCENDC_CPU_DEBUG
  // 用于CPU调试的调用程序
#else
  // NPU侧运行算子的调用程序
#endif
  return 0;
}
```

步骤3 CPU侧运行验证。完成算子核函数CPU侧运行验证的步骤如下：

[图: CPU侧验证步骤流程图：使用GmAlloc分配共享内存并进行数据初始化 -> 调用ICPU_RUN_KF调测宏完成核函数CPU侧的调用 -> 输出数据写出 -> 调用GmFree释放申请的资源]

```cpp
// 使用GmAlloc分配共享内存，并进行数据初始化
uint8_t* x = (uint8_t*)AscendC::GmAlloc(inputByteSize);
uint8_t* y = (uint8_t*)AscendC::GmAlloc(inputByteSize);
uint8_t* z = (uint8_t*)AscendC::GmAlloc(outputByteSize);

ReadFile("./input/input_x.bin", inputByteSize, x, inputByteSize);
ReadFile("./input/input_y.bin", inputByteSize, y, inputByteSize);
// 矢量算子需要设置内核模式为AIV模式
AscendC::SetKernelMode(KernelMode::AIV_MODE);
// 调用ICPU_RUN_KF调测宏，完成核函数CPU侧的调用
ICPU_RUN_KF(add_custom, blockDim, x, y, z);
// 输出数据写出
WriteFile("./output/output_z.bin", z, outputByteSize);
// 调用GmFree释放申请的资源
AscendC::GmFree((void *)x);
AscendC::GmFree((void *)y);
AscendC::GmFree((void *)z);
```

步骤4 NPU侧运行验证。完成算子核函数NPU侧运行验证的步骤如下：

[图: NPU侧验证步骤流程图：AscendCL初始化 -> 运行管理资源申请 -> 分配Host内存并进行数据初始化 -> 分配Device内存并将数据从Host上拷贝到Device上 -> 通过Kernel Launch接口或内核调用符<<<>>>调用核函数完成指定的运算并同步等待 -> 将Device上的运算结果拷贝回Host -> 释放申请的资源 -> AscendCL去初始化]

```cpp
// 初始化
CHECK_ACL(aclInit(nullptr));
// 运行管理资源申请
int32_t deviceId = 0;
CHECK_ACL(aclrtSetDevice(deviceId));
aclrtStream stream = nullptr;
CHECK_ACL(aclrtCreateStream(&stream));
// 分配Host内存
uint8_t *xHost, *yHost, *zHost;
uint8_t *xDevice, *yDevice, *zDevice;

CHECK_ACL(aclrtMallocHost((void**)(&xHost), inputByteSize));
CHECK_ACL(aclrtMallocHost((void**)(&yHost), inputByteSize));
CHECK_ACL(aclrtMallocHost((void**)(&zHost), outputByteSize));
// 分配Device内存
CHECK_ACL(aclrtMalloc((void**)&xDevice, inputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
CHECK_ACL(aclrtMalloc((void**)&yDevice, inputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
CHECK_ACL(aclrtMalloc((void**)&zDevice, outputByteSize, ACL_MEM_MALLOC_HUGE_FIRST));
// Host内存初始化
ReadFile("./input/input_x.bin", inputByteSize, xHost, inputByteSize);
ReadFile("./input/input_y.bin", inputByteSize, yHost, inputByteSize);
// 将数据从Host上拷贝到Device上
CHECK_ACL(aclrtMemcpy(xDevice, inputByteSize, xHost, inputByteSize,
ACL_MEMCPY_HOST_TO_DEVICE));
CHECK_ACL(aclrtMemcpy(yDevice, inputByteSize, yHost, inputByteSize,
ACL_MEMCPY_HOST_TO_DEVICE));
// 用内核调用符<<<>>>调用核函数完成指定的运算,add_custom_do中封装了<<<>>>调用
add_custom_do(blockDim, nullptr, stream, xDevice, yDevice, zDevice);
// 用ACLRT_LAUNCH_KERNEL接口调用核函数完成指定的运算
// ACLRT_LAUNCH_KERNEL(add_custom)(blockDim, stream, xDevice, yDevice, zDevice);
CHECK_ACL(aclrtSynchronizeStream(stream));
// 将Device上的运算结果拷贝回Host
CHECK_ACL(aclrtMemcpy(zHost, outputByteSize, zDevice, outputByteSize,
ACL_MEMCPY_DEVICE_TO_HOST));
WriteFile("./output/output_z.bin", zHost, outputByteSize);
// 释放申请的资源
CHECK_ACL(aclrtFree(xDevice));
CHECK_ACL(aclrtFree(yDevice));
CHECK_ACL(aclrtFree(zDevice));
CHECK_ACL(aclrtFreeHost(xHost));
CHECK_ACL(aclrtFreeHost(yHost));
CHECK_ACL(aclrtFreeHost(zHost));
// 去初始化
CHECK_ACL(aclrtDestroyStream(stream));
CHECK_ACL(aclrtResetDevice(deviceId));
CHECK_ACL(aclFinalize());
```

> 说明
> 针对<<<>>>调用方式在4.3核函数章节已有说明，这里仅对ACLRT_LAUNCH_KERNEL调用接口的使用方法介绍如下：
>
> `ACLRT_LAUNCH_KERNEL(kernel_name)(blockDim, stream, argument list);`
>
> - kernel_name：算子核函数的名称。
> - blockDim：规定了核函数将会在几个核上执行。每个执行该核函数的核会被分配一个逻辑ID，即block_idx，可以在核函数的实现中调用GetBlockIdx来获取block_idx。
> - stream，类型为aclrtStream，stream用于维护一些异步操作的执行顺序，确保按照应用程序中的代码调用顺序在Device上执行。stream创建等管理接口请参考"Stream管理"章节。
> - argument list：参数列表，与核函数的参数列表保持一致。

----结束

### CMake编译配置文件编写

本节会介绍CMake文件中一些关键环境变量和Cmake命令参数的说明，通常情况下不需要开发者修改，但是这些参数可以帮助开发者更好的理解编译原理，方便有能力的开发者对Cmake进行定制化处理。

表13-1 环境变量说明

| 环境变量 | 配置说明 |
|---------|--------|
| SOC_VERSION | AI处理器的型号。针对如下产品型号：在安装昇腾AI处理器的服务器执行npu-smi info命令进行查询，获取Name信息。实际配置值为AscendName，例如Name取值为xxxyy，实际配置值为Ascendxxxyy。Atlas A2训练系列产品/Atlas A2推理系列产品，Atlas 200I/500 A2推理产品，Atlas推理系列产品，Atlas训练系列产品。针对如下产品型号：在安装昇腾AI处理器的服务器执行npu-smi info -t board -i id -c chip_id命令进行查询，获取Chip Name和NPU Name信息，实际配置值为Chip Name_NPU Name。例如Chip Name取值为Ascendxxx，NPU Name取值为1234，实际配置值为Ascendxxx_1234。Atlas A3训练系列产品/Atlas A3推理系列产品 |
| ASCEND_CANN_PACKAGE_PATH | CANN软件包安装后的实际路径。 |
| CMAKE_BUILD_TYPE | 编译模式选项，可配置为："Release"，Release版本，不包含调试信息，编译最终发布的版本。"Debug"，Debug版本，包含调试信息，便于开发者开发和调试。 |
| CMAKE_INSTALL_PREFIX | 用于指定CMAKE执行install时，安装的路径前缀，执行install后编译产物（ascendc_library中指定的target以及对应的头文件）会安装在该路径下。默认路径为当前目录的out目录下。 |
| CMAKE_CXX_COMPILER_LAUNCHER | 用于配置C++语言编译器（如g++）、毕昇编译器的启动器程序为ccache，配置后即可开启cache缓存编译，加速重复编译并提高构建效率。使用该功能前需要安装ccache。配置方法如下，在对应的CMakeLists.txt进行设置：`set(CMAKE_CXX_COMPILER_LAUNCHER <launcher_program>)` 其中<launcher_program>是ccache的安装路径，比如ccache的安装路径为/usr/bin/ccache，示例如下：`set(CMAKE_CXX_COMPILER_LAUNCHER /usr/bin/ccache)` |

表13-2 Cmake命令语法说明

| Cmake命令 | 语法说明 |
|----------|--------|
| add_executable | 使用指定的源文件将可执行文件添加到项目中。和Cmake通用的命令参数使用方法一致。 |
| ascendc_library | 使用指定的核函数源文件向项目（project）添加库。语法格式如下：`ascendc_library(<target_name> [STATIC | SHARED] [<source>...])` 其中<target_name>表示库文件的名字，该库文件会根据命令里列出的源文件来建立。STATIC、SHARED的作用是指定生成的库文件的类型。STATIC库是目标文件的归档文件，在连接其它目标的时候使用。SHARED库会被动态连接（动态连接库），在运行时会被加载。<source>表示核函数源文件。 |
| ascendc_fatbin_library | 使用指定的核函数源文件编译生成一个Kernel二进制制文件，供Kernel加载和执行接口使用。语法格式如下：`ascendc_fatbin_library(<target_name> [<source>...])` <target_name>表示库文件的名字，该文件会根据命令里列出的核函数源文件编译生成<target_name>.o文件，放置于${CMAKE_INSTALL_PREFIX}/fatbin/${target_name}/路径下。<source>表示核函数源文件。说明：Kernel加载与执行接口的调用流程和上文介绍的<<<...>>>等调用流程有所差异，具体流程请参考《应用开发指南（C&C++）》中的"Kernel加载与执行"章节。该编译选项暂不支持printf、DumpTensor、DumpAccChkPoint、assert接口。 |
| ascendc_compile_definitions | 添加编译宏。可以添加Ascend C提供的编译宏和开发者自定义的编译宏。语法格式如下：`ascendc_compile_definitions(<target_name> [PRIVATE] [<xxx>...])` Ascend C提供的编译宏介绍如下：ASCENDC_DUMP用于控制Dump开关，默认开关打开，开发者调用printf/DumpTensor/assert后会有信息打印（需要注意直调工程的kernel文件内存在host函数，如果在host函数数内调用了printf接口，也会触发kernel内的printf相关初始化动作，进而影响kernel的执行性能）；设置为0后，表示开关关闭。ASCENDC_DEBUG用于控制Ascend C API的调测开关，默认开关关闭；增加该编译宏后，表示开关打开，此时接口内部的assert校验生效，校验不通过会有assert日志打屏。开启该功能会对算子实际运行的性能带来一定影响，通常在调测阶段使用。HAVE_WORKSPACE用于表示kernel入口是否包含workspace入参。默认情况下为不包含；增加该编译宏后，表示包含，此时框架会获取kernel入参的倒数第一个参数（未配置HAVE_TILING），或倒数第二个参数（配置HAVE_TILING），自动在kernel侧设置系统workspace，开发者在kernel侧入参处获取的workspace为偏移了系统workspace后的用户workspace。建议开启此参数，入参排布、系统workspace的设置逻辑与13.2工程化算子开发保持一致，可减少算子实现在不同开发方式间切换带来的修改成本。HAVE_TILING用于表示kernel入口是否含有tiling入参。在配置了HAVE_WORKSPACE之后，此编译宏才会生效。默认情况下为不包含，开关关闭；增加该编译宏后，表示包含，此时框架会将kernel入参的最后一个参数当做tiling，将倒数第二个参数当做workspace。框架不会对此tiling入参做任何处理，仅通过该入参来判断workspace参数的位置。使用此编译宏可以和13.2工程化算子开发保持入参一致，减少算子实现在不同开发方式间切换带来的修改成本。 |
| ascendc_compile_options | 添加编译选项。可以添加相应的编译选项用于host侧与device侧的编译过程。语法格式如下：`ascendc_compile_options(<target_name> PRIVATE [<xxx>...])` 默认情况下，指定的编译选项都将传递给device侧编译器进行编译。若想传递编译选项给host侧编译器，则需要使用"-forward-options-to-host-compiler"编译选项，该选项后的编译选项将传递给host侧编译器。备注：host侧编译选项只支持g++与clang编译器共同支持的编译选项。 |
| ascendc_include_directories | 添加开发者自定义的头文件搜索路径。语法格式如下：`ascendc_include_directories(<target_name> [PRIVATE] [<xxx>...])` |

简化的编译流程图如下图所示：将算子核函数源文件编译生成kernel侧的库文件（*.so或*.a库文件）；工程框架自动生成核函数调用接口声明头文件；编译main.cpp（算子调用应用程序）时依赖上述头文件，将编译应用程序生成的目标文件和kernel侧的库文件进行链接，生成最终的可执行文件。

[图: 编译简化流程图，展示add_custom.cpp等源文件经Compile生成libkernels1.a/libkernels2.so，自动生成的aclrtlaunch_add_custom.h等头文件和main.cpp经Compile和Link生成可执行文件main]

编译安装结束后在CMAKE_INSTALL_PREFIX目录下生成的编译产物示例如下；最终的可执行文件会生成在cmake命令的执行目录下。

```
out
|-- lib
|   |-- libkernels1.a
|   |-- libkernels2.so
|-- include
    |-- kernels1
    |   |-- aclrtlaunch_matmul_custom.h
    |   |-- aclrtlaunch_add_custom.h
    |-- kernels2
        |-- aclrtlaunch_xxx.h
        |-- ...
```

对于lib目录下生成的库文件可通过msobjdump工具进一步解析得到kernel信息，具体操作参见13.6 msobjdump工具。

### 输入数据和真值数据生成以及验证脚本文件

以固定shape的add_custom算子为例，输入数据和真值数据生成的脚本样例如下：根据算子的输入输出编写脚本，生成输入数据和真值数据。

```python
#!/usr/bin/python3
# -*- coding:utf-8 -*-
import numpy as np

def gen_golden_data_simple():
  input_x = np.random.uniform(1, 100, [8, 2048]).astype(np.float16)
  input_y = np.random.uniform(1, 100, [8, 2048]).astype(np.float16)
  golden = (input_x + input_y).astype(np.float16)

  input_x.tofile("./input/input_x.bin")
  input_y.tofile("./input/input_y.bin")
  golden.tofile("./output/golden.bin")

if __name__ == "__main__":
  gen_golden_data_simple()
```

验证输出数据和真值数据是否一致的验证脚本样例如下：当前使用numpy接口计算了输出数据和真值数据的绝对误差和相对误差，误差在容忍偏差范围内，视为精度符合要求，输出"test pass"字样。

```python
import os
import sys
import numpy as np

loss = 1e-3 # 容忍偏差，一般fp16要求绝对误差和相对误差均不超过千分之一
minimum = 10e-10

def verify_result(real_result, golden):
  real_result = np.fromfile(real_result, dtype=np.float16) # 从bin文件读取实际运算结果
  golden = np.fromfile(golden, dtype=np.float16) # 从bin文件读取预期运算结果
  result = np.abs(real_result - golden) # 计算运算结果和预期结果偏差
  deno = np.maximum(np.abs(real_result), np.abs(golden)) # 获取最大值并组成新数组
  result_atol = np.less_equal(result, loss) # 计算绝对误差
  result_rtol = np.less_equal(result / np.add(deno, minimum), loss) # 计算相对误差
  if not result_rtol.all() and not result_atol.all():
    if np.sum(result_rtol == False) > real_result.size * loss and np.sum(result_atol == False) >
    real_result.size * loss:
      print("[ERROR] result error")
      return False
  print("test pass")
  return True

if __name__ == '__main__':
  verify_result(sys.argv[1],sys.argv[2])
```

### 修改并执行一键式编译运行脚本

您可以基于样例工程中提供的一键式编译运行脚本进行快速编译，并在CPU侧和NPU侧执行Ascend C算子。一键式编译运行脚本主要完成以下功能：

[图: 一键式编译运行脚本流程图：开始 -> 输入数据和真值数据生成脚本(生成input_x.bin、input_y.bin和golden.bin) -> CMAKE编译算子生成可执行文件(ascendc_kernels_bbit) -> 执行编译生成的可执行文件生成算子实际输出结果(output_z.bin) -> 使用numpy接口计算输出数据和真值数据的绝对误差和相对误差在容忍偏差范围内视为精度符合要求 -> Compare -> 结束]

> 须知：样例中提供的一键式编译运行脚本并不能适用于所有的算子运行验证场景，使用时请根据实际情况进行修改。
> - 根据Ascend C算子的算法原理的不同，自行实现输入和真值数据的生成脚本。

完成上述文件的编写后，可以执行一键式编译运行脚本，编译和运行应用程序。

脚本执行方式和脚本参数介绍如下：

```bash
bash run.sh --run-mode=npu --soc-version=<soc_version> --install-path=<install_path> --build-
type=Debug --install-prefix=<install-prefix>

bash run.sh -r npu -v <soc_version> -i <install_path> -b Debug -p <install-prefix>
```

表13-3 脚本参数介绍

| 参数名 | 参数简写 | 参数介绍 |
|--------|--------|--------|
| --run-mode | -r | 表明算子以cpu模式或npu模式运行。取值为cpu或npu。 |
| --soc-version | -v | 算子运行的AI处理器型号。AI处理器的型号请通过如下方式获取：针对如下产品型号：在安装昇腾AI处理器的服务器执行npu-smi info命令进行查询，获取Name信息。实际配置值为AscendName，例如Name取值为xxxyy，实际配置值为Ascendxxxyy。Atlas A2训练系列产品/Atlas A2推理系列产品，Atlas 200I/500 A2推理产品，Atlas推理系列产品，Atlas训练系列产品。针对如下产品型号：在安装昇腾AI处理器的服务器执行npu-smi info -t board -i id -c chip_id命令进行查询，获取Chip Name和NPU Name信息，实际配置值为Chip Name_NPU Name。例如Chip Name取值为Ascendxxx，NPU Name取值为1234，实际配置值为Ascendxxx_1234。Atlas A3训练系列产品/Atlas A3推理系列产品 |
| --install-path | -i | 配置为CANN软件的实际安装路径，请根据实际安装路径进行修改。默认值为$HOME/Ascend/ascend-toolkit/latest。 |
| --build-type | -b | 编译模式选项，可配置为：Release，Release版本，不包含调试信息，编译最终发布的版本。Debug，Debug版本，包含调试信息，便于开发者开发和调试。默认值为Debug。 |
| --install-prefix | -p | 用于指定CMAKE执行install时，安装的路径前缀，执行install后编译产物（ascendc_library中指定的target以及对应的头文件）会安装在该路径下。默认路径为当前目录的out目录下。 |

脚本执行完毕输出"test pass"字样表示算子精度符合要求。

## 13.2 工程化算子开发

### 13.2.1 概述

工程化算子开发是指基于自动生成的自定义算子工程完成算子实现、编译部署、单算子调用代码自动生成等一系列流程。

该开发流程是标准的开发流程，建议开发者按照该流程进行算子开发。该方式下，算子开发的代码会更规范、统一、易于维护；同时该方式考虑了单算子API调用、算子入图、AI框架调用等功能的集成，使得开发者易于借助CANN框架实现上述功能。

[图: 工程化算子开发流程图：环境准备(CANN软件安装、创建算子工程) -> 算子实现(算子原型定义、Kernel侧算子实现、Host侧tiling实现) -> 编译部署(算子工程编译部署) -> 算子调用(单算子API调用)]

步骤1 环境准备。
1. CANN软件安装请参考2环境准备。
2. 创建算子工程。使用msOpGen工具创建算子开发工程。

步骤2 算子实现。
- 算子原型定义。通过原型定义来描述算子输入输出、属性等信息以及算子在AI处理器上相关实现信息，并关联tiling实现等函数。
- Kernel侧算子实现和host侧tiling实现请参考10.1 SIMD算子实现；工程化算子开发，支持开发者调用Tiling API基于CANN提供的编程框架进行tiling开发，kernel侧也提供对应的接口方便开发者获取tiling参数，具体内容请参考13.2.4 Kernel侧算子实现和13.2.5 Host侧Tiling实现，由此而带来的额外约束也在上述章节说明。

步骤3 编译部署。通过工程编译脚本完成算子的编译部署，分为算子包编译和算子动态库编译两种方式。

步骤4 单算子API调用：调用单算子API接口，基于C语言的API执行算子。

----结束

### 13.2.2 创建算子工程

CANN开发套件包中提供了自定义算子工程生成工具msOpGen，可基于算子原型定义输出算子工程：包括算子host侧代码实现文件、算子kernel侧实现文件以及工程编译配置文件等。

> 说明：使用msOpGen工具创建算子工程之前，需要参考2环境准备章节安装驱动固件和CANN软件包，完成开发环境和运行环境的准备。

使用msOpGen工具创建算子开发工程的步骤如下：

步骤1 编写算子的原型定义json文件，用于生成算子开发工程。json文件的配置参数详细说明请参考表1。

例如，AddCustom算子的json文件命名为add_custom.json，文件内容如下：

```json
[
  {
    "op": "AddCustom",
    "input_desc": [
      {
        "name": "x",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["float16", "float", "int32"]
      },
      {
        "name": "y",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["float16", "float", "int32"]
      }
    ],
    "output_desc": [
      {
        "name": "z",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["float16", "float", "int32"]
      }
    ]
  }
]
```

例如，ReduceMaxCustom算子（包含属性）的json文件命名为reduce_max_custom.json，文件内容如下：

```json
[
  {
    "op": "ReduceMaxCustom",
    "input_desc": [
      {
        "name": "x",
        "param_type": "required",
        "format": ["ND"],
        "type": ["float16"]
      }
    ],
    "output_desc": [
      {
        "name": "y",
        "param_type": "required",
        "format": ["ND"],
        "type": ["float16"]
      },
      {
        "name": "idx",
        "param_type": "required",
        "format": ["ND"],
        "type": ["int32"]
      }
    ],
    "attr": [
      {
        "name": "reduceDim",
        "param_type": "required",
        "type": "int"
      },
      {
        "name": "isKeepDim",
        "param_type": "optional",
        "type": "int",
        "default_value": 1
      }
    ]
  }
]
```

> 说明：原型定义json文件中的算子类型字段op需要采用大驼峰的命名方式，即采用大写字符区分不同的语义。

步骤2 使用msOpGen工具生成算子的开发工程。以生成AddCustom的算子工程为例，下文仅针对关键参数进行解释，详细参数说明请参见算子工程创建（msOpGen）。

```bash
${INSTALL_DIR}/python/site-packages/bin/msopgen gen -i $HOME/sample/add_custom.json -c ai_core-
<soc_version> -lan cpp -out $HOME/sample/AddCustom
```

- ${INSTALL_DIR}为CANN软件安装后文件存储路径，请根据实际环境进行替换。
- -i：指定算子原型定义文件add_custom.json所在路径，请根据实际情况修改。
- -c：ai_core-<soc_version>代表算子在AI Core上执行，<soc_version>为昇腾AI处理器的型号。
- -lan：参数cpp代表算子基于Ascend C编程框架，使用C/C++编程语言开发。
- -out：生成文件所在路径，可配置为绝对路径或者相对路径，并且工具执行用户对路径具有可读写权限。若不配置，则默认生成在执行命令的当前路径。

步骤3 命令执行完后，会在-out指定目录或者默认路径下生成算子工程目录，工程中包含算子实现的模板文件，编译脚本等，以AddCustom算子为例，目录结构如下所示：

```
AddCustom
|-- build.sh              // 编译入口脚本
|-- cmake                 // 算子工程编译所需脚本及公共编译文件存放目录
|-- CMakeLists.txt        // 算子工程的CMakeLists.txt
|-- CMakePresets.json      // 编译配置项
|-- framework              // AI框架适配时，算子插件实现文件目录
|-- op_host                // host侧实现文件
|   |-- add_custom_tiling.h   // 算子tiling定义文件
|   |-- add_custom.cpp        // 算子原型注册、shape推导、信息库、tiling实现等内容文件
|   |-- CMakeLists.txt
|-- op_kernel              // kernel侧实现文件
|   |-- add_custom.cpp        // 算子代码实现文件
|   |-- CMakeLists.txt
|-- scripts                // 自定义算子工程打包相关脚本所在目录
```

> 说明
> - 上述目录结构中的粗体文件为后续算子开发过程中需要修改的文件，其他文件无需修改。
> - kernel侧实现依赖的所有文件需全部放置在op_kernel目录下，不遵循该约束会导源码发布场景在线编译失败。因为在后续的算子源码发布时，只会打包发布op_kernel目录下的文件。

----结束

工程目录中的op_kernel和op_host包含了算子的核心实现文件。op_kernel下存放kernel侧算子实现。op_host下存放host侧代码实现，包括算子原型定义、host侧tiling实现。其中kernel侧算子实现和host侧tiling实现在10.1 SIMD算子实现章节已经介绍了其核心的实现方法，在该章节会侧重于介绍接入CANN框架后的编程模式和API的使用。工程目录中的CMakePresets.json，用于开发者完成工程编译相关配置，之后即可进行编译部署。

### 13.2.3 算子原型定义

算子原型主要描述了算子的输入输出、属性等信息以及算子在AI处理器上相关实现信息，并关联tiling实现等函数。算子原型通过自定义的算子类来承载，该算子类继承自OpDef类。完成算子的原型定义等操作后，需要调用OP_ADD接口，传入算子类型（自定义算子类的类名），进行算子原型注册。下面是一个简单的Add算子原型定义和注册的例子。

```cpp
namespace ops {
class AddCustom : public OpDef {
public:
  AddCustom(const char* name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Input("y")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Output("z")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    // 如下的shape/datatype推导函数仅在算子入图场景使用
    this->SetInferShape(ge::InferShape);
    this->SetInferDataType(ge::InferDataType);
    this->AICore()
      .SetTiling(optiling::TilingFunc);
    // 请替换为实际的昇腾AI处理器型号
    this->AICore().AddConfig("ascendxxx");
  }
};
OP_ADD(AddCustom);
} // namespace ops
```

> 说明
> - 基于算子原型定义，自定义算子工程可以实现如下自动化能力：
>   - 自动生成单算子API调用的实现和接口，开发者可以直接使用生成的API实现单算子调用。
>   - 自动生成图模式场景使用的算子原型定义REG_OP，开发者可以使用生成的算子原型定义进行构图、图编译、图执行等操作。
> - 注册算子类型后，框架会根据算子类型获取算子注册信息，同时在编译和运行时按照一定的规则匹配算子实现文件名称和kernel侧核函数名称。为了保证正确匹配，算子类型、算子实现文件名称、核函数名称需要遵循如下定义规则。通常情况下，开发者只需要保证创建算子工程时原型定义json文件中的算子类型op的参数值为大驼峰命名方式即可，工程创建后自动生成的代码即满足该规则定义。在手动编写算子原型定义和算子实现文件时需要按照如下规则定义。
>   - 算子类型需要采用大驼峰的命名方式，即采用大写字符区分不同的语义。
>   - 算子实现文件名称、核函数名称需相同。均为算子类型转换为下划线命名方式后的值。下文描述了通过算子类型转成算子实现文件名称和核函数名称的过程：
>     - 首字符的大写字符转换为小写字符。例如：Abc -> abc。
>     - 大写字符的前一个字符为小写字符或数字，则在大写字符前插一个下划线"_"，并将该字符转换为小写字符。例如：AbcDef -> abc_def。
>     - 大写字符前一个字符为大写字符且后一个字符是小写字符，则在大写字符前插一个下划线"_"，并将该字符转换为小写字符。例如：AbcAAc -> abc_a_ac。
>     - 其他大写字符转换为小写字符，小写字符保持不变。

#### 算子输入/输出/属性定义

算子原型定义描述了算子的输入输出、属性等信息。输入输出支持的datatype、format格式的数量需要一致，并保持一一对应的关系。

如下的代码片段呈现了Add算子输入x的描述信息。

```cpp
this->Input("x")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
```

表13-4 输入输出参数说明

| 原型定义 | 参数 | 具体描述 |
|--------|------|--------|
| Input/Output | ParamType | 参数类型，Option取值为：OPTIONAL（可选）、REQUIRED（必选）、DYNAMIC（动态输入）。类似于上文中的Add样例，其输入输出是必选的。有些算子的输入或者输出个数是动态的，例如AddN，将N个输入Tensor累加到一起，输出一个Tensor；SplitV，将一个Tensor在某个轴上，拆分为N个Tensor输出。有些算子的输入/输出是可选的，例如BatchNorm算子，在训练的时候没有均值和方差输入，在推理的时候有均值和方差的输入。 |
| | DataType | 算子输入输出支持的datatype。datatype的取值请参考DataType。 |
| | Format | 算子输入输出支持的format。format的取值请参考Format。 |

从上文的原型定义中可以看出，列出了输入输出所有datatype和format的组合，保持一一对应。使用如下接口，可以达到简化这种代码逻辑的目的。

- 在指定输入/输出的datatype信息时，如果某个输入/输出的datatype支持和其他所有输入/输出的datatype/format组合使用，其datatype可以通过DataTypeList来表达；在指定输入/输出的format信息时，如果某个输入/输出的format支持和其他所有输入/输出的datatype/format组合使用，其format可以通过FormatList来表达。示例如下，以下两种代码表达含义相同。

```cpp
// 列出所有一一对应的组合
class XxxCustom : public OpDef {
public:
  XxxCustom(const char* name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT16, ge::DT_FLOAT16})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Input("y")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Output("z")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_INT32, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    ...
  }
};
// 通过DataTypeList和FormatList来表达，无需重复列出
class XxxCustom : public OpDef {
public:
  XxxCustom(const char* name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      .DataTypeList({ge::DT_FLOAT16})
      .FormatList({ge::FORMAT_ND});
    this->Input("y")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Output("z")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_INT32, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    ...
  }
};
```

- 通过Follow接口指定当前输入/输出的datatype/format/shape信息与之前定义的某个输入一致。示例如下：输出"y1"Follow输入"x1"场景，此时"y1"的datatype、format以及shape都将会和"x1"保持一致。使用Follow接口指定shape一致时通常比shape推导函数逻辑更加简单，能用Follow表达的逻辑，建议使用Follow接口，则无需再编写注册InferShape函数。

```cpp
this->Input("x1")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT, ge::DT_FLOAT})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND});
this->Input("x2")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND});
this->Output("y1")
  .ParamType(REQUIRED)
  .Follow("x1")
  .OutputShapeDependOnCompute();
```

原型定义中还包括算子属性信息，如下的代码片段呈现了ReduceMax算子的属性reduceDim和isKeepDim的描述信息。

```cpp
this->Attr("reduceDim")
  .AttrType(REQUIRED)
  .Int();
this->Attr("isKeepDim")
  .AttrType(OPTIONAL)
  .Int(1);
```

表13-5 属性参数说明

| 原型定义 | 注册方式 | 具体描述 |
|--------|--------|--------|
| Attr | AttrType | 设置算子属性类型，取值为：OPTIONAL（可选）、REQUIRED（必选）。 |
| | Bool/Float/Int... | 设置算子属性数据类型为Bool/Float/Int...。具体说明请参考OpAttrDef。 |

#### AI处理器上相关实现信息

通过AddConfig注册算子支持的AI处理器型号以及相关的配置信息。AddConfig接口原型如下：soc参数表示AI处理器型号，aicore_config表示其他配置信息。

```cpp
void AddConfig(const char *soc);
void AddConfig(const char *soc, OpAICoreConfig &aicore_config);
```

通过该接口注册AI处理器型号的样例如下，ascendxxx填写规则请参考算子工程目录下编译配置项文件CMakePresets.json中的ASCEND_COMPUTE_UNIT字段取值，在使用msOpGen创建工程时自动生成。

```cpp
this->AICore().AddConfig("ascendxxx");
```

其他AI Core配置信息的配置方式请参考OpAICoreConfig。

#### 注册Tiling实现、Shape推导等函数

通过SetInferShape、SetInferDataType、SetTiling接口来注册对应的Tiling实现和Shape推导等函数，样例如下。注册的Tiling实现等函数由框架侧进行调用，并在调用时传入对应的Context上下文，供开发者使用。Tiling函数的实现方法请参考13.2.5 Host侧Tiling实现，入图相关的Shape推导等函数实现请参考13.3算子入图（GE图）开发。

```cpp
// 如下的shape/datatype推导函数仅在算子入图场景使用
this->SetInferShape(ge::InferShape);
this->SetInferDataType(ge::InferDataType);
this->AICore()
  .SetTiling(optiling::TilingFunc);
```

#### 多硬件平台注册差异化的算子原型

算子类继承基类OpDef，使用Input、Output、Attr等注册算子原型信息，硬件平台支持相同的算子原型的情况下，直接通过AICore().AddConfig添加支持的AI处理器型号即可；不同的硬件形态算子原型定义不同的情况，可以通过新增OpAICoreConfig的方式，针对不同的AI处理器型号注册差异化的算子原型。

差异化的算子原型生效规则如下：

- 对于算子类的输入输出原型信息，OpAICoreConfig未配置的会继承OpDef定义的原型，比如算子类中定义了输出y，OpAICoreConfig中没有定义输出y，OpAICoreConfig会继承y的原型定义；
- 对于算子类和新增OpAICoreConfig中定义的算子原型相同的情况，新增OpAICoreConfig中定义的算子原型信息会覆盖OpDef定义的原型信息，比如算子类中定义了输入x支持DT_FLOAT16数据类型，新增OpAICoreConfig中也定义了输入x，但是支持DT_FLOAT16、DT_BF16数据类型，则以OpAICoreConfig新增定义为准。

如下样例中ascendxxx1、ascendxxx2（AI处理器型号）使用相同的算子原型，算子类通过继承基类OpDef，使用Input、Output、Attr等注册算子原型信息，再通过AICore().AddConfig添加支持的AI处理器型号；对于ascendxxx3支持的算子原型需要定制化处理，新增了DT_BF16的类型，通过新增OpAICoreConfig的方式进行注册，x、y、z的定义会覆盖算子类中对应定义的原型信息。

如下的样例中，只有几个参数原型信息在不同硬件平台不一致，开发者也可以通过OpAICoreConfig定制部分算子原型信息，复用OpDef定义的其他算子原型信息，达到部分原型信息硬件平台定制化的目的。

### 13.2.4 Kernel侧算子实现

在10.1 SIMD算子实现章节已经介绍了kernel侧算子核心的实现方法，本章节侧重于介绍接入CANN框架时编程模式和API的使用。

#### 自动生成kernel侧实现模板

在算子工程目录下的"op_kernel/xxx.cpp"文件中实现算子的核函数。核函数的定义模板已通过msOpGen工具自动生成，样例如下所示。注意这里参数的顺序按照"输入、输出、workspace、tiling"的顺序排布，开发者不要调整其顺序。

```cpp
#include "kernel_operator.h"
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
workspace, GM_ADDR tiling) {
  GET_TILING_DATA(tiling_data, tiling);// 获取Tiling参数，详见下文介绍
  // TODO: user kernel impl
}
```

> 说明：算子原型定义中的输入和输出同名的情况下，自动生成的核函数中，输出参数增加ref后缀予以区分。

#### GET_TILING_DATA获取Tiling参数

提供GET_TILING_DATA，用于获取算子kernel入口函数传入的tiling信息，并填入注册的Tiling结构体中，此函数会以宏展开的方式进行编译。注意，对应的算子host实现中需要定义TilingData结构体，实现并注册计算TilingData的Tiling函数。具体请参考13.2.5 Host侧Tiling实现。

核函数中调用GET_TILING_DATA获取TilingData的样例如下：

```cpp
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
workspace, GM_ADDR tiling)
{
  GET_TILING_DATA(tilingData, tiling);
  KernelAdd op;
  op.Init(x, y, z, tilingData.totalLength, tilingData.tileNum);
  if (TILING_KEY_IS(1)) {
    op.Process();
  }
}
```

#### 核函数内推导输入数据类型和格式

算子工程在核函数内提供了DTYPE_<Arg>、ORIG_DTYPE_<Arg>、FORMAT_<Arg>三种宏用于推导核函数入参的数据类型、原始数据类型和数据格式。其中<Arg>会自动大写。样例如下：

```cpp
template<class T> func() {}
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
workspace, GM_ADDR tiling)
{
  DTYPE_X temp;
  func<DTYPE_Z>();
  if (FORMAT_Y == FORMAT_ND) {
    ...
  }
}
```

#### 输出shape依赖计算的算子kernel实现

某些算子，比如NonZero（统计tensor中非零值的个数），计算完成前无法得知算子输出的shape信息，算子计算完成后才能获取。该类算子在原型定义时，需要使用OutputShapeDependOnCompute接口进行标识，同时在算子核函数中将实际输出shape写入到出参中，便于框架侧基于该信息进行输出内存的管理。

在核函数所有输出的最后增加一个GM_ADDR类型的输出参数，并在核函数计算完成后，将输出shape信息写入到该出参中。shape信息的排布格式如下，大小为n * (8 + 1)，每个元素的数据类型为uint64_t。其中n表示待刷新shape信息的输出个数，每个输出的shape信息都通过第1个元素来保存实际的shape维度（dim），后续的8个元素来保存具体每个维度的shape信息。

[图: shape信息排布格式示意图，展示n个输出的shape信息，每个输出包含dim和8个shape值]

> 说明
> - 输出的顺序和原型定义中输出的顺序保持一致。
> - 对于uint64_t的输出数据类型（对于tensor而言），需要将dim的uint32_t的高位设置为1，表示以uint64_t类型解析该tensor。

### 13.2.5 Host侧Tiling实现

#### 13.2.5.1 基本流程

在10.1 SIMD算子实现章节已经介绍了host侧Tiling核心的实现方法，本章节侧重于介绍接入CANN框架时编程模式和API的使用。

大多数情况下，Local Memory的存储，无法完整的容纳算子的输入与输出，需要每次搬运一部分输入进行计算然后搬出，再搬运下一部分输入进行计算，直到得到完整的最终结果，这个数据切分、分块计算的过程称之为Tiling。根据算子的shape等信息来确定数据切分算法相关参数（比如每次搬运的块大小，以及总共循环多少次）的计算程序，称之为Tiling实现。

Tiling实现完成后，获取到的Tiling切分算法相关参数，会传递给kernel侧，用于指导并行数据的切分。由于Tiling实现中完成的均为标量计算，AI Core并不擅长，所以我们将其独立出来放在host CPU上执行。

[图: Tiling实现的输入输出示意图。输入：算子的输入、输出以及属性信息通过TilingContext* context获取。输出：TilingData、BlockDim、TilingKey、workspace size...设置到TilingContext* context中]

如上图所示，Tiling实现即为根据算子shape等信息来确定切分算法相关参数的过程，这里的算子shape等信息可以理解为是Tiling实现的输入，切分算法相关参数可以理解为是Tiling实现的输出。输入和输出都通过Tiling函数的参数（TilingContext* context上下文结构）来承载。也就是说，开发者可以从上下文结构中获取算子的输入、输出以及属性信息，也就是Tiling实现的输入，经过Tiling计算后，获取到TilingData数据结构（切分算法相关参数）、blockDim变量、用于选择不同的kernel实现分支的TilingKey、算子workspace的大小，也就是Tiling实现的输出，并将这些输出设置到上下文结构中。

TilingData、blockDim、TilingKey、workspace这些概念的具体解释如下：

- TilingData：切分算法相关参数，比如每次搬运的块大小，以及总共循环多少次，通过结构体存储，由开发者自行设计。

  TilingData结构定义支持单结构定义方法，也支持结构体嵌套：

  - 单结构定义方法，以平铺的形式定义：

    ```cpp
    namespace optiling {
    BEGIN_TILING_DATA_DEF(MyAddTilingData) // 声明tiling结构名字
      TILING_DATA_FIELD_DEF(uint32_t, field1); // 结构成员的类型和名字
      TILING_DATA_FIELD_DEF(uint32_t, field2);
      TILING_DATA_FIELD_DEF(uint32_t, field3);
    END_TILING_DATA_DEF;

    REGISTER_TILING_DATA_CLASS(MyAdd, MyAddTilingData) // tiling结构注册给算子
    }
    ```

  - 支持结构体嵌套：

    ```cpp
    namespace optiling {
    BEGIN_TILING_DATA_DEF(MyStruct1) // 声明结构1名字
      TILING_DATA_FIELD_DEF(uint32_t, field1); // 结构成员的类型和名字
      TILING_DATA_FIELD_DEF(uint32_t, field2); // 结构成员的类型和名字
    END_TILING_DATA_DEF;
    REGISTER_TILING_DATA_CLASS(MyStruct1Op, MyStruct1) // 注册结构体到<op_type>Op

    BEGIN_TILING_DATA_DEF(MyStruct2) // 声明结构2名字
      TILING_DATA_FIELD_DEF(uint32_t, field3); // 结构成员的类型和名字
      TILING_DATA_FIELD_DEF(uint32_t, field4); // 结构成员的类型和名字
    END_TILING_DATA_DEF;
    REGISTER_TILING_DATA_CLASS(MyStruct2Op, MyStruct2) // 注册结构体到<op_type>Op

    BEGIN_TILING_DATA_DEF(MyAddTilingData) // 声明tiling结构名字
      TILING_DATA_FIELD_DEF_STRUCT(MyStruct1, st1); // 结构成员的引用结构体
      TILING_DATA_FIELD_DEF_STRUCT(MyStruct2, st2); // 结构成员的引用结构体
    END_TILING_DATA_DEF;
    REGISTER_TILING_DATA_CLASS(MyAdd, MyAddTilingData) // tiling结构注册给算子
    }
    ```

- blockDim：规定了核函数将会在几个核上执行。例如，需要计算8M的数据，blockDim设置为8，但是为了充分利用硬件资源，一般将blockDim设置为硬件平台的核数，根据核数进行数据切分。

  > 说明
  > blockDim是逻辑核的概念，取值范围为[1,65535]。为了充分利用硬件资源，一般设置为物理核的核数或其倍数。
  > - 对于耦合模式和分离模式，blockDim在运行时的意义和设置规则有一些区别。
  > - 耦合模式：由于其Vector、Cube单元是集成在一起的，blockDim用于设置启动多个AI Core核实例执行，不区分Vector、Cube。AI Core的核数可以通过GetCoreNumAiv或者GetCoreNumAic获取。
  > - 分离模式：
  >   - 针对仅包含Vector计算的算子，blockDim用于设置启动多少个Vector（AIV）实例执行。
  >   - 针对仅包含Cube计算的算子，blockDim用于设置启动多少个Cube（AIC）实例执行。
  >   - 针对Vector/Cube融合计算的算子，启动时，按照AIV和AIC组合启动，blockDim用于设置启动多少个组合执行。注意：该场景下，设置的blockDim逻辑核的核数不能超过物理核（2个Vector核和1个Cube核组合为1个物理核）的核数。

- TilingKey（可选）：TilingKey是一个算子内为了区分不同的实现而将kernel代码进行区分的方法，该方法类似于C++的Template模板机制，可减少不必要的icache miss以及scalar耗时，有助于优化单次调用kernel的性能。不同的kernel实现分支可以通过TilingKey来标识，host侧设置TilingKey后，可以选择对应的分支。例如，一个算子在不同的shape下，有不同的算法逻辑，kernel侧可以通过TilingKey来选择不同的算法逻辑，在host侧Tiling算法也有差异，host/kernel侧通过相同的TilingKey进行关联。

  假如有如下kernel代码：

  ```cpp
  if (condition) {
    ProcessA();
  } else {
    ProcessB();
  }
  ```

  如果函数ProcessA、ProcessB两个函数是个非常大的函数，那么上述代码在编译后会变得更大，而每次kernel运行只会选择1个分支，条件的判断和跳转在代码大到一定程度（16-32K，不同芯片存在差异）后会出现icache miss。通过TilingKey可以对这种情况进行优化，给2个kernel的处理函数设置不同的TilingKey 1和2：

  ```cpp
  if (TILING_KEY_IS(1)) {
    ProcessA();
  } else if (TILING_KEY_IS(2)) {
    ProcessB();
  }
  ```

  这样device kernel编译时会自动识别到2个TilingKey并编译2个kernel入口函数，将条件判断进行常量折叠。同时需要和host tiling函数配合，判断走ProcessA的场景设置TilingKey为1，走ProcessB的场景设置TilingKey为2：

  ```cpp
  static ge::graphStatus TilingFunc(gert::TilingContext* context)
  {
    // some code
    if (condition) {
      context->SetTilingKey(1);
    } else {
      context->SetTilingKey(2);
    }
    return ge::GRAPH_SUCCESS;
  }
  ```

  > 说明：编译时，可以通过设置--tiling_key编译选项指定TilingKey，编译时只编译指定TilingKey相关的kernel代码，用于加速编译过程。

- WorkspaceSize：workspace是设备侧Global Memory上的一块内存。在Tiling函数中可以设置workspace的大小。设置后：单算子API执行行场景，可以通过单算子API调用第一段接口获取workspace的大小，然后由开发者申请对应大小的Global Memory；入图场景，框架会根据设置的大小自动申请对应大小的Global Memory。申请workspace后，在算子Kernel实现时，可以使用这块workspace内存。

  workspace内存分为两部分：Ascend C API需要的workspace内存和算子实际使用到的workspace内存（按需）。

  整体的workspace内存就是上述两部分之和，在Tiling函数中设置方法如下：

  ```cpp
  auto workspaceSizes = context->GetWorkspaceSizes(1); // 只使用1块workspace
  workspaceSizes[0] = sysWorkspaceSize + usrWorkspaceSize;
  ```

#### Tiling实现基本流程

Tiling实现开发的流程图如下：

[图: Tiling开发流程图。左侧：算子Tiling结构定义头文件流程(TilingData参数设计 -> TilingData结构定义 -> 注册TilingData结构)。右侧：算子host实现cpp文件流程(获取TilingContext上下文 -> 通过上下文获取输入输出的Shape信息 -> 根据Shape信息设置TilingData、序列化TilingData并保存至TilingContext -> 设置block_dim -> 可选设置TilingKey -> 可选设置workspace size -> 结束)]

下面将从一个简单的Add算子为例介绍Tiling的实现流程。本样例中待处理数据的Shape大小可以平均分配到每个核上，并且可以对齐到一个datablock（32B）的大小。

首先完成算子TilingData结构定义头文件的编写，该文件命名为"算子名称_tiling.h"，位于算子工程的op_host目录下。样例代码如下：

```cpp
#ifndef ADD_CUSTOM_TILING_H
#define ADD_CUSTOM_TILING_H
#include "register/tilingdata_base.h"

namespace optiling {
BEGIN_TILING_DATA_DEF(TilingData)        // 注册一个tiling的类，以tiling的名字作为入参
  TILING_DATA_FIELD_DEF(uint32_t, totalLength); // 添加tiling字段，总计算数据量
  TILING_DATA_FIELD_DEF(uint32_t, tileNum);     // 添加tiling字段，每个核上总计算数据分块个数
END_TILING_DATA_DEF;
// 注册算子tilingdata类到对应的AddCustom算子
REGISTER_TILING_DATA_CLASS(AddCustom, TilingData)
}
#endif // ADD_CUSTOM_TILING_H
```

具体的编写步骤如下：

步骤1 代码框架编写，需要增加#ifndef...的判断条件，防止头文件的重复包含；需要包含register/tilingdata_base.h头文件，tilingdata_base.h中定义了多个用于tilingdata注册的宏。

步骤2 TilingData参数设计，TilingData参数本质上是和并行数据切分相关的参数，本示例算子使用了2个tiling参数：totalLength、tileNum。totalLength是指需要计算的数据量大小，tileNum是指每个核上总计算数据分块个数。比如，totalLength这个参数传递到kernel侧后，可以通过除以参与计算的核数，得到每个核上的计算量，这样就完成了多核数据的切分。

步骤3 TilingData结构定义，通过BEGIN_TILING_DATA_DEF接口定义一个TilingData的类，通过TILING_DATA_FIELD_DEF接口增加TilingData的两个字段totalLength、tileNum，通过END_TILING_DATA_DEF接口结束TilingData定义。

步骤4 注册TilingData结构，通过REGISTER_TILING_DATA_CLASS接口，注册TilingData类，和自定义算子相关联。REGISTER_TILING_DATA_CLASS第一个参数为op_type(算子类型)，本样例中传入AddCustom，第二个参数为TilingData的类名。

----结束

然后完成算子host实现cpp文件中Tiling函数实现，该文件命名为"算子名称.cpp"，位于算子工程的op_host目录下。Tiling函数的原型是固定的，接受一个TilingContext作为输入，在此context上可以获取到输入、输出的Shape指针等内容。注册的Tiling函数由框架调用，调用时会传入TilingContext参数。样例代码如下：

```cpp
namespace optiling {
const uint32_t BLOCK_DIM = 8;
const uint32_t TILE_NUM = 8;
static ge::graphStatus TilingFunc(gert::TilingContext *context)
{
  TilingData tiling;
  uint32_t totalLength = context->GetInputShape(0)->GetOriginShape().GetShapeSize();
  context->SetBlockDim(BLOCK_DIM);
  tiling.set_totalLength(totalLength);
  tiling.set_tileNum(TILE_NUM);
  tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->
GetCapacity());
  context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
  size_t *currentWorkspace = context->GetWorkspaceSizes(1);
  currentWorkspace[0] = 0;
  return ge::GRAPH_SUCCESS;
}
} // namespace optiling
```

具体步骤如下：

步骤1 获取TilingContext的上下文，即Tiling函数的入参gert::TilingContext* context。

步骤2 设置TilingData。在步骤3中定义了TilingData类后，可以创建该类的一个实例，并通过调用set_{field_name}方法来设置各个字段值（其中field_name是步骤3中定义的tiling字段名）。设置完tiling字段后，通过调用SaveToBuffer方法完成TilingData实例的序列化和保存。

1. 通过上下文获取输入输出shape信息。本样例中通过TilingContext的GetInputShape接口获取输入的shape大小。

  ```cpp
  // 获取输入shape信息
  uint32_t totalLength = context->GetInputShape(0)->GetOriginShape().GetShapeSize();
  ```

2. 设置TilingData。通过调用set_{field_name}方法来设置TilingData的字段值。

  ```cpp
  // 用TilingData定义一个具体的实例
  TilingData tiling;
  // 设置TilingData
  tiling.set_totalLength(totalLength);
  tiling.set_tileNum(TILE_NUM);
  ```

3. 调用TilingData类的SaveToBuffer接口完成序列化并保存至TilingContext上下文。SaveToBuffer的第一个参数为存储Buffer的首地址，第二个参数为Buffer的长度。通过调用GetRawTilingData获取无类型的TilingData的地址，再通过GetData获取数据指针，作为Buffer的首地址；通过调用GetRawTilingData获取无类型的TilingData的地址，再通过GetCapacity获取TilingData的长度，作为Buffer的长度。完成SaveToBuffer操作后需要通过SetDataSize设置TilingData的长度，该长度通过TilingData类的GetDataSize接口获取。

  ```cpp
  // 序列化并保存
  tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->
  GetCapacity());
  context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
  ```

步骤3 通过SetBlockDim接口设置blockDim。

```cpp
context->SetBlockDim(BLOCK_DIM);
```

步骤4 （可选）通过SetTilingKey设置TilingKey。

```cpp
context->SetTilingKey(1);
```

步骤5 （可选）通过GetWorkspaceSizes获取workspace size指针，并设置size大小。此处仅作为举例，设置workspace的大小为0。

```cpp
size_t *currentWorkspace = context->GetWorkspaceSizes(1);
currentWorkspace[0] = 0;
```

----结束

#### 13.2.5.2 通过TilingData传递属性信息

如果算子包含属性信息，该属性信息可以通过TilingData传递到kernel侧，参与kernel侧算子核函数的计算。以ReduceMaxCustom算子为例，该算子用于对输入数据按维度dim返回最大值，并且返回索引。ReduceMaxCustom算子有两个属性，reduceDim和isKeepDim，reduceDim表示按照哪一个维度进行reduce操作；isKeepDim表示是否需要保持输出的维度与输入一样。本样例仅支持对最后一维做reduce操作，输入数据类型为half。

1. ReduceMaxCustom算子的TilingData的定义如下：这里我们重点关注reduceAxisLen。参数reduceAxisLen表示获取reduceDim轴的长度，这里也就是最后一维的长度。该参数后续会通过TilingData传递到kernel侧参与计算。

  ```cpp
  #ifndef REDUCE_MAX_CUSTOM_TILING_H
  #define REDUCE_MAX_CUSTOM_TILING_H
  #include "register/tilingdata_base.h"
  namespace optiling {
  BEGIN_TILING_DATA_DEF(ReduceMaxTilingData)
    TILING_DATA_FIELD_DEF(uint32_t, reduceAxisLen); // 添加tiling字段，reduceDim轴的长度
    //其他TilingData参数的定义
    ...
  END_TILING_DATA_DEF;
  // 注册算子tilingdata类到对应的ReduceMaxCustom算子
  REGISTER_TILING_DATA_CLASS(ReduceMaxCustom, ReduceMaxTilingData)
  }
  #endif // REDUCE_MAX_CUSTOM_TILING_H
  ```

2. ReduceMaxCustom算子的Tiling实现如下。这里我们重点关注属性信息通过TilingData传递的过程：首先通过TilingContext上下文从attr获取reduceDim属性值；然后根据reduceDim属性值获取reduceDim轴的长度并设置到TilingData中。

  ```cpp
  namespace optiling {
  static ge::graphStatus TilingFunc(gert::TilingContext* context)
  {
    ReduceMaxTilingData tiling;
    // 从attr获取reduceDim属性值，因为reduceDim是第一个属性，所以GetAttrPointer传入的索引值为0
    const gert::RuntimeAttrs* attrs = context->GetAttrs();
    const uint32_t* reduceDim = attrs->GetAttrPointer<uint32_t>(0);
    // 获取reduceDim轴的长度
    const gert::StorageShape* xShapePtr = context->GetInputShape(0);
    const gert::Shape& xShape = xShapePtr->GetStorageShape();
    const uint32_t reduceAxisLen = xShape.GetDim(*reduceDim);
    ...
    // 将reduceAxisLen设置到tiling结构体中，传递到kernel函数使用
    tiling.set_reduceAxisLen(reduceAxisLen);
    // 设置TilingData中除了reduceAxisLen之外其他成员变量的值
    ...
    // TilingData序列化保存
    tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->
  GetCapacity());
    context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
    ...
    return ge::GRAPH_SUCCESS;
  }} // namespace optiling
  ```

#### 13.2.5.3 使用高阶API时配套的Tiling实现

1. 首先进行tiling结构定义：

  ```cpp
  namespace optiling {
  BEGIN_TILING_DATA_DEF(MyAddTilingData) // 声明tiling结构名字
    TILING_DATA_FIELD_DEF_STRUCT(TCubeTiling, cubeTilingData); // 引用高阶API的tiling结构体
    TILING_DATA_FIELD_DEF(uint32_t, field); // 结构成员的引用结构体
  END_TILING_DATA_DEF;
  REGISTER_TILING_DATA_CLASS(MyAdd, MyAddTilingData) // tiling结构注册给算子
  }
  ```

2. 通过高阶API配套的tiling函数对tiling结构初始化：

  ```cpp
  static ge::graphStatus TilingFunc(gert::TilingContext* context) {
    int32_t M = 1024;
    int32_t N = 640;
    int32_t K = 256;
    int32_t baseM = 128;
    int32_t baseN = 128;
    auto ascendcPlatform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
    MultiCoreMatmulTiling cubeTiling(ascendcPlatform);
    cubeTiling.SetDim(2);
    cubeTiling.SetAType(TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT16);
    cubeTiling.SetBType(TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT16);
    cubeTiling.SetCType(TPosition::LCM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT);
    cubeTiling.SetBiasType(TPosition::GM, CubeFormat::ND, matmul_tiling::DataType::DT_FLOAT);
    cubeTiling.SetShape(M, N, K);
    cubeTiling.SetOrgShape(M, N, K);
    cubeTiling.SetFixSplit(baseM, baseN, -1);
    cubeTiling.SetBias(true);
    cubeTiling.SetBufferSpace(-1, -1, -1);
    MyAddTilingData tiling;
    if (cubeTiling.GetTiling(tiling.cubeTilingData) == -1){
      return ge::GRAPH_FAILED;
    }
    // some code
  }
  ```

#### 13.2.5.4 使用标准C++语法定义Tiling结构体

具体步骤

在定义Tiling结构体时，可以使用标准C++语法定义一个POD类型（Plain Old Data），即与C语言兼容的数据类型。具体步骤如下。完整样例请参考标准C++语法定义Tiling结构体样例。

步骤1 使用C++语法定义Tiling结构体。

该结构体定义所在的头文件应放置在算子工程的op_kernel目录下。由于只有该目录下的文件会被打包进算子包，供在线编译场景中使用，若将文件放置在其他目录中，可能导致在线编译时因找不到相关文件而失败。

用户在使用高阶API的Tiling结构体时，通过AscendC::tiling命名空间引用"kernel_tiling/kernel_tiling.h"中预定义的Tiling结构体，如下代码所示。

```cpp
#ifndef MATMUL_CUSTOM_TILING_H
#define MATMUL_CUSTOM_TILING_H
#include <cstdint>
#include "kernel_tiling/kernel_tiling.h"  // for TCubeTiling

struct MatmulCustomTilingData {
  uint64_t localMemSize;
  AscendC::tiling::TCubeTiling cubeTilingData;
};
#endif // MATMUL_CUSTOM_TILING_H
```

步骤2 Host侧Tiling函数中对Tiling结构体赋值。

- 需要包含Tiling结构体定义头文件。
- 通过GetTilingData获取Tiling结构体指针，并对其成员变量进行赋值。

```cpp
#include "./op_kernel/matmul_custom_tiling.h" // 包含Tiling结构体定义头文件
...

namespace optiling {
static ge::graphStatus TilingFunc(gert::TilingContext *context)
{
  ...
  MultiCoreMatmulTiling cubeTiling(ascendcPlatform);
  ...
  // 获取Tiling结构体指针
  MatmulCustomTilingData *tiling = context->GetTilingData<MatmulCustomTilingData>();
  // 对tiling的成员变量赋值
  if (cubeTiling.GetTiling(tiling->cubeTilingData) == -1) {
    return ge::GRAPH_FAILED;
  }
  uint64_t localMemSize;
  ascendcPlatform.GetCoreMemSize(platform_ascendc::CoreMemType::UB, localMemSize);
  tiling->localMemSize = localMemSize;
  ...
  return ge::GRAPH_SUCCESS;
}
} // namespace optiling
```

步骤3 Kernel侧注册Tiling结构体，解析Tiling数据至TilingData结构并使用。

- 需要包含Tiling结构体定义头文件。
- 通过REGISTER_TILING_DEFAULT或者REGISTER_TILING_FOR_TILINGKEY注册Tiling结构体；通过GET_TILING_DATA解析Tiling数据至TilingData结构并使用。其中REGISTER_TILING_DEFAULT同时也用于标识使用标准C++语法定义TilingData结构体。

```cpp
#include "kernel_operator.h"
#include "matmul_custom_tiling.h" // 包含Tiling结构体定义头文件

extern "C" __global__ __aicore__ void matmul_custom(GM_ADDR a, GM_ADDR b, GM_ADDR bias,
GM_ADDR c, GM_ADDR workspace, GM_ADDR tiling)
{
  REGISTER_TILING_DEFAULT(MatmulCustomTilingData);
  GET_TILING_DATA(tilingData, tiling);
  MatmulKernel<half, half, float, float> matmulKernel;
  AscendC::TPipe pipe;
  REGIST_MATMUL_OBJ(&pipe, GetSysWorkSpacePtr(), matmulKernel.matmulObj,
&tilingData.cubeTilingData); // Initialize the matmul object.
  matmulKernel.Init(a, b, bias, c, workspace, tilingData.localMemSize, tilingData.cubeTilingData);
  ...
}
```

#### 使用标准C++语法定义Tiling结构体的优势

相比较使用BEGIN_TILING_DATA_DEF等宏进行定义的方式，该方式不仅更符合C++开发者的开发习惯，并且提供了强大的灵活性。

- 支持bool类型，支持数组、结构体数组及列表初始化。

```cpp
class A {
public:
  bool xxx;
  uint32_t xxx[2][128] = {0};
};

class B {
public:
  bool xxx = false;
  uint8_t xxx[2][2]{0};
  A[10];
};
```

- 不同算子可以支持定义同名但结构不同的Tiling结构体，通过算子引用对应的头文件即可实现区分。这种方式允许每个算子使用符合自身需求的Tiling结构定义，而不会与其他算子产生冲突。

  相比之下，使用BEGIN_TILING_DATA_DEF等宏方式定义同名但结构不同的Tiling结构体时，由于这些结构体会被注册到全局的Tiling结构体管理变量中，可能导致后续通过结构体名称访问时，无法准确获取当前算子实际使用的Tiling结构体，从而引发未定义行为。

```cpp
// 算子A:
class TilingData {
public:
  uint32_t length;
};
// 算子B:
class TilingData {
public:
  uint32_t length;
  uint16_t coreNum;
};
```

- 支持自定义Tiling赋值，用户可以通过访问Tiling结构体成员变量直接赋值，或自定义Tiling赋值函数（宏定义方式下，用户仅可通过框架生成的set_xx/get_xx方法赋值/访问）

Tiling结构体定义：

```cpp
class TilingData {
public:
  uint32_t xxx1;
  uint32_t xxx2;
  uint8_t xxx3;
  bool xxx4;
};
```

Host侧Tiling函数：

```cpp
#include "../op_kernel/xxx_custom_tiling.h" // 包含Tiling结构体定义头文件
...

namespace optiling {
static void ComputeTiling(TilingData* tiling, ...)
{
  // 计算Tiling逻辑
  ...
  tiling->xxx1 = xxx;
  tiling->xxx2 = xxx;
  tiling->xxx3 = xxx;
  tiling->bool = xxx;
}
static ge::graphStatus TilingFunc(gert::TilingContext *context)
{
  ...
  TilingData *tiling = context->GetTilingData<TilingData>();
  ...
  ComputeTiling(tiling, ...)
  ...
  return ge::GRAPH_SUCCESS;
}
} // namespace optiling
```

#### 使用约束

使用标准C++语法定义Tiling结构体时存在如下约束限制：

- Tiling结构体内不支持定义成员函数，因为成员函数存在Device侧和Host侧的差异（Device侧的函数需要__aicore__修饰符），而Tiling结构体Device侧和Host侧共用，所以会在编译或执行时出现问题：

```cpp
class TilingData {
public:
  uint32_t xxx;
  __aicore__ funcA() { ... } // 错误，host侧编译时不支持__aicore__修饰符，会出现编译错误
  void func() { ... }        // 错误，device侧缺少__aicore__修饰符，无法执行
};
```

- Tiling结构体成员变量不支持指针、引用类型，此类数据类型会导致Host侧到Device侧数据解析异常：

```cpp
class TilingData {
public:
  uint32_t* totalLength; // 指针场景不支持，Host无法传递指针到Device
  uint32_t& tileNum;     // 引用场景不支持，Host无法传递指针到Device
};
```

- Tiling结构体仅支持POD类型，没有虚函数、虚继承等面向对象特性，也不支持模板类：

```cpp
class A {
public:
  uint32_t totalLength;
  uint32_t tileNum;
};
class B: public A {
  uint32_t xxx;
  uint32_t xxx;
};
static ge::graphStatus TilingFunc(gert::TilingContext* context)
{
  // 错误用法
  B *tiling = context->GetTilingData<A>(); // 不支持，会触发未知问题
  // 正确用法
  B *tiling = context->GetTilingData<B>();
  ......
  return ge::GRAPH_SUCCESS;
}
```

- GetTilingData获取的Tiling不包含初值，需显式赋值或在Tiling结构体定义并调用Tiling赋值函数；

```cpp
static ge::graphStatus TilingFunc(gert::TilingContext* context)
{
  TilingData *tiling = context->GetTilingData<TilingData>(); //获取Tiling结构体，此时totalLength、
  // tileNum为0，并不会带入初始值

  // 需显式赋值
  tiling->totalLength = totalLength; // 赋值Tiling结构体成员变量
  tiling->tileNum = TILE_NUM;        // 赋值Tiling结构体成员变量
  ......
  return ge::GRAPH_SUCCESS;
}
```

#### 如何将宏定义Tiling结构体修改为标准C++语法

本节介绍如何将使用BEGIN_TILING_DATA_DEF等宏进行定义的方式改造成使用标准C++语法的方式。

步骤1 首先将之前位于op_host目录下的Tiling结构体定义头文件移至op_kernel目录下，内容前后对比如下，注意此时包含的头文件变化，不需要再包含宏定义相关的头文件。

表 13-6 两种方式对比

| 宏定义方式 | 标准C++语法定义方式 |
|---------|-------------|
| `#include "register/tilingdata_base.h"` `#include "tiling/tiling_api.h"` // TCubeTiling结构体通过宏定义 | `#include <cstdint>` `#include "kernel_tiling/kernel_tiling.h"` // TCubeTiling结构体通过C++语法定义 |
| `namespace optiling {` `BEGIN_TILING_DATA_DEF(MatmulCustomTilingData)` `TILING_DATA_FIELD_DEF(uint64_t, localMemSize);` `TILING_DATA_FIELD_DEF_STRUCT(TCubeTiling, cubeTilingData);` `END_TILING_DATA_DEF;` `REGISTER_TILING_DATA_CLASS(MatmulCustom, MatmulCustomTilingData)` `} // namespace optiling` | `struct MatmulCustomTilingData {` `uint64_t localMemSize;` `AscendC::tiling::TCubeTiling cubeTilingData;` `};` |

步骤2 然后修改Host侧的Tiling函数实现，此时对Tiling结构体的成员变量赋值无需使用宏定义生成的set方法，而是使用用户熟悉的C++指针赋值方式。

表 13-7 两种方式对比

| 宏定义方式 | 标准C++语法定义方式 |
|---------|-------------|
| `namespace optiling {` `static ge::graphStatus TilingFunc(gert::TilingContext *context)` `{` `...` `MultiCoreMatmulTiling cubeTiling(ascendcPlatform);` `...` `MatmulCustomTilingData tiling;` `if (cubeTiling.GetTiling(tiling.cubeTilingData) == -1) {` `return ge::GRAPH_FAILED;` `}` `uint64_t localMemSize;` `ascendcPlatform.GetCoreMemSize(platform_ascendc::CoreMemType::UB, localMemSize);` `tiling.set_localMemSize(localMemSize);` // 需要使用宏定义方式生成的set方法 `...` `tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->GetCapacity());` `...` `return ge::GRAPH_SUCCESS;` `}` `} // namespace optiling` | `#include "../op_kernel/matmul_custom_tiling.h"` // 包含Tiling结构体定义头文件 `...` `namespace optiling {` `static ge::graphStatus TilingFunc(gert::TilingContext *context)` `{` `...` `MultiCoreMatmulTiling cubeTiling(ascendcPlatform);` `...` `MatmulCustomTilingData *tiling = context->GetTilingData<MatmulCustomTilingData>();` `if (cubeTiling.GetTiling(tiling->cubeTilingData) == -1) {` `return ge::GRAPH_FAILED;` `}` `uint64_t localMemSize;` `ascendcPlatform.GetCoreMemSize(platform_ascendc::CoreMemType::UB, localMemSize);` `tiling->localMemSize = localMemSize;` // 使用用户友好的C++指针方式赋值成员变量 `...` `return ge::GRAPH_SUCCESS;` `}` `} // namespace optiling` |

步骤3 最后，在Kernel函数入口处新增REGISTER_TILING_DEFAULT调用，用于注册Tiling结构体。该注册操作的作用是：告知框架用户已使用标准C++语法定义Tiling结构体，并明确其类型，以便框架在进行Tiling数据解析时能够正确识别和使用该结构体。

```cpp
#include "matmul_custom_tiling.h"
...

extern "C" __global__ __aicore__ void matmul_custom(GM_ADDR a, GM_ADDR b, GM_ADDR bias,
GM_ADDR c, GM_ADDR workspace, GM_ADDR tiling)
{
  REGISTER_TILING_DEFAULT(MatmulCustomTilingData); // 新增REGISTER_TILING_DEFAULT调用注册Tiling结构体
  ...
}
```

#### 13.2.5.5 Tiling模板编程

在TilingKey编程章节介绍的TilingKey编程方式中，TilingKey不易于记忆和理解，因为它们通常是较长又没有明确含义的数字。

在涉及多个TilingKey的场景中，开发者依赖TilingKey来管理kernel的实现，无论是在管理还是使用上都会遇到相当大的复杂性。为了简化这一过程，可以采用模板编程的方法来替代传统的TilingKey编程，从而减少对TilingKey数值标识的依赖，使kernel的管理更加直观和高效。使用步骤如下：

步骤1 在自定义算子工程的op_kernel目录下，新增定义模板参数和模板参数组合的头文件，本示例中头文件命名为tiling_key_add_custom.h。

- 该头文件中需要包含模板头文件ascendc/host_api/tiling/template_argument.h。
- 定义模板参数ASCENDC_TPL_ARGS_DECL和模板参数组合ASCENDC_TPL_ARGS_SEL（即可使用的模板）。具体API参考见模板参数定义。

```cpp
#include "ascendc/host_api/tiling/template_argument.h"

// 模板参数
ASCENDC_TPL_ARGS_DECL(AddCustomTemplate, // 算子OpType
ASCENDC_TPL_DATATYPE_DECL(D_T_X, C_DT_FLOAT, C_DT_FLOAT16, ASCENDC_TPL_INPUT(0)), //
// DataType类型的模板参数定义：输入参数x的数据类型，取值范围为float16/float32，ASCENDC_TPL_INPUT(0)说明对应kernel侧第0个输入
ASCENDC_TPL_DATATYPE_DECL(D_T_Y, C_DT_FLOAT, C_DT_FLOAT16, ASCENDC_TPL_INPUT(1)), //
// DataType类型的模板参数定义：输入参数y的数据类型，取值范围为float16/float32，ASCENDC_TPL_INPUT(1)说明对应kernel侧第1个输入
ASCENDC_TPL_DATATYPE_DECL(D_T_Z, C_DT_FLOAT, C_DT_FLOAT16, ASCENDC_TPL_OUTPUT(0)), //
// DataType类型的模板参数定义：输入参数z的数据类型，取值范围为float16/float32，ASCENDC_TPL_OUTPUT(0)说明对应kernel侧第0个输出
ASCENDC_TPL_UINT_DECL(TILE_NUM, ASCENDC_TPL_8_BW, ASCENDC_TPL_UI_MIX, 2, 0, 2, 3, 5, 10, 12, 13, 9, 8),
// 自定义UINT类型（无符号整形）的模板参数定义：模板参数为切分的块数，编码位宽为ASCENDC_TPL_8_BW即8比特
// ASCENDC_TPL_UI_MIX表示通过混合模式表达取值范围，有2组数据{0-2}、{3-5}和穷举10、12、13、8，最后结果为{0,1,2,3,4,5,10,12,13,9,8}
ASCENDC_TPL_BOOL_DECL(IS_SPLIT, 0, 1), // 自定义bool类型的模板参数定义：模板参数为是否切分标志位，取值范围为0和1，1表示切分，0表示不切分
);

// 模板参数组合
// 用于调用GET_TPL_TILING_KEY获取TilingKey时，接口内部校验TilingKey是否合法
ASCENDC_TPL_SEL(
  ASCENDC_TPL_ARGS_SEL(
    ASCENDC_TPL_KERNEL_TYPE_SEL(ASCENDC_TPL_AIV_ONLY), // Kernel类型选择
    ASCENDC_TPL_DATATYPE_SEL(D_T_X, C_DT_FLOAT16),
    ASCENDC_TPL_DATATYPE_SEL(D_T_Y, C_DT_FLOAT16),
    ASCENDC_TPL_DATATYPE_SEL(D_T_Z, C_DT_FLOAT16),
    ASCENDC_TPL_UINT_SEL(TILE_NUM, ASCENDC_TPL_UI_LIST, 1, 8),
    ASCENDC_TPL_BOOL_SEL(IS_SPLIT, 0, 1)
  ),
  ASCENDC_TPL_ARGS_SEL(
    ASCENDC_TPL_KERNEL_TYPE_SEL(ASCENDC_TPL_AIV_ONLY),
    ASCENDC_TPL_DATATYPE_SEL(D_T_X, C_DT_FLOAT),
    ASCENDC_TPL_DATATYPE_SEL(D_T_Y, C_DT_FLOAT),
    ASCENDC_TPL_DATATYPE_SEL(D_T_Z, C_DT_FLOAT),
    ASCENDC_TPL_UINT_SEL(TILE_NUM, ASCENDC_TPL_UI_LIST, 1, 8),
    ASCENDC_TPL_BOOL_SEL(IS_SPLIT, 0, 1)
  ),
);
```

步骤2 host侧调用ASCENDC_TPL_SEL_PARAM接口自动生成并配置TilingKey。

- host实现文件中包含步骤1中定义模板参数和模板参数组合的头文件。
- 调用ASCENDC_TPL_SEL_PARAM接口自动生成并配置TilingKey，ASCENDC_TPL_SEL_PARAM输入参数为模板参数的具体值，传入时需要与定义模板参数和模板参数组合的头文件中的模板参数顺序保持一致。

```cpp
#include "tiling_key_add_custom.h"
static ge::graphStatus TilingFunc(gert::TilingContext *context)
{
  TilingData tiling;
  uint32_t totalLength = context->GetInputShape(0)->GetOriginShape().GetShapeSize();
  ge::DataType dtype_x = context->GetInputDesc(0)->GetDataType();
  ge::DataType dtype_y = context->GetInputDesc(1)->GetDataType();
  ge::DataType dtype_z = context->GetOutputDesc(1)->GetDataType();
  int32_t D_T_X = static_cast<int>(dtype_x), D_T_Y = static_cast<int>(dtype_y), D_T_Z =
static_cast<int>(dtype_z), TILE_NUM = 1, IS_SPLIT = 0;
  if(totalLength< MIN_LENGTH_FOR_SPLIT){
    IS_SPLIT = 0;
    TILE_NUM = 1;
  }else{
    IS_SPLIT = 1;
    TILE_NUM = DEFAULT_TILE_NUM;
  }
  context->SetBlockDim(BLOCK_DIM);
  tiling.set_totalLength(totalLength);
  tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->GetCapacity());
  context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
  ASCENDC_TPL_SEL_PARAM(context, D_T_X, D_T_Y, D_T_Z, TILE_NUM, IS_SPLIT);
  size_t *currentWorkspace = context->GetWorkspaceSizes(1);
  currentWorkspace[0] = 0;
  return ge::GRAPH_SUCCESS;
}
```

步骤3 kernel侧实现

- kernel实现文件中包含步骤1中定义模板参数和模板参数组合的头文件。
- 核函数添加template模板，以便支持模板参数的传入，参数顺序需要与定义模板参数和模板参数组合的头文件中的模板参数顺序保持一致。
- 通过对模板参数的分支判断，选择不同的kernel侧实现。

```cpp
#include "tiling_key_add_custom.h"
...

template<typename D_T_X, typename D_T_Y, typename D_T_Z, int TILE_NUM, int IS_SPLIT>
__global__ __aicore__ void add_custom_template(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
workspace, GM_ADDR tiling)
{
  GET_TILING_DATA(tiling_data, tiling);
  KernelAdd<D_T_X, D_T_Y, D_T_Z> op;
  op.Init(x, y, z, tiling_data.totalLength, TILE_NUM);
  if constexpr (std::is_same_v<D_T_X, float> && std::is_same_v<D_T_Y, float> && std::is_same_v<D_T_Z,
float>) {
    op.Process1();
  } else if constexpr (std::is_same_v<D_T_X, half> && std::is_same_v<D_T_Y, half> && std::is_same_v<D_T_Z,
half>){
    if (IS_SPLIT == 0) {
      op.Process1();
    } else if(IS_SPLIT == 1) {
      op.Process2();
    }
  }
}
```

完整样例请参考Tiling模板编程样例。

## 13.2.6 算子包编译

### 13.2.6.1 算子工程编译

算子kernel侧和host侧实现开发完成后，需要对算子工程进行编译，生成自定义算子安装包\*.run，详细的编译操作包括：

- 编译Ascend C算子kernel侧代码实现文件\*.cpp，分为源码发布和二进制发布两种方式。
  - 源码发布：不对算子kernel侧实现进行编译，保留算子kernel源码文件\*.cpp。该方式可以支持算子的在线编译、通过ATC模型转换的方式编译算子的场景。
  - 二进制发布：对算子kernel侧实现进行编译，生成描述算子相关信息的json文件\*.json和算子二进制文件\*.o。算子调用时，如果需要直接调用算子二进制，则使用该编译方式，比如通过13.2.9单算子API调用的方式完成单算子的调用，PyTorch框架中单算子调用的场景，动态网络中调用算子的场景。

- 编译Ascend C算子host侧代码实现文件\*.cpp、\*.h。
  - 将原型定义和shape推导实现编译成算子原型定义动态库libcust_opsproto_\*.so，并生成算子原型对外接口op_proto.h。
  - 将Tiling实现编译成Tiling动态库liboptiling.so等。
  - 基于算子原型定义，自动生成单算子API调用代码和头文件aclnn_\*.h，并编译生成单算子API调用的动态库libcust_opapi.so。

[图: 算子工程编译示意图 - op_kernel编译生成源码发布(保留源码文件*.cpp)或二进制发布(*.json和*.o)；op_host编译生成Tiling动态库、原型定义动态库和对外头文件、以及自动生成单算子API调用代码并编译生成动态库；最终打包为custom_opp_xxx.run]

编译步骤

步骤1 完成工程编译相关配置。

- 修改工程目录下的CMakePresets.json cacheVariables的配置项。CMakePresets.json文件内容如下，需要配置的参数请参考表13-8，其他参数会在工程创建时自动生成。

```json
{
  "version": 1,
  "cmakeMinimumRequired": {
    "major": 3,
    "minor": 19,
    "patch": 0
  },
  "configurePresets": [
    {
      "name": "default",
      "displayName": "Default Config",
      "description": "Default build using Unix Makefiles generator",
      "generator": "Unix Makefiles",
      "binaryDir": "${sourceDir}/build_out",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": {
          "type": "STRING",
          "value": "Release"
        },
        "ENABLE_SOURCE_PACKAGE": {
          "type": "BOOL",
          "value": "True"
        },
        "ENABLE_BINARY_PACKAGE": {
          "type": "BOOL",
          "value": "True"
        },
        "ASCEND_COMPUTE_UNIT": {
          "type": "STRING",
          "value": "ascendxxx"
        },
        "ENABLE_TEST": {
          "type": "BOOL",
          "value": "True"
        },
        "vendor_name": {
          "type": "STRING",
          "value": "customize"
        },
        "ASCEND_PYTHON_EXECUTABLE": {
          "type": "STRING",
          "value": "python3"
        },
        "CMAKE_INSTALL_PREFIX": {
          "type": "PATH",
          "value": "${sourceDir}/build_out"
        },
        "ENABLE_CROSS_COMPILE": {
          "type": "BOOL",
          "value": "False"
        },
        "CMAKE_CROSS_PLATFORM_COMPILER": {
          "type": "PATH",
          "value": "/usr/bin/aarch64-linux-gnu-g++"
        },
        "ASCEND_PACK_SHARED_LIBRARY": {
          "type": "BOOL",
          "value": "False"
        }
      }
    }
  ]
}
```

表 13-8 需要开发者配置的参数列表

| 参数名称 | 参数描述 | 默认值 |
|--------|--------|------|
| CMAKE_BUILD_TYPE | 编译模式选项，可配置为："Release"，Release版本，不包含调试信息，编译最终发布的版本。"Debug"，Debug版本，包含调试信息，便于开发者开发和调试。 | "Release" |
| ENABLE_SOURCE_PACKAGE | 是否开启源码编译 | "True" |
| ENABLE_BINARY_PACKAGE | 是否开启二进制编译 | "True" |
| vendor_name | 标识自定义算子所属厂商的名称。建议开发者自行指定所属厂商名称，避免和其他厂商提供的算子包冲突。 | "customize" |
| ASCEND_PACK_SHARED_LIBRARY | 是否开启动态库编译功能。 | "False" |

- 配置编译相关环境变量（可选）

表 13-9 环境变量说明

| 环境变量 | 配置说明 |
|--------|--------|
| CMAKE_CXX_COMPILER_LAUNCHER | 用于配置C++语言编译器（如g++）、毕昇编译器的启动器程序为ccache，配置后即可开启cache缓存编译，加速重复编译并提高构建效率。使用该功能前需要安装ccache。配置方法如下，在对应的CMakeLists.txt进行设置：`set(CMAKE_CXX_COMPILER_LAUNCHER <launcher_program>)` 其中<launcher_program>是ccache的安装路径，比如ccache的安装路径为/usr/bin/ccache，示例如下：`set(CMAKE_CXX_COMPILER_LAUNCHER /usr/bin/ccache)` |

步骤2 在算子工程目录下执行如下命令，进行算子工程编译。

```
./build.sh
```

编译成功后，会在当前目录下创建build_out目录，并在build_out目录下生成自定义算子安装包custom_opp_<target os>_<target architecture>.run。

用户如果需要编译过程日志存盘，可以使用环境变量ASCENDC_BUILD_LOG_DIR来控制存储路径。用户设置该选项之后，如果编译过程中无错误产生，则对应的log文件后缀会添加"_success"，若编译过程有错误产生，则会在屏幕打印对应的报错信息，以及指示用户log文件的具体路径与文件名，同时，对应log文件后缀会添加"_error"。

```bash
# 如希望编译日志存储在/home/build_log/，则可以按照如下设置，默认不打开日志存储
export ASCENDC_BUILD_LOG_DIR=/home/build_log/
```

算子包交叉编译

完成算子代码实现后，如果当前平台架构和运行环境一致则参考上一节的内容进行编译即可，如果需要实现算子包的交叉编译，您可以参考如下流程。

步骤1 交叉编译工具下载，下表以Ubuntu系列操作系统为例，展示了编译工具下载命令的样例。其他操作系统，请替换为实际的下载命令。

表 13-10 Ubuntu系列操作系统交叉编译工具下载命令样例

| 当前平台架构 | 运行环境平台架构 | 编译工具下载命令 |
|----------|----------|----------|
| x86_64 | aarch64 | sudo apt-get install -y g++-aarch64-linux-gnu |
| aarch64 | x86_64 | sudo apt-get install g++-x86-64-linux-gnu |

步骤2 自定义算子工程交叉编译，构建生成自定义算子包。

1. 修改CMakePresets.json中ENABLE_CROSS_COMPILE为True，使能交叉编译。

```json
"ENABLE_CROSS_COMPILE": {
  "type": "BOOL",
  "value": "True"
}
```

2. 修改CMakePresets.json中CMAKE_CROSS_PLATFORM_COMPILER为安装后的交叉编译工具路径。

```json
"CMAKE_CROSS_PLATFORM_COMPILER": {
  "type": "PATH",
  "value": "/usr/bin/aarch64-linux-gnu-g++"
}
```

3. 在算子工程目录下执行如下命令，进行算子工程交叉编译。

```
./build.sh
```

编译成功后，会在当前目录下创建build_out目录，并在build_out目录下生成自定义算子安装包custom_opp_<target os>_<target architecture>.run

支持自定义编译选项

在算子工程中，如果开发者想对算子kernel侧代码增加一些自定义的编译选项，可以参考如下内容进行编译选项的定制。

修改算子工程op_kernel目录下的CMakeLists.txt，使用add_ops_compile_options来增加编译选项，方法如下：

```
add_ops_compile_options(OpType COMPUTE_UNIT soc_version1 soc_version2 ... OPTIONS option1 option2 ...)
```

表 13-11 具体参数介绍

| 参数名称 | 可选/必选 | 参数描述 |
|--------|---------|--------|
| OpType（算子类型） | 必选 | 第一个参数应传入算子类型，如果需要对算子工程中的所有算子生效，需要配置为ALL。 |
| COMPUTE_UNIT | 可选 | 标识编译选项在哪些AI处理器型号上生效，多个型号之间通过空格间隔。不配置时表示对所有AI处理器型号生效。具体配置如下：针对如下产品型号：在安装昇腾AI处理器的服务器执行npu-smi info命令进行查询，获取Name信息。实际配置值为AscendName，例如Name取值为xxxyy，实际配置值为Ascendxxxyy。Atlas A2训练系列产品/Atlas A2推理系列产品、Atlas 200I/500 A2推理产品、Atlas推理系列产品、Atlas训练系列产品。针对如下产品型号，在安装昇腾AI处理器的服务器执行npu-smi info -t board -i id -c chip_id命令进行查询，获取Chip Name和NPU Name信息，实际配置值为Chip Name_NPU Name。例如Chip Name取值为Ascendxxx，NPU Name取值为1234，实际配置值为Ascendxxx_1234。Atlas A3训练系列产品/Atlas A3推理系列产品 |
| OPTIONS | 必选 | 自定义的编译选项。多个编译选项之间通过空格间隔。增加-D编译选项，用于在编译时定义宏：`OPTIONS -Dname=definition`。增加-g -O0等调试用编译选项。支持传入毕昇编译器编译选项：比如--cce-auto-sync=off，设置该选项可以关闭自动同步功能。自定义算子工程已默认开启，通常无需开发者手动设置。更多编译选项可以参考毕昇编译器编译选项。Ascend C框架提供的编译选项介绍如下：-DASCENDC_DUMP用于控制Dump开关，默认开关打开，开发者调用printf/DumpTensor/assert后会有信息打印；设置为0后，表示开关关闭：`OPTIONS -DASCENDC_DUMP=0`。-DASCENDC_DEBUG用于控制Ascend C API的调测开关，默认开关关闭；增加该编译选项后，表示开关打开，此时接口内部的assert校验生效，校验不通过会有assert日志打屏。开启该功能会对算子实际运行的性能带来一定影响，通常在调测阶段使用：`OPTIONS -DASCENDC_DEBUG`。当前-DASCENDC_DEBUG功能支持的产品型号为：Atlas推理系列产品、Atlas A2训练系列产品/Atlas A2推理系列产品。--tiling_key，设置该选项后，只编译指定的TilingKey相关的kernel代码，用于加速编译过程。若不指定TilingKey编译，则默认编译所有的TilingKey。配置多个TilingKey时，TilingKey之间不能有空格。示例如下，其中1、2为tiling_key：`--tiling_key=1,2` |

须知：
- 编译选项是基于"算子类型+AI处理器型号系列"进行配置的，也就是说不同的"算子类型+AI处理器型号系列"可以配置不同的编译选项。
- 对相同算子类型+AI处理器型号系列，做多次编译选项配置，以后配置的为准。
- 对ALL生效的编译选项和对单一算子生效的编译选项如果没有冲突，同时生效，如果有冲突，以单一算子的编译选项为准。

### 13.2.6.2 算子包部署

算子包部署指执行自定义算子包的安装，算子工程的编译结果会自动部署到算子包安装目录下。

算子包部署

步骤1 自定义算子包安装部署。

在自定义算子包所在路径下，执行如下命令，安装自定义算子包。

```
./custom_opp_<target os>_<target architecture>.run --install-path=<path>
```

--install-path为可选参数，用于指定自定义算子包的安装目录。支持指定绝对路径，运行用户需要对指定的安装路径有可读写权限。

下文描述中的<vendor_name>为算子工程编译时CMakePresets.json配置文件中字段"vendor_name"的取值，默认为"customize"。

- 默认安装场景，不配置--install-path参数，安装成功后会将编译生成的自定义算子相关文件部署到${INSTALL_DIR}/opp/vendors/<vendor_name>目录。${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。例如，若安装的Ascend-cann-toolkit软件包，安装后文件存储路径示例为：$HOME/Ascend/ascend-toolkit/latest。

  须知：自定义算子包默认安装路径${INSTALL_DIR}/opp/vendors的目录权限与CANN软件包安装用户和安装配置有关。如果因权限不足导致自定义算子包安装失败，可使用--install-path参数并配置环境变量ASCEND_CUSTOM_OPP_PATH来指定安装目录（参考指定目录安装）或者联系CANN软件包的安装用户修改vendors目录权限来解决。

- 指定目录安装场景，配置--install-path参数，安装成功后会将编译生成的自定义算子相关文件部署到<path>/vendors/<vendor_name>目录，并在<path>/vendors/<vendor_name>/bin目录下新增set_env.bash，写入当前自定义算子包相关的环境变量。

  须知：如果部署算子包时通过配置--install-path参数指定了算子包的安装目录，则在使用自定义算子前，需要执行source <path>/vendors/<vendor_name>/bin/set_env.bash命令，set_env.bash脚本中将自定义算子包的安装路径追加到环境变量ASCEND_CUSTOM_OPP_PATH中，使自定义算子在当前环境中生效。

命令执行成功后，自定义算子包中的相关文件将部署至当前环境中。

步骤2 以默认安装场景为例，可查看部署后的目录结构，如下所示：

```
opp                  //算子库目录
├── vendors          //自定义算子所在目录
│   ├── config.ini
│   └── vendor_name1 // 存储对应厂商部署的自定义算子，此名字为编译自定义算子安装包时配置的vendor_name，若未配置，默认值为customize
│       ├── framework    //自定义算子插件库
│       ├── op_api
│       │   ├── include
│       │   │   └── aclnn_xx.h  //算子调用API声明文件
│       │   └── lib
│       │       └── libcust_opapi.so
│       ├── op_impl
│       │   └── ai_core
│       │       └── tbe
│       │           ├── config
│       │           └── vendor_name1_impl  //自定义算子实现代码文件
│       │               ├── dynamic
│       │               │   ├── xx.cpp
│       │               │   └── xx.py
│       │           ├── kernel    //自定义算子二进制文件
│       │           │   └── ${soc_version}  //昇腾AI处理器类型
│       │           │       └── config
│       │           ├── op_tiling
│       │           │   ├── lib
│       │           │   └── liboptiling.so
│       ├── op_proto   //自定义算子原型库所在目录
│       │   ├── inc
│       │   │   └── op_proto.h
│       │   └── lib
│       │       └── libcust_opsproto_*.so
│       └── vendor_name2  // 存储厂商vendor_name2部署的自定义算子
```

步骤3 配置自定义算子优先级。

多算子包共存的情况下，若不同的算子包目录下存在相同OpType的自定义算子，则以优先级高的算子包目录下的算子为准。下面介绍如何配置算子包优先级：

- 默认安装场景：当"opp/vendors"目录下存在多个厂商的自定义算子时，您可通过配置"opp/vendors"目录下的"config.ini"文件，配置自定义算子包的优先级。config.ini文件的配置示例如下：

```
load_priority=vendor_name1,vendor_name2,vendor_name3
```

  - "load_priority"：优先级配置序列的关键字，不允许修改。
  - "vendor_name1,vendor_name2,vendor_name3"：自定义算子厂商的优先级序列，按照优先级从高到低的顺序进行排列。

- 指定目录安装场景：如果需要多个自定义算子包同时生效，分别执行各算子包安装路径下的set_env.bash脚本即可。每次脚本执行都会将当前算子包的安装路径追加到ASCEND_CUSTOM_OPP_PATH环境变量的最前面。因此可以按执行顺序确定优先级：脚本执行顺序越靠后，算子包优先级越高。

  比如先执行source <path>/vendor_name1/bin/set_env.bash，后执行source <path>/vendor_name2/bin/set_env.bash，vendor2算子包的优先级高于vendor_name1。

  ASCEND_CUSTOM_OPP_PATH示例如下：
  `ASCEND_CUSTOM_OPP_PATH=<path>/vendor_name2:<path>/vendor_name1`

- 指定目录安装场景下安装的算子包优先级高于默认方式安装的算子包。

多平台算子包部署

支持安装多平台的自定义算子包，安装时同样支持默认路径和自定义路径安装。

以默认路径安装为例，在aarch64平台上分别安装aarch64平台和x86_64平台算子包，安装成功后可查看目录结构兼容两种平台类型，如下所示：

```
opp                  //算子库目录
├── vendors          //自定义算子所在目录
│   ├── config.ini
│   └── vendor_name1 // 存储对应厂商部署的自定义算子
│       ├── framework    //自定义算子插件库
│       ├── op_impl
│       │   └── ai_core
│       │       └── tbe
│       │           ├── config
│       │           └── vendor_name1_impl  //自定义算子实现代码文件
│       │           ├── kernel    //自定义算子二进制文件
│       │           │   └── ${soc_version}
│       │           │       └── config
│       │           ├── op_tiling
│       │           │   ├── lib
│       │           │   │   ├── linux
│       │           │   │   │   ├── aarch64
│       │           │   │   │   │   └── libcust_opmaster_rt2.0.so
│       │           │   │   │   └── x86_64
│       │           │   │   │       └── libcust_opmaster_rt2.0.so
│       │           │   └── liboptiling.so  -> lib/linux/aarch64/libcust_opmaster_rt2.0.so
│       ├── op_proto   //自定义算子原型库所在目录
│       │   ├── inc
│       │   │   └── op_proto.h
│       │   └── lib
│       │       ├── linux
│       │       │   ├── aarch64
│       │       │   │   └── libcust_opsproto_rt2.0.so
│       │       │   └── x86_64
│       │       │       └── libcust_opsproto_rt2.0.so
│       └── vendor_name2  // 存储厂商vendor_name2部署的自定义算子
```

## 13.2.7 算子动态库和静态库编译

算子库编译

算子动态库和静态库编译是将算子实现代码及相关文件编译为动态库和静态库的过程。相比自定义算子包编译，动态库和静态库编译能够显著简化集成与部署流程。该过程包括将算子Kernel实现、Host侧Tiling实现、入图适配文件以及自动生成的单算子调用实现代码编译链接成动态库和静态库。

同时会自动生成以下头文件：
- 单算子调用aclnn头文件，用于单算子调用场景；
- 算子原型定义头文件，用于算子入图场景。

算子动态库和静态库编译支持如下型号：
- Atlas A3训练系列产品/Atlas A3推理系列产品
- Atlas A2训练系列产品/Atlas A2推理系列产品
- Atlas推理系列产品

算子动态库和静态库编译的具体步骤如下：

步骤1 完成工程编译相关配置。

除了上文介绍的基础配置，算子动态库和静态库编译需要在工程目录下的CMakePresets.json cacheVariables的配置项中配置ASCEND_PACK_SHARED_LIBRARY为True，默认为False（会生成run包）。

```json
"ASCEND_PACK_SHARED_LIBRARY": {
  "type": "BOOL",
  "value": "True"
}
```

步骤2 在算子工程目录下执行如下命令，进行算子工程编译。

```
./build.sh
```

编译成功后，会在${CMAKE_INSTALL_PREFIX}/op_api目录生成以下文件：

- 算子原型定义头文件，用于算子入图场景，定义算子的原型。
- 单算子调用aclnn头文件，用于单算子调用场景，提供直接调用算子的接口。
- 动态库libcust_opapi.so，用于动态链接。
- 静态库lib${vendor_name}.a，用于静态链接。
- 安装文件${vendor_name}-config.cmake和${vendor_name}-targets.cmake，方便开发者将多个厂商的算子动态库或静态库集成到一个公共的动态库中，其中${vendor_name}是厂商名，也可以理解成同一个算子工程生成的算子package名称。

具体目录结构如下：

```
op_api
├── include
│   ├── aclnn_optype1.h          // 单算子调用aclnn头文件
│   ├── aclnn_optype2.h
│   ├── aclnn_optypexxx.h
│   └── op_proto.h               // 算子原型定义头文件
├── lib
│   ├── cmake
│   │   └── ${vendor_name}
│   │       ├── ${vendor_name}-config.cmake  // 安装文件
│   │       └── ${vendor_name}-targets.cmake // 安装文件
│   ├── libcust_opapi.so          // 算子动态库
│   └── lib${vendor_name}.a      // 算子静态库
```

算子库集成和使用

- 单算子调用场景

  单算子调用场景可以通过如下方式对算子库编译中生成的动态库和静态库进行集成和使用。完整样例可参考静态库集成和使用样例。

  - 动态库集成

```cmake
find_package(${vendor_name} REQUIRED    # ${vendor_name}为编译生成的算子package名称
  PATHS ${CUST_PKG_PATH}                # ${CUST_PKG_PATH}为编译生成的算子package的存放路径
  NO_DEFAULT_PATH
)
target_link_libraries(op_runner PRIVATE
  ...
  ${vendor_name}::shared          # 已自动包含相关的target依赖
  ...
)
```

  - 静态库集成

```cmake
find_package(${vendor_name} REQUIRED    # ${vendor_name}为编译生成的算子package名称
  PATHS ${CUST_PKG_PATH}                # ${CUST_PKG_PATH}为编译生成的算子package的存放路径
  NO_DEFAULT_PATH
)
target_link_libraries(op_runner PRIVATE
  ...
  ${vendor_name}::static          # 已自动包含相关的target依赖
  ...
)
```

  - 静态库和动态库混合使用场景

```cmake
find_package(${vendor_name1} REQUIRED    # ${vendor_name1}为编译生成的算子package名称
  PATHS ${CUST_PKG_PATH_1}              # ${CUST_PKG_PATH_1}为编译生成的算子package的存放路径
  NO_DEFAULT_PATH
)
find_package(${vendor_name2} REQUIRED    # ${vendor_name2}为编译生成的算子package名称
  PATHS ${CUST_PKG_PATH_2}              # ${CUST_PKG_PATH_2}为编译生成的算子package的存放路径
  NO_DEFAULT_PATH
)
target_link_libraries(op_runner PRIVATE
  ...
  ${vendor_name1}::static         # 已自动包含相关的静态库target依赖
  ${vendor_name2}::shared         # 已自动包含相关的动态库target依赖
  ...
)
```

  说明：
  - 上文中的算子package存放路径默认指算子工程${CMAKE_INSTALL_PREFIX}/op_api目录，开发者可以自行将op_api目录下的文件拷贝到自己的目录下，此时${CUST_PKG_PATH}可以设置为该自定义目录。
  - 由于不同算子工程生成的算子动态库名称相同，如果需要将多个动态库进行集成，需要对libcust_opapi.so名称进行修改：编译前，修改op_host目录中的CMakeLists.txt文件，添加如下代码，设置不同的输出文件名，从而可以区分不同动态库。`set_target_properties(cust_opapi PROPERTIES OUTPUT_NAME ${vendor_name})` 编译后，修改生成的op_api目录中的${vendor_name}-targets.cmake文件，将其中的libcust_opapi.so修改为lib${vendor_name}.so。保证动态库名称的一致性。

- 算子入图场景

  配置ASCEND_CUSTOM_OPP_PATH环境变量，将动态库libcust_opapi.so的绝对路径追加到该环境变量。由GE框架在图编译和执行时根据该环境变量搜索算子动态库并使用。环境变量位置越靠前，算子的优先级越高。

```bash
export ASCEND_CUSTOM_OPP_PATH=${CMAKE_INSTALL_PREFIX}/op_api/lib/:${ASCEND_CUSTOM_OPP_PATH}
```

  说明：动态库编译和算子包编译功能同时使用时，前者生成的动态库优先级更高。如下示例中，path1和path3是算子包编译生成的目录，path2和path4是动态库编译产物的存放目录，则编译产物的优先级为2>4>1>3。
  `ASCEND_CUSTOM_OPP_PATH=<path1>/vendor_name1:<path2>/op_api/lib/<path3>/vendor_name3:<path4>/op_api/lib`

  如果开发者通过Ascend Graph进行图开发，除了配置环境变量的方式也可以采用直接在应用程序的编译文件中链接动态库libcust_opapi.so的方式。动态库链接方式的so加载优先级高于环境变量配置方式。

## 13.2.8 算子工程编译拓展

使用msOpGen工具创建算子工程时，相关编译脚本被固化在本地，若需使用算子工程提供的新特性（比如支持MC²算子等），需重新运行msOpGen工具生成工程。为了解决开发者因功能更新而频繁重建工程的问题，将算子工程的cmake脚本打包到CANN软件包中。开发者可通过find_package的形式来查找对应的cmake modules包，从而使用算子工程对外提供的cmake函数接口。

该开发方式下，开发者参考如下工程目录，自行组织算子工程。目录结构与通过msOpGen生成的目录结构类似，但无需再创建cmake与scripts目录，这些目录被打包至CANN软件包中。以AddCustom算子为例，目录结构如下：

```
AddCustom
├── CMakeLists.txt       // 算子工程顶层CMakeLists.txt
├── CMakePresets.json    // 编译配置项，cmake内置功能，若使能则需3.19及以上cmake版本
├── framework            // AI框架适配时，算子插件实现文件目录
├── op_host              // Host侧实现文件
│   ├── add_custom.cpp   // 算子原型注册、Shape推导、Tiling实现等内容文件
│   ├── add_custom_tiling.h  // 算子Tiling定义文件
│   └── CMakeLists.txt   // Host侧CMakeLists文件
├── op_kernel            // Kernel侧实现文件
│   ├── Add
│   │   └── add_custom.cpp  // 算子代码实现文件
│   └── CMakeLists.txt   // Kernel侧CMakeLists文件
```

CMakeLists.txt编写方法

- 算子工程顶层CMakeLists.txt

  a. 使用find_package找到对应的编译库。
  b. 使用npu_op_package设置算子工程的编译产物形态，支持RUN/SHARED/STATIC，分别对应算子run包形式、算子动态库形式与算子静态库形式，同时该接口还可配置package（即编译产物）的内容和package的安装位置。
  c. 添加需要进行编译的子目录。

```cmake
cmake_minimum_required(VERSION 3.16.0)          # 指定所需的最低cmake版本号
project(opp)
# 1、使用find_package找到对应的编译库。
# ${ASCEND_CANN_PACKAGE_PATH}为对应的CANN软件包路径，开发者可直接使用
find_package(ASC REQUIRED HINTS ${ASCEND_CANN_PACKAGE_PATH}/compiler/tikcpp/ascendc_kernel_cmake)
# 2、使用npu_op_package设置算子工程的编译产物形态。
set(package_name ${vendor_name})
npu_op_package(${package_name}          # package name
  # RUN模式下会使用该name命名算子部署后的目录，静态库模式下输出产物为lib${package_name}.a
  TYPE RUN                              # 指定编译产物形态，[RUN|STATIC|SHARED]
  CONFIG
    ENABLE_SOURCE_PACKAGE True          # 是否将源码打包到run包中
    ENABLE_BINARY_PACKAGE True          # 是否编译Kernel二进制，SHARED/STATIC模式下必须指定为True，默认值为True
    INSTALL_PATH ${CMAKE_BINARY_DIR}/   # package的安装位置
)
# 3、添加需要进行编译的子目录。
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/op_host)
  add_subdirectory(op_host)             # 添加子目录op_host进行编译
endif()
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/op_kernel)
  add_subdirectory(op_kernel)           # 添加子目录op_kernel进行编译
endif()
```

- Host侧CMakeLists.txt

  a. 使用npu_op_code_gen生成aclnn单算子调用代码、入图所需的原型定义代码等。
  b. 单算子调用场景，使用npu_op_library编译aclnn单算子调用库。
  c. 算子入图场景，使用npu_op_library编译算子入图所需的算子原型库。
  d. 使用npu_op_library编译Tiling相关库。
  e. 使用npu_op_package_add添加上述Host侧库至对应package中。

```cmake
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR} ops_srcs)
#1、使用npu_op_code_gen生成aclnn单算子调用代码、入图所需的原型定义代码等。
npu_op_code_gen(
  SRC ${ops_srcs}                       # Host侧实现文件集合
  PACKAGE ${package_name}               # 与npu_op_package的package_name对应
  COMPILE_OPTIONS -g                    # 编译选项
  OUT_DIR ${ASCEND_AUTOGEN_PATH}        # 指定生成文件的保存路径
)
#2、单算子调用场景，使用npu_op_library编译aclnn单算子调用库。
file(GLOB autogen_aclnn_src ${ASCEND_AUTOGEN_PATH}/aclnn_*.cpp)     # 将通过npu_op_code_gen生成的aclnn相关源文件添加至源文件列表中
set_source_files_properties(${autogen_aclnn_src} PROPERTIES GENERATED TRUE) # 设置对应的文件属性为编译过程中生成的文件
npu_op_library(cust_opapi ACLNN                                     # 指定library名称与类型，以及参与编译的源文件
  ${autogen_aclnn_src}
)
target_compile_options(cust_opapi PRIVATE      # 为对应library添加编译选项，与通用的cmake语法一致
  -fvisibility=hidden
)

#3、算子入图场景，使用npu_op_library编译算子入图所需的算子原型库。
file(GLOB proto_src ${ASCEND_AUTOGEN_PATH}/op_proto.cc)              # 将通过npu_op_code_gen生成的原型定义相关源文件添加至源文件列表中
set_source_files_properties(${proto_src} PROPERTIES GENERATED TRUE)  # 设置对应的文件属性为编译过程中生成的文件
npu_op_library(cust_op_proto GRAPH                                   # 指定library名称与类型，以及参与编译的源文件
  ${ops_srcs}
  ${proto_src}
)
target_compile_options(cust_op_proto PRIVATE    # 为对应library添加编译选项
  -fvisibility=hidden
)

#4、使用npu_op_library编译Tiling相关库。
file(GLOB fallback_src ${ASCEND_AUTOGEN_PATH}/fallback_*.cpp)        # 将通过npu_op_code_gen生成的Tiling相关源文件添加至源文件列表中
# 若算子配置了EnableFallBack会生成该文件
set_source_files_properties(${fallback_src} PROPERTIES GENERATED TRUE) # 设置对应的文件属性
npu_op_library(cust_optiling TILING                                  # 指定library名称与类型，以及参与编译的源文件
  ${ops_srcs}
  ${fallback_src}
)
target_compile_options(cust_optiling PRIVATE    # 为对应library添加编译选项
  -fvisibility=hidden
)

#5、使用npu_op_package_add添加上述Host侧库到对应package中。
npu_op_package_add(${package_name}             # 与npu_op_package的package_name对应
  LIBRARY                                      # 添加Host侧相关library，library名称与前述步骤的library名称对应
    cust_optiling
    cust_opapi
    cust_op_proto
)
```

- Kernel侧CMakeList.txt

  a. 使用npu_op_kernel_options添加算子编译选项。支持的编译选项可参考编译选项。
  b. 使用npu_op_kernel_sources指定算子特定目录与编译源文件。
    - 若算子的源码文件没有平铺在SRC_BASE目录（通过npu_op_kernel_library设置）下，可以通过KERNEL_DIR指定特定目录。
    - 若算子的Kernel实现cpp文件需要自定义命名，需同时指定OP_TYPE（算子类型）和KERNEL_FILE（Kernel实现cpp文件名），以配置两者之间的对应关系。不配置时，Kernel实现cpp文件名和OpType之间需满足转换规则。
  c. 使用npu_op_kernel_library编译Kernel库。
  d. 使用npu_op_package_add添加上述Kernel侧库到对应package中。

```cmake
#1、使用npu_op_kernel_options添加算子编译选项。
npu_op_kernel_options(ascendc_kernels ALL OPTIONS --save-temp-files -g) # 为算子添加编译选项

#2、使用npu_op_kernel_sources指定算子特定目录与编译源文件。
npu_op_kernel_sources(ascendc_kernels          # 对应的Kernel库名称，与npu_op_kernel_library保持一致
  OP_TYPE AddCustom                            # 若算子的Kernel实现cpp文件需要自定义命名，需配置算子类型
  KERNEL_DIR ./Add                             # 将KERNEL_DIR目录与SRC_BASE目录进行拼接，作为Kernel实现cpp文件所在的目录
  COMPUTE_UNIT Ascendxxxyy                     # 设置KERNEL_FILE在Ascendxxxyy型号生效
  KERNEL_FILE add_custom.cpp                   # 若算子的Kernel实现cpp文件需要自定义命名，需配置Kernel实现cpp文件名
)

#3、使用npu_op_kernel_library编译Kernel库。
npu_op_kernel_library(ascendc_kernels          # Kernel库名称
  SRC_BASE ${CMAKE_SOURCE_DIR}/op_kernel/      # 指定kernel侧源码的根目录
  TILING_LIBRARY cust_optiling                 # 指定依赖的Tiling库
)

#4、添加kernel侧library到package
npu_op_package_add(${package_name}             # 与npu_op_package的package_name对应
  LIBRARY
    ascendc_kernels
)
```

编译环境选项说明

下表列出了供用户配置和使用的编译环境选项：

说明：暂不支持设置Release、Debug版本相关选项。暂不支持交叉编译相关选项。

表 13-12 编译环境选项

| 类型 | 环境选项 | 默认值 | 说明 |
|------|--------|------|------|
| STRING | vendor_name | customize | 标识自定义算子所属厂商的名称。建议开发者自行指定所属厂商名称，避免和其他厂商提供的算子包冲突。 |
| STRING | ASCEND_COMPUTE_UNIT | - | 昇腾AI处理器型号。编译对应型号的package。 |
| PATH | ASCEND_AUTOGEN_PATH | <CMAKE_BINARY_DIR>/autogen | 通过npu_op_code_gen生成的aclnn单算调用、入图所需算子原型库等源文件存放路径。 |
| BOOL | ENABLE_SOURCE_PACKAGE | TRUE | 是否开启源码编译。如果使用npu_op_package配置了源码和二进制编译相关配置，npu_op_package配置的优先级更高。 |
| BOOL | ENABLE_BINARY_PACKAGE | TRUE | 是否开启二进制编译。package的类型配置为SHARED或STATIC时，必须指定为TRUE。如果使用npu_op_package配置了源码和二进制编译相关配置，npu_op_package配置的优先级更高。 |
| PATH | ASCEND_CANN_PACKAGE_PATH | - | CANN软件包路径，默认情况下用户无需配置。开发者可以在编写cmake文件时直接使用。路径示例如下：/usr/local/Ascend/ascend-toolkit/latest。 |

上述编译环境选项可通过两种方式进行配置：

- 支持直接通过set命令进行设置。

```cmake
# CMakeLists.txt直接使用
set(vendor_name "customize")
set(ASCEND_COMPUTE_UNIT "ascendxxxyy")
```

- 通过CMakePresets.json进行设置，若通过该方式设置，则需要安装3.19及以上的cmake。

```json
{
  "version": 1,
  "cmakeMinimumRequired": {
    "major": 3,
    "minor": 19,
    "patch": 0
  },
  "configurePresets": [
    {
      "name": "default",
      "displayName": "Default Config",
      "description": "Default build using Unix Makefiles generator",
      "generator": "Unix Makefiles",
      "binaryDir": "${sourceDir}/build_out",
      "cacheVariables": {
        "ASCEND_COMPUTE_UNIT": {
          "type": "STRING",
          "value": "ascendxxxyy"
        },
        "vendor_name": {
          "type": "STRING",
          "value": "customize"
        }
      }
    }
  ]
}
```

然后使用如下命令进行构建，根据--preset指定的preset进行编译。

```bash
// Shell环境中cmake构建编译
cmake -S . -B build_out --preset=default // 在当前目录(.)中寻找CMakePresets.json
```

编译命令说明

package的类型配置为RUN时，可参考如下编译命令进行编译，使用cmake的pack能力生成run包。

```bash
# 工程根目录下执行
mkdir -p build_out
rm -rf build_out/*
cmake -S . -B build_out --preset=default
cmake --build build_out --target binary -j${nproc}
cmake --build build_out --target package -j${nproc}
```

package的类型配置为SHARED或STATIC时，可参考如下编译命令进行编译：

```bash
# 工程根目录下执行
mkdir -p build_out
rm -rf build_out/*
cmake -S . -B build_out --preset=default
cmake --build build_out --target binary -j${nproc}
cmake --build build_out --target install -j${nproc}
```

cmake函数说明

提供如下cmake函数供开发者使用：

| 类型 | 接口 | 功能 |
|------|------|------|
| package | npu_op_package | 创建一个package。 |
| package | npu_op_package_add | 将目标或文件添加到package中。 |
| library | npu_op_library | 创建Host侧库。 |
| library | npu_op_kernel_library | 创建Kernel侧库。 |
| library | npu_op_kernel_options | 添加Kernel目标编译选项。 |
| library | npu_op_kernel_sources | 描述Kernel目标的源码信息。 |
| library | npu_op_device_tiling_library | 创建Device侧Tiling库。 |
| 其他 | npu_op_code_gen | 执行代码生成过程，生成aclnn单算子调用代码和入图所需的原型定义代码。 |

1. npu_op_package

   创建一个package。

   ```
   npu_op_package(<package_name> TYPE <type> [CONFIG] [ENABLE_SOURCE_PKG <value>]
   [ENABLE_BINARY_PACKAGE <value>] [INSTALL_PATH <path>])
   ```

   参数说明如下：
   - <package_name>：必选，package的名称。
   - TYPE <type>：必选，package的类型，取值为RUN、SHARED、STATIC。分别对应算子run包形式、算子动态库形式与算子静态库形式。
   - [CONFIG]：可选，用于配置package的内容和安装位置。
     - [ENABLE_SOURCE_PKG <value>]：可选，是否将源码打包到package中，默认为True。
     - [ENABLE_BINARY_PACKAGE <value>]：可选，是否将二进制文件打包到package中，默认为True。
     - [INSTALL_PATH <path>]：可选，指定包的安装路径，默认为CMAKE_BINARY_DIR。

2. npu_op_package_add

   将目标或文件添加到package中。

   ```
   # 添加目标
   npu_op_package_add(<package_name> LIBRARY <target_name1> [<target_name2>...] )
   # 添加文件，仅给run包模式使用
   npu_op_package_add(<package_name> FILES <file_name1> [<file_name2>...] [TYPE <target_type>]
   [PACKAGE_PATH <pkg_path>])
   ```

   参数说明如下：
   - <package_name>：必选，package的名称。
   - LIBRARY：必选，指定需要添加到package中的目标名称。
     - <target_name1> [<target_name2>...]：必选，目标名称列表。
   - FILES：必选，指定需要添加到package中的文件名称。
     - <file_name1> [<file_name2>...]：必选，文件名称列表。
   - [TYPE <target_type>]：可选，指定文件类型，将文件安装到对应的目录中，取值为ACLNN、GRAPH。配置为ACLNN，会将文件打包至run包目录下aclnn单算子调用头文件所在目录，配置为GRAPH，会将文件打包至run包目录下入图原型定义头文件目录下。
   - [PACKAGE_PATH <pkg_path>]：可选，指定文件在包中的相对路径位置。TYPE和PACKAGE_PATH参数互斥，即只能选择其中一个进行配置。

3. npu_op_library

   创建Host侧库。

   ```
   npu_op_library(<library_name> TYPE <library_type> <files>)
   ```

   参数说明如下：
   - <library_name>：必选，Host侧库的名称。
   - TYPE <library_type>：必选，Host库的类型，可选值为TILING、ACLNN、GRAPH、TF_PLUGIN。
     - TILING，Tiling相关库。
     - ACLNN，aclnn单算子调用库。
     - GRAPH，算子入图所需的算子原型库。
     - TF_PLUGIN，TensorFlow框架适配相关库。
   - <files>：必选，设置参与编译的源文件。

4. npu_op_kernel_library

   创建Kernel侧库。

   ```
   npu_op_kernel_library(<target_name> SRC_BASE <path> TILING_LIBRARY <tiling_target>)
   ```

   参数说明如下：
   - <target_name>：必选，目标的名称。
   - SRC_BASE <path>：必选，指定Kernel源码的base目录，要求配置绝对路径。比如示例中的op_kernel目录的绝对路径。
   - TILING_LIBRARY <tiling_target>：必选，指定依赖的Tiling目标。

5. npu_op_kernel_options

   添加Kernel目标编译选项。

   ```
   npu_op_kernel_options(<target_name> <op_type> [COMPUTE_UNIT <soc_version>] OPTIONS ...)
   ```

   参数说明如下：
   - <target_name>：必选，目标的名称。
   - <op_type>：必选，定义配置生效的范围，取值为ALL、OP_TYPE。ALL表示对所有算子生效，OP_TYPE表示对特定算子生效。
   - [COMPUTE_UNIT <soc_version>]：可选，用于设置算子在具体AI处理器型号上的编译选项，不填写该选项时默认对所有型号生效。
   - OPTION...：必选，传递给编译器的编译选项。

6. npu_op_kernel_sources

   描述Kernel目标的源码信息，包括设置算子的Kernel实现文件和源码路径等。

   ```
   npu_op_kernel_sources(<target_name> [OP_TYPE <op_type>] [KERNEL_DIR <path>] [COMPUTE_UNIT
   <soc_version>] [KERNEL_FILE <file>])
   ```

   参数说明如下：
   - <target_name>：必选，目标的名称。
   - [OP_TYPE <op_type>]：可选，算子类型，必须与KERNEL_FILE同时存在。
   - [KERNEL_DIR <path>]：可选，指定Kernel源码相对于SRC_BASE的相对路径。若算子的源码文件没有平铺在SRC_BASE目录（通过npu_op_kernel_library设置）下，可以通过KERNEL_DIR指定特定目录。
   - [COMPUTE_UNIT <soc_version>]：可选，设置KERNEL_FILE在<soc_version>型号生效。默认KERNEL_FILE对所有型号生效。
   - [KERNEL_FILE <file>]：可选，指定算子入口的Kernel实现文件名。若算子的Kernel实现cpp文件需要自定义命名，需同时指定OP_TYPE（算子类型）和KERNEL_FILE（Kernel实现cpp文件名），以配置两者之间的对应关系。不配置时，Kernel实现cpp文件名和OpType之间需满足转换规则。

7. npu_op_device_tiling_library

   创建Device侧Tiling库。使用该选项时，package的类型仅支持配置为RUN（run包模式）。

   ```
   npu_op_device_tiling_library(<target_name> <type> <files>)
   ```

   参数说明如下：
   - <target_name>：必选，目标的名称。
   - <type>：必选，指定Tiling产物的类型。支持取值为SHARED、STATIC。
   - <files>：必选，指定Tiling源文件。

8. npu_op_code_gen

   执行代码生成过程，生成aclnn单算子调用代码和入图所需的原型定义代码。

   ```
   npu_op_code_gen(SRC <src_files> OUT_DIR <output_dir> PACKAGE <pkg_name>
   [COMPILE_OPTIONS ...])
   ```

   参数说明如下：
   - SRC <src_files>：必选，参与代码生成的源文件范围。
   - OUT_DIR <output_dir>：必选，生成代码的输出路径。
   - PACKAGE <pkg_name>：必选，指定生成代码的package名称。
   - [COMPILE_OPTIONS ...]：可选，自定义编译过程中的编译选项。

## 13.2.9 单算子API调用

单算子API调用方式，是指直接调用单算子API接口，基于C语言的API执行算子。算子工程创建完成后，基于工程代码框架完成算子原型定义、kernel侧算子实现、host侧tiling实现，通过工程编译脚本完成算子的编译部署，之后再进行单算子API的调用。

基本原理

完成自定义算子编译后，会自动生成单算子API，可以直接在应用程序中调用。

单算子API的形式一般定义为"两段式接口"，形如：

```cpp
aclnnStatus aclnnXxxGetWorkspaceSize(const aclTensor *src, ..., aclTensor *out, uint64_t *workspaceSize,
aclOpExecutor **executor);
aclnnStatus aclnnXxx(void *workspace, uint64_t workspaceSize, aclOpExecutor *executor, aclrtStream
stream);
```

其中aclnnXxxGetWorkspaceSize/aclnnXxxTensorGetWorkspaceSize为第一段接口，主要用于计算本次API调用过程中需要多少workspace内存，获取到本次计算所需的workspaceSize后，按照workspaceSize申请NPU内存，然后调用第二段接口aclnnXxx执行计算。Xxx代表算子原型注册时传入的算子类型。

aclnnXxxGetWorkspaceSize接口的输入输出参数生成规则如下：

- 可选输入的命名增加Optional后缀。如下样例中，x是可选输入。
  `aclnnStatus aclnnXxxGetWorkspaceSize(const aclTensor *xOptional, ..., aclTensor *out, uint64_t *workspaceSize, aclOpExecutor **executor);`

- 输入输出同名、使用同一个Tensor承载的情况下，生成的aclnn接口中只保留input参数同时去掉input的const修饰，并以Ref作为后缀。如下样例中，原型定义input、output都定义为x，xRef既作为输入，又作为输出。
  `aclnnStatus aclnnXxxGetWorkspaceSize(aclTensor *xRef, ..., uint64_t *workspaceSize, aclOpExecutor **executor);`

- 如果仅有一个输出，输出参数命名为out；如果存在多个输出，每个输出后面都以Out作为后缀。
  `// 仅有一个输出`
  `aclnnStatus aclnnXxxGetWorkspaceSize(const aclTensor *src, ..., aclTensor *out, uint64_t *workspaceSize, aclOpExecutor **executor);`
  `// 存在多个输出`
  `aclnnStatus aclnnXxxGetWorkspaceSize(const aclTensor *src, ..., aclTensor *yOut, aclTensor *y1Out, uint64_t *workspaceSize, aclOpExecutor **executor);`

- 如果算子包含属性，则属性参数的位置介于输入输出之间。如下示例中，x是算子输入，negativeSlope是算子属性，out是算子输出。
  `aclnnStatus aclnnXxxGetWorkspaceSize(const aclTensor *x, double negativeSlope, aclTensor *out, uint64_t *workspaceSize, aclOpExecutor **executor);`

当算子原型注册时使用ValueDepend接口标识输入为数据依赖输入时，会额外生成一个API，该API支持值依赖场景输入数据为空的一阶段计算。

`aclnnStatus aclnnXxxTensorGetWorkspaceSize(const aclTensor *src, ..., aclTensor *out, uint64_t *workspaceSize, aclOpExecutor **executor);`

在aclnnXxxTensorGetWorkspaceSize中，aclnnXxxGetWorkspaceSize参数的数据类型（aclIntArray、aclFloatArray和aclBoolArray）将被转换为aclTensor数据类型，其他输入输出参数生成规则与aclnnXxxGetWorkspaceSize一致。

前置步骤

- 参考13.2.2创建算子工程完成自定义算子工程的创建。
- 参考13.2.4 Kernel侧算子实现完成kernel侧实现的相关准备，参考13.2.5 Host侧Tiling实现、13.2.3算子原型定义完成host侧实现相关准备。
- 对于算子包编译场景，参考13.2.6.1算子工程编译、13.2.6.2算子包部署完成算子的编译部署，编译部署时需要开启算子的二进制编译功能：修改算子工程中的编译配置项文件CMakePresets.json，将ENABLE_BINARY_PACKAGE设置为True。编译部署时可将算子的二进制部署到当前环境，便于后续算子的调用。

  ```json
  "ENABLE_BINARY_PACKAGE": {
    "type": "BOOL",
    "value": "True"
  },
  ```

  算子编译部署后，会在算子包安装目录下的op_api目录生成单算子调用的头文件aclnn_xx.h和动态库libcust_opapi.so。

  以默认安装场景为例，单算子调用的头文件.h和动态库libcust_opapi.so所在的目录结构，如下所示：

  ```
  opp    //算子库目录
  ├── vendors    //自定义算子所在目录
  │   ├── config.ini
  │   └── vendor_name1
  │       ├── op_api
  │       │   ├── include
  │       │   │   └── aclnn_xx.h
  │       │   └── lib
  │       │       └── libcust_opapi.so
  ...
  ```

- 对于算子动态库和静态库编译场景，参考13.2.7算子动态库和静态库编译完成算子的编译。编译完成后会在如下路径生成单算子调用的头文件aclnn_xx.h和动态库libcust_opapi.so。其中CMAKE_INSTALL_PREFIX为开发者在cmake文件中配置的编译产物存放路径。
  - 动态库路径：${CMAKE_INSTALL_PREFIX}/op_api/lib/libcust_opapi.so
  - 头文件路径：${CMAKE_INSTALL_PREFIX}/op_api/include

准备验证代码工程

代码工程目录结构如下，您可以单击LINK，获取样例工程的完整样例：

```
├── input                  // 存放脚本生成的输入数据目录
├── output                 // 存放算子运行输出数据和真值数据的目录
├── inc                    // 头文件目录
│   ├── common.h           // 声明公共方法类，用于读取二进制文件
│   ├── operator_desc.h    // 算子描述声明文件，包含算子输入/输出，算子类型以及输入描述与输出描述
│   └── op_runner.h        // 算子运行相关信息声明文件，包含算子输入/输出个数，输入/输出大小等
├── src
│   ├── CMakeLists.txt     // 编译规则文件
│   ├── common.cpp         // 公共函数，读取二进制文件函数的实现文件
│   ├── main.cpp           // 单算子调用应用的入口
│   ├── operator_desc.cpp  // 构造算子的输入与输出描述
│   └── op_runner.cpp      // 单算子调用主体流程实现文件
├── scripts
│   ├── verify_result.py   // 真值对比文件
│   ├── gen_data.py        // 输入数据和真值数据生成脚本文件
│   └── acl.json           // acl配置文件
```

下文将重点介绍和单算子调用流程相关的main.cpp、op_runner.cpp文件、CMakeLists.txt文件如何编写，其他文件请自行参考。

单算子调用流程

单算子API执行流程如下：

[图: 单算子API执行接口调用流程 - 开始 -> 初始化 -> 运行时资源申请 -> (单算子调用: 申请内存存放执行算子的输入/输出数据aclrtMalloc -> 传输数据aclrtMemcpy或aclrtMemcpyAsync -> 计算workspace大小并申请内存aclxxXxxGetWorkspaceSize+aclrtMalloc -> 执行算子aclxxXxx -> 同步等待aclrtSynchronizeStream -> 释放内存aclrtFree) -> 运行时资源释放 -> 去初始化 -> 结束]

本节以AddCustom自定义算子调用为例，介绍如何编写单算子调用的代码逻辑。其他算子的调用逻辑与Add算子大致一样，请根据实际情况自行修改代码。

以下是关键步骤的代码示例，不可以直接拷贝编译运行，仅供参考，调用接口后，需增加异常处理的分支，并记录报错日志、提示日志，此处不一一列举。

说明：因为单算子API执行方式，会自动在编译工程的build_out/autogen目录下生成.cpp和.h，编写单算子的调用代码时，要包含自动生成的单算子API执行接口头文件。示例如下：
`#include "aclnn_add_custom.h"`

```cpp
// 1.初始化
aclRet = aclInit("../scripts/acl.json");

// 2.运行管理资源申请
int deviceId = 0;
aclRet = aclrtSetDevice(deviceId);
// 获取软件栈的运行模式，不同运行模式影响后续的接口调用流程（例如是否进行数据传输等）
aclrtRunMode runMode;
bool g_isDevice = false;
aclError aclRet = aclrtGetRunMode(&runMode);
g_isDevice = (runMode == ACL_DEVICE);

// 3.申请内存存放算子的输入输出
// ......

// 4.传输数据
if (aclrtMemcpy(devInputs_[i], size, hostInputs_[i], size, kind) != ACL_SUCCESS) {
  return false;
}

// 5.计算workspace大小并申请内存
size_t workspaceSize = 0;
aclOpExecutor *handle = nullptr;
auto ret = aclnnAddCustomGetWorkspaceSize(inputTensor_[0], inputTensor_[1], outputTensor_[0],
                    &workspaceSize, &handle);
// ...
void *workspace = nullptr;
if (workspaceSize != 0) {
  if (aclrtMalloc(&workspace, workspaceSize, ACL_MEM_MALLOC_HUGE_FIRST) != ACL_SUCCESS) {
    ERROR_LOG("Malloc device memory failed");
  }
}

// 6.执行算子
if (aclnnAddCustom(workspace, workspaceSize, handle, stream) != ACL_SUCCESS) {
  (void)aclrtDestroyStream(stream);
  ERROR_LOG("Execute Operator failed. error code is %d", static_cast<int32_t>(ret));
  return false;
}

// 7.同步等待
aclrtSynchronizeStream(stream);

// 8.处理执行算子后的输出数据，例如在屏幕上显示、写入文件等，由用户根据实际情况自行实现
// ......

// 9.释放运行管理资源
aclRet = aclrtResetDevice(deviceId);
// ....

// 10.去初始化
aclRet = aclFinalize();
```

CMakeLists文件

算子编译后，会生成单算子调用的头文件aclnn_xx.h和动态库libcust_opapi.so。具体路径请参考前置步骤。

编译算子调用程序时，需要在头文件的搜索路径include_directories中增加单算子调用的头文件目录，便于找到该头文件；同时需要链接cust_opapi动态库并在库文件的搜索路径link_directories中增加libcust_opapi.so所在目录。

- 在头文件的搜索路径include_directories中增加单算子调用的头文件目录。以下样例仅为参考，请根据头文件的实际目录位置进行设置。

```cmake
include_directories(
  ${INC_PATH}/runtime/include
  ${INC_PATH}/atc/include
  ../inc
  ${OP_API_PATH}/include
)
```

- 链接cust_opapi链接库。

```cmake
target_link_libraries(execute_add_op
  ascendcl
  cust_opapi
  acl_op_compiler
  nnopbase
  stdc++
)
```

- 在库文件的搜索路径link_directories中增加libcust_opapi.so所在目录。以下样例仅为参考，请根据库文件的实际目录位置进行设置。

```cmake
link_directories(
  ${LIB_PATH}
  ${LIB_PATH1}
  ${OP_API_PATH}/lib
)
```

生成测试数据

在样例工程目录下，执行如下命令：

```
python3 scripts/gen_data.py
```

会在工程目录下input目录中生成两个shape为(8,2048)，数据类型为float16的数据文件input_0.bin与input_1.bin，用于进行AddCustom算子的验证。

代码样例如下：

```python
import numpy as np
a = np.random.randint(100, size=(8, 2048,)).astype(np.float16)
b = np.random.randint(100, size=(8, 2048,)).astype(np.float16)
a.tofile('input_0.bin')
b.tofile('input_1.bin')
```

编译与运行

1. 开发环境上，设置环境变量，配置单算子验证程序编译依赖的头文件与库文件路径，如下为设置环境变量的示例。${INSTALL_DIR}表示CANN软件安装目录，例如$HOME/Ascend/ascend-toolkit/latest。{arch-os}为运行环境的架构和操作系统，arch表示操作系统架构，os表示操作系统，例如x86_64-linux或aarch64-linux。

```bash
export DDK_PATH=${INSTALL_DIR}
export NPU_HOST_LIB=${INSTALL_DIR}/{arch-os}/devlib
```

2. 编译样例工程，生成单算子验证可执行文件。
   a. 切换到样例工程根目录，然后在样例工程根目录下执行如下命令创建目录用于存放编译文件，例如，创建的目录为"build"。
      `mkdir -p build`
   b. 进入build目录，执行cmake编译命令，生成编译文件
      `cd build`
      `cmake ../src -DCMAKE_SKIP_RPATH=TRUE`
   c. 执行如下命令，生成可执行文件。
      `make`
      会在工程目录的output目录下生成可执行文件execute_add_op。

3. 执行单算子
   a. 以运行用户（例如HwHiAiUser）拷贝开发环境中样例工程output目录下的execute_add_op到运行环境任一目录。
      说明：若您的开发环境即为运行环境，此拷贝操作可跳过。
   b. 在运行环境中，执行execute_add_op文件：
      `chmod +x execute_add_op`
      `./execute_add_op`
      会有如下屏显信息：
      ```
      [INFO] Set device[0] success
      [INFO] Get RunMode[1] success
      [INFO] Init resource success
      [INFO] Set input success
      [INFO] Copy input[0] success
      [INFO] Copy input[1] success
      [INFO] Create stream success
      [INFO] Execute aclnnAddCustomGetWorkspaceSize success, workspace size 0
      [INFO] Execute aclnnAddCustom success
      [INFO] Synchronize stream success
      [INFO] Copy output[0] success
      [INFO] Write output success
      [INFO] Run op success
      [INFO] Reset Device success
      [INFO] Destroy resource success
      ```
      如果有Run op success，表明执行成功，会在output目录下生成输出文件output_z.bin。

4. 比较真值文件
   切换到样例工程根目录，然后执行如下命令：
   `python3 scripts/verify_result.py output/output_z.bin output/golden.bin`
   会有如下屏显信息：
   `test pass`
   可见，AddCustom算子验证结果正确。

## 13.3 算子入图（GE图）开发

### 13.3.1 概述

图模式是神经网络模型的一种运行模式，在图模式下用户首先将模型的计算过程构造成一张图，然后通过GE将图下发到昇腾硬件执行。相对于单个算子依次下发的方式，图模式下，GE可以通过计算图优化、多流并行、内存复用、模型下沉等技术手段，加速模型执行效率，减少模型内存占用。

算子入图的开发流程如下图所示：算子工程创建完成后，基于工程代码框架完成算子原型定义、kernel侧算子实现、host侧tiling实现并完成算子入图开发，通过工程编译脚本完成算子的编译部署，之后即可基于图IR执行算子，比如单算子模型执行或者IR构图的方式调用自定义算子。该开发流程以13.2工程化算子开发为基础，除了需要提供工程化算子开发中的算子实现文件外，还需要额外交付算子入图的代码文件。

[图: 算子入图开发流程 - 环境准备(CANN软件安装+创建算子工程) -> 算子实现(算子原型定义+Kernel侧算子实现+Host侧tiling实现) -> 入图开发(算子入图GE图开发) -> 编译部署(算子工程编译部署) -> 图编译和图执行(单算子模型执行/IR构图)]

步骤1 环境准备。
1. CANN软件安装请参考2环境准备。
2. 创建算子工程。使用msOpGen工具创建算子开发工程。

步骤2 算子实现。
- 算子原型定义。通过原型定义来描述算子输入输出、属性等信息以及算子在AI处理器上相关实现信息，并关联tiling实现等函数。
- Kernel侧算子实现和host侧tiling实现请参考10.1 SIMD算子实现；工程化算子开发，支持开发者调用Tiling API基于CANN提供的编程框架进行tiling开发，kernel侧也提供对应的接口方便开发者获取tiling参数，具体内容请参考13.2.4 Kernel侧算子实现和13.2.5 Host侧Tiling实现，由此而带来的额外约束也在上述章节说明。

步骤3 算子入图（GE图）开发。算子入图场景下，需要提供shape推导等算子入图适配函数的实现。

步骤4 编译部署。通过工程编译脚本完成算子的编译部署，分为算子包编译和算子动态库编译两种方式。

步骤5 图编译和图执行：基于图IR执行算子，比如单算子模型执行或者IR构图的方式调用自定义算子。

### 13.3.2 基本开发流程

该开发流程以13.2工程化算子开发为基础，除了需要提供工程化算子开发中的算子实现文件外，还需要额外交付算子入图的代码文件。本节仅提供算子入图代码文件的开发指导。

假设下图是我们需要使用的网络模型，您可能会想直接逐个算子调用，根据输入tensor得到输出tensor就可以完成网络的运行，但在图模式场景下，实际的网络模型生成过程中，会先进行tensor shape以及datatype的推导。这样可以让我们在图执行之前，就知道各tensor的数据类型和形状，提前校验其正确性；同时提前推理出算子的输出张量描述，包括张量的形状、数据类型及数据排布格式等信息，算子构图准备阶段就可以为所有的张量静态分配内存，避免动态内存分配带来的开销。

[图: shape与datatype推导示意图 - 展示Conv2D->ReduceMean->Squeeze网络，输入Data(1,112,112,3)NHWC和Data(3,3,3,64)HWCN经过Conv2D后输出NHWC shape=(1,112,112,64) dtype="float16"，经过ReduceMean后输出NHWC shape=(1,1,1,64) dtype="float16"，经过Squeeze后输出NHWC shape=(64) dtype="float16"]

除了tiling实现外，算子入图时需要额外提供的实现代码有以下几种：

- datatype推导：根据算子的输入datatype、算子逻辑及算子属性等信息，推理出算子的输出张量datatype。
- shape推导：根据算子的输入shape、算子逻辑及算子属性等信息，推理出算子的输出张量shape。
- ShapeRange推导：编译时无法推导输出shape，只能推导输出shape range，执行完才能得到输出shape。在下发时需要按照输出shape range来申请最大输出内存，该类算子需要提供ShapeRange推导函数。
- 声明数据依赖：部分算子在InferShape时，需要依赖某个输入的具体值才可以进行，这类算子被称为"数据依赖算子"，对应的输入被称为"数据依赖输入"。该类算子在注册时，需要声明其数据依赖输入。

表 13-13 不同类型的算子对入图实现代码的要求

| 分类 | 对入图实现代码的要求 |
|------|------------|
| 根据输入shape可以推导出输出shape。 | shape推导、datatype推导 |
| 依赖输入的value才能推导出输出shape，即数据依赖算子。如Reshape算子，依赖shape输入的value才能推导出输出shape。 | shape推导、datatype推导、声明数据依赖 |
| 编译时无法推导输出shape，只能推导输出shape range，执行完才能得到输出shape。 | Shape推导（必选）、DataType推导（必选）、ShapeRange推导（必选）、声明数据依赖（按需） |

实际开发时通过固定的datatype和shape推导原型实现推导函数，然后再通过SetInferShape、SetInferDataType接口来关联对应的shape推导函数，样例如下。

```cpp
namespace ge {
static graphStatus InferShape(gert::InferShapeContext *context)
{
  ...
  return GRAPH_SUCCESS;
}

static graphStatus InferDataType(gert::InferDataTypeContext *context)
{
  ...
  return ge::GRAPH_SUCCESS;
}
} // namespace ge

namespace ops {
class AddCustom : public OpDef {
public:
  AddCustom(const char* name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Input("y")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    this->Output("z")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT16, ge::DT_FLOAT, ge::DT_INT32})
      .Format({ge::FORMAT_ND, ge::FORMAT_ND, ge::FORMAT_ND});
    // 根据用户的算子调用方式决定需不需要注册 图模式调用方式下需要
    this->SetInferShape(ge::InferShape);
    this->SetInferShapeRange(ge::InferShapeRange);
    this->SetInferDataType(ge::InferDataType);
    this->AICore()
        .SetTiling(optiling::TilingFunc);
    // 请替换为实际的昇腾AI处理器型号
    this->AICore().AddConfig("ascendxxx");
  }
};
OP_ADD(AddCustom);
} // namespace ops
```

#### datatype推导

以AddCustom算子为例，InferDataType的实现如下所示。该样例中输出tensor的数据类型与输入tensor的数据类型相同，所以直接将任意一个输入tensor的数据类型赋给输出tensor即可。

```cpp
namespace ge {
static graphStatus InferDataType(gert::InferDataTypeContext* context)
{
  const auto inputDataType = context->GetInputDataType(0);
  context->SetOutputDataType(0, inputDataType);
  return ge::GRAPH_SUCCESS;
}
} // namespace ge
```

如下示例则给出了更灵活的datatype推导样例，当输入的数据类型为DT_INT4时，其输出的数据类型为DT_INT32。

```cpp
ge::graphStatus InferDataTypeForFoo(gert::InferDataTypeContext* context) {
  if (context->GetInputDataType(0) == DT_INT4) {
    context->SetOutputDataType(0, DT_INT32);
  }
}
```

#### shape推导

简单的shape推导逻辑可以使用Follow接口来表达，比如输出shape和输入shape相同的情况。示例如下：输出"y1"Follow输入"x1"场景，指定Follow模式为SHAPE，此时"y1"的shape将会和"x1"保持一致。

```cpp
this->Input("x1")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT, ge::DT_FLOAT})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND});
this->Input("x2")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT, ge::DT_FLOAT})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND});
this->Output("y1")
  .ParamType(REQUIRED)
  .DataType({ge::DT_FLOAT, ge::DT_FLOAT})
  .Format({ge::FORMAT_ND, ge::FORMAT_ND})
  .Follow("x1", FollowType::SHAPE);
```

无法在原型定义中通过Follow表达的情况需要开发者编写InferShape函数，InferShape函数的原型是固定的，接受一个InferShapeContext作为输入，从此context上可以获取到输入、输出的shape指针等内容。输入shape为const类型，因此InferShape时，输入shape是只读、不允许修改的。InferShape成功后，返回ge::GRAPH_SUCCESS，其他返回值默认为推导失败。推导失败后，执行过程结束退出。

以ReShape算子为例，InferShape的实现如下所示。根据第1个输入（shape输入）的值，Reshape算子将第0个输入（x输入）的shape做变换，并输出到其第0个输出（y输出）上。Reshape的InferShape实现为：

```cpp
ge::graphStatus InferShapeForReshape(InferShapeContext *context) {
const gert::Shape *x_shape = context->GetInputShape(0);     // 获取第0个输入的shape
const gert::Tensor *shape_tensor = context->GetInputTensor(1); // 获取第1个输入的tensor
gert::Shape *output_shape = context->GetOutputShape(0);
if (x_shape == nullptr || shape_tensor == nullptr || output_shape == nullptr) {
  // 防御式编程，不应该出现的场景，打印错误并返回失败
  return ge::GRAPH_FAILED;
}

auto reshape_size = static_cast<int32_t>(shape_tensor->GetShapeSize());
if (reshape_size < 1) {
  // 防御式编程，不应该出现的场景，打印错误并返回失败
  return ge::GRAPH_FAILED;
}

// 根据原型信息，Reshape的shape输入支持INT32与INT64两类，根据不同的类型进入对应的模板函数中做真正的shape变换操作
if (shape_tensor->GetDataType() == ge::DT_INT32) {
  int32_t *reshape_data = shape_tensor->GetData<int32_t>();
  return ReshapeInferShapeImpl<int32_t>(reshape_data, *x_shape, *output_shape, reshape_size);
} else {
  int64_t *reshape_data = shape_tensor->GetData<int64_t>();
  return ReshapeInferShapeImpl<int64_t>(reshape_data, *x_shape, *output_shape, reshape_size);
}
}
```

注意：
- InferShape推导函数和Follow接口不能混用，即不支持部分输出采用Infershape推导、部分输出采用Follow推导的情况。若用户同时使用了InferShape函数和Follow接口，以用户的InferShape函数为准，需要保证在InferShape函数中能够推导出所有的输出shape。
- 为了效率考虑，调用InferShape函数时，框架不会为输出shape做初始化，因此，在InferShape函数中，可以认为输出是未初始化的状态。如果在InferShape时，希望通过Append方式操作输出shape，需要先将输出shape的DimNum清零，以防止出现未定义行为。

#### InferShapeRange实现

某些算子的输出Shape在计算完成后才能确定。比如unique算子，其Shape的推导逻辑如下：

给定一维Tensor x，找到其中不重复的元素，返回去重后的Tensor y，输出idx与输入x大小相同，保存x每个元素在y中的索引。

```
# tensor 'x' is [1, 1, 2, 4, 4, 4, 7, 8, 8]     x shape[9]
y, idx = unique(x)
y ==> [1, 2, 4, 7, 8]                             y shape[5]
idx ==> [0, 0, 1, 2, 2, 2, 3, 4, 4]               idx shape[9]
```

由此可知，y的shape在编译时为[-1]，unique执行后shape才确定。

在入图场景执行时，需要在执行前分配输出内存，而内存的大小依赖于输出Shape和数据类型。对于此类算子，由于输出Shape在执行后才能确定，因此需要根据输出Shape的范围，按照最大范围申请输出内存，以确保有足够的空间供计算函数写入输出Tensor。

这种场景下，开发者需要自行实现InferShapeRange函数，来推导输出Shape的范围。下面以unique算子为例子，介绍InferShapeRange函数的实现方法。

```cpp
ge::graphStatus UniqueInferShapeRangeFunc(gert::InferShapeRangeContext *context) {
  // 取输入的shape range
  auto x_shape_range = context->GetInputShapeRange(0U);
  OPS_CHECK_NULL_WITH_CONTEXT(context, x_shape_range);
  OPS_CHECK_NULL_WITH_CONTEXT(context, x_shape_range->GetMax());
  OPS_CHECK_NULL_WITH_CONTEXT(context, x_shape_range->GetMin());

  // 开始计算y输出的shape range
  auto y_shape_range = context->GetOutputShapeRange(0U);
  OPS_CHECK_NULL_WITH_CONTEXT(context, y_shape_range);
  y_shape_range->GetMax()->SetDimNum(1); // 一维向量，rank为1
  y_shape_range->GetMin()->SetDimNum(1);

  auto x_max_shape = x_shape_range->GetMax();
  auto x_shape_dimnum = x_max_shape->GetDim(0); // x为一维Tensor，其shape为[n]，x_shape_dimnum表示x输入的元素个数n
  if (x_shape_dimnum == 1) {
    // 若x输入只有1个元素，不存在去重，y的shape轴最小最大均为1，因此range为[1~1]
    y_shape_range->GetMax()->SetDim(0, 1);
    y_shape_range->GetMin()->SetDim(0, 1);
  } else {
    // 若x输入有0个元素，或者大于1个元素，去重后，y的元素个数最小为x的min，最大为x的max
    y_shape_range->GetMax()->SetDim(0, x_shape_dimnum);
    y_shape_range->GetMin()->SetDim(0, x_shape_range->GetMin()->GetDim());
  }

  // 开始计算输出idx的shape range
  // 输出idx表示x元素在y中的索引，其元素个数与x相等，因此shape range与x一致
  auto idx_shape_range = context->GetOutputShapeRange(1U);
  OPS_CHECK_NULL_WITH_CONTEXT(context, idx_shape_range);
  *(idx_shape_range->GetMax()) = *(x_shape_range->GetMax());
  *(idx_shape_range->GetMin()) = *(x_shape_range->GetMin());

  return ge::GRAPH_SUCCESS;
}
```

#### InferShape时获取属性、输入

在InferShape、Tiling时，可以通过context实例获取算子IR属性值，所谓IR属性，是指在IR注册时定义的属性，以TransData算子为例：

```cpp
namespace ops {
class TransData : public OpDef {
public:
  explicit TransData(const char *name) : OpDef(name)
  {
    this->Input("src")
      ...
    this->Output("dst")
      ...
    this->Attr("src_format")
      .AttrType(REQUIRED)
      .String();
    this->Attr("dst_format")
      .AttrType(REQUIRED)
      .String();
    this->Attr("group")
      .AttrType(OPTIONAL)
      .Int(1);
    ...
  }
};
OP_ADD(TransData);
} // namespace ops
```

其原型定义中声明了src_format、dst_format、group三个属性，可以通过如下方式获取算子属性：

```cpp
ge::graphStatus ExampleGetTransDataAttr(TilingContext *context) {
  // 获取所有属性
  const RuntimeAttrs *attrs = context->GetAttrs();
  ASSERT_NOT_NULL(attrs);

  // 按照在原型定义中的顺序，使用index获取属性，index从0开始计数
  const char *src_format = attrs->GetAttrPointer<char>(0); // 获取src_format，src_format是第一个属性，因此index为0
  const char *dst_format = attrs->GetAttrPointer<char>(1); // 获取dst_format，dst_format是第二个属性，因此index为1
  const int64_t group = attrs->GetAttrPointer<int64_t>(2); // 获取group，group是第三个属性，因此index为2

  return ge::GRAPH_SUCCESS;
}
```

通过index而不是字符串name来索引输入输出，对于带有OPTIONAL、DYNAMIC类型输入的算子，可能出现实例化后，单纯通过index无法索引到具体输入的问题，以DynamicRNNV3算子为例：

```cpp
namespace ops {
class DynamicRNNV3 : public OpDef {
public:
  explicit DynamicRNNV3(const char *name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      ...
    this->Input("w")
      .ParamType(REQUIRED)
      ...
    this->Input("b")
      .ParamType(REQUIRED)
      ...
    this->Input("seq_length")
      .ParamType(OPTIONAL)
      ...
    this->Input("init_h")
      .ParamType(OPTIONAL)
      ...
    this->Input("init_c")
      .ParamType(OPTIONAL)
      ...
    this->Input("wci")
      .ParamType(OPTIONAL)
      ...
    this->Input("wcf")
      .ParamType(OPTIONAL)
      ...
    this->Input("mask")
      .ParamType(OPTIONAL)
      ...
    this->Input("mask")
      .ParamType(OPTIONAL)
      ...
    this->Input("project")
      .ParamType(OPTIONAL)
      ...
    ...
  }
};
OP_ADD(DynamicRNNV3);
} // namespace ops
```

由于DynamicRNNV3算子有连续的多个optional输入，这导致init_h及其后面的输入的实例化后index都是不确定的，对于这种类型的算子，可以通过GetOptionalInputShape传入原型对应的index来获取对应的输入shape等数据，以InferShape为例：

```cpp
ge::graphStatus InferShapeForDynamicRNNV3(InferShapeContext *context) {
  // 对于前两个输入，不受到optional或dynamic的影响，可以按照常规方法获取输入shape
  auto x_shape = context->GetInputShape(0);
  auto w_shape = context->GetInputShape(1);
  if (x_shape == nullptr || w_shape == nullptr) {
    return ge::GRAPH_FAILED;
  }

  int64_t state_size = 0;
  // 在原型定义上，project是第11个输入(从0开始计数)
  constexpr int64_t kProjectInputIndex = 11;

  // 受到前面optional输入影响的，project实例化后输入的index是不确定的，通过GetOptionalInputShape来获取对应的输入shape，
  // GetOptionalInputShape的入参为原型上对应的index
  auto project_shape = context->GetOptionalInputShape(kProjectInputIndex);
  if (project_shape != nullptr) {
    if (project_shape->GetDimNum() < 2) {
      return ge::GRAPH_FAILED;
    }
    state_size = project_shape->GetDim(1);
  }
  // 更多的infershape逻辑...
  return ge::GRAPH_SUCCESS;
}
```

对于dynamic类型的输入，实例化后的输入可能是一到多个，对于此类输入，获取方式为：

```cpp
// ir_index：此输入在原型定义中的index，从0开始计数
// relative_index：该输入实例化后的相对index，从0开始计数，例如某个DYNAMIC_INPUT实例化了3个，要取第二个，那么relative_index = 1
auto shape = context->GetDynamicInputShape(ir_index, relative_index);
```

本节举例的获取optional、dynamic输入的方式，在InferShape、Tiling函数中均可以调用。

#### 数据依赖

一般来说，具备输入shape后，算子可以通过InferShape推导出输出shape。然而部分算子在InferShape时，需要依赖某个输入的具体值才可以进行，这类算子被称为"数据依赖算子"，对应的输入被称为"数据依赖输入"。以Reshape算子为例，其依据shape输入的描述，对输入的shape做调整，因此Reshape算子依赖shape输入的值。这类算子需要在原型定义时通过ValueDepend接口声明对应的输入为数据依赖输入。

```cpp
namespace ops {
class Reshape : public OpDef {
public:
  explicit Reshape(const char *name) : OpDef(name)
  {
    ...
    this->Input("shape")
      .ParamType(REQUIRED)
      ...
      .ValueDepend(REQUIRED) // 声明 ReShape算子的shape输入为数据依赖输入
    ...
  }
};
OP_ADD(Reshape);
} // namespace ops
```

根据第1个输入（shape输入）的值，Reshape算子将第0个输入（x输入）的shape做变换，并输出到其第0个输出（y输出）上。Reshape的InferShape实现为：

```cpp
// shape变换具体实现
template<typename T>
ge::graphStatus ReshapeInferShapeImpl(const T *reshape_dims, const gert::Shape &x_shape, gert::Shape
&output_shape, int32_t reshape_rank) {
  constexpr T UNKNOWN_DIM = -1;

```cpp
  // 将算子输出的维度数设置为reshape后的维度数reshape_rank
  output_shape.SetDimNum(reshape_rank);
  auto x_shape_size = x_shape.GetShapeSize();
  int64_t output_shapesize = 1;
  size_t unknown_dim_idx = std::numeric_limits<size_t>::max();
  for (int32_t i = 0; i < reshape_rank; i++) {
    if (reshape_dims[i] != UNKNOWN_DIM) { // reshape后某一轴的维度值不为-1
      output_shape.SetDim(i, reshape_dims[i]); // 设置输出的维度值为reshape后的维度值
      output_shapesize *= reshape_dims[i]; // 计算当前输出元素数量
    } else {
      output_shape.SetDim(i, 1); // reshape后某一轴的维度值为-1，临时设置输出的维度值为1，后续计算看是否可以推导出确定值，并记录未知维度的索引
      unknown_dim_idx = i;
    }
  }
  if (unknown_dim_idx == std::numeric_limits<size_t>::max() && output_shapesize == x_shape_size) {
    return ge::GRAPH_SUCCESS; // 不存在未知维度，且输出shape size和输入x的shape size一致，直接返回成功
  } else if (unknown_dim_idx != std::numeric_limits<size_t>::max() && x_shape_size % output_shapesize == 0) {
    output_shape.SetDim(unknown_dim_idx, x_shape_size / output_shapesize); // 存在未知维度，shape动态调整未知维度值保持总元素个数不变
    return ge::GRAPH_SUCCESS;
  }
  return ge::GRAPH_FAILED;
}

ge::graphStatus InferShapeForReshape(InferShapeContext *context) {
  const gert::Shape *x_shape = context->GetInputShape(0);     // 获取第0个输入的shape
  const gert::Tensor *shape_tensor = context->GetInputTensor(1); // 获取第1个输入的tensor
  gert::Shape *output_shape = context->GetOutputShape(0);
  if (x_shape == nullptr || shape_tensor == nullptr || output_shape == nullptr) {
    // 防御式编程，不应该出现的场景，打印错误并返回失败
    return ge::GRAPH_FAILED;
  }

  auto reshape_size = static_cast<int32_t>(shape_tensor->GetShapeSize());
  if (reshape_size < 1) {
    // 防御式编程，不应该出现的场景，打印错误并返回失败
    return ge::GRAPH_FAILED;
  }

  // 根据原型信息，Reshape的shape输入支持INT32与INT64两类，根据不同的类型进入对应的模板函数中做真正的shape变换操作
  if (shape_tensor->GetDataType() == ge::DT_INT32) {
    int32_t *reshape_data = shape_tensor->GetData<int32_t>();
    return ReshapeInferShapeImpl<int32_t>(reshape_data, *x_shape, *output_shape, reshape_size);
  } else {
    int64_t *reshape_data = shape_tensor->GetData<int64_t>();
    return ReshapeInferShapeImpl<int64_t>(reshape_data, *x_shape, *output_shape, reshape_size);
  }
}
```

注意：
- 只有声明过数据依赖的输入，才可以在InferShape时调用GetInputTensor等获取tensor的接口获取其对应的tensor数据。若对一个未声明数据依赖的输入调用GetInputTensor等获取tensor的接口，只能在tensor中获取到正确的shape、format、datatype信息，无法获取到真实的tensor数据地址（获取到的地址为nullptr）。
- 从tensor中获取tensor_data时(GetData<int32_t>或GetData<int64_t>)，使用者需要保证获取的数据类型是正确的，否则行为是未定义的。

### 13.3.3 使能Tiling下沉

在静态图模式下，可以通过整图下沉优化调度性能。将完整的计算图一次性下发至Device侧，后续执行则无需Host参与，由Device自主完成计算，从而减少Host-Device交互开销，提升执行效率。部分算子的Tiling计算依赖运行时的输入的具体数值（Tiling值依赖），需在执行时动态计算Tiling参数。针对该场景，可采用Tiling下沉优化方案：将Tiling计算下沉至Device侧的AI CPU上执行，从而实现计算全程在Device侧高效完成。

说明：
- 基于新版本CANN包（支持Tiling下沉特性）编译生成的Tiling下沉算子，不兼容旧版CANN（不支持Tiling下沉特性）运行环境。
- 当前仅融合算子（矢量计算和矩阵计算融合）支持进行Tiling下沉。
- Tiling下沉功能仅支持如下产品型号：
  - Atlas A3训练系列产品/Atlas A3推理系列产品
  - Atlas A2训练系列产品/Atlas A2推理系列产品

自定义算子使能Tiling下沉的步骤如下，完整样例请参考Tiling下沉算子样例。

Tiling下沉场景下，算子工程的op_host目录结构如下，Tiling实现文件需要单独放在在一个cpp文件中，示例中为add_custom_tiling_sink_tiling.cpp。

```
op_host
├── add_custom_tiling_sink.cpp // 算子原型定义、InferShape、InferDataType实现
├── add_custom_tiling_sink_tiling.cpp // Tiling函数实现
├── add_custom_tiling_sink_tiling.h // TilingData结构体定义、Tiling函数声明
└── CMakeLists.txt
```

以AddCustom算子为例，讲解关键代码文件的具体实现方法如下：

- TilingData结构体定义、Tiling函数声明头文件add_custom_tiling_sink_tiling.h
  - 进行TilingData结构体的定义
  - 进行Tiling实现函数的声明

```cpp
#ifndef ADD_CUSTOM_TILING_SINK_TILING_H
#define ADD_CUSTOM_TILING_SINK_TILING_H
#include "register/tilingdata_base.h"
#include "register/op_def_registry.h"

namespace optiling {
BEGIN_TILING_DATA_DEF(TilingSinkTilingData)
TILING_DATA_FIELD_DEF(uint32_t, totalLength);
TILING_DATA_FIELD_DEF(uint32_t, tileNum);
END_TILING_DATA_DEF;

REGISTER_TILING_DATA_CLASS(AddCustomTilingSink, TilingSinkTilingData) // Tiling结构体定义

ge::graphStatus AddCustomSinkTilingFunc(gert::TilingContext* context); // Tiling函数声明
} // namespace optiling
#endif // ADD_CUSTOM_TILING_SINK_TILING_H
```

- 算子原型定义、InferShape、InferDataType实现文件
  add_custom_tiling_sink.cpp，需包含add_custom_tiling_sink_tiling.h，进行Tiling函数和算子原型定义的关联。

  Tiling下沉仅适用于存在Tiling值依赖（即当InferShape不依赖输入值，仅Tiling计算需要输入值）且算子输入为非Const类型的场景，本示例中的输入y通过ValueDepend配置了非Const类型的Tiling值依赖。

```cpp
namespace ops {
class AddCustomTilingSink : public OpDef {
public:
  explicit AddCustomTilingSink(const char *name) : OpDef(name)
  {
    this->Input("x")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT})
      .Format({ge::FORMAT_ND});
    this->Input("y")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT})
      .Format({ge::FORMAT_ND})
      .ValueDepend(OPTIONAL, DependScope::TILING); // 表示输入y为Tiling值依赖
    this->Output("z")
      .ParamType(REQUIRED)
      .DataType({ge::DT_FLOAT})
      .Format({ge::FORMAT_ND});

    this->SetInferShape(ge::InferShape).SetInferDataType(ge::InferDataType);

    this->AICore().SetTiling(optiling::AddCustomSinkTilingFunc); // Tiling函数和算子原型定义的关联

    // 请替换为实际的昇腾AI处理器型号
    this->AICore().AddConfig("ascendxxx");
  }
};
OP_ADD(AddCustomTilingSink);
} // namespace ops
```

- Tiling函数的实现文件add_custom_tiling_sink_tiling.cpp
  - Tiling函数中通过判断值依赖InputTensor即输入y的数据指针是否为空指针来确认当前是否处于编译期。Tiling下沉场景，编译期需要为算子分配内存，包括其所需的workspace。为了保证运行时的高效性，编译期应根据算子的执行需求，合理设置所需的workspace最大值，以避免内存不足或浪费。AddCustomTilingSink样例不需要用户workspace空间，此处设置为固定值仅作为示例。
  - 完成下沉Tiling函数注册：包含device_op_impl_registry.h头文件，使用宏DEVICE_IMPL_OP_OPTILING进行注册。

```cpp
#include "add_custom_tiling_sink_tiling.h"
#include "register/device_op_impl_registry.h"
#include "tiling/platform/platform_ascendc.h"

namespace optiling {
static constexpr uint32_t BLOCK_DIM = 8;
static constexpr uint32_t TILE_NUM = 3;
static constexpr size_t MAX_WORKSPACE_SIZE = 32; // 算子所需用户workspace空间最大值，
// AddCustomTilingSink算子本身逻辑无需用户workspace空间，此处设置为固定值仅作为示例
static constexpr size_t DEFAULT_WORKSPACE_SIZE = 0;
ge::graphStatus AddCustomSinkTilingFunc(gert::TilingContext *context)
{
  TilingSinkTilingData tiling;
  uint32_t totalLength = context->GetInputTensor(0)->GetShapeSize();
  context->SetBlockDim(BLOCK_DIM);
  tiling.set_totalLength(totalLength);
  tiling.set_tileNum(TILE_NUM);
  tiling.SaveToBuffer(context->GetRawTilingData()->GetData(), context->GetRawTilingData()->GetCapacity());
  context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
  size_t *currentWorkspace = context->GetWorkspaceSizes(1);
  auto platform = platform_ascendc::PlatformAscendC(context->GetPlatformInfo());
  size_t sysWorkspaceSize = platform.GetLibApiWorkSpaceSize();
  currentWorkspace[0] = sysWorkspaceSize + DEFAULT_WORKSPACE_SIZE; // 设置运行时workspace大小，此处为系统workspace空间 + 用户workspace空间
  if (context->GetInputTensor(1) != nullptr && context->GetInputTensor(1)->GetData<float>() ==
nullptr) {
    // 通过判断值依赖InputTensor的Data是否为空指针来确认当前是否处于编译期。
    // Tiling下沉场景，编译期需要为算子分配内存，包括其所需的workspace。为了保证运行时的高效性，编译期应根据算子的执行需求，合理设置所需的workspace最大值，以避免内存不足或浪费。
    currentWorkspace[0] = sysWorkspaceSize + MAX_WORKSPACE_SIZE; // 设置编译期workspace大小，此处为系统workspace空间 + 用户workspace空间最大值
  }
  return ge::GRAPH_SUCCESS;
}
DEVICE_IMPL_OP_OPTILING(AddCustomTilingSink).Tiling(optiling::AddCustomSinkTilingFunc); // 下沉Tiling函数注册
} // namespace optiling
```

- 算子核函数实现

  当前Tiling下沉仅支持融合算子场景，为了模拟融合算子场景，通过KERNEL_TASK_TYPE_DEFAULT接口强制指定算子在AIC、AIV混合场景运行。

```cpp
extern "C" __global__ __aicore__ void add_custom_tiling_sink(GM_ADDR x, GM_ADDR y, GM_ADDR z,
GM_ADDR workspace, GM_ADDR tiling)
{
  GET_TILING_DATA(tiling_data, tiling);
  KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_AIC_1_2); // 将算子强制指定在AIC、AIV混合场景运行，模拟融合算子场景
  if ASCEND_IS_AIC {
    return;
  }
  AscendC::KernelAdd op;
  op.Init(x, y, z, tiling_data.totalLength, tiling_data.tileNum);
  op.Process();
}
```

- 修改op_host目录下的编译脚本CMakeLists.txt，添加Tiling下沉编译命令。具体代码如下所示：

```cmake
# 通过ascendc_device_library添加Tiling下沉编译任务
ascendc_device_library( TARGET cust_opmaster # 任务名称，固定为cust_opmaster
              OPTION SHARED # 动态库（当前仅支持动态库入图下沉）
              SRC ${CMAKE_CURRENT_SOURCE_DIR}/add_custom_tiling_sink_tiling.cpp ) # Tiling函数实现代码源文件
```

### 13.3.4 SuperKernel开发

SuperKernel是一种算子的二进制融合技术，与源码融合不同，它聚焦于内核函数(Kernel)的二进制制的调度方案，展开深度优化，于已编译的二进制代码基础上融合创建一个超级Kernel函数（SuperKernel），以调用子函数的方式调用多个其他内核函数，也就是子Kernel。相对于单算子下发，SuperKernel技术可以减少任务调度等待时间和调度开销，同时利用Task间隙资源进一步优化算子头开销。

说明：
- SuperKernel仅适用于静态图场景。
- SuperKernel适用于如下型号：
  - Atlas A3训练系列产品/Atlas A3推理系列产品

自定义算子支持SuperKernel

自定义算子支持SuperKernel与普通算子在开发流程上并无显著差异，但需注意一些特定约束（详见下文）。当前SuperKernel特性仅支持在Pytorch框架使用，所以完成算子入图（GE图）开发后，还需要参考《PyTorch图模式使用指南(TorchAir)》中的"自定义算子入图"章节，完成Pytorch入图。同时，TorchAir提供标定SuperKernel范围的能力，用户可根据实际业务需求对融合范围内的算子进行标记和优化配置。

开发时的特定约束说明如下：

说明：
- 自定义算子若进行全核同步，需注意子Kernel（即该算子）启动的核数与SuperKernel的核数一致。若子Kernel启动的核数少于SuperKernel的核数，全核同步会等待所有核完成，导致卡住超时。
  注：SuperKernel启动核数为子Kernel的最大启动核数。假设SuperKernel包括算子a（启动核数为4）和算子b（启动核数为2），此时SuperKernel启动核数为4。
  - 使用SyncAll时，为了解决该问题，可以通过在标定SuperKernel范围时开启feed-sync-all功能，此时系统会在SuperKernel内子Kernel的其余核中插入SyncAll指令，防止卡住超时。
  - 若使用CrossCoreSetFlag和CrossCoreWaitFlag硬同步接口实现全核同步，仅支持子Kernel启动核数与SuperKernel核数相同。

- 若自定义算子的Kernel类型设置为KERNEL_TYPE_MIX_AIC_1_1，并且算子内部使用了AIC与AIV之间的硬同步接口（CrossCoreSetFlag和CrossCoreWaitFlag），因为SuperKernel会根据启动核数等信息调整SuperKernel的启动比例，此时需特别注意该算子也可以适应SuperKernel的1:2启动比例，确保AIC与AIV之间的硬同步操作正确执行。如：不单独指定某些AIV核调用硬同步接口，使所有AIV核均调用硬同步接口，防止因为硬同步数量不匹配而导致卡死超时。

- 在开发自定义算子时，开发者必须确保所有对GM的标量读写操作都按需正确插入DataCacheCleanAndInvalid指令：在单算子编译场景下，毕昇编译器自动在算子末尾添加DataCacheCleanAndInvalid指令，刷新整个DCache（数据缓存）。在SuperKernel中，子Kernel被当做普通函数处理，编译器不会自动插入该指令，采确保数据缓存一致性。开发者需要自行保证避免因容错机制改变而导致错误。

- 不支持使能Tiling下沉的自定义算子融合成SuperKernel。

- 在子Kernel中调用GetBlockNum接口获取核数时，无论是否融合SuperKernel，获取的核数保持不变，不受SuperKernel启动核数的影响。因此，在使用该接口时，开发者无需特别关注SuperKernel的启动核数，使用方法和开发普通算子时一样。

此外，开发者在进行Kernel侧编程时，可以通过调用SetNextTaskStart和WaitPreTaskEnd两个任务间接口，进一步提升性能。

- 调用SetNextTaskStart后的指令可以和后续其他的子Kernel实现并行，提升整体性能。如图13-10所示，SuperKernel按序调用子Kernel，为保证子Kernel之间数据互不干扰，会在子Kernel间插入算子间同步进行保序，子KernelN-1调用该接口后，之后的指令会和后续子KernelN实现并行。

[图: 通过SetNextTaskStart实现并行示意图 - 子KernelN-1在SetNextTaskStart后，其后续指令与子KernelN和子KernelN+1并行执行]

- 调用WaitPreTaskEnd前的指令可以和前序其他的子Kernel实现并行，提升整体性能。如图13-11所示，SuperKernel按序调用子Kernel，为保证子Kernel之间数据互不干扰，会在子Kernel间插入算子间同步进行保序，子KernelN+1调用该接口之前的指令会和前序子KernelN实现并行。

[图: 通过WaitPreTaskEnd实现并行示意图 - 子KernelN+1在WaitPreTaskEnd之前的指令与子KernelN和子KernelN-1并行执行]

### 13.3.5 图编译和图执行

本节通过单算子模型执行的样例来介绍图模式下图编译和图执行流程。单算子模型执行是指基于图IR执行算子，先编译算子（例如，使用ATC工具将Ascend IR定义的单算子描述文件编译成算子om模型文件），再调用acl接口加载算子模型，最后调用acl接口执行算子。下文仅提供单算子模型执行的样例和基础内容讲解，详细内容请参考单算子模型执行。

环境要求

- 已参考2环境准备，完成CANN驱动和软件的安装，配置CANN软件所需基本环境变量。
  安装CANN软件后，使用CANN运行用户进行编译、运行时，需要以CANN运行用户登录环境，执行source ${install_path}/set_env.sh命令设置环境变量。其中${install_path}为CANN软件的安装目录，例如：/usr/local/Ascend/ascend-toolkit。
- 已参考13.2工程化算子开发完成算子的开发和部署。

准备验证代码工程

代码工程目录结构如下，您可以单击LINK，获取样例工程的完整样例：

```
├── input               // 存放脚本生成的输入数据目录
├── output              // 存放算子运行输出数据和真值数据的目录
├── inc                 // 头文件目录
│   ├── common.h        // 声明公共方法类，用于读取二进制文件
│   ├── operator_desc.h // 算子描述声明文件，包含算子输入/输出，算子类型以及输入描述与输出描述
│   └── op_runner.h     // 算子运行相关信息声明文件，包含算子输入/输出个数，输入/输出大小等
├── src
│   ├── CMakeLists.txt  // 编译规则文件
│   ├── common.cpp      // 公共函数，读取二进制文件函数的实现文件
│   ├── main.cpp        // 将单算子编译为om文件并加载om文件执行
│   ├── operator_desc.cpp  // 构造算子的输入与输出描述
│   └── op_runner.cpp   // 单算子编译与运行函数实现文件
├── scripts
│   ├── verify_result.py  // 真值对比文件
│   ├── gen_data.py     // 输入数据和真值数据生成脚本文件
│   ├── acl.json        // acl配置文件
│   ├── add_custom_static_shape.json  // 算子描述文件，用于构造静态shape单算子模型文件
│   └── add_custom_dynamic_shape.json // 算子描述文件，用于构造动态shape单算子模型文件
```

生成单算子离线模型文件

步骤1 构造静态shape单算子描述文件add_custom_static_shape.json，描述算子的输入、输出及属性等信息。

AddCustom静态shape算子的描述文件示例如下：

```json
[
  {
    "op": "AddCustom",
    "input_desc": [
      {
        "name": "x",
        "param_type": "required",
        "format": "ND",
        "shape": [8, 2048],
        "type": "float16"
      },
      {
        "name": "y",
        "param_type": "required",
        "format":"ND",
        "shape": [8, 2048],
        "type": "float16"
      }
    ],
    "output_desc": [
      {
        "name": "z",
        "param_type": "required",
        "format": "ND",
        "shape": [8, 2048],
        "type": "float16"
      }
    ]
  }
]
```

AddCustom动态shape算子的描述文件示例如下：

```json
[
  {
    "op": "AddCustom",
    "input_desc": [
      {
        "name": "x",
        "param_type": "required",
        "format": "ND",
        "shape": [-1, -1],
        "shape_range": [[1,-1],[1,-1]],
        "type": "float16"
      },
      {
        "name": "y",
        "param_type": "required",
        "format":"ND",
        "shape": [-1, -1],
        "shape_range": [[1,-1],[1,-1]],
        "type": "float16"
      }
    ],
    "output_desc": [
      {
        "name": "z",
        "param_type": "required",
        "format": "ND",
        "shape": [-1, -1],
        "shape_range": [[1,-1],[1,-1]],
        "type": "float16"
      }
    ]
  }
]
```

步骤2 使用ATC工具，将该算子描述文件编译成单算子模型文件（\*.om文件）

ATC工具的命令示例如下：

```
atc --singleop=$HOME/op_verify/run/out/test_data/config/add_custom_static_shape.json --output=op_models/ --soc_version=<soc_version>
```

关键参数解释如下（详细参数说明，请参见《ATC离线模型编译工具用户指南》）：

- --singleop：单算子描述文件（json格式）的路径。
- --output：存放om模型文件的目录。
- --soc_version：配置为AI处理器的型号，请根据实际环境进行替换。

以上命令执行后，会在output参数指定路径下生成\*.om后缀的离线模型文件。

生成测试数据

在样例工程目录下，执行如下命令：

```
python3 scripts/gen_data.py
```

会在input目录下生成两个shape为(8,2048)，数据类型为float16的数据文件input_0.bin与input_1.bin，用于进行AddCustom算子的验证。

代码样例如下：

```python
import numpy as np
a = np.random.randint(100, size=(8, 2048,)).astype(np.float16)
b = np.random.randint(100, size=(8, 2048,)).astype(np.float16)
a.tofile('input_0.bin')
b.tofile('input_1.bin')
```

编写验证代码

您可以参考如下样例编写单算子加载、执行的代码逻辑。

以下是关键步骤的代码示例，不可以直接拷贝编译运行，仅供参考，调用接口后，需增加异常处理的分支，并记录报错日志、提示日志，此处不一一列举。

```cpp
// 1.初始化
aclRet = aclInit("../scripts/acl.json");

// 2.运行管理资源申请
int deviceId = 0;
aclRet = aclrtSetDevice(deviceId);
// 获取软件栈的运行模式，不同运行模式影响后续的接口调用流程（例如是否进行数据传输等）
aclrtRunMode runMode;
bool g_isDevice = false;
aclError aclRet = aclrtGetRunMode(&runMode);
g_isDevice = (runMode == ACL_DEVICE);

// 3.加载单算子模型文件（*.om文件）
// 该目录相对可执行文件所在的目录，例如，编译出来的可执行文件存放在output目录下，此处就表示工程目录下的op_models目录
aclRet = aclopSetModelDir("../op_models");

// 4.设置算子的输入，申请内存，然后读取输入数据input_0.bin与input_1.bin并保存至申请的内存中
// ......

// 5.创建Stream流
aclrtStream stream = nullptr;
aclrtCreateStream(&stream);

// 6.执行算子
// opType表示算子类型名称，例如AddCustom
// numInputs表示算子输入个数，例如AddCustom算子是2个输入
// inputDesc表示算子输入tensor描述的数组，描述每个输入的format、shape、数据类型
// inputs表示算子输入tensor数据
// numOutputs表示算子输出个数，例如AddCustom算子是1个输出
// outputDesc表示算子输出tensor描述的数组，描述每个输出的format、shape、数据类型
// outputs表示算子输出tensor数据
// attr表示算子属性，如果算子没有属性，也需要调用aclopCreateAttr接口创建aclopAttr类型的数据
// stream用于维护一些异步操作的执行顺序
aclopExecuteV2(opType, numInputs, inputDesc, inputs,
        numOutputs, outputDesc, outputs, attr, nullptr);

// 7.阻塞应用运行，直到指定Stream中的所有任务都完成
aclrtSynchronizeStream(stream);

// 8.处理执行算子后的输出数据，例如在屏幕上显示、写入文件等，由用户根据实际情况自行实现，本示例会将结果写入output_z.bin文件中
// ......

// 9.释放stream流
aclrtDestroyStream(stream);

// 10.释放运行管理资源
aclRet = aclrtResetDevice(deviceId);
aclRet = aclFinalize();
// ....
```

运行和验证

1. 开发环境上，设置环境变量，配置单算子验证程序编译依赖的头文件与库文件路径，如下为设置环境变量的示例。${INSTALL_DIR}表示CANN软件安装目录，例如$HOME/Ascend/ascend-toolkit/latest。{arch-os}为运行环境的架构和操作系统，arch表示操作系统架构，os表示操作系统，例如x86_64-linux。

```bash
export DDK_PATH=${INSTALL_DIR}
export NPU_HOST_LIB=${INSTALL_DIR}/{arch-os}/devlib
```

2. 编译样例工程，生成单算子验证可执行文件。
   a. 切换到样例工程根目录，然后在样例工程根目录下执行如下命令创建目录用于存放编译文件，例如，创建的目录为"build"。
      `mkdir -p build`
   b. 进入build目录，执行cmake编译命令，生成编译文件
      `cd build`
      `cmake ../src -DCMAKE_SKIP_RPATH=TRUE`
   c. 执行如下命令，生成可执行文件。
      `make`
      会在工程目录的output目录下生成可执行文件execute_add_op。

3. 执行单算子
   a. 以运行用户（例如HwHiAiUser）拷贝开发环境中样例工程output目录下的execute_add_op到运行环境任一目录。
      说明：若您的开发环境即为运行环境，此拷贝操作可跳过。
   b. 在运行环境中，执行execute_add_op文件，验证单算子模型文件。
      `chmod +x execute_add_op`
      `./execute_add_op`
      会有如下屏显信息：
      ```
      [INFO] static op will be called
      [INFO] Set device[0] success
      [INFO] Get RunMode[1] success
      [INFO] aclopSetModelDir op model success
      [INFO] Init resource success
      [INFO] Set input success
      [INFO] Copy input[0] success
      [INFO] Copy input[1] success
      [INFO] Create stream success
      [INFO] Execute AddCustom success
      [INFO] Synchronize stream success
      [INFO] Copy output[0] success
      [INFO] Write output success
      [INFO] Run op success
      [INFO] Reset Device success
      [INFO] Destroy resource success
      ```
      如果有Run op success，表明执行成功，会在output目录下生成输出文件output_z.bin。

4. 比较真值文件
   切换到样例工程根目录，然后执行如下命令：
   `python3 scripts/verify_result.py output/output_z.bin output/golden.bin`
   会有如下屏显信息，可见AddCustom算子验证结果正确。
   `test pass`

## 13.4 AI框架算子适配

### 13.4.1 概述

本章节内容介绍AI框架调用自定义算子的方法。如下图所示，Pytorch支持单算子和图模式两种，TensorFlow仅支持图模式。

AI框架调用时，除了需要提供CANN框架调用时需要的代码实现文件，还需要进行插件适配开发。

[图: AI框架调用架构图 - 展示Pytorch调用和TensorFlow调用两条路径。Pytorch适配层通过插件适配开发连接到CANN层（包含算子原型定义、Kernel侧算子实现、Host侧Tiling实现），支持单算子和图模式（包含算子入图GE图开发）。TensorFlow适配层通过插件适配开发连接到CANN层的图模式。底层为Runtime和AI处理器。]

### 13.4.2 PyTorch框架

通过PyTorch框架进行模型的训练、推理时，会调用很多算子进行计算。开发者开发的自定义算子如果需要集成部署到PyTorch框架，有如下几种方式：

- Kernel直调：通过适配Pybind调用，可以实现PyTorch框架调用算子Kernel程序。
- 单算子API调用：该模式下的适配插件开发流程和具体样例参见《Ascend Extension for PyTorch框架特性指南》中的"基于OpPlugin算子适配开发"章节。
- 图模式调用：自定义算子在Pytorch图模式下的适配开发指导请参考《PyTorch图模式使用指南(TorchAir)》中的"自定义算子入图"章节。

[图: Pytorch框架部署方式 - 展示三种算子部署方式：Kernel直调（嵌入方式/嵌入应用程序，自立自闭的体验版）、单算子API调用（依赖CANN算子库）、算子入图（依赖GE图框架），均通过AI框架算子适配连接到PyTorch框架]

本节主要提供通过Pybind实现PyTorch框架调用算子Kernel程序的指导。

Pybind调用介绍

Pybind是一个用于将C++代码与Python解释器集成的库，实现原理是通过将C++代码编译成动态链接库（DLL）或共享对象（SO）文件，使用Pybind提供的API将算子核函数与Python解释器进行绑定。在Python解释器中使用绑定的C++函数、类和变量，从而实现Python与C++代码的交互。

环境准备

基于2环境准备，还需要安装以下依赖：
- 安装PyTorch框架
- 安装torch_npu插件
- 安装pybind11
  `pip3 install pybind11`

工程目录

样例工程目录结构如下所示：

```
├── add_custom_test.py  // Python调用脚本
├── add_custom.asc      // 算子实现 + pybind11函数封装
└── CMakeLists.txt      // 编译工程文件
```

基于该算子工程，开发者进行算子开发的步骤如下：

- 完成算子Kernel侧实现；编写算子调用应用程序并定义pybind模块。这两部分代码均在add_custom.asc中实现。
- 编写Python调用脚本add_custom_test.py，包括生成输入数据和真值数据，调用封装的模块以及验证结果。
- 编写CMake编译配置文件CMakeLists.txt。
- 完成算子的编译运行和结果验证。

算子Kernel实现、调用和Pybind模块定义

下面代码以add_custom算子为例，介绍算子核函数实现及调用的应用程序add_custom.asc如何编写。您在实现自己的应用程序时，需要关注由于算子核函数不同带来的修改，包括算子核函数名、入参出参的不同等，相关API的调用方式直接复用即可。同时该文件中定义了Pybind模块，便于后续在Python脚本中调用。

步骤1 按需包含头文件。

```cpp
// Pybind和PyTorch调用所需的头文件
#include <pybind11/pybind11.h>
#include <torch/extension.h>

#include "torch_npu/csrc/core/npu/NPUStream.h"
// Kernel侧实现需要的头文件
#include "kernel_operator.h"
```

步骤2 算子Kernel实现。

```cpp
constexpr int32_t BUFFER_NUM = 2; // tensor num for each queue

class KernelAdd {
public:
  __aicore__ inline KernelAdd() {}
  __aicore__ inline void Init(GM_ADDR x, GM_ADDR y, GM_ADDR z, uint32_t totalLength)
  {
    this->blockLength = totalLength / AscendC::GetBlockNum();
    this->tileNum = 8;
    this->tileLength = this->blockLength / this->tileNum / BUFFER_NUM;
    xGm.SetGlobalBuffer((__gm__ half *)x + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    yGm.SetGlobalBuffer((__gm__ half *)y + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    zGm.SetGlobalBuffer((__gm__ half *)z + this->blockLength * AscendC::GetBlockIdx(), this->blockLength);
    pipe.InitBuffer(inQueueX, BUFFER_NUM, this->tileLength * sizeof(half));
    pipe.InitBuffer(inQueueY, BUFFER_NUM, this->tileLength * sizeof(half));
    pipe.InitBuffer(outQueueZ, BUFFER_NUM, this->tileLength * sizeof(half));
  }
  __aicore__ inline void Process()
  {
    int32_t loopCount = this->tileNum * BUFFER_NUM;
    for (int32_t i = 0; i < loopCount; i++) {
      CopyIn(i);
      Compute(i);
      CopyOut(i);
    }
  }

private:
  // ... CopyIn, Compute, CopyOut 实现
  AscendC::TPipe pipe;
  AscendC::TQue<AscendC::TPosition::VECIN, BUFFER_NUM> inQueueX, inQueueY;
  AscendC::TQue<AscendC::TPosition::VECOUT, BUFFER_NUM> outQueueZ;
  AscendC::GlobalTensor<half> xGm, yGm, zGm;
  uint32_t blockLength, tileNum, tileLength;
};

__global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, uint32_t totalLength)
{
  KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIV_ONLY);
  KernelAdd op;
  op.Init(x, y, z, totalLength);
  op.Process();
}
```

步骤3 算子调用程序编写，使用<<<>>>接口调用算子核函数完成指定的运算。样例中的c10_npu::getCurrentNPUStream接口用于获取当前npu流，返回值类型NPUStream，使用方式请参考《Ascend Extension for PyTorch自定义API参考》中的"（beta）c10_npu::getCurrentNPUStream"章节。

需要注意的是，本样例的输入x、y的内存是在Python调用脚本add_custom_test.py中分配的。

```cpp
namespace my_add {
at::Tensor run_add_custom(const at::Tensor &x, const at::Tensor &y)
{
  // 运行资源申请，通过c10_npu::getCurrentNPUStream()的函数获取当前NPU上的流
  auto aclStream = c10_npu::getCurrentNPUStream().stream(false);
  // 分配Device侧输出内存
  at::Tensor z = at::empty_like(x);
  uint32_t blockDim = 8;
  uint32_t totalLength = 1;
  for (uint32_t size : x.sizes()) {
    totalLength *= size;
  }
  // 用<<<>>>接口调用核函数完成指定的运算
  auto xGm = static_cast<uint8_t *>(const_cast<void *>(x.storage().data()));
  auto yGm = static_cast<uint8_t *>(const_cast<void *>(y.storage().data()));
  auto zGm = static_cast<uint8_t *>(const_cast<void *>(z.storage().data()));
  add_custom<<<blockDim, nullptr, aclStream>>>(xGm, yGm, zGm, totalLength);
  // 将Device上的运算结果拷贝回Host并释放申请的资源
  return z;
}
} // namespace my_add
```

步骤4 定义Pybind模块，将C++函数封装成Python函数。PYBIND11_MODULE是Pybind11库中的一个宏，用于定义一个Python模块。它接受两个参数，第一个参数是封装后的模块名，第二个参数是一个Pybind11模块对象，用于定义模块中的函数、类、常量等。通过调用m.def()方法，可以将步骤3中函数my_add::run_add_custom()转成Python函数run_add_custom，使其可以在Python代码中被调用。

```cpp
PYBIND11_MODULE(add_custom, m) { // 模块名add_custom，模块对象m
  m.doc() = "add_custom pybind11 interfaces"; // optional module docstring
  m.def("run_add_custom", &my_add::run_add_custom, ""); // 将函数run_add_custom与Pybind模块进行绑定
}
```

Python调用脚本

在Python调用脚本中，使用torch接口生成随机输入数据并分配内存，通过导入封装的自定义模块add_custom，调用自定义模块add_custom中的run_add_custom函数，从而在NPU上执行算子。算子核函数NPU侧运行验证的步骤如图13-13。

[图: NPU侧运行验证原理图 - 分配Host侧输入内存并进行数据初始化 -> 分配Device侧输入内存并将数据从Host上拷贝到Device上 -> 调用pybind接口从而调用其绑定的核函数 -> 运行管理资源申请 -> 分配Device侧输出内存 -> 通过<<<>>>接口完成指定的运算并同步等待 -> 将Device上的运算结果拷贝回Host -> 释放申请的资源]

```python
import sys
import os
import torch
import torch_npu
from torch_npu.testing.testcase import TestCase, run_tests
sys.path.append(os.getcwd())
import add_custom
torch.npu.config.allow_internal_format = False
class TestCustomAdd(TestCase):
    def test_add_custom_ops(self):
        # 分配Host侧输入内存，并进行数据初始化
        length = [8, 2048]
        x = torch.rand(length, device='cpu', dtype=torch.float16)
        y = torch.rand(length, device='cpu', dtype=torch.float16)
        # 分配Device侧内存，并将数据从Host上拷贝到Device上
        x_npu = x.npu()
        y_npu = y.npu()
        output = add_custom.run_add_custom(x_npu, y_npu)
        cpuout = torch.add(x, y)
        self.assertRtolEqual(output, cpuout)
if __name__ == "__main__":
    run_tests()
```

CMake编译配置文件编写

该示例中通过CMake脚本生成算子对应的动态库后，Python中会通过import加载该动态库后执行计算。如果需要了解算子编译更多内容，请参考5.1.4通过CMake编译。

```cmake
cmake_minimum_required(VERSION 3.16)

find_package(ASC REQUIRED HINTS $ENV{ASCEND_INSTALL_PATH}/compiler/tikcpp/ascendc_kernel_cmake)

execute_process(COMMAND python3 -c "import os; import torch; print(os.path.dirname(torch.__file__))"
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE TORCH_PATH
)
message("TORCH_PATH is ${TORCH_PATH}")

execute_process(COMMAND python3 -c "import os; import torch_npu;
print(os.path.dirname(torch_npu.__file__))"
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE TORCH_NPU_PATH
)
message("TORCH_NPU_PATH is ${TORCH_NPU_PATH}")

execute_process(COMMAND python3 -m pybind11 --includes
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE PYBIND11_INC
)
string(REPLACE " " ";" PYBIND11_INC ${PYBIND11_INC})

execute_process(COMMAND python3-config --extension-suffix
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE PYBIND11_SUFFIX
)

project(kernel_samples LANGUAGES ASC CXX)

add_library(pybind11_lib SHARED
  add_custom.asc
)

target_link_libraries(pybind11_lib PRIVATE
  torch_npu
)

target_link_directories(pybind11_lib PRIVATE
  ${TORCH_PATH}/lib
  ${TORCH_NPU_PATH}/lib
)

target_include_directories(pybind11_lib PRIVATE
  ${TORCH_NPU_PATH}/include
  ${TORCH_PATH}/include
  ${TORCH_PATH}/include/torch/csrc/api/include
)

target_compile_definitions(pybind11_lib PRIVATE
  _GLIBCXX_USE_CXX11_ABI=0
)

target_compile_options(pybind11_lib PRIVATE
  ${PYBIND11_INC}
  $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=dav-2201>
  -fPIC
)

set_target_properties(pybind11_lib PROPERTIES
  OUTPUT_NAME add_custom${PYBIND11_SUFFIX}
  PREFIX "" SUFFIX ""
)
```

编译和运行程序

完成上述文件的编写后，可以执行如下命令编译和运行应用程序。

```bash
rm -rf build; mkdir -p build; cd build  # 创建并进入build目录
cmake ..; make -j                        # 编译算子so
python3 ../add_custom_test.py            # 执行样例
```

### 13.4.3 ONNX框架

#### 13.4.3.1 适配插件开发

说明：针对Atlas A3训练系列产品/Atlas A3推理系列产品，暂不支持ONNX框架算子调用。

您可以参考本章节进行算子适配插件的开发，将ONNX框架的算子映射成适配昇腾AI处理器的算子（下文简称CANN算子），从而完成从ONNX框架调用Ascend C自定义算子的过程。

完成算子工程创建后，会在算子工程目录下生成framework/onnx_plugin目录，用于存放ONNX框架适配插件实现文件。以自定义CANN算子LeakyReluCustom为例，算子工程目录如下：

```
LeakyReluCustom
├── build.sh         // 编译入口脚本
├── cmake
├── CMakeLists.txt   // 算子工程的CMakeLists.txt
├── CMakePresets.json // 编译配置项
├── framework        // 框架适配插件实现文件目录
│   └── onnx_plugin  // ONNX框架适配插件实现文件目录
│       ├── CMakeLists.txt
│       └── leaky_relu_custom_plugin.cc // ONNX框架适配插件实现文件
├── op_host          // host侧实现文件
├── op_kernel        // kernel侧实现文件
└── scripts          // 自定义算子工程打包相关脚本所在目录
```

下文主要讲解ONNX框架适配插件实现文件（leaky_relu_custom_plugin.cc）的开发流程。

```cpp
#include "register/register.h"
#include "graph/operator.h"
#include "json.hpp"
namespace domi {
  Status ParseParamByOpFunc(const ge::Operator& op_src, ge::Operator& op_dest) {
    //...
  }
  REGISTER_CUSTOM_OP("OpType")
    .FrameworkType(ONNX)
    .OriginOpType("OriginOpType")
    .ParseParamsByOperatorFn(ParseParamByOpFunc) //用来注册解析算子属性的函数
    .ImplType(ImplType::TVM); // Ascend C算子实现类型设置为TVM
}
```

步骤1 包含所需头文件。
- register.h，存储在CANN软件安装后文件存储路径的"include/register/"目录下，包含该头文件，可以使用算子注册相关类，调用算子注册相关的接口。
- operator.h（可选），存储在CANN软件安装后文件存储路径的"include/graph/"目录下，包含该头文件，可以使用Operator类相关接口，获取算子输入输出及属性等算子信息。
- json.hpp，用于进行ONNX数据定义的解析，将String类型的算子参数定义转换为json格式。若样例工程中未提供"json.hpp"文件，用户可以自行下载，并将"json.hpp"放在工程可以找到的任意路径下，然后包含此头文件即可，下载路径可参见json.hpp。

步骤2 使用REGISTER_CUSTOM_OP宏，完成CANN算子和ONNX框架的算子映射关系注册。使用方法如下：
- REGISTER_CUSTOM_OP：注册自定义算子，OpType为算子类型名称，需要与算子原型注册中的OpType保持一致。
- FrameworkType：ONNX代表原始框架为ONNX。
- OriginOpType：算子在原始框架中的类型。例如自定义算子OpTypeA，对应ONNX算子库版本opset_version=11，应传入"ai.onnx::11::OpTypeA"，当前支持的ONNX版本范围为9~15。
- ParseParamsByOperatorFn(ParseParamByOpFunc)：用来注册解析算子参数实现映射关系的回调函数，需要用户自定义实现回调函数ParseParamByOpFunc。具体实现方式参考步骤3。
- ImplType：指定算子的实现方式。Ascend C算子实现类型设置为TVM。

步骤3 实现回调函数ParseParamByOpFunc。其函数声明如下所示：

```cpp
Status ParseParamByOpFunc(const ge::Operator& op_src, ge::Operator& op_dest)
```

- ParseParamByOpFunc：函数名称，用户自定义。
- op_src：ONNX框架定义的Operator类对象，包含ONNX模型中自定义的算子属性信息，定义来源于ONNX框架的原始模型文件。
- op_dest：CANN算子数据结构，保存算子信息，Operator类的详细描述请参见Operator类。

开发者需要在回调函数中实现属性的解析和映射，具体实现方式如下：

ONNX原始模型中，属性为repeated message类型，对于repeated message类型的参数，可使用GetAttr(const char \*name, ge::AscendString &attr_value)接口获取其属性值，然后将AscendString类型的属性值转换为String类型，再将其转换为json格式进行属性字段的解析。

实现如下所示：

```cpp
Status ParseParamLeakyReluAscend(const ge::Operator& op_src, ge::Operator& op_dest) {
  float negative_slope = 0.01f;
  string negative_slope_str;
  AscendString attrs_string;
  // 使用固定属性名称"attribute"获取ONNX算子中的属性，并赋值给AscendString类型对象
  if (ge::GRAPH_SUCCESS == op_src.GetAttr("attribute", attrs_string)) {
    // 转换为json格式
    json attrs = json::parse(attrs_string.GetString());
    for (json attr : attrs["attribute"]) {
      if (attr["name"] == "alpha" && attr["type"] == kTypeFloat) {
        negative_slope_str = attr["f"]; // float type in json has accuracy loss, so we use string type to store it
        negative_slope = atof(negative_slope_str.c_str());
      }
    }
  }
  op_dest.SetAttr("negative_slope", negative_slope);
  return SUCCESS;
}
```

须知：
- 当前版本GetAttr与SetAttr接口不支持对原始文件中数据类型为double和uint64的字段进行解析。
- 使用ATC工具执行模型转换时，对属性的获取情况不会进行强校验。所以进行算子适配插件实现时，若用户调用GetAttr失败，建议根据算子实际情况增加相应的处理逻辑，例如，针对必选属性，可返回失败，针对可选属性，可设置默认值。

#### 13.4.3.2 调用样例

完成了ONNX框架的适配插件开发后，即可实现从ONNX框架调用Ascend C自定义算子。下面以一个仅包含LeakyRelu算子的ONNX框架网络为例（该网络中的LeakyRelu算子通过适配插件映射为自定义的LeakyRelu算子），呈现一个使用推理工具进行推理的过程，目的在于让您快速体验推理场景下网络中自定义算子调用的过程。

在完成如下步骤之前，您需要先参考上文内容完成自定义LeakyRelu算子kernel侧和host侧的开发、ONNX适配插件的开发，并完成算子的编译部署。LeakyRelu算子实现的完整样例请参考LINK。ONNX框架调用的完整示例请参考LINK。

步骤1 通过如下命令获取ONNX框架网络模型。作为示例，该模型中仅包含一个LeakyRelu算子。
`wget https://obs-9be7.obs.cn-east-2.myhuaweicloud.com/AscendC/leaky_relu.onnx`

步骤2 执行如下命令生成离线模型。（如下命令中使用的目录以及文件均为样例，请以实际为准）
`atc --model=$HOME/module/leaky_relu.onnx --framework=5 --soc_version=<soc_version> --output=$HOME/module/out/leaky_relu --input_shape="X:8,16,1024" --input_format=ND`

关键参数的解释如下：
- --model：ONNX框架网络模型文件（\*.onnx）的路径。
- --framework：原始框架类型。5表示ONNX。
- --output：转换后的离线模型的路径以及文件名。请注意，记录保存该om模型文件的路径，后续开发应用时需要使用。
- --soc_version：昇腾AI处理器的型号。
- --input_shape：指定模型输入数据的shape，请基于算子支持的shape范围和实际使用场景进行设置，这里设置输入X为固定shape [8,16,1024]。
- --input_format：指定模型输入数据的格式，请基于算子支持的格式和实际使用场景进行设置，这里配置为ND。

步骤3 使用export ASCEND_GLOBAL_LOG_LEVEL=1改变日志级别为INFO，若出现如下提示信息，则说明进入了Ascend C自定义算子编译流程且模型转换成功。
```
...
start compile Ascend C operator LeakyReluCustom. kernel name is leaky_relu_custom
compile Ascend C operator: LeakyReluCustom success!
...
ATC run success
```
成功执行命令后，在--output参数指定的路径下，可查看离线模型（如：leaky_relu.om）。

步骤4 准备符合模型输入要求的\*.bin格式的输入数据，单击LINK，获取msame工具，参考该工具配套的README，使用推理工具快速完成推理体验，样例如下。
`./msame --model "$HOME/module/out/leaky_relu.om" --output "$HOME/module/out/" --outfmt TXT --loop 1`

### 13.4.4 TensorFlow框架

本章节介绍TensorFlow框架算子适配的流程，用于将TensorFlow框架的算子映射成CANN算子（开发者基于CANN框架自定义开发的算子），从而完成从TensorFlow框架调用到CANN算子的过程。同时给出TensorFlow框架侧算子调用的示例，便于开发者了解完整流程。

[图: TensorFlow框架算子适配开发流程图，包含环境准备(CANN软件安装、创建算子工程)、算子实现与入图开发(算子原型定义、Kernel侧算子实现、Host侧tiling实现、算子入图GE图开发)、框架适配(TensorFlow框架适配插件开发、判断是否需要TensorFlow自定义算子)、编译部署(算子工程编译部署)、算子调用(TensorFlow框架算子调用)]

步骤1 环境准备。
1. CANN软件安装请参考2 环境准备。
2. 创建算子工程。使用msOpGen工具创建算子开发工程。TensorFlow框架算子适配场景下，需要通过framework参数指定具体的框架为tf或者tensorflow，工具会自动生成框架适配代码。以自定义CANN算子AddCustom为例，使用msOpGen工具创建算子开发工程的具体命令如下：
`${INSTALL_DIR}/python/site-packages/bin/msopgen gen -i $HOME/sample/add_custom.json -f tf -c ai_core-<soc_version> -lan cpp -out $HOME/sample/AddCustom`

步骤2 算子实现。
- 算子原型定义。通过原型定义来描述算子输入输出、属性等信息以及算子在AI处理器上相关实现信息，并关联tiling实现等函数。
- Kernel侧算子实现和host侧tiling实现请参考10.1 SIMD算子实现；工程化算子开发，支持开发者调用Tiling API基于CANN提供的编程框架进行tiling开发，kernel侧也提供对应的接口方便开发者获取tiling参数，具体内容请参考13.2.4 Kernel侧算子实现和13.2.5 Host侧Tiling实现，由此而带来的额外约束也在上述章节说明。

步骤3 算子入图（GE图）开发。算子入图场景下，需要提供shape推导等算子入图适配函数的实现。

步骤4 TensorFlow框架适配插件开发。详细说明见适配插件开发。

步骤5 编译部署。通过工程编译脚本完成算子的编译部署，分为算子包编译和算子动态库编译两种方式。

步骤6 TensorFlow框架算子调用。详细说明见TensorFlow原生算子映射到CANN算子和TensorFlow自定义算子开发并映射到CANN算子。完整样例请参考LINK。

----结束

#### 适配插件开发

完成算子工程创建后，会在算子工程目录下生成framework/tf_plugin目录，用于存放TensorFlow框架适配插件实现文件。以自定义CANN算子AddCustom为例，算子工程目录如下：

```
AddCustom
├── build.sh              // 编译入口脚本
├── cmake
├── CMakeLists.txt        // 算子工程的CMakeLists.txt
├── CMakePresets.json      // 编译配置项
├── framework              // 框架适配插件实现文件目录
│   └── tf_plugin          // TensorFlow框架适配插件实现文件目录
│       ├── CMakeLists.txt
│       ├── tensorflow_add_custom_plugin.cc  // TensorFlow框架适配插件实现文件
│       └── CMakeLists.txt
├── op_host                // host侧实现文件
├── op_kernel              // kernel侧实现文件
└── scripts                // 自定义算子工程打包相关脚本所在目录
```

当TensorFlow算子与CANN算子原型定义一致时，TensorFlow框架适配插件实现代码如下：

```cpp
#include "register/register.h"
namespace domi {
REGISTER_CUSTOM_OP("AddCustom")
  .FrameworkType(TENSORFLOW)
  .OriginOpType("AddCustom")
  .ParseParamsByOperatorFn(AutoMappingByOpFn);
}
```

当TensorFlow算子与CANN算子原型定义不一致时，TensorFlow框架适配插件实现代码如下：

```cpp
#include "register/register.h"
REGISTER_CUSTOM_OP("FlashAttentionScore")
  .FrameworkType(TENSORFLOW)
  .OriginOpType({"FlashAttentionScore"})
  .ParseParamsByOperatorFn(FlashAttentionScoreMapping)
  .ParseOpToGraphFn(AddOptionalPlaceholderForFA);
```

- 包含插件实现函数相关的头文件。register.h存储在CANN软件安装后文件存储路径的"include/register/"目录下，包含该头文件，可使用算子注册相关类，调用算子注册相关的接口。
- REGISTER_CUSTOM_OP：注册自定义算子，传入算子的OpType，需要与算子原型注册中的OpType保持一致。
- FrameworkType：TENSORFLOW代表原始框架为TensorFlow。
- OriginOpType：算子在原始框架中的类型。对于TensorFlow自定义算子，还需要完成TensorFlow自定义算子的开发，这里的OriginOpType与REGISTER_OP注册算子名相同，对于TensorFlow原生算子，即为原生算子名。
- ParseParamsByOperatorFn：用来注册解析算子参数实现映射关系的回调函数，需要用户自定义实现回调函数ParseParamByOpFunc。原始TensorFlow算子中参数与CANN算子中参数一一对应时，可直接使用自动映射回调函数AutoMappingByOpFn自动实现映射。
- ParseOpToGraphFn：当TensorFlow算子与CANN算子原型定义不一致（比如CANN算子原型定义中有可选输入，但TensorFlow原型定义中不支持可选输入，没有可选输入）的情况时，用来注册调整算子原型映射关系的回调函数。

#### TensorFlow原生算子映射到CANN算子

以自定义算子AddCustom为例，将该算子映射到TensorFlow内置算子Add上，需要先修改AddCustom自定义算子目录framework/tf_plugin下插件代码，完成算子名映射：

```cpp
#include "register/register.h"
namespace domi {
REGISTER_CUSTOM_OP("AddCustom")   // 当前Ascend C自定义算子名
  .FrameworkType(TENSORFLOW)       // 第三方框架类型TENSORFLOW
  .OriginOpType("Add")             // 映射到TensorFlow原生算子Add
  .ParseParamsByOperatorFn(AutoMappingByOpFn);
}
```

完成算子工程的编译部署后，构造单算子的TensorFlow 1.15版本测试用例进行验证。

步骤1 编写测试用例"tf_add.py"。

步骤2 导入python库。

```python
import logging           # Python标准库日志模块
import tensorflow as tf  # 导入TensorFlow开源库
from npu_bridge.estimator import npu_ops  # 导入TensorFlow开源库中的npu_ops模块
import numpy as np       # 导入Python的数学基础库
```

步骤3 通过config()定义昇腾AI处理器和CPU上的运行参数。

当"execute_type"为"ai_core"时，代表在昇腾AI处理器上运行单算子网络，最终会调用到Ascend C算子。
当"execute_type"为"cpu"时，代表在Host侧的CPU运行单算子网络，调用的是TensorFlow算子。

```python
def config(execute_type):
    if execute_type == 'ai_core':
        session_config = tf.ConfigProto(
            allow_soft_placement=True,
            log_device_placement=False,)
        custom_op = session_config.graph_options.rewrite_options.custom_optimizers.add()
        custom_op.name = "NpuOptimizer"
        custom_op.parameter_map["enable_data_pre_proc"].b = True  # 开启数据预处理下沉到Device侧执行
        custom_op.parameter_map["mix_compile_mode"].b = True
        custom_op.parameter_map["use_off_line"].b = True   # True表示在昇腾AI处理器上执行训练
    elif execute_type == 'cpu':
        session_config = tf.ConfigProto(
            allow_soft_placement=True,
            log_device_placement=False)
    return session_config
```

步骤4 单算子网络测试用例主函数。
- 算子输入请根据算子实际输入个数及shape进行构造。
- 算子输出的计算，请根据算子逻辑调用TensorFlow相关接口进行实现。

```python
# 设置np.allclose比较函数的公差参数。
atol = 0.001
rtol = 0.001

def main(unused_argv):
    shape_params = (8, 2048)
    dtype_params = np.float16
    # 构造Add算子的两个输入数据,shape为shape_params，范围在[-2,2]之间的随机数
    x_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    y_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    # 分别对Add算子的两个输入数据进行占位
    x = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    y = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    # 计算算子输出
    out = tf.math.add(x, y)
    # 在Host侧CPU上运行单算子，得到期望运行结果
    with tf.compat.v1.Session(config=config('cpu')) as session:
        result_cpu = session.run(out, feed_dict={x: x_data, y: y_data})
    # 在昇腾AI处理器上运行单算子，得到实际运行结果
    with tf.compat.v1.Session(config=config('ai_core')) as session:
        result_ai_core = session.run(out, feed_dict={x: x_data, y: y_data})

    np.array(result_ai_core).astype(dtype_params)
    np.array(result_cpu).astype(dtype_params)
    print('====================================')
    # 通过np.allclose比较昇腾AI处理器上运行的实际结果和cpu上运行的期望结果
    cmp_result = np.allclose(result_ai_core, result_cpu, atol, rtol)
    print(cmp_result)
    print('====================================')
```

步骤5 运行单算子网络。

```python
if __name__ == "__main__":
    tf.app.run()
```

----结束

#### TensorFlow自定义算子开发并映射到CANN算子

步骤1 适配插件代码开发。以自定义算子AddCustom为例，将该算子映射到TensorFlow自定义算子AddCustom上，需要先修改CANN AddCustom自定义算子工程目录framework/tf_plugin下插件代码，完成算子名映射：

```cpp
REGISTER_CUSTOM_OP("AddCustom")
  .FrameworkType(TENSORFLOW)
  .OriginOpType("AddCustom")
  .ParseParamsByOperatorFn(AutoMappingByOpFn);
```

步骤2 TensorFlow自定义算子的开发。本节仅给出示例说明，详细内容请参考TensorFlow官方文档。

创建TensorFlow原型注册文件custom_assign_add_custom.cc，内容如下：

```cpp
// 通过TensorFlow提供的REGISTER_OP接口完成算子原型的注册
REGISTER_OP("AddCustom")        // TensorFlow注册算子名
  .Input("x: T")                // 算子原型，输入参数x，类型为T
  .Input("y: T")                // 算子原型，输入参数y，类型为T
  .Output("z: T")               // 算子原型，输入参数z，类型为T
  .Attr("T: {half}")            // T类型支持范围
  .SetShapeFn(shape_inference::BroadcastBinaryOpShapeFn);  // 算子shape信息推导

// 实现一个CPU版本的kernel函数，因为Tensorflow的计算图在构建时会检查所有的算子是否有任意设备上的
// kernel函数（NPU Kernel无法被感知），如果没有将会报错。这里实现一个固定返回错误的CPU kernel函数：
class AddCustomOp : public OpKernel {
public:
  explicit AddCustomOp(OpKernelConstruction* context) : OpKernel(context) {}
  void Compute(OpKernelContext* context) override {
    OP_REQUIRES_OK(context, errors::Unimplemented("AddCustomOp is not supported on CPU"));
  }
};

REGISTER_KERNEL_BUILDER(Name("AddCustom").Device(DEVICE_CPU), AddCustomOp);  // 注册AddCustom算子的CPU实现内核，该算数当前仅打印日志提示CPU不支持
```

使用如下命令对上述代码进行编译，产物为libcustom_ops.so，后续的算子调用脚本中可通过load_op_library接口加载该so为python模块，从而调用自定义算子。

```bash
TF_CFLAGS=( $(python3 -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_compile_flags()))') )  # 获取TensorFlow编译选项
TF_LFLAGS=( $(python3 -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_link_flags()))') )    # 获取TensorFlow链接选项
SOURCE_FILES=custom_assign_add_custom.cc    # 包含TensorFlow算子注册和CPU内核实现的cc文件
g++ -std=c++14 -shared $SOURCE_FILES -o ${Path}/libcustom_ops.so -fPIC ${TF_CFLAGS[@]} ${TF_LFLAGS[@]} -O2
```

步骤3 测试脚本中加载上一步骤编译好的动态库，实现自定义算子的调用。

TensorFlow 1.15.0调用代码示例：

```python
import os
import tensorflow as tf
import numpy as np
from npu_bridge.npu_init import *
tf.enable_resource_variables()
atol = 0.001
rtol = 0.001
def main(unused_argv):
    custom_op_lib = tf.load_op_library('./outputs/libcustom_ops.so')  # 加载so为python模块
    shape_params = (8, 2048)
    dtype_params = np.float16
    x_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    y_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    x = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    y = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    tf_z = tf.math.add(x, y)                        # 调用TensorFlow原生算子
    ac_z = custom_op_lib.add_custom(x, y)            # 调用Ascend C AddCustom自定义算子;
    # add_custom是将REGISTER_OP(AddCustom)中的AddCustom由大驼峰命名转为下划线格式
    config = tf.ConfigProto()
    custom_op = config.graph_options.rewrite_options.custom_optimizers.add()
    custom_op.name = "NpuOptimizer"  # 配置在昇腾AI处理器上运行单算子
    config.graph_options.rewrite_options.remapping = RewriterConfig.OFF
    config.graph_options.rewrite_options.memory_optimization = RewriterConfig.OFF

    with tf.Session(config=config) as sess:
        sess.run(tf.global_variables_initializer())
        tf_golden = sess.run(tf_z, feed_dict={x: x_data, y: y_data})
    with tf.Session(config=config) as sess:
        sess.run(tf.global_variables_initializer())
        ascend_out = sess.run(ac_z, feed_dict={x: x_data, y: y_data})
    np.array(tf_golden).astype(dtype_params)
    np.array(ascend_out).astype(dtype_params)
    print('====================================')
    # 通过np.allclose比较昇腾AI处理器上运行的实际结果和使用TensorFlow原生算子运行的期望结果
    cmp_result = np.allclose(tf_golden, ascend_out, atol, rtol)
    print(cmp_result)
    print('====================================')
if __name__ == "__main__":
    tf.app.run()
```

TensorFlow 2.6.5调用代码：

```python
import os
import tensorflow as tf
import numpy as np
import npu_device
from npu_device.compat.v1.npu_init import *
npu_device.compat.enable_v1()
tf.compat.v1.enable_resource_variables()
atol = 0.001
rtol = 0.001
def main(unused_argv):
    custom_op_lib = tf.load_op_library('./outputs/libcustom_ops.so')  # 加载so为python模块
    shape_params = (8, 2048)
    dtype_params = np.float16
    x_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    y_data = np.random.uniform(-2, 2, size=shape_params).astype(dtype_params)
    x = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    y = tf.compat.v1.placeholder(dtype_params, shape=shape_params)
    tf_z = tf.math.add(x, y)                        # 调用TensorFlow原生算子
    ac_z = custom_op_lib.add_custom(x, y)            # 调用Ascend C AddCustom自定义算子;
    config = tf.compat.v1.ConfigProto()
    custom_op = config.graph_options.rewrite_options.custom_optimizers.add()
    custom_op.name = "NpuOptimizer"
    config.graph_options.rewrite_options.remapping = RewriterConfig.OFF
    config.graph_options.rewrite_options.memory_optimization = RewriterConfig.OFF

    with tf.compat.v1.Session(config=config) as sess:
        sess.run(tf.compat.v1.global_variables_initializer())
        tf_golden = sess.run(tf_z, feed_dict={x: x_data, y: y_data})
    with tf.compat.v1.Session(config=config) as sess:
        sess.run(tf.compat.v1.global_variables_initializer())
        ascend_out = sess.run(ac_z, feed_dict={x: x_data, y: y_data})
    np.array(tf_golden).astype(dtype_params)
    np.array(ascend_out).astype(dtype_params)
    print('====================================')
    cmp_result = np.allclose(tf_golden, ascend_out, atol, rtol)
    print(cmp_result)
    print('====================================')
if __name__ == "__main__":
    tf.app.run()
```

----结束

#### 可选输入算子映射关系开发

TensorFlow的原型定义中不支持可选输入，对于包含可选输入的算子，其从TensorFlow到CANN的映射关系，不满足简单的一对一映射，需要在插件适配代码中，将输入转换为可选输入，调整原型的映射关系。下文以CANN算子库中的FlashAttentionScore算子为例，介绍针对此类算子的框架适配插件如何开发。

步骤1 适配插件开发

和上文中介绍的简单的一对一映射不同，进行插件适配开发时，需要调用ParseOpToGraphFn注册回调函数，回调函数中用于调整算子原型映射关系。此时：
- 通过ParseParamsByOperatorFn注册回调函数，回调函数中将TensorFlow原生算子映射到一个IR和TensorFlow一致的中间算子（调用AutoMappingByOpFn完成属性映射）。
- 通过ParseOpToGraphFn注册回调函数，调整算子原型映射关系，将中间算子最终映射到CANN算子库中的算子，这里映射到Graph图的概念是指一个算子构成的单算子图。

需要注意：在ParseParamsByOperatorFn的回调函数中，需要将TensorFlow算子名称设置到中间算子的original_type属性中，用于后续ParseOpToGraphFn回调函数的触发。

示例代码如下（AddOptionalPlaceholderForFA函数）：

```cpp
#include <string>
#include <vector>
#include "register/register.h"
#include "graph/operator.h"
#include "graph/graph.h"
#include "graph/operator_factory.h"

namespace domi {
using namespace ge;

static Status AddOptionalPlaceholderForFA(const ge::Operator &tf_op, ge::Graph &graph) {
    // 1. 创建一个FlashAttentionScore算子npu_fa_op
    ge::AscendString op_name;
    tf_op.GetName(op_name);
    auto npu_fa_op = OperatorFactory::CreateOperator(op_name.GetString(), "FlashAttentionScore");
    // 2. 将TensorFlow算子属性映射到npu_fa_op算子上
    float scale_value = 1.0;
    (void)tf_op.GetAttr("scale_value", scale_value);
    (void)npu_fa_op.SetAttr("scale_value", scale_value);
    // ... (更多属性映射，如keep_prob, pre_tockens, next_tockens, head_num等)

    // 3. 创建输入Data
    std::vector<Operator> inputs;
    for (size_t i = 0UL; i < tf_op.GetInputsSize(); i++) {
        const std::string data_name = "Data_" + std::to_string(i);
        Operator data_op = OperatorFactory::CreateOperator(data_name.c_str(), "Data");
        (void)data_op.SetAttr("index", static_cast<int32_t>(i));
        inputs.emplace_back(data_op);
    }

    // 4. 必选输入直接设置Data到算子输入
    size_t index = 0UL;
    (void)npu_fa_op.SetInput("query", inputs[index++]);
    (void)npu_fa_op.SetInput("key", inputs[index++]);
    (void)npu_fa_op.SetInput("value", inputs[index++]);

    // 5. 可选输入需要判断type属性的个数是否为0，不为0则表示可选输入已经使能
    std::vector<DataType> real_shift_type;
    (void)tf_op.GetAttr("real_shift_type", real_shift_type);
    if (!real_shift_type.empty()) {
        (void)npu_fa_op.SetInput("real_shift", inputs[index++]);
    }
    // ... (更多可选输入: drop_mask, padding_mask, atten_mask, prefix等)

    // 6. 使用npu_fa_op算子的输出构造图的输出
    std::vector<std::pair<Operator, std::vector<size_t>>> output_indexs;
    std::vector<size_t> node_output_index;
    for (size_t i = 0UL; i < npu_fa_op.GetOutputsSize(); i++) {
        node_output_index.emplace_back(i);
    }
    (void)output_indexs.emplace_back(std::make_pair(npu_fa_op, node_output_index));
    (void)graph.SetInputs(inputs).SetOutputs(output_indexs);
    return SUCCESS;
}

static Status FlashAttentionScoreMapping(const ge::Operator& op_src, ge::Operator& op_dst) {
    // 1. 调用默认映射函数即可
    if (AutoMappingByOpFn(op_src, op_dst) != ge::GRAPH_SUCCESS) {
        return FAILED;
    }
    // 2. 需要将TensorFlow算子名称设置到op_dst的original_type属性中，用于后续ParseOpToGraphFn回调函数的触发
    op_dst.SetAttr("original_type", "FlashAttentionScore");
    return SUCCESS;
}

REGISTER_CUSTOM_OP("FlashAttentionScore")
    .FrameworkType(TENSORFLOW)
    .OriginOpType({"FlashAttentionScore"})
    .ParseParamsByOperatorFn(FlashAttentionScoreMapping)  // 注册此函数用于实现算子本身属性的映射
    .ParseOpToGraphFn(AddOptionalPlaceholderForFA);       // 注册此函数用于实现将tf中的输入转化为可选输入，改变连边关系
}  // namespace domi
```

步骤2 在TensorFlow开源框架里注册FlashAttentionScore算子的原型定义，由于TensorFlow不支持可选输入，需要将其可选输入在TensorFlow原型中表示为动态输入，并通过属性来标记动态输入的个数，这些可选输入需要放置在原型定义的最后。示例代码（FlashAttentionScore.cc）如下：

```cpp
#include <algorithm>
#include <atomic>
#include <map>
#include "tensorflow/core/framework/common_shape_fns.h"
#include "tensorflow/core/framework/op.h"
#include "tensorflow/core/framework/op_kernel.h"
using namespace tensorflow;
// ... (using declarations)

namespace {
class CustOps : public OpKernel {
public:
    explicit CustOps(OpKernelConstructionPtr context) : OpKernel(context) {}
    void Compute(OpKernelContextPtr context) override {
        std::cout << "Cust Ops not installed!!" << std::endl;
    }
    ~CustOps() override = default;};
}  // namespace

namespace tensorflow {
REGISTER_OP("FlashAttentionScore")
    .Input("query: T")
    .Input("key: T")
    .Input("value: T")
    .Input("real_shift: real_shift_type")  // 可选输入在TensorFlow原型中注册为动态输入
    .Input("drop_mask: drop_mask_type")
    .Input("padding_mask: padding_mask_type")
    .Input("atten_mask: atten_mask_type")
    .Input("prefix: prefix_type")
    .Input("actual_seq_qlen: actual_seq_qlen_type")
    .Input("actual_seq_kvlen: actual_seq_kvlen_type")
    .Input("q_start_idx: q_start_idx_type")
    .Input("kv_start_idx: kv_start_idx_type")
    .Input("d_scale_q: d_scale_q_type")
    .Input("d_scale_k: d_scale_k_type")
    .Input("d_scale_v: d_scale_v_type")
    .Input("query_rope: query_rope_type")
    .Input("key_rope: key_rope_type")
    .Output("softmax_max: float32")
    .Output("softmax_sum: float32")
    .Output("softmax_out: T")
    .Output("attention_out: T")
    .Attr("scale_value: float = 1.0")
    .Attr("keep_prob: float = 1.0")
    .Attr("pre_tockens: int = 2147483647")
    .Attr("next_tockens: int = 2147483647")
    .Attr("head_num: int")
    .Attr("input_layout: string")
    .Attr("inner_precise: int = 0")
    .Attr("sparse_mode: int = 0")
    .Attr("pse_type: int = 1")
    .Attr("seed: int = 0")
    .Attr("offset: int = 0")
    .Attr("out_dtype: int = 0")
    .Attr("T: {float16, float32, bfloat16} = DT_FLOAT")
    .Attr("real_shift_type: list({float16, float32, bfloat16}) >= 0")  // 通过属性来标记动态输入个数
    .Attr("drop_mask_type: list({uint8}) >= 0")
    // ... (更多属性)
    .SetShapeFn([](InferenceContext *c) {
        return Status::OK();
    });
REGISTER_KERNEL_BUILDER(Name("FlashAttentionScore").Device(DEVICE_CPU), CustOps)
}
```

编译命令：

```bash
TF_CFLAGS=( $(python3 -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_compile_flags()))') )
TF_LFLAGS=( $(python3 -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_link_flags()))') )
SOURCE_FILES=FlashAttentionScore.cc  # 包含TensorFlow算子注册和CPU内核实现的cc文件
g++ -std=c++14 -shared $SOURCE_FILES -o ${Path}/libflashattention.so -fPIC ${TF_CFLAGS[@]} ${TF_LFLAGS[@]} -O2
```

步骤3 封装一个TensorFlow的算子调用接口，在此接口中处理可选输入。在该脚本中需要加载上一步骤编译好的动态库。

```python
from tensorflow.python.framework import ops
import tensorflow as tf
tfOpLib = tf.load_op_library("../build/tf_ops/libflashattention.so")

# 假如外部未使能该可选输入，则给底层传入空列表
def create_optional_input_list(input):
    input_list = []
    if not input is None:
        input_list.append(input)
    return input_list

# flash_attention_score 封装函数
def npu_flash_attention(query, key, value, head_num, input_layout, real_shift=None, drop_mask=None,
                        padding_mask=None, atten_mask=None, prefix=None, actual_seq_qlen=None,
                        actual_seq_kvlen=None, q_start_idx=None, kv_start_idx=None,
                        d_scale_q=None, d_scale_k=None, d_scale_v=None, query_rope=None,
                        key_rope=None, scale_value=1.0, keep_prob=1.0,
                        pre_tockens=2147483647, next_tockens=2147483647, inner_precise=0,
                        sparse_mode=0, pse_type=1, seed=0, offset=0, out_dtype=0):
    output = tfOpLib.flash_attention_score(
        query=query, key=key, value=value,
        real_shift=create_optional_input_list(real_shift),
        drop_mask=create_optional_input_list(drop_mask),
        padding_mask=create_optional_input_list(padding_mask),
        atten_mask=create_optional_input_list(atten_mask),
        prefix=create_optional_input_list(prefix),
        actual_seq_qlen=create_optional_input_list(actual_seq_qlen),
        actual_seq_kvlen=create_optional_input_list(actual_seq_kvlen),
        q_start_idx=create_optional_input_list(q_start_idx),
        kv_start_idx=create_optional_input_list(kv_start_idx),
        d_scale_q=create_optional_input_list(d_scale_q),
        d_scale_k=create_optional_input_list(d_scale_k),
        d_scale_v=create_optional_input_list(d_scale_v),
        query_rope=create_optional_input_list(query_rope),
        key_rope=create_optional_input_list(key_rope),
        scale_value=scale_value, keep_prob=keep_prob,
        pre_tockens=pre_tockens, next_tockens=next_tockens,
        head_num=head_num, input_layout=input_layout, inner_precise=inner_precise,
        sparse_mode=sparse_mode, pse_type=pse_type, seed=seed, offset=offset, out_dtype=out_dtype)
    return output
```

步骤4 测试脚本中实现自定义算子的调用。假设上一步骤中代码文件保存为ops.py，从ops中导入npu_flash_attention函数并使用。TensorFlow 2.6.5调用代码如下：

```python
import sys
from ops import npu_flash_attention
import tensorflow as tf
import numpy as np

tf.compat.v1.disable_eager_execution()
import npu_device
from npu_device.compat.v1.npu_init import *
npu_device.compat.enable_v1()

def sess_config():
    config = tf.compat.v1.ConfigProto()
    custom_op = config.graph_options.rewrite_options.custom_optimizers.add()
    custom_op.name = "NpuOptimizer"
    config.graph_options.rewrite_options.remapping = RewriterConfig.OFF
    config.graph_options.rewrite_options.memory_optimization = RewriterConfig.OFF
    return config

shape = [1, 32, 32]
query_np = np.random.randn(*shape).astype(np.float16)
key_np = np.random.randn(*shape).astype(np.float16)
value_np = np.random.randn(*shape).astype(np.float16)

query = tf.Variable(query_np, tf.float16)
key = tf.Variable(key_np, tf.float16)
value = tf.Variable(value_np, tf.float16)

mask = tf.zeros(shape=(shape[0], 1, shape[1], shape[1]), dtype=tf.uint8)

head_num = 1
input_layout = "BSH"
flash_result_t = npu_flash_attention(query, key, value, head_num, input_layout, atten_mask=mask)

with tf.compat.v1.Session(config=sess_config()) as sess:
    sess.run(tf.compat.v1.global_variables_initializer())
    flash_result = sess.run(flash_result_t)
    print(flash_result)
```

----结束

#### 动态输入算子映射关系开发

对于存在动态输入/输出的算子，需要在插件的回调函数ParseParamByOpFunc中使用AutoMappingByOpFnDynamic实现TensorFlow算子和CANN算子的匹配。通过DynamicInputOutputInfo结构类描述动态输入/输出的信息，将动态输入/输出的名称和描述其个数的属性名绑定，再传入AutoMappingByOpFnDynamic实现自动匹配。

以ParseSingleExample算子为例，插件适配代码如下：

```cpp
#include "register/register.h"
namespace domi {
Status ParseSingleExampleMapping(const ge::Operator& op_src, ge::Operator& op) {
    std::vector<DynamicInputOutputInfo> value;
    // 描述动态输入
    const std::string dynamic_input_name_dense_defaults = "dense_defaults";
    const std::string dynamic_input_attr_name_dense_defaults = "Tdense";
    DynamicInputOutputInfo input(kInput,
        dynamic_input_name_dense_defaults.c_str(), dynamic_input_name_dense_defaults.size(),
        dynamic_input_attr_name_dense_defaults.c_str(), dynamic_input_attr_name_dense_defaults.size());
    value.push_back(input);

    // 描述动态输出 sparse_indices
    const std::string dynamic_output_name_sparse_indices = "sparse_indices";
    const std::string dynamic_output_attr_name_sparse_indices = "num_sparse";
    DynamicInputOutputInfo output(kOutput, ...);
    value.push_back(output);

    // 描述动态输出 sparse_values
    // ... (类似模式)

    // 描述动态输出 sparse_shapes
    // ... (类似模式)

    // 描述动态输出 dense_values
    // ... (类似模式)

    AutoMappingByOpFnDynamic(op_src, op, value);
    return SUCCESS;
}

// register ParseSingleExample op to GE
REGISTER_CUSTOM_OP("ParseSingleExample")
    .FrameworkType(TENSORFLOW)
    .OriginOpType("ParseSingleExample")
    .ParseParamsByOperatorFn(ParseSingleExampleMapping)
}
```

> 说明：暂不支持同时有可选输入和动态输入的算子映射。

## 13.5 show_kernel_debug_data工具

静态图场景下，整图算子全部下沉到NPU侧执行，kernel侧单算子调试信息（通过printf接口）需要在模型执行结束后才能获取。本工具提供了离线解析能力，帮助用户获取并解析调试信息（将bin文件解析成可读格式）。

> 说明：show_kernel_debug_data支持多用户并发调用，但用户需要指定不同的落盘路径，否则可能出现落盘内容被覆盖等问题。

产品支持情况：

| 产品 | 是否支持 |
|------|---------|
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √ |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √ |
| Atlas 200I/500 A2 推理产品 | √ |
| Atlas 推理系列产品 | √ |
| Atlas 训练系列产品 | x |

#### 工具安装

步骤1 安装工具。

工具跟随CANN软件包发布（参考2 环境准备完成CANN安装），其路径默认为"${INSTALL_DIR}/tools/show_kernel_debug_data"，其中${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：/usr/local/Ascend/ascend-toolkit/latest。

步骤2 设置环境变量。

- root用户安装Ascend-cann-toolkit包时：
```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/ascend-toolkit/latest/toolkit/bin/setenv.bash
```

- 非root用户安装Ascend-cann-toolkit包时：
```bash
source ${HOME}/Ascend/ascend-toolkit/set_env.sh
source ${HOME}/Ascend/ascend-toolkit/latest/toolkit/bin/setenv.bash
```

步骤3 检查工具是否安装成功。

执行如下命令，若能正常显示--help或-h信息，则表示工具环境正常，功能可正常使用。
`show_kernel_debug_data -h`

----结束

#### 使用方法

- 命令行方式
`show_kernel_debug_data <bin_file_path> [<output_path>]`

| 参数 | 可选/必选 | 说明 |
|------|----------|------|
| <bin_file_path> | 必选 | kernel侧调试信息落盘的bin文件路径，例如"/input/dump_workspace.bin"。 |
| <output_path> | 可选 | 解析结果的保存路径，例如"/output_dir"。默认是当前命令行执行目录下。 |

- API方式

表13-14 show_kernel_debug_data接口说明表

| 项目 | 说明 |
|------|------|
| 函数原型 | `def show_kernel_debug_data(bin_file_path: str, output_path: str = './') -> None` |
| 函数功能 | 获取kernel侧调试信息并解析成可读文件。 |
| 参数(IN) bin_file_path | kernel侧调试信息落盘的bin文件路径，字符串类型。 |
| 参数(IN) output_path | 解析结果的保存路径，字符串类型，默认是当前接口调用脚本所在目录下。 |
| 参数(OUT) | NA |
| 返回值 | NA |
| 使用约束 | 无 |
| 调用示例 | `from show_kernel_debug_data import show_kernel_debug_data`<br>`show_kernel_debug_data('./input/dump_workspace.bin')` |

#### 产物说明

工具解析结果文件目录结构如下：

```
${output_path}
├── PARSER_${timestamp}      // ${timestamp}表示时间戳。
│   └── parser.log           // 工具解析的日志，包含kernel侧日常流程和printf打印信息。
```

## 13.6 msobjdump工具

本工具主要针对Kernel直调工程（NPU模式）、标准自定义算子工程、简易自定义算子工程编译生成的算子ELF文件（Executable and Linkable Format）提供解析和解压功能，并将结果信息以可读形式呈现，方便开发者直观获得kernel文件信息。

> 说明：
> - ELF文件是一种用于二进制文件、可执行文件、目标代码、共享库和核心转储的文件格式，包括常见的\*.a、\*.so文件等。ELF文件常见构成如下：
>   - ELF头部：描述了整个文件的组织结构，包括文件类型、机器类型、版本号等信息。
>   - 程序头部表：描述了文件中各种段（segments）信息，包括程序如何加载到内存中执行的信息。
>   - 节区头部表：描述了文件中各个节（sections）信息，包括程序的代码、数据、符号表等。
> - 工具使用过程中，若出现如下场景，请根据日志提示信息，分析排查问题。
>   - ELF文件未找到
>   - ELF文件权限错误
>   - ELF文件存在但不支持解析或解压

产品支持情况：

| 产品 | 是否支持 |
|------|---------|
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √ |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √ |
| Atlas 200I/500 A2 推理产品 | √ |
| Atlas 推理系列产品 | √ |
| Atlas 训练系列产品 | √ |

#### 工具安装

步骤1 安装msobjdump工具。

工具跟随CANN软件包发布（参考2 环境准备完成CANN安装），其路径默认为"${INSTALL_DIR}/tools/msobjdump"，其中${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。若安装的Ascend-cann-toolkit软件包，以root安装举例，则安装后文件存储路径为：/usr/local/Ascend/ascend-toolkit/latest。

步骤2 设置环境变量。

- root用户安装Ascend-cann-toolkit包时：
```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/ascend-toolkit/latest/toolkit/bin/setenv.bash
```

- 非root用户安装Ascend-cann-toolkit包时：
```bash
source ${HOME}/Ascend/ascend-toolkit/set_env.sh
source ${HOME}/Ascend/ascend-toolkit/latest/toolkit/bin/setenv.bash
```

步骤3 检查工具是否安装成功。

执行如下命令，若能正常显示--help或-h信息，则表示工具环境正常，功能可正常使用。
`msobjdump -h`

----结束

#### 命令格式

- 解析ELF文件的命令
`msobjdump --dump-elf <elf_file> [--verbose]`

表13-15 参数说明

| 参数(区分大小写) | 可选/必选 | 说明 |
|----------------|----------|------|
| --dump-elf <elf_file>, -d | 必选 | 解析ELF文件中包含的device信息，如文件名、文件类型、文件长度、符号表等，并终端打屏显示。<elf_file>表示待解析ELF文件路径，如/home/op_api/lib_api.so。支持两种打印模式：简单打印（默认仅打印部分device信息）；全量打印（与--verbose配套使用，开启全量device信息打屏显示）。不同工程打印字段信息不同，具体参见表13-18和表13-19。 |
| --verbose, -V | 可选 | 必须与--dump-elf配套使用，用于开启ELF文件中全量打印device信息功能。 |

- 解压ELF文件的命令
`msobjdump --extract-elf <elf_file> [--out-dir <out_path>]`

表13-16 参数说明

| 参数(区分大小写) | 可选/必选 | 说明 |
|----------------|----------|------|
| --extract-elf <elf_file>, -e | 必选 | 解压ELF文件中包含的device信息，并按原始文件夹规则落盘到输出路径下。<elf_file>表示待解压ELF文件路径，如/home/op_api/lib_api.so。默认路径：解压结果文件默认落盘到当前执行路径下。自定义路径：可与--out-dir配套使用，设置落盘路径。 |
| --out-dir <out_path>, -o | 可选 | 必须与--extract-elf配套使用，用于设置解压文件的落盘路径。<out_path>为落盘文件目录，如/home/extract/。说明：msobjdump支持多用户并发调用，但需要指定不同的--out-dir，否则可能出现落盘内容被覆盖的问题。 |

- 获取ELF文件列表的命令
`msobjdump --list-elf <elf_file>`

表13-17 参数说明

| 参数(区分大小写) | 可选/必选 | 说明 |
|----------------|----------|------|
| --list-elf <elf_file>, -l | 可选 | 获取ELF文件中包含的device信息文件列表，并打印显示。<elf_file>表示待打印的ELF文件路径，如/home/op_api/lib_api.so。 |

表13-18 ELF解析字段说明（Kernel直调工程）

| 字段名 | 含义 | 是否必选 | 打印说明 |
|--------|------|----------|----------|
| VERSION | 表示版本号。 | 是 | 不设置--verbose，默认打印。 |
| TYPE COUNT | 表示ELF文件中包含的kernel文件个数。 | | |
| ELF FILE ${id} | 表示ELF文件中包含的kernel文件名，${id}表示kernel文件序号。kernel文件名的命名规则如下：按${sec_prefix}_${file_index}_${kernel_type}.o拼接，其中${sec_prefix}为section段名（工具根据".ascend.kernel"关键字搜索获取），${file_index}表示文件编号，${kernel_type}表示kernel类型。 | | |
| KERNEL LEN | 表示kernel文件的长度。 | | |
| KERNEL TYPE | 表示kernel类型，映射关系为{0:'mix', 1:'aiv', 2:'aic'}。 | 否 | |
| ASCEND META | 表示算子执行时核间同步、Cube/Vector核占比（task_ration）等信息。若没有获取到该信息，默认显示None。 | | |
| elf heard infos | 包括ELF Header、Section Headers、Key to Flags、Program Headers、Symbol表等信息。 | 否 | 设置--verbose，开启全量打印。 |

表13-19 ELF解析字段说明（标准/简易自定义算子工程）

| 字段名 | 含义 | 是否必选 | 打印说明 |
|--------|------|----------|----------|
| .ascend.meta.${id} | 表示算子kernel函数名称，其中${id}表示meta信息的索引值。 | 是 | 不设置--verbose，默认打印。 |
| VERSION | 表示版本号。 | 是 | |
| DEBUG | 调试相关信息，包含如下两部分内容：debugBufSize（调试信息需要的内存空间）；debugOptions（调试开关状态，取值：0:关闭，1:通过DumpTensor/print打印进行调试，2:通过assert断言进行调试，4:通过时间戳打点功能进行调试，8:通过内存越界检测进行调试）。 | 否 | |
| DYNAMIC_PARAM | 算子kernel函数是否启用动态参数。取值：0:关闭动态参数模式，1:开启动态参数模式。 | | |
| OPTIONAL_PARAM | 可选参数信息，包含两部分：optionalInputMode（可选输入在算子kernel函数中是否需要占位，0:不占位，1:占位）；optionalOutputMode（可选输出在算子kernel函数中是否需要占位，0:不占位，1:占位）。 | | |
| KERNEL_TYPE | 表示kernel函数运行时core类型，取值参见表13-20。 | | |
| CROSS_CORE_SYNC | 表示硬同步syscall类型。USE_SYNC:使用硬同步。NO_USE_SYNC:不使用硬同步。 | | |
| MIX_TASK_RATION | 表示kernel函数运行时的Cube核/Vector核占比分配类型。 | | |
| DETERMINISTIC_INFO | 表示算子是否为确定性计算。0:不确定计算，1:确定性计算。 | | |
| BLOCK_DIM | 表示算子执行核数，该字段当前暂不支持，只打印默认值0xFFFFFFFF。 | | |
| FUNCTION_ENTRY | 算子TilingKey的值。 | | |
| elf heard infos | 包括ELF Header、Section Headers、Key to Flags、Program Headers、Symbol表等信息。 | 否 | 设置--verbose，开启全量打印。 |

表13-20 kernel type信息

| KERNEL_TYPE | 说明 |
|-------------|------|
| AICORE | 该参数为预留参数，当前版本暂不支持。算子执行时仅会启动AI Core，比如用户在host侧设置blockdim为5，则会启动5个AI Core。 |
| AIC | 算子执行时仅启动AI Core上的Cube核：比如用户在host侧设置blockdim为10，则会启动10个Cube核。 |
| AIV | 算子执行时仅启动AI Core上的Vector核：比如用户在host侧设置blockdim为10，则会启动10个Vector核。 |
| MIX_AIC_MAIN | AIC、AIV混合场景下，设置核函数的类型为MIX，算子执行时会同时启动AI Core上的Cube核和Vector核，比如用户在host侧设置blockdim为10，且设置task_ration为1:2，则会启动10个Cube核和20个Vector核。 |
| MIX_AIV_MAIN | AIC、AIV混合场景下，使用了多核控制相关指令时，设置核函数的类型为MIX，算子执行时会同时启动AI Core上的Cube核和Vector核，比如用户在host侧设置blockdim为10，且设置task_ration为1:2，则会启动10个Vector核和20个Cube核。 |
| AIC_ROLLBACK | 算子执行时会同时启动AI Core和Vector Core，此时AI Core会当成Cube Core使用。 |
| AIV_ROLLBACK | 算子执行时会同时启动AI Core和Vector Core，此时AI Core会当成Vector Core使用。 |

#### 使用样例（Kernel直调算子工程）

以MatMulInvocationNeo算子为例（NPU模式），完整的工程可参考Matmul多核Kernel直调样例。假设${cmake_install_dir}为算子Cmake编译产物根目录，目录结构如下（仅为示例，具体以实际算子工程为准），类似CMake编译配置文件编写。

```
out
├── lib
│   └── libascendc_kernels_npu.so
├── include
│   └── ascendc_kernels_npu
│       ├── aclrtlaunch_matmul_custom.h
│       └── aclrtlaunch_triple_chevrons_func.h
......
```

工具对编译生成的库文件（如\*.so、\*.a等）进行解析和解压，功能实现命令样例如下：

- 解析包含device信息的库文件
支持两种打印方式，请按需选取，解析字段含义参见表13-18。
  - 简单打印
`msobjdump --dump-elf ${cmake_install_dir}/out/libascendc_kernels_npu.so`
执行上述命令，终端打印基础device信息，示例如下：
```
============================
[VERSION]: 1
[TYPE COUNT]: 1
============================
[ELF FILE 0]: ascendxxxb1_ascendc_kernels_npu_0_mix.o
```

  - 全量打印
`msobjdump --dump-elf ${cmake_install_dir}/out/libascendc_kernels_npu.so --verbose`
执行上述命令，终端打印所有device信息，示例如下：
```
============================
[VERSION]: 1
[TYPE COUNT]: 1
============================
[ELF FILE 0]: ascendxxxb1_ascendc_kernels_npu_0_mix.o
[KERNEL TYPE]: mix
[KERNEL LEN]: 511560
[ASCEND META]: None
====== [elf heard infos] ======
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  Type:                              EXEC (Executable file)
  Machine:                           <unknown>: 0x1029
  Entry point address:               0x0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          510280 (bytes into file)
  Flags:                             0x940000
  Number of program headers:         2
  Number of section headers:         20
  ...
```

- 解压包含device信息的库文件并落盘
`msobjdump --extract-elf ${cmake_install_dir}/out/libascendc_kernels_npu.so`
执行上述命令，默认在当前执行路径下落盘ascendxxxb1_ascendc_kernels_npu_0_mix.o文件。

- 获取包含device信息的库文件列表
`msobjdump --list-elf ${cmake_install_dir}/out/libascendc_kernels_npu.so`
执行上述命令，终端会打印所有文件，屏显信息形如：
`ELF file   0: ascendxxxb1_ascendc_kernels_npu_0_mix.o`

#### 使用样例（标准/简易自定义算子工程）

以下面的算子工程为例（仅为示例，具体以实际算子工程为准），假设${cmake_install_dir}为算子Cmake编译产物根目录，目录结构如下，类似算子编译。

```
op_api
├── include
│   ├── aclnn_acos_custom.h
│   ├── aclnn_matmul_leakyrelu_custom.h
│   └── ........
├── lib
│   └── libcust_opapi.so
```

工具对编译生成的库文件（如\*.so、\*.a等）进行解析和解压，功能实现命令样例如下：

- 解析包含device信息的库文件
支持两种打印方式，请按需选取，解析字段含义参见表13-19。
  - 简单打印
`msobjdump --dump-elf ${cmake_install_dir}/op_api/lib/libcust_opapi.so`
执行上述命令，终端打印基础device信息，示例如下：
```
.ascend.meta META INFO
VERSION: 1
DEBUG: debugBufSize=0, debugOptions=0
DYNAMIC_PARAM: dynamicParamMode=0
OPTIONAL_PARAM: optionalInputMode=1, optionalOutputMode=1
.ascend.meta. [0]: AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26_1
KERNEL_TYPE: AIV
DETERMINISTIC_INFO: 1
BLOCK_DIM: 0xFFFFFFFF
FUNCTION_ENTRY: 1
.ascend.meta. [0]: AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26_2_mix_aiv
KERNEL_TYPE: MIX_AIV_MAIN
MIX_TASK_RATION: [0:1]
DETERMINISTIC_INFO: 1
BLOCK_DIM: 0xFFFFFFFF
FUNCTION_ENTRY: 2
......
```

  - 全量打印
`msobjdump --dump-elf ${cmake_install_dir}/op_api/lib/libcust_opapi.so --verbose`
执行上述命令，终端打印基础device信息，包含elf heard infos部分（ELF Header、Section Headers等详细信息）。

- 解压包含device信息的库文件并落盘
`msobjdump --extract-elf ${cmake_install_dir}/op_api/lib/libcust_opapi.so`
执行上述命令，默认在当前执行路径下保存解压文件，产物目录如下：

```
├── config                          // 算子原型配置文件目录
│   ├── ${soc_version}
│   │   ├── acos_custom.json
│   │   ├── matmul_leakyrelu_custom.json
│   │   └── ......
├── ${soc_version}                   // 昇腾AI处理器名
│   ├── acos_custom                  // 基础单算子编译文件*.o和对应的*.json文件
│   │   ├── AcosCustom_da824ede53d7e754f85c14b9446ec2fc.json
│   │   ├── AcosCustom_da824ede53d7e754f85c14b9446ec2fc.o
│   │   ├── AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26.json
│   │   ├── AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26.o
│   ├── matmul_leakyrelu_custom
│   │   ├── MatmulLeakyreluCustom_e052bee3255764ac919095f3bdf83389.json
│   │   ├── MatmulLeakyreluCustom_e052bee3255764ac919095f3bdf83389.o
│   ├── axpy_custom
│   │   └── ....
```

以acos_custom算子编译产物解压为例：
  - 查看算子原型（acos_custom.json）：包含binList数组，每个元素描述一种编译配置（implMode、int64Mode、simplifiedKeyMode、staticKey、inputs、outputs、attrs、opMode、optionalInputMode、deterministic等字段），以及binInfo中的jsonFilePath指向具体的编译产物json文件。
  - 解析\*.o文件获取.ascend.meta段信息：
`msobjdump --dump-elf ./AcosCustom_da824ede53d7e754f85c14b9446ec2fc.o`
  - 查看\${parm_info}.json，直观获取device文件中算子信息（binFileName、coreType、intercoreSync、kernelName、workspace、kernelList等字段）。

- 获取包含device信息的库文件列表
`msobjdump --list-elf ${cmake_install_dir}/op_api/lib/libcust_opapi.so`
执行上述命令，终端会打印所有文件，屏显信息形如：
```
ELF file   0: ascendxxx_acos_custom_AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26.json
ELF file   1: ascendxxx_acos_custom_AcosCustom_dad9c8ca8fcbfd789010c8b1c0da8e26.o
......
ELF file   2: ascendxxx_acos_custom_AcosCustom_da824ede53d7e754f85c14b9446ec2fc.json
ELF file   3: ascendxxx_acos_custom_AcosCustom_da824ede53d7e754f85c14b9446ec2fc.o
```

## 13.7 简易自定义算子工程

本章节介绍的简易自定义算子工程，是上文中介绍的自定义算子工程的简化版，对算子的编译、打包、部署过程进行简化，便于开发者将该工程集成到自己的算子工程。

基于简易自定义算子工程进行算子开发的完整样例请参考简易自定义算子工程。

> 说明：
> - 使用该工程，支持在如下平台进行自定义算子开发：
>   - Atlas A2训练系列产品
>   - Atlas 推理系列产品
> - 使用本工程开发的算子，只支持通过单算子API执行（aclnn）方式进行调用。
> - 本工程暂不支持算子的交叉编译功能。

[图: 简易自定义算子工程的算子开发流程图，包含环境准备(CANN软件安装)、算子分析、创建算子工程、Kernel侧算子实现、Host侧算子实现、算子编译、单算子API(aclnn)调用]

#### 创建算子工程

和13.2.2 创建算子工程类似，简易自定义算子工程通过msOpGen生成，基于算子原型定义输出算子工程，包括算子host侧代码实现文件、算子kernel侧实现文件以及工程编译配置文件等。主要差异点在于：创建简易算子工程需要通过-f参数配置framework框架为aclnn。

> 说明：使用msOpGen工具创建算子工程之前，需要参考2 环境准备章节安装驱动固件和CANN软件包，完成开发环境和运行环境的准备。

步骤1 编写算子的原型定义json文件，用于生成算子开发工程。

例如，AddCustom算子的json文件命名为add_custom.json，文件内容如下：

```json
[
  {
    "op": "AddCustom",
    "input_desc": [
      {
        "name": "x",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["fp16", "float", "int32"]
      },
      {
        "name": "y",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["fp16", "float", "int32"]
      }
    ],
    "output_desc": [
      {
        "name": "z",
        "param_type": "required",
        "format": ["ND", "ND", "ND"],
        "type": ["fp16", "float", "int32"]
      }
    ]
  }
]
```

步骤2 使用msOpGen工具生成算子的开发工程。以生成AddCustom的算子工程为例，下文仅针对关键参数进行解释，详细参数说明请参见msOpGen工具。

`${INSTALL_DIR}/python/site-packages/bin/msopgen gen -i $HOME/sample/add_custom.json -c ai_core-<soc_version> -lan cpp -out $HOME/sample/AddCustom -f aclnn`

- ${INSTALL_DIR}为CANN软件安装后文件存储路径，请根据实际环境进行替换。
- -i：指定算子原型定义文件add_custom.json所在路径，请根据实际情况修改。
- -c：ai_core-<soc_version>代表算子在AI Core上执行，<soc_version>为昇腾AI处理器的型号。
- -lan：参数cpp代表算子基于Ascend C编程框架，使用C/C++编程语言开发。
- -out：生成文件所在路径，可配置为绝对路径或者相对路径。
- -f：表示框架类型，aclnn表示生成简易工程。

步骤3 命令执行完后，会在-out指定目录或者默认路径下生成算子工程目录，工程中包含算子实现的模板文件、编译脚本等，以AddCustom算子为例，目录结构如下所示：

```
AddCustom
├── build.sh              // 编译入口脚本
├── cmake
│   ├── config.cmake      // 编译配置项
│   ├── func.cmake
│   ├── intf.cmake
│   └── util              // 算子工程编译所需脚本及公共编译文件存放目录
├── CMakeLists.txt        // 算子工程的CMakeLists.txt
├── op_host               // host侧实现文件
│   ├── add_custom_tiling.h   // 算子tiling定义文件
│   ├── add_custom.cpp        // 算子原型注册、tiling实现等内容文件
│   └── CMakeLists.txt
├── op_kernel             // kernel侧实现文件
│   ├── CMakeLists.txt
│   └── add_custom.cpp   // 算子代码实现文件
```

> 说明：上述目录结构中的粗体文件为后续算子开发过程中需要修改的文件，其他文件无需修改。

----结束

#### 算子实现

参考13.2.4 Kernel侧算子实现、13.2.5 Host侧Tiling实现、13.2.3 算子原型定义完成算子实现。

#### 算子编译

算子kernel侧和host侧实现开发完成后，需要对算子进行编译，生成算子静态库；自动生成aclnn调用实现代码和头文件，链接算子静态库生成aclnn动态库，以支持后续的单算子API执行方式（aclnn）的算子调用。编译过程如下：

- 根据host侧实现文件自动生成aclnn接口aclnn_\*.h和aclnn实现文件aclnn_.cpp。
- 编译Tiling实现和算子原型定义生成Tiling动态库liboptiling.so（libcust_opmaster_rt2.0）。
- 编译kernel侧实现文件，并加载Tiling动态库，生成kernel静态库libkernels.a。
- 编译aclnn实现文件，并链接kernel静态库libkernels.a生成单算子API调用的动态库libcust_opapi.so。

[图: 编译过程示意图，op_host代码生成aclnn接口和实现(aclnn_*.h和aclnn_.cpp)，编译生成libcust_opapi.so；op_host编译生成Tiling动态库liboptiling.so；op_kernel编译生成kernel静态库libkernels.a，链接到libcust_opapi.so]

步骤1 完成工程编译相关配置。

- 修改cmake目录下config.cmake中的配置项，config.cmake文件内容如下：

```cmake
set(CMAKE_CXX_FLAGS_DEBUG "")
set(CMAKE_CXX_FLAGS_RELEASE "")

if (NOT DEFINED CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "")
endif()
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}/build_out" CACHE PATH "" FORCE)
endif()
if (NOT DEFINED ASCEND_CANN_PACKAGE_PATH)
    set(ASCEND_CANN_PACKAGE_PATH /usr/local/Ascend/ascend-toolkit/latest CACHE PATH "")
    # 请替换为CANN软件包安装后的实际路径
endif()
if (NOT DEFINED ASCEND_PYTHON_EXECUTABLE)
    set(ASCEND_PYTHON_EXECUTABLE python3 CACHE STRING "")
endif()
if (NOT DEFINED ASCEND_COMPUTE_UNIT)
    set(ASCEND_COMPUTE_UNIT ascendxxx CACHE STRING "")
endif()
# ... 其他配置
```

表13-21 需要开发者配置的常用参数列表

| 参数名称 | 参数描述 | 默认值 |
|--------|--------|-------|
| ASCEND_CANN_PACKAGE_PATH | CANN软件包安装路径，请根据实际情况进行修改。 | "/usr/local/Ascend/ascend-toolkit/latest" |
| CMAKE_BUILD_TYPE | 编译模式选项，可配置为："Release"（Release版本，不包含调试信息）；"Debug"（Debug版本，包含调试信息，便于开发者开发和调试）。 | "Release" |
| CMAKE_INSTALL_PREFIX | 编译产物存放的目录，不指定则为默认值。 | ${CMAKE_SOURCE_DIR}/build_out：算子工程目录下的build_out目录 |

- 配置编译相关环境变量（可选）

表13-22 环境变量说明

| 环境变量 | 配置说明 |
|--------|--------|
| CMAKE_CXX_COMPILER_LAUNCHER | 用于配置C++语言编译器（如g++）、毕昇编译器的启动器程序为ccache，配置后即可开启cache缓存编译，加速重复编译并提高构建效率。用法：`set(CMAKE_CXX_COMPILER_LAUNCHER <launcher_program>)`，其中<launcher_program>是ccache的安装路径，示例：`set(CMAKE_CXX_COMPILER_LAUNCHER /usr/bin/ccache)` |

步骤2 （可选）如果需要编译多个算子，在op_kernel目录下的CMakeLists.txt中增加要编译的算子。

```cmake
# set custom compile options
if ("${CMAKE_BUILD_TYPE}x" STREQUAL "Debugx")
    add_ops_compile_options(ALL OPTIONS -g -O0)
endif()
# 多算子编译通过add_kernel_compile命令增加算子源码文件即可
add_kernel_compile(AddCustom ${CMAKE_CURRENT_SOURCE_DIR}/add_custom.cpp)
add_kernel_compile(SubCustom ${CMAKE_CURRENT_SOURCE_DIR}/sub_custom.cpp)
```

步骤3 （可选）在算子工程中，如果开发者想对算子kernel侧代码增加一些自定义的编译选项，可以参考支持自定义编译选项进行编译选项的定制。

步骤4 在算子工程目录下执行如下命令，进行算子工程编译。
`./build.sh`

编译成功后，会在CMAKE_INSTALL_PREFIX/op_api目录存放生成的aclnn头文件和lib库，每一个算子都会对应一个单独的头文件。具体目录结构如下：

```
op_api
├── include
│   ├── aclnn_optype1.h
│   ├── aclnn_optype2.h
│   └── aclnn_optypexxx.h
├── lib
│   └── libcust_opapi.so
```

对于lib目录下生成的库文件可通过msobjdump工具进一步解析得到kernel信息，具体操作参见13.6 msobjdump工具。

----结束

#### 算子调用

完成单算子API调用。

## 13.8 FAQ

### 13.8.1 核函数运行验证时算子存在精度问题

现象描述：在进行算子NPU域的运行验证时，实际数据和真值数据不一致，算子存在精度问题。

问题根因：算子出现精度问题，一般是由于算子的实现逻辑有误。

定位步骤：

Ascend C提供孪生调试的功能，通过CPU域的功能验证、gdb单步调试、printf数值打印来定位算子的实现逻辑问题。本样例仅展示了可能会出现的场景，便于演示定位步骤。实际使用过程中，请根据代码情况进行调试。

步骤1 进行CPU域的功能验证，观察是否有日志报错。

参考13.1 基于样例工程完成Kernel直调章节，编写CPU侧的运行验证代码，并进行运行验证，发现CPU域的精度比对也存在不一致的问题。

观察打屏日志中是否有报错信息，可搜索关键词"failed"。比如，下图的报错示例指示，错误出现在代码中调用LeakyRelu接口的地方。

```
leakyrelu_custom_cpu: .../kernel_operator_vec_binary_scalar_intf.h:447: void AscendC::LeakyRelu(...)
Assertion 'false && "check vlrelu instr failed"' failed
```

通过上述报错日志，一般只能定位到报错的代码行，无法明确具体错误，接下来需要通过gdb调试的方式或者printf打印的方式进一步精确定位。

步骤2 gdb调试。下面的样例展示了拉起leakyrelu算子CPU侧运行程序的样例，该样例程序会直接抛出异常，直接gdb运行，查看调用栈信息分析定位即可。

1. 使用gdb拉起待调试程序，进入gdb界面进行debug。
`gdb leakyrelu_custom_cpu`
2. 单独调试一个子进程。
`(gdb) set follow-fork-mode child`
3. 运行程序。
`(gdb) r`
4. 通过bt查看程序调用栈。
`(gdb) bt`
5. 查看具体层的堆栈信息，打印具体变量的值。本示例中，打印了tileLength为1024，该程序中表示需要处理1024个half类型的数，大小为1024\*sizeof(half)=2048字节；输入Tensor xLocal的值，其中dataLen表示LocalTensor的size大小为1024字节，只能计算512个half类型的数据。可以看出两者的长度不匹配，由此可以定位问题。

步骤3 printf打印。在合适的位置增加变量打印。样例代码如下：

```cpp
printf("xLocal size: %d\n", xLocal.GetSize());
printf("tileLength: %d\n", tileLength);
```

可以看到如下打屏日志输出，打印了tileLength为1024，输入Tensor xLocal的size大小为512，可以看出两者的长度不匹配，由此可以定位问题。

```
xLocal size: 512
tileLength: 1024
```

----结束

### 13.8.2 运行验证时AllocTensor/FreeTensor失败

现象描述：通过NPU进行核函数的运行验证时，出现挂死现象；通过CPU进行核函数的运行验证时，出现AllocTensor/FreeTensor失败的报错，日志报错和调用栈打印如下：

```
[ERROR][Core_0][/usr/local/Ascend/latest/x86_64-linux/tikcpp/tikcfw/interface/kernel_tpipe.h:730]
[AllocEventID][321678] current size is 4, max buffer number in same queue position is 4
[ERROR][CORE_0][pid 321674] error happened! =========
SIGABRT Signal (Abort Signal from abort) catched, backtrace info:
...
```

问题根因：根据日志信息"current size is 4, max buffer number in same queue position is 4"可以明确该问题是因为同一个TPosition上QUE Buffer的数量超出限制导致。

同一个TPosition上的所有Queue，连续调用AllocTensor接口申请的Tensor数量，根据AI处理器型号的不同，有数量约束。申请Buffer时，需要满足该约束。

| 产品 | 限制 |
|------|------|
| Atlas 训练系列产品 | AI Core不超过4个。 |
| Atlas 推理系列产品 | AI Core不超过8个。 |
| Atlas 推理系列产品 | Vector Core不超过8个。 |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | 不超过8个。 |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | 不超过8个。 |
| Atlas 200I/500 A2 推理产品 | 不超过8个。 |

不满足该约束，在后续使用AllocTensor/FreeTensor可能会出现分配资源失败。

处理步骤：如果确实有多块buffer使用，可以将多个buffer合并到一块buffer，通过偏移使用。样例如下：

```cpp
// 将多个buffer合并到一块buffer，通过偏移使用
pipe.InitBuffer(que0, 1, len * 3);
pipe.InitBuffer(que1, 1, len * 3);
/*
 * 分配出3块内存大小的LocalTensor, local1的地址为que0中buffer的起始地址，
 * local2的地址为local1的地址偏移len后的地址, local3的地址为local1的地址偏移len * 2的地址
 */
int32_t offset1 = len;
int32_t offset2 = len * 2;
AscendC::LocalTensor<T> local1 = que0.AllocTensor<T>();
AscendC::LocalTensor<T> local2 = local1[offset1];
AscendC::LocalTensor<T> local3 = local1[offset2];
```

### 13.8.3 kernel侧获取Tiling信息不正确

现象描述：通过算子在kernel侧实现代码中添加PRINTF打印发现kernel侧获取的Tiling信息不正确。

比如下文样例，增加的打印代码如下：
```cpp
PRINTF("tiling_data.totalLength: %d tiling_data.tileNum: %d.\n", tiling_data.totalLength,
tiling_data.tileNum);
```

打印的Tiling数据如下，全为0：
`tiling_data.totalLength: 0 tiling_data.tileNum: 0.`

问题根因：kernel侧获取Tiling信息不正确的原因一般有以下两种：
- host侧计算Tiling的逻辑不正确
- kernel侧核函数的参数未按照正确顺序填写

处理步骤：

步骤1 参考如下示例，打印TilingData的数据，确认host侧序列化保存的TilingData是否正确。如果此时打印值有误，说明Tiling的计算逻辑可能不正确，需要进一步检查host侧Tiling实现代码，排查计算逻辑是否有误。

```cpp
std::cout<<*reinterpret_cast<uint32_t *>(context->GetRawTilingData()->GetData())<<std::endl;
// 按照实际数据类型打印TilingData第一个参数值，如需确认其他值，取值指针向后偏移即可
```

步骤2 如果上一步骤中打印的TilingData正确，需要排查kernel侧核函数的参数是否按照正确顺序填写。

使用msOpGen工具创建算子工程，并基于工程进行kernel侧算子开发时，核函数的定义模板已通过msOpGen工具自动生成，样例如下所示。参数按照"输入、输出、workspace、tiling"的顺序排布。请检查是否调整过参数顺序导致和正确顺序不一致。

```cpp
#include "kernel_operator.h"
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR
workspace, GM_ADDR tiling) {
    GET_TILING_DATA(tiling_data, tiling);// 获取Tiling参数
    // TODO: user kernel impl
}
```

----结束

### 13.8.4 Kernel编译时报错"error: out of jump/jumpc imm range"

现象描述：使用工程化算子开发方式，基于自定义算子工程进行算子开发。编译算子时失败，报如下错误：
`[ERROR] [ascendxxxx] PowerCustom_88a695f03edfbc0af76b9eaae9e4556c error: out of jump/jumpc imm range`

问题根因：该编译错误的原因是算子kernel代码过大，导致在编译时跳转指令跳转的偏移值超过了限定的大小(int16_t的数据范围)，可通过添加编译选项"-mllvm -cce-aicore-jump-expand=true"通过间接跳转的方式来避免该问题，让编译器能够正常编译。

处理步骤：

步骤1 在kernel侧的CMakeLists中通过add_ops_compile_options针对报错算子添加编译选项"-mllvm -cce-aicore-jump-expand=true"，示例如下：
`add_ops_compile_options(PowerCustom OPTIONS -mllvm -cce-aicore-jump-expand=true)`

步骤2 重新编译该算子。正常编译无报错。

----结束

### 13.8.5 使用跨版本的自定义算子包时，含有Matmul高阶API的算子存在编译或执行行报错

现象描述：
1. 基于CANN-7.2及之前版本（<=7.2）的CANN开发套件包，编译含有Matmul高阶API的自定义算子包，将编译后的自定义算子包安装至CANN-7.3及之后版本（>=7.3）的CANN包环境，然后对该含有Matmul高阶API的算子，执行图模式在线编译时，报如下错误：
`struct.error: unpack_from requires a buffer of at least 52 bytes for unpacking 4 bytes at offset 48`

2. 基于CANN-7.2及之前版本（<=7.2）的CANN开发套件包，编译sample样例仓中含有Matmul高阶API的算子（例如MatmulLeakyReluCustomSample），将编译后的自定义算子包安装至CANN-7.3及之后版本（>=7.3）的CANN包环境，然后对该含有Matmul高阶API的算子，执行单算子API的调用时，报如下错误：
`ERROR: acl executable run failed! please check your project!`

问题根因：该错误的原因是编译自定义算子包的软件版本过老，可通过更新自定义算子包编译环境上的CANN开发套件包版本，然后重新编译和部署自定义算子包，来避免出现该问题。

处理步骤：

步骤1 查看自定义算子包编译时使用的CANN开发套件包版本号，示例如下：
```bash
cd ${CANN包安装路径}
cat version.cfg
# version: 1.0
# runtime_running_version=[7.2.T11.0.B218:8.0.RC2.alpha001]
```

步骤2 基于CANN-7.3及之后版本（>=7.3）的CANN开发套件包，重新编译该自定义算子包。部署编译生成的自定义算子包后，正常编译或者执行算子，无报错。重新编译和部署自定义算子包的具体方法可参考13.2.6 算子包编译。

----结束

### 13.8.6 含有Matmul高阶API的算子精度问题

本节针对含有Matmul高阶API的算子，为排查算子精度问题是否为算子中Matmul高阶API调用方式导致，提供初步的问题定界和定位指导。如未特殊说明，下面均以Atlas A2 训练系列产品/Atlas A2 推理系列产品上的案例为例。

具体排查过程主要有如下六个步骤：

1. CPU域调试，观察报错信息；
2. Matmul Tiling是否有修改，修改是否合理；
3. 算子隐藏Vector计算，仅调用Matmul API，算子功能是否正确；
4. 单核执行，算子功能是否正确；
5. 排查Matmul API的使用是否正确；
6. 用于算子调测的golden脚本是否正确。

步骤1 CPU域调试，观察报错信息

在完成算子代码的开发后，优先通过Kernel直调中的CPU调测工程，调试算子的功能。在CPU域调试时，若编译或执行报错，日志中一般会有明显的报错信息。根据报错信息的提示内容，通常可以快速定位到问题所对应的代码位置。这种方法尤其对DataCopy参数设置错误导致的地址越界、算子Tiling参数设置不正确、其他内存访问等基础参数的使用问题，可以快速定位到具体原因。

案例：以下为matmul算子核函数的代码片段。该段代码实现了根据Global Memory上的A、B矩阵和Tiling信息，计算每个核要使用数据的地址偏移、创建Matmul对象，计算得到Matmul结果。

CPU域调试时输出执行结果，根据报错信息提示的矩阵B的transpose未定义，查看矩阵B的相关设置代码，发现Matmul对象定义时未设置矩阵B的B_TYPE::isTrans，而SetTensorB接口设置了isTransB = true，导致执行报错。所以，此问题的根因为SetTensorB设置的isTransB值与B_TYPE不符。

步骤2 Matmul Tiling是否有修改，修改是否合理

一般含有Matmul的算子Tiling实现中，通过调用GetTiling接口获取Matmul Tiling，其数据类型为TCubeTiling结构体，此时这组Tiling值是合法的。某些情况下，用户自定义了一组TCubeTiling参数值，或者基于GetTiling接口返回的TCubeTiling自行修改了其中的部分值，这样的修改需要满足参数间的制约条件。

为获取所有Tiling参数值，需要打印Tiling参数相关的日志。设置日志环境变量，获取MatmulTiling参数值。设置环境变量的命令如下：

```bash
export ASCEND_GLOBAL_LOG_LEVEL=1
export ASCEND_SLOG_PRINT_TO_STDOUT=1
```

在日志中搜索"MatmulTiling"关键字，参照TCubeTiling约束条件，检查Tiling取值是否合法。若不满足某条约束条件，需要修改对应的相关参数，使该组TCubeTiling参数值均合法。

步骤3 算子隐藏Vector计算，仅调用Matmul API，算子功能是否正确

融合算子的代码既包含Matmul API，也包含Vector计算API。通过在算子代码中删除Vector计算API，只保留Matmul API，快速定界是否为Matmul API的错误使用导致了融合算子的精度问题。具体排查过程为：修改算子代码逻辑，删除Vector计算的代码，同步完成golden脚本相应修改，完成适配修改后，CPU域或NPU域上执行算子，观察算子结果是否正确。若算子结果正确，说明代码中Matmul API的使用方式正确，需要继续排查Vector计算是否正确；反之，若算子结果不正确，需要继续排查Matmul API的使用是否正确。

案例：以融合算子matmul_leakyrelu为例，执行算子后，出现精度问题。

修改算子代码，注释屏蔽LeakyRelu API计算，同时需要适配修改相应的内存分配和涉及的同步等代码；然后注释golden脚本中LeakyRelu计算。具体修改示例如下。

以下代码为算子核函数的代码片段（注释LeakyRelu计算部分）：

```cpp
template <typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::Process(AscendC::TPipe *pipe)
{
    uint32_t computeRound = 0;
    matmulObj.SetTensorA(aGlobal);
    matmulObj.SetTensorB(bGlobal);
    matmulObj.SetBias(biasGlobal);
    while (matmulObj.template Iterate<true>()) {
        MatmulCompute();
        // LeakyReluCompute(); // 注释LeakyReluCompute Vector计算
        CopyOut(computeRound);
        computeRound++;
    }
    matmulObj.End();
}
```

MatmulCompute中将LeakyReluCompute()接口里的reluOutLocal结果输出提前到这里：

```cpp
template <typename aType, typename bType, typename cType, typename biasType>
__aicore__ inline void MatmulLeakyKernel<aType, bType, cType, biasType>::MatmulCompute()
{
    reluOutLocal = reluOutQueue_.AllocTensor<cType>();
    matmulObj.template GetTensorC<true>(reluOutLocal, false, true);
    reluOutQueue_.EnQue(reluOutLocal); // 将结果输出提前到这里
}
```

以下代码为golden生成脚本的代码片段：

```python
def gen_golden_data():
    M = 1024
    N = 640
    K = 256
    input_a = np.random.randint(-10, 10, [M, K]).astype(np.float16)
    input_b = np.random.randint(-10, 10, [K, N]).astype(np.float16)
    input_bias = np.random.randint(-10, 10, [N]).astype(np.float32)
    alpha = 0.001
    golden = (np.matmul(input_a.astype(np.float32), input_b.astype(np.float32)) + input_bias).astype(np.float32)
    # golden = np.where(golden >= 0, golden, golden * alpha)  # 与kernel保持一致，注释相应的LeakyRelu计算
    # ... 保存数据
```

删除LeakyRelu计算后，执行用例，算子结果比对正确。由此可确定，算子代码中已正确使用Matmul API，并得到了正确的Matmul API计算结果，需要继续定位LeakyReluCompute函数内LeakyRelu接口使用中存在的问题。

步骤4 单核执行，算子功能是否正确

验证单核场景下算子的功能是否正确，可以帮助快速定界是Matmul API的计算结果不符合预期，还是算子代码中错误调用Matmul API导致。由于Matmul API内部实现的是单核的计算逻辑，所以单核的计算结果正确，而多核的计算结果错误的情况，说明单核上的Matmul API的使用及计算正确，此时需要排查与多核切分相关的代码逻辑是否正确，比如每个核的输入和输出地址偏移是否正确，每个核上的尾块地址设置是否正确。如果验证单核场景下，算子精度不正确，需要排查Matmul API的使用是否正确，具体可参考步骤5。

提示：包含Matmul的算子Tiling实现中，Matmul的多核Tiling需要使用MultiCoreMatmulTiling构造多核Tiling对象，通过SetDim接口设置Matmul计算所用的核数。注意：这里设置的核数为Matmul计算所用的核数，仅在多核场景下设置，用于计算tiling参数。如下两个案例为MIX模式的算子，SetDim的设置规则请参考MIX场景核数设置规则。

- 案例1：多核切分场景，输出地址偏移不正确

以M=512，N=1024，K=512的Matmul为例，MIX模式的算子代码中设置AIC核数为4，AIV核数为8，因为本案例以分离模式为例，所以SetDim设置为AIV核数的取值8。多核场景下执行该算子，计算结果精度错误。

修改调测代码，只启动单核（blockDim设置为1，AIC核数为1，AIV核数为2，SetDim设置为AIV核数2）。修改为单核场景后，执行算子，结果比对正确（test pass）。

从上述比对结果可看出，单核验证结果正确，此时可以定界导致精度的问题与多核逻辑相关。

首先排查多核切分后的输入和输出地址偏移。分析CalcGMOffset函数，定位到矩阵C的偏移地址offsetC计算错误，正确的偏移应该是mCoreIndx \* tiling.N \* tiling.singleCoreM + nCoreIndx \* tiling.singleCoreN。将offsetC修改为正确的偏移地址后，执行算子，结果比对正确。

进一步验证：在上述单核场景的修改验证中，AIC核数为1，AIV核数为2；若想进一步验证，不引入任何多核切分，AIC核数和AIV核数均修改为1，代码修改示例如下：
  - 在核函数中REGIST_MATMUL_OBJ接口后，利用判断代码，BlockIdx不为0的AIV核退出。

- 案例2：尾块设置不正确

多核场景下，当最后一个核的singleCoreM/singleCoreN/singleCoreK值与前面的核取值不同时，需要在最后一个核上，即尾核，调用SetTail接口，调整singleCoreM/singleCoreN/singleCoreK为实际尾核上的对应数值；若尾核未设置这些参数值，或者设置的参数值大小不正确，也会导致多核精度错误，单核精度正确。

以下为算子核函数的代码片段（CalcGMOffset和核函数主体），尾核对应的M/N计算方式正确，但核函数中未调用mm.SetTail(tailM, tailN)，若此处未更新尾块会导致单核精度正确，多核失败。

步骤5 排查Matmul API的使用是否正确

经过上述步骤，可定界出是否为Matmul API使用问题。如果由于Matmul API使用错误导致了算子的精度问题，需要根据Matmul各接口的使用说明、约束条件等，检查接口的使用是否正确。

- 案例1：未遵循接口约束条件
在Matmul MDL模板下，调用IterateBatch接口，导致算子执行失败。这是由于不满足该接口的约束条件，IterateBatch接口仅支持Norm模板。
此类问题，应仔细阅读Matmul各接口中的约束条件，并排查算子实现使用的相关接口，是否满足对应接口的约束条件。

- 案例2：未遵循模板约束条件
在使能doMTE2Preload预加载模板时，若K方向非全载，不满足模板约束条件，则会导致精度比对失败。
除了满足函数接口约束条件外，也需要满足模板参数相应的约束条件，排查模板参数的使用。

步骤6 用于算子调测的golden脚本是否正确

算子的golden生成脚本，根据自定义算子的功能逻辑自行实现，用于比对算子执行结果是否正确。因此，该golden脚本的逻辑需要与算子的实现逻辑保持一致，如果golden脚本实现错误，会导致算子计算结果的精度比对失败，这种情况是golden数据不可信。

所以，在算子精度定界定位的过程中，用户需要自行根据自定义算子的逻辑，检查golden脚本的正确性，尤其是对于复杂计算逻辑的算子，需重点排查该项。

----结束

### 13.8.7 算子工程编译时出现文件名过长报错

现象描述：工程化算子开发场景，在进行算子工程编译时，提示以下报错信息中的一种：

- file name is too long (cannot be split); not dumped
```
ERROR: failed to create temporary archive: /tmp/mkself336430.tar
CMake Error at /addcustom/cmake/makeself.cmake:12 (message):
  CPack Command error: 1
tar: file name is too long (cannot be split); not dumped
tar: Exiting with failure status due to previous errors
```

- file name is too long (max 256); not dumped
```
ERROR: failed to create temporary archive: /tmp/mkself133003.tar
tar: file name is too long (max 256); not dumped
tar: Exiting with failure status due to previous errors
```

问题根因：在构建过程中，由于文件名或路径长度超出系统限制，使用tar命令打包算子工程生成的文件时发生了错误。

定位步骤：

出现此类报错，需要根据提示的报错信息（通常包含超长的文件名或者路径），减少对应的文件名长度或路径长度。

下面列出了常见错误的解决方案：

- 文件名过长报错
  - 位于算子工程op_kernel目录下的kernel侧代码和位于op_host目录下的host侧代码等文件，文件名是根据创建算子工程时传入的算子OpType自动生成的。如果因为此类文件名过长报错，应减少OpType的长度。
  - 使用Comment接口设置算子分组名称后，会对应生成同名的供GE调用的原型定义代码文件。如果因为此类文件名导致文件名过长报错，应减少算子分组名称的长度。

- 文件路径过长报错
完成工程编译相关配置时，如果在CMakePresets.json文件中配置vendor_name，编译时会在vendor目录下生成以vendor_name为名称的路径。如果因为此类文件路径过长报错，应减少配置的vendor_name长度。

### 13.8.8 调用算子时出现无法打开config.ini的报错

现象描述：自定义算子包安装部署后，在调用已部署的算子时，出现如下json文件获取失败的报错信息：

```
[INFO] Start get path and read binary_info_config.json.
[WARNING] Get jsonfile path for */binary_info_config.json failed, errmsg:No such file or directory.
[ERROR] Get path and read binary_info_config.json failed, please check if the opp_kernel package is installed!
```

通过查询前文的报错信息，上述json文件获取失败的原因是前置流程中无法打开config.ini，提示信息如下：

```
[INFO] Start to get opp kernel base path, default custom opp kernel is in ASCEND_OPP_PATH.
[INFO] The real path of config.ini is */opp/vendors/config.ini.
[WARNING] Can not open file: */opp/vendors/config.ini.
```

问题根因：根因在于当前调用算子的用户缺少对算子包部署目录下的config.ini（\*/opp/vendors/config.ini）文件的读权限。config.ini文件权限默认为640，仅允许部署用户和同属组用户访问，当前执行用户与安装用户非同一属组，缺少读权限，导致算子调用失败。

处理步骤：联系自定义算子包安装用户修改config.ini权限为644：
`chmod 644 config.ini`

### 13.8.9 算子包部署时出现权限不足报错

现象描述：部署自定义算子包时，出现如下报错信息：

```
[WARNING] The directory /usr/local/Ascend/latest/opp does not have sufficient permissions. Please check
and modify the folder permissions (e.g., using chmod), or use the --install-path option to specify an
installation path and change the environment variable ASCEND_CUSTOM_OPP_PATH to the specified path.
...
[ERROR] create /usr/local/Ascend/latest/opp/vendors/customize/framework failed
```

问题根因：当前操作用户缺少对部署路径下vendors目录的写权限。

自定义算子包默认安装路径${INSTALL_DIR}/opp/vendors的目录权限与CANN软件包安装用户和安装配置有关：root用户安装CANN，${INSTALL_DIR}/opp/vendors权限为755；非root用户携带--install for all参数安装CANN，该目录权限为755，非root用户不带--install for all参数安装CANN时，该目录权限为750。

例如，root用户安装CANN软件包后，HwHiAiUser属组用户在对应目录部署自定义算子包，因为其他用户没有写权限，会出现上述报错信息，提示权限不足导致自定义算子包部署失败。

处理步骤：

- 方法一：使用--install-path参数并配置环境变量ASCEND_CUSTOM_OPP_PATH来指定安装目录（参考指定目录安装）。运行用户需要对指定的安装路径有可读写权限。
```bash
./custom_opp_<target os>_<target architecture>.run --install-path=<path>
source <path>/vendors/<vendor_name>/bin/set_env.bash
```

- 方法二：联系CANN软件包安装用户修改默认安装路径下的vendors目录权限，比如修改为777：
`chmod 777 /usr/local/Ascend/latest/opp/vendors/`
