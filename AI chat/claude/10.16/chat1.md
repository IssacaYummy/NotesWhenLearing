# M1 操作手册：D430 + 飞控 IMU 的 VIO 上板

面向第一次复现开源 VIO、且只用过 ArduPilot 的读者 · 2026-10-16

## 0. 先建立直觉

VIO（Visual-Inertial Odometry，视觉惯性里程计）的原理是：相机告诉你“画面里的特征点怎么动了”，IMU 告诉你“机体在转、在加速”，算法把两者互相校正，输出位置和姿态。它对三件事极其敏感：

1. 相机内参，以及相机到 IMU 的外参（extrinsics，相对位置和朝向）；
2. 相机与 IMU 的时间偏移（time offset）；
3. IMU 噪声参数。

任何一个错了，轨迹就会“一碰就飞”。社区里用 D435i 跑 VINS-Fusion 的 IMU 模式，最常见的抱怨就是位姿一动就乱跳，通常要靠 Kalibr 标定才能解决。所以本手册的重点不是“把程序跑起来”，而是“把标定做对，并且分步定位问题”。

分步原则：每次只引入一个新变量。顺序是：公开数据集（数据和标定都是对的）→ 自己的数据离线 → 笔记本实时 → 板子实时 → 接 PX4。哪一步坏了，问题就在那一步新增的东西里。

## 1. 路线图

|   |   |   |
|---|---|---|
|步骤|在哪做|通过线（我给的起始参考，按实测调整）|
|S1|PC|EuRoC 数据集上 VINS-Fusion 不发散，轨迹形状与真值一致|
|S2|PC|D430 红外双目稳定录包，发射器已关，曝光固定|
|S3|PC|PX4 的 IMU 经 MAVROS 出到 ROS，≥200 Hz、抖动小、静止时重力约 9.8|
|S4|PC|Kalibr 相机重投影误差小，相机-IMU 标定残差大多在 3σ 内|
|S5|PC|自己的 bag 上跑 VINS，手持走 10 m 方形回到起点，漂移 <3%|
|S6|Zero 3W|实时 ≥20 Hz，CPU <70%，端到端延迟已测量|
|S7|Zero 3W + 飞控|PX4 收到 VIO，EKF2 位置随手动移动跟随 VIO（不装桨）|

## 2. 开工前的三个决定

### 2.1 PX4 还是 ArduPilot

- 你的板子是 MicoAir743v2 一类。厂商页面写明 v2 官方支持 PX4，PX4 目标名为 micoair_h743-v2；AIO 版的目标是 micoair_h743-aio，需 PX4 1.16.0 及以上。ArduPilot 官方支持从 4.6 起。先确认手上是 v2 还是 AIO，固件目标不同。
- 厂商页面还写明：PX4 目前不支持双向 DShot（bidirectional DShot），ArduPilot 支持。双向 DShot 提供电机转速，可用于转速陷波滤波来降低 IMU 振动噪声，这对 VIO 的 IMU 质量有帮助。PX4 这边少一个工具，只能靠减振安装和软件滤波补。这是我的推理，PX4 的替代方案没有核实。
- M1 的 S1–S6 不需要飞起来，所以两个固件的区别只影响 S3（取 IMU）和 S7（喂 VIO）。真正在 PX4 上调飞行是 M2 的事。M1 先别纠结，按本手册用 PX4。

### 2.2 ROS 版本策略

- VINS-Fusion 是 ROS1 项目，Ubuntu 20.04 加 ROS Noetic 最省事。Kalibr 同样是 ROS1，社区指南里要求较老的 Ubuntu/ROS，用官方 Docker 最省事（我没核实镜像细节）。所以 S1–S5 在一台 Ubuntu 20.04 的 PC 或 Docker 里做，不建议用 WSL（USB 直通很麻烦）。
- Zero 3W 的官方 Ubuntu 镜像基于厂商 sunxi 内核（社区反馈为 6.6.98），不是主线内核。上板时优先用 Docker 装 ROS Noetic 的 arm64 镜像，绕开系统版本问题。先跑 `docker run hello-world`，确认 Docker 在这个内核上能用（未核实）。
- ROS2 加 uXRCE-DDS 的路线以后再看。PX4 的 ROS2 桥有现成话题，但先别同时学两套。

