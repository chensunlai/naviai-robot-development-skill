# Sensor Access

Available inputs include the head RealSense, MID360, LIO registered clouds, upper-limb and hand state, finger pressure, and wrist force. Native ROS is the usual choice for images and point clouds. rosbridge is suitable for lower-rate joint, pose, and state data.

## Contents

- [Active Camera Data](#active-camera-data)
- [Lidar, IMU, and Localization Output](#lidar-imu-and-localization-output)
- [Upper-Limb, Hand, and Force Data](#upper-limb-hand-and-force-data)
- [Reserved Names Without Active Publishers](#reserved-names-without-active-publishers)
- [Native ROS Example](#native-ros-example)
- [rosbridge Pose Example](#rosbridge-pose-example)
- [Read-Only Availability Checks](#read-only-availability-checks)

## Active Camera Data

| Data | Topic | Type | Publisher |
|---|---|---|---|
| Head color | `/zj_humanoid/sensor/realsense_head/color/image_raw` | `sensor_msgs/Image` | head RealSense node |
| Compressed color | `/zj_humanoid/sensor/realsense_head/color/image_raw/compressed` | `sensor_msgs/CompressedImage` | same |
| Head depth | `/zj_humanoid/sensor/realsense_head/depth/image_rect_raw` | `sensor_msgs/Image` | same |
| Depth aligned to color | `/zj_humanoid/sensor/realsense_head/aligned_depth_to_color/image_raw` | `sensor_msgs/Image` | same |
| Color intrinsics | `/zj_humanoid/sensor/realsense_head/color/camera_info` | `sensor_msgs/CameraInfo` | same |
| Depth intrinsics | `/zj_humanoid/sensor/realsense_head/depth/camera_info` | `sensor_msgs/CameraInfo` | same |

For RGB-depth registration, use `aligned_depth_to_color` with the corresponding CameraInfo rather than inferring intrinsics from image dimensions.

## Lidar, IMU, and Localization Output

| Data | Topic | Type | Publisher |
|---|---|---|---|
| MID360 raw cloud | `/livox/lidar` | `livox_ros_driver2/CustomMsg` | `/livox_lidar_publisher2` |
| MID360 IMU | `/livox/imu` | `sensor_msgs/Imu` | `/livox_lidar_publisher2` |
| Registered body-frame cloud | `/cloud_registered_body` | `sensor_msgs/PointCloud2` | `/laserMapping` |
| Raw LIO odometry | `/odom_imu` | `nav_msgs/Odometry` | `/laserMapping` |
| Navigation localization | `/zj_humanoid/navigation/odom_info` | `nav_msgs/Odometry` | `/laserMapping` |
| Local obstacle map | `/zj_humanoid/navigation/local_map` | `navigation/LocalMap` | `/test_grid` |

`/livox/lidar` uses the Livox custom format with `point_num` and `points[]`. `/cloud_registered_body` provides standard `PointCloud2`, but only while a map is loaded and `/laserMapping` runs.

## Upper-Limb, Hand, and Force Data

| Data | Topic | Type | Publisher |
|---|---|---|---|
| Upper-limb joints | `/zj_humanoid/upperlimb/joint_states` | `sensor_msgs/JointState` | Pico upper-limb node |
| Upper-limb work state | `/zj_humanoid/upperlimb/uplimb_state` | `upperlimb/UplimbState` | same |
| Left/right TCP pose | `/zj_humanoid/upperlimb/tcp_pose/left_arm`, `right_arm` | `upperlimb/Pose` | same |
| Dual-arm TCP speed | `/zj_humanoid/upperlimb/tcp_speed/dual_arm` | `upperlimb/TcpSpeed` | same |
| Hand joints | `/zj_humanoid/hand/joint_states` | `sensor_msgs/JointState` | Pico hand node |
| Left/right finger pressure | `/zj_humanoid/hand/finger_pressures/left`, `right` | `hand/PressureSensor` | same |
| Left/right wrist wrench | `/zj_humanoid/hand/wrist_force_sensor/left`, `right` | `geometry_msgs/WrenchStamped` | same |

`upperlimb/Pose` includes `position`, quaternion, `rpy_rad`, and `rpy_deg`. Preserve `header.frame_id` in application data.

## Reserved Names Without Active Publishers

| Topic Group | Observed State |
|---|---|
| `/zj_humanoid/sensor/CAM_A/image_raw` through `CAM_D/image_raw` | no publisher |
| `/zj_humanoid/sensor/left_wrist/image_raw`, `right_wrist/image_raw` | no publisher; type unresolved |
| `/zj_humanoid/sensor/realsense_down/*` | no publisher |

Monitoring subscriptions can make these names visible. Determine availability from publisher count and first-message timeout, not `rostopic list` alone.

## Native ROS Example

```python
#!/usr/bin/env python3
import rospy

from livox_ros_driver2.msg import CustomMsg
from sensor_msgs.msg import Image
from upperlimb.msg import Pose


def on_image(msg):
    rospy.loginfo_throttle(2.0, 'rgb: %dx%d %s', msg.width, msg.height, msg.encoding)


def on_lidar(msg):
    rospy.loginfo_throttle(2.0, 'mid360 points: %d', msg.point_num)


def on_left_tcp(msg):
    p = msg.position
    rospy.loginfo_throttle(1.0, 'left tcp [%s]: %.3f %.3f %.3f',
                           msg.header.frame_id, p.x, p.y, p.z)


rospy.init_node('naviai_sensor_reader')
rospy.Subscriber('/zj_humanoid/sensor/realsense_head/color/image_raw',
                 Image, on_image, queue_size=1, buff_size=2**24)
rospy.Subscriber('/livox/lidar', CustomMsg, on_lidar, queue_size=1)
rospy.Subscriber('/zj_humanoid/upperlimb/tcp_pose/left_arm',
                 Pose, on_left_tcp, queue_size=5)
rospy.spin()
```

Use `cv_bridge` for OpenCV processing and `queue_size=1` to avoid stale-frame backlog.

## rosbridge Pose Example

```js
const leftTcp = new ROSLIB.Topic({
  ros,
  name: '/zj_humanoid/upperlimb/tcp_pose/left_arm',
  messageType: 'upperlimb/Pose'
})

leftTcp.subscribe(({ header, position, rpy_deg }) => {
  console.log(header.frame_id, position, rpy_deg)
})
```

Use `sensor_msgs/CompressedImage` and rate limiting for browser previews. Do not continuously transfer MID360 raw clouds or `PointCloud2` as rosbridge JSON.

## Read-Only Availability Checks

```bash
rostopic info /zj_humanoid/sensor/realsense_head/color/image_raw
rostopic hz /livox/imu
rostopic info /cloud_registered_body
rostopic echo -n 1 /zj_humanoid/upperlimb/tcp_pose/left_arm
rostopic echo -n 1 /zj_humanoid/hand/wrist_force_sensor/left
```

Sensor restart Services change the runtime chain and should not be an automatic first response in an ordinary reader.
