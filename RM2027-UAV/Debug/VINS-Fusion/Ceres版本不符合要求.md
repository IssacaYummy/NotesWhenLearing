
# bug 信息

终端报错信息：
```
yummy@yummy-asus-ubuntu:~/UAV_NAV/vins_ws$ colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
Starting >>> camera_models
Starting >>> global_fusion
--- stderr: global_fusion                                               
CMake Deprecation Warning at ThirdParty/GeographicLib/CMakeLists.txt:19 (cmake_minimum_required):
  Compatibility with CMake < 2.8.12 will be removed from a future version of
  CMake.

  Update the VERSION argument <min> value or use a ...<max> suffix to tell
  CMake that the project does not need compatibility with older versions.


/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOptNode.cpp: In function ‘void vio_callback(nav_msgs::msg::Odometry_<std::allocator<void> >::SharedPtr)’:
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOptNode.cpp:101:47: warning: ‘sensor_msgs::msg::NavSatFix_<std::allocator<void> >::ConstPtr’ is deprecated [-Wdeprecated-declarations]
  101 |         sensor_msgs::msg::NavSatFix::ConstPtr GPS_msg = gpsQueue.front();
      |                                               ^~~~~~~
In file included from /opt/ros/humble/include/sensor_msgs/sensor_msgs/msg/nav_sat_fix.hpp:7,
                 from /home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.h:28,
                 from /home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOptNode.cpp:13:
/opt/ros/humble/include/sensor_msgs/sensor_msgs/msg/detail/nav_sat_fix__struct.hpp:186:5: note: declared here
  186 |     ConstPtr;
      |     ^~~~~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp: In member function ‘void GlobalOptimization::optimize()’:
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp:115:20: error: ‘Manifold’ is not a member of ‘ceres’
  115 |             ceres::Manifold* quaternion_manifold = new ceres::QuaternionManifold();
      |                    ^~~~~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp:115:30: error: ‘quaternion_manifold’ was not declared in this scope
  115 |             ceres::Manifold* quaternion_manifold = new ceres::QuaternionManifold();
      |                              ^~~~~~~~~~~~~~~~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp:115:56: error: expected type-specifier
  115 |             ceres::Manifold* quaternion_manifold = new ceres::QuaternionManifold();
      |                                                        ^~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp:121:20: warning: ISO C++ forbids variable length array ‘t_array’ [-Wvla]
  121 |             double t_array[length][3];
      |                    ^~~~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOpt.cpp:122:20: warning: ISO C++ forbids variable length array ‘q_array’ [-Wvla]
  122 |             double q_array[length][4];
      |                    ^~~~~~~
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOptNode.cpp: In function ‘int main(int, char**)’:
/home/yummy/UAV_NAV/vins_ws/src/VINS-Fusion-ROS2/global_fusion/src/globalOptNode.cpp:179:10: warning: variable ‘qos_profile’ set but not used [-Wunused-but-set-variable]
  179 |     auto qos_profile = rclcpp::QoS(rclcpp::KeepLast(100));
      |          ^~~~~~~~~~~
gmake[2]: *** [CMakeFiles/global_fusion_node.dir/build.make:90：CMakeFiles/global_fusion_node.dir/src/globalOpt.cpp.o] 错误 1
gmake[2]: *** 正在等待未完成的任务....
gmake[1]: *** [CMakeFiles/Makefile2:154：CMakeFiles/global_fusion_node.dir/all] 错误 2
gmake: *** [Makefile:146：all] 错误 2
---
Failed   <<< global_fusion [14.4s, exited with code 2]
Aborted  <<< camera_models [41.0s]                                 

Summary: 0 packages finished [41.2s]
  1 package failed: global_fusion
  1 package aborted: camera_models
  2 packages had stderr output: camera_models global_fusion
  2 packages not processed

```

重点为
```
error: ‘Manifold’ is not a member of ‘ceres’
```

对应`globalOpt.cpp`中的第115行
```cpp
ceres::Manifold* quaternion_manifold = new ceres::QuaternionManifold();
```

