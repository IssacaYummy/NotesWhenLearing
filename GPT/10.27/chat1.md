**你的设备可以按“飞控基础校准 → 双目标定 → IMU 噪声测量 → 相机与飞控刚性固定 → 相机—IMU 联合时空标定 → 独立数据验证”的顺序进行。**其中最容易出问题的，是棋盘格识别方向、飞控 IMU 数据的实际含义，以及相机和飞控的时间戳。

你已确认我读取的设备记录正确。下面按这套硬件编写：

|项目|本教程采用的设备条件|
|---|---|
|相机|RealSense D430 双目模组＋D4|
|飞控|MicoAir743V2-AIO-35A，PX4 1.16|
|采集端|Orange Pi，Ubuntu 22.04、ROS 2 Humble|
|飞控通信|UART＋uXRCE-DDS，已有约 100 Hz `SensorCombined` 输出记录|
|图像模式|已有双路 Y8、848×480、30 Hz 的设备记录|
|标定板|金属棋盘格，标称单格 15 mm；行列数、实际尺寸仍应实物复核|
|计算端|笔记本负责 Kalibr 和噪声分析|

硬件信息来自你确认的[既有设备记录](/home/yummy/OpenVINS/docs/D430_PX4_IMU_标定流程.md)。**这些记录不表示设备现在已经启动，也不表示标定已经完成。**

本次我只读取了既有文件并检索资料，**没有新增或修改任何本地文件**。下面的命令、目录和 YAML 都是供你以后操作的教程示例；录包和标定命令实际执行后才会生成数据、配置和报告。

---

**1．先理解每项标定到底解决什么问题。**

可以把相机和 IMU 理解成两个人同时描述同一次运动：相机根据图像判断运动，IMU 根据角速度和加速度判断运动。要把它们结合起来，必须知道它们各自如何测量、安装在哪里，以及各自的时间对应关系。

|标定项目|通俗解释|最终需要得到什么|
|---|---|---|
|左右相机内参|每个镜头怎样把空间中的点拍到图像上|焦距、主点、畸变参数|
|双目外参|右相机相对左相机的位置和朝向|左右相机之间的旋转和平移|
|飞控 IMU 基础校准|修正传感器的基本零偏、尺度等误差|飞控内部应用的校正参数|
|IMU 噪声测量|描述读数短时间抖动和偏置缓慢变化的程度|四个噪声参数|
|相机—IMU 空间标定|相机相对实际输入 IMU 坐标系的位置和朝向|`T_cam_imu`|
|相机—IMU 时间标定|同一运动在两种时间戳中相差多少|`timeshift_cam_imu`|

