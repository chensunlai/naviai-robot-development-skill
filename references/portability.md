# Portability and Snapshot Boundaries

This Skill is designed to be copied as one directory. Its references contain the robot architecture, interface snapshot, examples, and local conventions without requiring another documentation repository.

## What Is Self-Contained

- Architecture, network landmarks, container responsibilities, nodes, and mounts.
- Topic, Action, and Service names, types, ownership, and known state notes.
- Sensor, mapping/navigation, upper-limb/hand, audio, and display development examples.
- rosbridge and native Docker ROS templates.
- Project paths, naming conventions, command behavior, and typical trajectories.

## What Remains Runtime-Dependent

- Whether a container, node, publisher, Action server, or Service is currently alive.
- Current map names, map metadata, motion aliases, media files, device names, and network reachability.
- Installed message package version and exact fields if runtime software has been upgraded.
- Passwords, credentials, registry access, private media, maps, model weights, and source packages. These are intentionally not embedded.

## Source Snapshot

The facts describe a NAVIAI WA2-class deployment observed in July 2026. Important recorded values include:

```text
Orin lab address:       192.168.5.200
Orin internal address:  192.168.217.100
ROS master:             192.168.217.1:11311
Pico service device:    192.168.217.66
MID360:                 192.168.217.17
ROS:                    ROS1 Noetic
```

When the Skill runs on another host or inside a container, these are facts about the robot, not necessarily addresses assigned to the new execution environment.

## Host-Only Helper Commands

The Skill deliberately contains no executable helper scripts. Custom `naviai_*`, `llm_*`, `nviz`, and `setup` commands are available only on the original Orin host under `/home/naviai/Desktop/Tools`.

- A copied Skill or container must not assume these commands are installed.
- Use them only from a shell on the Orin host after `command -v <tool>` succeeds.
- Docker wrappers require access to the host Docker daemon.
- ROS and RViz wrappers expect the host's existing `naviai_nviz` container and display environment.
- Mapping helpers call state-changing robot Services except `list_maps`.
- Project helpers use `/home/naviai/Desktop` and are specific to the host layout.

Read `references/commands.md` before using a host helper. On migrated systems, use standard Docker, ROS, or filesystem commands instead.

## Updating the Snapshot

When a target robot differs, keep the stored snapshot and record verified differences rather than silently rewriting uncertain facts. Runtime introspection can update container state, publisher ownership, and type versions while preserving the distinction between documented baseline and observed target.
