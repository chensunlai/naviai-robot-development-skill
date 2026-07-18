# Path and Naming Conventions

These conventions are defaults for newly created projects. An existing project or container keeps its established workspace name and location, directory structure, mounts, startup mechanism, and environment-source order. When an existing layout conflicts with this reference, follow the existing layout. Do not rename, move, or normalize it unless the user explicitly requests a migration or restructuring.

Contents:

- [Desktop Directory Roles](#desktop-directory-roles)
- [Project Directory](#project-directory)
- [Content Placement](#content-placement)
- [Host and Container Boundary](#host-and-container-boundary)
- [Naming Conventions](#naming-conventions)
- [Container Paths and Mounts](#container-paths-and-mounts)
- [Skill and Tool Extensions](#skill-and-tool-extensions)

## Desktop Directory Roles

| Host Path | Contents |
|---|---|
| `/home/naviai/Desktop/Project` | application-owned optional Compose definitions, exceptional Dockerfiles, configuration, documentation, rosbridge clients, non-ROS modules, and host-persisted source |
| `/home/naviai/Desktop/Dataset` | Datasets; data shared across projects lives in `Dataset/share` |
| `/home/naviai/Desktop/Model` | Weights, checkpoints, inference engines, and exported models; shared models live in `Model/share` |
| `/home/naviai/Desktop/Runs` | Logs, metrics, predictions, and other run artifacts |
| `/home/naviai/Desktop/Cache` | Machine-wide cache that can be downloaded or regenerated |
| `/home/naviai/Desktop/Tools` | Local management scripts, currently available through the user `PATH` |

The existing robot deployment lives in `/home/naviai/navi_project`. It is not the directory for new application projects and must not be reorganized to match this reference without an explicit request. This Desktop classification belongs to the Orin host filesystem; it is not a template for paths inside application containers.

## Project Directory

Use the following convention only for new projects. Existing projects retain their current names and paths.

```text
/home/naviai/Desktop/Project/omni_<business>
```

Typical host structure:

```text
/home/naviai/Desktop/Project/
`-- omni_ward_guide/
    |-- README.md
    |-- .gitignore
    |-- compose.yaml         # optional, owned only by this project
    |-- cache/
    |-- config/
    |-- scripts/
    |-- src/
    |-- ros_src/             # optional host-persisted ROS package source
    |-- data  -> /home/naviai/Desktop/Dataset/omni_ward_guide
    |-- model -> /home/naviai/Desktop/Model/omni_ward_guide
    `-- runs  -> /home/naviai/Desktop/Runs/omni_ward_guide
```

A native ROS project can keep package source on the host for editing and persistence:

```text
omni_ward_guide/
`-- ros_src/
    `-- omni_ward_guide/
```

For a new native ROS project, `ros_src` is mounted to `/omni_ws/src`; the Catkin workspace itself belongs to the container. Do not create a second Desktop-style hierarchy inside it.

On the original Orin host, `llm_create_pkg omni_ward_guide` creates the base host directories and the `data`, `model`, and `runs` links. The script uses relative paths, so run it from `/home/naviai/Desktop/Project` to produce this host layout. See `references/commands.md` for side effects.

## Content Placement

| Content | Location |
|---|---|
| Git-managed source and small configuration files | `Project/<project>` |
| Optional Compose definition for a project | `Project/<project>/compose.yaml` |
| Dockerfile for an explicitly justified derived-image exception | The owning application project root |
| Project scripts | `Project/<project>/scripts` |
| rosbridge clients and modules without a strong ROS build dependency | `Project/<project>/src` or another project-owned host directory |
| New-project ROS package source edited from the host | `Project/<project>/ros_src`, mounted at `/omni_ws/src` |
| Raw, annotated, and processed data | `Dataset/<project>`, accessed as `data` inside the project |
| Weights, checkpoints, ONNX files, and TensorRT engines | `Model/<project>`, accessed as `model` inside the project |
| Logs, metrics, predictions, and deliverables | `Runs/<project>`, accessed as `runs` inside the project |
| Data or models shared across projects | `Dataset/share`, `Model/share` |
| Disposable cache | Project `cache` or Desktop `Cache` |
| Machine-wide short commands | `Desktop/Tools` |

Do not copy datasets, models, or run results into the source tree or save them as Docker image layers. Inference data and models are normally mounted read-only; run output is mounted read-write.

## Host and Container Boundary

| Environment | Preferred Organization |
|---|---|
| Orin host | Use `Desktop/Project`, `Dataset`, `Model`, `Runs`, `Cache`, and `Tools` to keep Docker definitions, non-ROS/rosbridge code, large resources, and outputs discoverable. |
| New native ROS container | Use one normal Catkin workspace rooted at `/omni_ws`, with packages under `/omni_ws/src` and generated `build` and `devel` directories beside it. |

Do not create `Project`, `Dataset`, `Model`, `Runs`, or a Desktop hierarchy inside a ROS container. Container code should see ROS workspace paths, not host classification paths. Mount only what the process needs, using simple container locations such as `/config`, `/data`, `/models`, and `/runs`.

The application project remains the ownership boundary for version control, persistent source, and any optional Compose definition. New containers do not need to join a shared Compose project. The container remains the runtime environment for ROS. A bind mount can connect these boundaries without making their directory structures identical.

## Naming Conventions

| Object | Form | Example |
|---|---|---|
| Project | `omni_<business>` | `omni_ward_guide` |
| Container for a single-container project | `omni_<business>` | `omni_ward_guide` |
| Containers for a multi-container project | `omni_<business>_<role>` | `omni_ward_guide_ros` |
| Optional Compose name | project-specific `omni_<business>` | `omni_ward_guide` |
| Image | exact existing repository and tag | `10.51.33.201:30002/navi_project/environment:ros1_260310` |
| ROS package | `omni_<business>` | `omni_ward_guide` |
| ROS node | `/omni_<business>_<role>` | `/omni_ward_guide_executor` |
| Application-owned Topic or Service | `/omni/<business>/...` | `/omni/ward_guide/task_state` |

Current naming boundaries:

- `naviai_*`: existing robot runtime containers and services.
- `/zj_humanoid/*`: existing robot interface namespace.
- `omni_*` and `/omni/*`: new application projects and their interfaces.

Use the same business stem for the application directory, containers, and ROS package so processes, logs, and source can be traced back to one another. When Compose is needed, keep its definition in that application directory and use the same project-specific stem. The image name does not need that stem: project containers directly reuse an inspected existing image and keep its exact repository and tag.

For a blank standalone development container on the original Orin, prefer `naviai_container create`. When an application requires persistent mounts, declarative startup, or multiple services, define them in `/home/naviai/Desktop/Project/omni_<business>/compose.yaml`, set each final container name with `container_name: omni_<business>[_<role>]`, and invoke it with an explicit project-specific name such as `docker compose -p omni_ward_guide ...`. The explicit name prevents the host's `COMPOSE_PROJECT_NAME=navi_project` from grouping the application with the robot runtime.

Do not build or retag a derived image merely to make its visible name match the project. Add a Dockerfile only when no existing image contains a required image-layer dependency, and record that exception in the owning project.

## Container Paths and Mounts

Current example convention for native ROS projects:

| Host | Container | Suggested Access |
|---|---|---|
| `./ros_src` | `/omni_ws/src` | Read-write during development |
| `./config` | `/config` | Read-only |
| `Desktop/Dataset/<project>` | `/data` | Read-only by default |
| `Desktop/Model/<project>` | `/models` | Read-only for inference |
| `Desktop/Runs/<project>` | `/runs` | Read-write |

For a new native ROS container, `WORKDIR` should normally be `/omni_ws`. Catkin creates `/omni_ws/build` and `/omni_ws/devel`; these are container workspace products and do not need host Desktop classifications. Data, models, configuration, and output mounts are optional and should be omitted when the ROS node does not use them. When checkpoints must be written, make only the target subdirectory writable.

`/omni_ws` is the default for a new native ROS container. Existing containers may use `/navi_ws`, `/workspace`, a package-specific path, or another established layout; use that container's actual path and setup files instead of converting it to `/omni_ws`.

## Skill and Tool Extensions

- Put new local tools in `/home/naviai/Desktop/Tools`; the filename is the command name.
- Treat those tools as Orin-host utilities; do not assume they are available in containers or on migrated machines.
- Document path/layout changes in `references/paths-and-naming.md`.
- Document helper behavior and side effects in `references/commands.md`.
- Put new robot knowledge in the most specific bundled reference and update `references/documentation-index.md`.
