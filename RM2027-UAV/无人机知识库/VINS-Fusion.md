# 克隆源码并编译

安装依赖
```bash
sudo apt install -y ros-humble-cv-bridge ros-humble-image-transport libgoogle-glog-dev libgflags-dev libatlas-base-dev libeigen3-dev libsuitesparse-dev

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
colcon build -j4 --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
```

# 使用数据集运行VINS



```bash
source ~/vinsfusion_ws/install/setup.bash 
ros2 run vins vins_node /你的VINS配置文件完整路径.yaml
```

```bash
ros2 bag record -s mcap -o ~/bags/vins_visual_$(date +%Y%m%d_%H%M%S) /imu /camera/camera/infra1/image_rect_raw /camera/camera/infra2/image_rect_raw /odometry /path /image_track /point_cloud /camera_pose
```