### 2.3 硬件接线风险（先单独验证）

- Zero 3W 只有一个 USB 3.1 口（Type-C OTG），另一个 Type-C 是 USB 2.0 且用来供电。D430 需要 USB3，占用这个口；飞控不能再走 USB，要走 40-pin 排针上的 UART（3.3V 电平，引脚号请对照官方针脚图，我没核实）。
- 需要实测：OTG 口在主机模式下能否给 D430 供足电，D430 能否稳定出 848×480 红外流不掉线。不行才考虑带供电的 USB 集线器，它会增加重量。
- 飞控与相机必须刚性固定在同一个机架上。标定之后两者的相对位置一旦变了，外参就作废，要重标。

## 3. 逐步操作

### S1 在 PC 上跑通 VINS-Fusion（EuRoC 数据集）

依赖：ROS Noetic、OpenCV、Eigen、Ceres（官方推荐 1.14；新版 Ceres 可能要改 CMake 的 C++ 标准，报错先查仓库 issues）。数据集用 EuRoC MAV 的 MH_01_easy。下面每条命令各开一个终端：

```
mkdir -p ~/catkin_ws/src && cd ~/catkin_ws/src
git clone https://github.com/HKUST-Aerial-Robotics/VINS-Fusion.git
cd ~/catkin_ws && catkin_make
source devel/setup.bash
roslaunch vins vins_rviz.launch
rosrun vins vins_node ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_stereo_imu_config.yaml
rosbag play MH_01_easy.bag
```

做完后：

- 在 rviz 里看轨迹是否平滑、是否与房间形状一致。
- 再换成 `euroc_stereo_config.yaml`（不用 IMU）跑一遍，对比有无 IMU 的差别。
- 打开 yaml，逐个认识这些字段：imu_topic、image0_topic、image1_topic、body_T_cam0、body_T_cam1、estimate_extrinsic、estimate_td、td、acc_n、gyr_n。S5 要改它们。
- 可选：用 evo 工具算轨迹误差（ATE，Absolute Trajectory Error，绝对轨迹误差）。输出文件的格式需要按 evo 文档转换。

通过线：不要求逐个数字复现论文，要求不发散、轨迹形状对。

### S2 用 D430 录数据

1. 在 PC 上装 librealsense 和 realsense-ros（ROS1 分支，我没核实当前分支名）。
2. 先用 realsense-viewer 手动试：只开 Infrared 1 和 Infrared 2，分辨率 848×480、30 fps；关闭 Emitter（红外发射器）；关闭自动曝光，手动设一个较短的曝光（室内可从 5–10 ms 试起，是经验值，目标是不拖影又不太暗）。
3. 用 launch 启动（参数名以你装的版本为准）：

```
roslaunch realsense2_camera rs_camera.launch enable_infra1:=true enable_infra2:=true enable_depth:=false enable_color:=false infra_width:=848 infra_height:=480 infra_fps:=30
rostopic hz /camera/infra1/image_rect_raw
```

为什么关发射器：D430 的点阵投射器会在场景里打出固定图案，这些点跟着相机一起动，会被当成特征点，污染 VIO。这个解释是我的；“要关发射器”本身是各个 VIO 教程的共识。为什么用红外而不是 RGB：D430 没有 RGB 相机，红外是全局快门，适合运动场景。

通过线：左右图帧率稳定，在 realsense-viewer 里图像清晰不过曝。

### S3 从 PX4 取 IMU 数据

```
sudo apt install ros-noetic-mavros ros-noetic-mavros-extras
sudo /opt/ros/noetic/lib/mavros/install_geographiclib_datasets.sh
roslaunch mavros px4.launch fcu_url:=/dev/ttyACM0:921600
rosrun mavros mavcmd long 511 105 5000 0 0 0 0 0
rostopic hz /mavros/imu/data_raw
```

