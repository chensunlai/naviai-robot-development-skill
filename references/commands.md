# Local Command Reference

This page documents custom commands installed on the original Orin host under `/home/naviai/Desktop/Tools`. The Skill does not carry executable copies. These commands are host-only: do not assume they exist inside a container or on a machine to which the Skill was copied. Before using one, confirm that the current shell is on the Orin host and run `command -v <tool>`.

## Behavior Categories

| Category | Typical Commands | Meaning |
|---|---|---|
| Read-only query | `docker ps`, `docker top`, `naviai_topic info` | Inspect state without intentionally changing files or robot state |
| State change | `naviai_hosts`, `naviai_container create`, mapping/map-selection tools, Service/Action controls, container start/stop | Modify files, containers, or robot runtime state |
| Data deletion | `llm_remove_pkg`, `naviai_container remove` | Delete project data or a selected non-official container |

`docker exec`, `naviai_enter`, and `naviai_service` are not inherently read-only or state-changing. Their effect depends on the command or Service used afterward.

## Project Directory Tools

| Command | Run From | Current Behavior |
|---|---|---|
| `llm_create_pkg <project-name>` | `/home/naviai/Desktop/Project` | Create the project skeleton and data/model/runs links |
| `llm_mkddir <project-name>` | Target project root | Create the Dataset directory plus `data` and `data/share` links |
| `llm_mkmdir <project-name>` | Target project root | Create the Model directory plus `model` and `model/share` links |
| `llm_mkrdir <project-name>` | Target project root | Create the Runs directory and `runs` link |
| `llm_remove_pkg <project-name>` | `/home/naviai/Desktop/Project` | Recursively delete the project and same-name Dataset, Model, and Runs directories |
| `setup` | Uses the current directory as the classification root | Create classification directories and regenerate the `llm_*` scripts |

Important implementation details:

- `llm_create_pkg` does not check whether the target already exists and does not validate its path argument.
- `llm_mk*dir` uses `ln -sf`; it does not move content from an existing real directory, and an existing real target may not produce the expected link layout.
- `llm_remove_pkg` uses `rm -rf` without confirmation and deletes source, data, models, and results together.
- `setup` overwrites generated `Tools/llm_*` scripts and writes the PATH entry to `.bashrc` under the directory from which it runs. It does not automatically modify the actual `~/.bashrc` unless it runs from the home directory. This machine's real `~/.bashrc` already adds `Desktop/Tools` separately.

Typical new-project command: run `cd /home/naviai/Desktop/Project`, then `llm_create_pkg omni_ward_guide`. This creates a host-side classification only. Do not run it inside a container or reproduce its Desktop layout there; newly created native ROS containers default to `/omni_ws`.

## Container Tools

| Command | Current Behavior |
|---|---|
| `naviai_enter ls` | List every container name and state |
| `naviai_enter <container>` | Run an interactive `bash` in an existing container |
| `naviai_copy ls` | List every container name and state |
| `naviai_copy <container> <local-path> <container-path>` | Copy from the host into the container with `docker cp` |
| `naviai_copy -r <container> <container-path> <local-path>` | Copy from the container to the host with `docker cp` |
| `naviai_hosts ls` | List every container name and state |
| `naviai_hosts <container>` | Add `jzrobot-a` and `pico.zjrx.com` entries to the container `/etc/hosts` |
| `naviai_container create <name> [ssh_port]` | Create and start one blank development container from the exact local image used by `naviai_demos`; omit `ssh_port` to select a currently unused random port from `20000-65535` |
| `naviai_container remove <container>` | Force-remove exactly one named non-official container; an official container is rejected even when addressed by container ID |

`naviai_enter`, `naviai_copy`, and `naviai_hosts` reuse existing containers. `naviai_hosts` skips a hostname that already exists and does not validate or correct its IP. Changes disappear when the container is recreated. Use Compose `extra_hosts` for a persistent application container.

`naviai_container create` has these implementation boundaries:

