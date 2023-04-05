# ros1-build-docs
## 序
此文档解释了在Fedora RISC-V上编译ROS1 Noetic Ninjemys的指南和奇奇怪怪的问题

## 流程
1. 启动一个 Fedora RISC-V 虚拟机
2. 执行 `sudo dnf install gcc-c++ python3-rosdep python3-rosinstall_generator python3-vcstool @buildsys-build` 安装依赖，`@buildsys-build`可能不存在，只需要保证常见的编译工具都已安装即可
3. 确保在有正常的国际互联网的前提下执行`sudo rosdep init && rosdep update`
4. 随后建立工作文件夹
```
$ mkdir ~/ros_catkin_ws
$ cd ~/ros_catkin_ws
```
5. 随后使用`rosinstall_generator`来生成依赖列表
```
$ rosinstall_generator desktop --rosdistro noetic --deps --tar > noetic-desktop.rosinstall
$ mkdir ./src
$ vcs import --input noetic-desktop.rosinstall ./src
```
6. 确保在有正常的国际互联网的前提下执行`rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic -y`来下载&安装依赖
7. 执行`$ ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release` 来编译
8. 然后就炸了，可能遇到的问题如ROS1中的rosconsole依赖log4cxx，但从v0.11开始会编译失败，目前openkoji里该库版本为v0.13
9. 可以应用 https://github.com/ros/rosconsole/pull/58.patch 这里的 patch 来修复此问题
10. 继续编译会发现出现类似诸如以下错误
```
/usr/include/log4cxx/boost-std-configuration.h:10:18: 错误：‘shared_mutex’不是命名空间‘std’中的一个类型名
   10 |     typedef std::shared_mutex shared_mutex;
      |                  ^~~~~~~~~~~~
/usr/include/log4cxx/boost-std-configuration.h:10:13: 附注：‘std::shared_mutex’ is only available from C++17 onwards
   10 |     typedef std::shared_mutex shared_mutex;
      |             ^~~
```
这种情况需要将编译命令改成`$ ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_STANDARD=17`，这是由于log4cxx新版本使用了C++17的特性

11. 编译后会发现继续报错，提示RPM_ARCH not defined，这理应是rpmbuild提供的宏，但作为workaround将编译命令改成`$ RPM_ARCH=riscv64 RPM_PACKAGE_RELEASE=1 RPM_PACKAGE_VERSION="1.0" RPM_PACKAGE_NAME="foobar-fedora" ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_STANDARD=17`
12. 随后会发现部分包还是会报11的错误，记得到相应包的源码目录，找到CMakeLists.txt，找到`set(CMAKE_CXX_STANDARD 14)`将14改成17
13. 这时候理应编译成功
