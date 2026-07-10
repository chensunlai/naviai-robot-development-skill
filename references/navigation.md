# Mapping, Localization, and Navigation

## Contents

- [Active Navigation Chain](#active-navigation-chain)
- [Navigation Preconditions](#navigation-preconditions)
- [Navigation Goal](#navigation-goal)
- [Navigation State and Result](#navigation-state-and-result)
- [Python Actionlib Example](#python-actionlib-example)
- [Target Coordinates](#target-coordinates)
- [From No Map to Navigation](#from-no-map-to-navigation)
- [Successful Closed-Loop Criteria](#successful-closed-loop-criteria)

## Active Navigation Chain

The primary goal interface is the ROS1 Action:

```text
/zj_humanoid/navigation/navigation
navigation/NavigationAction
```

`/move_base_simple/goal` had no subscriber in the observed runtime and is not the primary entry.

| Order | Module | Input | Output |
|---:|---|---|---|
| 1 | MID360 driver in `naviai_sensor_lidar` | MID360 `192.168.217.17` | `/livox/lidar`, `/livox/imu` |
| 2 | Fast-LIO `/laserMapping` in `naviai_perception` | MID360 data, map `merged_map.pcd` | `navigation/odom_info`, `/cloud_registered_body`, `/odom_imu` |
| 3 | Local grid `/test_grid` | registered cloud, LIO odometry | `navigation/local_map` |
| 4 | `naviai_map_server` | selected map directory | `navigation/map` |
| 5 | `/navigation_node` in `naviai_navigation` | Action goal, global map, localization, local map | paths, state, `/calib_vel` |
| 6 | `/speed_manager` on `jzrobot-a` | `/calib_vel` | chassis execution |

The planner is a custom ROS1 `navigation` 1.3.0 Actionlib server. Cartographer, `/nav_manager_node`, and `/vtr/*` nodes on `jzrobot-a` belong to a separate external chain and are not current `/navigation_node` inputs.

## Navigation Preconditions

| Condition | Check | Provider |
|---|---|---|
| Map selected | `/zj_humanoid/navigation/get_cur_map_info` | `/nav_map_server_*` |
| Map contains data | `/zj_humanoid/navigation/map` | `/nav_map_server_*` |
| Localization updates | `/zj_humanoid/navigation/odom_info` | `/laserMapping` |
| Local obstacles update | `/zj_humanoid/navigation/local_map` | `/test_grid` |
| Action server exists | `/navigation_node` subscribes to `.../navigation/goal` | `/navigation_node` |
| Velocity reaches chassis | `/speed_manager` subscribes to `/calib_vel` | Orin to `jzrobot-a` |

One observed map was named `test`, with `0.05 m` resolution and dimensions `370 x 328`. Do not hardcode this mutable snapshot; query `get_cur_map_info`.

An application-specific education workflow also required head and waist home plus lowered dual arms before navigation:

```text
/zj_humanoid/upperlimb/go_home/neck       std_srvs/Trigger
/zj_humanoid/upperlimb/go_home/waist      std_srvs/Trigger
/zj_humanoid/upperlimb/go_down/dual_arm   std_srvs/Trigger
```

Treat that as a scenario preflight policy, not a universal requirement of the navigation server.

## Navigation Goal

| Field | Meaning |
|---|---|
| `header.frame_id` | coordinate frame; map goals use `map` |
| `task_type.value` | `Routine=0`, `Charge=1`, `Parking=2`, `Carry=3`, `Pushing=4`, `Towing=5` |
| `waypoints[]` | one or more target poses |
| `waypoints[].distance_tolerance` | arrival distance tolerance in meters |
| `waypoints[].heading_tolerance` | arrival heading tolerance in radians |
| `translation.enable` | enable terminal translation/heading adjustment |
| `translation.heading` | terminal target heading |

The current protocol temporarily reuses Header timestamp integers:

| Field | Current Meaning | Out-of-Range Behavior |
|---|---|---|
| `header.stamp.secs` | speed in cm/s, valid open interval `(0, 100)` | default `0.3 m/s` |
| `header.stamp.nsecs` | safety distance in cm, valid open interval `(0, 100)` | default `0.1 m` |

Centralize this non-time interpretation in the client so a protocol upgrade changes one place.

## Navigation State and Result

Feedback and result use the same `state.value` enumeration:

| Value | State | Meaning |
|---:|---|---|
| 0 | `Idle` | not executing |
| 1 | `Active` | goal accepted |
| 2 | `Running` | planning or moving |
| 3 | `Arrived` | intermediate arrival |
| 4 | `Canceling` | cancellation in progress |
| 5 | `Cancelled` | cancelled |
| 6 | `Succeeded` | succeeded |
| 7 | `Failed` | failed |
| 8 | `Error` | module error |
| 9 | `Aborted` | aborted |

Result also contains duration, `distance_deviation`, `heading_deviation`, and `causes[]`. Check both the Action terminal state and result `state.value`.

`/zj_humanoid/navigation/navigation_code` is module state: `IDLE=0`, `RUNNING=1`, `PAUSED=2`, `COMPLETED=3`, `ERROR=4`. It does not replace a goal result.

## Python Actionlib Example

```python
#!/usr/bin/env python3
import actionlib
import rospy

from navigation.msg import NavigationAction, NavigationGoal, Waypoint

ACTION = '/zj_humanoid/navigation/navigation'
TARGET_X = 0.0
TARGET_Y = 0.0
TARGET_QZ = 0.0
TARGET_QW = 1.0


def on_feedback(feedback):
    rospy.loginfo('navigation state=%d faults=%s',
                  feedback.state.value, feedback.faults)


rospy.init_node('naviai_navigation_client')
client = actionlib.SimpleActionClient(ACTION, NavigationAction)
if not client.wait_for_server(rospy.Duration(10.0)):
    raise RuntimeError('navigation action server unavailable')

goal = NavigationGoal()
goal.header.frame_id = 'map'
goal.header.stamp.secs = 30
goal.header.stamp.nsecs = 10
goal.task_type.value = goal.task_type.Routine

waypoint = Waypoint()
waypoint.pose.position.x = TARGET_X
waypoint.pose.position.y = TARGET_Y
waypoint.pose.orientation.z = TARGET_QZ
waypoint.pose.orientation.w = TARGET_QW
waypoint.distance_tolerance = 0.15
waypoint.heading_tolerance = 0.15
goal.waypoints = [waypoint]
goal.translation.enable = False
goal.translation.heading = 0.0

client.send_goal(goal, feedback_cb=on_feedback)
if not client.wait_for_result(rospy.Duration(180.0)):
    client.cancel_goal()
    raise TimeoutError('navigation timed out and was cancelled')

result = client.get_result()
if result.state.value != result.state.Succeeded:
    raise RuntimeError(
        f'navigation failed: state={result.state.value}, causes={result.causes}')
```

Before compiling, confirm that `rosmsg show navigation/NavigationGoal` includes `task_type`, `waypoints`, and `translation`.

## Target Coordinates

Targets must use the current map frame. Common sources:

1. Select a point or pose in RViz while displaying the map and TF.
2. Maintain named locations such as rooms, charging points, or presentation points in application configuration.
3. Save manually calibrated waypoints after mapping.

Pixel coordinates are not meter coordinates. `map.yaml` resolution, origin, and image-axis direction define conversion.

Cancel only the application's current goal with `cancel_goal()` when possible. `cancel_all_goals()` can affect other clients.

## From No Map to Navigation

Mapping, map selection, and navigation change robot state. Use an open area with emergency stop and manual takeover available.

### 1. Verify Inputs and Services

```bash
naviai_topic info /livox/lidar
naviai_topic hz /livox/imu
naviai_service info /perception/mapping_service
naviai_service info /perception/post_processing
```

Expected providers: `/livox_lidar_publisher2`, `/mapping_service`, `/post_processing_service`.

### 2. Start Mapping

```bash
naviai_utils_start_mapping <map_name>
```

Default request:

```yaml
map_name: <map_name>
z_floor: -0.1
z_ceil: 2.0
resolution: 0.05
scene: 0
```

| Field | Meaning |
|---|---|
| `map_name` | new directory name without spaces |
| `z_floor` | minimum retained cloud height, meters |
| `z_ceil` | maximum retained cloud height, meters |
| `resolution` | output 2D grid resolution, meters/cell |
| `scene` | Voxel-SLAM scene configuration; normal observed flow uses `0` |

`success: true` means the Voxel-SLAM process started, not that a map already exists.

### 3. Observe and Cover the Site

Move with the available manual-control method. Cover corridors, turns, doors, docking points, and planned goals. Avoid prolonged sharp spinning or excessive speed.

| Topic | Type | Purpose |
|---|---|---|
| `/map_odom` | `nav_msgs/Odometry` | mapping pose |
| `/map_scan` | `sensor_msgs/PointCloud2` | current scan |
| `/map_cmap` | `sensor_msgs/PointCloud2` | accumulated map cloud |
| `/map_pmap` | `sensor_msgs/PointCloud2` | processed map cloud |
| `/perception/mapping_code` | `module_common_msgs/ModuleStatus` | module state |
| `/perception/feedback` | `naviai_localization_msgs/Feedback` | status/error text |
| `/perception/mapping_result` | `naviai_localization_msgs/MappingResult` | one asynchronous end event |

In RViz, use `Fixed Frame=camera_init`, add `/map_cmap`, `/map_pmap`, and TF. `/map_odom` has previously reported `/camera_init` with a leading slash, which RViz may reject; point clouds and TF can still show progress.

Start `naviai_topic echo /perception/mapping_result` before stopping mapping if the event must be captured. Waiting with no new end event is normal.

### 4. Stop and Save

```bash
naviai_utils_stop_mapping
```

The default calls `/perception/post_processing` with `method: 0` for normal completion and map generation. Nonzero methods discard the current mapping result.

Map directory:

```text
/home/naviai/navi_project/containers/perception/datasets/<map_name>/
```

Required artifacts:

```text
map.yaml
map.pgm
merged_map.pcd
edge.txt
map/*.pcd
```

PGM/YAML drive the 2D map server; `merged_map.pcd` is loaded for Fast-LIO localization.

### 5. List and Load

```bash
naviai_utils_list_maps
naviai_utils_set_map <map_name>
```

Success commonly returns `code: 0`. `set_map` loads PGM/YAML in the map server and calls `/perception/lio_service` to start Fast-LIO with `merged_map.pcd`.

```bash
naviai_service call /zj_humanoid/navigation/get_cur_map_info "{}"
naviai_topic echo -n 1 /zj_humanoid/navigation/map_metadata
```

### 6. Verify Localization and Local Obstacles

```bash
naviai_topic info /zj_humanoid/navigation/odom_info
naviai_topic echo -n 1 /zj_humanoid/navigation/odom_info
naviai_topic info /cloud_registered_body
naviai_topic info /zj_humanoid/navigation/local_map
```

In RViz, use `Fixed Frame=map`, display the occupancy grid, `/cloud_registered_body`, and TF. Small robot motion should produce continuous, correctly oriented localization and cloud movement.

### 7. Verify or Start Navigation

`set_map` does not create the navigation container. Navigation is an independent Compose service.

```bash
docker ps --filter name=naviai_navigation
docker exec naviai_navigation bash -lc \
  'source /opt/ros/noetic/setup.bash; rosnode ping -c 1 /navigation_node'
naviai_topic info /zj_humanoid/navigation/navigation/goal
```

If the service has not been created in the WA2 deployment:

```bash
cd /home/naviai/navi_project
docker compose --profile wa2 up -d navigation
```

Normal wiring: `/navigation_node` subscribes to goal; `/speed_manager` subscribes to `/calib_vel`.

### 8. Send and Observe a Goal

Use an Action client and observe status, feedback, result, and module code. Set a timeout and cancel the goal if the timeout expires, the user cancels, or preconditions become invalid.

## Successful Closed-Loop Criteria

| Stage | Evidence |
|---|---|
| Mapping | post-processing succeeds; PGM, YAML, PCD, edge file exist |
| Map load | `get_cur_map_info.code == 0`; valid map dimensions/resolution |
| Localization | `/laserMapping` runs; `odom_info` updates continuously |
| Local perception | `local_map` updates with the environment |
| Planning | `/navigation_node` and Action server exist; all three inputs connected |
| Execution | Action result `Succeeded`; arrival deviations acceptable; chassis stops |
