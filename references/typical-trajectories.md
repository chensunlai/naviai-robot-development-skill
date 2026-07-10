# Typical Reference Trajectories

These trajectories show how bundled information can be combined. They are not entry requirements. Start anywhere, skip known context, or combine multiple trajectories.

## Rapid Machine Orientation

Reference trajectory: `machine-overview -> runtime-architecture -> exact interface relevant to the task`.

Use `machine-overview.md` for the short model and `runtime-architecture.md` only when container, node, process, mount, or external-device detail matters.

## Create a Local Project

Reference trajectory: `omni_<business> host project -> choose rosbridge/non-ROS host code or native ROS container -> mount only required persistent resources`.

The host Desktop classification stores Docker definitions, non-ROS or rosbridge code, datasets, models, and results. A newly created native ROS container defaults to `/omni_ws`; it does not reproduce the host's `Project/Dataset/Model/Runs` layout. Existing containers retain their established paths. On the original Orin host, `llm_create_pkg` can generate host links for a new project. See `paths-and-naming.md` and `commands.md`.

## Develop a Non-ROS Application

Reference trajectory: `select rosbridge endpoint -> find Topic/Service/Action type -> build client -> handle connection, timeout, cancellation, result`.

Use `rosbridge-development.md`. High-rate images/clouds or a full ROS toolchain can go directly to `docker-ros-development.md`.

## Develop a Native ROS Application

Reference trajectory: `omni_* project and container -> ROS image/types -> network and hostnames -> workspace mounts -> interactive or persistent launch`.

Use `docker-ros-development.md`. Add GPU, USB, X11, `/dev`, and privileged access only when required.

## Find and Use an Interface

Reference trajectory: `ros-topics-actions or ros-services -> capability reference -> optional runtime introspection -> current message fields`.

When the complete interface name is known, search bundled references directly instead of reading every capability overview.

## Acquire Sensor Data

Reference trajectory: `sensor object -> publisher/type/rate/bandwidth -> rosbridge or native ROS -> project data or runs`.

Use `sensor-access.md` plus exact entries in `ros-topics-actions.md`.

## Navigate Starting Without a Map

Reference trajectory: `MID360 input -> Voxel-SLAM mapping -> post-process/save -> set map -> Fast-LIO localization -> local map -> navigation Action -> goal/result/cancel`.

When map and localization are already available, start at any later stage. All commands, fields, and success criteria are in `navigation.md`.

## Upper Limbs, Audio, and Display

Reference trajectory: `current state -> suitable high-level Service/Action -> request/result -> stop or cancel capability`.

- Upper limbs and hands: `upper-limb-and-hand.md`
- ASR, TTS, media, dialogue, display: `audio-and-display.md`
- Combined business task: `task-orchestration.md`

## Explain Current Runtime State

Reference trajectory: `bundled baseline + read-only target Docker/ROS introspection -> running / registered but silent / not started / unresolved`.

A single `rostopic list` does not prove every Topic is publishing. Identical images do not prove identical mounts, configuration, or startup commands.

## Move the Skill

Reference trajectory: `copy whole Skill directory -> read portability.md -> select bundled reference -> verify mutable runtime facts only when needed`.

No external documentation checkout is required for normal use.
