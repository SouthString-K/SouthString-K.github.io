# 浙江省机器人大赛：树莓派使用

这一篇主要整理我在准备浙江省机器人大赛过程中，和树莓派相关的实际内容。

在机器人项目里，树莓派经常作为一个轻量级控制与部署平台使用。无论是运行 Python 脚本、接相机、接串口，还是远程调试，都会频繁和它打交道。

## 常见准备工作

如果是新拿到的树莓派，通常需要先完成下面几步：

1. 烧录系统镜像
2. 配置网络
3. 开启 SSH 或 VNC
4. 确认 IP 地址
5. 远程连接调试

## 一些常见命令

下面这些指令在使用树莓派时非常常见。

### 查看当前 IP 地址

```bash
hostname -I
```

这是我最常用的方法，会直接输出当前树莓派在局域网中的 IP 地址。

也可以用：

```bash
ip addr
```

如果系统里安装了 `net-tools`，还可以使用：

```bash
ifconfig
```

### 查看主机名

```bash
hostname
```

很多时候局域网里也可以通过主机名访问，比如：

```bash
ping raspberrypi.local
```

### 更新软件源

```bash
sudo apt update
sudo apt upgrade -y
```

### 查看系统版本

```bash
cat /etc/os-release
```

### 查看磁盘空间

```bash
df -h
```

### 查看内存使用情况

```bash
free -h
```

### 查看连接的 USB 设备

```bash
lsusb
```

### 查看摄像头设备

```bash
ls /dev/video*
```

### 查看串口设备

```bash
ls /dev/tty*
```

## SSH 远程连接

如果树莓派和电脑处于同一局域网，可以通过 SSH 远程连接。

### 开启 SSH

在图形界面中可以通过 `Raspberry Pi Configuration` 打开 SSH。

如果使用终端，也可以输入：

```bash
sudo raspi-config
```

然后在菜单中启用 SSH。

### 电脑端连接方式

Windows 的 PowerShell 或终端中输入：

```bash
ssh pi@192.168.1.100
```

这里：

- `pi` 是用户名
- `192.168.1.100` 是树莓派 IP

第一次连接时会提示是否信任主机，输入：

```bash
yes
```

然后输入密码即可。

## VNC 远程连接

如果你需要看到树莓派桌面界面，而不是只用命令行，那么 VNC 是非常方便的方式。

### 开启 VNC

同样可以先输入：

```bash
sudo raspi-config
```

然后在 `Interface Options` 里启用 VNC。

### 电脑端连接方式

在电脑上安装 `RealVNC Viewer`，然后输入树莓派的 IP 地址，例如：

```text
192.168.1.100
```

输入树莓派用户名和密码后，即可看到桌面。

## 文件传输

有时候我们需要把本地代码、模型文件、图片数据集传到树莓派上。常见方法有：

### 使用 `scp`

把本地文件复制到树莓派：

```bash
scp test.py pi@192.168.1.100:/home/pi/
```

把整个文件夹复制过去：

```bash
scp -r yolov5 pi@192.168.1.100:/home/pi/
```

从树莓派复制回本地：

```bash
scp pi@192.168.1.100:/home/pi/output.txt .
```

## 树莓派部署时常见操作

### 新建 Python 虚拟环境

```bash
python3 -m venv robot_env
source robot_env/bin/activate
```

### 安装依赖

```bash
pip install -r requirements.txt
```

### 后台运行程序

如果你不希望关掉终端后程序停止，可以使用：

```bash
nohup python3 main.py > run.log 2>&1 &
```

查看后台进程：

```bash
ps aux | grep python
```

结束进程：

```bash
kill -9 进程号
```

### 开机自启

有些比赛项目需要上电自动运行，这时候可以考虑使用 `systemd` 服务进行管理。基本思路是：

1. 在 `/etc/systemd/system/` 下写一个服务文件
2. 重新加载服务
3. 设置开机启动

常见指令如下：

```bash
sudo systemctl daemon-reload
sudo systemctl enable my_robot.service
sudo systemctl start my_robot.service
sudo systemctl status my_robot.service
```

## 一些经验

树莓派部署最容易出问题的地方通常是：

- IP 地址变化
- 依赖版本不一致
- 权限问题
- 摄像头或串口设备名变化
- 电源供电不足

所以在比赛前，最好把这些内容都提前排查一遍。

## 最后

树莓派解决的是“系统怎么部署和运行”的问题。比赛现场很多时候并不是算法写不出来，而是设备联不上、程序跑不稳、远程调试不顺利。

所以这部分基础工作越扎实，后面的算法部署就会越顺利。
