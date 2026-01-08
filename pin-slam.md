# PIN-SLAM

## Introduction
PIN-SLAM is a full-fledged implicit neural LiDAR-Odometry-based SLAM system including odometry, loop closure detection, and globally consistent mapping.

<!-- TABLE OF CONTENTS -->
<details open="open" style='padding: 10px; border-radius:5px 30px 30px 5px; border-style: solid; border-width: 1px;'>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#platform">Platform</a>
    </li>
    <li>
      <a href="#docker-installation">Docker Installation</a>
    </li>
    <li>
      <a href="#run">RUN</a>
    </li>
    <li>
      <a href="#io">I/O</a>
    </li>
    <li>
      <a href="#pros">Pros</a>
    </li>
    <li>
      <a href="#cons">Cons</a>
    </li>
    <li>
      <a href="#reference">Reference</a>
    </li>
  </ol>
</details>

## Platform
- GPU: NVIDIA GeForce RTX 5080

## Docker Installation
### 1. Build Container
Build the docker container:
The dockerfile is simplified to the most stable status, only install "system low-level" to build the container and setting environment variables. Installation includes CUDA image, system dependencies, PIN-SLAM. 

Due to Pytorch keeps conflicting with the RTX5080 blackwell, PyTorch will be installed after the container is built.
```
cd docker
sudo chmod +x ./build_docker.sh
./build_docker.sh
```

Note: If you need to view the progress error while building the container, please modifiy the "./docker/build_docker.sh"
<details>
  <summary>(click to open)</summary>

```
#!/bin/bash
echo "Build docker"
sudo docker build --progress=plain -f cu117.Dockerfile -t pinslam:localbuild .
echo "docker successfully build!"
```
</details>

Note: If you need the "simplified.Dockerfile", it is shown as follows: 
<details>
  <summary>(click to open)</summary>
  
```
# 使用支援 sm_120 的最新 CUDA 12.6 鏡像
FROM nvidia/cuda:12.6.2-devel-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1

# 只安裝系統核心依賴
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-dev python3-pip python3-setuptools \
    build-essential cmake git curl wget pkg-config ninja-build \
    libgoogle-glog-dev libgflags-dev libatlas-base-dev libeigen3-dev \
    libboost-all-dev libpcl-dev libsuitesparse-dev \
    libopencv-dev python3-opencv \
    xfce4-terminal neovim && \
    ln -sf /usr/bin/python3 /usr/bin/python && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# 先不裝 Torch，只裝 PIN-SLAM 基礎代碼
WORKDIR /src
RUN git clone https://github.com/PRBonn/PIN_SLAM.git
WORKDIR /src/PIN_SLAM

# 修改啟動環境
ENV NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=all \
    QT_GRAPHICSSYSTEM=native \
    CUDA_MODULE_LOADING=LAZY \
    TORCH_CUDA_ARCH_LIST="12.0"

RUN git config --global --add safe.directory '*'
```
</details>

After building the container, configure the storage path in start_docker.sh and then run it by:
```
sudo chmod +x ./start_docker.sh
./start_docker.sh
```

Note: If you need the modified "start_docker.sh", it is shown as follows: 
<details>
  <summary>(click to open)</summary>
  
```
#!/bin/bash

# 賦予 Docker 存取顯示器的權限
xhost +local:docker

# 填入你實際存放 KITTI 或其他數據的本機路徑
# 記得把下面這行改成你電腦裡真正的路徑
DATA_PATH="/home/your_username/data" 

docker run -it --rm \
  --name pin_slam_instance \
  --gpus all \
  --privileged \
  --network host \
  --ipc host \
  --env="DISPLAY=$DISPLAY" \
  --env="QT_X11_NO_MITSHM=1" \
  --env="DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus" \
  --env="CUDA_MODULE_LOADING=LAZY" \
  --env="TORCH_CUDA_ARCH_LIST=12.0" \
  -v /run/user/$(id -u)/bus:/run/user/$(id -u)/bus \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v "$DATA_PATH":/storage \
  pinslam:localbuild \
  /bin/bash -c "export CUDA_MODULE_LOADING=LAZY; export TORCH_CUDA_ARCH_LIST=10.0; xfce4-terminal --title=PIN-SLAM"
```
</details>

