# rosbridge WebSocket Development

`naviai_rosbridge` runs on the Orin. A WebSocket client can subscribe/publish Topics, call Services, and use ROS1 Actions through Actionlib Topics without installing ROS on the client host.

## Contents

- [Connection Parameters](#connection-parameters)
- [Observed Server Configuration](#observed-server-configuration)
- [JavaScript Connection](#javascript-connection)
- [Subscribe to a Topic](#subscribe-to-a-topic)
- [Call a Service](#call-a-service)
- [Use a Navigation Action](#use-a-navigation-action)
- [Query the ROS Graph Through rosapi](#query-the-ros-graph-through-rosapi)
- [Non-ROS Python](#non-ros-python)
- [Data-Volume Guidance](#data-volume-guidance)
- [Deployment Checklist](#deployment-checklist)

## Connection Parameters

| Client Location | URL | Notes |
|---|---|---|
| Process on Orin host | `ws://127.0.0.1:9090` | local process only |
| Development host on lab Wi-Fi | `ws://192.168.5.200:9090` | main lab endpoint |
| Host on robot-internal network | `ws://192.168.217.100:9090` | direct `192.168.217.0/24` access |
| Browser loaded over HTTPS | proxy to `wss://` | browsers block insecure mixed content |

The client does not set `ROS_MASTER_URI` or `ROS_IP`; the rosbridge container owns ROS networking.

## Observed Server Configuration

| Item | Value |
|---|---|
| Container | `naviai_rosbridge` |
| Nodes | `/rosbridge_websocket`, `/rosapi` |
| Listen | `0.0.0.0:9090` |
| ROS master | `http://192.168.217.1:11311` |
| Authentication | `authenticate=false` |
| Topic filter | `topics_glob=[*]` |
| Service filter | `services_glob=[*]` |

This exposes all ROS interfaces without application authentication. Keep it on a controlled robot network and expose only business-approved capabilities to end users.

## JavaScript Connection

```bash
npm install roslib
```

```js
import ROSLIB from 'roslib'

export const ros = new ROSLIB.Ros({
  url: 'ws://192.168.5.200:9090'
})

ros.on('connection', () => console.log('connected'))
ros.on('error', (error) => console.error('rosbridge error', error))
ros.on('close', () => console.log('disconnected'))
```

Store the URL in deployment configuration, for example `VITE_ROSBRIDGE_URL`.

## Subscribe to a Topic

```js
const battery = new ROSLIB.Topic({
  ros,
  name: '/zj_humanoid/robot/battery_info',
  messageType: 'sensor_msgs/BatteryState',
  throttle_rate: 500,
  queue_length: 1
})

const onBattery = (message) => {
  console.log(message.percentage, message.voltage)
}

battery.subscribe(onBattery)
// On component teardown: battery.unsubscribe(onBattery)
```

`throttle_rate` is milliseconds and limits only this WebSocket client's delivery rate.

## Call a Service

```js
const currentMap = new ROSLIB.Service({
  ros,
  name: '/zj_humanoid/navigation/get_cur_map_info',
  serviceType: 'map_server_msgs/GetCurMapInfo'
})

currentMap.callService(
  new ROSLIB.ServiceRequest({}),
  (response) => {
    if (response.code !== 0) {
      console.error(response.code, response.message)
      return
    }
    console.log(response.map_info)
  },
  (error) => console.error('service failed', error)
)
```

Wrap calls with an application timeout because a disconnected WebSocket or silent provider may never invoke the callback.

## Use a Navigation Action

```js
const navigation = new ROSLIB.ActionClient({
  ros,
  serverName: '/zj_humanoid/navigation/navigation',
  actionName: 'navigation/NavigationAction'
})

const goal = new ROSLIB.Goal({
  actionClient: navigation,
  goalMessage: {
    header: {
      frame_id: 'map',
      stamp: { secs: 30, nsecs: 10 }
    },
    task_type: { value: 0 },
    waypoints: [{
      pose: {
        position: { x: TARGET_X, y: TARGET_Y, z: 0 },
        orientation: { x: 0, y: 0, z: TARGET_QZ, w: TARGET_QW }
      },
      distance_tolerance: 0.15,
      heading_tolerance: 0.15
    }],
    translation: { enable: false, heading: 0 }
  }
})

goal.on('feedback', (feedback) => console.log('feedback', feedback))
goal.on('result', (result) => console.log('result', result))
goal.send()
// On timeout or user cancellation: goal.cancel()
```

The observed rosbridge environment has `navigation` 1.3.0 types. Clients still own timeout, cancellation, reconnect recovery, and concurrent-task arbitration. See `references/navigation.md` for field semantics.

## Query the ROS Graph Through rosapi

| Service | Purpose |
|---|---|
| `/rosapi/topics` | Topic names and types |
| `/rosapi/topic_type` | one Topic type |
| `/rosapi/services` | Service list |
| `/rosapi/service_type` | one Service type |
| `/rosapi/nodes` | node list |
| `/rosapi/node_details` | node publishers/subscribers/Services |
| `/rosapi/action_servers` | Action servers |
| `/rosapi/message_details` | message field structure |

```js
const topicType = new ROSLIB.Service({
  ros,
  name: '/rosapi/topic_type',
  serviceType: 'rosapi/TopicType'
})

topicType.callService(
  new ROSLIB.ServiceRequest({ topic: '/zj_humanoid/navigation/odom_info' }),
  (response) => console.log(response.type)
)
```

## Non-ROS Python

```bash
python3 -m pip install roslibpy
```

```python
from threading import Event
import roslibpy

client = roslibpy.Ros(host='192.168.5.200', port=9090)
client.run(timeout=10)
topic = roslibpy.Topic(
    client,
    '/zj_humanoid/robot/battery_info',
    'sensor_msgs/BatteryState')
received = Event()


def on_message(message):
    print(message['percentage'])
    received.set()


topic.subscribe(on_message)
if not received.wait(10.0):
    raise TimeoutError('no battery message received')
topic.unsubscribe()
client.terminate()
```

Long-running clients should reconnect and recreate subscriptions after reconnect.

## Data-Volume Guidance

| Data | Guidance |
|---|---|
| robot state, battery, mode, joints | suitable; throttle to UI rate |
| Services and low-rate Action feedback | suitable |
| compressed preview images | usable at limited rate/resolution/subscriber count |
| raw `sensor_msgs/Image` | prefer native ROS |
| MID360 custom cloud, `PointCloud2`, high-rate TF | use native ROS |

rosbridge converts ROS binary messages to JSON/CBOR over WebSocket, so high-volume sensor processing belongs in native ROS.

## Deployment Checklist

1. Configure URL per deployment and handle `ws` versus `wss`.
2. Unsubscribe when a component is removed.
3. Add timeout, error, and cancel behavior for Service/Action calls.
4. Recreate subscriptions and recover task state after reconnect.
5. Expose only approved business controls, not a generic ROS console.
6. Rate-limit image and state traffic.
