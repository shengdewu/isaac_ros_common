# 一、安装Docker

参考链接：https://segmentfault.com/a/1190000044229597

## 1.增加Docker中科大源

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

或者使用阿里源
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo sh -c 'echo "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list'

sudo apt-get update
```

## 2.安装Docker软件包

```shell
sudo apt install -y docker-ce docker-ce-cli containerd.io     
```

## 3.设置非root用户使用Docker

```shell
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```



# 二、安装和配置nvidia-container-toolkit

参考链接：https://nvidia-isaac-ros.github.io/getting_started/dev_env_setup.html

## 1.Docker 配置

安装方法（x86_64平台）：

```shell
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```shell
sudo apt-get update
```

```shell
sudo apt-get install -y nvidia-container-toolkit
```

配置方法（x86_64平台）：

```shell
sudo nvidia-ctk runtime configure --runtime=docker
```

```shell
sudo systemctl restart docker
```

## 2.重新启动 Docker

```shell
sudo systemctl daemon-reload && sudo systemctl restart docker
```

## 3.安装git-lfs下载大文件

```shell
sudo apt-get install git-lfs
git lfs install --skip-repo
```

## 4.创建ROS2个工作空间用于验证Isaac ROS

```shell
mkdir -p  ~/workspaces/isaac_ros-dev/src #替换为自己的工作目录
echo "export ISAAC_ROS_WS=${HOME}/workspaces/isaac_ros-dev/" >> ~/.bashrc #替换为自己的工作目录
source ~/.bashrc
```



# 三、安装isaac_ros_common功能包

参考链接：https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_pose_estimation/isaac_ros_centerpose/index.html#quickstart

## 1.下载isaac_ros_common功能包

```shell
cd ${ISAAC_ROS_WS}/src && \
   git clone https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common.git
```

## 2.使用脚本启动 Docker 容器（重要步骤，出问题参考下文）

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

正常启动后终端名称会变成：

```shell
admin@nvidia:/workspaces/isaac_ros-dev$
```



# 四、解决run_dev.sh报错问题

## 1.pip安装python包速度太慢问题

在isaac_ros_common/docker/Dockerfile.x86_64里的RUN python3 -m pip install -U 前面加上如下命令，替换为中科大源之后下载python包会更快：

```shell
RUN python3 -m pip config set global.index-url https://mirrors.ustc.edu.cn/pypi/web/simple
```

## 2.安装ROS2下载ros.key网络问题（大概率会报错）

报错：

```shell
 > [stage-0  5/22] RUN --mount=type=cache,target=/var/cache/apt     curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg     && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(source /etc/os-release && echo $UBUNTU_CODENAME) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null     && apt-get update:
0.072 curl: (7) Failed to connect to raw.githubusercontent.com port 443 after 7 ms: Connection refused
```

修改isaac_ros_common/docker/Dockerfile.ros2_humble：

解决方法：将raw.githubusercontent.com修改为raw.gitmirror.com

修改前：

```shell
curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg \ 
```

修改后：

```shell
curl -sSL https://raw.gitmirror.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg \ 
```

## 3.rosdep init网络问题（大概率会报错）

报错：

```shell
 > [stage-0 10/22] RUN --mount=type=cache,target=/var/cache/apt     rosdep init     && echo "yaml file:///etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml" | tee /etc/ros/rosdep/sources.list.d/00-nvidia-isaac.list     && rosdep update:                                      
0.321 ERROR: cannot download default sources list from:                                                                                 
0.321 https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/sources.list.d/20-default.list                                      
0.321 Website may be down.
0.321 <urlopen error <urlopen error [Errno 111] Connection refused> (https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/sources.list.d/20-default.list)>
```

修改isaac_ros_common/docker/Dockerfile.ros2_humble：



### 方案一：使用国内的rosdepc工具（但是后面还会报错）

解决方法：使用国内的rosdepc工具

pip增加一行安装rosdepc：

修改前：

