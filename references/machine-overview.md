# Machine and Robot Architecture

## Core Model

- The main NAVIAI compute services run on the Orin.
- The Orin separates and manages robot functions with Docker. ROS packages are installed and ROS nodes are started inside those containers.
- The communication environment is ROS1 Noetic. The ROS master runs on the chassis-side host `jzrobot-a`, not inside an Orin container.
- The ROS graph also includes nodes from external devices. Orin containers are not the only node source.
- Applications can connect through rosbridge WebSocket or an independent Docker ROS environment.

Containers are the deployment and process-management boundary. ROS packages organize software inside containers. Topics, Services, and Actions expose robot capabilities.

## Key Network Information

| Object | Current Record | Purpose |
|---|---|---|
| Orin lab Wi-Fi | `wlP1p1s0 / 192.168.5.200` | Lab access for SSH, rosbridge, and Web services |
| Orin robot-internal network | `eno1 / 192.168.217.100` | Communication between Orin ROS containers and internal devices |
| ROS master / chassis side | `jzrobot-a / 192.168.217.1:11311` | ROS master, chassis hardware, and velocity execution |
| rosbridge, lab network | `ws://192.168.5.200:9090` | Application access from the lab network |
| rosbridge, internal network | `ws://192.168.217.100:9090` | Application access from the robot network |
| Upper limbs, hands, and head display | `pico.zjrx.com / 192.168.217.66` | External ROS service device |
| MID360 | `192.168.217.17` | Lidar point-cloud and IMU source |

Logging in through the lab Wi-Fi address does not change the native ROS container address. Orin ROS containers currently use `ROS_IP=192.168.217.100`. Confirm addresses with read-only host-network and container-environment queries when current availability matters.

## Main Orin Service Domains

| Domain | Representative Containers | Main Capabilities |
|---|---|---|
| Sensor access | `naviai_sensor`, `naviai_sensor_lidar` | Cameras, MID360 point clouds, and IMU |
| Mapping and localization | `naviai_perception` | Mapping, localization, and local obstacle information |
| Map management | `naviai_map_server` | Map listing, loading, and occupancy-grid publication |
| Navigation | `naviai_navigation` | Navigation Action, planning, and velocity output |
| Chassis bridge | `naviai_chassis` | Chassis state, odometry, and control-interface bridging |
| Robot state | `naviai_robot` | Basic information, resources, errors, network, and module state |
| Audio and dialogue | `naviai_navbrain_ros` | ASR, TTS, media playback, and dialogue services |
| Non-ROS access | `naviai_rosbridge` | WebSocket port `9090` and rosapi |

Auxiliary containers provide visualization, demos, and remote interfaces. Use the live Docker and ROS state for the current container, process, and node list. See `references/documentation-index.md` for detailed nodes, mounts, and the navigation chain.
