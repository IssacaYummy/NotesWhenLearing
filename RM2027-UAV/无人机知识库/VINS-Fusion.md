# 克隆源码并编译

安装依赖
```bash
sudo apt install -y ros-humble-cv-bridge ros-humble-image-transport \ libgoogle-glog-dev libgflags-dev libatlas-base-dev libeigen3-dev libsuitesparse-dev

```

克隆源码
```bash
mkdir -p vins_ws/src
cd vins_ws/src

git clone https://github.com/zinuok/VINS-Fusion-ROS2.git
```

打开 `vins/src/featureTracker/feature_tracker.h`，把第 14 行左右的 `#define GPU_MODE 1` 注释掉，改用 CPU 版

编译

```bash
cd ~/vins_ws
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
```

# 使用数据集运行VINS

