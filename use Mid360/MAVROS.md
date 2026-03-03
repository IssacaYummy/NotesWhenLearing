首先分别开两个窗口启动livox驱动和slam算法

启动livox驱动

```bash
cd livox_ws/
source install/setup.bash
ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

启动slam算法（这里以fast-lio2）为例

```bash
cd livox_ws/
source install/setup.bash 
ros2 launch fast_lio mapping.launch.py
```

然后启动MAVROS通信

首先查看连接飞控的串口是哪个

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

![[Pasted image 20260227121939.png]]、

我这里是/dev/ttyACM0

然后在地面站查看飞控中设置的串口的波特率

在参数列表中搜索SERIAL，可以看到各个串口对应的物理接口，协议，以及波特率等信息。连接飞控的串口为SERIAL0，波特率57600。

使用下面的命令启动MAVROS，要注意后面的参数，串口号和波特率要对应

```bash
ros2 run mavros mavros_node --ros-args -p fcu_url:=serial:///dev/ttyACM0:57600
```

最后新开窗口启动slam_to_vision节点

```bash
cd livox_ws/
source install/setup.bash 
ros2 run slam_to_vision_cpp slam_to_vision_node
```