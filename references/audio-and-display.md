# Audio, Dialogue, and Head Display

## Contents

- [Audio Architecture](#audio-architecture)
- [Speech Input](#speech-input)
- [Text to Speech](#text-to-speech)
- [Audio File Playback](#audio-file-playback)
- [Audio Devices and Volume](#audio-devices-and-volume)
- [LLM Dialogue](#llm-dialogue)
- [Head Display](#head-display)
- [Interface Checks](#interface-checks)

## Audio Architecture

`naviai_navbrain_ros` provides audio Services. `/avvtn_node` handles microphone, wake, and recognition data. `/navbrain` provides TTS, file playback, device management, volume, and LLM dialogue.

## Speech Input

| Topic | Type | Relationship | Content |
|---|---|---|---|
| `/zj_humanoid/audio/listen` | `std_msgs/Bool` | `/avvtn_node` subscribes | start/stop listening |
| `/zj_humanoid/audio/listen_state` | `std_msgs/Bool` | `/avvtn_node` publishes | current listen state |
| `/zj_humanoid/audio/asr_text` | `std_msgs/String` | `/avvtn_node`, `/navbrain` publish | completed ASR text |
| `/zj_humanoid/audio/microphone/audio_data` | `audio/AudioData` | `/avvtn_node` to `/navbrain` | VAD/recognition stream |
| `/zj_humanoid/audio/microphone/wake_data` | `audio/AudioData` | `/avvtn_node` publishes | wake audio segment |
| `/zj_humanoid/audio/microphone/wake_info` | `avvtn_multi_demo/AudioWakeInfo` | `/avvtn_node` publishes | wake event |

`audio/AudioData` includes `channel`, `vad_status`, `is_final`, and `uint8[] audio_data`. Most applications only need `/asr_text`.

Reference ASR sequence: subscribe to `listen_state` and `asr_text`, publish `true` to `listen`, wait for state and a new result, then publish `false` when the listening turn ends.

```js
const listen = new ROSLIB.Topic({
  ros,
  name: '/zj_humanoid/audio/listen',
  messageType: 'std_msgs/Bool'
})
const asrText = new ROSLIB.Topic({
  ros,
  name: '/zj_humanoid/audio/asr_text',
  messageType: 'std_msgs/String'
})
asrText.subscribe((message) => console.log('ASR:', message.data))
listen.publish(new ROSLIB.Message({ data: true }))
```

One business-level owner should manage listening start, timeout, and completion rather than allowing multiple clients to toggle it concurrently.

### Wake Word Configuration

The live `naviai_navbrain_ros:v1.2.4` snapshot has two similarly named wake mechanisms. Do not treat them as interchangeable:

- `/avvtn_node` publishes `/zj_humanoid/audio/microphone/wake_info`. Its active four-microphone IVW resource is selected by `/navi_ws/src/avvtn_multi_demo/resource/vtn_4mic/vtn.ini` and currently resolves to `vtn_4mic/keywords.bin`.
- `/Nav/snowboy_vad/hotword` is currently `xiaolv.pmdl`, and `navbrain_ros` contains Snowboy code and that model. It is not the resource producing `/avvtn_node`'s `wake_info`; do not infer the live ROS wake phrase from this parameter.

In the observed image, `keywords.bin` is identical to `three.bin` and contains three phonetic entries: `xiao3 yi4 xiao3 yi4`, `ni3 hao3 xiao3 ai4`, and `xiao3 ai4 xiao3 ai4`. Treat each entry as one complete wake phrase and say that complete phrase once; only the first and third entries repeat a name within the phrase. For example, say `ni3 hao3 xiao3 ai4` once rather than repeating the whole phrase. No `xiao3 lv4` entry was found, so the live resource does not support “小律” as an evidenced wake phrase. The IVW result reports the matched entry in `AudioWakeInfo.keyword`.

An IVW wake phrase is a compiled vendor resource, not an editable string parameter. Use a compatible AVVTN/iFlytek-generated `.bin`; do not rename a Snowboy `.pmdl` or edit strings inside the binary. Reinitializing `/avvtn_node` through `/avvtn_node/restart` reloads the AVVTN process after a resource change. The Service request is empty and its response contains `bool success` and `string message`.

The AVVTN package and `vtn_4mic` resources are baked into the navbrain image, not bind-mounted in the observed Compose configuration. An edit made with `docker exec` survives a process or container restart but is lost when the container is recreated. For a persistent deployment, store the approved resource on the Orin host and bind-mount it over `/navi_ws/src/avvtn_multi_demo/resource/vtn_4mic/keywords.bin`, or build a derived image. Resource replacement and restart are global audio-stack changes and require explicit authorization.

## Text to Speech

Service `/zj_humanoid/audio/tts_service`, type `audio/TTS`:

```text
Request
  string[] text
  bool isPlay
Response
  bool success
  string message
  int32 status
  string file_path
```

`isPlay=true` generates and immediately plays; `false` only creates the file.

```python
import rospy
from audio.srv import TTS, TTSRequest

rospy.init_node('naviai_tts_client')
rospy.wait_for_service('/zj_humanoid/audio/tts_service', timeout=10.0)
tts = rospy.ServiceProxy('/zj_humanoid/audio/tts_service', TTS)
response = tts(TTSRequest(text=['Welcome to the NAVIAI robot.'], isPlay=True))
if not response.success:
    raise RuntimeError(f'TTS failed: {response.status} {response.message}')
print(response.file_path)
```

```js
const tts = new ROSLIB.Service({
  ros,
  name: '/zj_humanoid/audio/tts_service',
  serviceType: 'audio/TTS'
})
tts.callService(new ROSLIB.ServiceRequest({
  text: ['Welcome to the NAVIAI robot.'],
  isPlay: true
}), (response) => {
  if (!response.success) console.error(response.status, response.message)
})
```

## Audio File Playback

Service `/zj_humanoid/audio/media_play`, type `audio/MediaPlay`:

```text
Request:  string file_path
Response: bool success, string message, int32 status
```

The container resolves `file_path`. Host directory:

```text
/home/naviai/navi_project/containers/shared/audio/
```

is mounted as:

```text
/navi_ws/Audio/
```

Therefore host `audio/notice.wav` is requested as `/navi_ws/Audio/notice.wav`.

```python
import rospy
from audio.srv import MediaPlay

rospy.wait_for_service('/zj_humanoid/audio/media_play', timeout=10.0)
play = rospy.ServiceProxy('/zj_humanoid/audio/media_play', MediaPlay)
response = play('/navi_ws/Audio/notice.wav')
print(response.success, response.status, response.message)
```

Playback is queued and the Service waits for completion. Set client timeout longer than the media duration.

## Audio Devices and Volume

| Service | Type | Purpose |
|---|---|---|
| `microphone/get_devices_list` | `audio/GetDeviceList` | microphone names |
| `microphone/select_device` | `audio/SetDevice` | select microphone |
| `speaker/get_devices_list` | `audio/GetDeviceList` | speaker names |
| `speaker/select_device` | `audio/SetDevice` | select speaker |
| `speaker/get_volume` | `audio/GetVolume` | current volume |
| `speaker/set_volume` | `audio/SetVolume` | set integer volume |

Prefix with `/zj_humanoid/audio/`. Device and volume changes are global; manage them from one configuration owner.

## LLM Dialogue

Service `/zj_humanoid/audio/LLM_chat`, type `audio/LLMChat`:

```text
Request
  string raw_input
  bool enable_context
  bool enable_save
  string context_id
Response
  bool success
  string message
  int32 status
  string response
```

The response is text. Pass it separately to TTS for speech output.

## Head Display

Pico node `/video_display` provides:

```text
/zj_humanoid/robot/face_show/media_play
zj_robot/FaceShow
```

```text
Request
  string media_path
  bool loop
  float32 duration
Response
  bool success
  string message
  int32 status_code
```

`duration` is seconds; `0` means play to media completion. `media_path` is resolved on Pico. An Orin or development-container path is not automatically available on Pico; deploy media to Pico first and configure the Pico path.

```python
import rospy
from zj_robot.srv import FaceShow, FaceShowRequest

rospy.init_node('naviai_face_show_client')
rospy.wait_for_service('/zj_humanoid/robot/face_show/media_play', timeout=10.0)
face_show = rospy.ServiceProxy(
    '/zj_humanoid/robot/face_show/media_play', FaceShow)
response = face_show(FaceShowRequest(
    media_path='/home/robot/videos/notice.mp4',
    loop=False,
    duration=0.0))
if not response.success:
    raise RuntimeError(
        f'display failed: {response.status_code} {response.message}')
```

```js
const faceShow = new ROSLIB.Service({
  ros,
  name: '/zj_humanoid/robot/face_show/media_play',
  serviceType: 'zj_robot/FaceShow'
})
faceShow.callService(new ROSLIB.ServiceRequest({
  media_path: '/home/robot/videos/notice.mp4',
  loop: false,
  duration: 0.0
}), (response) => {
  if (!response.success) console.error(response.status_code, response.message)
})
```

Keep logical media names mapped to Pico paths in configuration instead of scattering device paths through business code.

## Interface Checks

```bash
rostopic info /zj_humanoid/audio/asr_text
rostopic echo /zj_humanoid/audio/listen_state
rosservice type /zj_humanoid/audio/tts_service
rossrv show audio/TTS
rosservice call /zj_humanoid/audio/speaker/get_volume "{}"
rosnode info /video_display
rosservice info /zj_humanoid/robot/face_show/media_play
rossrv show zj_robot/FaceShow
```

`/video_display/get_loggers` and `/video_display/set_logger_level` are ROS logging Services, not display controls.
