# KISS-ICP

## Introduction
KISS-ICP is short for "keep it small and simple", which is a SLAM method based on point-to-point ICP, to conduct LiDAR Odometry.

<!-- TABLE OF CONTENTS -->
<details open="open" style='padding: 10px; border-radius:5px 30px 30px 5px; border-style: solid; border-width: 1px;'>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#i/o">I/O</a>
    </li>
    <li>
      <a href="#i/o">RUN</a>
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


<a name="i/o"></a>
## I/O
### 1. Input Data
Only LiDAR data is needed, it supports formats include bin, pcd, ply, xyz, etc.

### 2. Output Data
- **'TUM' trajectory files**

Every row has 8 entries containing timestamp (in seconds), position and orientation (as quaternion) with each value separated by a space:
```
timestamp x y z q_x q_y q_z q_w
```
More infos: https://vision.in.tum.de/data/datasets/rgbd-dataset/file_formats

NOTE: the TUM visual inertial dataset provides groundtruth data in a similar format like the EuRoC MAV dataset. It is not using the TUM RGB-D trajectory format for the groundtruth. With evo 1.13.4 or later you can use the euroc mode also for this dataset.

  
- **'KITTI' pose files**

This is actually not a real trajectory format because it has no timestamps - it only contains the poses in a text file. This means that you have to be careful when you want to compare two files in this format with a metric because the number of poses must be exactly the same.

Every row of the file contains the first 3 rows of a 4x4 homogeneous pose matrix (SE(3) matrix) flattened into one line, with each value separated by a space. For example, this pose matrix:

```
a b c d
e f g h
i j k l
0 0 0 1
```
will appear in the file as the row:

```
a b c d e f g h i j k l
```
More infos: http://www.cvlibs.net/datasets/kitti/eval_odometry.php

![kiiti_and_tum_coordinate info](https://github.com/ashley17happy17/slam/blob/3438fc6d5f6d6bb567ad33c52bc5d5f7430c0c5b/img/kitti_and_tum_format_info.png)


<a name="run"></a>
## RUN
If the code runs in virtual environment, please source the following shell script first. (optional)
```
#!/bin/bash
source .venv/bin/activate
echo "KISS-ICP Activated！"
```

After the virtual environment is activated, please enter following command.
```
kiss_icp_pipeline --visualize [path_to_your_lidar_data_folder]
```


<a name="pros"></a>
## Pros
1. It supports python and ROS.
2. It can commit registration in several kinds of scenarios, without tuning any parameters.


<a name="cons"></a>
## Cons
1. LiDAR Odometry will fail in freeway scenario.
2. No point cloud map output.


<a name="reference"></a>
## Reference
1. KISS-ICP: https://github.com/PRBonn/kiss-icp.git
2. evo: https://github.com/MichaelGrupp/evo.git
