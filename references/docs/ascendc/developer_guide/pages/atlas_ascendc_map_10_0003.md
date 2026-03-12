# 环境准备-入门教程-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_map_10_0003
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_map_10_0003.html
---

# 环境准备

- 进行Ascend C算子开发之前，需要安装驱动固件和CANN软件包，请参考《CANN软件安装指南》完成环境准备。安装驱动固件（仅昇腾设备需要），安装步骤请参见“安装NPU驱动和固件”章节。安装CANN软件包，可参考“快速安装CANN”完成快速安装，可参考其他章节了解更多场景的安装步骤。安装CANN软件后，使用CANN运行用户进行编译、运行时，需要以CANN运行用户登录环境，执行source ${install_path}/set_env.sh命令设置环境变量。其中${install_path}为CANN软件的安装目录，例如：/usr/local/Ascend/cann。
- 安装cmake。通过cmake编译Ascend C算子时，要求安装3.16及以上版本的cmake，如果版本不符合要求，可以参考如下示例安装满足要求的版本。示例：安装3.16.0版本的cmake（x86_64架构）。mkdir -p cmake-3.16 && wget -qO- "https://cmake.org/files/v3.16/cmake-3.16.0-Linux-x86_64.tar.gz" | tar --strip-components=1 -xz -C cmake-3.16
export PATH=`pwd`/cmake-3.16/bin:$PATH

![](../images/atlas_ascendc_10_00043_img_002.png)
