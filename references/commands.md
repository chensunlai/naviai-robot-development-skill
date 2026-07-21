# Local Command Reference

This page documents custom commands installed on the original Orin host under `/home/naviai/Desktop/Tools`. The Skill does not carry executable copies. These commands are host-only: do not assume they exist inside a container or on a machine to which the Skill was copied. Before using one, confirm that the current shell is on the Orin host and run `command -v <tool>`.

## Behavior Categories

| Category | Typical Commands | Meaning |
|---|---|---|
| Read-only query | `docker ps`, `docker top`, `naviai_topic info` | Inspect state without intentionally changing files or robot state |
| State change | `naviai_hosts`, `naviai_sync`, `naviai_container create`, mapping/map-selection tools, Service/Action controls, container start/stop | Modify files, containers, or robot runtime state |
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

For a new single-container ROS project, use `naviai_container create omni_ward_guide`; it creates the minimal host project, persisted `omni_ws`, and Compose definition together. Use `llm_create_pkg` only when the separate Dataset/Model/Runs classification and links are required and the project path does not already exist, because that helper does not handle an existing target safely. Do not run either project helper inside a container or reproduce the Desktop classification there.

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
| `naviai_sync ls` | List container names, states, and images |
| `naviai_sync <container>` | Merge and deduplicate host `authorized_keys` into one running container, overwrite-sync host Codex `config.toml` and `auth.json`, set secure root permissions, and verify without printing contents or checksums |
| `naviai_container create <name> [ssh_port]` | Reuse or create `/home/naviai/Desktop/Project/<name>/omni_ws` and `compose.yaml`, then use its `workspace` service to create the container with the whole-workspace and PulseAudio mounts and invoke `naviai_sync`; a new Compose file defaults to `10.51.33.201:30002/navi_project/demos:v1.0.2`, and omitting `ssh_port` selects an unused random port for a new file |
| `naviai_container remove <container>` | Force-remove exactly one named non-official container; an official container is rejected even when addressed by container ID |

`naviai_enter`, `naviai_copy`, and `naviai_hosts` reuse existing containers. `naviai_hosts` skips a hostname that already exists and does not validate or correct its IP. Changes disappear when the container is recreated. Use Compose `extra_hosts` for a persistent application container.

`naviai_sync` is a host-only, explicit one-container state change. The target must exist and be running. It stages all three readable host sources in a temporary container directory before installing them:

| Host Source | Container Destination | Behavior |
|---|---|---|
| `/home/naviai/.ssh/authorized_keys` | `/root/.ssh/authorized_keys` | Keep existing non-empty lines first, append only host lines not already present, and deduplicate the final file by exact non-empty line |
| `/home/naviai/.codex/config.toml` | `/root/.codex/config.toml` | Overwrite with the host file |
| `/home/naviai/.codex/auth.json` | `/root/.codex/auth.json` | Overwrite with the host file |

It installs `/root/.ssh` and `/root/.codex` as root-owned mode `0700`, installs all three files as root-owned mode `0600`, verifies staged and final exact-copy files using SHA-256 comparisons without printing hashes, verifies the merged `authorized_keys` against the generated merged file, and removes the temporary staging directory. A failure returns nonzero. It does not update `.bashrc`, restart the target, or modify host files.

`naviai_container create` has these implementation boundaries:

