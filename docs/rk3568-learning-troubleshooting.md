# RK3568 学习与问题排查记录

> 本文整理自学习过程中的 Word 记录，保留已经实际遇到的问题、采用的解决办法和验证结果。尚未解决或尚未验证的事项会明确标注，不将其写成确定结论。

## 1. 学习环境与工具

本文涉及的环境包括：

- Windows 主机；
- VMware 中的 Ubuntu；
- RK3568 开发板；
- MobaXterm；
- Visual Studio Code 与 Remote - SSH；
- Miniconda 与 `rknn` 虚拟环境；
- RKNN Toolkit/Runtime；
- CMake、GCC/G++ 与 AArch64 交叉编译器。

### 问题索引

| 主题 | 主要问题 | 当前状态 |
|---|---|---|
| Ubuntu SSH | 连接被拒绝 | 已解决 |
| Miniconda | 安装被中断、目录已存在 | 已解决 |
| Ubuntu 虚拟机 | 启动较慢 | 待进一步排查 |
| VS Code Remote - SSH | 扩展更新后无法连接 | 通过切换/回退版本解决 |
| 开发板连接 | 串口与 USB 设备识别 | 已连接成功 |
| 板端网络 | Wi-Fi、SSH 与文件传输 | 已验证 |
| RKNN | 环境部署、Runtime/NPU 版本 | 环境可用，版本更新待处理 |
| 摄像头 | OpenCV 依赖与取帧 | 已验证 |
| C++ 构建 | 头文件、CMake、GCC 与交叉编译器 | 已解决并完成构建 |
| 图像输入 | `int8` 与 `uint8` 类型不匹配 | 已定位并修改 |
| RKNN C API | CMake 构建、通用 API 与零拷贝流程 | 已完成学习与板端验证 |
| 实时检测 | 4 维输入、图形显示与返回值检查 | 已定位并解决 |

## 2. Ubuntu 未开启 SSH，连接被拒绝

### 问题现象

MobaXterm 连接 Ubuntu 时提示：

```text
Network error: Connection refused
```

![MobaXterm 提示 SSH 连接被拒绝](images/rk3568-troubleshooting/ssh-connection-refused.png)

### 原因

Ubuntu 尚未安装或启动 OpenSSH Server，因此远程客户端无法建立 SSH 连接。

### 解决办法