- 511 是 MAV_CMD_SET_MESSAGE_INTERVAL，105 是 HIGHRES_IMU，5000 微秒对应 200 Hz（这些编号凭记忆，请用 MAVLink 文档核对）。
- PC 阶段用飞控的 USB 口（设备名 ttyACM0），上板时改走 UART。
- 达不到 200 Hz 时，查 PX4 对应 mavlink 实例的模式（MAV_n_MODE）和串口波特率。PX4 实际能给出的最高 IMU 速率没有核实，请实测。
- 静止放桌上看 `rostopic echo /mavros/imu/data_raw`，线加速度 z 应约 9.8；转动飞控，角速度应有合理变化。
- 时间同步：MAVROS 会做 timesync（飞控与主机的时钟对齐）。PX4 论坛的案例里，timesync 往返时延 18 ms 被认为是串口链路有问题的信号。同步的残余误差由 S4 的 time offset 估计来补。

通过线：≥200 Hz、`rostopic hz` 的抖动小，静止重力约 9.8。

### S4 Kalibr 标定（决定成败的一步）

1. 打印 Kalibr 的 AprilGrid 标定板（april_6x6 的 yaml），贴平，实测方格尺寸后写进 yaml。
2. 录相机标定包：标定板在不同距离和角度，覆盖整个视野，约 1–2 分钟。

```
rosbag record -O cam.bag /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw
rosrun kalibr kalibr_calibrate_cameras --bag cam.bag --topics /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw --models pinhole-radtan pinhole-radtan --target april_6x6.yaml --show-extraction
```

3. IMU 噪声参数：理想做法是 Allan 方差（Allan variance）标定，把 IMU 静置数小时，工具如 allan_variance_ros（Kalibr 论坛有人静置了 20 小时）。入门替代：用 BMI088 数据手册的噪声密度，再放大数倍（5–10 倍是经验做法，未核实），后面在 S5 里调。写成 imu.yaml：

```
rostopic: /mavros/imu/data_raw
update_rate: 200.0
accelerometer_noise_density: 0.01
accelerometer_random_walk: 0.001
gyroscope_noise_density: 0.001
gyroscope_random_walk: 0.0001
```

（上面的数字只是占位，不要照抄。）

4. 相机-IMU 联合标定：录包时三个轴都要充分激励，平移加旋转，同时保持标定板在视野内，约 1–2 分钟。

```
rosrun kalibr kalibr_calibrate_imu_camera --target april_6x6.yaml --cam camchain.yaml --imu imu.yaml --bag cam_imu.bag
```

5. 读结果：重投影误差大多应落在 3σ 范围内（社区经验），并得到 T_cam_imu 和 timeshift_cam_imu。
6. 一个常见的坑：VINS 的 body_T_cam0 是“相机到 IMU”，而 Kalibr 给的 T_cam_imu 是“IMU 到相机”，需要求逆。社区里很多人在这里搞混。timeshift 的符号约定也要核对。

通过线：重投影误差小，联合标定残差大多在 3σ 内，并且手量的相机与飞控距离与标定的平移量大致吻合。

### S5 自己的数据离线跑 VINS

1. 把 `euroc_stereo_imu_config.yaml` 复制成 `d430_stereo_imu.yaml`，修改：话题名、分辨率、`cam0_calib` 和 `cam1_calib`（左右相机内参文件）、`body_T_cam0` 和 `body_T_cam1`、噪声参数、`estimate_extrinsic: 1`（把标定值当初值让优化微调，确信标定很准时可设 0）、`estimate_td: 1`、`td` 初值取 Kalibr 的结果。
2. 排错阶梯，每步只改一项（社区案例里“不用 IMU 正常、一开 IMU 就乱”很常见）：

- 先用不带 IMU 的双目配置，验证相机标定；
- 再开 IMU，固定外参；
- 再放开外参和 td 的估计；
- 最后调噪声参数。

4. 采集动作：开始静止 2–3 秒，再缓慢平移加转动。避免纯旋转，避免只对着白墙。
5. 评估：手持走 10 m 方形回到起点，漂移 = 起终点距离 ÷ 路程。关闭 loop_fusion 来评估纯 VIO。

通过线：漂移 <3%（我给的起始线），且 rviz 里轨迹没有突然跳变。

### S6 实时运行与上板

