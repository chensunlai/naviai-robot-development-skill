# Native ROS Development in Docker

Run native ROS applications in an independent Docker container connected directly to the robot ROS master. On the original Orin host, prefer `naviai_container create` for a blank standalone development container. When persistent mounts, declarative startup, or multiple services are required, keep an optional `compose.yaml` in the owning application directory and directly reuse an existing local image. Do not join new containers to a shared Compose project, and do not build or retag a project-specific derived image by default. Keep persistent source under the host Project directory and large resources under Dataset, Model, and Runs. Use a conventional ROS workspace at `/omni_ws` inside a new container; do not reproduce the host Desktop classification.

This page defines a new-project baseline. When working in an existing container, preserve its current workspace path, package layout, mounts, startup mechanism, and setup-source order. Do not retrofit `/omni_ws` or the `omni_*` naming convention unless the user explicitly requests that migration.

Do not treat existing `naviai_*` runtime containers as development workspaces. Their core components are installed from packages or images, and full source is generally not bind-mounted.

## Contents

- [Project and Interface Setup](#project-and-interface-setup)
- [Host and Container Layout](#host-and-container-layout)
- [Existing Image Selection](#existing-image-selection)
- [Project-Owned compose.yaml](#project-owned-composeyaml)
- [Create and Enter](#create-and-enter)
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
mkdir -p ros_src config
```

These are host directories. `ros_src` contains only package source that must persist or be edited from the host. See `references/paths-and-naming.md` for the full boundary.

## Host and Container Layout

| Host Path | Container Path or Role | Purpose |
|---|---|---|
| `Desktop/Project/omni_ward_guide/compose.yaml` | Optional Docker/Compose metadata | Project-owned definition used only when persistent mounts or declarative lifecycle are required |
| `Desktop/Project/omni_ward_guide/ros_src` | `/omni_ws/src` | Persistent ROS package source |
| `Desktop/Project/omni_ward_guide/config` | `/config` | Optional external configuration |
| `Desktop/Dataset/<project>` | `/data` | Optional datasets or captured input |
| `Desktop/Model/<project>` | `/models` | Optional model artifacts |
| `Desktop/Runs/<project>` | `/runs` | Optional logs and application output |

The container owns `/omni_ws/build` and `/omni_ws/devel`. Keep those generated directories inside the container or a Docker volume. Do not create `/home/naviai/Desktop`, `Project`, `Dataset`, `Model`, or `Runs` inside the container.

## Existing Image Selection

When project-owned Compose is required, inspect the exact existing image before adding the service. Reuse its current repository and tag instead of creating a shorter local alias or a project-specific derived image. A blank container created by `naviai_container create` instead uses the exact local image of `naviai_demos`; see `references/commands.md`.

```bash
docker image inspect \
  10.51.33.201:30002/navi_project/environment:ros1_260310
```

Use `10.51.33.201:30002/navi_project/environment:ros1_260310` for a plain ROS Noetic project when its installed packages satisfy the application. If another existing container already has the required runtime, inspect its exact image name and contents before reusing that image:

```bash
docker inspect --format '{{.Config.Image}} {{.Image}}' <reference-container>
docker run --rm --pull never --entrypoint bash <exact-image> -lc \
  'source /opt/ros/noetic/setup.bash; rospack find <required-package>'
```

Direct image reuse does not add the NAVIAI custom interface installer. Verify every required package, such as `zj_robot`, `sensor`, `audio`, `upperlimb`, or `hand`, in the selected image. Prefer another existing image that already contains the dependency. Add a Dockerfile only when no existing image can satisfy an explicit image-layer dependency, and document why the exception needs a derived image.

The repository prefix displayed by Docker belongs to the image name. It does not determine a container's ownership or optional project-specific Compose name.

## Project-Owned compose.yaml

Skip this section when a blank standalone container from `naviai_container create` is sufficient. Use project-owned Compose only when the helper cannot express required mounts or lifecycle configuration.

```yaml
# /home/naviai/Desktop/Project/omni_ward_guide/compose.yaml
name: omni_ward_guide

services:
  omni_ward_guide:
    image: 10.51.33.201:30002/navi_project/environment:ros1_260310
    pull_policy: never
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

`container_name` is the final Docker name. Keep only this application's services in its project-owned file. A multi-container application can use `omni_ward_guide_ros`, `omni_ward_guide_web`, and similar role suffixes.

The original Orin shell exports `COMPOSE_PROJECT_NAME=navi_project` for the robot runtime. Because that environment variable can override the top-level `name`, pass an explicit application-specific name such as `-p omni_ward_guide`. Never run an application file as project `navi_project`.

Before adding another project or container, check whether the new component can share the existing environment and lifecycle. Related packages and nodes should normally remain in the same `omni_*` project or container. Split them only for a concrete boundary such as incompatible dependencies, independent deployment or restart, device permissions, resource isolation, or a distinct operational owner.

## Create and Enter

```bash
# Blank standalone development container
command -v naviai_container
naviai_container create omni_ward_guide
naviai_enter omni_ward_guide

# Or, when the project-owned Compose definition is required
cd /home/naviai/Desktop/Project/omni_ward_guide
docker compose -p omni_ward_guide up -d omni_ward_guide
naviai_enter omni_ward_guide
```

The helper creates a standalone SSH-enabled environment without bind mounts or Compose labels. The Compose option creates the container directly from the selected existing image; there is no `docker compose build` step. List it with `docker compose -p omni_ward_guide ps -a`. `sleep infinity` is only a development keepalive. With the Compose option, host-mounted package source and optional resources survive container deletion. `/omni_ws/build` and `/omni_ws/devel` are generated container workspace products.

## Create a ROS Package

Inside the container:

```bash
source /opt/ros/noetic/setup.bash

mkdir -p /omni_ws/src
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

If a container shows fewer Topics than expected, first verify its sourced ROS setup, `ROS_MASTER_URI`, `ROS_IP` or `ROS_HOSTNAME`, and hostname mappings against a known-working container. A wrong master can expose a different ROS graph; a wrong callback address can make a Topic visible but unreadable. Do not conclude that a publisher is missing until these settings are correct.

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
docker compose -p omni_ward_guide up -d omni_ward_guide
docker compose -p omni_ward_guide stop omni_ward_guide
```

Use `naviai_container remove <container>` for a standalone helper-created container. A Compose service can be recreated without changing its bind-mounted project data with `docker compose -p omni_ward_guide up -d --force-recreate <service>`.

## Other Lab Servers

| Network Condition | Practical Option |
|---|---|
| only Orin `192.168.5.200` reachable | rosbridge at `ws://192.168.5.200:9090` |
| bidirectional reachability to `192.168.217.0/24`; callback succeeds | native ROS container can run remotely |

For a remote native ROS container, set `ROS_IP` to an address robot nodes can reach. The current base image is arm64. An x86 host needs an x86 ROS Noetic image with matching interface types or deliberate cross-architecture emulation.

## Application Checklist

Apply this checklist to newly created projects. For an existing project, first follow its established structure and change it only when the user explicitly asks.

1. Project lives at `/home/naviai/Desktop/Project/omni_<business>`.
2. A blank standalone container is created with `naviai_container create`; if persistence or declarative lifecycle is required, its optional Compose definition lives in the owning project directory.
3. Container names use `omni_` and do not occupy `naviai_*` names.
4. The new container uses `/omni_ws`; a standalone container creates it internally, while project-owned Compose mounts host `ros_src` at `/omni_ws/src` when persistence is required.
5. A Compose service directly references an existing, inspected image with its exact repository and tag; the standalone helper reuses the exact `naviai_demos` image. Neither path creates a project-specific derived image by default.
6. ROS package, node, and application interfaces share the project prefix.
7. Host networking, ROS master, ROS IP, and hostname mappings are correct.
8. Host Dataset, Model, and Runs directories mount only when the node needs them, at simple paths such as `/data`, `/models`, and `/runs`.
9. No Desktop or Project/Dataset/Model/Runs hierarchy is created inside the container.
10. Privileged, GPU, and device access are absent unless needed.
11. Persistent apps use `restart: unless-stopped` and a real launch entry.
12. Control nodes implement timeout, cancellation, shutdown stop, and control ownership.
13. A new container is introduced only when the existing project or container cannot reasonably own the component.
14. New containers do not join a shared Compose project; when Compose is needed, commands explicitly pass an application-specific `-p omni_<business>` name.