- New names must match lowercase Compose-compatible form `[a-z0-9][a-z0-9_-]*`. Docker Compose must be available.
- When no Compose file exists, it verifies local image `10.51.33.201:30002/navi_project/demos:v1.0.2` against expected image ID `sha256:a29db54844c1db6e3f61e856b9713d0de5c8b55f0493bc1a0985406b2a573e38`, generates `Project/<name>/compose.yaml` with `pull_policy: never`, and neither builds nor downloads an image. The generated default uses the image's Ubuntu 20.04 environment, root account, sshd, supervisor, host networking, NVIDIA runtime, privileged mode, robot hostname mappings, and recorded ROS master and host IP.
- The generated Compose file has project name and `container_name` equal to `<name>` and service name `workspace`. If the file already exists, the helper validates and reuses it without overwriting, so edited image, mount, environment, and startup settings become the source of truth. Keep the `workspace` service, the same `container_name`, and a numeric `NAVIAI_SSH_PORT` so the helper can validate startup.
- When generating a new Compose file, omitting the port selects a currently non-listening random host port in `20000-65535`; an explicit port remains supported. The selected port becomes sshd's actual listen port because host networking does not use Docker port publishing.
- When reusing an existing Compose file, the helper does not select a new port: it reads `NAVIAI_SSH_PORT` from the `workspace` service. If an explicit command-line port differs, it refuses the request and directs the user to edit the Compose file.
- The resulting command is `ssh -p <selected-port> root@localhost`. Authentication state comes from the fixed demos image; the helper does not create a new password or SSH key.
- The project path is fixed at `/home/naviai/Desktop/Project/<name>`. If the project directory, `omni_ws`, or Compose file already exists, the helper reuses it without recreating, clearing, or overwriting files. Missing paths are created.
- It creates these bind mounts:

  | Host | Container | Access |
  |---|---|---|
  | `/home/naviai/Desktop/Project/<name>/omni_ws` | `/omni_ws` | read-write |
  | `/home/naviai/.config/pulse/cookie` | `/root/.config/pulse/cookie` | read-only |
  | `/run/user/1000/pulse` | `/run/user/1000/pulse` | read-write |

- After Compose creates the container, the helper invokes `naviai_sync` for these host and container paths:

  | Host Source | Container Destination |
  |---|---|
  | `/home/naviai/.ssh/authorized_keys` | `/root/.ssh/authorized_keys` |
  | `/home/naviai/.codex/config.toml` | `/root/.codex/config.toml` |
  | `/home/naviai/.codex/auth.json` | `/root/.codex/auth.json` |

  Credential contents are not written into `compose.yaml`. This synchronization is a `naviai_container create` post-create step for both new and existing Compose files. Directly running `docker compose up` bypasses it; run `naviai_sync <container>` explicitly or use the helper when a recreated container must receive the current host files.

- After synchronization, the helper removes any previous `naviai_container`-managed ROS block from `/root/.bashrc` and appends exactly one current block:

  ```bash
  source /opt/ros/noetic/setup.bash
  if [ -f /omni_ws/devel/setup.bash ]; then
    source /omni_ws/devel/setup.bash
  fi

  export ROS_MASTER_URI="http://192.168.217.1:11311"
  export ROS_IP="${ROS_IP:-192.168.217.100}"
  ```

  The conditional workspace source avoids errors before the first Catkin build. Appending the managed block after the image's existing shell setup makes `/omni_ws` the final ROS overlay when its setup file exists. `ROS_MASTER_URI` is forced after sourcing so an image-provided `http://localhost:11311` cannot redirect the shell to a container-local graph; `ROS_IP` keeps its fallback behavior. Direct Compose creation bypasses this `.bashrc` initialization.

- Creation requires the PulseAudio cookie, `authorized_keys`, Codex config, and Codex auth files to be readable and the PulseAudio runtime directory to exist. The generated container uses `restart: unless-stopped`. A failed Compose, `naviai_sync`, `.bashrc` initialization, or sshd startup removes the failed container and removes only the Compose file and empty project paths created by that invocation; pre-existing files and non-empty data are retained.

`naviai_container create` is the default new single-container workflow. It creates a Compose-managed SSH development environment with the three generated mounts above. Edit the generated project-owned `compose.yaml` for additional mounts, another inspected local image, or a different startup command. After editing, use `docker compose -p <name> -f Project/<name>/compose.yaml up -d --force-recreate workspace`, or remove the container and run `naviai_container create <name>` again. Multi-service applications may extend the project file and use Docker Compose directly; they do not join the shared `navi_project`.

`naviai_container remove` resolves the supplied name or ID to the container's current canonical name, refuses names in its protected official snapshot, and then calls `docker rm --force` for that one container. The snapshot includes all containers present when the snapshot was taken, including exited `test_rosenv`: `naviai_chassis`, `naviai_demos`, `naviai_map_server`, `naviai_navbrain_ros`, `naviai_navigation`, `naviai_novnc`, `naviai_nviz`, `naviai_perception`, `naviai_robot`, `naviai_robot_viewer`, `naviai_rosbridge`, `naviai_sensor`, `naviai_sensor_lidar`, and `test_rosenv`. It has no bulk-delete command.

Removing a helper-created container deletes the container copies of SSH/Codex credentials with the container layer. It does not modify the host credential sources or remove the host project directory, `omni_ws` contents, or `compose.yaml`.

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
