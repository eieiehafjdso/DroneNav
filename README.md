# **Ubuntu 2204+Ros2+PX4+QGC无人机仿真及真机部署自动导航项目**
## 环境准备
### 准备工作目录
```
mkdir -p ~/Desktop/DroneNav
cd ~/Desktop/DroneNav
```
### PX4-Autopilot v1.17.0和QGC v5.1.4源码获取
```
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
git checkout v1.17.0
git submodule update --init --recursive
git clone -b v5.1.4 --recursive -j8 https://github.com/mavlink/qgroundcontrol.git
```
### 下载并启动Qt 框架官方提供的在线安装器
```
curl -OL https://download.qt.io/official_releases/online_installers/qt-online-installer-linux-x64-online.run
chmod +x qt-online-installer-linux-x64-online.run
./qt-online-installer-linux-x64-online.run
```
 安装 Qt 6.11.1，并勾选了所有附加库（QGC v5.1.4编译严格要求Qt 6.11.1版本），若后续编译过程出现报错可能是ubuntu2204默认cmake版本过低导致，请确保cmake版本大于3.28另gcc和g++版本不兼容亦会导致编译失败请确保编译过程中使用正确版本
### QGC编译所需依赖安装
```
# 确保在 qgroundcontrol 源码根目录下
cd ~/Desktop/DroneNav/qgroundcontrol
# 运行官方依赖安装脚本
python3 tools/setup/install_dependencies --platform debian
#C++ 项目基础构建工具
sudo apt update
sudo apt install -y build-essential cmake ninja-build ccache
#易遗漏的系统运行库
sudo apt install -y libxcb-cursor0 speech-dispatcher libudev-dev libsdl2-dev patchelf
```
### QGC编译
```
cd ~/Desktop/DroneNav/qgroundcontrol
mkdir build && cd build
#确认Qt安装路径，若不是该路径请换为对应Qt路径
~/Qt/6.11.3/gcc_64/bin/qt-cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -j4
```
编译过程中没有报错那么恭喜你！编译成功了！可以双击进入 build 文件夹，找到一个叫 Release 的文件夹，双击进入 Release，里面那个叫 QGroundControl 的文件，就是最终编译出来的神器！双击它或者在命令行输入./Release/QGroundControl及可启动QGC地面站了！
### 若编译失败可运行下来命令清除编译产物并重新编译
```
# 回到源码根目录，删除旧的 build 文件夹
cd ~/Desktop/DroneNav/qgroundcontrol
rm -rf build
mkdir build && cd build

# 重新配置并编译
~/Qt/6.11.1/gcc_64/bin/qt-cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -j4
```
