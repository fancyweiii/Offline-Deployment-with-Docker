# Offline-Deployment-with-Docker

出于某些原因，某些程序需要部署到离线机器，由于解决依赖过于折磨，故整理了网络上部分文档到此readme中，以cpu版本的三维重建为例

## Docker

### 麒麟/其余Linux Docker安装

根据处理器架构下载docker：[Index of linux/static/stable/](https://download.docker.com/linux/static/stable/)

解压安装包```docker```并转移到```/usr/bin```下

```
mv docker/* /usr/bin/
```

### 配置文件

编辑配置文件（可能为空）```/usr/lib/systemd/system/docker.service```

放入内容：

```
[Unit]
 
Description=Docker Application Container Engine
 
Documentation=https://docs.docker.com
 
After=network-online.target firewalld.service
 
Wants=network-online.target
 
 
 
[Service]
 
Type=notify
 
ExecStart=/usr/bin/dockerd
 
ExecReload=/bin/kill -s HUP $MAINPID
 
LimitNOFILE=infinity
 
LimitNPROC=infinity
 
TimeoutStartSec=0
 
Delegate=yes
 
KillMode=process
 
Restart=on-failure
 
StartLimitBurst=3
 
StartLimitInterval=60s
 
 
 
[Install]
 
WantedBy=multi-user.target
```

添加docker.service文件的权限

```
chmod +x /usr/lib/systemd/system/docker.service
 
systemctl daemon-reload
```

创建daemon.json文件

```
cd /etc
mkdir docker
cd docker
touch daemon.json
vi daemon.json
```

添加内容

```
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

开启服务并设置自启动

```
systemctl daemon-reload
systemctl start docker
systemctl enable docker
```

验证

```
docker -v
```

### GPU容器配置

#### 在线安装

container toolkit安装

```
# 添加仓库源
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```
sudo apt-get update

export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1
  sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

配置

```
sudo nvidia-ctk runtime configure --runtime=docker

sudo systemctl restart docker
```

#### 离线安装

在联网设备提前准备安装包，上述`apt-get install`更改为`apt download`将安装包打包到离线设备进行安装，并进行配置

#### gpu挂载

运行nvidia docker容器时添加`--rm --gpus all`挂载设备gpu至容器，例如: `docker run -it --rm --gpus all -v /mnt/e/code/kl/wx:/workspace colmap:gpu bash`

## Docker-compose







## 镜像构建与迁移

### 同架构（以cpu平台colmap部署为例）

这里只提供x86 => x86和x86 => arm参考步骤，同理arm => arm，但是某两个品牌的arm如何部署到x86暂时没有条件测试😁

#### 镜像构建

根据文档安装步骤同步配置Dockerfile

**在Windows平台使用Docker时注意最后使用Ninja编译时使用-j指定线程数（colmap文档未指定），不指定会导致docker守护进程内存不足，出现rpc error（Windows版本Docker运行在WSL上，但在WSL中直接编译colmap不会出现该现象）**

```
# builder stage
FROM ubuntu:latest AS builder

RUN apt-get update && apt-get install -y \
    git \
    cmake \
    ninja-build \
    build-essential \
    libboost-program-options-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libeigen3-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libgmock-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libceres-dev \
    libcurl4-openssl-dev \
	libmkl-full-dev


RUN apt-get install -y gcc-10 g++-10 && \
    export CC=/usr/bin/gcc-10 && \
    export CXX=/usr/bin/g++-10

COPY colmap /colmap

# 根据实际设置ninja线程数-j{$thread}
RUN cd /colmap && \
    mkdir build && \
    cd build && \
    # cmake .. -GNinja -DBLA_VENDOR=Intel10_64lp && \
    cmake .. -GNinja \
    -DCMAKE_INSTALL_PREFIX=/colmap-install && \
    ninja -j8 && \
    ninja install

# runtime stage
FROM ubuntu:latest AS runtime

# 实际上运行镜像的依赖包有些可能不再需要，有些可以使用release版本的包，根据需要确定，这里没有实验
RUN apt-get update && apt-get install -y \
    git \
    cmake \
    ninja-build \
    build-essential \
    libboost-program-options-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libeigen3-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libgmock-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libceres-dev \
    libcurl4-openssl-dev \
	libmkl-full-dev


COPY --from=builder /colmap-install/ /usr/local/
```

在Dockerfile所在文件夹中构建镜像

```
docker build -t colmap:offline .
```

测试：```docker run --rm colmap:offline colmap```，出现usage界面说明编译完成

#### 镜像导出与导入

```
# 导出
docker save -o colmap_offline.tar colmap:offline

# 导入
docker load -i colmap_offline.tar
```

#### 运行与数据挂载

以终端形式运行容器，**/data为需要挂载目录，/workspace为容器中位置，都是绝对位置**

```
docker run -it \
    -v /data:/workspace \
    colmap:offline bash
```

容器可能会出现权限不足的情况，使用sudo，并添加参数`--privileged`进行解决，例如：`sudo docker run -it --privileged -v /data:/workspace colmap:offline bash`

**服务器可能没有可用的显示服务器（X11）或虚拟 OpenGL 环境，需要额外配置，记得设置不使用gpu进行特征提取和匹配**

```
export QT_QPA_PLATFORM='offscreen'
colmap feature_extractor --database_path db.db --image_path images --FeatureExtraction.use_gpu 0
colmap exhaustive_matcher --database_path db.db --FeatureMatching.use_gpu 0
mkdir sparse
colmap mapper --database_path db.db --image_path images --output_path sparse
```

### X86_64 => ARM

X86_64平台以WSL上的发行版Ubuntu 22.04为例

#### 环境准备

##### Docker

有网环境可以apt-get安装，否则同上

```
# 准备依赖包
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 添加Docker仓库GPG密钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 添加Docker仓库
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
# 安装Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 无梯子推荐配置国内镜像源
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "...",
    ]
}
EOF

