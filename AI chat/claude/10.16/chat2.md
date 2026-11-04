教程已经按 Ubuntu 22.04 + ROS 2 Humble 重写，加进了同一份文档的第二部分，里面有完整的环境配置。

### 1. 必须跑 VINS-Fusion 吗？首个复现选谁？

不必须，我建议先复现 **OpenVINS**，理由在下表。

||OpenVINS|VINS-Fusion|SchurVINS|
|---|---|---|---|
|后端|EKF（滤波器）|滑窗非线性优化，可选回环|EKF，把完整残差模型用 Schur 补拆成自身运动和路标两部分，基于 SVO 2.0 开发|
|ROS 支持|ROS1、ROS 2 都有，并支持不依赖 ROS 的编译|官方是 ROS1，社区有 ROS 2 移植版（质量自己判断）|catkin 工程，用 vcs-import 拉依赖再 catkin build，没看到 ROS 2 支持|
|在线标定|能估计相机到 IMU 的变换和时间偏移|有外参和时间偏移估计，但社区反映配 D435i 开 IMU 时位姿容易乱跳|我没查到|
|文档|有独立的文档站，并能直接读 Kalibr 的相机-IMU 标定文件|主要靠 README 和社区|只提供了 EuRoC 的运行示例|
|算力|滤波后端，应当较省|优化后端较重|论文目标就是资源受限设备 [github.com +7](https://github.com/bytedance/schurvins)|

**算力方面的间接证据**：另一篇同类的 Schur 补滤波方法的论文，在 Jetson Orin NX 上 比 VINS-Fusion 快约 2.2 倍。注意它不是 SchurVINS 本身，只能说明“滤波后端更省”这个趋势，不能当成 SchurVINS 的数据。 [arxiv](https://arxiv.org/pdf/2512.20355)

**对你的适配**：

- SchurVINS 理论上最对症，因为你的 A733 只有 2 个大核。但它是 ROS1 加 SVO 依赖，文档少，对第一次复现的人风险最大。
- OpenVINS 在 Humble 上也有坑：社区有人遇到编译报错，需要改 tf2 相关头文件，还有人在树莓派 4 上跑 ROS 2 版出现发散（原因没定论，可能是算力）。所以板子上的表现一定要实测。 [github](https://github.com/rpng/open_vins/issues/403)[github](https://www.github.com/rpng/open_vins/issues/435)
- 顺序建议：先 OpenVINS，把“采集、标定、跑通”的流程做通。再读 VINS-Fusion 的源码，学 IMU 预积分和滑窗优化。算力不够时再试 SchurVINS。

**前置知识**：你学过的坐标变换、四元数、协方差和 Jacobian 够用了，还缺这几块，建议按顺序补：

1. EKF 和误差状态 EKF（预测加更新，姿态用误差扰动表示），Solà 的《Quaternion kinematics for the error-state Kalman filter》是常用参考；
2. IMU 的噪声模型（白噪声、随机游走）和 Allan 方差；
3. MSCKF 的思路：把多帧相机位姿放进状态，用特征约束它们，不把路标放进状态。OpenVINS 文档站有对应章节；
4. 针孔相机加径向切向畸变，以及 Kalibr 标定的流程；
5. ROS 2 的节点、话题、QoS、bag；Linux、CMake、colcon 的基本用法。

这些都是我凭经验列的，不是检索得来的清单。

### 2. 必须用 ROS 吗？

算法上不必须。OpenVINS 可以不依赖 ROS 编译。但 ROS 2 对你有用的地方是相机驱动、录包回放、rviz 可视化，所以你用 ROS 2 是合适的，我已经按它改了教程。

和 PX4 对接，ROS 2 确实更方便：PX4 有官方的 ROS 2 桥，话题里就有 /fmu/in/vehicle_visual_odometry。但有两个代价： [github](https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml)

- PX4 用 NED/FRD 坐标系，ROS 常用 ENU/FLU，要自己转换； [px4](https://discuss.px4.io/t/sending-gps-data-from-ros-2-to-px4-via-micro-xrce-dds/34301)
- PX4 的时间戳也不是 ROS 时间。

教程里的桥接节点把这两件事都处理了，并且标了“这是我自己的简单设计”。

还有一个限制：Kalibr 只认 ROS1 的 bag，而且 Ubuntu 22.04 不支持 ROS1 Noetic，所以标定这一步在 Docker 里做，流程是 ROS 2 录包、用 rosbags-convert 转成 bag 再标定。 [ros](https://answers.ros.org/question/403382/convertuse-rosbag1-with-ros2-on-ubuntu-2204)[github](https://github.com/ripl-lab/allan_variance_ros2/wiki/Kalibr-Camera-IMU-Calibration)

### 3. 教程里有什么

文档第二部分（P0–P12）：

- **环境配置**：从系统检查、ROS 2 Humble、OpenVINS 依赖、Docker、RealSense、QGC，到 PX4 的 ROS 2 桥，最后有自检清单。
- **S0–S7**：每步有命令和通过线，其中飞控 IMU 接入 ROS 2、Kalibr 标定、OpenVINS 自己的数据和注入 PX4 这几步最容易出问题。
- **PX4 的版本匹配**：这里最容易出问题。px4_msgs 要和飞控固件版本一致，Agent 版本也要对得上，文档里给了查法，但具体对应表我没有拿到，请以官方文档为准。

需要你实测后告诉我的一个数：PX4 的 sensor_combined 实际频率。我没有核实它，如果低于 100 Hz，VIO 效果会受影响，我们要换取 IMU 的办法。

文档末尾列了未核实项，开工前请对照官方文档核对。

教程已写好，可以直接阅读、修改或批注。