### 2. Install PyTorch
After finishing building the container, it will jump up the container command window. 
<details open="open" style='padding: 10px; border-radius:5px 30px 30px 5px; border-style: solid; border-width: 1px;'>
  <summary>Please insert the following steps in container cmd to install PyTorch.</summary>
  
```
# 1. 更新 pip
pip install --upgrade pip

# 2. 單獨安裝 Torch：
pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cu124

# 3. 安裝 2026 年最新支援 Blackwell (sm_120) 的版本
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128

# 4. 安裝依賴包
cd /src/PIN_SLAM
pip install -r requirements.txt

# 5. 設定環境變數
export CUDA_MODULE_LOADING=LAZY
export TORCH_CUDA_ARCH_LIST="12.0+PTX"

＃ 6. 測試RTX 5080 (Blackwell)能夠與PyTorch溝通
python3 -c "import torch; print(f'Testing GPU: {torch.cuda.get_device_name(0)}'); a=torch.ones(1).cuda(); print('Result:', a+a)"
# 應該要出現類似:
# Testing GPU: NVIDIA GeForce RTX 5080
# Result: tensor([2.], device='cuda:0')
```
</details>

## RUN
### Sanitary Test
For a sanity test, do the following to download an example part (first 100 frames) of the KITTI dataset (seq 00):
```
sh ./scripts/download_kitti_example.sh
cd /src/PIN-SLAM
python3 pin_slam.py ./config/lidar_slam/run_demo.yaml -vsm
```

### Test with Self-prepared Data
Follow the instructions on how to run PIN-SLAM by typing:
```
python3 pin_slam.py -h
```

Process all pointclouds in the given <data-dir> (*.ply, *.pcd, *.bin, etc.) using default config file.
```
python3 pin_slam.py -i </path/to/your/point/cloud/folder> -vsm
```

Process all pointclouds in the given <data-dir> (*.ply, *.pcd, *.bin, etc.) using default config file.
```
python3 pin_slam.py <path-to-config-file.yaml> -i <data-dir> -vsm  
```

Process a given ROS1/ROS2 rosbag file (directory, ".bag")
```
python3 pin_slam.py <path-to-config-file.yaml> rosbag -i <path-to-my-rosbag> -dvsm
```

## I/O
### 1. Input Data
The only data need to prepare is point cloud file (*.ply, *.pcd, *.bin, etc.).

### 2. Output Data
- /meta/config_all.yaml (detail config info)
- /map/neural_points.ply ('.ply' format point cloud map)
- /mesh/mesh_18cm.ply (object segmented point cloud map)
- /model/pin_map.pth (.pth is PyTorch state dictionary, which is a Python dictionary that contains the state of a PyTorch model, including the model's weights, biases, and other parameters.)
- /log/
- run.sh (shell script to rum the code)
- odom_poses_tum.txt ('TUM' trajectory files)
- odom_poses_kitti.txt ('KITTI' pose files)
- odom_poses.ply (trajectory point)
- run_demo_sem.yaml (basic config settings, including file path)
- memory_footprint.npy (numpy binary file?)
- time_table.npy (numpy binary file that records timestamps)
- time_details.png (time consuming at each part) 

Note: Coordinate about TUM.txt and KITTI.txt, please refer to the figure below. 

![kiiti_and_tum_coordinate info](https://github.com/ashley17happy17/slam/blob/3438fc6d5f6d6bb567ad33c52bc5d5f7430c0c5b/img/kitti_and_tum_format_info.png)


## Pros
1. MIT License. (commercial application enabled)

## Cons
1. Can't directly run with self-prepared data, it will fail to SLAM.

## Reference
1. PIN-SLAM https://github.com/PRBonn/PIN_SLAM.git