# 重启服务
sudo systemctl daemon-reload
sudo systemctl restart docker
```

确认安装了buildx插件

```
docker buildx version

# 报错进行安装
docker plugin install tonistiigi/docker-buildx:latest
docker buildx create --use
```

##### ARM模拟器Qemu

```
sudo apt-get install qemu
```

#### 镜像构建

拉取模拟器镜像推荐使用代理，镜像源不一定可用

代理配置

```
# 根据实际配置
export http_proxy="http:/127.0.0.1:7890"
export https_proxy="http:/127.0.0.1:7890"

sudo vim /etc/docker/daemon.json

# 下面写入json，设置的镜像源不能用可以删除
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7890",
    "https-proxy": "http://127.0.0.1:7890"
  }
}
```

拉取镜像并启动qemu模拟器，**每次重启WSL需要启动QEMU模拟器**：`docker run --privileged --rm tonistiigi/binfmt --install all`，否则会出现：`exec /bin/sh: exec format error`

```
docker run --privileged --rm tonistiigi/binfmt --install all
```

cd到Dockerfile所在目录

以colmap为例，arm没有libmkl-full-dev，根据实际调整

```
docker buildx build   --platform linux/arm64 \
	-t colmap:arm64 \
	--output type=docker .
```

以colmap + openmvs为例的Dockerfile（现有稠密重建方法大多需要gpu支持，这是目前测试过的支持cpu进行稠密重建的方法）

```
# builder stage
FROM ubuntu:latest AS builder

