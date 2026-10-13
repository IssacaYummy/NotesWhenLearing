# Ceres

Ceres求解最小二乘问题的一般形式如下（带边界的核函数最小二乘）：

$$\operatorname*{min}_{x}\mathrm{~\frac{1}{2}\sum_{i}\rho_{i}\left(||f_{i}\,(x_{i_{1}},\cdot\cdot\cdot x_{i_{n}})||^{2}\right)}
\mathrm\\{S.t.}\,\,\,\,\,\,\,\,\,l_{j}\ll x_{j}\ll\,u_{j}.$$
- $x_1, ... , x_n$为优化变量，又称参数块（Paramater blocks）
- $f_i$ 称为代价函数(Cost function)，也称残差块（Residual blocks），可以理解为误差项
- $l_j$ 和 $u_j$ 为第j个优化变量的上限和下限，在最简单的情况下，取$l_j = -\infty$ , $u_j = \infty$ 
- 目标函数由许多平方项经过一个核函数 $\rho ( )$ 之后求和组成，在简单情况下取 $\rho$ 为恒等函数

为了让Ceres帮我们求解最小二乘问题，需要做以下操作：
1. 定义每个参数块。通常为平凡的向量，在SLAM中也常定义为四元数、李代数等。如果是向量，我们需要为每个参数块分配一个double数组，来存储变量的值。
2. 定义残差块的计算方式。残差块通常关联若干参数块，对他们进行一些自定义的计算，然后返回残差值。Ceres对他们求平方之后，作为目标函数的值
3. 残差块往往也需要定义雅可比的计算方式。在Ceres中，可以使用自动求导功能，也可以手动指定雅可比的计算过程。
4. 最后将所有的参数块和残差块加入Ceres定义的Problem对象中，调用Solve函数求解即可。求解前我们可以传入一些配置信息，例如迭代次数、终止条件等等，也可以使用默认的配置

执行下面的命令安装ceres：
```bash
sudo apt install -y libceres-dev
```
# g2o

g2o全称General Graphic Optimization(G^2 O)，是一个基于图优化的库。图优化是一种将非线性优化和图论结合起来的理论

## 图优化理论

图优化是把优化问题表现成图（Graph）的一种方式。这里的图是图论意义上的图。

一个图由若干个顶点（Vertex）以及连接着这些顶点的辩（Edge）组成。进而，用顶点表示优化变量，用边表示误差项，对于任意一个上述形式的非线性最小二乘问题，我们可以构建一个与之对应的一个图，也成为贝叶斯图或因子图

g2o是一个通用的图优化库，意味着能够在g2o中求解任何能够表示为图优化的最小二乘问题

一般从源码安装g2o库。

首先安装依赖：
```bash
sudo apt update
sudo apt install -y libsuitesparse-dev qtbase5-dev libqglviewer-dev-qt5 
```

下载g2o源码，通常放在3rdparty目录下，-b参数可以选择分支，这里安装的是 20200410_git 版本分支：
```bash
cd /home/yummy/Coding\&Notes/Code/slambook2_learn/3rdparty
git clone -b 20200410_git https://github.com/RainerKuemmerle/g2o.git
```

编译并安装到系统目录：
```bash
cd g2o

mkdir build && cd build

cmake .. -DCMAKE_BUILD_TYPE=Release

make -j$(nproc)

sudo make install

sudo ldconfig
```

> 编译可能很耗性能和时间


## 使用g2o拟合曲线

为了使用g2o，首先要将曲线拟合问题抽象成图优化。在这个过程中，只需要记住节点为优化变量，边为误差项。对于函数$y = \exp{a x^2 + b x + c}$的图优化问题可以变为如下形式

![[PixPin_2026-10-13_16-10-32.png]]

在曲线拟合问题中，整个问题只有一个点：曲线模型的参数a，b，c

各个带噪点的数据点构成了一个个误差项，也就是图优化的边，他们是一元边（Unary Edge），即只连接一个顶点

使用g2o包括下面几个步骤：
1. 定义顶点和边的类型
2. 构建图
3. 选择优化算法
4. 调用g2o进行优化，返回结果


