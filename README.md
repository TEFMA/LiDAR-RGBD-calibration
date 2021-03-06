# LiDAR-RGBD-calibration
RoboSense LiDAR (Helios 5515) and RGBD camera (Intel RealSense D435i) calibration method.
modified from https://github.com/ankitdhall/lidar_camera_calibration

## 1. 相关代码仓库
(1) 激光雷达仓库：[rslidar_sdk](https://github.com/RoboSense-LiDAR/rslidar_sdk)  
(2) 激光雷达与相机标定仓库：[LiDAR-Camera Calibration using 3D-3D Point correspondences](https://github.com/ankitdhall/lidar_camera_calibration)  
(3) 激光雷达格式转换仓库：[RS to Velodyne](https://github.com/HViktorTsoi/rs_to_velodyne)
## 2. 相关仓库代码修改
### 2.1 激光雷达仓库
使用ROS-catkin进行ros包编译:  
(1) 在/home/slam/下新建一个名为catkin_ws的文件夹作为工作空间，然后在其中新建一个名为src的文件夹, 将rslidar_sdk工程放入src文件夹内  
(2) 打开工程内的CMakeLists.txt文件，将文件顶部的**set(COMPILE_METHOD ORIGINAL)**改为**set(COMPILE_METHOD CATKIN)**
```
#=======================================
# Compile setup (ORIGINAL,CATKIN,COLCON)
#=======================================
set(COMPILE_METHOD CATKIN)
```
(3) 将rslidar_sdk工程目录下的package_ros1.xml文件复制一份到当前目录下，并重命名为package.xml  
(4) 更改配置文件中的雷达型号，否则会提示zero_points，rviz为空白:  
打开rslidar_sdk/config/config.yaml，将**lidar_type: RS128**改为**lidar_type: RSHLIOS**  
(5) 在工作空间目录catkin_ws下，执行以下命令即可编译&运行 
```
catkin_make
source devel/setup.bash
roslaunch rslidar_sdk start.launch
```

## 2.2 激光雷达格式转换仓库
激光雷达与相机标定仓库依赖于有环的激光雷达点云数据格式，所以需要将雷达点云格式变换  
激光雷达格式转换仓库提供了多数的rslidar转velodyne格式的雷达类型，将其变换为有ring的数据格式。但该代码仓没有对32线激光雷达进行配置，需要修改[32线类型激光雷达处理的代码](https://github.com/HViktorTsoi/rs_to_velodyne/blob/c7125ffe8616d26a74f45f91299824de0167b63d/src/rs_to_velodyne.cpp#L115)
```  
// remap ring id
if (pc->height == 32){
    new_point.ring = RING_ID_MAP_32[point_id % pc->height];
}
else if (pc->height == 16) {
    new_point.ring = RING_ID_MAP_16[point_id / pc->width];
} else if (pc->height == 128) {
    new_point.ring = RING_ID_MAP_RUBY[point_id % pc->height];
}
```
以及添加[ring映射方法](https://github.com/HViktorTsoi/rs_to_velodyne/blob/c7125ffe8616d26a74f45f91299824de0167b63d/src/rs_to_velodyne.cpp#L25)
```
static int RING_ID_MAP_32[] = {
        31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0
};
```

## 2.3 激光雷达与相机标定仓库
(1) 参照[Setup and Installation](https://github.com/ankitdhall/lidar_camera_calibration/wiki/Welcome-to-%60lidar_camera_calibration%60-Wiki!)对ros包进行安装配置  
(2) 配置后需要对订阅话题进行对应修改：  
打开lidar_camera_calibration/launch/find_transform.launch文件：  
a. 对<!-- ArUco mapping -->进行解注释(否则将导致标定程序在标定信息读取之后卡住)：  
```
<?xml version="1.0"?>
<launch>
  <!-- <param name="/use_sim_time" value="true"/> -->
  
  <!-- ArUco mapping -->
  <node pkg="aruco_mapping" type="aruco_mapping" name="aruco_mapping" output="screen">
    <remap from="/image_raw" to="/camera/color/image_raw"/>

    <param name="calibration_file" type="string" value="$(find aruco_mapping)/data/zed_left_uurmi.ini" /> 
    <param name="num_of_markers" type="int" value="2" />
    <param name="marker_size" type="double" value="0.125"/>
    <param name="space_type" type="string" value="plane" />
    <param name="roi_allowed" type="bool" value="false" />
  </node>  

  <rosparam command="load" file="$(find lidar_camera_calibration)/conf/lidar_camera_calibration.yaml" />
  <node pkg="lidar_camera_calibration" type="find_transform" name="find_transform" output="screen">
  </node>
</launch>
```
b. 将remap字段按照实际相机的图像topic(在实际使用的相机ros驱动中可配置)进行修改，这里realsense的topic是/camera/color/image_raw  
(4) 修改lidar_camera_calibration/dependencies/aruco_mapping/data/zed_left_uurmi.ini配置文件中对应的相机参数(实际使用的realsense相机参数)  
(5) 修改lidar_camera_calibration/config/lidar_camera_calibration.yaml文件中的相机和雷达对应的topic  
(6) realsense相机相关代码修改:  
原代码仓库默认使用zed相机并使用灰度图进行图像显示，而realsense发布的topic为彩色图像，需要在lidar_camera_calibration/dependencies/aruco_mapping/src/aruco_mapping.cpp文件中对[sensor_msgs::image_encodings](https://github.com/ankitdhall/lidar_camera_calibration/blob/13d52954fa18ee3eef86272757555a28a2532c71/dependencies/aruco_mapping/src/aruco_mapping.cpp#L167)其编码方式进行更改，将MONO8修改为BGR8，否则会出现cvtColor报错问题  
(7) 修改lidar_camera_calibration/include/lidar_camera_calibration/PreprocessUtils.h文件中[Line200](https://github.com/ankitdhall/lidar_camera_calibration/blob/13d52954fa18ee3eef86272757555a28a2532c71/include/lidar_camera_calibration/PreprocessUtils.h#L200)，由rings(16)修改为rings(32)
(8) 如果雷达点云不是带有ring的格式将报错Failed to find match for field 'ring'.需要进行点云格式变换,见2.2

