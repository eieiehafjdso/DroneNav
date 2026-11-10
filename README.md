# Ubuntu 22.04 + ROS2 Humble + PX4 + QGC 无人机仿真及真机部署自动导航项目

> 本文档基于实测编译经历修订，所有版本号均对照源码仓库的权威配置（QGC `build-config.json`、PX4 官方文档）核实过。

## 0. 环境总览

| 组件           | 版本                        | 说明                               |
| -------------- | --------------------------- | ---------------------------------- |
| 系统           | Ubuntu 22.04 (jammy)        | —                                 |
| ROS2           | Humble                      | 需提前装好                         |
| PX4-Autopilot  | v1.17.0                     | 固件 + SITL 仿真                   |
| QGroundControl | v5.1.4                      | 地面站，需 Qt 6.11.1 编译          |
| Qt             | **6.11.1**            | QGC 官方要求 6.11.1（最低 6.11.0） |
| 仿真器         | Gazebo Harmonic (gz 8.x)    | PX4 gz 仿真目标 `gz_x500`        |
| ROS2↔PX4 桥接 | Micro-XRCE-DDS-Agent v2.4.3 | uXRCE-DDS 通信                     |

---

## 1. 前置准备（务必先做，否则后面必失败）

### 1.1 升级 cmake（QGC 要求 ≥ 3.25，Ubuntu 22.04 默认 3.22 不够）

```bash
# 添加 Kitware 官方源
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null

sudo apt update
sudo apt install cmake
cmake --version   # 应 ≥ 3.25（通常会是 3.30+/4.x）
```

### 1.2 统一 gcc/g++ 为 11（版本混搭会导致编译/链接阶段报错）

```bash
sudo apt install -y gcc-11 g++-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 100

gcc --version | head -1   # 应显示 gcc (Ubuntu 11.x)
g++ --version | head -1   # 应显示 g++ (Ubuntu 11.x)，两者版本必须一致
```

### 1.3 基础构建工具

```bash
sudo apt update
sudo apt install -y build-essential git ninja-build ccache libfuse2 \
  libxcb-cursor0 libxcb-xinerama0 libxkbcommon-x11-0 \
  libudev-dev libsdl2-dev patchelf speech-dispatcher
```

---

## 2. 准备工作目录

```bash
mkdir -p ~/Desktop/DroneNav
cd ~/Desktop/DroneNav
```

---

## 3. PX4-Autopilot v1.17.0（先编译固件，最耗时）

```bash
cd ~/Desktop/DroneNav

# 直接指定版本克隆，避免默认分支不一致
git clone -b v1.17.0 --recursive https://github.com/PX4/PX4-Autopilot.git
cd PX4-Autopilot
git submodule update --init --recursive

# 安装 PX4 依赖
#   --no-nuttx：跳过真机交叉编译工具链（只做仿真可加；真机部署请去掉该参数）
bash ./Tools/setup/ubuntu.sh --no-nuttx

# 编译 gz_x500 仿真固件（对应 Gazebo Harmonic）
make px4_sitl gz_x500
```

> **常见坑**：若编译报 `idlc` 找不到 `ddsc` / `libdds` 相关库，是 Fast DDS 库路径问题，手动指定后重编：
>
> ```bash
> export LD_LIBRARY_PATH=$(pwd)/build/px4_sitl_default/msg/idlc/lib:$LD_LIBRARY_PATH
> make px4_sitl gz_x500
> ```

> 其它 gz 机型变体（可选）：`gz_x500_vision`、`gz_x500_depth`、`gz_px4vision` 等；切换世界用 `PX4_GZ_WORLD=baylands make px4_sitl gz_x500`。查看全部目标：`make px4_sitl list_vmd_make_targets`。

---

## 4. QGC v5.1.4 编译

### 4.1 源码获取（务必指定版本，别 clone 默认分支）

```bash
cd ~/Desktop/DroneNav
git clone -b v5.1.4 --recursive -j8 https://github.com/mavlink/qgroundcontrol.git
```

### 4.2 下载并安装 Qt 6.11.1

```bash
# 国内建议用镜像加速，避免官网下载过慢/失败
curl -OL https://mirrors.ustc.edu.cn/qtproject/official_releases/online_installers/qt-online-installer-linux-x64-online.run
chmod +x qt-online-installer-linux-x64-online.run
./qt-online-installer-linux-x64-online.run --mirror https://mirrors.ustc.edu.cn/qtproject
```

安装时：选择 **Qt 6.11.1** → **Desktop gcc_64**，并勾选以下 **14 个附加模块**（QGC `build-config.json` 明确要求，缺一编译会报找不到模块）：

```
qtgraphs  qtlocation  qtpositioning  qtspeech  qtmultimedia
qtserialport  qtimageformats  qtshadertools  qtconnectivity
qtquick3d  qtsensors  qtscxml  qtwebsockets  qthttpserver
```

> 装完后确认路径是 `~/Qt/6.11.1/gcc_64/bin/qt-cmake`（**不是 6.8.x，也不是 6.11.3**）。

### 4.3 安装 QGC 依赖

```bash
cd ~/Desktop/DroneNav/qgroundcontrol
# 官方依赖安装脚本（QGC v5.1.4 内置）
python3 tools/setup/install_dependencies --platform debian
```

### 4.4 编译

```bash
cd ~/Desktop/DroneNav/qgroundcontrol
mkdir build && cd build
~/Qt/6.11.1/gcc_64/bin/qt-cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -j4
```

编译成功后，可执行文件在 `build/Release/QGroundControl`：

```bash
./Release/QGroundControl
```

### 4.5 若编译失败

```bash
cd ~/Desktop/DroneNav/qgroundcontrol
rm -rf build
mkdir build && cd build
~/Qt/6.11.1/gcc_64/bin/qt-cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -j4
```

---

## 5. Micro-XRCE-DDS-Agent v2.4.3（ROS2 ↔ PX4 桥接）

```bash
cd ~/Desktop/DroneNav

git clone --recursive -b v2.4.3 https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
cd Micro-XRCE-DDS-Agent
mkdir -p build && cd build

# 先 source ROS2，让它找到 fastcdr / fastrtps 依赖
source /opt/ros/humble/setup.bash
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)

sudo make install
sudo ldconfig /usr/local/lib/

# 出现 help 信息即安装成功
MicroXRCEAgent --help
```

---

## 6. 常见坑速查表

| 现象                                           | 原因                       | 解决                                                            |
| ---------------------------------------------- | -------------------------- | --------------------------------------------------------------- |
| QGC 报找不到 Qt6 模块                          | Qt 模块没勾全              | 重装 Qt，勾 14 个模块（见 4.2）                                 |
| `qt-cmake` 路径不存在                        | 版本号写错（6.8.3/6.11.3） | 用 `~/Qt/6.11.1/gcc_64/bin/qt-cmake`                          |
| cmake 报版本过低                               | Ubuntu 默认 3.22 < 3.25    | 按 1.1 用 Kitware 源升级                                        |
| 链接阶段报 `undefined reference` 或 ABI 错误 | gcc/g++ 版本不一致         | 按 1.2 统一到 gcc-11                                            |
| PX4 `make` 报 `idlc` 找不到 dds 库         | Fast DDS 库路径问题        | `export LD_LIBRARY_PATH=.../idlc/lib:$LD_LIBRARY_PATH` 后重编 |
| QGC 编译很慢                                   | 默认并行度低               | `ninja -j$(nproc)` 用满核                                     |