在 Ubuntu 终端执行：

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh
hostname -I
```

命令作用：

- 安装 OpenSSH Server；
- 设置 SSH 服务开机启动并立即启动；
- 检查服务状态；
- 查询 Ubuntu 的 IP 地址。

### 验证结果

使用 `hostname -I` 输出的 IP 地址重新连接，MobaXterm 可以正常进入 Ubuntu。

![SSH 连接成功后的 MobaXterm 界面](images/rk3568-troubleshooting/ssh-connection-success.png)

## 3. Miniconda 安装被 Ctrl+C 中断

### 问题现象

Miniconda 安装过程被 `Ctrl+C` 中断。再次直接运行安装脚本时，安装程序检测到目标目录已经存在，无法按照全新安装流程继续。

![Miniconda 检测到已有安装目录](images/rk3568-troubleshooting/miniconda-interrupted-install.png)

### 解决办法

使用更新参数重新执行安装脚本：

```bash
bash Miniconda3-latest-Linux-x86_64.sh -u
```

进入 Conda 基础环境：

```bash
source ~/miniconda3/bin/activate
```

激活已经创建的 RKNN 环境：

```bash
conda activate rknn
```

### 补充现象

给普通用户 `topeet` 安装 Miniconda 后，切换到该用户会自动进入 `base` 环境。原记录确认这一行为可以关闭，但没有记录具体操作命令，因此本文不补写未经本次实践验证的步骤。

## 4. Ubuntu 虚拟机启动较慢

### 现象

虚拟机启动后可能需要约两分钟，屏幕上才开始出现启动文字。

### 当前结论

原记录只记录了现象，没有给出明确原因和修复结果。遇到相同情况时，应先等待系统完成启动，不要因为短时间黑屏立即判断启动失败。

## 5. VS Code 通过 Remote - SSH 连接 Ubuntu

### 初次配置

1. 在 VS Code 安装 **Remote - SSH** 扩展；
2. 配置 Ubuntu 的 IP、用户名和 SSH 端口；
3. 确保 Ubuntu 端 SSH 服务已启动；
4. 从 VS Code 发起远程连接。

![VS Code Remote - SSH 主机配置](images/rk3568-troubleshooting/vscode-remote-ssh-config.png)

![VS Code 成功连接 Ubuntu](images/rk3568-troubleshooting/vscode-remote-ssh-connected.png)

### Remote - SSH 更新后无法连接

#### 问题现象

Remote - SSH 自动更新后，出现 `CANNOT use API proposal: terminalDataWriteEvent` 等错误。SSH 服务本身没有变化，但扩展无法正常建立远程会话。

![Remote - SSH API proposal 报错](images/rk3568-troubleshooting/remote-ssh-proposed-api-error.png)

#### 解决办法

将 Remote - SSH 从预发布版本切回发布版本，或通过扩展菜单安装以前可用的特定版本。

![将 Remote - SSH 切换为发布版本](images/rk3568-troubleshooting/remote-ssh-switch-release.png)

![Remote - SSH 与 VS Code 版本不匹配的诊断界面](images/rk3568-troubleshooting/remote-ssh-version-mismatch.png)

#### 排查原则

如果命令行 SSH 或 MobaXterm 仍能连接，而 VS Code 突然不能连接，应优先检查：

- Remote - SSH 是否刚刚自动更新；
- 当前安装的是发布版还是预发布版；
- 扩展版本是否与当前 VS Code 版本匹配。

## 6. 烧录、串口与开发板连接

### 连接线与虚拟机 USB 设备

开发板调试时同时连接：

- USB 3.0 数据线；
- 串口线。

在 VMware 中需要把对应 USB 串口设备连接给 Ubuntu 虚拟机。

![VMware 中选择 USB 串口设备](images/rk3568-troubleshooting/vmware-usb-serial-device.png)

### Windows 无法看到串口

原记录中，重启电脑后系统能够识别 `COM3`，随后串口连接成功。

![MobaXterm 通过 COM3 连接开发板](images/rk3568-troubleshooting/serial-com3-connected.png)

### 板卡连接判断

主机端工具出现设备标志，且终端能够看到开发板信息时，说明连接已经建立。

![主机识别到 RK3568 开发板](images/rk3568-troubleshooting/board-usb-detected.png)

### 关于 `rknn_server`

本次实践中，所运行的连接或示例不需要手动启动 `rknn_server`。这个结论来自当前使用场景，不应推断为所有 RKNN 工具和所有部署方式都不需要该服务。

## 7. 板端用户、Wi-Fi 与文件传输

### 切换普通用户

从 `root` 切换到普通用户：

```bash
su - topeet
```

![切换到 topeet 用户](images/rk3568-troubleshooting/switch-to-topeet-user.png)

### 连接 Wi-Fi

开启 Wi-Fi 并扫描网络：

```bash
sudo nmcli radio wifi on
nmcli device wifi list
nmcli -t -f SSID,SIGNAL,SECURITY device wifi list
```

连接指定网络：

```bash
sudo nmcli --ask device wifi connect '<SSID>' ifname wlan0
```

使用 `--ask` 可以在终端交互式输入密码，避免把密码直接写进命令历史或 Git 仓库。

### 通过 SSH 传文件

开发板或 Ubuntu 开启 SSH 后，执行：

```bash
hostname -I
```

得到 IP 地址后，可以使用 MobaXterm 的 SSH/SFTP 面板传输文件。

![MobaXterm 的 SSH 与文件传输界面](images/rk3568-troubleshooting/moba-file-transfer.png)

修改环境变量后，可执行：

```bash
source ~/.bashrc
```

### 待确认事项

原记录提出“板端是否需要配置国内软件源”的问题，但没有完成验证，因此暂不写成配置建议。

## 8. RKNN 环境与推理记录

### 环境部署

在 Ubuntu/远程环境中部署 RKNN 相关 Python 包和示例。

![RKNN 环境安装过程](images/rk3568-troubleshooting/rknn-environment-install.png)

![RKNN 环境准备完成](images/rk3568-troubleshooting/rknn-environment-ready.png)

### Runtime 与 NPU 驱动版本

检查时发现 Runtime 和 NPU 驱动版本偏低，可能影响后续模型运行。

> 状态：原记录明确写着“到目前为止还没有更新”，因此这是待处理项，不是已经解决的问题。

### 模型运行方式的实践结论

- `.rknn` 模型需要结合对应 RKNN Runtime/板端环境运行；
- 加载 PyTorch 等源模型并执行转换、量化时，可以使用工具提供的模拟运行流程进行验证；
- MobileNetV2 示例在本次实验中正常运行。

### 板端推理结果

RKNN 示例已经能够在 RK3568 目录下运行并输出推理耗时与 Top5 结果。

![RK3568 板端 RKNN 推理输出](images/rk3568-troubleshooting/rknn-inference-output.png)

截图中还出现了输入大小不一致提示，因此“程序能输出结果”不等于输入配置完全正确，后续仍需核对模型输入尺寸与实际缓冲区大小。

## 9. 摄像头与 OpenCV

### 硬件确认

插入摄像头模块并给开发板上电，先检查系统是否识别设备。

![实验使用的摄像头模块](images/rk3568-troubleshooting/camera-module.jpeg)

### Python 环境依赖

在 `rknn` 环境中安装 OpenCV：

```bash
pip install opencv-python
```

![OpenCV 安装与运行日志](images/rk3568-troubleshooting/opencv-install-log.png)

### 验证结果

通过 MobaXterm SSH 进入远程 `rknn` 环境，执行 OpenCV 取帧代码，成功获取摄像头画面。

![摄像头取帧成功](images/rk3568-troubleshooting/camera-capture-result.jpeg)

![摄像头取帧时的终端输出](images/rk3568-troubleshooting/camera-capture-terminal.png)

## 10. VS Code、CMake、编译器与头文件

### 先区分各组件职责

VS Code 的 C/C++ 扩展、CMake 和编译器不是同一个程序：

```text
CMakeLists.txt
      ↓ cmake读取配置
