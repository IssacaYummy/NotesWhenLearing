# 环境配置

## 1. 安装基础工具
```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential cmake git curl wget vim htop python3-pip python3-venv net-tools software-properties-common
```

## 2. 安装ROS2-Humble
略

## 3. 创建工作空间目录
```bash
cd #返回Home目录

mkdir -p ~/vio_ws/src ~/px4_ws/src ~/data/bags ~/data/calib ~/data/results  # -p参数为创建子目录的同时创建父目录，否则会报错
```

- `~/vio_ws`：OpenVINS 和自己的节点。
- `~/px4_ws`：PX4 的 ROS 2 桥（Agent 和 px4_msgs），单独一个工作区，避免版本互相拖累。
- `~/data`：数据包、标定文件、结果。

## 4. 安装OpenVINS依赖和常用库
```bash
sudo apt install -y libeigen3-dev libboost-all-dev libceres-dev libopencv-dev ros-humble-cv-bridge ros-humble-image-transport ros-humble-tf2-geometry-msgs ros-humble-tf2-eigen ros-humble-rosbag2-storage-mcap
# 都是一些常用库，如eigen，ceres，opencv等，不认识的话回家吧
pip3 install --user rosbags evo scipy numpy
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc # 添加到环境变量
source ~/.bashrc
```
- rosbags：不依赖 ROS 的 bag 转换工具，能够完成ROS1 与 ROS 2 的包互转。
- evo：算轨迹误差（ATE）和画图。
- mcap：ROS 2 推荐的 bag 存储格式。
- 如果 apt 报某个包找不到，用 `apt search 包名` 看 Humble 下的实际名字。

## 5. 安装Docker（用来跑Kalibr标定程序）
```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER # $USER替换为自己的用户名
```
为什么要 Docker?
Kalibr 是 ROS1 项目，而 Ubuntu 22.04 没有官方的 ROS1 Noetic，用 Docker 里的 Ubuntu 20.04 环境最省事。

验证docker环境是否正常
```bash
docker run hello-world
```
出现下面的输出即为正常
![[Pasted image 20261023153257.png]]

# 使用数据集跑通 OpenVINS

## 1. 克隆OpneVINS算法并编译

```bash
cd ~/vio_ws/src
git clone https://github.com/rpng/open_vins.git
cd ~/vio_ws
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```
- `--symlink-install` 让你以后改 yaml 配置不必重编译。
- 编译吃内存。如果内存不够或者电脑卡死，改成：`MAKEFLAGS="-j2" colcon build --executor sequential --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release`。 `- j2`的意思为限定两个核心编译，如果内存够多也可以 `- j4` 或者 `- j8`。
## 2. 下载数据集并转为ros2格式
从 EuRoC MAV 数据集官方页面下载 MH_01_easy 的 ROS bag（ROS1 格式的 .bag 文件），放到 `~/data/bags/`。然后使用下面的命令转为ros2格式
也可以从这个链接下载： https://www.kaggle.com/datasets/chunai/euroc-mh-01-easy-ros-bag-dataset

```bash
rosbags-convert \
  --src ~/data/bags/MH_01_easy.bag \
  --dst ~/data/bags/MH_01_easy_ros2_humble \
  --dst-storage sqlite3 \
  --dst-version 8 \
  --dst-typestore ros2_humble
```

## 3. 运行算法和数据集

第一个窗口
```bash
cd ~/vio_ws
source install/setup.bash

ros2 launch ov_msckf subscribe.launch.py   config:=euroc_mav   rviz_enable:=true

```

- `config:=euroc_mav`：使用 EuRoC 的相机标定、IMU 参数和算法配置。
- `rviz_enable:=true`：同时打开 RViz，并自动加载仓库中的显示配置。

第二个窗口
```bash
cd ~/vio_ws
source install/setup.bash

ros2 bag play ~/OpenVINS/data/bags/MH_01_easy_ros2 --rate 0.5
```
`--rate 0.5` 是指以0.5倍数播放，也可以不加，以正常速度

![[Pasted image 20261023161814.png]]


# 使用D430和微空飞控上的IMU运行OpenVINS

首先OpenVINS等VIO首先分别需要相机和IMU的数据。在这里我们使用Intel RealSense D430 双目深度相机和微空H743V2 AIO 35A飞控上的IMU。

接线参考微空文档和香橙派的用户手册
https://micoair.com/zh/docs/flight-controller/micoair743-aio-series/micoair743v2-aio-35a-manual

