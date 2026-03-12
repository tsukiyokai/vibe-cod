# HelloWorld-快速入门-入门教程-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_map_10_0004
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_map_10_0004.html
---

# HelloWorld

本示例展示了如何使用Ascend C编写一个简单的"Hello World"程序，包括核函数（设备侧实现的入口函数）的实现、调用流程以及编译运行的完整步骤。通过本示例，您可以快速了解Ascend C的基本开发流程。完整样例请参考LINK。

代码文件hello_world.asc包括核函数实现和主函数实现。

- 核函数实现：该核函数的核心逻辑是输出"Hello World!"字符串。
- 主函数实现：在主函数中，进行初始化环境、申请资源、通过<<<>>>调用核函数以及释放资源等操作。完整的代码流程和逻辑可以通过代码注释查看。

| 12345678910111213141516171819202122232425262728293031 | // Host侧应用程序需要包含的头文件#include"acl/acl.h"// Kernel侧需要包含的头文件#include"kernel_operator.h"__global____vector__voidhello_world(){AscendC:printf("[Block (%lu/%lu)]: Hello World!\n",AscendC:GetBlockIdx(),AscendC:GetBlockNum());}int32_tmain(intargc,charconst*argv[]){// 初始化aclInit(nullptr);// 运行管理资源申请int32_tdeviceId=0;aclrtSetDevice(deviceId);aclrtStreamstream=nullptr;aclrtCreateStream(&stream);// 设置参与计算的核数为8（核数可根据实际需求设置）constexpruint32_tblockDim=8;// 用内核调用符<<<>>>调用核函数hello_world<<<blockDim,nullptr,stream>>>();aclrtSynchronizeStream(stream);// 资源释放和去初始化aclrtDestroyStream(stream);aclrtResetDevice(deviceId);aclFinalize();return0;} |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

完成代码实现后，可以通过两种方式，对上述代码进行编译：

- 使用bisheng命令行进行编译bisheng hello_world.asc --npu-arch=dav-2201 -o demo
./demo
- 使用CMake进行编译CMake编译配置如下：cmake_minimum_required(VERSION 3.16)

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
)编译和运行步骤如下：mkdir -p build && cd build; 
cmake ..;make -j;
./demo

![](../images/atlas_ascendc_10_00043_img_002.png)

运行结果如下，本样例共调度8个核，打印了核号和"Hello World!"等信息。
