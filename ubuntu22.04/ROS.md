从Ubuntu22开始，仅能安装Ros2，没有Ros1的apt安装方法，因为官方并没有做适配。
[Ros官网](https://zhuanlan.zhihu.com/p/688413327/wiki.ros.org)中有关于Ros1源码安装攻略，但仅适用于对应的Ubuntu版本，如果在Ubuntu22上安装本应在Ubuntu20上安装的Ros1，需要结合官方源码安装方法进行一定的修改。

Ubuntu20的最新内核版本仅为5.15，内核版本在2024年来说有一点低了，可能无法满足2023年以及往后的新硬件的使用。
本文内容参考自[Ros官方源码安装](https://link.zhihu.com/?target=https%3A//wiki.ros.org/noetic/Installation/Source)以及Jean-Guillaume Durand编写的[安装方法](https://link.zhihu.com/?target=https%3A//medium.com/@jean.guillaume.durand/installing-ros-noetic-on-ubuntu-22-04-1678e9dab1f5)，同时加上了我自己的尝试，最终可以在Ubuntu22上面运行Ros Noetic并进行编译。

### 添加Ros2源

按照[Ros2 Humble](https://zhida.zhihu.com/search?content_id=241111139&content_type=Article&match_order=1&q=Ros2+Humble&zhida_source=entity)的[官方安装说明](https://link.zhihu.com/?target=http%3A//docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)添加Ros2的官方源。

### 添加ROS2 GPG密钥

```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```

### 添加官方仓库

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

国内可以考虑使用镜像，例如使用中科大的Ros镜像。

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://mirrors.ustc.edu.cn/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

### 安装引导程序依赖项

主要是rosdep、vcstools等一系列用于源码安装的工具。

```bash
sudo apt-get install python3-rosdep python3-rosinstall-generator python3-vcstools python3-vcstool build-essential
```

初始化rosdep。

```bash
sudo rosdep init
```

**此处与官方方式有所不同**，如果按照官方更新方式`rosdep update`，必定会在后续过程中出现下面的错误：

```bash
# rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y
ERROR: the following packages/stacks could not have their rosdep keys resolved to system dependencies:
diagnostic_common_diagnostics: [hddtemp] defined as "not available" for OS version [*]
```

我们需要手动安装hddtemp包，同时修改rosdep初始化中的文件信息以解决问题。

### 安装hddtemp

```bash
cd ~/Downloads
wget http://archive.ubuntu.com/ubuntu/pool/universe/h/hddtemp/hddtemp_0.3-beta15-53_amd64.deb
sudo apt install ~/Downloads/hddtemp_0.3-beta15-53_amd64.deb
```

### 修改rosdep初始化文件

下载base.yaml文件。

```bash
cd ~/Downloads
wget https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml
```

打开base.yaml文件，Ubuntu22可以使用gedit修改。

```bash
gedit ~/Downloads/base.yaml
```

搜索hddtemp，在ubuntu那一部分，添加上ubuntu22（最后一行，jammy那一行）的内容，如下所示：

```yaml
hddtemp:
  arch: [hddtemp]
  debian: [hddtemp]
  fedora: [hddtemp]
  freebsd: [python27]
  gentoo: [app-admin/hddtemp]
  macports: [python27]
  nixos: [hddtemp]
  openembedded: [hddtemp@meta-oe]
  opensuse: [hddtemp]
  rhel: [hddtemp]
  slackware: [hddtemp]
  ubuntu:
    '*': null
    bionic: [hddtemp]
    focal: [hddtemp]
    jammy: [hddtemp]
```

### 修改20-default.list文件内容

修改`Rosdep init`创建的20-default.list文件内容，使其读取**本地的base.yaml文件**。

```bash
sudo gedit /etc/ros/rosdep/sources.list.d/20-default.list
```

修改为如下内容：

```yaml
# os-specific listings first
yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/osx-homebrew.yaml osx

# generic
# yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yaml
# 确保下面的base文件路径和你的保存位置一致，前缀file://即为读取本地文件，/home/your_usernme/需要根据你的设置进行修改
# 也可以移动base.yaml文件到你需要的地方，保证下面的位置是你移动的位置即可
yaml file:///home/your_username/Downloads/base.yaml
yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yaml
yaml https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/ruby.yaml
gbpdistro https://raw.githubusercontent.com/ros/rosdistro/master/releases/fuerte.yaml fuerte

# newer distributions (Groovy, Hydro, ...) must not be listed anymore, they are being fetched from the rosdistro index.yaml instead
```

### 更新rosdep

```bash
rosdep update
```

可能会需要一定时间（需要连接到Github），可以考虑修改hosts文件，抑或其他帮助连接Github的方法，具体请自行查阅。

### 正式安装

### 创建catkin工作区

主要用于存放安装Ros的源码工作区。

```bash
mkdir ~/ros_catkin_ws
cd ~/ros_catkin_ws
```

使用vcstools下载Ros包，安装Desktop版本

```bash
rosinstall_generator desktop --rosdistro noetic --deps --tar > noetic-desktop.rosinstall
mkdir ./src
vcs import --input noetic-desktop.rosinstall ./src
```

个人更推荐安装Desktop full版本，更全面一点，代码如下：

```bash
rosinstall_generator desktop_full --rosdistro noetic --deps --tar > noetic-desktop.rosinstall
mkdir ./src
vcs import --input noetic-desktop.rosinstall ./src
```

### 解决依赖问题

安装Ros包可能有相互依赖的情况，如果缺少系统依赖则无法使用，需要使用rosdep工具自动检测并安装相应的依赖文件。

```bash
rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y
```

### 解决兼容问题

构建[Ros1 Noetic](https://zhida.zhihu.com/search?content_id=241111139&content_type=Article&match_order=1&q=Ros1+Noetic&zhida_source=entity)之前，我们需要手动修补src文件夹中的两个包，以兼容[Ubuntu22.04](https://zhida.zhihu.com/search?content_id=241111139&content_type=Article&match_order=1&q=Ubuntu22.04&zhida_source=entity)：

- rosconsole：Ros的控制台日志记录使用程序
- urdf：统一机器人描述格式（URDF）文件的解析器

相关的修补程序在Github上有开源实现，作者[Daniel Reuter](https://link.zhihu.com/?target=https%3A//github.com/dreuter)，参考链接：

- [GitHub — dreuter/rosconsole](https://link.zhihu.com/?target=https%3A//github.com/dreuter/rosconsole/tree/noetic-jammy)
- [GitHub — dreuter/urdf](https://link.zhihu.com/?target=https%3A//github.com/dreuter/urdf/tree/set-cxx-version)

使用Github上的修改包直接替换初始源码包。

```bash
cd ~/ros_catkin_ws

# --- 修补 rosconsole ---
rm -rf src/rosconsole
git clone https://github.com/dreuter/rosconsole.git src/rosconsole
cd src/rosconsole
git checkout noetic-jammy
cd ../..

# --- 修补 urdf ---
rm -rf src/urdf
git clone https://github.com/dreuter/urdf.git src/urdf
cd src/urdf
git checkout set-cxx-version
cd ../..
```

### 构建Ros1 Noetic

使用`catkin_make_isolated`命令构建Ros1 Noetic，并安装

```bash
./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
```

Ros1将会自动安装到所设置的catkin工作区中的install_isolated文件夹中，更新后即可正常使用Ros1了！

```bash
source ~/ros_catkin_ws/install_isolated/setup.bash
```

可以直接将source命令写入bashrc中，避免每次使用时都需要更新。

### 其他

安装的Ros1 Desktop full版本并非完全包含所需要的Ros packages，可能还需要单独安装包。例如octomap、mavros等等。在Ubuntu20下安装方式为：

```bash
sudo apt install ros-noetic-octomap ros-noetic-mavros
```

但从源码安装Ros1，相应的包也需要源码安装。过程如下。

### 生成包文件并自动下载依赖

```bash
cd ~/ros_catkin_ws
rosinstall_generator package1 package2 --rosdistro noetic --deps --tar > noetic-packages.rosinstall
vcs import --input noetic-packages.rosinstall ./src
```

package1和package2的位置替换为需要安装的包即可，注意不需要添加ros-noetic-的内容，也可以继续在后面放更多的包以一次性安装，但是个人建议还是单独安装比较稳妥(因为Github连接问题，vcs下载包的方式会导致必须所有依赖项都正确下载才能正常编译)。

哪怕第一次依赖A下载成功，依赖B失败，第二次依赖B成功，依赖A失败，两次结合起来，看似A和B都成功了，但实际上在第二次下载开始时，vcs工具会删除所有依赖。因此下载必须全部成功才行。

以 mavros 举例，安装方式为：

```bash
cd ~/ros_catkin_ws
rosinstall_generator mavros --rosdistro noetic --deps --tar > noetic-packages.rosinstall
vcs import --input noetic-packages.rosinstall ./src
```

### 替换rosconsole和urdf

新生成的包文件可能对 rosconsole 和 urdf 存在依赖，参考本文解决依赖问题部分进行替换即可。

### 安装

```bash
./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
source ~/ros_catkin_ws/install_isolated/setup.bash
```



form https://zhuanlan.zhihu.com/p/688413327