# 飞控参数

| EKF2_EV_CTRL   | 15                                  | 同时勾选 Horizontal position、Vertical position、3D velocity、Yaw            |
| -------------- | ----------------------------------- | --------------------------------------------------------------------- |
| EKF2_HGT_REF   | 3                                   | 将主高度参考源设为双目视觉                                                         |
| EKF2_BARO_CTRL | 0                                   | 在测试 VIO + Position/Offboard 时设为 0，彻底屏蔽螺旋桨下洗气流干扰                       |
| SYS_HAS_MAG    | 0                                   | 禁用磁力计                                                                 |
| RC_MAP_OFFB_SW |                                     | **绑定遥控器上的一个二档或三档拨杆通道**（例如 Channel 6 或 Channel 7，用于随时拨杆切入/退出 Offboard） |
| COM_OBL_RC_ACT | **`0` (Position)** 或 **`3` (Land)** | 当 Offboard 心跳包丢失且遥控器在线时，自动切回定点悬停或原地降落                                 |
| COM_OBC_LOSS_T | 0.5                                 | 机载电脑信号丢失超过 0.5 秒立即触发保护                                                |
| COM_RCL_EXCEPT | `4` (Offboard)                      | 勾选 Offboard，允许不開遥控器时也能保持 Offboard 模式                                  |
|                |                                     |                                                                       |
# 编写offboard模式控制节点

# 启动服务

`px4_offboard/offboard_mission` 按飞控反馈完成：预发送设定点 → 切入 Offboard → 解锁 → 平滑起飞 → 悬停 → 航点飞行 → PX4 AUTO_LAND → 确认着地后普通上锁。先启动原有 OpenVINS 链路；本包不启动相机、VIO 或 DDS Agent。

默认起飞高度 **1 m**，按起飞机头方向飞行 **0.5 m 边长的方形**并返回起飞点，随后降落。参数位于 `config/mission.yaml`。每个航点为相对起飞点的 `[前向, 右向]` 偏移，单位米，**不是相对前一航点的增量**。设置 `fly_route: false` 表示只起飞、悬停、降落；自定义航线若不以 `[0, 0]` 结束，会在最后一个航点降落。

内部使用 PX4 本地 NED 反馈，目标高度为 `起飞点 z - takeoff_height_m`，保持起飞航向。航段采用三次平滑插值和速度前馈，配置速度为插值最大速度。到达位置容差、速度低于阈值且完成停留后才进入下一段。

## 编译

本地笔记本：

```bash

source /opt/ros/humble/setup.bash

cd /home/yummy/UAV_NAV/px4_ws

colcon build --packages-select px4_offboard --symlink-install --parallel-workers 1

source install/local_setup.bash

```

远程香橙派：

```bash

source ~/px4_ws/env.sh
cd ~/px4_ws
colcon build --packages-select px4_offboard --symlink-install --parallel-workers 1
source install/local_setup.bash

```

## 执行任务（香橙派）

先启动 OpenVINS 链路，确认 EV 融合、位置/速度有效及预飞检查正常。每次只在一台电脑运行一个任务节点，避免两端同时发布控制指令。首次无桨检查通信和模式应答，飞行时保留遥控器接管能力。

```bash

source ~/px4_ws/env.sh

ros2 run px4_offboard offboard_mission --ros-args \

--params-file ~/px4_ws/src/px4_offboard/config/mission.yaml

```

默认 `auto_start: false`，启动节点不会切模式或解锁。在另一个香橙派终端执行：

```bash

source ~/px4_ws/env.sh

ros2 service call /offboard_mission/start std_srvs/srv/Trigger '{}'

```

只有状态、位置/速度、着地反馈、VIO 新鲜度/频率和 PX4 预飞检查通过，且飞机未解锁时才接受请求。拒绝时返回原因；消除原因后重新调用。接受后自动完成整段任务，无需再发送解锁指令。以 20 Hz 发送心跳和设定点，先预发送 2 秒，再根据模式反馈申请解锁。命令以 1 Hz 重试，并有反馈超时；ACK 接受不替代状态确认。不使用强制解锁/上锁或绕过预飞检查。

任务中请求在当前位置降落：

```bash

ros2 service call /offboard_mission/land std_srvs/srv/Trigger '{}'

```

显式添加 `-p auto_start:=true` 会等待条件满足后自动执行一次任务。结束后不自动循环；下一次任务需要着地、未解锁并重新调用启动服务。

## 异常与接管

  

- 默认状态话题 `/fmu/out/vehicle_status_v1` 对应当前 `VehicleStatus.MESSAGE_VERSION=1`，可通过 `status_topic` 调整。

- 定位/VIO 失效、EKF 坐标重置、越界、跟踪误差或任务超时：已解锁时申请 `NAV_LAND`，未解锁时停止任务。`require_vio: false` 用于不使用 VIO 的 SITL/GPS 测试。

- 确认 PX4 进入 `AUTO_LAND` 后停止 Offboard 心跳/设定点，让飞控管理下降；收到持续着地反馈后才发送普通上锁。不在空中主动上锁。

- 遥控器或 PX4 切出任务模式后，停止控制和心跳，不会抢回模式或重新解锁。

- 状态断流或降落模式申请超时后停止心跳；节点退出或电脑失联后的动作由 PX4 `COM_OF_LOSS_T`、`COM_OBL_RC_ACT` 等参数决定。`Ctrl+C` 仅结束节点，不等于已完成降落；正常停止飞行应使用降落服务或遥控器。

  

`target_system/target_component` 默认 `1/1`，`source_system/source_component` 默认 `1/191`。半径、高度、跟踪误差上限和反馈超时等在 YAML 中调整。本包生成任务设定点，不替代 PX4 的碰撞规避、围栏和失联策略。

