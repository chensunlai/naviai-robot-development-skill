---
name: naviai-robot-development
description: NAVIAI 机器人自包含本机知识与开发参考。用于快速了解 Docker/ROS 架构和网络，查询容器、节点、Topic、Action、Service、自定义 msg/srv/action 类型与功能链路，选择 rosbridge 或 Docker ROS 开发方式，创建和组织 omni 项目，区分宿主机 Project/Dataset/Model/Runs/Tools 分类与新建容器内 /omni_ws ROS 工作空间，了解仅限原 Orin 宿主机的工具命令，以及迁移或离线查询 NAVIAI 开发资料。用户询问“这台机器怎么组织”“代码或模型放哪里”“自定义消息包在哪里”“该用什么命令”“如何接入某项机器人能力”时使用。
---

# NAVIAI 开发参考

把本 Skill 作为可迁移的 NAVIAI 知识包。按问题选取相关 reference，提供已记录事实、可选方案和典型轨迹；轨迹用于说明常见组合，不表示唯一流程。

## 版本检查

每次对话首次使用本 Skill 时，通过 Git 检查是否为最新版；本地无 Git 信息时以上游 [chensunlai/naviai-robot-development-skill](https://github.com/chensunlai/naviai-robot-development-skill) 为准。如果有更新，建议用户进行更新以保证信息准确性；检查失败则继续使用本地版本。

## 当前测试不可用功能

- 显示屏：目前只能连接 Jetson，且 Jetson 启动成功后需重新拔插背部线缆才能正常显示；Pico 侧需厂家改线，因此所有显示屏相关节点目前均不可用。

## 参考文件

| 问题 | 读取 |
|---|---|
| 系统组成、网络和主要服务域 | `references/machine-overview.md` |
| 完整容器、节点、挂载和启动架构 | `references/runtime-architecture.md` |
| Topic 与 Action 类型、来源和作用 | `references/ros-topics-actions.md` |
| Service 类型、字段和作用 | `references/ros-services.md` |
| 自定义 ROS msg/srv/action 包、类型分类与宿主机归档位置 | `references/naviai-custom-types.md` |
| 相机、点云、IMU、位姿和力传感器 | `references/sensor-access.md` |
| 建图、地图、定位和导航 | `references/navigation.md` |
| 上肢、灵巧手和示教 | `references/upper-limb-and-hand.md` |
| ASR、TTS、音频、对话和头顶屏 | `references/audio-and-display.md` |
| rosbridge 开发 | `references/rosbridge-development.md` |
| Docker ROS 原生开发 | `references/docker-ros-development.md` |
| 跨能力任务编排 | `references/task-orchestration.md` |
| 宿主机与容器的路径边界、文件放置、命名和挂载 | `references/paths-and-naming.md` |
| 原 Orin 宿主机工具的语法和副作用 | `references/commands.md` |
| 开发方式概览 | `references/development-reference.md` |
| 资料总索引 | `references/documentation-index.md` |
| 典型信息与开发轨迹 | `references/typical-trajectories.md` |
| 迁移边界和快照说明 | `references/portability.md` |

本 Skill 不包含工具脚本。在原 Orin 宿主机上，Docker 和 ROS 操作优先使用 `/home/naviai/Desktop/Tools` 中经 `command -v` 确认可用的工具；无对应工具时再用原生命令。详见 `references/commands.md`。

Skill 根目录可能包含 `naviai_env.md`，它是机器连接信息的唯一来源。询问或需要连接信息时先读取该文件；若不存在，直接请用户提供并放到 Skill 根目录，不得搜索或推断其他线索。

## 使用原则

- 回答用户实际询问的层级；单个接口查询无需展开整机上手流程。
- 把典型轨迹当作示例路径，可从任意节点进入、删减或组合。
- 区分 Docker 容器、ROS 节点和物理设备。
- 区分 Skill 中的基线快照与目标机器当前运行态。
- 区分 Topic 名称存在、存在发布者和正在产生新消息三种状态。
- 接口名称、类型或字段要求精确时，先使用本 Skill 的完整接口表；目标 ROS 环境可用时再自省当前类型。
- 当前约定用 `omni_*` 标识新增项目，`naviai_*` 标识现有机器人运行栈；涉及后者时说明影响范围。
- 已有项目和容器的现状优先于本文档的新建规范。处理已有环境时沿用其工作空间名称与位置、目录结构、挂载、启动方式和 source 顺序；除非用户明确要求，不要重命名、迁移或重构已有结构。`omni_*`、`/omni_ws` 等规范只用于新建项目。
- 桌面 `Project/Dataset/Model/Runs/Tools` 分类仅用于宿主机；新建 ROS 容器默认使用根目录下的普通工作空间 `/omni_ws`，不要复制宿主机分类。
- 在原 Orin 上创建新的项目容器时，默认直接引用经 `docker image inspect` 确认存在的本地镜像，不为项目重复构建或重标记派生镜像；只有现有镜像都不能满足明确的镜像层依赖时，才增加 Dockerfile，并记录原因。
- 新增项目容器统一由 `/home/naviai/Desktop/Project/omni_project/compose.yaml` 管理，并归入 Compose project `omni_project`；执行 Compose 时显式使用 `-p omni_project`，避免宿主机的 `COMPOSE_PROJECT_NAME=navi_project` 把项目容器归入现有机器人运行栈。
- 规划开发环境时，若新功能与现有 `omni_*` project/container 的依赖、用途和生命周期相近，应提醒用户优先在同一项目或容器中扩展，减少功能相似的重复容器。只有环境冲突、独立部署重启、设备权限或资源隔离等边界明确时再拆分。
- 自定义工具只能在原 Orin 宿主机使用；容器和迁移机器使用标准 Docker、ROS 或系统命令。
- 仅查看任务使用只读命令。发布控制消息、调用状态变更 Service/Action、重启容器或编辑配置需要用户明确要求。

## 信息可信度

| 问题 | 优先证据 |
|---|---|
| 基线架构和接口语义 | 本 Skill 对应 reference |
| 目标机器当前进程、发布者和 Service | 目标 Docker/ROS 运行时 |
| 目标机器已安装工具的实际行为 | 目标机器上的脚本实现 |
| 当前消息与 Service 字段 | 目标 ROS 环境的 `rosmsg`、`rossrv`、Action 类型 |
| 未确认设备或能力 | 明确标注“暂未确认”，不要补成确定结论 |

## 自包含约束

完整知识应位于本 Skill 的 `references/` 内。不要把其他本地文档仓库、网页或网络资料当作 Skill 正常工作的必需依赖。发现新事实时，更新最相关的 bundled reference，避免在多个文件维护同一份长表。