生成Makefile
      ↓ make调度
调用gcc/g++编译
```

C/C++ 扩展主要负责编辑器中的代码分析、补全和报错提示。它不会自动等价于真实编译器，也不一定自动理解 `CMakeLists.txt` 中的所有包含路径。

### 编辑器提示找不到外部库

需要在 `.vscode/c_cpp_properties.json` 中配置相关头文件搜索路径，让 C/C++ 扩展知道外部库的位置。

### 已配置外部库，标准头文件仍报错

#### 原因

`c_cpp_properties.json` 可以告诉扩展外部库在哪里，但系统没有安装 GCC/G++ 时，扩展仍可能无法定位 C/C++ 标准库头文件。

#### 解决办法

```bash
sudo apt install gcc g++ -y
```

![安装系统 GCC/G++ 以修复标准头文件查找](images/rk3568-troubleshooting/system-gcc-include-fix.png)

## 11. AArch64 交叉编译器解压与构建

### Windows 端无法正常解压

在 Windows 端解压交叉编译器压缩包时出现错误。

![Windows 解压交叉编译器失败](images/rk3568-troubleshooting/toolchain-windows-extract-error.png)

### 解决办法

把压缩包复制到 Ubuntu，在 Linux 中解压到目标目录：

```bash
sudo tar -xzf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.gz \
  -C /usr/local/arm64
```

![在 Ubuntu 中解压 AArch64 交叉编译器](images/rk3568-troubleshooting/toolchain-linux-extract.png)

### 缺少 CMake 和 Make

```bash
sudo apt update
sudo apt install cmake make -y
```

安装依赖并配置工具链后，`build.sh` 可以成功运行。

## 12. 使用 ADB 推送安装目录

在 Ubuntu 终端把 `install` 目录推送到开发板根目录：

```bash
adb push install /
```

![ADB 推送目录及输入类型修改记录](images/rk3568-troubleshooting/adb-push-and-uint8-fix.png)

执行前应确认开发板已通过 ADB 正常连接，并确认目标路径具有写入权限。

### `adb push` 传输失败的原因

#### 1. 命令中缺少 `push`

错误写法：

```text
adb <源路径> <目标路径>
```

ADB 会把源路径当成子命令，因此提示：

```text
unknown command
```

正确格式：

```text
adb push <本机源路径> <目标设备路径>
```

#### 2. 把绝对路径写成相对路径

错误路径：

```text
./home/topeet/...
```

路径开头的 `.` 表示当前目录，所以该写法实际指向“当前目录下的 `home/topeet`”，并不是文件系统根目录下的 `/home/topeet`。

正确的绝对路径应从 `/` 开始：

```text
/home/topeet/...
```

如果 ADB 在当前机器上找不到源文件或目录，会提示：

```text
cannot stat ... No such file or directory
```

#### 3. 没有区分本机复制与 ADB 传输

同一台机器内复制文件或目录，应使用 `cp`：

```bash
cp -a <源目录> <目标目录>
```

向 ADB 连接的另一台设备传输，应使用 `adb push`：

```text
adb push <本机源路径> <目标设备路径>
```

本次传输示例：

```bash
adb devices

