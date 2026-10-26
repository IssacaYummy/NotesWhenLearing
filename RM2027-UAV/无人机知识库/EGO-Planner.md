# 配置环境

## 1. 安装依赖

```bash

sudo apt update

sudo apt install \
  build-essential cmake git \
  python3-colcon-common-extensions python3-rosdep \
  libeigen3-dev libarmadillo-dev libboost-all-dev \
  libopencv-dev libpcl-dev \
  libvtk9-dev libvtk9-qt-dev qtbase5-dev \
  ros-humble-rviz2 \
  ros-humble-cv-bridge ros-humble-image-transport \
  ros-humble-pcl-ros ros-humble-pcl-conversions \
  ros-humble-message-filters \
  ros-humble-tf2-ros ros-humble-tf2-geometry-msgs \
  ros-humble-rclcpp-components \
  ros-humble-rmw-cyclonedds-cpp
```

## 2. 克隆源码并切换到ROS2分支

```bash
mkdir -p ~/planner_ws/src
cd ~/planner_ws/src
git clone https://github.com/ZJU-FAST-Lab/ego-planner-swarm.git
git checkout ros2_version
```

## 3.配置CyclonDDS

该分支作者报告默认 Fast DDS 下存在明显卡顿，并建议使用 Cyclone DDS。[分支说明](https://github.com/ZJU-FAST-Lab/ego-planner-swarm/blob/ros2_version/Readme.md)

在后续每一个运行终端中设置：

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

## 4. 安装剩余依赖

首次使用 `rosdep` 时初始化
```bash
sudo rosdep init
rosdep update
```

然后安装依赖
```bash
rosdep install --from-paths src --ignore-src \
  --rosdistro humble -y \
  --skip-keys "Eigen3 PCL OpenCV Boost Armadillo dynamic_reconfigure"
```

## 5. 编译源码

首先保持 CPU 感知模式。检查：
`/planner_ws/src/uav_simulator/local_sensing/CMakeLists.txt`

其中应为：

```CMakeLists.txt
set(ENABLE_CUDA false)
```

然后在工作空间根目录编译：

```bash
cd ~/planner_ws

colcon build --symlink-install \
  --parallel-workers 2 \
  --cmake-args -DCMAKE_BUILD_TYPE=Release

source install/setup.bash
```

# 文件结构
# 仿真
## 单机默认仿真

终端一：
```bash
cd planner_ws/
source install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

ros2 launch ego_planner rviz.launch.py
```

终端二：
```bash
cd setup.bash/
source install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

ros2 launch ego_planner single_run_in_sim.launch.py \
  use_mockamap:=False \
  use_dynamic:=False
```

两个参数含义如下：

|参数|`False`|`True`|
|---|---|---|
|`use_mockamap`|Random Forest 地图|mockamap 噪声地图|
|`use_dynamic`|使用位置指令转里程计的简化仿真|使用四旋翼动力学与 SO3 控制器|

