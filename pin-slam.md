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
      <a href="#config">Config</a>
    </li>
    <li>
      <a href="#nvidia_container_toolkit_failure">NVIDIA Container Toolkit Failure</a>
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


<a name="platform"></a>
## Platform
- GPU: NVIDIA GeForce RTX 5080


<a name="docker-installation"></a>
## Docker Installation
### 1. Build Container
Build the docker container:
The dockerfile is simplified to the most stable status, only install "system low-level" to build the container and setting environment variables. Installation includes CUDA image, system dependencies, PIN-SLAM. 

Due to PyTorch keeps conflicting with the RTX5080 blackwell, PyTorch will be installed after the container is built.
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

docker run -it \
  --name pin_slam_instance \
  --gpus all \
  --privileged \
  --network host \
  --ipc host \
  --env="DISPLAY=$DISPLAY" \
  --env="QT_X11_NO_MITSHM=1" \
  --env="DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus" \
  --env="CUDA_MODULE_LOADING=LAZY" \
  --env="TORCH_CUDA_ARCH_LIST=12.0+PTX" \
  -v /run/user/$(id -u)/bus:/run/user/$(id -u)/bus \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v "$DATA_PATH":/storage \
  pinslam:localbuild \
  /bin/bash -c "export CUDA_MODULE_LOADING=LAZY; export TORCH_CUDA_ARCH_LIST=12.0+PTX; xfce4-terminal --title=PIN-SLAM"
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

### 3. Save Fine-installed Image (Optional)
Back to your host CMD, try to search the docker which is named "pinslam:localbuild" and in "Exited" status. Save it as the new image that is fine-installed.
```
# 設定環境變數
echo 'export TORCH_CUDA_ARCH_LIST="12.0+PTX"' >> ~/.bashrc
echo 'export CUDA_MODULE_LOADING=LAZY' >> ~/.bashrc
# 確認目前docker list有哪些物件
docker ps -a
# 找到目標容器並輸出成新的映像檔
docker commit <你的容器ID> pinslam_rtx5080_fixed
```

If you gonna to move to new pc to start the repository, then save the image to a .tar compressed file. Then you can move this .tar file to another pc.

```
# 格式：docker save -o [存檔路徑] [映像檔名稱]
docker save -o pinslam_transfer.tar pinslam_rtx5080_fixed
```

As you move to new environment, please make sure the pc is installed with docker and nvidia toolkit.
```
docker load -i pinslam_transfer.tar
```

And check it loads successfully, insert the following command, then you'll see "pinslam_rtx5080_fixed" in the list.
```
docker images
```

Please modify the "start_docker.sh" first, it is shown as follows. And then you can run the "start_docker.sh" to build new container. 

<details>
  <summary>(click to open)</summary>
  
```
# 賦予 Docker 存取顯示器的權限
xhost +local:docker

# 填入你實際存放 KITTI 或其他數據的本機路徑
# 記得把下面這行改成你電腦裡真正的路徑
DATA_PATH="/home/your_username/data" 

docker run -it \
  --name pin_slam_rtx5080_fixed_instance \
  --gpus all \
  --privileged \
  --network host \
  --ipc host \
  --env="DISPLAY=$DISPLAY" \
  --env="QT_X11_NO_MITSHM=1" \
  --env="DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus" \
  --env="CUDA_MODULE_LOADING=LAZY" \
  --env="TORCH_CUDA_ARCH_LIST=12.0+PTX" \
  -v /run/user/$(id -u)/bus:/run/user/$(id -u)/bus \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v "$DATA_PATH":/storage \
  pinslam_rtx5080_fixed \
  /bin/bash -c "export CUDA_MODULE_LOADING=LAZY; export TORCH_CUDA_ARCH_LIST=12.0+PTX"
```
</details>

As the new image is save, please enter the host CMD with shell script to open the new container. No need to run "start_docker.sh" after the container is built.
```
cd ./docker
bash run_docker.sh
```

Note: If you need the "run_docker.sh", it is shown as follows: 
<details>
  <summary>(click to open)</summary>

```
docker start pin_slam_rtx5080_fixed_instance
# 進入容器
docker exec -it pin_slam_rtx5080_fixed_instance /bin/bash
```
</details>