adb push /home/topeet/rknn/06_yolov5_demo/02_toolkit_lite2 \
  /home/topeet/project/
```

执行前先检查源目录是否存在，以及目标设备是否在线：

```bash
ls -ld /home/topeet/rknn/06_yolov5_demo/02_toolkit_lite2
adb devices
```

核心原则：`adb push` 的第一个路径必须在执行 ADB 命令的当前机器上真实存在，第二个路径属于 ADB 连接的目标设备；Linux 绝对路径必须从 `/` 开始。

## 13. 图像输入使用 `int8` 导致数值异常

### 问题背景

普通 8 位图像的像素通常位于 `0～255`。如果将这类无符号数据直接按照有符号 `int8` 解释，其有效范围只有：

```text
-128～127
```

因此大于 127 的像素无法按原值表达，可能发生截断、溢出或错误解释，最终造成图像或推理输入异常。具体表现取决于复制、类型转换和 RKNN 输入配置方式，不能一概写成“所有值都会变成 127”。

### 本次修改

代码中将图像输入类型改为无符号 8 位：

```cpp
input_attr[0].type = RKNN_TENSOR_UINT8;
```

同时保持输出类型符合模型与后处理要求，例如截图中记录为：

```cpp
output_attr[0].type = RKNN_TENSOR_FLOAT32;
```

### 排查原则

遇到图像亮度、颜色或推理结果异常时，检查：

- 原始图像缓冲区是 `uint8_t` 还是 `int8_t`；
- RKNN 输入属性中的 tensor type 是否一致；
- 模型是否要求量化输入，以及对应的 scale/zero-point；
- 输入尺寸和 `size_with_stride` 是否匹配。

## 14. RKNN 通用 C API：从构建到板端运行

### 项目整体流程

```text
CMakeLists.txt + main.cc
        ↓
build.sh
        ├─ cmake：读取 CMakeLists.txt，生成 Makefile
        ├─ make：调用交叉编译器完成编译和链接
        └─ make install：整理可执行文件、模型和运行库
        ↓
install/example_Linux/
        ├─ example
        ├─ model/
        └─ lib/librknnrt.so
        ↓
推送到开发板并运行
```

本次交叉编译器位于：

```text
/usr/local/arm64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/
```

第三方依赖放在工程的 `3rdparty/` 目录，包括 RKNN API 和 OpenCV 的头文件与库。

### CMake、Make 与编译器的职责

```text
CMakeLists.txt → cmake → Makefile → make → gcc/g++
   构建配置       生成器              调度器   实际编译器
```

常用 CMake 指令：

| 指令 | 作用 |
|---|---|
| `set(变量 值)` | 定义路径、架构等构建变量 |
| `${变量}` | 读取变量值 |
| `include_directories(...)` | 配置编译阶段的头文件搜索路径 |
| `find_package(OpenCV)` | 查找 OpenCV 配置 |
| `add_executable(目标 源文件...)` | 定义可执行程序 |
| `target_link_libraries(...)` | 链接 RKNN Runtime、OpenCV 等库 |
| `install(...)` | 定义发布目录中的安装规则 |

编译阶段根据头文件声明检查“函数怎么用”并生成 `.o`；链接阶段在库中寻找“函数在哪里实现”。动态库 `.so` 在运行时仍需能够被加载，否则会出现：

```text
error while loading shared libraries: xxx.so
```

可以临时指定动态库目录：

```bash
export LD_LIBRARY_PATH=lib
```

更稳定的做法是在构建时设置相对运行路径：

```cmake
set(CMAKE_INSTALL_RPATH "$ORIGIN/lib")
```

其中 `$ORIGIN` 表示可执行文件所在目录。

### CMake 安装规则

`make` 负责编译和链接，`make install` 根据安装规则把产物整理到发布目录。`CMAKE_INSTALL_PREFIX` 是安装根目录，各个 `DESTINATION` 都以它为基准。

```cmake
install(TARGETS example DESTINATION ./)
install(DIRECTORY model DESTINATION ./)
install(PROGRAMS librknnrt.so DESTINATION lib)
```

三种安装对象分别对应：

- `TARGETS`：CMake 构建出的目标；
- `DIRECTORY`：已有目录及其内容；
- `PROGRAMS`：已有文件，并赋予程序权限。

### 板端运行

把整个安装目录推送到开发板，进入目录后运行：

```bash
export LD_LIBRARY_PATH=lib
./example model/RK3588/resnet18.rknn image.jpg
```

运行前可用 `file example` 检查文件格式和目标架构。当前目录下的程序要写成 `./example`；只写 `example` 时，Shell 会到 `PATH` 中查找。

### RKNN 通用 API 主线

```text
1. rknn_init                         加载模型并创建会话
2. rknn_query(IN_OUT_NUM)            查询输入输出数量
3. OpenCV 读取并预处理图片
4. rknn_inputs_set                   提交输入缓冲区
5. rknn_run                          执行推理
6. rknn_outputs_get                  获取输出
7. 后处理                            解码、TopN 或 NMS
8. rknn_outputs_release              释放输出缓冲区
9. rknn_destroy                      销毁会话
```

核心调用示意：

```cpp
rknn_context context;
rknn_init(&context, model_path, 0, 0, nullptr);

