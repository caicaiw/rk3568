# RK3568 系统烧写、驱动安装与板端环境配置

> 本文整理自实际操作记录，覆盖系统镜像烧写、Rockchip USB 驱动安装、LVDS 屏点亮、网络与 SSH、Miniconda/RKNN 环境以及 ADB 安装。截图中的板卡为 RK3568；按键、跳线和镜像文件必须以实际开发板资料为准。

## 1. 基本概念与操作环境

### 什么是烧写

烧写是把系统镜像写入开发板的 eMMC 等存储芯片。系统镜像不是普通文件的简单复制，而是一份可用于还原开发板系统分区和内容的完整镜像。

本次操作使用：

- Windows 主机；
- RK3568 开发板；
- Rockchip 烧录工具；
- `update.img` 系统镜像；
- DriverAssistant 5.1.1；
- USB 数据线与串口线；
- MobaXterm。

烧写前应确认镜像与具体板卡、存储配置和屏幕版本匹配，并保证供电稳定。烧写会覆盖板端原有系统，重要数据需要提前备份。

## 2. 准备镜像并进入下载模式

把对应版本的 `update.img` 下载到 Windows，并放到烧录工具使用的目录中。

![烧录工具目录中的 update.img](images/rk3568-flashing-setup/update-image-folder.png)

本次开发板通过以下方式进入下载模式：

1. 关闭开发板电源；
2. 按住音量 `+` 键；
3. 保持按键并给开发板上电；
4. 用 USB 数据线连接开发板与 Windows 主机。

> 不同板卡可能使用 Recovery、Maskrom 或其他按键组合，应优先查阅对应开发板手册。

串口中可以观察开发板的启动或下载模式日志。

![串口启动日志](images/rk3568-flashing-setup/serial-boot-log.png)

## 3. 烧录工具无法发现设备

### 问题现象

开发板已经进入下载模式，但烧录工具没有显示设备。

![烧录工具未发现设备](images/rk3568-flashing-setup/flashing-tool-no-device.png)

此时 Windows 设备管理器中出现 `USB download gadget`，说明开发板的 USB 下载设备已经枚举，但主机缺少可用的 Rockchip USB 驱动。

![设备管理器中的 USB download gadget](images/rk3568-flashing-setup/device-manager-usb-download-gadget.png)

### 安装 Rockchip USB 驱动

1. 下载与工具包配套的 `DriverAssistant_v5.1.1.rar`；
2. 解压压缩包；
3. 右键选择“以管理员身份运行” `DriverInstall.exe`；
4. 在驱动助手中点击“驱动安装”；
5. Windows 弹出驱动安装确认时，确认发布者为 Rockchip 后继续安装；
6. 等待驱动助手提示安装成功。

![下载 DriverAssistant](images/rk3568-flashing-setup/driver-assistant-download.png)

![解压后的驱动助手文件](images/rk3568-flashing-setup/driver-assistant-files.png)

![DriverAssistant 驱动安装界面](images/rk3568-flashing-setup/driver-assistant-window.png)

![Windows 驱动安装确认](images/rk3568-flashing-setup/windows-driver-confirmation.png)

![驱动安装成功](images/rk3568-flashing-setup/driver-install-success.png)

### 验证结果

驱动安装后，设备管理器能够识别 Rockchip 下载设备，烧录工具也会显示 Loader 设备。

![设备管理器识别 RockUSB 设备](images/rk3568-flashing-setup/device-manager-rockusb-device.png)

![烧录工具识别到 Loader 设备](images/rk3568-flashing-setup/flashing-tool-device-detected.png)

如果仍然无法识别，应依次检查：

- USB 线是否支持数据传输；
- 开发板是否确实进入下载模式；
- 是否使用了稳定的 USB 接口，必要时避免未经供电的 USB Hub；
- 设备管理器中是否仍有带黄色感叹号的未知设备；
- 驱动安装程序是否以管理员身份运行。

## 4. 执行烧写并确认启动

在烧录工具中确认镜像路径和设备状态后开始烧写。烧写过程中不要拔掉 USB 线或切断电源。

![系统镜像烧写过程](images/rk3568-flashing-setup/flashing-progress.png)

烧写完成后，开发板自动重启。串口进入 Ubuntu 登录界面并能够登录，说明系统已经启动。

![烧写完成后的首次启动](images/rk3568-flashing-setup/system-first-boot.png)

## 5. LVDS 屏已连接但不亮

### 问题现象

系统日志显示 LVDS/显示相关设备已连接，但屏幕没有点亮。检查日志时可使用：

```bash
uname -a
cat /etc/os-release
ls /sys/class/drm/
dmesg | grep -Ei 'lvds|panel|backlight|display'
```

日志中出现背光供电相关提示，说明除了数据链路，还需要检查背光供电。

![LVDS 背光相关日志](images/rk3568-flashing-setup/lvds-backlight-dmesg.png)

### 本次处理结果

根据板卡原理图检查 LVDS 背光供电连接。

![LVDS 背光供电原理图](images/rk3568-flashing-setup/lvds-backlight-schematic.png)

本次板卡需要连接 5 V 跳线。连接 J38 跳线后，屏幕成功点亮。

![连接 J38 后 LVDS 屏点亮](images/rk3568-flashing-setup/lvds-screen-lit.jpeg)

