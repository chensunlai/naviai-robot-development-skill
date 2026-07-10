# ROS Topics and Actions

This is the bundled interface snapshot. Event Topics can remain silent when no event occurs. A Topic name in the master does not by itself prove that an active publisher exists.

## Contents

- [Robot and Mode State](#robot-and-mode-state)
- [Cameras, Lidar, and IMU](#cameras-lidar-and-imu)
- [Mapping and Localization](#mapping-and-localization)
- [Global Map](#global-map)
- [Navigation Action](#navigation-action)
- [Upper-Limb Actions](#upper-limb-actions)
- [Navigation Planner Output](#navigation-planner-output)
- [Chassis State and Low-Level Control](#chassis-state-and-low-level-control)
- [Upper Limbs and Hands](#upper-limbs-and-hands)
- [Audio](#audio)
- [ROS Infrastructure](#ros-infrastructure)
- [Names Observed Without Active Publishers](#names-observed-without-active-publishers)

## Robot and Mode State

| Topic | Type | Publisher | Meaning |
|---|---|---|---|
| `/zj_humanoid/robot/robot_state` | `zj_robot/RobotState` | Pico `/zj_humanoid/robot` | Whole-robot state |
| `/zj_humanoid/robot/battery_info` | `sensor_msgs/BatteryState` | `/agv_websocket_node` | Battery state |
| `/zj_humanoid/robot/orin_states/resource` | `zj_robot/Resource` | `/orin_resource_publisher` | CPU, temperature, memory, disk usage |
| `/zj_humanoid/robot/orin_states/errors` | `zj_robot/Errors` | `/orin_errors_publisher` | Resource-threshold error flags |
| `/zj_humanoid/robot/pico_states/resource` | `zj_robot/Resource` | Pico `/resource_publisher` | Pico resource state |
| `/zj_humanoid/robot/pico_states/errors` | `zj_robot/Errors` | Pico `/errors_publisher` | Pico error flags |
| `/zj_humanoid/mode_manager/mode` | `std_msgs/String` | Pico mode manager | Current mode name |
| `/zj_humanoid/mode_manager/is_manual` | `std_msgs/Bool` | Pico mode manager | Manual-mode flag |
| `/zj_humanoid/mode_manager/upperbody_enabled` | `std_msgs/Bool` | Pico mode manager | Upper-body enabled state |

`zj_robot/RobotState.state`: `NULL=0`, `CONFIG=1`, `ON=2`, `START=3`, `INIT=4`, `RUN=5`, `HALT=6`, `STOP=7`, `OFF=8`, `ERR=9`.

## Cameras, Lidar, and IMU

| Topic | Type | Publisher | Meaning |
|---|---|---|---|
| `/zj_humanoid/sensor/realsense_head/color/image_raw` | `sensor_msgs/Image` | head RealSense | Raw RGB |
| `/zj_humanoid/sensor/realsense_head/color/image_raw/compressed` | `sensor_msgs/CompressedImage` | head RealSense | Compressed RGB |
| `/zj_humanoid/sensor/realsense_head/color/camera_info` | `sensor_msgs/CameraInfo` | head RealSense | RGB intrinsics/distortion |
| `/zj_humanoid/sensor/realsense_head/depth/image_rect_raw` | `sensor_msgs/Image` | head RealSense | Rectified depth |
| `/zj_humanoid/sensor/realsense_head/depth/camera_info` | `sensor_msgs/CameraInfo` | head RealSense | Depth intrinsics |
| `/zj_humanoid/sensor/realsense_head/aligned_depth_to_color/image_raw` | `sensor_msgs/Image` | head RealSense | Depth aligned to RGB pixels |
| `/livox/lidar` | `livox_ros_driver2/CustomMsg` | `/livox_lidar_publisher2` | MID360 raw point cloud |
| `/livox/imu` | `sensor_msgs/Imu` | `/livox_lidar_publisher2` | MID360 IMU |

`livox_ros_driver2/CustomMsg` includes `timebase`, `point_num`, `lidar_id`, and `points[]`. Each point includes time offset, XYZ, reflectivity, tag, and line.

## Mapping and Localization

### Resident Control and Status

| Topic | Type | Publisher | Meaning |
|---|---|---|---|
| `/perception/feedback` | `naviai_localization_msgs/Feedback` | perception service nodes | State, error code, text |
| `/perception/mapping_code` | `module_common_msgs/ModuleStatus` | mapping service | Mapping module state |
| `/perception/mapping_result` | `naviai_localization_msgs/MappingResult` | `/mapping_service` | Mapping-end event: map name, success, message |
| `/perception/perception_code` | `module_common_msgs/ModuleStatus` | `/test_grid` | Local perception state |

Subscribe to `/perception/mapping_result` before stopping mapping if the event must be captured. Also preserve the post-processing Service result and verify map files.

### Mapping-Time Output

| Topic | Type | Meaning |
|---|---|---|
| `/map_odom` | `nav_msgs/Odometry` | Mapping pose |
| `/map_scan` | `sensor_msgs/PointCloud2` | Current scan |
| `/map_cmap` | `sensor_msgs/PointCloud2` | Accumulated map cloud |
| `/map_pmap` | `sensor_msgs/PointCloud2` | Backend-processed cloud |

### Map-Localization Output

| Topic | Type | Publisher | Meaning |
|---|---|---|---|
| `/zj_humanoid/navigation/odom_info` | `nav_msgs/Odometry` | `/laserMapping` | Navigation localization, currently `frame_id=map` |
| `/odom_imu` | `nav_msgs/Odometry` | `/laserMapping` | Raw LIO odometry |
| `/cloud_registered_body` | `sensor_msgs/PointCloud2` | `/laserMapping` | Registered cloud in body frame |
| `/cloud_registered` | `sensor_msgs/PointCloud2` | `/laserMapping` | Registered cloud |
| `/Laser_map`, `/globalmap` | `sensor_msgs/PointCloud2` | `/laserMapping` | Loaded-map cloud output |
| `/zj_humanoid/navigation/local_map` | `navigation/LocalMap` | `/test_grid` | Local obstacle, semantic, dynamic data |

`navigation/LocalMap` contains map metadata and `LocalMapData[]`; each cell contains `occupancy`, `semantic`, `dynamic`, `speed`, and `direction`.

## Global Map

| Topic | Type | Publisher | Meaning |
|---|---|---|---|
| `/zj_humanoid/navigation/map` | `nav_msgs/OccupancyGrid` | `/nav_map_server_*` | Current 2D occupancy grid |
| `/zj_humanoid/navigation/map_metadata` | `nav_msgs/MapMetaData` | `/nav_map_server_*` | Resolution, dimensions, origin, load time |

## Navigation Action

| Item | Value |
|---|---|
| Name | `/zj_humanoid/navigation/navigation` |
| Type | `navigation/NavigationAction` |
| Server | `/navigation_node` in `naviai_navigation` |
| Goal | task type, one or more Waypoints, terminal translation/heading adjustment |
| Feedback | current `NavigationState`, `faults[]` |
| Result | state, duration, arrival deviations, `causes[]` |

Actionlib wire Topics:

| Topic | Type | Direction |
|---|---|---|
| `/zj_humanoid/navigation/navigation/goal` | `navigation/NavigationActionGoal` | client to server |
| `/zj_humanoid/navigation/navigation/cancel` | `actionlib_msgs/GoalID` | client to server |
| `/zj_humanoid/navigation/navigation/status` | `actionlib_msgs/GoalStatusArray` | server to client |
| `/zj_humanoid/navigation/navigation/feedback` | `navigation/NavigationActionFeedback` | server to client |
| `/zj_humanoid/navigation/navigation/result` | `navigation/NavigationActionResult` | server to client |

Use Actionlib or a roslibjs ActionClient rather than publishing a handcrafted `/goal` message.

## Upper-Limb Actions

| Action | Type | Server | Purpose |
|---|---|---|---|
| `/zj_humanoid/upperlimb/motion` | `upperlimb/MotionExecuteAction` | Pico upper-limb node | Execute a loaded named motion |
| `/zj_humanoid/upperlimb/teach_mode/start_record` | `upperlimb/DragTeachRecordAction` | Pico upper-limb node | Record drag teaching |
| `/zj_humanoid/upperlimb/teach_mode/replay` | `upperlimb/DragTeachReplayAction` | Pico upper-limb node | Replay teaching |

## Navigation Planner Output

| Topic | Type | Publisher | Consumer or Purpose |
|---|---|---|---|
| `/zj_humanoid/navigation/navigation_code` | `navigation/ModuleStatus` | `/navigation_node` | module state/faults; monitored by `/monitor_node` |
| `/zj_humanoid/navigation/path` | `navigation_dbg/Trajectory` | `/navigation_node` | global/planned path debug |
| `/zj_humanoid/navigation/trajectory` | `navigation_dbg/Trajectory` | `/navigation_node` | local trajectory debug |
| `/zj_humanoid/navigation/mpc_trajectory` | `navigation_dbg/MpcTrajectory` | `/navigation_node` | MPC trajectory debug |
| `/zj_humanoid/navigation/navigation_debug` | `navigation_dbg/DebugInfo` | `/navigation_node` | planner debug data |
| `/calib_vel` | `geometry_msgs/Twist` | `/navigation_node` | consumed by `/speed_manager` |

## Chassis State and Low-Level Control

| Topic | Type | Source | Meaning |
|---|---|---|---|
| `/zj_humanoid/chassis/odom_info` | `nav_msgs/Odometry` | `naviai_chassis` | Chassis-bridge odometry; distinct from LIO `navigation/odom_info` |
| `/zj_humanoid/chassis/agv_imu` | `sensor_msgs/Imu` | `naviai_chassis` | Chassis IMU |
| `/zj_humanoid/chassis/agv_state` | `chassis_msgs/AGVState` | `naviai_chassis` | Chassis state |
| `/zj_humanoid/chassis/motor_info` | `chassis_msgs/MotorInfo` | `naviai_chassis` | Motor state |
| `/zj_humanoid/chassis/steer_info` | `chassis_msgs/SteerInfo` | `naviai_chassis` | Steering state |
| `/webService/cmd_vel` | `geometry_msgs/Twist` | `jzrobot-a` input | External Web velocity control |
| `/platform_control/cmd_vel` | `jdrv_msgs/MotionSpeed` | `jzrobot-a` input | Low-level platform velocity input |

Use the Navigation Action for goal-oriented business navigation. Velocity Topics are low-level control surfaces.

## Upper Limbs and Hands

| Topic | Type | Meaning |
|---|---|---|
| `/zj_humanoid/upperlimb/joint_states` | `sensor_msgs/JointState` | Real upper-limb joint state |
| `/zj_humanoid/upperlimb/uplimb_state` | `upperlimb/UplimbState` | Control mode and singularity state |
| `/zj_humanoid/upperlimb/occupancy_state` | `std_msgs/Int8` | Occupancy state |
| `/zj_humanoid/upperlimb/tcp_pose/left_arm` | `upperlimb/Pose` | Left TCP position, quaternion, RPY |
| `/zj_humanoid/upperlimb/tcp_pose/right_arm` | `upperlimb/Pose` | Right TCP position, quaternion, RPY |
| `/zj_humanoid/upperlimb/tcp_speed/dual_arm` | `upperlimb/TcpSpeed` | Dual-arm TCP velocity |
| `/zj_humanoid/upperlimb/jacobian/left_arm` | `std_msgs/Float64MultiArray` | Left-arm Jacobian matrix |
| `/zj_humanoid/upperlimb/jacobian/right_arm` | `std_msgs/Float64MultiArray` | Right-arm Jacobian matrix |
| `/zj_humanoid/upperlimb/servoj/left_arm`<br>`/zj_humanoid/upperlimb/servoj/right_arm`<br>`/zj_humanoid/upperlimb/servoj/dual_arm`<br>`/zj_humanoid/upperlimb/servoj/whole_body` | `upperlimb/Joints` | Continuous joint-position input |
| `/zj_humanoid/upperlimb/speedj/left_arm`<br>`/zj_humanoid/upperlimb/speedj/right_arm`<br>`/zj_humanoid/upperlimb/speedj/dual_arm` | `upperlimb/SpeedJ` | Continuous joint-velocity input |
| `/zj_humanoid/upperlimb/servol/left_arm`<br>`/zj_humanoid/upperlimb/servol/right_arm`<br>`/zj_humanoid/upperlimb/servol/dual_arm` | `upperlimb/DualPose` | Continuous TCP-pose input |
| `/zj_humanoid/hand/joint_states` | `sensor_msgs/JointState` | Hand joint state |
| `/zj_humanoid/hand/finger_pressures/left`<br>`/zj_humanoid/hand/finger_pressures/right` | `hand/PressureSensor` | Finger pressure |
| `/zj_humanoid/hand/wrist_force_sensor/left`<br>`/zj_humanoid/hand/wrist_force_sensor/right` | `geometry_msgs/WrenchStamped` | Wrist force/torque |

## Audio

| Topic | Type | Relationship | Meaning |
|---|---|---|---|
| `/zj_humanoid/audio/listen` | `std_msgs/Bool` | `/avvtn_node` subscribes | Start/stop listening |
| `/zj_humanoid/audio/listen_state` | `std_msgs/Bool` | `/avvtn_node` publishes | Current listening state |
| `/zj_humanoid/audio/asr_text` | `std_msgs/String` | `/avvtn_node`, `/navbrain` publish | ASR text result |
| `/zj_humanoid/audio/microphone/audio_data` | `audio/AudioData` | `/avvtn_node` to `/navbrain` | Audio and VAD data |
| `/zj_humanoid/audio/microphone/wake_data` | `audio/AudioData` | `/avvtn_node` publishes | Wake audio |
| `/zj_humanoid/audio/microphone/wake_info` | `avvtn_multi_demo/AudioWakeInfo` | `/avvtn_node` publishes | Wake event |

## ROS Infrastructure

| Topic | Type | Purpose |
|---|---|---|
| `/tf` | `tf2_msgs/TFMessage` | Dynamic transforms |
| `/tf_static` | `tf2_msgs/TFMessage` | Static transforms |
| `/rosout`, `/rosout_agg` | `rosgraph_msgs/Log` | ROS logs |
| `/connected_clients` | `rosbridge_msgs/ConnectedClients` | rosbridge client state |

## Names Observed Without Active Publishers

| Group | Observation |
|---|---|
| `/zj_humanoid/sensor/CAM_A/image_raw` through `CAM_D/image_raw` | monitor subscriptions exist; no camera publisher observed |
| `/zj_humanoid/sensor/left_wrist/image_raw`, `right_wrist/image_raw` | no publisher; type unresolved from graph |
| `/zj_humanoid/sensor/realsense_down/*` | no publisher observed |
| `/move_base_simple/goal` | no subscriber; not the main navigation entry |

Use `rostopic info <topic>` to distinguish an active publisher, subscription-only name, and a temporarily silent event Topic.