Kalibr 官方将多相机标定、相机—IMU 时空标定，以及 IMU 内参标定列为不同功能。因此，**“运行一次 Kalibr”并不自动完成上表所有工作**。[Kalibr 官方功能说明](https://github.com/ethz-asl/kalibr/wiki)

还有两点与你的硬件直接相关：

- 本教程的 IMU 来自**飞控**。Kalibr 的 RealSense 示例主要使用 D455 内置 IMU，不能直接套用其 IMU 话题、噪声值或外参。[Kalibr RealSense 教程](https://github.com/ethz-asl/kalibr/wiki/Calibrating-Intel-RealSense-Cameras)
- D430 左右成像器是**固定焦距、全局快门**，官方标称双目基线为 **50 mm**。全局快门仍然会因曝光过长产生运动模糊；50 mm 可用于核查结果，但不能代替实际标定。[RealSense D400 数据手册，表 3-11、3-13](https://www.realsenseai.com/download/21345/?tmstv=1778232235)

---

**2．先检查棋盘格：角点数、尺寸和红外可见性都必须正确。**

Kalibr 支持普通棋盘格。配置中的 `targetCols`、`targetRows` 填的是**内部角点数**，不是黑格数量，也不是黑白格总数。[Kalibr 标定板说明](https://github.com/ethz-asl/kalibr/wiki/Calibration-targets)

例如：

```
一行共有 18 个黑白方格
→ 这一方向共有 17 个内部角点

一列共有 18 个黑白方格
→ 另一方向也有 17 个内部角点
```

内部角点是四个相邻方格交会的位置，板外边缘不算内部角点。

按照你原有记录，如果实物确实是 **18×18 个方格、每格 15 mm**，则：

- 内部角点：17×17。
- 黑白图案区域：约 270×270 mm，不含外框。
- 单格边长：`0.015 m`。

对应的示例配置为：

```
target_type: checkerboard
targetCols: 17
targetRows: 17
rowSpacingMeters: 0.015
colSpacingMeters: 0.015
```

**只有数过实物并测量一致，才能使用这些数值。**

建议你这样核查：

1. 沿横向数完整方格数量，再沿纵向数一次。
2. 横向跨 10 个完整方格测量，测得长度除以 10；纵向也测一次。
3. 检查板面是否平整，是否存在翘曲、凹陷。
4. 保留棋盘格外围白边；官方建议白边至少有一个网格单元宽。
5. 分别从左右红外图像检查黑白格是否清楚。

其中跨多个方格测量，是我建议的降低测量误差的方法；板面平整、实测尺寸和外围白边要求来自 Kalibr 官方说明。[Kalibr 标定板注意事项](https://github.com/ethz-asl/kalibr/wiki/Calibration-targets#tipsproblems)

**你的金属板尤其要检查反光。**肉眼看起来清楚，不代表在 D430 红外图像中同样清楚。实际画面如果出现大片发白、黑白对比很弱，或倾斜后格子突然消失，应先调整照明、板面方向和拍摄距离，再开始录包。

**棋盘格还有一个无法忽略的限制：对称性。**

Kalibr 官方明确警告，棋盘格、圆点阵列等对称图案可能发生目标姿态翻转。你的正方形棋盘格需要特别留意这一点。[Kalibr 联合标定中的棋盘格警告](https://github.com/ethz-asl/kalibr/wiki/Camera-IMU-calibration#2-collect-images)

采集时建议：

- 保持整个棋盘图案可见。
- 尽量避免图案出画后反复重新进入。
- 避免大幅绕光轴转动，使图案方向容易混淆。
- 检查角点编号及估计姿态是否连续，左右图像对应是否一致。

这些做法只能降低风险，**不能保证消除对称歧义**。如果姿态反复翻转、基线异常或多次外参明显不一致，应先解决目标识别问题；必要时再换用官方推荐的 AprilGrid。

---

**3．完成飞控基础校准，并准备最终的机械安装。**

先通过 QGroundControl 完成飞控本身的基础校准。

加速度计操作：

1. 连接飞控，打开 QGroundControl。
2. 进入车辆设置中的传感器页面。
3. 核对飞控安装方向。
4. 选择加速度计校准。
5. 按屏幕提示依次摆放各个方向。
6. 每个方向都保持静止，直到对应提示完成。

具体姿态以 QGC 界面为准，不需要凭感觉追求绝对精确的 90°。[PX4 1.16 加速度计校准](https://docs.px4.io/v1.16/en/config/accelerometer)

陀螺仪操作：

1. 选择陀螺仪校准。
2. 将设备放在稳定表面。
3. 开始后保持完全静止。
4. 等待校准完成。

校准过程中移动设备会影响过程。[PX4 1.16 陀螺仪校准](https://docs.px4.io/v1.16/en/config/gyroscope)

**QGC 校准完成后，仍然要做 IMU 噪声测量和相机—IMU 联合标定。**它们解决的是不同问题。另外，不能据此认定 IMU 所有轴不正交、比例因子和温度误差都已经完全消除；Kalibr 对输入 IMU 内参校正有明确要求。[Kalibr IMU 输入要求](https://github.com/ethz-asl/kalibr/wiki/Camera-IMU-calibration#1-requirements)

机械安装按下面的原则准备：

- 相机和飞控在联合标定时必须形成一个整体。
- 固定好 D430 模组及其实际支架，避免手持时模组弯曲或松动。
- 移动组件时，相机与飞控之间不能发生相对转动、平移。
- 整理 USB、UART 和供电线，避免线缆拉动相机。
- 尽量采用以后实际运行时的安装方式。
- 如果飞控通过软连接安装，要关注它是否会相对相机晃动：固定外参模型无法描述这种持续变化。

这里的刚性要求，是由“求固定旋转和平移”的标定模型直接决定的。

操作时拆下螺旋桨、保持电机不解锁，用手移动组件即可。双目标定和 IMU 静态噪声测量可以先分别进行；**联合标定必须等相机与飞控固定好以后再做**。

---

**4．准备 D430 的左右图像，先确认图像质量，再考虑算法。**

本教程固定使用：

```
cam0：infra1
cam1：infra2

分辨率：848×480
格式：Y8
帧率：30 Hz
```

此模式来自你已确认的设备记录。仍建议先运行：

```
rs-enumerate-devices
```

查看实际连接设备支持的模式。RealSense 官方也要求根据设备支持的 profile 设置图像模式。[RealSense ROS 参数说明](https://github.com/realsenseai/realsense-ros#parameters)

在 Orange Pi 的一个终端中，加载你已有环境，然后启动相机：

```
source /home/orangepi/px4_ws/env.sh

ros2 launch realsense2_camera rs_launch.py \
  camera_namespace:=camera \
  camera_name:=camera \
  enable_color:=false \
  enable_depth:=false \
  enable_infra1:=true \
  enable_infra2:=true \
  depth_module.infra_profile:=848x480x30 \
  depth_module.infra1_format:=Y8 \
  depth_module.infra2_format:=Y8 \
  enable_sync:=true
```

参数名可能随驱动版本变化；若报参数不存在，应先核对安装版本的参数，而不是忽略报错继续采集。

另开终端检查：

```
source /home/orangepi/px4_ws/env.sh

ros2 topic list -t
```

找到左右图像，再分别观察频率：

```
ros2 topic hz /camera/camera/infra1/image_rect_raw
```

观察数秒后按 `Ctrl+C`，再运行：

```
ros2 topic hz /camera/camera/infra2/image_rect_raw
```

检查尺寸：

```
ros2 topic echo /camera/camera/infra1/camera_info --once
```

应重点看：

- 两路都有持续图像。
- 实际尺寸为 848×480。
- 发布频率接近所设的 30 Hz。
- 日志没有持续断流或 USB 错误。

实际话题名以 `ros2 topic list -t` 为准，后面所有命令都必须同步使用实际名字。

通过图像查看器观察：

```
ros2 run rqt_image_view rqt_image_view
```

这个命令需要图形桌面环境；仅有普通 SSH 终端时不会直接显示窗口。

左右画面都要检查：

|观察项目|合适的状态|
|---|---|
|棋盘格边缘|清晰，没有明显拖影|
|黑白对比|角点容易辨认|
|反光|没有大片过曝区域覆盖角点|
|大小|格子足够清楚，整块图案仍可见|
|动态画面|组件移动时仍能分辨角点|

D430 是固定焦距镜头，通常需要调整的是拍摄距离、照明和曝光。[RealSense 数据手册，表 3-13](https://www.realsenseai.com/download/21345/?tmstv=1778232235)

对于投射器，我建议标定时关闭，避免散斑叠加到棋盘格上。先检查参数：

```
ros2 param list /camera/camera
```

如果存在对应参数，再执行：

```
ros2 param set /camera/camera depth_module.emitter_enabled 0
```

这是针对角点提取的操作建议；参数用法来自 RealSense 官方驱动。[RealSense 参数读取和设置方法](https://github.com/realsenseai/realsense-ros#parameters)

还要分清两个问题：

- `enable_sync` 是 RealSense 驱动内部对不同图像流的处理选项，**不会把外接飞控 IMU 自动变成硬件同步设备**。
- 话题名包含 `image_rect_raw`，不能单凭名字判断全部几何处理过程。标定、联合标定和 OpenVINS 必须使用相同图像模式与处理链；如果另外做了裁剪、缩放或去畸变，参数也必须与其对应。[RealSense `enable_sync` 定义](https://github.com/realsenseai/realsense-ros#parameters)

---

**5．把飞控 IMU 整理成标准消息，首先保证单位、坐标系和采样时间正确。**

你的飞控链路是：

```
飞控 IMU
  ↓ PX4
SensorCombined
  ↓ uXRCE-DDS / Agent
/fmu/out/sensor_combined
  ↓ 转换
/imu：sensor_msgs/msg/Imu
```

`SensorCombined` 不能直接当作 Kalibr 的标准 IMU 输入。

PX4 1.16 官方定义中：

|输入字段|含义|
|---|---|
|`gyro_rad`|FRD 机体系中的角速度，rad/s|
|`accelerometer_m_s2`|FRD 机体系中的加速度读数，m/s²|
|`timestamp`|陀螺仪对应时间，PX4 内部以微秒表示|
|`accelerometer_timestamp_relative`|加速度计时间相对 `timestamp` 的差值|
|`gyro_integral_dt`、`accelerometer_integral_dt`|对应测量的采样周期|

这些读数是经过相应尺度、偏置补偿的采样周期平均值，不能简单称作“未经处理的芯片原始数据”。[PX4 1.16 `SensorCombined` 定义](https://raw.githubusercontent.com/PX4/PX4-Autopilot/release/1.16/msg/SensorCombined.msg)

标准 `/imu` 至少要满足：

- `angular_velocity`：rad/s。
- `linear_acceleration`：m/s²。
- `header.stamp`：正确的测量时间。
- 明确记录实际使用的坐标系。
- 没有姿态测量时，用 `orientation_covariance[0] = -1` 标记不可用。[ROS 2 Humble `Imu.msg` 定义](https://raw.githubusercontent.com/ros2/common_interfaces/humble/sensor_msgs/msg/Imu.msg)

**不要在转换时减去重力，也不要把静止加速度强行改为零。**

静止检查时应看到：

```
角速度三个分量接近 0
加速度向量模长接近 9.81 m/s²
```

不能要求静止时加速度三个分量都接近零。

你的既有[转换脚本](/home/yummy/OpenVINS/tools/calibration/px4_sensor_combined_to_imu.py)保留 FRD：前、右、下。只要标定和运行始终一致，外参可以描述它与相机坐标系的关系。

注意：ROS 常见机体系是 FLU，相机光学系通常是右、下、前。**修改 `frame_id` 字符串不会旋转数值**；后续也不能在已经转换过的数据上再做一次 FRD→FLU。[ROS 坐标约定 REP-103](https://raw.githubusercontent.com/ros-infrastructure/rep/master/rep-0103.rst)

**这里还有一个已核对出的版本问题，正式采集前必须检查。**

当前本地安装的 `px4_msgs` 包含 `accel_calibration_count`、`gyro_calibration_count`，既有转换脚本会访问它们；我读取的官方 `release/1.16` 消息定义中没有这两个字段。因此，不能仅凭“PX4 1.16”这个名称就认定飞控、ROS 消息包和脚本完全兼容。

应在**实际采集的 Orange Pi 环境**检查：

```
ros2 interface show px4_msgs/msg/SensorCombined
```

并核对实际固件、消息定义及消息翻译机制是否匹配。PX4 官方对 ROS 2 消息版本兼容有专门说明。[PX4 ROS 2 用户指南](https://docs.px4.io/v1.16/en/ros2/user_guide)

已有脚本还明确要求：

```
accelerometer_timestamp_relative == 0
```

如果不是零，它会拒绝该样本。**不能删除检查后直接把两个不同时刻的测量拼成一个“同步”IMU 消息**；需要先实现正确的时间对齐。本次我只读了脚本，没有运行或修改它。

链路启动后检查：

```
ros2 topic echo /fmu/out/sensor_combined \
  --once \
  --qos-reliability best_effort
```

转换节点运行后，再检查：

```
ros2 topic hz /imu

ros2 topic echo /imu \
  --once \
  --qos-reliability best_effort
```

如果采用现有脚本，应确认 `published` 持续增加，各种拒绝计数没有持续增长。

---

**6．先弄清时间戳：时钟同步、测量时刻和时间偏移不是同一件事。**

对你的系统，要检查三个层次：

|层次|要解决的问题|
|---|---|
|时钟域一致|两种时间戳是否能放在同一条时间轴上|
|时间戳语义正确|标记的是测量时刻，还是电脑收到消息的时刻|
|联合时间标定|正确时间戳之间是否仍有稳定的残余偏移|

PX4 官方说明，真实飞控与伴随计算机之间默认由 uXRCE-DDS 管理时间同步，可通过 `/fmu/out/timesync_status` 观察统计。[PX4 1.16 时间同步说明](https://docs.px4.io/v1.16/en/ros2/user_guide#ros-gazebo-and-px4-time-synchronization)

但“存在同步机制”不等于你这次采集已满足精度要求。应检查：

1. Agent 和飞控客户端正常连接。
2. 同步状态已稳定。
3. 相机与 `/imu` 的时间戳覆盖同一采集时段。
4. 没有开机时间与 Unix 时间混用。
5. 没有在已转换的时间上重复加偏移。
6. 单个传感器的时间戳严格递增。
7. 没有明显大段缺失、重复时间或周期性跳变。

最重要的操作原则是：

> 保留每个样本自己的测量时间，不能统一用回调中的 `now()` 替换。

例如，系统突然收到一批 IMU 消息，并不意味着这些测量真的同时发生。Kalibr 官方专门展示过批量发送、错误时间戳造成的异常采样间隔，并要求修复这类问题。[Kalibr 时间戳异常示例](https://github.com/ethz-asl/kalibr/wiki/Camera-IMU-calibration#4-the-output)

对于采样间隔，可以先用这些数建立直觉：

|实际频率|理想相邻间隔|
|---|---|
|图像 30 Hz|约 33.3 ms|
|IMU 100 Hz|约 10 ms|
|IMU 200 Hz|约 5 ms|

**检查的是消息中的时间戳间隔，不只是终端显示的到达频率。**

你目前约 100 Hz 的输出可以用于先检查链路和尝试标定，但不能预先保证结果质量。Kalibr 给出的成功采集经验为相机 20 Hz、IMU 200 Hz；OpenVINS 建议相机 20–30 Hz、IMU 200–500 Hz。[Kalibr 采集建议](https://github.com/ethz-asl/kalibr/wiki/Camera-IMU-calibration#2-collect-images)、[OpenVINS 动态标定建议](https://docs.openvins.com/gs-calibration.html#calib-dynamic)

这里的 200 Hz 不是官方声明的绝对最低门槛。对你的设备，我建议优先争取**稳定、真实的 200 Hz 或以上输出**；若暂时只有 100 Hz，则必须更加依赖报告、多组重复和独立验证判断。

提高 UART 波特率不会自动增加 IMU 发布频率，重复旧消息也不会增加有效信息。

---

**7．采集第一类数据：双目标定数据。此时可以固定相机、移动棋盘格。**

这一阶段只求左右相机自身的几何参数，不要求飞控参与。

建议先录一个短试验包，确认两路都有图像、棋盘格能检测，再正式采集。下面用 `stereo01` 作为示例名称。

Orange Pi 上的示例命令：

```
ros2 bag record -s sqlite3 \
  -o /home/orangepi/calibration/bags/stereo01 \
  /camera/camera/infra1/image_rect_raw \
  /camera/camera/infra2/image_rect_raw \
  /camera/camera/infra1/camera_info \
  /camera/camera/infra2/camera_info
```

输出目录的父目录需要事先准备好。每次换新的包名，避免与已有目录冲突。ROS 2 bag 用于保存消息，之后可以回放检查。[ROS 2 官方录包教程源码](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.rst)

以下是我为你的棋盘格拟定的采集动作：

1. 相机固定，板先放在画面中央。
2. 调整距离，使整块图案在左右画面中都清晰可见。
3. 将板分别移向画面左、右、上、下及四角。
4. 改变板与相机的距离。
5. 让板左右倾斜、前后倾斜，形成不同透视。
6. 每到一个姿态稍作停顿，再换下一个姿态。
7. 在接近图像边缘时，仍保持完整棋盘图案可见。

**只把始终正对相机的板左右平移，姿态变化不够。**官方要求覆盖不同距离、方向和图像区域。[Kalibr 多相机数据采集](https://github.com/ethz-asl/kalibr/wiki/Multiple-camera-calibration#1-collect-images)

建议第一组先录约 1–2 分钟，作为操作起点；这不是官方规定时长。关键是有效姿态丰富，而非视频越长越好。

录完按 `Ctrl+C`，让 rosbag 正常结束，再检查：

```
ros2 bag info /home/orangepi/calibration/bags/stereo01
```

确认左右都有消息、帧数相近、时长符合预期，再抽看开始、中间和结束的画面。

还要提前考虑存储。按你的模式计算：

```
848 × 480 × 1 字节 × 2 路 × 30 Hz
≈ 24.4 MB/s
≈ 1.47 GB/分钟
```

这是纯像素数据的十进制估算，尚未计入消息和存储开销。历史记录中的剩余空间不能当作当前空间，录制前实际检查：

```
df -h /home/orangepi/calibration
```

---

**8．在笔记本上运行 Kalibr，求双目内参和双目外参。**

你的采集环境是 ROS 2 Humble，而官方 Kalibr 工作流使用 ROS 1 bag。推荐采用：

```
Orange Pi 录 ROS 2 bag
        ↓ 完整复制录包目录
笔记本转换为 ROS 1 .bag
        ↓
ROS 1 Noetic Docker 中运行 Kalibr
```

这是 Kalibr 官方 ROS 2 指南推荐的接入方式。[Kalibr ROS 2 标定指南](https://github.com/ethz-asl/kalibr/wiki/ROS2-Calibration-Using-Kalibr)

假设以后将数据放到：

```
/home/yummy/OpenVINS/data/calib/
```

其中包括完整的 `stereo01` 录包目录，以及按实物尺寸准备的 `target.yaml`。

转换示例：

```
rosbags-convert \
  --src /home/yummy/OpenVINS/data/calib/stereo01 \
  --dst /home/yummy/OpenVINS/data/calib/stereo01.bag
```

`rosbags-convert` 需要先安装在合适的 Python 环境中。当前官方文档使用 `--src`、`--dst` 形式。[Rosbags 转换文档](https://ternaris.gitlab.io/rosbags/topics/convert.html)

转换时注意：

- 保留原始 ROS 2 包作为备份。
- 输入完整录包目录，包括 metadata 和所有分包。
- 优先录制标准 `Image`、`CameraInfo`、`Imu`。
- 不要指望转换工具把 `SensorCombined` 自动变成标准 `Imu`。
- 转换后重新检查话题、消息数及时间戳。

你的本地已经有 Kalibr 源码和官方 Dockerfile。以后需要构建时，可按官方方式执行：

```
cd /home/yummy/OpenVINS/Kalibr/kalibr

docker build \
  -t kalibr:noetic \
  -f Dockerfile_ros1_20_04 .
```

前提是笔记本已安装可用 Docker，并有足够的磁盘和内存。[Kalibr 官方 Docker 安装方法](https://github.com/ethz-asl/kalibr/wiki/Installation#using-the-docker-images)

进入容器的无界面示例：

```
docker run --rm -it \
  --entrypoint /bin/bash \
  -e MPLBACKEND=Agg \
  -v /home/yummy/OpenVINS/data/calib:/data \
  kalibr:noetic
```

这里：

```
笔记本 /home/yummy/OpenVINS/data/calib
              ↕
容器 /data
```

容器里运行：

```
source /catkin_ws/devel/setup.bash

rosrun kalibr kalibr_calibrate_cameras \
  --bag /data/stereo01.bag \
  --target /data/target.yaml \
  --models pinhole-radtan pinhole-radtan \
  --topics \
    /camera/camera/infra1/image_rect_raw \
    /camera/camera/infra2/image_rect_raw \
  --bag-freq 10.0 \
  --dont-show-report
```

参数含义：

|参数|作用|
|---|---|
|`--bag`|输入录包|
|`--target`|棋盘格定义|
|`--models`|左右相机分别使用的模型|
|`--topics`|左右图像话题，顺序决定相机编号|
|`--bag-freq`|降低用于角点提取的频率，减少计算量|
|`--dont-show-report`|保存报告，结束时不弹出报告窗口|

`--topics` 中第一个对应 `cam0`，第二个对应 `cam1`，模型顺序必须一致。[Kalibr 多相机命令说明](https://github.com/ethz-asl/kalibr/wiki/Multiple-camera-calibration#2-running-the-calibration)

这里先用 `pinhole-radtan`，是结合 D430 镜头特性作出的模型选择；最终仍要根据残差判断。不能仅因视场较宽就直接认定必须使用鱼眼模型。[Kalibr 支持的相机模型](https://github.com/ethz-asl/kalibr/wiki/Supported-models)

按当前官方源码，常见输出为：

```
stereo01-camchain.yaml
stereo01-results-cam.txt
stereo01-report-cam.pdf
```

以实际终端输出的路径为准。[Kalibr 当前输出文件命名源码](https://github.com/ethz-asl/kalibr/blob/master/aslam_offline_calibration/kalibr/python/kalibr_calibrate_cameras)

**不要只检查是否生成 YAML。**打开 PDF，检查角点覆盖、重投影误差和双目几何结果。OpenVINS 给出的良好相机标定参考约为 **0.2–0.5 像素**；这是质量参考，不是单一合格线。[OpenVINS 相机标定检查](https://docs.openvins.com/gs-calibration.html)

对于 D430，还应检查双目平移向量的长度是否接近 **0.05 m**。若结果是几十厘米、几米，应优先查棋盘格尺寸单位、角点对应及目标姿态翻转。

---

**9．采集第二类数据：IMU 长时间静态数据，用于噪声分析。**

这一阶段不需要棋盘格，也不需要持续录制双目图像。

操作建议：

1. 先完成飞控基础校准。
2. 保持以后实际使用的 IMU 输出路径、滤波和发布频率。
3. 通电后等待温度和读数趋于稳定。
4. 放在稳定、低振动的表面。
5. 保持供电、通信和整个装置稳定。
6. 只录 `/imu`。

可以先录 20–30 分钟排查数据质量，但这段时间主要用于发现断流、时间跳变或校准变化，不能当作完整长时间噪声分析。

正式时长要区分来源：

|来源|采集时长建议|
|---|---|
|`allan_variance_ros` 作者说明|至少 3 小时|
|Kalibr 噪声模型说明|约 15–24 小时静态记录|
|OpenVINS 标定教程|20 小时以上|

所以，对你这套设备，建议**先用短包排错，正式噪声测量尽量安排约 20 小时或更长**。工具能分析 3 小时数据，不代表已充分观察到所有较慢变化。[Allan 原工具说明](https://github.com/ori-drs/allan_variance_ros)、[Kalibr IMU 噪声模型](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model)、[OpenVINS IMU 噪声标定](https://docs.openvins.com/gs-calibration.html)

对 ROS 2 Humble，可选择作者提供的 ROS 2 版本 `ripl-lab/allan_variance_ros2`。它是另一个项目，不能称为 Kalibr 自带工具。[ROS 2 Allan 工具说明](https://github.com/ripl-lab/allan_variance_ros2/wiki)

该工具使用 MCAP。采集端如果缺少存储插件，需要以后安装：

```
sudo apt install ros-humble-rosbag2-storage-mcap
```

录包：

```
ros2 bag record -s mcap \
  -o /home/orangepi/calibration/bags/imu_static01 \
  /imu
```

正式开始前确认：

- 选用同一个物理 IMU，避免传感器切换后混成一段。
- 没有持续丢包和拒绝样本。
- 没有量程裁剪。
- 没有在采集中重新执行基础校准。
- 录制时温度条件尽量稳定。
- 远程连接断开不会意外终止采集。

最后一项可通过持久终端会话安排；不要在长时间采集中反复操作设备。

---

**10．计算 Allan 曲线，得到四个连续时间噪声参数。**

下面给出 ROS 2 工具的一条操作路径。计算建议在笔记本完成，先加载 Humble 环境，并按项目说明准备 `rosbag2_py`、MCAP 插件及 Python 依赖。[工具前提条件](https://github.com/ripl-lab/allan_variance_ros2/wiki#prerequisites)

假设以后将该工具放在：

```
/home/yummy/OpenVINS/tools/allan_variance_ros2
```

噪声分析配置示例：

```
imu_topic: /imu
imu_rate: 100
measure_rate: 100
sequence_time: 72000
```

含义：

- `imu_rate`：实际稳定的输入频率。
- `measure_rate`：分析时保留的频率。
- `sequence_time`：分析时长，单位秒。

这里 `100` 只适用于实际为 100 Hz 的数据；`72000` 对应 20 小时。实际录制不足 20 小时就不能照填。该工具配置字段来自作者示例。[工具配置示例](https://github.com/ripl-lab/allan_variance_ros2/blob/main/config/realsense_d435i.yaml)

先找到真实 MCAP 文件名。以下仅假设原始文件是：

```
/home/yummy/OpenVINS/data/calib/imu_static01/imu_static01_0.mcap
```

按作者推荐方式，以消息头时间重新写入分析用包：

```
python3 /home/yummy/OpenVINS/tools/allan_variance_ros2/src/cookmcap.py \
  --input /home/yummy/OpenVINS/data/calib/imu_static01/imu_static01_0.mcap \
  --output /home/yummy/OpenVINS/data/calib/imu_cooked.mcap
```

脚本会按输出文件的 stem 创建目录，实际生成文件名要查看后确认。这个操作**不会修复原本错误的测量时间戳**。[`cookmcap.py` 源码](https://github.com/ripl-lab/allan_variance_ros2/blob/main/src/cookmcap.py)

假设其输出为 `imu_cooked/imu_cooked_0.mcap`：

```
python3 /home/yummy/OpenVINS/tools/allan_variance_ros2/src/allan_variance_computer.py \
  --input_mcap /home/yummy/OpenVINS/data/calib/imu_cooked/imu_cooked_0.mcap \
  --output_dir /home/yummy/OpenVINS/data/calib/allan_result \
  --imu_config_yaml /home/yummy/OpenVINS/data/calib/allan_config.yaml
```

再分析并输出：

```
python3 /home/yummy/OpenVINS/tools/allan_variance_ros2/src/allan_variance_analysis.py \
  --data /home/yummy/OpenVINS/data/calib/allan_result/allan_variance.csv \
  --config /home/yummy/OpenVINS/data/calib/allan_config.yaml \
  --output /home/yummy/OpenVINS/data/calib/imu.yaml
```

这些参数已与作者脚本核对。[计算脚本](https://github.com/ripl-lab/allan_variance_ros2/blob/main/src/allan_variance_computer.py)、[分析脚本](https://github.com/ripl-lab/allan_variance_ros2/blob/main/src/allan_variance_analysis.py)

如果原始数据有多个分包，上面的单文件示例不能直接代表完整记录；需要按工具支持的方式处理完整序列，不能遗漏后半段。

最后得到的核心参数是：

|YAML 字段|含义|单位|
|---|---|---|
|`gyroscope_noise_density`|陀螺仪白噪声密度|rad/s/√Hz|
|`accelerometer_noise_density`|加速度计白噪声密度|m/s²/√Hz|
|`gyroscope_random_walk`|陀螺仪偏置随机游走|rad/s²/√Hz|
|`accelerometer_random_walk`|加速度计偏置随机游走|m/s³/√Hz|

**这些不是直接对静止读数求标准差得到的四个数。**Kalibr 使用连续时间噪声模型，采样频率参与离散化。[Kalibr 噪声参数定义](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model)

读图时先掌握：

- 白噪声主导的 Allan deviation 区域，双对数图中斜率约为 `−1/2`。
- 偏置随机游走主导区域，斜率约为 `+1/2`。
- 如果没有清楚的对应区段，自动拟合结果就需要进一步审查。
- 曲线最低点对应的偏置不稳定性，不能直接当成 YAML 的 `random_walk`。

该 ROS 2 分析脚本会在输出陀螺仪 YAML 参数时转换回弧度，不能因图上显示度制而再次转换。[分析脚本单位处理](https://github.com/ripl-lab/allan_variance_ros2/blob/main/src/allan_variance_analysis.py)

此外，静态噪声模型没有包含全部运动和温度误差。Kalibr、OpenVINS 都讨论过适当增大噪声参数的需要，OpenVINS 提到过 10–20 倍的经验范围。**这不应变成对所有参数机械乘同一个倍率的固定规则**；保留原始结果，结合残差和独立数据调整。[Kalibr 实际噪声建模讨论](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model#kalibr-imu-noise-parameters-in-practice)、[OpenVINS 噪声说明](https://docs.openvins.com/gs-calibration.html)

---

**11．采集第三类数据：固定棋盘格，移动相机与飞控整体。**

开始这一阶段前，应已满足：

```
双目报告基本合理
＋
标准 /imu 数据正确
＋
噪声参数已准备
＋
时间戳链路已检查
＋
相机和飞控刚性固定
```

**这一阶段必须固定棋盘格、移动整个相机—飞控组件。**如果只移动板子而 IMU 不动，就无法让图像观测与 IMU 测到的自身运动对应。[Kalibr 联合标定采集要求](https://github.com/ethz-asl/kalibr/wiki/Camera-IMU-calibration#2-collect-images)

录包示例：

```
ros2 bag record -s sqlite3 \
  -o /home/orangepi/calibration/bags/camimu01 \
  /camera/camera/infra1/image_rect_raw \
  /camera/camera/infra2/image_rect_raw \
  /imu
```

下面是我为你的棋盘格拟定的一组动作顺序。每段长度可调整，重点是连续、清楚、有变化。

|阶段|操作|目的|
|---|---|---|
|开始数秒|静止，左右都看到完整棋盘格|留出稳定起始段|
|第一段|平滑“点头”，往复改变俯仰|激励转动|
|第二段|平滑左右侧倾，改变横滚|增加另一方向转动|
|第三段|小幅左右转头，改变偏航|增加转动变化|
|第四段|前后、左右、上下平移，改变速度|激励平移和加速度|
|后半段|自然组合平移与转动|避免动作过于单一|
|结束数秒|停稳后结束录制|避免末尾碰撞|

这里“平滑”不等于“始终匀速”。需要有变化的角速度和加速度，但不能猛烈甩动到图像模糊、IMU 裁剪或支架变形。

OpenVINS 官方指出，应避免退化运动，尽量丰富方向变化；常见有效动态片段约为 **30–60 秒**。[OpenVINS 动态标定流程](https://docs.openvins.com/gs-calibration.html)

对你目前的小棋盘格和约 100 Hz 链路，我建议先录约 60 秒有效运动，并录制至少两组独立数据，例如 `camimu01`、`camimu02`。这是针对当前设备的操作建议，不是时长越长结果越好。

采集中持续注意：

- 整个棋盘图案尽量留在左右共同视野中。
- 角点清晰，没有大片反光。
- 没有明显图案方向翻转。
- 设备不能碰桌、敲击或受到线缆拉扯。
- 相机和飞控之间没有相对晃动。

**全程静止、只做单轴旋转、只做匀速平移，都不适合作为完整的联合标定运动。**

---

**12．运行联合标定，正确理解 IMU 模型、矩阵方向和时间偏移。**

先将 `camimu01` 完整复制到笔记本，再转换：

```
rosbags-convert \
  --src /home/yummy/OpenVINS/data/calib/camimu01 \
  --dst /home/yummy/OpenVINS/data/calib/camimu01.bag
```

准备好的输入应包括：

```
camimu01.bag
stereo01-camchain.yaml
imu.yaml
target.yaml
```

在 Kalibr 容器中运行：

```
source /catkin_ws/devel/setup.bash

rosrun kalibr kalibr_calibrate_imu_camera \
  --bag /data/camimu01.bag \
  --cams /data/stereo01-camchain.yaml \
  --imu /data/imu.yaml \
  --imu-models calibrated \
  --target /data/target.yaml \
  --dont-show-report
```

当前官方源码的完整相机参数名是 `--cams`；一些 Wiki 示例写作 `--cam`。这里使用完整名称，运行前也可以用 `--help` 核对本地版本。[Kalibr 联合标定命令源码](https://github.com/ethz-asl/kalibr/blob/master/aslam_offline_calibration/kalibr/python/kalibr_calibrate_imu_camera)

几个参数必须理解清楚：

- `calibrated` 表示假定输入 IMU 已完成相应内参校正。
- 它不是声明你的 IMU 已达到理想精度。
- 默认会进行相机—IMU 时间标定。
- 不应添加 `--no-time-calibration` 来关闭你本来要估计的时间偏移。
- 默认使用已有相机链外参；先不要启用重新优化相机链的调试选项。

Kalibr 还支持 `scale-misalignment` 等 IMU 模型，用于估计更多内参。若基础模型持续出现有规律的残差，再结合数据质量和模型要求考虑，而不是把换模型当成修复时间戳的方法。[Kalibr IMU 模型说明](https://github.com/ethz-asl/kalibr/wiki/Multi-IMU-and-IMU-intrinsic-calibration)

常见输出：

```
camimu01-camchain-imucam.yaml
camimu01-imu.yaml
camimu01-results-imucam.txt
camimu01-report-imucam.pdf
```

以终端实际打印的路径为准。[联合标定输出源码](https://github.com/ethz-asl/kalibr/blob/master/aslam_offline_calibration/kalibr/python/kalibr_calibrate_imu_camera)

结果中的三个定义尤其重要：

|参数|正确含义|
|---|---|
|`cam1/T_cn_cnm1`|把 cam0 坐标转换到 cam1|
|`T_cam_imu`|把 IMU 坐标转换到相机|
|`timeshift_cam_imu`|秒，满足 `t_imu = t_cam + shift`|

这些定义来自 Kalibr 官方 YAML 文档。[Kalibr YAML 格式](https://github.com/ethz-asl/kalibr/wiki/Yaml-formats)

空间变换可以写成：

\[ p_C=R_{CI}p_I+t_{CI} \]

因此：

- `T_cam_imu` 的平移列表示 IMU 原点在相机坐标系中的坐标。
- 若要得到相机原点在 IMU 坐标系中的坐标，需要求逆变换。
- 求逆时平移为 `−Rᵀt`，不能仅把平移列变号。

这是由变换定义直接推导出的关系。

时间偏移也应按公式理解。例如假设结果为：

```
timeshift_cam_imu: 0.012
```

它表示：

```
相机时间 100.000 s
对应 IMU 时间 100.012 s
```

这是解释用的假设值，不是你的标定结果。

**该偏移不必接近零。**应关心它是否与链路相符、是否在多组数据中稳定，以及补偿后残差是否合理。本文命令估计的固定偏移，不能修复随机抖动、丢包、时间跳变或持续变化的时钟关系。

---

**13．判断结果是否可信，要同时看图像、IMU、物理尺寸和重复性。**

建议按下面的顺序检查，前一项明显错误时先修复，不要急着调整后面的优化参数。

|检查对象|看什么|异常时优先检查|
|---|---|---|
|角点提取|左右检测稳定，编号和姿态连续|角点数、反光、模糊、对称翻转|
|图像覆盖|中央、边缘及不同姿态都有观测|采集动作是否过于单一|
|重投影残差|整体较小，没有明显位置相关趋势|模型、板面、图像处理链|
|双目基线|长度接近 D430 的 50 mm|格子尺寸单位、对应关系|
|IMU 时间间隔|单调、稳定，没有大量缺口|时间戳、链路、录包负载|
|陀螺仪拟合|预测与测量随运动变化一致|时间、轴向、模糊、内参|
|加速度计拟合|预测与测量整体吻合|平移激励、噪声、基础校正|
|IMU 残差|与噪声模型大体相符，无长期系统偏差|时间、噪声、模型、安装|
|相机—IMU 位置|与实际安装距离数量级相符|变换方向、退化、支架松动|
|多组结果|同一安装下外参和时间偏移相近|某组数据质量或链路不稳定|

OpenVINS 官方特别要求检查 IMU 拟合、残差与 3σ 范围、偏置行为，并用手工测量核查最终外参。[OpenVINS 结果检查要求](https://docs.openvins.com/gs-calibration.html)

3σ 是统计范围，不要求每个样本都绝不越界；但如果长期、大面积越界，或者存在明显周期性结构，就需要查原因。也不能通过无限增大噪声，把所有错误都包进统计范围。

对于重复性，我建议：

1. 对 `camimu01`、`camimu02` 分别标定。
2. 比较同一方向定义下的旋转、平移和时间偏移。
3. 不直接平均明显冲突的结果。
4. 若结果差异较大，回头查姿态翻转、运动激励、时间戳和机械安装。
5. 再录第三组，确认问题是否重现。

我没有找到针对你这套硬件可以通用照搬的“平移必须小于几毫米、时间偏移必须小于几毫秒”官方验收值，因此不在这里编造统一阈值。

条件允许时，还可以用 Kalibr validator 在新图像中检查角点重投影及跨相机误差。该工具属于 ROS 工作流，你的 ROS 2 数据需要通过合适环境接入。[Kalibr 官方验证工具](https://github.com/ethz-asl/kalibr/wiki/Calibration-validator)

---

**14．将结果用于 OpenVINS，再用一段没有参与标定的数据验证。**

你现有 OpenVINS 配置通常涉及三类内容：

|配置|需要放入什么|
|---|---|
|主估计器配置|双目模式、相机数量、外部配置路径|
|相机—IMU 配置|左右内参、外参、时间偏移、图像话题|
|IMU 配置|实测噪声、IMU 话题、频率及对应模型参数|

不能把 EuRoC 等数据集的数值直接当作你的设备数值。

尤其注意：

- Kalibr 噪声输入 `imu.yaml` 常为平铺字段。
- OpenVINS 的 IMU 链配置包含 `imu0:` 层级及其他字段。
- 不能只重命名文件，就假定结构已经兼容。
- 使用 OpenCV FileStorage 的配置，应遵循已有示例的 `%YAML:1.0` 格式。[OpenVINS 官方 IMU 配置示例](https://github.com/rpng/open_vins/blob/master/config/euroc_mav/kalibr_imu_chain.yaml)

我核对了你当前本地源码：

- [外参解析器 (line 492)](/home/yummy/OpenVINS/vio_ws/src/open_vins/ov_core/src/utils/opencv_yaml_parse.h:492)能识别 `T_cam_imu` 与 `T_imu_cam`，在需要时求逆。
- [状态参数加载代码 (line 233)](/home/yummy/OpenVINS/vio_ws/src/open_vins/ov_msckf/src/core/VioManagerOptions.h:233)读取 `cam0` 的时间偏移，并注明当前使用共同的相机—IMU 时间偏移。

因此，**保留 Kalibr 的键名及相应矩阵方向，避免手工求逆后仍沿用旧键名。**对于你的双目链，还应检查左右时间关系是否一致，不能假定 OpenVINS 会独立补偿任意左右时差。

第一次验证，我建议先固定离线标定结果，便于判断离线参数本身的质量：

```
use_stereo: true
max_cameras: 2

calib_cam_extrinsics: false
calib_cam_intrinsics: false
calib_cam_timeoffset: false
calib_imu_intrinsics: false
```

这是我的排查建议；这些开关的含义可在官方配置中核对。待固定参数运行稳定后，再评估在线标定。[OpenVINS 官方估计器配置](https://github.com/rpng/open_vins/blob/master/config/euroc_mav/estimator_config.yaml)

另外准备一段测试数据：

1. 开始保持静止，让系统初始化。
2. 面向有纹理的普通环境，缓慢前后、左右、上下移动。
3. 加入温和转动。
4. 绕一小圈回到起点附近。
5. 停稳，观察轨迹和日志。

这段数据应与标定片段独立；不要求一直看棋盘格。

如果以后已准备好实际配置，例如：

```
/home/yummy/OpenVINS/data/calib/d430_px4/estimator_config.yaml
```

可通过显式路径启动：

```
source /home/yummy/OpenVINS/vio_ws/install/setup.bash

ros2 launch ov_msckf subscribe.launch.py \
  config_path:=/home/yummy/OpenVINS/data/calib/d430_px4/estimator_config.yaml \
  use_stereo:=true \
  max_cameras:=2 \
  rviz_enable:=true
```

当前启动文件支持 `config_path` 参数。[OpenVINS ROS 2 启动文件](https://github.com/rpng/open_vins/blob/master/ov_msckf/launch/subscribe.launch.py)

离线验证时关闭真实传感器的同名话题发布，再回放测试包，避免混入两套数据：

```
ros2 bag play /home/yummy/OpenVINS/data/calib/test01
```

重点观察：

- 初始化是否正常。
- 静止后轨迹是否明显发散。
- 平移、转动是否符合实际运动。
- 尺度是否合理。
- 特征跟踪是否稳定。
- 是否持续报告缺少 IMU、时间异常或掉帧。

回到起点附近可以帮助发现严重问题，但不能单独证明高精度。OpenVINS 本身的世界系方向也不能直接当作 PX4 的 NED；接入飞控外部视觉融合还需要单独处理坐标、参考点和消息语义。[PX4 坐标约定说明](https://docs.px4.io/v1.16/en/ros2/user_guide#ros-2-px4-frame-conventions)

---

**15．按你目前的条件，实际动手时逐项完成下面这些工作。**

1. **核对板子。**确认方格总数、内部角点和横纵实测边长；确认金属板在左右红外画面中都能清楚成像。
2. **完成 QGC 基础校准。**固定之后使用的安装方向和传感器配置。
3. **检查图像模式。**确认双路 848×480、Y8、30 Hz 实际生效，处理反光与运动模糊。
4. **检查飞控链路。**在 Orange Pi 上核对 `SensorCombined` 消息版本，再检查标准 `/imu`。
5. **检查时间。**确认 DDS 同步、采样时间语义、单调性和间隔；先排除错误时间戳。
6. **录一组短双目试验包。**通过角点提取后，再录正式双目数据并求内外参。
7. **安排长时间 IMU 静态采集。**只录 `/imu`，计算并审查噪声参数。
8. **完成相机—飞控刚性安装。**记录安装方向、粗测距离并固定线缆。
9. **录两组联合运动数据。**棋盘格固定，移动整个组件，兼顾多轴激励与图像清晰。
10. **分别求联合标定并比较。**检查报告、物理尺寸和重复性。
11. **用独立数据验证 OpenVINS。**先离线，再实时。
12. **保留可追溯信息。**保存原始包、板尺寸、图像模式、飞控与驱动版本、IMU 选择、同步方式及报告。

如果以后改变了相机与飞控的相对安装，应重新做联合空间标定；改变图像几何处理，应重新核对相机参数；改变 IMU 输出链、滤波、频率或物理传感器，应重新评估噪声和时间关系。这样才能知道每份标定结果究竟适用于哪套实际数据。