1. 先在笔记本上同时接 D430 和飞控，实时跑通。用 `rostopic delay /vins_estimator/odometry` 看消息时间戳到收到时刻的延迟，记下来，S7 要用。
2. 上板：用 Docker 的 arm64 ROS Noetic 镜像。编译 Ceres 加 VINS 在小板子上可能要一小时以上，可以在 PC 上用 `docker buildx build --platform linux/arm64` 交叉构建（走 qemu，速度也慢），或者直接在板上编译。
3. 板上先 `lscpu` 找出 2 颗 A76 大核的编号，用 `taskset -c` 把 vins_node 绑到大核上，A55 留给相机驱动和 MAVROS（绑核是我的建议）。
4. 算力不够时，调参旋钮按顺序：

- 降分辨率（需要重新标定，因为内参变了）；
- 减少特征点数 max_cnt（默认 150 可试 100）；
- 缩短 max_solver_time（0.04 可试 0.03）。

6. 监控：`htop` 看 CPU，`cat /sys/class/thermal/thermal_zone*/temp` 看温度，防止降频。

通过线：实时 ≥20 Hz，CPU <70%，延迟已记录。

### S7 接入 PX4 的 EKF2

**固件与参数**（均以官方参数表为准）：

- 用 QGroundControl 烧录对应你板子的 PX4 目标，确认版本满足 2.1 的要求。
- `EKF2_EV_CTRL`：外部视觉融合的位掩码，对应水平位置、垂直位置、速度、yaw。起步只开位置，之后再逐步加速度和 yaw。社区有人用值 3 并关掉 GPS 融合、把高度参考设为 Vision。
- 关闭 GPS 融合；高度参考设为 Vision（参数名我记为 EKF2_HGT_REF，请核对）。
- `EKF2_EV_DELAY`：设为 S6 测得的延迟（参数名凭记忆，请核对）。
- `SDLOG_PROFILE`：勾选第 7 位 Computer Vision and Avoidance，这样日志里会记录视觉数据速率。
- 如果 QGC 因为没有 GPS 拒绝解锁，去官方参数文档里找与 GPS 或位置相关的预解锁检查项。

**中继节点**：PX4 官方推荐用 MAVROS 把 `nav_msgs/Odometry` 发到 `/mavros/odometry/out`，frame_id 设为 odom，child_frame_id 设为 base_link，MAVROS 会转换成 MAVLink 的 ODOMETRY（LOCAL_FRD）再进入 PX4。ODOMETRY 是唯一能同时带线速度的消息。VINS 输出的话题是 `/vins_estimator/odometry`，它的机体系就是你飞控 IMU 的坐标系，所以只需要改两个 frame 名：

```
#!/usr/bin/env python3
import rospy
from nav_msgs.msg import Odometry
rospy.init_node('vins_to_mavros')
pub = rospy.Publisher('/mavros/odometry/out', Odometry, queue_size=10)
def cb(m):
    m.header.frame_id = 'odom'
    m.child_frame_id = 'base_link'
    pub.publish(m)
rospy.Subscriber('/vins_estimator/odometry', Odometry, cb, queue_size=10)
rospy.spin()
```

协方差字段 PX4 如何使用，请查官方 EKF2 外部视觉章节。

**验证步骤（不装桨）**：

1. 启动 mavros（上板时 `fcu_url` 用 UART 设备名，以板子为准）、vins_node 和中继节点。
2. 在 QGC 的 MAVLink Inspector 里看 ODOMETRY。PX4 会把收到的外部位姿回传，数据一停它也停，可以当健康检查。
3. 在 MAVLink Console 里执行 `listener vehicle_visual_odometry` 和 `listener vehicle_local_position`。
4. 手持机体沿 1 m 方形缓慢移动，`vehicle_local_position` 应当跟着 VIO 走。事后用 Flight Review 或 PlotJuggler 对比外部位姿和 EKF2 估计。
5. 航向问题：EKF2 的参考系航向与外部位姿的航向通常不一致，开启 yaw 融合后让 EKF2 去对齐，细节见官方 EKF2 外部视觉章节。

**EKF2 不融合时先查**：消息频率是否太低（过低 EKF2 会拒绝融合）、坐标系是否有误（PX4 用 NED/FRD，ROS 用 ENU/FLU）、`EKF2_EV_CTRL` 是否已设置、timesync 往返时延是否过大。另外 PX4 不支持 GLOBAL_VISION_POSITION_ESTIMATE、VISION_SPEED_ESTIMATE、VICON_POSITION_ESTIMATE 这几种消息，不要用它们。