<a name="run"></a>
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

Process all pointclouds in the given <data-dir> using specific config file (e.g. run_kitti.yaml).
```
python3 pin_slam.py <path-to-config-file.yaml> -vsm  
```

Process a given ROS1/ROS2 rosbag file (directory, ".bag")
```
python3 pin_slam.py <path-to-config-file.yaml> rosbag -i <path-to-my-rosbag> -dvsm
```

<a name="i_o"></a>
## I/O
### 1. Input Data
 - point cloud file, ex: *.ply, *.pcd, *.bin, etc.
 - pose file, ex: kitti format
 - label file: point-wise label path, for semantic mapping (optional)
 - calib file, ex: lidar to sensor frame, example format as follows (optional)
   
```
P0: 7.188560000000e+02 0.000000000000e+00 6.071928000000e+02 0.000000000000e+00 0.000000000000e+00 7.188560000000e+02 1.852157000000e+02 0.000000000000e+00 0.000000000000e+00 0.000000000000e+00 1.000000000000e+00 0.000000000000e+00
P1: 7.188560000000e+02 0.000000000000e+00 6.071928000000e+02 -3.861448000000e+02 0.000000000000e+00 7.188560000000e+02 1.852157000000e+02 0.000000000000e+00 0.000000000000e+00 0.000000000000e+00 1.000000000000e+00 0.000000000000e+00
P2: 7.188560000000e+02 0.000000000000e+00 6.071928000000e+02 4.538225000000e+01 0.000000000000e+00 7.188560000000e+02 1.852157000000e+02 -1.130887000000e-01 0.000000000000e+00 0.000000000000e+00 1.000000000000e+00 3.779761000000e-03
P3: 7.188560000000e+02 0.000000000000e+00 6.071928000000e+02 -3.372877000000e+02 0.000000000000e+00 7.188560000000e+02 1.852157000000e+02 2.369057000000e+00 0.000000000000e+00 0.000000000000e+00 1.000000000000e+00 4.915215000000e-03
Tr: 4.276802385584e-04 -9.999672484946e-01 -8.084491683471e-03 -1.198459927713e-02 -7.210626507497e-03 8.081198471645e-03 -9.999413164504e-01 -5.403984729748e-02 9.999738645903e-01 4.859485810390e-04 -7.206933692422e-03 -2.921968648686e-01
```

