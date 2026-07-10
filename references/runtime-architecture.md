# Runtime Architecture

This reference captures the observed NAVIAI runtime architecture. It is a portable snapshot, not a claim that every future deployment has the same live state.

## Contents

- [Runtime Baseline](#runtime-baseline)
- [Container Inventory](#container-inventory)
- [Core ROS Containers](#core-ros-containers)
- [Auxiliary Containers](#auxiliary-containers)
- [External ROS Devices](#external-ros-devices)
- [Source and Mount Model](#source-and-mount-model)
- [Network and Startup Model](#network-and-startup-model)
- [Map Data Layout](#map-data-layout)

## Runtime Baseline

| Item | Observed Value |
|---|---|
| Orin lab Wi-Fi | `wlP1p1s0 / 192.168.5.200` |
| Orin robot-internal network | `eno1 / 192.168.217.100` |
| Compose project | `navi_project` |
| Runtime containers | 13 containers named `naviai_*` |
| ROS | ROS1 Noetic |
| ROS master | `http://192.168.217.1:11311` |
| Networking | 12 host-network containers and 1 bridge-network container |
| Main process pattern | container entrypoint installs packages, then starts `supervisord` |
| Shared base image | `10.51.33.201:30002/navi_project/environment:ros1_260310` |

## Container Inventory

| Container | Network | Image Short Name | Process Manager | Responsibility |
|---|---|---|---|---|
| `naviai_sensor` | host | `sensor:22CUDA_v1` | supervisor | RealSense, OAK launcher, camera information, sensor restart |
| `naviai_sensor_lidar` | host | `environment:ros1_260310` | supervisor | Livox MID360 driver |
| `naviai_perception` | host | `environment:ros1_260310` | supervisor | Mapping, localization, post-processing, local obstacle map |
| `naviai_map_server` | host | `environment:ros1_260310` | supervisor | Map directory, map Topics, map Services |
| `naviai_navigation` | host | `environment:ros1_260310` | supervisor | Action navigation, path planning, velocity output |
| `naviai_chassis` | host | `environment:ros1_260310` | supervisor | Orin-to-chassis bridge |
| `naviai_robot` | host | `environment:ros1_260310` | supervisor | Basic information, resources, errors, Wi-Fi, module monitoring |
| `naviai_navbrain_ros` | host | `navbrain_ros:v1.2.4` | startup script | ASR, TTS, media, dialogue, startup checks |
| `naviai_rosbridge` | host | `environment:ros1_260310` | supervisor | rosbridge WebSocket and rosapi |
| `naviai_demos` | host | `demos:v1.0.2` | supervisor | Demo environment and SSH port `2222` |
| `naviai_nviz` | host | `nviz:ros1_260311` | supervisor | RViz environment |
| `naviai_novnc` | host | `novnc:v1.0.2` | supervisor | Xvfb, VNC, noVNC virtual desktop |
| `naviai_robot_viewer` | bridge | `robot_viewer:v1.0.2` | Node/Vite | Web robot viewer, host `10002` to container `3000` |

The shared base image is only the starting environment. Service-specific packages from `.dists/<service>/`, mounts, entrypoints, and commands produce different final containers.

## Core ROS Containers

### `naviai_sensor`

- Nodes: `/zj_humanoid/sensor/realsense_head/realsense_head`, `/zj_humanoid/sensor/realsense_head/realsense2_camera`, `/camera_info_server_node`, `/sensor_restart_service`.
- Outputs: head RGB, depth, aligned depth, and CameraInfo.
- Observed state: RealSense, OAK launcher, and restart service run; supervisor reported `wrist_camera` as `FATAL`.
- `CAM_A` through `CAM_D` and wrist-camera Topic names may appear because monitoring nodes subscribe to them, even when no camera publisher exists.

### `naviai_sensor_lidar`

- Node: `/livox_lidar_publisher2`.
- Driver: `livox_ros_driver2`.
- Device: MID360 at `192.168.217.17`.
- Outputs: `/livox/lidar`, `/livox/imu`.

### `naviai_perception`

| Process or Node | Lifetime | Responsibility |
|---|---|---|
| `/lio_service` | supervisor resident | Provides `/perception/lio_service`; starts Fast-LIO for a selected map |
| `/mapping_service` | supervisor resident | Provides `/perception/mapping_service`; starts Voxel-SLAM |
| `/post_processing_service` | supervisor resident | Stops mapping, post-processes, creates PGM/YAML/PCD artifacts |
| `/bodynorm_to_imu1` | supervisor resident | Publishes static sensor transforms |
| `/test_grid` | supervisor resident | Builds `navigation/LocalMap` from registered cloud and odometry |
| `/laserMapping` | on demand after map load | Fast-LIO localization and registered-cloud output |

Map mount: host `/home/naviai/navi_project/containers/perception/datasets` to container `/datasets`.

### `naviai_map_server`

- Dynamic node name: `/nav_map_server_<suffix>`.
- Container map path: `/navi_ws/map_config/<map_name>`.
- Topics: `/zj_humanoid/navigation/map`, `/zj_humanoid/navigation/map_metadata`.
- Services: `get_map_list`, `get_cur_map_info`, `set_map` under `/zj_humanoid/navigation`.
- Clients should depend on stable Topic and Service names, not the changing node suffix.

### `naviai_navigation`

- Processes: `./navigation wa2`, `navigation_version_node`.
- Nodes: `/navigation_node`, `/navigation_version_node`.
- Inputs: `/zj_humanoid/navigation/map`, `odom_info`, `local_map`.
- Action: `/zj_humanoid/navigation/navigation`.
- Outputs: `navigation_code`, `path`, `trajectory`, `mpc_trajectory`, `navigation_debug`, `/calib_vel`.
- Observed connection: all three inputs connected; `/calib_vel` connected to `/speed_manager` on `jzrobot-a`.

### `naviai_chassis`

- Process: `wa2_node`.
- Node: `/agv_websocket_node`.
- Outputs: battery, chassis state, odometry, IMU, motor, steering information.
- Services: chassis connection, reset, charging, velocity, steering control.
- This is the Orin-side bridge. Hardware management and navigation velocity execution also involve `jzrobot-a`.

### `naviai_robot`

| Node | Responsibility |
|---|---|
| `/basic_info_node` | `/zj_humanoid/robot/basic_info` |
| `/monitor_node` | Aggregates module states into monitoring information |
| `/orin_resource_publisher` | CPU, temperature, memory, disk state |
| `/orin_errors_publisher` | Resource-threshold errors |
| `/orin_wifi_list_server` | Wi-Fi query |
| `/orin_connect_wifi_server` | Wi-Fi connection |
| `/work_status_from_start` | Startup work status |

### `naviai_navbrain_ros`

| Node | Responsibility |
|---|---|
| `/avvtn_node` | Microphone, wake word, audio streams, ASR text |
| `/navbrain` | TTS, media playback, device management, volume, LLM Services |
| `/startup_check` | Startup module checks and prompt audio |

It is started by `/navi_ws/start_ros_nodes_product.sh`, not supervisor. The name `navbrain` is unrelated to ROS2 Nav2.

### `naviai_rosbridge`

- `/rosbridge_websocket`: listens on `0.0.0.0:9090` and forwards ROS messages and Service calls.
- `/rosapi`: graph, type, and Action-server introspection.
- Observed configuration permits all Topics and Services and does not enable rosbridge authentication.

## Auxiliary Containers

| Container | Processes or Entry Point | Use |
|---|---|---|
| `naviai_demos` | `sshd` under supervisor | SSH through `192.168.5.200:2222`; bundled interface types are older |
| `naviai_nviz` | `roscore`, `rosmaster`, `rosout` | RViz is installed but is not resident by default; `naviai_rviz` starts it on demand |
| `naviai_novnc` | `Xvfb :99`, `x11vnc :5999`, `websockify :10086`, Fluxbox, xterm | Virtual desktop for GUIs using `DISPLAY=:99`; Web access at `http://192.168.5.200:10086` |
| `naviai_robot_viewer` | Vite/Node/esbuild | Web viewer at `http://192.168.5.200:10002` |

## External ROS Devices

### `jzrobot-a / 192.168.217.1`

| Node | Role |
|---|---|
| `/rosmaster`, `/rosout` | ROS master and log aggregation |
| `/speed_manager` | Consumes `/calib_vel` and participates in chassis velocity execution |
| `/driver_manager_node`, `/jzhw_node` | Chassis and hardware management |
| `/cartographer_node`, `/cartographer_tf_publisher` | Separate Cartographer-chain evidence |
| `/nav_manager_node`, `/costmap_pub_node`, `/trajectory_generator` | Separate navigation management, costmap, trajectory functions |
| `/scan_filter`, `/scan_localiser` | External 2D lidar filtering and localization functions |

These nodes existing in the graph does not prove that the Orin navigation Action uses them. The current `/navigation_node` inputs define the active Action chain.

### `pico.zjrx.com / 192.168.217.66`

| Node | Role |
|---|---|
| `/zj_humanoid/zj_humanoid_naviai_robot` | Upper-limb state and control Services/Actions |
| `/zj_humanoid/hand` | Dexterous hands, pressure, wrist force |
| `/zj_humanoid/mode_manager` | Mode and upper-body enabled state |
| `/video_display` | Head-display media |
| `/zj_humanoid/robot` | Robot state |
| `/resource_publisher`, `/errors_publisher` | Pico resources and errors |

Unresolved addresses observed on the internal subnet: `192.168.217.50`, `.107`, and `.254`.

## Source and Mount Model

The runtime is not a host-source bind-mount development environment. Core code normally comes from images or packages installed from host `.dists` directories at container startup.

Key source-Orin paths:

```text
/home/naviai/navi_project/config/                 shared runtime configuration
/home/naviai/navi_project/containers/<service>/   per-container Compose assets and mounts
/home/naviai/navi_project/.dists/<service>/       service installation packages
/package/*.deb                                    package staging path used by some entrypoints
/navi_ws/src/navigation/config/                   navigation configuration inside its container
/var/log/                                         common container log target
```

| Container Group | Host-Mounted Content | Program Location |
|---|---|---|
| sensor, lidar | `.dists`, entrypoint, supervisor, config, logs, devices | Drivers/nodes installed into container; full source not mounted |
| perception | `.dists/perception`, config, maps, logs | Voxel-SLAM, Fast-LIO, octree installed from packages |
| map server | `.dists/map_server`, startup config, maps, logs | Map server installed from packages |
| navigation | `.dists/navigation`, planner config, maps, logs | Navigation binary installed from package; mounted config is not full source |
| chassis, robot | `.dists`, entrypoint/supervisor, config, logs | Nodes installed from packages |
| navbrain | shared audio, release, license tool | Main code inside `navbrain_ros:v1.2.4` image |
| rosbridge | entrypoint, supervisor, host `src`, shared directory, logs | rosbridge from image; observed host `src` had no business source |
| demos | devices, X11, shared directory, logs | Demo environment inside image |
| nviz | config, bag, startup config, logs | RViz/plugins from image or packages |
| robot viewer | no host mounts | Web application inside image |

Changing a mounted configuration can affect that configuration. It does not directly edit installed core programs.

## Network and Startup Model

ROS1 nodes register XMLRPC addresses with the master, then establish direct node-to-node connections. Host networking reduces Docker NAT callback issues. Native ROS application containers still need:

```bash
ROS_MASTER_URI=http://192.168.217.1:11311
ROS_IP=<host-address-reachable-from-the-robot-network>
```

Required hostname mappings:

```text
192.168.217.1   jzrobot-a
192.168.217.66  pico.zjrx.com
```

| Startup Pattern | Used By | Meaning |
|---|---|---|
| entrypoint to supervisor | sensors, perception, map, navigation, chassis, robot, rosbridge, nviz, demos, novnc | Supervisor manages processes only inside its own container |
| direct startup script | `naviai_navbrain_ros` | Starts launch files and Python nodes |
| Node entry | `naviai_robot_viewer` | `npm run dev -- --host 0.0.0.0` |

Compose `restart: unless-stopped` provides container-level restart. Supervisor provides process-level management inside selected containers.

## Map Data Layout

```text
Host:        /home/naviai/navi_project/containers/perception/datasets/<map_name>/
Perception:  /datasets/<map_name>/
Map server:  /navi_ws/map_config/<map_name>/
```

A post-processed map normally contains `map.yaml`, `map.pgm`, `merged_map.pcd`, `edge.txt`, and point-cloud chunks.