这里有三个概念：

- `ceres`：Ceres 库使用的 C++ 命名空间。
- `ceres::Manifold`：描述优化变量所在流形的接口。
- `ceres::QuaternionManifold`：针对单位四元数的具体实现。

VINS 使用四元数表示旋转。四元数虽然用 4 个数存储，但单位四元数具有约束，旋转实际上只有 3 个自由度。优化时需要通过这类接口正确更新旋转，而不是随意修改 4 个分量。

**Ceres 2.0.0 使用旧的 `LocalParameterization` 接口；2.1.0 引入了新的 `Manifold` 接口。** 当前代码调用新版接口，旧版头文件没有这些类型，因此编译器报错。

# 重新安装Ceres 2.1.0

在用户目录安装2.1.0, 保留系统的 2.0.0, 为这个工作空间显式指定版本。

## 安装Ceres所需依赖

```bash
sudo apt update

sudo apt install \
  build-essential \
  cmake \
  git \
  libgoogle-glog-dev \
  libgflags-dev \
  libeigen3-dev \
  libatlas-base-dev \
  libsuitesparse-dev
```

## 获取Ceres 2.1.0 源码

```bash
mkdir third_party
cd third_party
git clone https://github.com/ceres-solver/ceres-solver.git
cd ceres-solver/
git checkout 2.1.0

```

## 编译
```bash
cmake \
  -S "~/UAV_NAV/third_party/ceres-solver" \
  -B "~/UAV_NAV/third_party/ceres-solver/build" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$HOME/.local/ceres-2.1.0" \
  -DCMAKE_INSTALL_LIBDIR=lib \
  -DBUILD_SHARED_LIBS=OFF \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
  -DBUILD_TESTING=OFF \
  -DBUILD_EXAMPLES=OFF \
  -DUSE_CUDA=OFF
  
cmake --build "~/UAV_NAV/third_party/ceres-solver-2.1.0/build" --parallel 2
# 或者直接在build目录下使用make

```

## 安装
```bash
cmake --install \
  "$HOME/third_party/ceres-solver-2.1.0/build"
```

# 重新编译
```bash
cd /home/yummy/UAV_NAV/vins_ws

CMAKE_BUILD_PARALLEL_LEVEL=2 colcon build \
  --symlink-install \
  --executor sequential \
  --cmake-clean-cache \
  --cmake-args \
    -DCMAKE_BUILD_TYPE=Release \
    -DCeres_DIR="$HOME/.local/ceres-2.1.0/lib/cmake/Ceres"
# 如果安装到用户目录（使用sudo安装）
MAKEFLAGS="-j2" colcon build --parallel-workers 1 --symlink-install --cmake-clean-cache --cmake-args -DCMAKE_BUILD_TYPE=Release -DCeres_DIR=/usr/local/lib/cmake/Ceres
```
-  `--cmake-clean-cache`: 清除之前缓存的 CMake 配置，让项目重新查找依赖。
-  `-DCeres_DIR=...`： 明确告诉 CMake 使用新安装的 Ceres 配置目录。`Ceres_DIR` 必须指向 **包含 `CeresConfig.cmake` 的目录**

使用下面的命令查找ceres安装的目录：
```bash
grep -E "CERES_VERSION_(MAJOR|MINOR|REVISION) " /usr/local/include/ceres/version.h /usr/include/ceres/version.h 2>/dev/null
```

有以下输出的目录即为2.1.0版本
```
/usr/local/include/ceres/version.h:#define CERES_VERSION_MAJOR 2 /usr/local/include/ceres/version.h:#define CERES_VERSION_MINOR 1 /usr/local/include/ceres/version.h:#define CERES_VERSION_REVISION 0
```

也可以直接查找CeresConfig.cmake文件
```bash
find /usr/local -name CeresConfig.cmake -print
```



- `--executor sequential`：不同 ROS 包按顺序构建。
- `CMAKE_BUILD_PARALLEL_LEVEL=2`：限制单个包内部的编译并行度。