> J38 是本次板卡的实际结果，不代表所有 RK3568 开发板都使用相同跳线。操作前应核对原理图，避免短接错误电源。

## 6. 配置网络与 SSH

启用 SSH 前，先让开发板通过网线或 Wi-Fi 接入网络。使用 NetworkManager 时可以执行：

```bash
sudo nmcli radio wifi on
nmcli device wifi list
sudo nmcli --ask device wifi connect '<SSID>' ifname wlan0
```

使用 `--ask` 交互输入密码，避免把 Wi-Fi 密码写入命令历史或仓库。

![扫描并连接 Wi-Fi](images/rk3568-flashing-setup/wifi-scan-and-connect.png)

安装并启动 SSH 服务：

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh
hostname -I
```

![安装 SSH 并检查服务状态](images/rk3568-flashing-setup/openssh-install-and-status.png)

使用 `hostname -I` 显示的地址建立 SSH 连接。

![SSH 连接开发板](images/rk3568-flashing-setup/ssh-session.png)

## 7. SFTP 上传提示权限不足

### 原因

本次问题中，目标目录由 `root` 创建，而 SSH/SFTP 使用 `topeet` 登录，因此普通用户没有写权限。

![SFTP 上传时提示权限不足](images/rk3568-flashing-setup/sftp-permission-error.png)

### 推荐处理

优先在 `topeet` 自己的主目录中创建项目目录：

```bash
su - topeet
mkdir -p ~/project
```

如果目录必须由管理员预先创建，可把所有者改为实际登录用户：

```bash
sudo mkdir -p /home/topeet/project
sudo chown -R topeet:topeet /home/topeet/project
```

检查目录权限：

```bash
ls -ld /home/topeet/project
```

![检查项目目录权限](images/rk3568-flashing-setup/project-directory-permissions.png)

不建议使用 `chmod 777` 作为常规解决办法；修正目录所有者通常更符合最小权限原则。

## 8. 安装 Miniconda 并创建 RKNN 环境

本次板端系统为 AArch64，因此使用 AArch64 版 Miniconda 安装脚本。先把安装脚本上传到开发板，再赋予执行权限：

```bash
chmod +x Miniconda3-latest-Linux-aarch64.sh
./Miniconda3-latest-Linux-aarch64.sh
```

![上传并启动 Miniconda 安装脚本](images/rk3568-flashing-setup/miniconda-installer-upload.png)

按照安装程序提示阅读并接受许可协议，选择安装位置，并按需初始化 shell。

![Miniconda 安装过程](images/rk3568-flashing-setup/miniconda-installation.png)

重新打开 shell 后，创建并激活 Python 3.8 环境：

```bash
conda create -n rknn python=3.8
conda activate rknn
```

![创建并进入 rknn 环境](images/rk3568-flashing-setup/conda-rknn-environment.png)

## 9. 安装 RKNN Toolkit Lite 2

确认 wheel 文件与 Python 版本、CPU 架构匹配。本次记录使用 CPython 3.8、AArch64 对应的 wheel：

```bash
pip install ./rknn_toolkit_lite2-2.3.2-cp38-cp38-manylinux_2_17_aarch64.manylinux2014_aarch64.whl \
  -i https://pypi.mirrors.ustc.edu.cn/simple/
```

镜像地址必须包含完整的 `/simple/` 路径。只填写不完整的域名时，`pip` 可能无法找到 `numpy` 等依赖。

![镜像地址不完整导致依赖下载失败](images/rk3568-flashing-setup/rknn-pip-mirror-error.png)

安装完成后使用以下命令核对包：

```bash
pip list
```

本次记录中 `rknn-toolkit-lite2 2.3.2` 安装成功。

![RKNN Toolkit Lite 2 安装成功](images/rk3568-flashing-setup/rknn-toolkit-install-success.png)

## 10. 安装 ADB 时 APT 一直等待锁

安装 ADB 时出现类似信息：

```text
正在等待缓存锁：无法获得锁 /var/lib/dpkg/lock-frontend
锁正由进程 unattended-upgr 持有
```

这是系统后台的 `unattended-upgrades` 正在安装更新。APT 为避免两个进程同时修改软件包数据库，会让当前命令等待。

![APT 等待 unattended-upgrades 释放锁](images/rk3568-flashing-setup/apt-lock-waiting.png)

本次处理是等待后台更新完成，不手工删除锁文件。随后 ADB 正常安装完成。

![ADB 安装完成](images/rk3568-flashing-setup/adb-install-success.png)

![ADB 设备连接测试](images/rk3568-flashing-setup/adb-device-test.jpeg)

## 11. 快速检查清单

### 烧写前

- 镜像与开发板型号、存储和屏幕配置匹配；
- 已备份板端重要数据；
- USB 数据线可靠，供电稳定；
- 已准备烧录工具和 Rockchip USB 驱动。

### 烧写工具不识别设备

- 确认进入下载模式；
- 查看设备管理器是否出现 `USB download gadget` 或未知设备；
- 以管理员身份安装 DriverAssistant；
- 驱动完成后重新插拔 USB，并重新进入下载模式。

### 烧写后

- 通过串口确认 U-Boot 和 Linux 正常启动；
- 检查屏幕数据线与背光供电；
- 配置网络后再安装和启用 SSH；
- SFTP 上传失败时先检查目录所有者和权限；
- 安装 Python wheel 前核对 Python ABI、系统架构和软件版本。