```shell
# ROS Python fundamentals
RUN python3 -m pip install -U \
        flake8-blind-except \
        flake8-builtins \
        flake8-class-newline \
        flake8-comprehensions \
        flake8-deprecated \
        flake8-docstrings \
        flake8-import-order \
        flake8-quotes \
        numpy>=1.24.4 \
        matplotlib \
        pandas \
        rosbags \
        setuptools==65.7.0
```

修改后：

```shell
# ROS Python fundamentals
RUN python3 -m pip install -U \
        flake8-blind-except \
        flake8-builtins \
        flake8-class-newline \
        flake8-comprehensions \
        flake8-deprecated \
        flake8-docstrings \
        flake8-import-order \
        flake8-quotes \
        numpy>=1.24.4 \
        matplotlib \
        pandas \
        rosbags \
        setuptools==65.7.0 \
        rosdepc
```

rosdep init和rosdep update命令修改：

修改前：

```shell
RUN --mount=type=cache,target=/var/cache/apt \
    rosdep init \
    && echo "yaml file:///etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml" | tee /etc/ros/rosdep/sources.list.d/00-nvidia-isaac.list \
    && rosdep update
```

修改后：

```shell
RUN --mount=type=cache,target=/var/cache/apt \
    rosdepc init \
    && echo "yaml file:///etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml" | tee /etc/ros/rosdep/sources.list.d/00-nvidia-isaac.list \
    && rosdepc update
```



### 方案二：使用国内的6-rosdep工具 (可以彻底解决)

以上方案在执行bloom-generate rosdebian的时候还会报错，解决方法：

修改前：

```shell
RUN python3 -m pip install -U \
        flake8-blind-except \
        flake8-builtins \
        flake8-class-newline \
        flake8-comprehensions \
        flake8-deprecated \
        flake8-docstrings \
        flake8-import-order \
        flake8-quotes \
        numpy>=1.24.4 \
        matplotlib \
        pandas \
        rosbags \
        setuptools==65.7.0
```

修改后：

```shell
RUN python3 -m pip install -U \
        flake8-blind-except \
        flake8-builtins \
        flake8-class-newline \
        flake8-comprehensions \
        flake8-deprecated \
        flake8-docstrings \
        flake8-import-order \
        flake8-quotes \
        numpy>=1.24.4 \
        matplotlib \
        pandas \
        rosbags \
        setuptools==65.7.0 \
        6-rosdep
```

修改前：

```shell
# Setup rosdep
COPY rosdep/extra_rosdeps.yaml /etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml
RUN --mount=type=cache,target=/var/cache/apt \
    rosdep init \
    && echo "yaml file:///etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml" | tee /etc/ros/rosdep/sources.list.d/00-nvidia-isaac.list \
    && rosdep update
```

修改后：

```shell
#先执行一下该命令
RUN 6-rosdep

# Setup rosdep
COPY rosdep/extra_rosdeps.yaml /etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml
RUN --mount=type=cache,target=/var/cache/apt \
    rosdep init \
    && echo "yaml file:///etc/ros/rosdep/sources.list.d/nvidia-isaac.yaml" | tee /etc/ros/rosdep/sources.list.d/00-nvidia-isaac.list \
    && rosdep update
 
#6-rosdep貌似会改rosdistro/__init__.py文件，但是给的链接已经不可用了，需要执行以下命令修改
RUN sed -i 's|https://https://mirrors.tuna.tsinghua.edu.cn/rosdistro/master/index-v4.yaml|https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml|g' /usr/lib/python3/dist-packages/rosdistro/__init__.py
```



jetson本地需要执行：

```shell
cd /usr/lib/python3/dist-packages/rosdep2

RUN sed -i 's|https://https://|https://|g' gbpdistro_support.py
RUN sed -i 's|https://https://|https://|g' rep3.py
RUN sed -i 's|https://https://|https://|g' sources_list.py
```



## 4.rosdistro中的index-v4.yaml网络问题

**如果上面用的6-rosdep就不会有这个报错了**

报错：

