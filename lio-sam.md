# LIO-SAM

## Introduction
A real-time lidar-inertial odometry package. 

<!-- TABLE OF CONTENTS -->
<details open="open" style='padding: 10px; border-radius:5px 30px 30px 5px; border-style: solid; border-width: 1px;'>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#install">Install</a>
    </li>
    <li>
      <a href="#docker">Docker</a>
    </li>
    <li>
      <a href="#run">Run</a>
    </li>
    <li>
      <a href="#i/o">I/O</a>
    </li>
    <li>
      <a href="#frame_transformation">Frame Transformation</a>
    </li>
    <li>
      <a href="#config">Config</a>
    </li>
    <li>
      <a href="#build">Build</a>
    </li>
    <li>
      <a href="#save_result">Save Result</a>
    </li>
    <li>
      <a href="#rosbag_version_conversion">Rosbag Version Conversion</a>
    </li>
    <li>
      <a href="#reference">Reference</a>
    </li>
  </ol>
</details>


<a name="install"></a>
## Install
**ROS1 version**
```
cd ~/catkin_ws/src
git clone https://github.com/TixiaoShan/LIO-SAM.git
cd ..
catkin_make
```

**ROS2 version**
```
cd ~/ros2_ws/src
git clone https://github.com/TixiaoShan/LIO-SAM.git
cd LIO-SAM
git checkout ros2
cd ..
colcon build
```


<a name="docker"></a>
## Docker
**ROS1 version**
1. Build docker image.
```
docker build -t liosam-kinetic-xenial .
```
2. Start the container.
```
docker run --init -it -d \
  -v /etc/localtime:/etc/localtime:ro \
  -v /etc/timezone:/etc/timezone:ro \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /<your_pc_data_path>:/root/catkin_ws/src/LIO-SAM/data\
  -e DISPLAY=$DISPLAY \
  liosam-kinetic-xenial \
  bash
```

**ROS2 version**
1. Build docker image.
```
docker build -t liosam-humble-jammy .
```
2. Start the container.
```
docker run --init -it -d \
  --name liosam-humble-jammy-container \
  -v /etc/localtime:/etc/localtime:ro \
  -v /etc/timezone:/etc/timezone:ro \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /<your_pc_data_path>:/root/ros2_ws/src/LIO-SAM/data\
  -e DISPLAY=$DISPLAY \
  --runtime=nvidia --gpus all \
  liosam-humble-jammy \
  bash
```


<a name="run"></a>
## Run
**ROS1**
1. Run the launch file.
```
roslaunch lio_sam run.launch
```
2. Play existing bag files.
```
rosbag play <your_bag.bag> -r 3
```

**ROS2**
1. Run the launch file.
```
ros2 launch lio_sam run.launch.py
```
2. Play existing bag files. Add "--clock" to run the code with bag file recorded time, otherwise it will use wall time (unixtime now).
```
ros2 bag play <your_bag_folder> --clock
```
Note: If you want to speed up or down the rosbag play, then add "-r <speed>" after the "ros2 bag play".


<a name="i/o"></a>
## I/O
1.Input
- Bag file corresponding to ROS1 or ROS2. IMU and LiDAR are must, while GNSS and Odometer are optional.

Note: To check the content within each sensor, try the following command:
```
# ros2 topic list -v   # show all the topic name ongoing
# ros2 topic echo <topic_name> 
ros2 topic echo /points_raw    # add --once to show just one frame of data
```

2.Output
- CornerMap.pcd
- GlobalMap.pcd
- SurfMap.pcd
- trajectory.pcd
- transformations.pcd


<a name="frame_transformation"></a>
## Frame Transformation
1. Frames Introduction