RUN apt-get update && apt-get install -y \
    git \
    cmake \
    ninja-build \
    build-essential \
    libboost-program-options-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libboost-iostreams-dev \
    libboost-serialization-dev \
    libopencv-dev \
    libnanoflann-dev\
    libeigen3-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libgmock-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libceres-dev \
    libcurl4-openssl-dev \
    libsuitesparse-dev \
    libgmp-dev \
    libmpfr-dev \
    python3-dev \
    libpng-dev \
    libjpeg-dev \
    libtiff-dev \
    libglu1-mesa-dev \
    libglew-dev \
    libglfw3-dev \
    libboost-thread-dev \
    zlib1g-dev \
    libhwy-dev \
    libbrotli-dev \
    pkg-config \
    libgif-dev \
    libopenexr-dev \
    libpng-dev \
    libwebp-dev \
    libcgal-qt5-dev
# libmkl-full-dev



RUN apt-get install -y gcc-10 g++-10 && \
    export CC=/usr/bin/gcc-10 && \
    export CXX=/usr/bin/g++-10

# 目前arm没有编译包，需要这里自行编译
COPY libjxl /libjxl

RUN cd /libjxl && mkdir build && cd build && \
    cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .. && \
    cmake --build . -j8 && cmake --install .

COPY vcglib /usr/local/lib/vcglib

COPY openMVS /openMVS
COPY colmap /colmap
COPY cgal /cgal

# cgal编译时一个依赖包后面也要用，只用apt安装libcgal后面会缺少依赖
RUN cd /cgal && mkdir cgal_build && cd cgal_build &&\
    cmake .. &&\
    make -j8 && make install

RUN cd /openMVS && mkdir openMVS_build && cd openMVS_build && \
    cmake -DCMAKE_BUILD_TYPE=RELEASE \
    -DCMAKE_INSTALL_PREFIX=/openMVS-install -DVCG_ROOT="/usr/local/lib/vcglib" \
    .. && \
    make -j8 && make install

RUN cd /colmap && \
    mkdir build && \
    cd build && \
    # cmake .. -GNinja -DBLA_VENDOR=Intel10_64lp && \
    cmake .. -GNinja \
    -DCMAKE_INSTALL_PREFIX=/colmap-install && \
    ninja -j8 && \
    ninja install

# runtime stage
FROM ubuntu:latest AS runtime

RUN apt-get update && apt-get install -y \
    git \
    cmake \
    ninja-build \
    build-essential \
    libboost-program-options-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libboost-iostreams-dev \
    libboost-serialization-dev \
    libopencv-dev \
    libnanoflann-dev\
    libeigen3-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libgmock-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libceres-dev \
    libcurl4-openssl-dev \
    libsuitesparse-dev \
    libgmp-dev \
    libmpfr-dev \
    python3-dev \
    libpng-dev \
    libjpeg-dev \
    libtiff-dev \
    libglu1-mesa-dev \
    libglew-dev \
    libglfw3-dev \
    libboost-thread-dev \
    zlib1g-dev \
    libhwy-dev \
    libbrotli-dev \
    pkg-config \
    libgif-dev \
    libopenexr-dev \
    libpng-dev \
    libwebp-dev \
    libcgal-qt5-dev
# libmkl-full-dev

COPY --from=builder /colmap-install/ /usr/local/
COPY --from=builder /openMVS-install/ /usr/local/
COPY --from=builder /usr/local /usr/local

RUN ln -s /usr/local/bin/OpenMVS/* /usr/local/bin/
RUN echo "/usr/local/lib" > /etc/ld.so.conf.d/local.conf && ldconfig
ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
```

...

中间报错不少，慢慢调

比如可能会出现Type.inl中的IMWRITE_JPEGXL_QUALITY报错，直接改成IMWRITE_JPEG_QUALITY（后果不知道，反正可以编译，可能之前的libjxl包进行了更新）

...

最后保存镜像

```
docker save -o colmap_openmvs_arm64.tar colmap_openmvs:arm64
```

同上在目标机器加载镜像

#### 运行与数据挂载

X86测试同上，加上platform即可，记得提前打开qemu

```
docker run --rm -it \
	-v /data:/workspace \
	--platform linux/arm64 colmap_openmvs:arm64 bash
```

