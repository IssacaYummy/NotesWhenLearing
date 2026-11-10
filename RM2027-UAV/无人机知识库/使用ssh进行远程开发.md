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

## 免密登陆远程设备

首先查看本地有没有ssh-key

```bash
# 检查本地是否已有公钥
ls ~/.ssh/id_*.pub
```

如果没有的话就执行命令生成，可以一直按回车保持默认，有的话跳过这一步

```bash
ssh-keygen -t ed25519
```

将该电脑的公钥追加到被远程的电脑

```bash
ssh-copy-id 用户名@机载IP
```

然后可以使用下面的命令免密登陆

```bash
ssh "用户名@机载IP"
```
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

# 5. ssh 常用命令

## 使用scp
### 1. 复制单个文件到本地

```bash
scp username@remote_host:/path/to/remote/file.txt /path/to/local/dir/
```

### 2. 复制整个文件夹到本地

```bash
scp -r username@remote_host:/path/to/remote/folder /path/to/local/dir/
```
### 3. 上传单个文件

上传制定文件

```bash
scp /path/to/local/file.txt username@remote_host:/path/to/remote/dir/
```

**上传当前目录下的文件**：

```bash
scp file.txt username@remote_host:/home/username/
```
### 4. 上传整个文件夹（使用 `-r` 递归）

```bash
scp -r /path/to/local/folder username@remote_host:/path/to/remote/dir/
```

## 使用 `rsync`

对于大文件、大量文件或不稳定的网络连接，`rsync` 远优于 `scp`。

### 1. 复制单个文件到本地

```bash
rsync -avzP username@remote_host:/path/to/remote/file.txt /path/to/local/dir/
```

### 2. 复制文件夹到本地

```bash
rsync -avzP username@remote_host:/path/to/remote/folder /path/to/local/dir/

rsync -avzP orangepi@192.168.5.109:/home/orangepi/bags/ /home/yummy/UAV_NAV/data/bags/test-bags/raw_data/
```

### 3. 上传单个文件

```bash
rsync -avzP /path/to/local/file.txt username@remote_host:/path/to/remote/dir/
```

### 4. 上传文件夹

```bash
rsync -avzP /path/to/local/folder username@remote_host:/path/to/remote/dir/
```

 **注意末尾斜杠 `/` 的区别**：
 - `folder`（不带斜杠）：会在本地目标路径下创建名为 `folder` 的文件夹。
- `folder/`（带斜杠）：只会把该目录下的**所有内容**复制过去，不会在本地多套一层 `folder` 目录名。     

常用选项说明
- `-a`：归档模式，保留文件权限、属主、修改时间、软链接等属性。
- `-v`：输出详细传输日志（verbose）。  
- `-z`：传输时启用压缩，加速网络传输。
- `-P`：包含 `--progress`（显示进度条）和 `--partial`（保留断点文件，支持**断点续传**