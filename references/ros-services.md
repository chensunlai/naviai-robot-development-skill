# ROS Services

This reference lists Services with clear development roles. Generic logger and driver dynamic-parameter Services are omitted.

## Contents

- [Mapping and Localization](#mapping-and-localization)
- [Map Management](#map-management)
- [Robot Information, Mode, and State](#robot-information-mode-and-state)
- [Wi-Fi](#wi-fi)
- [Sensors](#sensors)
- [Audio and Dialogue](#audio-and-dialogue)
- [Head Display](#head-display)
- [Chassis](#chassis)
- [Upper Limbs](#upper-limbs)
- [Dexterous Hands](#dexterous-hands)
- [rosapi](#rosapi)

## Mapping and Localization

| Service | Type | Request | Response | Provider |
|---|---|---|---|---|
| `/perception/mapping_service` | `naviai_localization_msgs/Mapping` | `map_name`, `z_floor`, `z_ceil`, `resolution`, `scene` | `success`, `message` | `/mapping_service` |
| `/perception/post_processing` | `naviai_localization_msgs/Post_processing` | `method` | `success`, `message` | `/post_processing_service` |
| `/perception/lio_service` | `naviai_localization_msgs/Lio` | method, map path, initial pose | `success`, `message` | `/lio_service` |

`mapping_service` starts Voxel-SLAM. `post_processing method=0` performs normal completion and generates map files. `/perception/mapping_result` asynchronously reports mapping completion.

`naviai_localization_msgs/Lio` request:

```text
string method
string map_path
float32 x_pos, y_pos, z_pos
float32 x_ori, y_ori, z_ori, w_ori
```

Ordinary map loading should use `set_map`, which coordinates map publication and LIO startup. Direct `lio_service` use is mainly for localization-module development.

## Map Management

| Service | Type | Request | Response |
|---|---|---|---|
| `/zj_humanoid/navigation/get_map_list` | `map_server_msgs/GetMapList` | empty | `code`, `message`, `map_name_list[]` |
| `/zj_humanoid/navigation/get_cur_map_info` | `map_server_msgs/GetCurMapInfo` | empty | `code`, `message`, `map_info` |
| `/zj_humanoid/navigation/set_map` | `map_server_msgs/SetMap` | `map_name` | `code`, `message` |

`map_info` includes the map name and `nav_msgs/MapMetaData`: resolution, dimensions, origin, and load time.

## Robot Information, Mode, and State

| Service | Type | Effect or Response |
|---|---|---|
| `/zj_humanoid/robot/basic_info` | `zj_robot/BasicInfo` | `success`, `message`, robot/hardware/software versions, `ip_addr` |
| `/zj_humanoid/mode_manager/set_manual` | `std_srvs/Trigger` | Switch to manual mode |
| `/zj_humanoid/mode_manager/set_auto` | `std_srvs/Trigger` | Switch to automatic mode |
| `/zj_humanoid/mode_manager/toggle_mode` | `std_srvs/Trigger` | Toggle mode |
| `/zj_humanoid/robot/set_robot_state/run` | `std_srvs/Trigger` | Set RUN state |
| `/zj_humanoid/robot/set_robot_state/stop` | `std_srvs/Trigger` | Set STOP state |
| `/zj_humanoid/robot/set_robot_state/restart` | `std_srvs/Trigger` | Restart robot-state flow |
| `/zj_humanoid/robot/set_robot_state/OFF` | `std_srvs/Trigger` | Enter OFF state |

Mode and robot-state calls are global state changes, not connectivity tests. Observe `/zj_humanoid/mode_manager/mode`, `is_manual`, and robot state before changing them.

## Wi-Fi

| Service | Type | Device |
|---|---|---|
| `/zj_humanoid/robot/orin_states/wifi_list` | `zj_robot/WifiList` | Orin |
| `/zj_humanoid/robot/orin_states/connect_wifi` | `zj_robot/ConnectWifi` | Orin |
| `/zj_humanoid/robot/pico_states/wifi_list` | `zj_robot/WifiList` | Pico |
| `/zj_humanoid/robot/pico_states/connect_wifi` | `zj_robot/ConnectWifi` | Pico |

`connect_wifi` changes device network configuration; `wifi_list` is the query operation.

## Sensors

| Service | Type | Effect |
|---|---|---|
| `/zj_humanoid/sensor/restart_all` | `std_srvs/Trigger` | Restart sensor chain |
| `/zj_humanoid/sensor/realsense_head/restart` | `std_srvs/Trigger` | Restart head RealSense |
| `/zj_humanoid/sensor/realsense_head/enable` | `std_srvs/SetBool` | Enable/disable head RealSense |
| `/zj_humanoid/sensor/realsense_head/realsense2_camera/device_info` | `realsense2_camera/DeviceInfo` | Query device information |
| `/zj_humanoid/sensor/realsense_head/realsense2_camera/reset` | `std_srvs/Empty` | Reset RealSense driver |
| `/zj_humanoid/sensor/oak/restart` | `std_srvs/Trigger` | Restart OAK chain |
| `/zj_humanoid/sensor/wrist_camera/restart` | `std_srvs/Trigger` | Restart wrist-camera chain |
| `/zj_humanoid/sensor/CAM_A/camera_info` through `CAM_D/camera_info` | `sensor/CameraInfo` | Query four-camera information |

The CAM_A-D and wrist image publishers were not active in the observed graph. A registered Service does not prove that the device produces data.

## Audio and Dialogue

### TTS

`/zj_humanoid/audio/tts_service`, type `audio/TTS`:

```text
Request:  string[] text, bool isPlay
Response: bool success, string message, int32 status, string file_path
```

`isPlay=true` generates and plays; `false` only generates a file.

### Media Playback

`/zj_humanoid/audio/media_play`, type `audio/MediaPlay`:

```text
Request:  string file_path
Response: bool success, string message, int32 status
```

The path is interpreted inside `naviai_navbrain_ros`. Host shared audio maps to `/navi_ws/Audio/` in that container.

### Devices and Volume

| Service | Type | Key Field or Response |
|---|---|---|
| `/zj_humanoid/audio/microphone/get_devices_list` | `audio/GetDeviceList` | returns `devicelist[]` |
| `/zj_humanoid/audio/microphone/select_device` | `audio/SetDevice` | request `name` |
| `/zj_humanoid/audio/speaker/get_devices_list` | `audio/GetDeviceList` | returns `devicelist[]` |
| `/zj_humanoid/audio/speaker/select_device` | `audio/SetDevice` | request `name` |
| `/zj_humanoid/audio/speaker/get_volume` | `audio/GetVolume` | returns `volume` |
| `/zj_humanoid/audio/speaker/set_volume` | `audio/SetVolume` | request `volume` |
| `/zj_humanoid/audio/version` | `std_srvs/Trigger` | module version in message |

### LLM

`/zj_humanoid/audio/LLM_chat`, type `audio/LLMChat`:

```text
Request
  string raw_input
  bool enable_context
  bool enable_save
  string context_id
Response
  bool success
  string message
  int32 status
  string response
```

## Head Display

`/zj_humanoid/robot/face_show/media_play`, type `zj_robot/FaceShow`:

```text
Request
  string media_path
  bool loop
  float32 duration
Response
  bool success
  string message
  int32 status_code
```

`duration` is seconds; `0` means play to media completion. `media_path` is resolved on Pico, not on Orin.

## Chassis

| Service | Type | Request | Effect |
|---|---|---|---|
| `/zj_humanoid/chassis/agv_connect` | `chassis_msgs/Connect` | connection parameters | Manage chassis connection |
| `/zj_humanoid/chassis/agv_charge` | `chassis_msgs/ChargeControl` | charging parameters | Control charging |
| `/zj_humanoid/chassis/agv_reset` | `std_srvs/Trigger` | empty | Reset chassis |
| `/zj_humanoid/chassis/agv_version` | `std_srvs/Trigger` | empty | Query version |
| `/zj_humanoid/chassis/speed_control` | `chassis_msgs/SpeedControl` | `start`, `vx`, `vy`, `omega` | Low-level velocity control |
| `/zj_humanoid/chassis/steer_control` | `chassis_msgs/SteerControl` | `start`, wheel velocities/angles | Low-level steering control |

Use the Navigation Action for goal-oriented motion. These velocity/steering Services bypass map, localization, and local obstacle planning.

## Upper Limbs

### Discrete Motion

| Service Group | Type | Request Summary |
|---|---|---|
| `/zj_humanoid/upperlimb/movej/<target>` | `upperlimb/MoveJ` | `joints[]`, `v`, `acc`, `t`, `is_async`, `arm_type` |
| `/zj_humanoid/upperlimb/movel/<target>` | `upperlimb/MoveL` | two Poses, `v`, `acc`, `is_async` |
| `/zj_humanoid/upperlimb/movej_by_path/<target>` | `upperlimb/MoveJByPath` | joint path, timestamps, async flag, target |
| `/zj_humanoid/upperlimb/movel_by_path/<target>` | `upperlimb/MoveLByPath` | left/right pose paths, timestamps, async flag |
| `/zj_humanoid/upperlimb/FK/left_arm`, `right_arm` | `upperlimb/FK` | `joints[]`, returns Pose |

No upper-limb IK Service and no MoveIt `/move_group` were registered in the observed runtime.

### Presets, Stop, and Protection

| Service | Type | Effect |
|---|---|---|
| `/zj_humanoid/upperlimb/go_home/left_arm`, `right_arm`, `dual_arm`, `neck`, `waist` | `std_srvs/Trigger` | Return to preset home |
| `/zj_humanoid/upperlimb/go_home/whole_body` | `upperlimb/ArmType` | Return selected target to home |
| `/zj_humanoid/upperlimb/go_down/left_arm`, `right_arm`, `dual_arm` | `std_srvs/Trigger` | Lower arms |
| `/zj_humanoid/upperlimb/stop` | `std_srvs/Trigger` | Stop upper-limb motion |
| `/zj_humanoid/upperlimb/safety_lock` | `std_srvs/Trigger` | Enter safety lock |
| `/zj_humanoid/upperlimb/unlock` | `std_srvs/Trigger` | Unlock |
| `/zj_humanoid/upperlimb/enable_speedj` | `std_srvs/SetBool` | Enable/disable joint-velocity mode |

### Collision, Payload, and Servo Parameters

| Service | Type | Effect |
|---|---|---|
| `/zj_humanoid/upperlimb/collision_detection/is_enable` | `std_srvs/Trigger` | Query collision detection |
| `/zj_humanoid/upperlimb/collision_detection/enable` | `std_srvs/SetBool` | Configure collision detection |
| `/zj_humanoid/upperlimb/collision_detection/get_params` | `upperlimb/ParamsGet` | Query parameters by `arm_type` |
| `/zj_humanoid/upperlimb/collision_detection/set_params` | `upperlimb/ParamsSet` | Set parameter array and `arm_type` |
| `/zj_humanoid/upperlimb/payload/get` | `upperlimb/PayLoadGet` | Query payload |
| `/zj_humanoid/upperlimb/payload/set` | `upperlimb/PayLoadSet` | Set payload |
| `/zj_humanoid/upperlimb/set_servo_params` | `upperlimb/Servo` | Configure velocity, acceleration, lookahead, gain, target |
| `/zj_humanoid/upperlimb/clear_servo_params` | `upperlimb/Servo` | Clear servo parameters |

### Motion Files and Teaching

| Service | Type | Effect |
|---|---|---|
| `/zj_humanoid/upperlimb/motion/lists` | `upperlimb/FilesLists` | List motion files |
| `/zj_humanoid/upperlimb/motion/load` | `upperlimb/MotionsManage` | Load by filename and alias |
| `/zj_humanoid/upperlimb/motion/loaded_lists` | `upperlimb/MotionsLoadedList` | List loaded motions |
| `/zj_humanoid/upperlimb/motion/unload` | `upperlimb/MotionsManage` | Unload motion |
| `/zj_humanoid/upperlimb/teach_mode/enter`, `exit` | `upperlimb/ArmType` | Enter/exit drag teaching |
| `/zj_humanoid/upperlimb/teach_mode/lists` | `upperlimb/FilesLists` | List teaching files |
| `/zj_humanoid/upperlimb/teach_mode/stop_record` | `std_srvs/Trigger` | Stop recording |

## Dexterous Hands

| Service | Type | Request or Effect |
|---|---|---|
| `/zj_humanoid/hand/gesture_switch/left`<br>`/zj_humanoid/hand/gesture_switch/right`<br>`/zj_humanoid/hand/gesture_switch/dual` | `hand/Gesture` | `string[] gesture_name` |
| `/zj_humanoid/hand/joint_switch/left`<br>`/zj_humanoid/hand/joint_switch/right`<br>`/zj_humanoid/hand/joint_switch/dual` | `hand/HandJoint` | `float32[] q` |
| `/zj_humanoid/hand/finger_pressures/left/zero`<br>`/zj_humanoid/hand/finger_pressures/right/zero` | `std_srvs/Trigger` | Zero finger pressure |
| `/zj_humanoid/hand/wrist_force_sensor/left/zero`<br>`/zj_humanoid/hand/wrist_force_sensor/right/zero` | `std_srvs/Trigger` | Zero wrist force sensor |
| `/zj_humanoid/hand/version` | `std_srvs/Trigger` | Query version |

## rosapi

| Service | Purpose |
|---|---|
| `/rosapi/topics` | Topic list and types |
| `/rosapi/topic_type` | One Topic type |
| `/rosapi/services` | Service list |
| `/rosapi/service_type` | One Service type |
| `/rosapi/nodes` | Node list |
| `/rosapi/node_details` | Node interfaces |
| `/rosapi/action_servers` | Action-server list |
| `/rosapi/message_details` | Message fields |
| `/rosapi/service_request_details` | Service request fields |
| `/rosapi/service_response_details` | Service response fields |

Native ROS clients can use `rosservice type/info` and `rossrv show`; non-ROS clients can use these rosapi Services for equivalent runtime discovery.