rknn_input_output_num io_num;
rknn_query(context, RKNN_QUERY_IN_OUT_NUM, &io_num, sizeof(io_num));

rknn_inputs_set(context, 1, inputs);
rknn_run(context, nullptr);
rknn_outputs_get(context, io_num.n_output, outputs, nullptr);

rknn_outputs_release(context, io_num.n_output, outputs);
rknn_destroy(context);
```

当 `rknn_init` 的模型大小参数为 `0` 时，传入的 `model_path` 按模型文件路径解释。所有 RKNN API 的返回值都应检查，初始化失败后不能继续推理。

资源释放遵循“谁分配，谁释放”：由 RKNN 分配的输出缓冲使用 `rknn_outputs_release`，会话使用 `rknn_destroy`。

### 通用 API 与零拷贝

| 项目 | 通用 API | 零拷贝 API |
|---|---|---|
| 输入内存 | 普通 CPU 内存，通过 `inputs[0].buf` 提交 | `rknn_create_mem` 创建共享内存 |
| 数据提交 | `rknn_inputs_set` | `rknn_set_io_mem` |
| 访问方式 | 结构体变量使用 `.` | 结构体指针使用 `->` |
| 输出释放 | `rknn_outputs_release` | `rknn_destroy_mem` |

零拷贝基本流程：

```text
rknn_query(INPUT_ATTR / OUTPUT_ATTR)
        ↓
rknn_create_mem(context, size_with_stride)
        ↓
memcpy(input_mem->virt_addr, image_data, ...)
        ↓
rknn_set_io_mem(context, memory, tensor_attr)
        ↓
rknn_run(context, NULL)
```

零拷贝省掉的是 CPU 内存与 NPU 可访问内存之间的一次复制，不会省掉图像预处理。申请内存时应使用 `size_with_stride`，因为它包含硬件对齐所需的填充空间。

### 模型转换链条

```text
.pt（PyTorch）
  ↓ torch.onnx.export
.onnx（通用中间格式）
  ↓ rknn-toolkit2：config → load → build → export_rknn
