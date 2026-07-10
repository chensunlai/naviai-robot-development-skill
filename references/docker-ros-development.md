# Native ROS Development in Docker

Run native ROS applications in an independent Docker container connected directly to the robot ROS master. For a new project, keep Docker definitions and persistent source under the host Project directory and large resources under Dataset, Model, and Runs. Use a conventional ROS workspace at `/omni_ws` inside a new container; do not reproduce the host Desktop classification.

This page defines a new-project baseline. When working in an existing container, preserve its current workspace path, package layout, mounts, startup mechanism, and setup-source order. Do not retrofit `/omni_ws` or the `omni_*` naming convention unless the user explicitly requests that migration.

Do not treat existing `naviai_*` runtime containers as development workspaces. Their core components are installed from packages or images, and full source is generally not bind-mounted.

## Contents

- [Project and Interface Setup](#project-and-interface-setup)
- [Host and Container Layout](#host-and-container-layout)
- [Dockerfile](#dockerfile)
- [compose.yaml](#composeyaml)
- [Build and Enter](#build-and-enter)
- [Create a ROS Package](#create-a-ros-package)
- [Connection Checks](#connection-checks)
- [Persistent Node Startup](#persistent-node-startup)
- [Other Lab Servers](#other-lab-servers)
- [Application Checklist](#application-checklist)

## Project and Interface Setup

```bash
mkdir -p /home/naviai/Desktop/Project/omni_ward_guide
mkdir -p /home/naviai/Desktop/Dataset/omni_ward_guide
mkdir -p /home/naviai/Desktop/Model/omni_ward_guide
mkdir -p /home/naviai/Desktop/Runs/omni_ward_guide
cd /home/naviai/Desktop/Project/omni_ward_guide
mkdir -p vendor ros_src config
install -m 0755 \
  /home/naviai/navi_project/containers/shared/zj_humanoid_types_dev-v1.3.0+78a9d8e+71.run \
  vendor/zj_humanoid_types.run
```

These are host directories. `ros_src` contains only package source that must persist or be edited from the host. See `references/paths-and-naming.md` for the full boundary.

## Host and Container Layout

| Host Project | Container | Purpose |
|---|---|---|
| `Dockerfile`, `compose.yaml` | Docker/Compose metadata | Build and runtime definition; no mirrored container directory is needed |
| `ros_src/` | `/omni_ws/src` | Persistent ROS package source |
| `config/` | `/config` | Optional external configuration |
| `Desktop/Dataset/<project>` | `/data` | Optional datasets or captured input |
| `Desktop/Model/<project>` | `/models` | Optional model artifacts |
| `Desktop/Runs/<project>` | `/runs` | Optional logs and application output |

The container owns `/omni_ws/build` and `/omni_ws/devel`. Keep those generated directories inside the container or a Docker volume. Do not create `/home/naviai/Desktop`, `Project`, `Dataset`, `Model`, or `Runs` inside the container.

## Dockerfile

```dockerfile
FROM 10.51.33.201:30002/navi_project/environment:ros1_260310

ARG DEBIAN_FRONTEND=noninteractive

COPY vendor/zj_humanoid_types.run /tmp/zj_humanoid_types.run
RUN chmod +x /tmp/zj_humanoid_types.run \
    && /tmp/zj_humanoid_types.run \
    && rm -f /tmp/zj_humanoid_types.run

SHELL ["/bin/bash", "-lc"]
RUN mkdir -p /omni_ws/src
WORKDIR /omni_ws

ENTRYPOINT []
CMD ["bash"]
```

The base image supplies ROS Noetic. The type installer supplies packages including `zj_robot`, `sensor`, `audio`, `upperlimb`, `hand`, `map_server_msgs`, and `navigation`.

## compose.yaml

```yaml
name: omni_ward_guide

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: omni/ward_guide:dev
    container_name: omni_ward_guide
    network_mode: host
    restart: unless-stopped
    init: true

    environment:
      ROS_MASTER_URI: http://192.168.217.1:11311
      ROS_IP: 192.168.217.100
      ROS_LOG_DIR: /runs/ros

    extra_hosts:
      - "jzrobot-a:192.168.217.1"
      - "pico.zjrx.com:192.168.217.66"

    volumes:
      - ./ros_src:/omni_ws/src
      - ./config:/config:ro
      - /home/naviai/Desktop/Dataset/omni_ward_guide:/data:ro
      - /home/naviai/Desktop/Model/omni_ward_guide:/models:ro
      - /home/naviai/Desktop/Runs/omni_ward_guide:/runs

    working_dir: /omni_ws
    command: ["bash", "-lc", "sleep infinity"]
    stdin_open: true
    tty: true

    labels:
      com.omni.project: ward_guide
      com.omni.role: ros-app
```

`container_name` is the final Docker name; Compose service name `app` is local to this file. A multi-container project can use `omni_ward_guide_ros`, `omni_ward_guide_web`, and similar role suffixes.

Before adding another project or container, check whether the new component can share the existing environment and lifecycle. Related packages and nodes should normally remain in the same `omni_*` project or container. Split them only for a concrete boundary such as incompatible dependencies, independent deployment or restart, device permissions, resource isolation, or a distinct operational owner.

## Build and Enter

```bash
cd /home/naviai/Desktop/Project/omni_ward_guide
docker compose build
docker compose up -d
docker exec -it omni_ward_guide bash
```

List application containers with `docker ps --filter 'name=omni_'`. `sleep infinity` is only a development keepalive. Host-mounted package source and optional resources survive container deletion. `/omni_ws/build` and `/omni_ws/devel` are generated container workspace products.

## Create a ROS Package

Inside the container:

```bash
source /opt/ros/noetic/setup.bash

cd /omni_ws/src
catkin_create_pkg omni_ward_guide \
  rospy roscpp std_msgs sensor_msgs geometry_msgs

cd /omni_ws
catkin_make
source devel/setup.bash
```

Per-shell source order:

```bash
source /opt/ros/noetic/setup.bash
source /omni_ws/devel/setup.bash
```

Add only interface packages actually imported by the project, such as `audio` or `upperlimb`.

## Connection Checks

An Orin-hosted container uses the robot-internal address even when SSH entered through `192.168.5.200`.

```bash
echo "$ROS_MASTER_URI"
echo "$ROS_IP"
rosnode ping -c 1 /basic_info_node
rostopic echo -n 1 /zj_humanoid/robot/battery_info
```

If names are visible but a connection reports `jzrobot-a: Name or service not known`, check `extra_hosts`. Host networking does not automatically copy host `/etc/hosts` entries into a container.

Reading ROS Topics does not require privileged mode, NVIDIA runtime, or `/dev`. Add GPU, USB, display, or device access only for a program that directly needs it.

## Persistent Node Startup

Replace the development keepalive with the application launch:

```yaml
command:
  - bash
  - -lc
  - |
    source /opt/ros/noetic/setup.bash
    source /omni_ws/devel/setup.bash
    exec roslaunch omni_ward_guide app.launch
```

Keep `restart: unless-stopped`. Compose manages container restart; roslaunch manages this application's nodes. A single launch entry normally does not need supervisor. Use a project-owned supervisor only for multiple independent processes requiring separate restart behavior.

```bash
docker compose up -d --build
docker compose down
```

`down` removes container and Compose network state, not host bind-mounted project data.

## Other Lab Servers

| Network Condition | Practical Option |
|---|---|
| only Orin `192.168.5.200` reachable | rosbridge at `ws://192.168.5.200:9090` |
| bidirectional reachability to `192.168.217.0/24`; callback succeeds | native ROS container can run remotely |

For a remote native ROS container, set `ROS_IP` to an address robot nodes can reach. The current base image is arm64. An x86 host needs an x86 ROS Noetic image with matching interface types or deliberate cross-architecture emulation.

## Application Checklist

Apply this checklist to newly created projects. For an existing project, first follow its established structure and change it only when the user explicitly asks.

1. Project lives at `/home/naviai/Desktop/Project/omni_<business>`.
2. Dockerfile and Compose are at project root.
3. Container names use `omni_` and do not occupy `naviai_*` names.
4. The new container uses `/omni_ws`; host `ros_src` mounts at `/omni_ws/src`.
5. Image uses `omni/<business>:<explicit-tag>`.
6. ROS package, node, and application interfaces share the project prefix.
7. Host networking, ROS master, ROS IP, and hostname mappings are correct.
8. Host Dataset, Model, and Runs directories mount only when the node needs them, at simple paths such as `/data`, `/models`, and `/runs`.
9. No Desktop or Project/Dataset/Model/Runs hierarchy is created inside the container.
10. Privileged, GPU, and device access are absent unless needed.
11. Persistent apps use `restart: unless-stopped` and a real launch entry.
12. Control nodes implement timeout, cancellation, shutdown stop, and control ownership.
13. A new container is introduced only when the existing project or container cannot reasonably own the component.
