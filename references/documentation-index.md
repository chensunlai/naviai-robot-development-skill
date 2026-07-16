# Bundled Documentation Index

All normal knowledge dependencies are inside this Skill. Select the narrowest reference that answers the question.

## Architecture and Runtime

| Reference | Contents |
|---|---|
| `machine-overview.md` | short system model, network landmarks, main service domains |
| `runtime-architecture.md` | all containers, processes, nodes, mounts, external devices, startup model |
| `portability.md` | snapshot date, migration assumptions, runtime-dependent facts |

## Exact ROS Interfaces

| Reference | Contents |
|---|---|
| `ros-topics-actions.md` | Topic and Action names, types, publishers, purposes, inactive names |
| `ros-services.md` | Service names, types, request/response fields, providers, effects |
| `naviai-custom-types.md` | 19 custom interface packages, all archived msg/srv/action names, functional categories, and host paths |

## Capability Development

| Reference | Contents |
|---|---|
| `sensor-access.md` | RealSense, MID360, LIO, arm/hand state, pressure, force, examples |
| `navigation.md` | active chain, mapping, map load, localization, Action fields, examples |
| `upper-limb-and-hand.md` | named motion, MoveJ/MoveL, servo, hands, presets, protection |
| `audio-and-display.md` | ASR, TTS, playback, devices, LLM dialogue, Pico display |
| `task-orchestration.md` | compose navigation, speech, motion, and display into business tasks |

## Development Environments

| Reference | Contents |
|---|---|
| `development-reference.md` | comparison of rosbridge and native ROS environments |
| `rosbridge-development.md` | endpoints, JavaScript/Python clients, Topic/Service/Action, rosapi |
| `docker-ros-development.md` | image, type installer, Dockerfile, Compose, Catkin, startup |
| `paths-and-naming.md` | host Desktop classification, new-container `/omni_ws` default, existing-layout precedence, naming, mounts |
| `commands.md` | original Orin host commands, availability boundary, and side effects |
| `typical-trajectories.md` | optional reference paths for common tasks |

## Search Inside the Skill

From the Skill directory:

```bash
# Exact interface or keyword
rg -n '<keyword-or-interface>' SKILL.md references

# Container or node ownership
rg -n '<container-or-node>' references/runtime-architecture.md references/ros-topics-actions.md

# Host-only tool semantics
rg -n '<tool-name>' references/commands.md
```

## Runtime Interpretation

- A Topic in `rostopic list` has at least a publisher or subscriber registration; it may have no active publisher.
- A registered publisher can be event-driven and currently silent.
- A Service registration still depends on reaching its provider.
- An Action needs an active server subscribing to goal and publishing status/result.
- If the bundled snapshot and a target runtime differ, state both the baseline and the verified target observation.