| Name | Definition | Source |
|------|----------|------|
| map | psudo global map origin (static frame) | lio_sam_mapOptimization |
| odom | 里程計原點：機器人剛啟動時的位置。它會隨著 Lidar 里程計的累計誤差而與 map 產生偏離。 | lio_sam_imuPreintegration |
| odom_mapping | SLAM 內部參考點：LIO-SAM 算法內部計算用的虛擬中心。在您的配置中，我們將它視為導航算法的邏輯起點。 | LIO-SAM node |
| base_link | robot actual body center，通常設定在機器人選轉中心或底盤中心 |  |
| chassis_link | 底盤實體座標：代表機器人的金屬底盤結構。通常 base_link 與 chassis_link 位置重合，但 chassis_link 更偏向描述物理結構。 | URDF model |
| lidar_link | origin: center of LiDAR data | 靜態 TF 或 URDF |
| imu_link | origin: center of IMU | 靜態 TF 或 URDF |
| navsat_link | origin: GPS antenna | 靜態 TF 或 URDF |

2. Nodes for Frame Transformation

```
Node(
    package='tf2_ros',
    executable='static_transform_publisher',
    # 請根據你的機器人實際安裝位置調整，若不確定先設全 0+
    name='tf_odom_mapping_navsat',
    arguments=['--x', '0', '--y', '0', '--z', '0', '--yaw', '0', '--pitch', '0', '--roll', '0', '--frame-id', 'odom', '--child-frame-id', 'odom_mapping'],
    parameters=[{'use_sim_time': False}], # use simulation time or not
    output='screen'
)
```


<a name="config"></a>
## Config
Make sure the the topic name (ex: "pointCloudTopic", "imuTopic"), frames (ex: "lidarFrame"), sensor settings, and EOP are corresponded with the bag file metadata.yaml.
<details> 
  <summary>(click to open)</summary>
  
```
/**:
  ros__parameters:

    # Topics
    pointCloudTopic: "/points_raw"                   # Point cloud data
    imuTopic: "/imu_raw"                        # IMU data
    odomTopic: "odometry/imu"                    # IMU pre-preintegration odometry, same frequency as IMU
    gpsTopic: "odometry/gpsz"                    # GPS odometry topic from navsat, see module_navsat.launch file

    # Frames
    lidarFrame: "velodyne"
    baselinkFrame: "base_link"
    odometryFrame: "odom"
    mapFrame: "map"

    # GPS Settings
    useImuHeadingInitialization: false           # if using GPS data, set to "true"
    useGpsElevation: false                       # if GPS elevation is bad, set to "false"
    gpsCovThreshold: 2.0                         # m^2, threshold for using GPS data
    poseCovThreshold: 25.0                       # m^2, threshold for using GPS data

    # Export settings
    savePCD: true                               # https://github.com/TixiaoShan/LIO-SAM/issues/3
    savePCDDirectory: "/Downloads/LOAM/"         # in your home folder, starts and ends with "/". Warning: the code deletes "LOAM" folder then recreates it. See "mapOptimization" for implementation

    # Sensor Settings
    sensor: velodyne                               # lidar sensor type, either 'velodyne', 'ouster' or 'livox'
    N_SCAN: 16                                   # number of lidar channels (i.e., Velodyne/Ouster: 16, 32, 64, 128, Livox Horizon: 6)
    Horizon_SCAN: 1800                            # lidar horizontal resolution (Velodyne:1800, Ouster:512,1024,2048, Livox Horizon: 4000)
    downsampleRate: 1                            # default: 1. Downsample your data if too many
    # points. i.e., 16 = 64 / 4, 16 = 16 / 1
    lidarMinRange: 1.0                           # default: 1.0, minimum lidar range to be used
    lidarMaxRange: 1000.0                        # default: 1000.0, maximum lidar range to be used

    # IMU Settings
    imuAccNoise: 3.9939570888238808e-03
    imuGyrNoise: 1.5636343949698187e-03
    imuAccBiasN: 6.4356659353532566e-05
    imuGyrBiasN: 3.5640318696367613e-05

    imuGravity: 9.80511
    imuRPYWeight: 0.01

    extrinsicTrans:  [0.0, 0.0, 0.0]
    extrinsicRot:    [1.0,  0.0,  0.0,
                      0.0, -1.0,  0.0,
                      0.0,  0.0, -1.0]
    extrinsicRPY:    [1.0,  0.0,  0.0,
                      0.0, -1.0,  0.0,
                      0.0,  0.0, -1.0]

    # LOAM feature threshold
    edgeThreshold: 1.0
    surfThreshold: 0.1
    edgeFeatureMinValidNum: 10
    surfFeatureMinValidNum: 100

    # voxel filter paprams
    odometrySurfLeafSize: 0.4                     # default: 0.4 - outdoor, 0.2 - indoor
    mappingCornerLeafSize: 0.2                    # default: 0.2 - outdoor, 0.1 - indoor
    mappingSurfLeafSize: 0.4                      # default: 0.4 - outdoor, 0.2 - indoor

    # robot motion constraint (in case you are using a 2D robot)
    z_tollerance: 1000.0                          # meters
    rotation_tollerance: 1000.0                   # radians

    # CPU Params
    numberOfCores: 4                              # number of cores for mapping optimization
    mappingProcessInterval: 0.15                  # seconds, regulate mapping frequency

    # Surrounding map
    surroundingkeyframeAddingDistThreshold: 1.0   # meters, regulate keyframe adding threshold
    surroundingkeyframeAddingAngleThreshold: 0.2  # radians, regulate keyframe adding threshold
    surroundingKeyframeDensity: 2.0               # meters, downsample surrounding keyframe poses   
    surroundingKeyframeSearchRadius: 50.0         # meters, within n meters scan-to-map optimization
    # (when loop closure disabled)

    # Loop closure
    loopClosureEnableFlag: true
    loopClosureFrequency: 1.0                     # Hz, regulate loop closure constraint add frequency
    surroundingKeyframeSize: 50                   # submap size (when loop closure enabled)
    historyKeyframeSearchRadius: 15.0             # meters, key frame that is within n meters from
    # current pose will be considerd for loop closure
    historyKeyframeSearchTimeDiff: 30.0           # seconds, key frame that is n seconds older will be
    # considered for loop closure
    historyKeyframeSearchNum: 25                  # number of hostory key frames will be fused into a
    # submap for loop closure
    historyKeyframeFitnessScore: 0.3              # icp threshold, the smaller the better alignment

    # Visualization
    globalMapVisualizationSearchRadius: 1000.0    # meters, global map visualization radius
    globalMapVisualizationPoseDensity: 10.0       # meters, global map visualization keyframe density
    globalMapVisualizationLeafSize: 1.0           # meters, global map visualization cloud density
```
</details>

