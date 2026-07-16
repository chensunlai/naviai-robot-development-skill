# NAVIAI Custom ROS Types

## Contents

- [Snapshot](#snapshot)
- [Host Archive](#host-archive)
- [Category Index](#category-index)
- [Robot State and System Information](#robot-state-and-system-information)
- [Chassis and Drivers](#chassis-and-drivers)
- [Upper Limb and Dexterous Hands](#upper-limb-and-dexterous-hands)
- [Audio, Wake, and Face Events](#audio-wake-and-face-events)
- [Navigation, Maps, and Localization](#navigation-maps-and-localization)
- [Manipulation and Perception](#manipulation-and-perception)
- [Sensors and Third-Party Drivers](#sensors-and-third-party-drivers)
- [Archive Scope and Provenance](#archive-scope-and-provenance)

## Snapshot

This reference describes the non-standard ROS interfaces collected on 2026-07-16 from the original Orin host, running NAVIAI containers, host deployment packages, and NAVIAI type installers.

| Item | Count |
|---|---:|
| ROS packages | 19 |
| `.msg` files | 148 |
| `.srv` files | 61 |
| `.action` files | 10 |
| Total interface definition files | 219 |

Action packages also contain generated message definitions such as `<Name>Action`, `<Name>ActionGoal`, `<Name>ActionFeedback`, `<Name>ActionResult`, `<Name>Goal`, `<Name>Feedback`, and `<Name>Result`. These generated definitions are included in the message counts and package inventories below.

## Host Archive

Archive root:

```text
/home/naviai/Desktop/Project/NaviaiMessage
```

Archive layout:

```text
NaviaiMessage/
|-- README.md
|-- packages/                    # Interface definitions grouped by ROS package
`-- archives/
    |-- installers/              # Two original zj_humanoid_types installers
    `-- host-dists/              # Host deployment debs containing interfaces
```

Interface file locations follow these patterns:

```text
/home/naviai/Desktop/Project/NaviaiMessage/packages/<package>/msg/<Type>.msg
/home/naviai/Desktop/Project/NaviaiMessage/packages/<package>/srv/<Type>.srv
/home/naviai/Desktop/Project/NaviaiMessage/packages/<package>/action/<Type>.action
```

Most package directories are copies of installed `share/<package>` trees rather than complete Catkin source packages. Most do not contain `CMakeLists.txt`. The archived `navigation_dbg` package contains its message definitions and generated CMake metadata but no `package.xml`.

## Category Index

| Domain | Packages | msg | srv | action | Representative terms |
|---|---|---:|---:|---:|---|
| Robot state and system information | `zj_robot` | 5 | 6 | 0 | robot state, resources, errors, Wi-Fi, face display |
| Chassis and drivers | `chassis_msgs`, `jcui_msgs`, `jdrv_msgs` | 15 | 5 | 0 | AGV, motor, steering, charging, versions |
| Upper limb and dexterous hands | `upperlimb`, `hand` | 32 | 20 | 3 | MoveJ, MoveL, IK, FK, teaching, gestures, pressure |
| Audio, wake, and face events | `audio`, `avvtn_multi_demo` | 5 | 13 | 0 | ASR, TTS, playback, devices, wake events, face events |
| Navigation, maps, and localization | `navigation`, `navigation_dbg`, `map_server_msgs`, `naviai_localization_msgs`, `module_common_msgs` | 34 | 6 | 1 | Navigation, LocalMap, mapping, LIO, trajectories |
| Manipulation and perception | `manipulation`, `fast_lio_localization` | 46 | 9 | 6 | segmentation, picking, placing, tracking, calibration, Pose6D |
| Sensors and third-party drivers | `sensor`, `livox_ros_driver2`, `realsense2_camera`, `legged_messages` | 11 | 2 | 0 | CameraInfo, Livox, RealSense, leg state |

## Robot State and System Information

### `zj_robot` 1.1.1

- Messages: `zj_robot/Errors`, `zj_robot/ModulesMonitor`, `zj_robot/Resource`, `zj_robot/RobotState`, `zj_robot/WorkStatus`
- Services: `zj_robot/BasicInfo`, `zj_robot/ConnectWifi`, `zj_robot/FaceScreen`, `zj_robot/FaceShow`, `zj_robot/SetZero`, `zj_robot/WifiList`
- Archive directory: `packages/zj_robot`

## Chassis and Drivers

### `chassis_msgs` 1.3.0

- Messages: `chassis_msgs/AGVState`, `chassis_msgs/MotorInfo`, `chassis_msgs/MotorState`, `chassis_msgs/OdomInfo`, `chassis_msgs/PowerStatus`, `chassis_msgs/PowerStatusStamped`, `chassis_msgs/SteerCommand`, `chassis_msgs/SteerInfo`, `chassis_msgs/SteerState`, `chassis_msgs/Trigger`, `chassis_msgs/TriggerStamped`
- Services: `chassis_msgs/ChargeControl`, `chassis_msgs/Connect`, `chassis_msgs/SpeedControl`, `chassis_msgs/SteerControl`
- Archive directory: `packages/chassis_msgs`

### `jcui_msgs` 0.0.0

- Services: `jcui_msgs/GetAllVersions`
- Archive directory: `packages/jcui_msgs`

### `jdrv_msgs` 0.0.0

- Messages: `jdrv_msgs/PowerStatus`, `jdrv_msgs/PowerStatusStamped`, `jdrv_msgs/Trigger`, `jdrv_msgs/TriggerStamped`
- Archive directory: `packages/jdrv_msgs`

## Upper Limb and Dexterous Hands

### `upperlimb` 1.3.0

- Messages: `upperlimb/ArmTypeEnum`, `upperlimb/DragTeachRecordAction`, `upperlimb/DragTeachRecordActionFeedback`, `upperlimb/DragTeachRecordActionGoal`, `upperlimb/DragTeachRecordActionResult`, `upperlimb/DragTeachRecordFeedback`, `upperlimb/DragTeachRecordGoal`, `upperlimb/DragTeachRecordResult`, `upperlimb/DragTeachReplayAction`, `upperlimb/DragTeachReplayActionFeedback`, `upperlimb/DragTeachReplayActionGoal`, `upperlimb/DragTeachReplayActionResult`, `upperlimb/DragTeachReplayFeedback`, `upperlimb/DragTeachReplayGoal`, `upperlimb/DragTeachReplayResult`, `upperlimb/DualPose`, `upperlimb/Joints`, `upperlimb/MotionExecuteAction`, `upperlimb/MotionExecuteActionFeedback`, `upperlimb/MotionExecuteActionGoal`, `upperlimb/MotionExecuteActionResult`, `upperlimb/MotionExecuteFeedback`, `upperlimb/MotionExecuteGoal`, `upperlimb/MotionExecuteResult`, `upperlimb/MotionInfo`, `upperlimb/PayLoad`, `upperlimb/Pose`, `upperlimb/SpeedJ`, `upperlimb/SpeedL`, `upperlimb/TcpSpeed`, `upperlimb/UplimbState`
- Services: `upperlimb/ArmType`, `upperlimb/Common`, `upperlimb/FK`, `upperlimb/FilesLists`, `upperlimb/IK`, `upperlimb/IsSingular`, `upperlimb/MotionsLoadedList`, `upperlimb/MotionsManage`, `upperlimb/MoveJ`, `upperlimb/MoveJByPath`, `upperlimb/MoveJByPose`, `upperlimb/MoveL`, `upperlimb/MoveLByPath`, `upperlimb/ParamsGet`, `upperlimb/ParamsSet`, `upperlimb/PayLoadGet`, `upperlimb/PayLoadSet`, `upperlimb/Servo`
- Actions: `upperlimb/DragTeachRecord`, `upperlimb/DragTeachReplay`, `upperlimb/MotionExecute`
- Archive directory: `packages/upperlimb`

### `hand` 1.0.0

- Messages: `hand/PressureSensor`
- Services: `hand/Gesture`, `hand/HandJoint`
- Archive directory: `packages/hand`

## Audio, Wake, and Face Events

### `audio` 1.0.0

- Messages: `audio/AudioData`, `audio/StreamTTSData`
- Services: `audio/AudioPlay`, `audio/AudioStop`, `audio/GetDeviceList`, `audio/GetVolume`, `audio/LLMChat`, `audio/Listen`, `audio/MediaPlay`, `audio/SetDevice`, `audio/SetVolume`, `audio/TTS`
- Archive directory: `packages/audio`

### `avvtn_multi_demo` 0.0.0

- Messages: `avvtn_multi_demo/AudioFaceInfo`, `avvtn_multi_demo/AudioFaceRecognitionResult`, `avvtn_multi_demo/AudioWakeInfo`
- Services: `avvtn_multi_demo/AvvtnNodeControl`, `avvtn_multi_demo/ChangeDevice`, `avvtn_multi_demo/Restart`
- Archive directory: `packages/avvtn_multi_demo`
- Source: `naviai_navbrain_ros:/navi_ws/src/avvtn_multi_demo`
- The archive contains the source package's three messages and three services. The single-message compatibility package added later to `naviai_demos` is not the archived source.

## Navigation, Maps, and Localization

### `navigation` 1.3.0

- Messages: `navigation/ErrorInfo`, `navigation/LocalMap`, `navigation/LocalMapData`, `navigation/ModuleStatus`, `navigation/NavigationAction`, `navigation/NavigationActionFeedback`, `navigation/NavigationActionGoal`, `navigation/NavigationActionResult`, `navigation/NavigationFeedback`, `navigation/NavigationGoal`, `navigation/NavigationResult`, `navigation/NavigationState`, `navigation/TaskType`, `navigation/Translation`, `navigation/Waypoint`
- Actions: `navigation/Navigation`
- Archive directory: `packages/navigation`

### `navigation_dbg` 1.3.0

- Messages: `navigation_dbg/BrakeType`, `navigation_dbg/Control`, `navigation_dbg/ControlStamped`, `navigation_dbg/ControllerState`, `navigation_dbg/Gear`, `navigation_dbg/MpcTrajectory`, `navigation_dbg/Phase`, `navigation_dbg/PhaseStamped`, `navigation_dbg/PlannerState`, `navigation_dbg/Segment`, `navigation_dbg/State`, `navigation_dbg/StateStamped`, `navigation_dbg/Trajectory`, `navigation_dbg/Version`
- Archive directory: `packages/navigation_dbg`
- The version is derived from the deb filename in the type installer. The upstream archive does not contain a `package.xml` for this package.

### `map_server_msgs` 1.3.0

- Messages: `map_server_msgs/MapInfo`
- Services: `map_server_msgs/GetCurMapInfo`, `map_server_msgs/GetMapList`, `map_server_msgs/SetMap`
- Archive directory: `packages/map_server_msgs`

### `naviai_localization_msgs` 1.3.0

- Messages: `naviai_localization_msgs/Feedback`, `naviai_localization_msgs/MappingResult`
- Services: `naviai_localization_msgs/Lio`, `naviai_localization_msgs/Mapping`, `naviai_localization_msgs/Post_processing`
- Archive directory: `packages/naviai_localization_msgs`

### `module_common_msgs` 1.3.0

- Messages: `module_common_msgs/ErrorInfo`, `module_common_msgs/ModuleStatus`
- Archive directory: `packages/module_common_msgs`

## Manipulation and Perception

### `manipulation` 1.0.0

- Messages: `manipulation/DetItem`, `manipulation/Grasp6d`, `manipulation/InstSegAction`, `manipulation/InstSegActionFeedback`, `manipulation/InstSegActionGoal`, `manipulation/InstSegActionResult`, `manipulation/InstSegFeedback`, `manipulation/InstSegGoal`, `manipulation/InstSegResult`, `manipulation/LoosenHandAction`, `manipulation/LoosenHandActionFeedback`, `manipulation/LoosenHandActionGoal`, `manipulation/LoosenHandActionResult`, `manipulation/LoosenHandFeedback`, `manipulation/LoosenHandGoal`, `manipulation/LoosenHandResult`, `manipulation/ObjPose`, `manipulation/PickAction`, `manipulation/PickActionFeedback`, `manipulation/PickActionGoal`, `manipulation/PickActionResult`, `manipulation/PickFeedback`, `manipulation/PickGoal`, `manipulation/PickResult`, `manipulation/PlaceAction`, `manipulation/PlaceActionFeedback`, `manipulation/PlaceActionGoal`, `manipulation/PlaceActionResult`, `manipulation/PlaceFeedback`, `manipulation/PlaceGoal`, `manipulation/PlaceResult`, `manipulation/SearchObjectAction`, `manipulation/SearchObjectActionFeedback`, `manipulation/SearchObjectActionGoal`, `manipulation/SearchObjectActionResult`, `manipulation/SearchObjectFeedback`, `manipulation/SearchObjectGoal`, `manipulation/SearchObjectResult`, `manipulation/TrackAction`, `manipulation/TrackActionFeedback`, `manipulation/TrackActionGoal`, `manipulation/TrackActionResult`, `manipulation/TrackFeedback`, `manipulation/TrackGoal`, `manipulation/TrackResult`
- Services: `manipulation/CameraCalibration`, `manipulation/ExecutePickTask`, `manipulation/GetScenePose`, `manipulation/GraspTeach`, `manipulation/InstSeg`, `manipulation/JointSpaceTrajPlan`, `manipulation/PoseEst`, `manipulation/PoseSpaceTrajPlan`, `manipulation/SceneUpdate`
- Actions: `manipulation/InstSeg`, `manipulation/LoosenHand`, `manipulation/Pick`, `manipulation/Place`, `manipulation/SearchObject`, `manipulation/Track`
- Archive directory: `packages/manipulation`

### `fast_lio_localization` 0.0.0

- Messages: `fast_lio_localization/Pose6D`
- Archive directory: `packages/fast_lio_localization`

## Sensors and Third-Party Drivers

### `sensor` 1.0.0

- Services: `sensor/CameraInfo`
- Archive directory: `packages/sensor`

### `livox_ros_driver2` 1.3.0

- Messages: `livox_ros_driver2/CustomMsg`, `livox_ros_driver2/CustomPoint`
- Archive directory: `packages/livox_ros_driver2`

### `realsense2_camera` 2.3.2

- Messages: `realsense2_camera/Extrinsics`, `realsense2_camera/IMUInfo`, `realsense2_camera/Metadata`
- Services: `realsense2_camera/DeviceInfo`
- Archive directory: `packages/realsense2_camera`

### `legged_messages` 1.0.0

- Messages: `legged_messages/base_info`, `legged_messages/contact_info`, `legged_messages/end_effector_info`, `legged_messages/joint_info`, `legged_messages/motor_info`, `legged_messages/wbc_command`
- Archive directory: `packages/legged_messages`

## Archive Scope and Provenance

- ROS Noetic standard packages such as `std_msgs`, `sensor_msgs`, `geometry_msgs`, `nav_msgs`, and `actionlib_msgs` are outside this archive.
- `livox_ros_driver2`, `realsense2_camera`, and `fast_lio_localization` are third-party packages used by the robot deployment rather than NAVIAI-owned interface packages.
- The primary package snapshots in `packages/` come from `zj_humanoid_types_dev-v1.3.0+78a9d8e+71.run` when that installer contains the package.
- The earlier R3 package set remains in `archives/installers/zj_humanoid_types_25_R3.run`.
- `avvtn_multi_demo`, `fast_lio_localization`, `jcui_msgs`, `jdrv_msgs`, and `realsense2_camera` were collected from a running container or host deployment deb because they are absent from the current type installer's primary package set.
- Package versions and definitions may differ between container images. This archive records the selected host snapshot rather than every observed runtime variant.
