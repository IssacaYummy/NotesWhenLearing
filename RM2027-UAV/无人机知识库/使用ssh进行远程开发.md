# 1. 在被远程的电脑上开启ssh

```bash
sudo apt update
sudo apt install openssh-server tmux
sudo systemctl enable --now ssh
systemctl status ssh --no-pager
```
![[PixPin_2026-10-24_09-49-55.png]]

看到 `active (running)`，表示 SSH 服务已启动

查看当前用户名和 IP 
```bash
whoami
hostname -I
```

检查防火墙并放行
```bash
sudo ufw status
sudo ufw allow OpenSSH
```

# 2. 连接被远程的电脑
确认你和被远程的电脑在同一个局域网，以OrangePi为例
```bash
ssh orangepi@192.168.5.109
```

首次连接会询问是否信任设备。

![[Pasted image 20261024100028.png]]

可以在 Orange Pi 上查看主机指纹：

```
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

![[PixPin_2026-10-24_10-00-44.png]]

与连接提示中的 ED25519 指纹核对后接受，再输入 **Orange Pi 用户的密码**。

出现下面的页面即为连接成功
![[Pasted image 20261024100112.png]]

# 3. 使用vscode进行远程编辑代码

搜索拓展 Remote -SSH 并安装，安装后按`Ctrl+Shift+P`，选择`Remote-SSH: Connect to Host`，添加需要远程连接的主机，输入
```bash
ssh orangepi@192.168.5.109
```
选择一个文件夹存储配置文件
![[Pasted image 20261024101105.png]]
保存后再次按`Ctrl+Shift+P`，选择`Remote-SSH: Connect to Host`，选择刚才添加的主机连接，并输入密码

连接后选择打开文件夹，打开OrangePi上面的项目目录打开即可编辑

# 4. 用tmux管理长期运行的程序

