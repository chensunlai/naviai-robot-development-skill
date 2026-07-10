# Capability and Task Orchestration

Robot capabilities are independently exposed through ROS. An application chooses interfaces, coordinates execution, waits for results, and handles timeout or failure.

## Capability Domains

| Capability | Input or Command | Result and State |
|---|---|---|
| Sensors | RealSense, MID360, joint, TCP pose, pressure, wrist-force Topics | images, clouds, IMU, JointState, pose, force |
| Navigation | `navigation/NavigationAction`, map Services | Action feedback/result, module state, path, odometry |
| Upper limbs | named-motion Action, MoveJ/MoveL Services, servo Topics | motion result, state, joints, TCP pose |
| Audio | ASR Topics, TTS/media Services | recognized text, Service response, generated file path |
| Head display | `zj_robot/FaceShow` Service | playback request result |

## Example: Navigate, Speak, and Move

| Stage | Operation | Success Evidence | Failure Handling |
|---|---|---|---|
| Accept | validate task ID, map, pose, text/media, motion alias | complete parameters, unique task ID | reject and record reason |
| Preflight | read mode, battery, selected map, localization, upper-limb state | correct map, live localization, available actuators | return precondition failure without control command |
| Navigate | send Navigation Action goal | result `Succeeded`, acceptable deviation | cancel on timeout; preserve faults/causes |
| Speak | call TTS or media Service | `success=true` | retry according to policy, then skip or terminate |
| Move | named-motion Action or explicit MoveJ/gesture Service | Action/Service success and acceptable final state | call upper-limb stop, end later motion stages |
| Display | call FaceShow | `success=true` | record error without blocking safe cleanup |
| Finish | persist stage times and responses | result stored and reported | ensure every Action is terminal or cancelled |

## State to Persist

| Data | Purpose |
|---|---|
| business task ID | idempotency and lookup |
| current stage | restart/recovery context |
| Action goal ID | correlate feedback/result/cancel |
| request snapshot | reproduce pose, map, motion, media inputs |
| raw code/message | preserve provider failure detail |
| timeout per stage | prevent indefinite Service/Action waits |

## Control Coordination Principles

- Observe state before sending a control request.
- Use an Action client API and wait for the server before a goal.
- Apply explicit timeout and retain complete Service responses.
- Give one application task control ownership of an actuator at a time.
- Cancel or stop a failed navigation/motion stage before cleanup.
- At task completion, cancel any still-active goal owned by the task.

## Directly Composable Entry Points

| Goal | Primary Entry |
|---|---|
| Query selected map | `/zj_humanoid/navigation/get_cur_map_info` |
| Navigate through waypoints | Action `/zj_humanoid/navigation/navigation` |
| Speak text | `/zj_humanoid/audio/tts_service` |
| Play WAV/media | `/zj_humanoid/audio/media_play` |
| Execute loaded motion | Action `/zj_humanoid/upperlimb/motion` |
| Control both-hand gesture | `/zj_humanoid/hand/gesture_switch/dual` |
| Play display media | `/zj_humanoid/robot/face_show/media_play` |