![MicoAir743v2-AIO-35A 接口说明图](https://micoair.com/api/media/file/docs/2026/09/micoair743-aio-35a-1-01-1-df3dbd14ac.webp)

![MicoAir743v2-AIO-35A 典型接线示意图](https://micoair.com/api/media/file/docs/2026/07/69d5fe1064742-9ed1a92387-20a4f96d0d.webp)

![[PixPin_2026-10-25_11-54-02.png]]

## 1. 与飞控建立连接

使用usb连接飞控，首先打开终端，执行命令
```bash
lsusb
ls -l /dev/serial/by-id/
```
查看USB设备和串口设备
![[Pasted image 20261023183033.png]]

打开QGC地面站，连接飞控，在MAVLink控制台中输入命令：
```bash
ver all
listener sensor_combined
```

分别用于查看固件版本，确认是否有IMU数据

![[Pasted image 20261023183217.png]]

### 安装PX4的ROS2桥（Micro XRCE-DDS Agent 和 px4_msgs）

#### 克隆并编译Micro XRCE-DDS Agent 和 px4_msgs

这里我们XRCE-DDS Agent选择分支 v2.4.2，px4_msgs选择1.16分支，注意选择的版本要与ROS2版本，飞控版本对应
```bash
cd ~/px4_ws/src
git clone --branch v2.4.2 https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
git clone --branch release/1.16 https://github.com/PX4/px4_msgs.git
```

先构建Agent
```bash
cd ~/px4_ws
set -o pipefail
CMAKE_BUILD_PARALLEL_LEVEL=2 MAKEFLAGS='-j2' colcon build \
  --packages-select microxrcedds_agent \
  --executor sequential \
  --event-handlers console_direct+ \
  --cmake-args \
    -DCMAKE_BUILD_TYPE=Release \
    -DUAGENT_SUPERBUILD=OFF \
    -DUAGENT_USE_SYSTEM_FASTCDR=ON \
    -DUAGENT_USE_SYSTEM_FASTDDS=ON \
    -DUAGENT_USE_SYSTEM_LOGGER=ON \
    -DUAGENT_P2P_PROFILE=OFF \
  2>&1 | tee build_agent.log
```

这些参数使用已安装的 Humble 中间件和系统 spdlog，并关闭当前串口 DDS 链路不需要的 Agent P2P 附加功能，避免构建过程中额外下载 Client、Micro-CDR 和日志库。Fast DDS 功能保持开启。

再构建消息包
```bash
CMAKE_BUILD_PARALLEL_LEVEL=2 MAKEFLAGS='-j2' colcon build \
  --packages-select px4_msgs \
  --executor sequential \
  --event-handlers console_direct+ \
  --cmake-args -DCMAKE_BUILD_TYPE=Release \
  2>&1 | tee build_px4_msgs.log
```

#### 验证软件安装

```bash
source ~/px4_ws/install/local_setup.bash
command -v MicroXRCEAgent
ros2 pkg prefix px4_msgs
ros2 interface show px4_msgs/msg/SensorCombined
MicroXRCEAgent --help
```

Agent 路径应位于 `/home/orangepi/px4_ws/install/microxrcedds_agent/bin/`；消息包应来自同一工作区。Agent 直接通过 `MicroXRCEAgent` 命令启动，不需要 `ros2 run microxrcedds_agent ...`。

以后每个运行终端统一使用
```bash
source /opt/ros/humble/setup.bash
source ~/px4_ws/install/local_setup.bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export ROS_DOMAIN_ID=0
export ROS_LOCALHOST_ONLY=0
```
可以创建一个env.sh脚本，可用`source ~/px4_ws/env.sh`代替上面五行

### 打开OeangePi的UART

开启方法参考OrangePi的用户手册，我们打开的是Uart7

#### 检查OrangePi的串口权限

```bash
ls -l /dev/ttyS7
id -nG
```
应看到设备所属组为 dialout，如果不是则执行下面的命令将其变为输入dialout

```bash
sudo usermod -aG dialout "$USER" # "$USER"替换为自己的用户名
```

#### 查看和修改串口的波特率

```bash
stty -F /dev/ttyS7 1500000 # 设置波特率
stty -F /dev/ttyS7 speed # 检查当前实际波特率
cat /sys/class/tty/ttyS7/device/ctrl_info # 检查驱动实际的值
```
运行结果如图
![[Pasted image 20261024175635.png]]

### 连接飞控

#### 修改飞控参数

使用的飞控为微空H743V2AIO 35V飞控，连接的为Telem2口，故将飞控的下列参数修改为表格内容

| 参数                 | 设置            |     |
| ------------------ | ------------- | --- |
| `UXRCE_DDS_CFG`    | 下拉选择 `TELEM2` |     |
| `SER_TEL2_BAUD`    | `1500000`     |     |
| `UXRCE_DDS_DOM_ID` | `0`           |     |
| `UXRCE_DDS_SYNCT`  | 启用            |     |
|                    |               |     |
波特率选择1500000是因为OrangePi无法设置为921600，且应该使波特率尽量高

#### 启动Agent

终端A：
```bash
source ~/px4_ws/env.sh
MicroXRCEAgent serial --dev /dev/ttyS7 -b 1500000 -v 4
```
其中`--dev /dev/ttyS7`  表示选择uart7
`-b 1500000` 表示设置波特率为1500000
`-v 4` 表示将日志详细程度设为 INFO（普通信息）级别。

终端B：
```bash
source ~/px4_ws/env.sh
ros2 topic list -t
ros2 topic echo /fmu/out/sensor_combined \
  px4_msgs/msg/SensorCombined \
  --qos-reliability best_effort --qos-durability volatile --once
```
之后观察频率
```bash
ros2 topic hz /fmu/out/sensor_combined
```


## 2. 安装RealSense SDK

 安装ROS驱动

```bash
sudo apt install -y 'ros-humble-librealsense2*' 'ros-humble-realsense2-*'
```

 启用两路红外图像
```bash
ros2 launch realsense2_camera rs_launch.py \
  enable_color:=false \
  enable_depth:=false \
  enable_infra1:=true \
  enable_infra2:=true \
  depth_module.infra_profile:=848x480x30
```

另外打开一个终端看是否有相关话题，并查看话题发布频率
```bash
ros2 topic list
ros2 topic hz /camera/camera/infra1/image_rect_raw
```

再使用rqt-image-viewer查看相机的图像

```bash
sudo apt install ros-humble-rqt-image-view
ros2 run rqt_image_view rqt_image_view
```

结果如图

![[PixPin_2026-10-24_18-15-37.png]]

## 3. 标定

### 确认标定板参数，填写配置
```yaml
target_type: checkerboard
targetCols: 17
targetRows: 17
rowSpacingMeters: 0.015
colSpacingMeters: 0.015
```
填的是内部角点数量，尺度单位是米

## 4. 导入自己的参数