If you want to check what's the value of each parameters are, try the following command during the rosbag play:
```
# ros2 param get <node> <parameter>
ros2 param get /lio_sam_imageProjection extrinsicRot
```


<a name="build"></a>
## Build
If modified any code in src, please build the project first.
```
cd ~/ros2_ws
rm -rf build/ install/ log/
colcon build --packages-select lio_sam
```

<a name="save_result"></a>
## Save Result
**ROS1 version**
```
# for example: rosservice call <service> <resolution> <destination>
rosservice call /lio_sam/save_map 0.2 "/Downloads/LOAM/"
```

**ROS2 version**

For now, only resolution and destination variables can be set.
```
# for example: ros2 service call <service> "{resolution: <resolution>, destination: <destination>}"
ros2 service call /lio_sam/save_map lio_sam/srv/SaveMap "{resolution: 0.2, destination: /ros2_ws/src/LIO-SAM/output/garden}" 
```


<a name="rosbag_version_conversion"></a>
## Rosbag Version Conversion
For rosbag1 to rosbag2, please enter the environment of ros1 and insert the following command.
```
pip3 install rosbags>=0.9.11  # for first time only
rosbags-convert --src <ros1.bag> --dst <ros2_bag_folder>
```

After conversion, open the metadata.yaml in the bag folder, delete the description of all
```
type_description_hash: xxxxxxxxx
```
And modified the part of "offered_qos_profiles". If the "offered_qos_profiles" involves some statements, just delete the statements and replace with "".
```
# original version: offered_qos_profiles: []
offered_qos_profiles: ""    # modified version
```
Finally, change the version: 4 to version: 9.


<a name="reference"></a>
## Reference
1. LIO-SAM: https://github.com/TixiaoShan/LIO-SAM/tree/ros2?tab=readme-ov-file#run-the-package
2. Rosbag version conversion: https://docs.ros.org/en/noetic/api/ov_core/html/dev-ros1-to-ros2.html
