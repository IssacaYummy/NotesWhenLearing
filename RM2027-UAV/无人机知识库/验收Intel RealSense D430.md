以ubuntu22.04系统为例
# 安装librealsense
参考realsense[官方github仓库](https://github.com/realsenseai/librealsense/blob/master/doc/distribution_linux.md) ，执行下面的命令安装SDK：
 ```bash
 # Ensure the directory exists
sudo mkdir -p /etc/apt/keyrings

# Download and dearmor
curl -sSf https://librealsense.realsenseai.com/Debian/librealsenseai.asc | \
gpg --dearmor | sudo tee /etc/apt/keyrings/librealsenseai.gpg > /dev/null

# Make sure apt HTTPS support is installed
udo apt-get install apt-transport-https

# Add the server to the list of repositories
echo "deb [signed-by=/etc/apt/keyrings/librealsenseai.gpg] https://librealsense.realsenseai.com/Debian/apt-repo `lsb_release -cs` main" | \
sudo tee /etc/apt/sources.list.d/librealsense.list
sudo apt-get update

# Install the libraries
sudo apt-get install librealsense2-dkms
sudo apt-get install librealsense2-utils

# Optionally install the developer and debug packages
sudo apt-get install librealsense2-dev
sudo apt-get install librealsense2-dbg
 ```


# 使用realsense-viewer查看图像

使用USB连接相机，**注意一定要使用USB3.0协议**。可以通过下面的命令查看
```bash
lsusb
lsusb -t
```
如果对应设备的速率为480M，则为USB2.0；如果为5000M/10000M，则为USB3.0
USB3.0
![[Pasted image 20260927154935.png]]
USB2.0
![[Pasted image 20260927155003.png]]

也可以通过realsense-viewer查看。

在终端输入命令
```bash
realsense-viewer
```

进入realsense-viewer后可以看到连接的设备和协议
![[Pasted image 20260927155252.png]]

将module打开后可以在右侧看到2d和3d图像
![[Pasted image 20260927155735.png]]
![[Pasted image 20260927155754.png]]