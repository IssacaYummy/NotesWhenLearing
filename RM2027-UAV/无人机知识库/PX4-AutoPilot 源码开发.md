# 环境准备
## 1. 创建工作空间并克隆源码

以目录/home/yummy/UAV_NAV/firmware-work/ 作为工作目录示例，同时创建固件备份目录和编译工具链目录

```bash
mkdir -p /home/yummy/UAV_NAV/firmware-work/
mkdir -p /home/yummy/UAV_NAV/firmware-work/backup
mkdir -p /home/yummy/UAV_NAV/firmware-work/toolchains
```

克隆PX4源码，并切换v1.16.2分支

```bash
cd 
cd UAV_NAV/firmware-work/
git clone --branch v1.16.2 --recursive \
  https://github.com/PX4/PX4-Autopilot.git
```

## 2. 安装编译依赖，并在独立python环境中工作

准备独立python环境
```bash
sudo apt update
sudo apt install python3-venv wget xz-utils
```

```bash
python3 -m venv \
  /home/yummy/UAV_NAV/firmware-work/px4-venv
```
使用下面的命令进入虚拟环境
```bash
source /home/yummy/UAV_NAV/firmware-work/px4-venv/bin/activate
```

进入源码目录并运行源码自带的依赖安装脚本

```bash
bash Tools/setup/ubuntu.sh
# bash Tools/setup/ubuntu.sh --no-sim-tools
```
添加-no-sim-tools为跳过仿真工具，该脚本会自动安装依赖包

## 3. 检查GCC工具

运行下面的命令检查GCC版本，推荐使用GCC 13.2进行构建
```bash
arm-none-eabi-gcc --version
```

如果不是的话则按照下面的步骤安装GCC 13.2

```bash
wget \
  https://developer.arm.com/-/media/Files/downloads/gnu/13.2.rel1/binrel/arm-gnu-toolchain-13.2.rel1-x86_64-arm-none-eabi.tar.xz \
  -O /home/yummy/UAV_NAV/firmware-work/toolchains/arm-13.2.tar.xz
```

创建独立安装目录并解压：

```bash
mkdir -p \
  /home/yummy/UAV_NAV/firmware-work/toolchains/arm-13.2
```

```bash
tar -xJf \
  /home/yummy/UAV_NAV/firmware-work/toolchains/arm-13.2.tar.xz \
  --strip-components=1 \
  -C /home/yummy/UAV_NAV/firmware-work/toolchains/arm-13.2
```

只在当前编译终端选择它：

```bash
export PATH="/home/yummy/UAV_NAV/firmware-work/toolchains/arm-13.2/bin:$PATH"
hash -r
```

检查：

```bash
command -v arm-none-eabi-gcc
arm-none-eabi-gcc --version
```

## 4. 初次编译

先更新子模块
```bash
git submodule update --init --recursive
```
然后编译
```bash
make -j4 micoair_h743-v2_default
```

通常应生成：

```text
build/micoair_h743-v2_default/micoair_h743-v2_default.px4
```

将编译产物保存到backup文件夹：

```bash
cp \
  build/micoair_h743-v2_default/micoair_h743-v2_default.px4 \
  /home/yummy/UAV_NAV/firmware-work/backup/baseline-micoair_h743-v2.px4
```

## 修改源码编译

**修改源码之前一定要新建自己的分支**

```bash
git switch -c my-px4
```

### 修改dds发布间隔

找到文件src/modules/uxrce_dds_client/dds_topics.h.em，并找到：

```cpp
orb_set_interval(fds[idx].fd, UXRCE_DEFAULT_POLL_RATE);
```

将这一行替换为：

```cpp
const unsigned int interval_ms =
    (send_subscriptions[idx].orb_meta == ORB_ID(sensor_combined))
        ? 5u
        : UXRCE_DEFAULT_POLL_RATE;

orb_set_interval(fds[idx].fd, interval_ms);
```

保存后重新编译：
```bash
make -j2 micoair_h743-v2_default
```

成功后，拷贝到back文件夹
```bash
cp \
  build/micoair_h743-v2_default/micoair_h743-v2_default.px4 \
  /home/yummy/UAV_NAV/firmware-work/backup/imu-dds-200hz-micoair_h743-v2.px4
```

然后可以通过QGC等上位机刷写固件

