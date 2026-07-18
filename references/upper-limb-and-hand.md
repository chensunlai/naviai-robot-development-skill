# Upper-Limb and Dexterous-Hand Development

WA2 upper-limb and hand Services run on `pico.zjrx.com / 192.168.217.66`. No `/move_group` or `naviai_manipulation` container was observed, so these interfaces do not pass through a MoveIt planning scene. They directly expose named motions, joint moves, linear TCP moves, continuous servo, and drag teaching.

## Contents

- [State Before Control](#state-before-control)
- [Control Levels](#control-levels)
- [Named Motion](#named-motion)
- [MoveJ](#movej)
- [Upper-Limb Joint Limits](#upper-limb-joint-limits)
- [MoveL and Kinematics](#movel-and-kinematics)
- [Continuous Servo](#continuous-servo)
- [Hands](#hands)
- [Presets, Stop, and Protection](#presets-stop-and-protection)
- [Interface Checks](#interface-checks)

## State Before Control

| Topic | Type | Purpose |
|---|---|---|
| `/zj_humanoid/upperlimb/joint_states` | `sensor_msgs/JointState` | joint names, position, velocity, effort |
| `/zj_humanoid/upperlimb/uplimb_state` | `upperlimb/UplimbState` | mode and singularity state |
| `/zj_humanoid/upperlimb/occupancy_state` | `std_msgs/Int8` | upper-limb occupancy |
| `/zj_humanoid/upperlimb/tcp_pose/left_arm`<br>`/zj_humanoid/upperlimb/tcp_pose/right_arm` | `upperlimb/Pose` | TCP position, quaternion, RPY |
| `/zj_humanoid/upperlimb/jacobian/left_arm`<br>`/zj_humanoid/upperlimb/jacobian/right_arm` | `std_msgs/Float64MultiArray` | current Jacobian |
| `/zj_humanoid/hand/joint_states` | `sensor_msgs/JointState` | hand joints |
| `/zj_humanoid/hand/finger_pressures/left`<br>`/zj_humanoid/hand/finger_pressures/right` | `hand/PressureSensor` | finger pressure |
| `/zj_humanoid/hand/wrist_force_sensor/left`<br>`/zj_humanoid/hand/wrist_force_sensor/right` | `geometry_msgs/WrenchStamped` | wrist force/torque |

Capture the actual `joint_states.name[]` order. `MoveJ` and `servoj` arrays must follow the target interface order.

## Control Levels

| Level | Interface | Typical Use |
|---|---|---|
| Named motion | Action `/zj_humanoid/upperlimb/motion` | validated reusable motions |
| One joint-space move | Service `movej/<target>` | target joint angles |
| One linear TCP move | Service `movel/<target>` | target end pose |
| Path move | `movej_by_path/*`, `movel_by_path/*` | multiple joint or pose points |
| Continuous servo | `servoj/*`, `speedj/*`, `servol/*` Topics | high-rate controller output |
| Drag teaching | `teach_mode/*` Services and Actions | record/manage/replay human-guided paths |
| Hands | `gesture_switch/*`, `joint_switch/*` Services | named gesture or direct joints |

Targets include `left_arm`, `right_arm`, `dual_arm`; joint moves may also expose `neck`, `waist`, and `whole_body`. Check the live Service list for the current target set.

## Named Motion

Reference sequence:

1. `/zj_humanoid/upperlimb/motion/lists`: list files.
2. `/zj_humanoid/upperlimb/motion/load`: load `filename` under an `alias`.
3. Send Action goal `name`, `v`, `acc`, `t` to `/zj_humanoid/upperlimb/motion`.
4. Observe feedback/result plus `uplimb_state` and `occupancy_state`.
5. Optionally unload after use.

```python
import actionlib
import rospy
from upperlimb.msg import MotionExecuteAction, MotionExecuteGoal

rospy.init_node('upperlimb_motion_client')
client = actionlib.SimpleActionClient(
    '/zj_humanoid/upperlimb/motion', MotionExecuteAction)
if not client.wait_for_server(rospy.Duration(10.0)):
    raise RuntimeError('upperlimb motion server unavailable')

goal = MotionExecuteGoal(name='LOADED_ACTION_ALIAS', v=0.2, acc=0.2, t=0.0)
client.send_goal(goal)
if not client.wait_for_result(rospy.Duration(60.0)):
    client.cancel_goal()
    raise TimeoutError('upperlimb motion timed out')
print(client.get_state(), client.get_result())
```

Aliases are device-local and must come from `motion/loaded_lists`.

## MoveJ

`upperlimb/MoveJ` request:

```text
float64[] joints
float64 v
float64 acc
float64 t
bool is_async
int8 arm_type
```

Response: `success`, `message`. With `is_async=true`, Service return does not mean motion is complete; monitor state and joint error.

```python
import rospy
from upperlimb.srv import MoveJ, MoveJRequest

TARGET_JOINTS = []  # radians, in the current target's documented order
if not TARGET_JOINTS:
    raise ValueError('TARGET_JOINTS is not configured')

rospy.wait_for_service('/zj_humanoid/upperlimb/movej/left_arm')
movej = rospy.ServiceProxy('/zj_humanoid/upperlimb/movej/left_arm', MoveJ)
req = MoveJRequest(joints=TARGET_JOINTS, v=0.2, acc=0.2,
                   t=0.0, is_async=False, arm_type=1)
response = movej(req)
print(response.success, response.message)
```

## Upper-Limb Joint Limits

Upper-limb joints have position, velocity, and acceleration limits. The WA2-LS limit data is stored in [`wa2-ls-joint-limits.csv`](wa2-ls-joint-limits.csv).

All upper-limb operations must use the target interface's joint order and keep every commanded position, velocity, and acceleration within these limits. Do not send an upper-limb command or trajectory that exceeds a limit.

## MoveL and Kinematics

`upperlimb/MoveL` uses two `geometry_msgs/Pose` elements plus `v`, `acc`, and `is_async`. Dual-arm calls use both poses. Confirm how a single-arm server interprets the array, position units, frame, and quaternion.

`/zj_humanoid/upperlimb/FK/left_arm` and `FK/right_arm` are available. No corresponding IK Service was observed. Do not assume an arbitrary TCP pose is reachable.

## Continuous Servo

| Topic | Type |
|---|---|
| `/zj_humanoid/upperlimb/servoj/left_arm`<br>`/zj_humanoid/upperlimb/servoj/right_arm`<br>`/zj_humanoid/upperlimb/servoj/dual_arm`<br>`/zj_humanoid/upperlimb/servoj/whole_body` | `upperlimb/Joints` |
| `/zj_humanoid/upperlimb/speedj/left_arm`<br>`/zj_humanoid/upperlimb/speedj/right_arm`<br>`/zj_humanoid/upperlimb/speedj/dual_arm` | `upperlimb/SpeedJ` |
| `/zj_humanoid/upperlimb/servol/left_arm`<br>`/zj_humanoid/upperlimb/servol/right_arm`<br>`/zj_humanoid/upperlimb/servol/dual_arm` | `upperlimb/DualPose` |

Continuous control requires a stable publish rate, timeout stop, and control ownership. Enable speedj through `/zj_humanoid/upperlimb/enable_speedj`.

## Hands

| Service | Type | Request |
|---|---|---|
| `/zj_humanoid/hand/gesture_switch/left`<br>`/zj_humanoid/hand/gesture_switch/right`<br>`/zj_humanoid/hand/gesture_switch/dual` | `hand/Gesture` | `string[] gesture_name` |
| `/zj_humanoid/hand/joint_switch/left`<br>`/zj_humanoid/hand/joint_switch/right`<br>`/zj_humanoid/hand/joint_switch/dual` | `hand/HandJoint` | `float32[] q` |

Gesture names must exist on the device. Confirm direct-joint array length, order, and range.

## Presets, Stop, and Protection

| Service | Purpose |
|---|---|
| `go_home/left_arm`, `right_arm`, `dual_arm`, `neck`, `waist` | preset home |
| `go_home/whole_body` | selected target home |
| `go_down/left_arm`, `right_arm`, `dual_arm` | lower arms |
| `/zj_humanoid/upperlimb/stop` | stop motion |
| `/zj_humanoid/upperlimb/safety_lock` | safety lock |
| `/zj_humanoid/upperlimb/unlock` | unlock |
| `collision_detection/is_enable` | query collision detection |
| `collision_detection/enable` | configure collision detection |

Prefix relative names in this table with `/zj_humanoid/upperlimb/`. Coordinate stop, lock, and unlock through one application control owner.

## Interface Checks

```bash
rostopic echo -n 1 /zj_humanoid/upperlimb/joint_states
rostopic echo -n 1 /zj_humanoid/upperlimb/tcp_pose/left_arm
rostopic info /zj_humanoid/upperlimb/motion/goal
rosservice type /zj_humanoid/upperlimb/movej/left_arm
rossrv show upperlimb/MoveJ
rosservice call /zj_humanoid/upperlimb/motion/loaded_lists "{}"
```