```shell
 > [stage-0 10/21] RUN --mount=type=cache,target=/var/cache/apt     mkdir -p /opt/ros/humble/src && cd /opt/ros/humble/src     && git clone https://github.com/osrf/negotiated && cd negotiated && git checkout master     && source /opt/ros/humble/setup.bash     && cd negotiated_interfaces && bloom-generate rosdebian && fakeroot debian/rules binary     && cd ../ && apt-get install -y ./*.deb && rm ./*.deb     && cd negotiated && bloom-generate rosdebian && fakeroot debian/rules binary     && cd ../ && apt-get install -y ./*.deb && rm ./*.deb
...
4.105 urllib.error.URLError: <urlopen error <urlopen error [Errno 111] Connection refused> (https://raw.githubusercontent.com/ros/rosdistro/master/index-v4.yaml)>
```

修改isaac_ros_common/docker/Dockerfile.ros2_humble：

解决方法：将/usr/lib/python3/dist-packages/rosdistro/__init__.p文件中的https://raw.githubusercontent.com/ros/rosdistro/master/index-v4.yaml改为国内链接https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml

在报错位置前面增加一行：

```shell
RUN sed -i 's|raw.githubusercontent.com/ros/rosdistro/master/index-v4.yaml|mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml|g' /usr/lib/python3/dist-packages/rosdistro/__init__.py #注意多修改前.com后面有个ros,修改后的没有
```

## 5.git clone网络报错

**如果用的v2raya不会有这个问题**

修改isaac_ros_common/docker/Dockerfile.ros2_humble：

解决方法：在clash科学上网之后，需要配置git代理

在报错位置前面增加如下两行：

```shell
RUN git config --global http.proxy http://127.0.0.1:7890
RUN git config --global https.proxy http://127.0.0.1:7890

或者（clash verge）
RUN git config --global http.proxy http://127.0.0.1:7897
RUN git config --global https.proxy http://127.0.0.1:7897
```

如果没有科学上网的时候可以取消代理：

```shell
#取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy

#查看代理
git config --global --get http.proxy
git config --global --get https.proxy
git  config --list
```

在网速过慢的时候也会可能导致git报错

## 6.bloom-generate rosdebian编译报错

**如果上面用的6-rosdep就不会有这个报错了**

下面的两段代码moveit_task_constructor和moveit2_tutorials在编译过程中会报错，目前先注释掉，问题暂未解决

```shell
# Install MoveIt task constructor from source.  The "demo" package depends on moveit_resources_panda_moveit_config,
# installed from source above.
#RUN --mount=type=cache,target=/var/cache/apt \
#    mkdir -p ${ROS_ROOT}/src && cd ${ROS_ROOT}/src \
#    && git clone https://github.com/ros-planning/moveit_task_constructor.git -b humble \
#    && cd moveit_task_constructor && source ${ROS_ROOT}/setup.bash \
#    && cd msgs && bloom-generate rosdebian && fakeroot debian/rules binary \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb \
#    && cd rviz_marker_tools && bloom-generate rosdebian && fakeroot debian/rules binary \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb \
#    && cd core && bloom-generate rosdebian && fakeroot debian/rules binary DEB_BUILD_OPTIONS=nocheck \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb \
#    && cd capabilities && bloom-generate rosdebian && fakeroot debian/rules binary DEB_BUILD_OPTIONS=nocheck \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb \
#    && cd visualization && bloom-generate rosdebian && fakeroot debian/rules binary DEB_BUILD_OPTIONS=nocheck \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb \
#    && cd demo && bloom-generate rosdebian && fakeroot debian/rules binary DEB_BUILD_OPTIONS=nocheck \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb

# MoveIt 2's hybrid planning package depends on moveit_resources_panda_moveit_config, installed from source above.
RUN --mount=type=cache,target=/var/cache/apt \
apt-get update && apt-get install -y \
    ros-humble-moveit-hybrid-planning

# Install moveit2_tutorials from source (depends on moveit_hybrid_planning).
#RUN --mount=type=cache,target=/var/cache/apt \
#    mkdir -p ${ROS_ROOT}/src && cd ${ROS_ROOT}/src \
#    && git clone https://github.com/ros-planning/moveit2_tutorials.git -b humble \
#    && cd moveit2_tutorials && source ${ROS_ROOT}/setup.bash \
#    && bloom-generate rosdebian && fakeroot debian/rules binary \
#    && cd ../ && apt-get install -y ./*.deb && rm ./*.deb
```

