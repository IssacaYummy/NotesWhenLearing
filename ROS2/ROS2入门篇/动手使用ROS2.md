# ROS2结点介绍
## 1.节点与节点的通信方式
- 话题-topic
- 服务-services
- 动作-Action
- 参数-parameters
![][https://fishros.com/d2lros2/humble/chapt2/get_started/1.ROS2%E8%8A%82%E7%82%B9%E4%BB%8B%E7%BB%8D/imgs/Nodes-TopicandService-16542449255392.gif]

## 2.通过命令界面查看节点信息
### ROS2命令行
### 结点相关的CLI
运行节点
```bash
ros2 run <package_name> <executable_name>
# 启动包下的某个结点
```

使用样例：
```bash
ros2 run turtlesim turtlesim_node
```

查看结点列表
```bash
ros2 node list
```

查看结点信息
```bash
ros2 node info <node_name>
```

重映射结点名称
```bash
ros2 run turtlesim turtlesim_node --ros-args --remap __node:=my_trurtle 
```

# ROS2工作空间
## 1.工作空间
工作空间是包含若干个功能包的目录，可以理解为一个文件夹，这个文件夹包含src，一般新建一个工作空间
```bash
cd delros2/chapter2/
mkdir -p chapt2_ws/src
```

## 2.功能包
功能包可以理解为存放节点的地方，功能包根据编译方式的不同可以分为三类：
- ament_python，适用于python程序
- cmake，适用于C++
- ament_cmake，适用于C++，是cmake的增强版

## 3. 功能包获取的两种方式
### 安装获取
安装一般使用
```bash
sudo apt install ros-<version>package_name
```
安装获取会自动放置到系统目录，不用再次手动source

### 手动编译获取
需要下载源码然后进行编译生成相关文件

> 当作者没来得及上传编译好的可执行文件时，需要我们自己手动编译安装
> 
> 当我们需要对包的源码进行修改，这时需要自己编译修改

手动编译后需要手动source工作空间的install目录

## 4. 与功能包相关的指令ros2pkg
```bash
creat        Creat a new ROS2 package
executables  Output a list of package specific excutables
list         Output a list of available packages
perfix       Output the perfix path of a package
xml          Output the XML of the Package manifest or a sprcific tag
```

1. 创建功能包
```bash
ros2 pkg create <package-name> --build-type {cmale,ament-cmake,ament-python} --dependencies <依赖名字>
```
2. 列出可执行文件
列出所有
```bash
ros2 pkg executables
```

列出`turtlesim`功能包的所有可执行文件
```bash
ros2 pkg excutables turtlesim
```

3. 列出所有的包
```bash
ros2 pkg list
```

4. 输出某个包所在路径的前缀
```bash
ros2 pkg perfix <package-name>
```

5. 列出包的清单描述文件
```bash
ros2 pkg xml turtlesim
```

# ROS2构建工具-Colcon
## 1.Colcon是啥
colcon是一个功能包构建工具，简单来说就是用来编译代码的
## 2.安装Colcon
一般安装ros2会自动安装，也可以自己手动安装
```bash
sudo apt-get install python3-colcon-common-extentions
```
安装完成后打开`colcon`即可看到使用方法

## 3.第一次编译
1. 创建一个工作区文件夹`colcon_test_ws`
```
cd d2lros2/chapt2/
mkdir colcon_test_ws && cd colcon_test_ws
```

2. 下载ROS2示例源码测试一下
```bash
git clone https://github.com/ros2/examples src/examples -b humble
```
3. 编译工程
```bash
colcon build
```
4. 构建完成后会看到和src同级的目录build，install和log
![[Pasted image 20260126101432.png]]

`build`：储存中间文件。对于每个包，将创建一个子文件夹，在其中调用如cmake
`install`：每个软件包安装的位置，默认情况下，每个包都将安装到单独的子目录中
`log`：包含有关每个colcon调用的各种日志信息

## 4.运行一个自己编译的结点
1. 打开终端进入刚刚创建的工作空间，先source一下资源
```bash
source install/setup.bash
```
2. 运行一个订阅者结点，我们看不到任何打印，因为没有发布者
```bash
ros2 run examples_rclcpp_minimal_subscriber subscriber_member_function
```
3. 打开新终端，运行发布者结点
```bash
source install/setup.bash
ros2 run examples_rclcpp_minimal_publisher publisher_member_function
```
![[Pasted image 20260126103215.png]]
## 5.指令
### 只编译一个包
```bash
colcon build --package-select YOUR_PKG_NAME
```

### 不编译测试单元
```bash
colcon build --package-select YOUR_PKG_NAME --cmake-args -DBUILD_TESTING=0
```

### 编译运行的包以测试
```bash
colcon test
```

### 允许通过更改src下的部分文件来改变install
每次调整python都不必重新build了
```bash
colcon build --symlink-istall
```

# 使用RCLCPP编写节点

结点需要存在于功能包中、功能包需要存在于工作空间中，所以要先创建工作空间，再创建功能包，最后创建节点

## 1.创建工作空间和功能包
### 工作空间
即创建一个文件夹
```bash
mkdir -p chapt2_ws/src/
```

### 创建一个example_cpp功能包
创建example_cpp功能包，使用ament-cmake作为编译类型，并为其添加rclcpp依赖。
```bash
ros2 pkg create example_cpp --build-type ament_cmake --dependencies rclcpp
```
- `pkg create <pkg-name>`：创建一个包
- `--build-type`：指定包的编译类型，一共三种选项`ament-python`,`ament-cmake`,`cmake`
- `--dependencies`：这个包的依赖，这里给了一个ros2的客户端接口`rclcpp`

## 2.创建节点

在`example_cpp/src`下创建一个`node_01.cpp`文件

## 3.编写代码
### 输入代码
```cpp
#include "rclcpp/rclcpp.hpp"

  

int main(int argc, char **argv)

{

// Initialize the ROS 2 client library

rclcpp::init(argc, argv);

// Create a node named "node_01"

auto node = std::make_shared<rclcpp::Node>("node_01");

// print a create message

RCLCPP_INFO(node -> get_logger(), "Node 'node_01' has been created.");

// Keep the node alive until detect ctrl-c

rclcpp::spin(node);

// Shutdown the ROS 2 client library

rclcpp::shutdown();

return 0;

}
```

### 修改 CMakeLists.txt

在`node_01.cpp`中输入内容后还需要修改CMakeLists.txt，将其添加为可执行文件，并使用`install`指令将其安装到`install`目录

在CMakeLists.txt最后一行加入下面的代码，让编译器编译node_01这个文件

```cmake
add_executable(node_01 src/node_01.cpp)
ament_target_dependencies(node_01 rclcpp)
```

接着添加
```bash
install(TARGETS

node_01

DESTINATION lib/${PROJECT_NAME}

)
```

## 4.运行编译节点
在`chapt2_ws`下依次输入命令
### 编译节点
```
colcon build
```
### source环境
```bash
source install/setup.sh
```
### 运行节点
```bash
ros2 run example_cpp node_01
```

## 5.测试
使用`ros2 node list`指令可以看到现有节点
![[Pasted image 20260126154111.png]]
# 使用RCLPY编写节点

### 1.创建python功能包

创建一个名字叫做 `example_py` python版本的功能包
```bash
ros2 pkg create example_py --build-type ament_python --dependencies rclpy
```

## 2.编写程序
编写ROS2的一般步骤

	1.导入库文件
	2.初始化客户端
	3.新增节点
	4.spin循环节点
	5.关闭客户端

在`example_py/example_py`下创建`node_02.py`，输入以下代码
```python
import rclpy
from rclpy.node import Node

def main(args=None):
'''
ROS2运行结点的入口函数
编写ROS2的一般步骤：
1.导入库文件
2.初始化客户端
3.新增节点
4.spin循环节点
5.关闭客户端
'''
rclpy.init(args=args) #初始化客户端
node = Node("node_02") #新建节点
node.get_logger().info("Hello ROS2 Node 02") #打印日志信息
rclpy.spin(node) #保持结点运行，检测是否收到退出指令（Ctrl+C）
node.shutdown() #关闭rclpy
```
保存后接着修改`setup.py`
```python
    entry_points={
        'console_scripts': [
            "node_02 = example_py.node_02:main"
        ],
    },
)
```
这段配置是声明一个ros2的节点，声明后使用`colcon build`才能检测到，从而将其添加到`install`目录下
## 3.编译运行节点
进入目录`chapt2/chapt2_ws`
### 编译节点
```bash
colcon build
```
### source环境
```bash
source install/setup.sh 
```
### 运行节点
```bash
ros2 run example_py node_02 
```

运行结果
![[Pasted image 20260126164958.png]]

### 测试
![[Pasted image 20260126165052.png]]