.rknn（板端 RKNPU 模型）
```

启用量化时：

```python
rknn.build(do_quantization=True, dataset='dataset.txt')
```

量化数据集用于确定 INT8 量化参数。`.rknn` 是否采用 INT8 不能只根据扩展名判断，应以实际转换配置和张量属性为准。

### YOLOv5 通用 API 检测流程

```text
解析 model/image 参数
→ rknn_init
→ 查询输入输出数量
→ imread、BGR→RGB、letterbox
→ rknn_inputs_set(UINT8 / NHWC)
→ rknn_run
→ rknn_outputs_get（三个检测头）
→ 解码、置信度过滤、分类别 NMS
→ 坐标映射回原图并绘制
→ 释放输出和会话
```

YOLOv5 的三个检测头负责不同尺度：

| 特征图 | stride | 主要目标尺度 |
|---|---:|---|
| `80×80` | 8 | 小目标 |
| `40×40` | 16 | 中目标 |
| `20×20` | 32 | 大目标 |

`stride` 是输入尺寸与特征图尺寸之比，不是卷积操作。使用 letterbox 时，检测框从模型输入坐标映射回原图，需要先减去补边，再除以缩放比例；不同类别的框不应互相做 NMS 抑制。

## 15. RKNN C API 与实时检测问题记录

### `Exec format error`

#### 现象与原因

执行 `bash xxx.tar.gz` 时提示：

```text
cannot execute binary file: Exec format error
```

压缩包不是 Shell 脚本或可执行程序，应解压而不是执行。

#### 解决办法

```bash
tar -xzf package.tar.gz
tar -xJf package.tar.xz
```

### `chmod -777` 导致脚本失去权限

`chmod -777 build.sh` 中的 `-` 表示移除权限，会导致脚本失去读、写和执行权限。只增加执行权限应使用：

```bash
chmod +x build.sh
```

不建议日常使用 `chmod 777`，应只授予实际需要的权限。

### SSH 运行 OpenCV 无法弹出窗口

#### 问题现象

```text
qt.qpa.xcb: could not connect to display
```

SSH 会话默认没有图形显示环境。若开发板的本地桌面正在 `:0` 显示器运行，可以尝试：

```bash
export DISPLAY=:0
```

如果没有可用的图形桌面，应改用 `cv2.imwrite` 保存结果，或通过其他方式传回主机查看。

### RKNNLite 输入维度不足

#### 问题现象

```text
The input[0] need 4dims input, but 3dims input buffer feed
```

模型需要带 batch 维度的 4 维输入，而实际只传入 `[640, 640, 3]`。推理时临时补充 batch 维度：

```python
input_data = np.expand_dims(image, 0)
outputs = rknn_lite.inference(inputs=[input_data])
```

不要直接把后续还要交给 OpenCV 绘图的 `image` 改成 4 维，否则 `cv2.cvtColor` 等接口可能因维度不符而失败。

### 未检查 `load_rknn` 或 `init_runtime` 返回值

初始化失败后继续执行，可能出现：

```text
Runtime environment is not inited, please call init_runtime to init it first!
```

每一步都应检查返回值，并在失败时立即退出：

```python
ret = rknn_lite.load_rknn(model_path)
if ret != 0:
    raise RuntimeError(f'load_rknn failed: {ret}')

ret = rknn_lite.init_runtime()
if ret != 0:
    raise RuntimeError(f'init_runtime failed: {ret}')
```

### `cv2.imwrite` 没有写入权限

程序在 `/` 等普通用户不可写目录运行时，保存图片可能提示 `permission denied`。应把结果保存到当前用户有写权限的目录，例如：

```python
cv2.imwrite('/home/topeet/result.jpg', result_image)
```

不应为了保存结果而直接使用 `sudo` 运行整个推理程序。

### ADB 找不到设备

#### 问题现象

```text
no devices/emulators found
```

ADB 使用 USB 通道，SSH 使用网络通道，两者互相独立；SSH 能连接不代表 ADB 已连接。

依次检查：

```bash
adb devices
adb kill-server
adb start-server
adb devices
```

只有状态为 `device` 才表示设备已经连接并可用；`unauthorized` 和 `offline` 都需要继续处理。若列表为空，应检查 USB 数据线、接口、板端状态，并在开发板重启后等待 ADB 服务恢复。

## 16. 快速命令索引

### SSH

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh
hostname -I
```

### Conda

```bash
bash Miniconda3-latest-Linux-x86_64.sh -u
source ~/miniconda3/bin/activate
conda activate rknn
```

### Wi-Fi

```bash
sudo nmcli radio wifi on
nmcli device wifi list
sudo nmcli --ask device wifi connect '<SSID>' ifname wlan0
```

### 编译依赖

```bash
sudo apt update
sudo apt install cmake make gcc g++ -y
```

### 交叉编译器

```bash
sudo tar -xzf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.gz \
  -C /usr/local/arm64
```

### ADB

```bash
adb devices
adb push install /
```

### RKNN 程序运行

```bash
export LD_LIBRARY_PATH=lib
./example model/RK3588/resnet18.rknn image.jpg
```

## 17. 待办事项

- 更新或核对 RKNN Runtime 与 NPU 驱动版本；
- 核对模型输入尺寸与 RKNN 输入缓冲区大小；
- 决定板端是否需要配置合适的软件源；
- 如果虚拟机持续启动缓慢，再单独排查虚拟机资源、磁盘和启动日志。