## 7.其他报错

还可能会有其他各种报错后续在此添加...





# 五、验证Isaac ROS是否安装成功

参考链接：https://nvidia-isaac-ros.github.io/repositories_and_packages/index.html

可以在以上链接中随便选一个功能包来验证在Isaac ROS环境下能否正常编译和运行，下面以目标检测为例：

参考链接：https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_object_detection/isaac_ros_detectnet/index.html

## 1.下载代码

```shell
sudo apt-get install -y curl tar
```

```shell
NGC_ORG="nvidia"
NGC_TEAM="isaac"
NGC_RESOURCE="isaac_ros_assets"
NGC_VERSION="isaac_ros_detectnet"
NGC_FILENAME="quickstart.tar.gz"

REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/versions/$NGC_VERSION/files/$NGC_FILENAME"

mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets/${NGC_VERSION} && \
    curl -LO --request GET "${REQ_URL}" && \
    tar -xf ${NGC_FILENAME} -C ${ISAAC_ROS_WS}/isaac_ros_assets/${NGC_VERSION} && \
    rm ${NGC_FILENAME}
```

## 2.编译isaac_ros_detectnet

### 方案一  使用二进制包

使用脚本启动 Docker 容器：

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

安装二进制软件包：

```shell
sudo apt-get install -y ros-humble-isaac-ros-detectnet ros-humble-isaac-ros-dnn-image-encoder ros-humble-isaac-ros-triton
```

### 方案二  从源码编译

下载代码：

```shell
cd ${ISAAC_ROS_WS}/src && \
   git clone https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_object_detection.git
```

使用脚本启动 Docker 容器：

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

使用rosdep安装依赖包：

```shell
rosdep install --from-paths ${ISAAC_ROS_WS}/src/isaac_ros_object_detection --ignore-src -y
```

编译源码：

```shell
cd ${ISAAC_ROS_WS} && \
   colcon build --symlink-install --packages-up-to isaac_ros_detectnet
```

因为是用源码安装的，必须执行source命令才会从自己编译的包启动：

```shell
source install/setup.bash
```

## 3.运行launch文件

以rosbag回放数据的方案为例：

继续在Docker容器中执行以下脚本，该脚本会下载 [PeopleNet 模型](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/tao/models/peoplenet)

```shell
ros2 run isaac_ros_detectnet setup_model.sh --height 632 --width 1200 --config-file quickstart_config.pbtxt
```

安装依赖包：

```shell
sudo apt-get install -y ros-humble-isaac-ros-examples
```

启动模型推理功能包：

```shell
ros2 launch isaac_ros_examples isaac_ros_examples.launch.py launch_fragments:=detectnet interface_specs_file:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_detectnet/quickstart_interface_specs.json
```

打开一个新的Docker容器终端，回放rosbog数据：

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

```shell
ros2 bag play -l ${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_detectnet/rosbags/detectnet_rosbag --remap image:=image_rect camera_info:=camera_info_rect
```

## 4.可视化

打开一个新的Docker容器终端，运行可视化功能包：

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

```shell
cd ${ISAAC_ROS_WS} && \
source install/setup.bash && \
ros2 run isaac_ros_detectnet isaac_ros_detectnet_visualizer.py --ros-args --remap image:=image_rect
```

打开一个新的Docker容器终端，运行rqt_image_view：

```shell
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```

```shell
ros2 run rqt_image_view rqt_image_view /detectnet_processed_image
```

在界面中选择话题/detectnet_processed_image，可以看到绘制了检测框的可视化图像

![rqt_visualizer](https://media.githubusercontent.com/media/NVIDIA-ISAAC-ROS/.github/main/resources/isaac_ros_docs/repositories_and_packages/isaac_ros_object_detection/isaac_ros_detectnet/rqt_visualizer.png/)
