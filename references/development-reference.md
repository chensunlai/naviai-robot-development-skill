# Development Environment Reference

## Two Access Options

| Environment | Suitable Needs | Detailed Reference |
|---|---|---|
| rosbridge WebSocket | browser, Node.js, business service, non-ROS Python; Topic/Service/Action without local ROS | `references/rosbridge-development.md` |
| independent Docker ROS | `rospy`, `roscpp`, Actionlib, ROS CLI, custom types, native image/cloud/high-rate data | `references/docker-ros-development.md` |

Both are clients of the same ROS graph. They do not represent capability tiers and do not inherently limit an application to read-only or control use.

The source-machine convention avoids installing ROS development dependencies directly on host operating systems. A non-ROS client can use rosbridge; native ROS code can run in an isolated container.

## Filesystem Boundary

| Development Location | File-Management Preference |
|---|---|
| Orin host | Put each application's project-owned `compose.yaml`, configuration, documentation, rosbridge clients, non-ROS modules, and persistent source under `/home/naviai/Desktop/Project/omni_<business>`. The default container helper generates the Compose file. Keep large data, models, and results in the matching Desktop classifications. |
| New native ROS container | Use a normal Catkin workspace at `/omni_ws`. Packages live in `/omni_ws/src`; Catkin generates `/omni_ws/build` and `/omni_ws/devel`. Do not reproduce the host Desktop classification. |

On the original Orin, the default `naviai_container create <name>` flow persists the complete workspace by mounting `Project/<name>/omni_ws` at `/omni_ws`; it also creates or reuses `Project/<name>/compose.yaml` and uses its `workspace` service to start the container. A manually designed Compose project may instead mount only `Project/<project>/ros_src` at `/omni_ws/src`. Both approaches keep the host Desktop classification outside the container. Existing containers retain their established workspace path and source order.

## Key Endpoints

| Item | Baseline Value |
|---|---|
| rosbridge, lab network | `ws://192.168.5.200:9090` |
| rosbridge, robot network | `ws://192.168.217.100:9090` |
| ROS master | `http://192.168.217.1:11311` |
| Orin native ROS IP | `192.168.217.100` |
| Required hostnames | `jzrobot-a:192.168.217.1`, `pico.zjrx.com:192.168.217.66` |

## New Native ROS Baseline

| Item | Baseline |
|---|---|
| image strategy | a newly generated helper Compose file uses the verified local demos image; an edited or manually designed Compose file directly reuses an inspected local image; do not build or retag a derived image by default |
| plain ROS image example | `10.51.33.201:30002/navi_project/environment:ros1_260310` |
| custom interfaces | verify required packages in the selected image; direct reuse does not add an installer |
| container creation | use `naviai_container create` by default; it creates or reuses the project, persisted workspace, and Compose-managed `workspace` service |
| generated Compose definition | `/home/naviai/Desktop/Project/omni_<business>/compose.yaml`; edit it for additional mounts, another inspected image, or startup changes |
| Compose project name | project-specific `omni_<business>`, passed explicitly with `docker compose -p omni_<business> ...` |
| network | host |
| base setup | `/opt/ros/noetic/setup.bash` |
| container ROS workspace | `/omni_ws` |
| application setup | `/omni_ws/devel/setup.bash` |

Other servers can run native ROS containers when bidirectional robot-network reachability and ROS callbacks work. When only Orin lab Wi-Fi is reachable, rosbridge is normally simpler. The baseline image is arm64; x86 needs a compatible image or cross-architecture emulation.

## Capability References

- Sensor data: `references/sensor-access.md`
- Mapping and navigation: `references/navigation.md`
- Upper limbs and hands: `references/upper-limb-and-hand.md`
- Audio and display: `references/audio-and-display.md`
- Cross-capability orchestration: `references/task-orchestration.md`
- Exact ROS interfaces: `references/ros-topics-actions.md`, `references/ros-services.md`