### 2. Output Data
- /meta/config_all.yaml (detail config info)
- /map/neural_points.ply ('.ply' format point cloud map)
- /mesh/mesh_18cm.ply (object segmented point cloud map)
- /model/pin_map.pth (.pth is PyTorch state dictionary, which is a Python dictionary that contains the state of a PyTorch model, including the model's weights, biases, and other parameters.)
- /log/
- run.sh (shell script to rum the code)
- odom_poses_tum.txt ('TUM' LO pose files)
- odom_poses_kitti.txt ('KITTI' LO pose files)
- odom_poses.ply (trajectory point)
- slam_poses_tum.txt ('TUM' SLAM pose files)
- slam_poses_kitti.txt ('KITTI' SLAM pose files)
- run_demo_sem.yaml (basic config settings, including file path)
- memory_footprint.npy (numpy binary file?)
- time_table.npy (numpy binary file that records timestamps)
- time_details.png (time consuming at each part)
- loop_log.txt (record start, end frame no., and the RT matrix)
- loop_plot.png (record the loop closure correction)

Note: Coordinate about TUM.txt and KITTI.txt, please refer to the figure below. 

![kiiti_and_tum_coordinate info](https://github.com/ashley17happy17/slam/blob/3438fc6d5f6d6bb567ad33c52bc5d5f7430c0c5b/img/kitti_and_tum_format_info.png)


<a name="config"></a>
## Config

The config settings are set in "./utils/config.py". <details> <summary>(click to open)</summary>

**1. Settings and DataLoader**

```
# settings
self.name: str = "dummy"  # experiment name
self.run_name: str = self.name # this would also include an unique timestamp

self.run_path: str = ""
self.output_root: str = "experiments"  # output root folder
self.pc_path: str = ""  # input point cloud folder
self.pose_path: str = ""  # input pose file
self.calib_path: str = ""  # input calib file (to sensor frame), optional
self.label_path: str = "" # input point-wise label path, for semantic mapping (optional)

# for uisng specific data loader
self.use_dataloader: bool = False
self.data_loader_name: str = "generic"
self.data_loader_seq: str = ""

self.load_model: bool = False  # load the pre-trained model or not
self.model_path: str = "/"  # pre-trained model path

self.first_frame_ref: bool = False  # if false, we directly use the world
# frame as the reference frame
self.begin_frame: int = 0  # begin from this frame
self.end_frame: int = 100000  # end at this frame
self.step_frame: int = 1  # process every x frame (跳過幾幀點雲)

self.seed: int = 42 # random seed for the experiment
self.num_workers: int = 12 # number of worker for the dataloader
self.device: str = "cuda"  # use "cuda" or "cpu"
self.gpu_id: str = "0"  # used GPU id

# dataset specific
self.kitti_correction_on: bool = False # intrinsic vertical angle correction # issue 11
self.correction_deg: float = 0.0
self.stop_frame_thre: int = 20 # determine if the robot is stopped when there's almost no motion in a time peroid

# motion undistortion
self.deskew: bool = False #(運動去畸變。如果你在 PCD 檔案中沒有每個點的時間戳，開啟這個也沒用；如果有，務必開啟以修正旋轉時的拉伸。)
self.lidar_type_guess: str = "velodyne"
```

**2. Pointcloud Preprocess**

```
# preprocess
# distance filter
self.min_range: float = 2.5 # filter too-close points (and 0 artifacts)
self.max_range: float = 60.0 # filter far-away points
self.adaptive_range_on: bool = False # use an adpative range

# filter for z coordinates (unit: m)
self.min_z: float = -5.0  
self.max_z: float = 80.0

self.rand_downsample: bool = False  # apply random or voxel downsampling to input original point clcoud
self.vox_down_m: float = 0.05 # the voxel size if using voxel downsampling (unit: m)
self.rand_down_r: float = 1.0 # the decimation ratio if using random downsampling (0-1)

# semantic related
self.semantic_on: bool = False # semantic shine mapping on [semantic]
self.sem_class_count: int = 20 # semantic class count: 20 for semantic kitti
self.sem_label_decimation: int = 1 # use only 1/${sem_label_decimation} of the available semantic labels for training (fitting)
self.freespace_label_on: bool = False
self.filter_moving_object: bool = True

# color (intensity) related 
self.color_map_on: bool = True # colorized mapping default on (if False, then we only visualize the colorized point cloud but do not use them for mapping or localization)
self.color_on: bool = True
self.color_channel: int = 1 # For RGB, channel=3, For Lidar with intensity, channel=1

# robust processing
self.reboot_frame_thre: int = 5 # if lose track for more than this frame, we consider the system failed and reboot the system

# map-based dynamic filtering (observations in certain freespace are dynamic) (移除free space裡動態物件造成的ghost point)
self.dynamic_filter_on: bool = True
self.dynamic_certainty_thre: float = 1.0 
self.dynamic_sdf_ratio_thre: float = 0.5 # type1 dynamic
self.dynamic_min_grad_norm_thre: float = 0.25 # type2 dynamic
```

**3. Neural Points**

```
# neural points
self.voxel_size_m: float = 0.3 # we use the voxel hashing structure to maintain the neural points, the voxel size is set as this value (關鍵，決定hash size和地圖解析度)
self.weighted_first: bool = True # weighted the neighborhood feature before decoding to sdf or do the weighting of the decoded sdf afterwards
self.layer_norm_on: bool = False # apply layer norm to the features
self.num_nei_cells: int = 2 # the neighbor searching padding voxel # NOTE: can even be set to 3 when the motion is dramastic
self.query_nn_k: int = 6 # query the point's k nearest neural points
self.use_mid_ts: bool = False # use the middle of the created and last updated timestamp for adjusting or just use the created timestamp
self.search_alpha: float = 0.2 # the larger this value is, the larger neighborhood region would be, the more robust to the highly dynamic motion and also the more time-consuming
self.idw_index: int = 2 # the index for IDW (inverse distance weighting), 2 means square inverse
self.buffer_size: int = int(5e7) # buffer size for hashing, the smaller, the more likely to collision

# shared by both kinds of feature 
self.feature_dim: int = 8  # length of the feature for each grid feature
self.feature_std: float = 0.0  # grid feature initialization standard deviation

# Use all the surface samples or just the exact measurements to build the neural points map
# If True may lead to larger memory consumption, but is more robust while the reconstruction.
self.from_sample_points: bool = True
self.from_all_samples: bool = False  # even use the freespace samples (for better ESDF mapping at a cost of larger memory consumption)
self.map_surface_ratio: float = 0.5 # ratio * surface sample std, use those samples for initializing neural points

# local map
self.diff_ts_local: float = 400.0 # deprecated (use travel distance instead)
self.local_map_travel_dist_ratio: float = 5.0
self.local_map_radius: float = 50.0

# map management
self.prune_map_on: bool = False
self.max_prune_certainty: float = 3.0 # neural point pruning threshold
self.prune_freq_frame: int = 100
```

**4. Training and Loss**
```
# training sampler
# spilt into 3 parts for sampling: close-to-surface (+ exact beam endpoint) + front-surface-freespace + behind-surface-freespace
self.surface_sample_range_m: float = 0.25 # better to be set according to the noise level (actually as the std for a gaussian distribution)
self.surface_sample_n: int = 3
self.free_sample_begin_ratio: float = 0.3 # minimum ray distance ratio in front of the surface 
self.free_sample_end_dist_m: float = 1.0 # maximum distance behind the surface (unit: m)
self.free_front_n: int = 2 # NOTE: increase this value to remove dynamic objects in the map
self.free_behind_n: int = 1

# training data pool related (for replay)
self.window_radius: float = 50.0 # unit: m
self.pool_capacity: int = int(1e7)
self.bs_new_sample: int = 2048 # number of the sample per batch for the current frame's data, half of all the data
self.new_certainty_thre: float = 1.0
self.pool_filter_freq: int = 10 

# MLP decoder
self.mlp_bias_on: bool = True
self.mlp_leaky_relu: bool = False
self.geo_mlp_level: int = 1
self.geo_mlp_hidden_dim: int = 64
self.sem_mlp_level: int = 1
self.sem_mlp_hidden_dim: int = 64
self.color_mlp_level: int = 1
self.color_mlp_hidden_dim: int = 64
# NOTE: For achieving better reconstruction results, you can increase the size of the MLPs,
# or the feature dim of the neural point latent feature,
# or the iteration number for mapping
# or decrease the voxel size (resolution) for neural points
self.decoder_freezed: bool = False
self.freeze_after_frame: int = 40  # if the decoder model is not loaded, it would be trained and freezed after such frame number # TODO: change to based on moving frames

# positional encoding related [not used]
self.use_gaussian_pe: bool = False # use Gaussian Fourier or original log positional encoding
self.pos_encoding_freq: int = 200 # 200
self.pos_encoding_band: int = 0 # if 0, without encoding
self.pos_input_dim: int = 3
self.pos_encoding_base: int = 2

# training (mapping) loss
# the main loss type, select from the sample sdf loss ('bce', 'l1', 'l2', 'zhong') 
self.main_loss_type: str = 'bce'
self.sigma_sigmoid_m: float = 0.1 # better to be set according to the noise level (used only for BCE loss as the sigmoid scale factor)
self.logistic_gaussian_ratio: float = 0.55 # the factor ratio for approximize a Gaussian distribution using the derivative of logistic function

self.proj_correction_on: bool = False ## conduct projective distance correction based on the sdf gradient or not, True does not work well 
self.loss_weight_on: bool = False  # if True, the weight would be given to the loss, if False, the weight would be used to change the sigmoid's shape
self.behind_dropoff_on: bool = False  # behind surface drop off weight
self.dist_weight_on: bool = True  # weight decrease linearly with the measured distance, reflecting the measurement noise
self.dist_weight_scale: float = 0.8 # weight changing range [0.6, 1.4]

self.numerical_grad: bool = True # use numerical SDF gradient as in the paper Neuralangelo for the Ekional regularization during mapping
self.gradient_decimation: int = 10 # use just a part of the points for the ekional loss when using the numerical grad, save computing time
self.num_grad_step_ratio: float = 0.2 # step as a ratio of the nerual point resolution, length = num_grad_step_ratio * voxel_size_m

self.ekional_loss_on: bool = True # Ekional regularization (default on)
self.ekional_add_to: str = 'all' # select from 'all', 'surface', 'freespace', the samples used for Ekional regularization
self.weight_e: float = 0.5

self.consistency_loss_on: bool = False # gradient consistency (smoothness) regularization (default off)
self.weight_c: float = 0.5 # weight for consistency loss, don't mix with the color weight 
self.consistency_count: int = 1000
self.consistency_range: float = 0.05 # the neighborhood points would be randomly select within the radius of consistency_range (unit: m)

self.weight_s: float = 1.0  # weight for semantic classification loss
self.weight_i: float = 1.0  # weight for color or intensity regression loss

# optimizer
self.mapping_freq_frame: int = 1
self.iters: int = 12 # training iterations per frame. to have a better reconstruction results, you need to set a larger iters, a smaller lr
self.init_iter_ratio: int = 40 # train init_iter_ratio x iters for the first frame to kick the SLAM off
self.opt_adam: bool = True  # use adam (default) or sgd as the gradient descent optimizer
self.bs: int = 16384 # batch size
self.lr: float = 0.01 # learning rate for map parameters and MLP
self.lr_pose: float = 1e-4 # learning rate for poses during bundle adjustment
self.lr_ba_map: float = 0.01 # learning rate for map during bundle adjustment
self.weight_decay: float = 0.0 # weight_decay is only applied to the latent codes for the l2 regularization
self.adam_eps: float = 1e-15
self.adaptive_iters: bool = False # adptive map optimization iterations on (train for fewer iterations when there's not much new information to learn)
self.new_sample_ratio_less: float = 0.02 # if smaller than this ratio, we think there's not much new information collected, train less
self.new_sample_ratio_more: float = 0.15 # if larger than this ratio, we think there are a lot new observations to learn, train more
self.new_sample_ratio_restart: float = 0.3 # if larger than this ratio, we think maybe tracking is lost, train much more

# local bundle adjustment (ba)  
self.ba_freq_frame: int = 0 # frame frequency for conducting ba
self.ba_frame: int = 50 # window size for ba
self.ba_iters: int = 80 # iteration number for ba optimization
self.ba_bs: int = 16384 # batch size for ba optimization
```

**5. Tracking and Loss**

```
# tracking (odometry estimation)
self.track_on: bool = False
self.photometric_loss_on: bool = False # add the color (or intensity) [photometric loss] to the tracking loss
self.photometric_loss_weight: float = 0.01 # weight for the photometric loss in tracking
self.consist_wieght_on: bool = True # weight for color (intensity) consistency for the measured and queried value
self.source_vox_down_m: float = 0.8 # downsample voxel resolution for source point cloud
self.uniform_motion_on: bool = True # use uniform motion (constant velocity) model for the transformation inital guess
self.reg_min_grad_norm: float = 0.5 # min norm of SDF gradient for valid source point
self.reg_max_grad_norm: float = 2.0 # max norm of SDF gradient for valid source point
self.track_mask_query_nn_k: int = self.query_nn_k # during tracking, a point without nn_k neighbors would be regarded as invalid
self.max_sdf_ratio: float = 5.0 # ratio * surface_sample sigma
self.max_sdf_std_ratio: float = 1.0 # ratio * surface_sample sigma
self.reg_dist_div_grad_norm: bool = False # divide the sdf by the sdf gradient's norm for fixing overshoting or not
self.reg_GM_dist_m: float = 0.3 # GM scale for SDF residual 
self.reg_GM_grad: float = 0.1 # GM scale for SDF gradient anomaly 
# for GM scale, the smaller the value, the smaller the weight would be (give even smaller weight to the outliers)
self.reg_lm_lambda: float = 1e-4 # lm damping factor
self.reg_iter_n: int = 50 # maximum iteration number for registration
self.reg_term_thre_deg: float = 0.01 # iteration termination criteria for rotation 
self.reg_term_thre_m: float = 0.001  # iteration termination criteria for translation
self.eigenvalue_check: bool = True # use eigen value of Hessian matrix for degenaracy check
self.eigenvalue_ratio_thre: float = 0.005 # threshold for eigenvalue ratio
self.final_residual_ratio_thre: float = 0.6
```

**6. Loop Closure and Pose Graph Opitimization**

```
# loop closure detection
self.global_loop_on: bool = True # global loop detection using context descriptor
self.local_map_context: bool = True # use local map context or scan context for loop closure description (使用局部地圖的特徵作為「指紋」來比對是否回到原點。)
self.loop_with_feature: bool = True # encode neural point feature in the context
self.min_loop_travel_dist_ratio: float = 4.0 # accumulated travel distance should be larger than this ratio * local map radius to be considered as an valid candidate
self.local_map_context_latency: int = 5 # only used for local map context, wait for local_map_context_latency before descriptor generation for enough training of the new observations
self.loop_local_map_by_travel_dist: bool = True # determine the local map for context description according to travel distance difference or frame (time) difference
self.loop_local_map_time_window: int = 100 # unit: frame, only be valid when self.loop_local_map_by_travel_dist = False
self.local_loop_dist_thre: float = 5.0 # unit: m, find local loop within this distance
self.context_shape = [20, 60] # ring, sector count for the descriptor
self.npmc_max_dist: float = 60.0  # max distance for the neural point map context descriptor
self.context_num_candidates: int = 1 # select the best K candidates after comparing the ring key for further checking
self.context_cosdist_threshold: float = 0.2 # cosine distance threshold for a candidate loop (回環判定的閾值（餘弦距離）。如果發現回環太難觸發，可以稍微調大此值（例如 0.25 或 0.3）。)
self.context_virtual_side_count: int = 5 # augment context_virtual_side_count virtual sensor positions on each side of the actual sensor position
self.context_virtual_step_m: float = 2.0 # voxel_size_m * 4.0 
self.loop_z_check_on: bool = False # check the z axix difference of the found loop frames to deal with the potential abitary issue in a multi-floor building
self.loop_dist_drift_ratio_thre: float = 5.0 # find the loop candidate only within the distance of (loop_dist_drift_ratio_thre * drift)

# pose graph optimization
self.pgo_on: bool = True # 開啟位姿圖優化
self.pgo_freq: int = 30 # frame interval for detecting loop closure and conducting pose graph optimization after a successful loop correction (每 30 幀檢查一次回環。)
self.pgo_with_isam: bool = True # use isam incremental optimization or lm batch optimization
self.pgo_max_iter: int = 50 # maximum number of iterations
self.pgo_with_pose_prior: bool = False # use the pose prior or not during the pgo
self.pgo_tran_std: float = 0.04 # m (對於PGO平移量的權重設定)
self.pgo_rot_std: float = 0.01 # deg (對於PGO旋轉量的權重設定)
self.use_reg_cov_mat: bool = False # use the covariance matrix directly calculated by the registration for pgo edges or not
self.pgo_error_thre_frame: float = 500.0 # the maximum error for rejecting a wrong edge (per frame)
self.pgo_merge_map: bool = True # merge the map (neural points) or not after the pgo (or we keep all the history neural points) (當閉環發生並修正軌跡後，是否將原本錯位的神經點雲「強行合併」。開啟後地圖會看起來更乾淨。)
self.rehash_with_time: bool = True # Do the rehashing based on smaller time difference or higher point stability

```

**7. Mesh and Marching Cubes**

```
# eval
self.wandb_vis_on: bool = False # monitor the training on weight and bias or not
self.silence: bool = True # print log in the terminal or not
self.o3d_vis_on: bool = False # visualize the mesh in-the-fly using o3d visualzier or not [press space to pasue/resume]
self.o3d_vis_raw: bool = False # visualize the raw point cloud or the weight source point cloud
self.log_freq_frame: int = 2000 # save the result log per x frames
self.mesh_default_on: bool = False
self.mesh_freq_frame: int = 20  # do the reconstruction per x frames
self.sdf_default_on: bool = False # visualize the SDF slice or not
self.sdfslice_freq_frame: int = 1 # visualize the SDF slice per x frames
self.vis_sdf_slice_v: bool = False # also visualize the vertical SDF slice or not (default only horizontal slice)
self.sdf_slice_height: float = -1.0 # initial height of the horizontal SDF slice (m) in sensor frame
self.vis_sdf_res_m: float = 0.2 # resolution for the SDF slice for visualization (m)
self.eval_traj_align: bool = True # do the SE3 alignment of the trajectory when evaluating the absolute error

# mesh reconstruction, marching cubes related
self.mc_res_m: float = 0.3 # resolution for marching cubes (生成 3D 模型 (Mesh) 時的解析度)
self.pad_voxel: int = 3 # pad x voxels on each side
self.skip_top_voxel: int = 2 # slip the top x voxels (mainly for visualization indoor, remove the roof)
self.mc_mask_on: bool = True # use mask for marching cubes to avoid the artifacts
self.mesh_min_nn: int = 8  # The minimum number of the neighbor neural points for a valid SDF prediction for meshing, too small would cause some artifacts (more complete but less accurate), too large would lead to lots of holes (more accurate but less complete) (一個區域至少要有幾個神經點才生成 Mesh。調大這個值可以過濾掉飄浮的噪點碎片，但太大的話地圖會有洞。)
self.min_cluster_vertices: int = 300 # if a connected's vertices number is smaller than this value, it would get filtered (as a postprocessing to filter outliers)
self.keep_local_mesh: bool = False # keep the local mesh in the visualizer or not (don't delete them could cause a too large memory consumption)
self.infer_bs: int = 4096 # batch size for inference
```

**8. Visualization and Output**
```
# visualization
self.local_map_default_on: bool = True
self.neural_point_map_default_on: bool = True
self.mesh_vis_normal: bool = False # normal colorization
self.vis_frame_axis_len: float = 0.8 # sensor frame axis length, for visualization, unit: m
self.vis_point_size: int = 2 # point size for visualization in o3d
self.sensor_cad_path = None # the path to the sensor cad file, "./cad/ipb_car.ply" for visualization

# gui 
self.visualizer_split_width_ratio: float = 0.7 # the width ratio of the visualizer split window

# result saving settings
self.save_map: bool = True # save the neural point map model and decoders or not
self.save_merged_pc: bool = True # save the merged point cloud pc or not
self.save_mesh: bool = True # save the reconstructed mesh map or not

# ROS related 
self.run_with_ros: bool = False
self.publish_np_map: bool = True # only for Rviz visualization, publish neural point map
self.publish_np_map_down_rate_list = [11, 23, 37, 53, 71, 89, 97, 113, 131, 151] # prime number list, downsampling for boosting neural point map pubishing speed 
self.republish_raw_input: bool = False # publish the raw input point cloud or not
self.timeout_duration_s: int = 30 # in seconds, exit after receiving no topic for x seconds
```
</details>


<a name="nvidia_container_toolkit_failure"></a>
## NVIDIA Container Toolkit Failure
Due to the container fails to recognize the GPU hardware. Please try to open CMD and insert following commands:
```
# 1. 加入 NVIDIA 套件來源
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/stderr > /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2. 安裝工具
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

Tell docker how to use Nvidia Runtime.
```
# 1. 自動設定 Docker 設定檔
sudo nvidia-ctk runtime configure --runtime=docker

# 2. 重啟 Docker 服務
sudo systemctl restart docker
```

Try simple test to make sure docker is able to attach to the GPU.
```
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu22.04 nvidia-smi
```

If success, you will see the information about GPU. Otherwise, it will return error to inform you that there's still trouble between host driver and docker connection.

Later on, please activate the docker nvironment.
```
./start_docker.sh
```


<a name="pros"></a>
## Pros
1. MIT License (commercial application enabled).
2. Prcoess SLAM fast.


<a name="cons"></a>
## Cons
1. Fail to SLAM in freeway tunnel.
2. Need to fine tune several parameters.
3. Output point cloud map no intensity info.


<a name="reference"></a>
## Reference
1. PIN-SLAM https://github.com/PRBonn/PIN_SLAM.git
