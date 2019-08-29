# ZCU102上SD卡的制作&ROS安装&SP-SLAM的编译运行教程

## I. SD卡的制作

### i. img烧写
1. 在[xilinx官网](https://www.xilinx.com/products/design-tools/ai-inference/ai-developer-hub.html#edge)上下载适配ZCU102的img文件
   ![img](/pic/img.jpg)
2. 将img文件烧入SD卡中，推荐使用[balenaEtcher](https://www.balena.io/etcher/)软件
3. 在Linux系统的电脑上使用GParted软件将ROOTFS分区扩展到填满整张SD卡
   ![gparted](/pic/gparted.png)
4. 将SD卡插在ZCU102板上，开机运行

### ii. 安装DHCP

> 使用深鉴科技的ZU9配合深鉴科技的BOOT.bin文件，文件配置中采用静态IP上网。文件系统没有安装 dhcp的客户端，导致服务dhcp上网。
> 
> 解决方法参考[Debian系统无法DHCP上网](http://www.yujincheng.net/2019/01/18/Debian%E7%B3%BB%E7%BB%9F%E6%97%A0%E6%B3%95dhcp%E4%B8%8A%E7%BD%91.html#debian系统无法dhcp上网)

1. 修改`/etc/network/interfaces`关于eth0的配置，设置IP地址，采用静态的IP的形式连上网。
2. 安装 dhcp 的客户端
   ```python
   sudo apt-get update
   sudo apt-get install dhcpcd5
   ```
3. 在终端中运行一次 dhcpcd5
   ```python
   dhcpcd5
   ```
4. 修改`/etc/network/interfaces`关于eth0的配置，使能无线dhcp上网
   ![dhcp](/pic/dhcp.jpg)

### iii. 重装OpenCV

> ZU9自带的OpenCV并不是编译安装的，而是在安装DNNDK的时候安排的，所以缺少OpenCV对应的cmake配置文件，导致在ROS中找不到OpenCV。即使能够找到，DNNDK自带的OpenCV也缺少Opencv的许多官方库，比如解码工具就没有安装。需要重新编译安装OpenCV。
> 
> 安装过程参考[Ubuntu下源码安装Opencv完全指南](https://m.oldpan.me/archives/ubuntu-install-opencv-from-source)和[linux(Ubuntu16.04)下opencv3.4.1的安装和验证](http://blog.sina.com.cn/s/blog_6622f5c30102xntt.html)。

1. 删除OpenCV
    ```python
    sudo rm -r /usr/local/include/opencv2 /usr/local/include/opencv /usr/include/opencv /usr/include/opencv2 /usr/local/share/opencv /usr/local/share/OpenCV /usr/share/opencv /usr/share/OpenCV /usr/local/bin/opencv* /usr/local/lib/libopencv*
    ```
2. 安装所有依赖件
    ```python
    # 首先我们先移除系统中已经存在的依赖，这一部必须要做
    sudo apt-get remove x264 libx264-dev
    
    # 然后安装我们需要的依赖
    sudo apt-get install build-essential checkinstall cmake pkg-config yasm
    sudo apt-get install git gfortran
    sudo apt-get install libjpeg8-dev libjasper-dev libpng12-dev
    
    # 下面根据版本选择安装 
    #  Ubuntu 14.04
    sudo apt-get install libtiff4-dev
    #  Ubuntu 16.04
    sudo apt-get install libtiff5-dev
    
    sudo apt-get install libavcodec-dev libavformat-dev libswscale-dev libdc1394-22-dev
    sudo apt-get install libxine2-dev libv4l-dev
    sudo apt-get install libgstreamer0.10-dev libgstreamer-plugins-base0.10-dev
    sudo apt-get install qt5-default libgtk2.0-dev libtbb-dev
    sudo apt-get install libatlas-base-dev
    sudo apt-get install libfaac-dev libmp3lame-dev libtheora-dev
    sudo apt-get install libvorbis-dev libxvidcore-dev
    sudo apt-get install libopencore-amrnb-dev libopencore-amrwb-dev
    sudo apt-get install x264 v4l-utils
    
    sudo apt-get install libprotobuf-dev protobuf-compiler
    sudo apt-get install libgoogle-glog-dev libgflags-dev
    sudo apt-get install libgphoto2-dev libeigen3-dev libhdf5-dev doxygen
    ```
3. 下载[OpenCV3.1.0](https://github.com/opencv/opencv/releases/tag/3.1.0)，并解压
4. 编译安装
    ```python
    cd opencv-3.1.0
    mkdir build
    cd build
    cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -D ENABLE_PRECOMPILED_HEADERS=OFF ..
    make
    sudo make install
    sudo sh -c 'echo "/usr/local/lib" >> /etc/ld.so.conf.d/opencv.conf'
    sudo ldconfig
    ```

    > 编译OpenCV时出错，致命错误：stdlib.h：没有这样的文件或目录(Error compiling OpenCV, fatal error: stdlib.h: No such file or directory)
    > 解决方案：禁用预编译的头文件`-D ENABLE_PRECOMPILED_HEADERS=OFF`

5. 配置bash
    ```python
    sudo vim /etc/bash.bashrc
    
    # 在最末尾添加
    PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig 
    export PKG_CONFIG_PATH
    
    # 保存，执行如下命令使得配置生效
    source /etc/bash.bashrc
    
    # 更新
    sudo updatedb
    ```

### iv. 安装DNNDK

1. 在[xilinx官网](https://www.xilinx.com/products/design-tools/ai-inference/ai-developer-hub.html#edge)上下载DNNDK软件包，并解压
   ![dnndk](/pic/dnndk.jpg)
2. 安装DNNDK
    ```python
    cd xilinx_dnndk_v2.08/ZCU102/
    ./install.sh
    ```
3. 重启板卡，并更换带有DPU和superpoint后处理模块的BOOT.bin文件


## II. 安装ROS

> ROS的安装参考[Debian install of ROS Melodic](http://wiki.ros.org/melodic/Installation/Debian)

1. 添加sources.list
    ```python
    # 可以使用官网的源，也可以使用镜像的源
    sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
    
    # 中科大
    sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ stretch main" > /etc/apt/sources.list.d/ros-latest.list'

    # 清华
    sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ stretch main" > /etc/apt/sources.list.d/ros-latest.list'
    ```
2. 添加keys
    ```python
    sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
    ```
3. 系统更新
    ```python
    sudo apt-get update
    ```
4. 安装ROS
    ```python
    sudo apt-get install ros-melodic-desktop-full
    ```
5. 初始化rosdep
    ```python
    sudo rosdep init
    rosdep update
    ```
6. ROS环境配置
    ```python
    echo "source /opt/ros/melodic/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```
7. 安装rosinstall工具和其他依赖项
    ```python
    sudo apt install python-rosinstall python-rosinstall-generator python-wstool build-essential
    ```

## III. SP-SLAM的编译运行

### i. 环境准备

1. 下载ORB_SLAM2_SP.zip和rgbd_dataset_freiburg1_room.zip文件，将ORB_SLAM2_SP.zip解压到`/root/catkin_ws/src/`目录下，将rgbd_dataset_freiburg1_room.zip解压到`/root/Data/`目录下
   > ORB_SLAM2_SP.zip和rgbd_dataset_freiburg1_room.zip文件存放在sqGPU服务器（172.16.0.34）的/home/xuzhl/SP_SLAM/目录下
2. 初始化ROS工作空间
    ```python
    cd root/catkin_ws/src
    catkin_init_workspace
    ```
3. 编译工作空间
    ```python
    cd root/catkin_ws
    catkin_make
    ```
4. 在.bashrc文件中加入
    ```python
    source /root/catkin_ws/devel/setup.bash
    export ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:/root/catkin_ws/src/ORB_SLAM2_SP/Examples/ROS
    ```
5. 链接lQt5::Widgets
    > 编译SLAM时会出现报错 error: cannot find -lQt5::Widgets 
    > 原因是程序去找libQt5:Widgets.so但是系统里面是没有冒号的libQt5Widgets.so，需要把文件libQt5Widgets.so链接到libQt5::Widgets.so
    
    ```python
    cd /usr/lib/aarch64-linux-gnu/
    sudo ln -s libQt5Widgets.so libQt5::Widgets.so
    ```

### ii. 安装Pangolin

1. 安装依赖项
    ```python
    sudo apt-get install libglew-dev
    ```
2. 安装Pangolin
    ```python
    git clone https://github.com/stevenlovegrove/Pangolin.git
    cd Pangolin
    mkdir build
    cd build
    cmake ..
    cmake --build .
    sudo make install
    ```

### iii. 开辟虚拟内存
> 板上内存较小，无法满足SLAM编译的要求，需要开辟虚拟内存。这里开辟了8G的虚拟内存。

1. 创建虚拟内存
    ```python
    mkdir /opt/images/
    dd if=/dev/zero of=/opt/images/swap bs=1024 count=8192000
    mkswap /opt/images/swap
    ```

2. 启用虚拟内存
    ```python
    swapon /opt/images/swap
    ```

> * `swapon /opt/images/swap`需要每次开机都运行一次，可以写在.bashrc中。
> * 用`free`命令可以看到当前的内存大小。

### iv. 编译运行SP-SLAM

1. 编译SP-SLAM
    ```python
    cd /root/catkin_ws/src/ORB_SLAM2_SP/
    ./build.sh
    ```
    
    > 编译到最后时会报错(神经网络重复定义)，这是因为在cmake中将dpu_superpoint.elf文件链接了多次。
    > 这是cmake的bug，目前还找不到修复的方法，临时解决方案是进入`/root/catkin_ws/src/ORB_SLAM2_SP/Examples/ROS/ORB_SLAM2_SP/build/CMakeFiles/`目录，将`ORB.dir`、`SP.dir`、`TUM.dir`文件夹下的`build.make`和`link.txt`文件中的含有`dpu_superpoint.elf`的重复项删除，每个文件中只保留一条。
    
    修改完成后再次make
    ```python
    cd /root/catkin_ws/src/ORB_SLAM2_SP/Examples/ROS/ORB_SLAM2_SP/build/
    make
    ```

2. 运行SP-SLAM
    ```python
    cd /root/catkin_ws/src/ORB_SLAM2_SP/Examples/ROS/ORB_SLAM2_SP/launch/
    roslaunch TUM.launch
    ```