通过线：手持移动时 EKF2 位置跟随 VIO，无跳变，日志里能看到视觉数据速率。

## 4. ArduPilot → PX4 对照（凭记忆，仅作对照，以官方文档为准）

|   |   |
|---|---|
|你熟悉的 ArduPilot|PX4 里的对应|
|Mission Planner|QGroundControl|
|外部定位源 EK3_SRC1_* 设为 ExternalNav，加 VISO_TYPE|EKF2_EV_CTRL 等 EKF2_* 参数|
|外部定位消息 VISION_POSITION_ESTIMATE 或 ODOMETRY|推荐 ODOMETRY|
|GUIDED 模式加 MAVLink 目标点|Offboard 模式，需要持续发送 setpoint（频率需大于约 2 Hz，否则会退出）|
|串口 SERIALx_PROTOCOL|MAV_x_CONFIG 等|
|飞行日志 .bin|.ulg，用 Flight Review、PlotJuggler 或 pyulog 看|

## 5. 故障速查

|   |   |   |
|---|---|---|
|现象|常见原因|先试这个|
|一碰相机轨迹就乱飞|外参或时间偏移错、IMU 噪声参数偏小、激励不足|回到 S5 的排错阶梯，先不用 IMU 验证相机标定|
|画面里有规律的亮点|红外发射器没关|在 viewer 或 launch 里关发射器|
|轨迹缓慢发散|自动曝光导致亮度变化，特征跟踪不稳|固定曝光|
|只有上板才发散|CPU 不够，或者降频过热|降特征点数，绑核，看温度|
|IMU 速率不稳或掉包|波特率低、MAVLink 实例模式不对|提高波特率，检查 MAV_n_MODE|
|D430 掉线|OTG 口供电不足或线材差|换短线，实测供电，最后才考虑带电集线器|
|PX4 不融合 VIO|频率低、坐标系错、参数没设、timesync 差|按 S7 的清单逐项查|
|Kalibr 编译或运行失败|ROS 或 Ubuntu 版本不匹配|用 Docker|

## 6. 验收清单与记录

M1 验收（全部通过再进入 M2）：

- S1 EuRoC 轨迹正确
- S2 D430 红外双目稳定，发射器关，曝光固定
- S3 IMU ≥200 Hz，重力正确
- S4 Kalibr 相机和联合标定通过
- S5 10 m 方形漂移 <3%
- S6 板上实时 ≥20 Hz，CPU <70%，延迟已测
- S7 PX4 位置跟随 VIO，且日志能回放

每次实验保存：rosbag、yaml 配置、代码提交哈希、相机与飞控的安装状态（有没有拆装过）、光照和场景描述、结果（漂移 %、帧率、CPU、延迟）。标定文件要和 bag 一起存，并记下标定日期，以便知道它是否还有效。

## 7. 来源与未核实项

来源：

- PX4 VIO 文档：https://docs.px4.io/main/en/computer_vision/visual_inertial_odometry
- PX4 外部位置估计：https://docs.px4.io/main/en/ros/external_position_estimation.html
- PX4 的 uXRCE-DDS 话题清单：https://github.com/PX4/PX4-Autopilot/blob/main/src/modules/uxrce_dds_client/dds_topics.yaml
- VINS-Fusion 运行示例（社区分叉）：https://github.com/rkuo2000/VINS-Fusion
- D435i 加 VINS 的 Kalibr 流程：https://github.com/cwsfa/kalibr_d435i
- 相机-IMU 标定指南：https://github.com/Zhefan-Xu/camera-imu-calibration-guide
- 社区里 D435i 配 VINS-Fusion 的坑：https://www.github.com/engcang/vins-application/issues/2
- MicoAir743v2 产品页：https://micoair.com/?p=4855
- Zero 3W 参数：https://www.cnx-software.com/?p=168818

未核实项：VINS-Fusion 官方仓库当前状态和 Ceres 兼容性、realsense-ros 的 ROS1 分支与参数名、Kalibr 的 Docker 镜像、Zero 3W 上 Docker 是否可用、40-pin 的 UART 引脚号、OTG 口供电能力、MAVLink 消息编号、PX4 的 EKF2 参数名与位掩码含义、各项通过线的数值。