- It reads the image name and image ID from the current `naviai_demos` container, verifies that the same image is still available locally, and runs with `--pull never`. It neither builds nor downloads an image.
- It uses the image's built-in Ubuntu 20.04 environment, root account, sshd, and supervisor. It runs with host networking, the NVIDIA runtime, privileged mode, the robot hostname mappings, and the observed ROS master and host IP values.
- With no port argument it selects a currently non-listening random host port in `20000-65535`; an explicit port remains supported. The selected port becomes sshd's actual listen port because host networking does not use Docker port publishing.
- The resulting command is `ssh -p <selected-port> root@localhost`. Authentication state comes from the `naviai_demos` image; the helper does not create a new password or SSH key.
- The container uses `restart: unless-stopped`. A failed sshd startup is treated transactionally: the helper prints recent logs and removes only the failed new container.

`naviai_container create` creates a standalone blank development container. It does not add bind mounts or Compose labels. Prefer it when those features are unnecessary. When persistent bind mounts, declarative startup, or multi-service lifecycle management are required, keep an optional project-owned `compose.yaml` in the application directory, directly reference the inspected existing image, and use a project-specific Compose name. There is no requirement to join a shared Compose project; this case has no matching custom helper, so native Docker Compose is appropriate.

`naviai_container remove` resolves the supplied name or ID to the container's current canonical name, refuses names in its protected official snapshot, and then calls `docker rm --force` for that one container. The snapshot includes all containers present when the snapshot was taken, including exited `test_rosenv`: `naviai_chassis`, `naviai_demos`, `naviai_map_server`, `naviai_navbrain_ros`, `naviai_navigation`, `naviai_novnc`, `naviai_nviz`, `naviai_perception`, `naviai_robot`, `naviai_robot_viewer`, `naviai_rosbridge`, `naviai_sensor`, `naviai_sensor_lidar`, and `test_rosenv`. It has no bulk-delete command.

Useful companion queries:

```bash
naviai_container list       # show all containers and official protection state
naviai_container official   # print the protected snapshot
```

## ROS Queries

`naviai_topic` and `naviai_service` load ROS Noetic and `/navi_ws/devel/setup.bash` when available inside `naviai_nviz`, then forward arguments to the native CLI.

```bash
naviai_topic list
naviai_topic type <topic>
naviai_topic info <topic>
naviai_topic echo -n 1 <topic>
naviai_topic hz <topic>

naviai_service list
naviai_service type <service>
naviai_service info <service>
```

`naviai_topic pub` publishes messages, and `naviai_service call` invokes a Service. There are no matching wrappers for node and type queries; use `rosnode`, `rosmsg`, and `rossrv` in a suitable ROS container.

Common read-only Docker queries include `docker ps`, `docker top <container>`, `docker logs --tail 100 <container>`, and `docker inspect <container>`.

## RViz

| Command | Current Behavior |
|---|---|
| `naviai_rviz` | Read the current `DISPLAY` or use `:1` when unset, then start blank RViz in the foreground inside `naviai_nviz` |
| `nviz` | Use `DISPLAY=:1` and start the preset RViz configuration in the background inside `naviai_nviz` |

Both commands reuse `naviai_nviz`; the RViz binary is `/opt/ros/noetic/bin/rviz` in that container. `naviai_rviz` first runs `xhost +SI:localuser:root`. The script does not revoke that X server authorization when RViz exits; revoke it manually with `xhost -SI:localuser:root` when no longer needed.

## Mapping and Map Tools

| Command | Current Behavior |
|---|---|
| `naviai_utils_start_mapping <map_name> [z_floor] [z_ceil] [resolution] [scene]` | Call `/perception/mapping_service` |
| `naviai_utils_stop_mapping [method]` | Call `/perception/post_processing`; default `method=0` |
| `naviai_utils_list_maps` | Call `/zj_humanoid/navigation/get_map_list` |
| `naviai_utils_set_map <map_name>` | Call `/zj_humanoid/navigation/set_map` |

See `references/navigation.md` for complete semantics and output. `list_maps` is a query; the other three commands change mapping or selected-map state.

## Availability Boundary

The custom helpers are not portable dependencies and are not installed by this Skill. On another machine or inside a container, use the equivalent native Docker, ROS, or filesystem command. Do not recreate or install the host helpers unless the user explicitly requests that work and the target environment has been checked.
