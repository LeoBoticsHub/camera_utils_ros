# camera_utils_ros (ROS 2 Humble)

Publishes RGB / depth / camera_info streams from Intel RealSense, Stereolabs ZED, and generic webcams as ROS 2 nodes.

> **Port from ROS Noetic.** This package is the **ROS 2 Humble** re-adaptation of the original `camera_utils_ros` (ROS 1 / catkin) by Federico Rollo — IASRobolab. The capture and publishing logic is unchanged; what changed is the ROS API, the build system, and QoS handling. See [What changed in the port](#what-changed-in-the-port).

---

## Table of contents

- [Architecture](#architecture)
- [Dependencies](#dependencies)
- [Installation](#installation)
- [Parameters](#parameters)
- [Usage](#usage)
- [Published topics](#published-topics)
- [QoS](#qos)
- [Compressed images](#compressed-images)
- [Docker](#docker)
- [What changed in the port](#what-changed-in-the-port)
- [Troubleshooting](#troubleshooting)

---

## Architecture

The project has three components, only two of which are ROS packages:

| Component | Type | Build | Role |
|---|---|---|---|
| `camera_utils` | Python library | `pip` | Camera abstraction classes: `IntelRealsense`, `Zed`, `Webcam`, `CameraInterface`. **Not** a ROS package. |
| `camera_utils_msgs` | ROS interface package | `ament_cmake` + `rosidl` | Defines the `Frames` message (rgb + depth + camera_info bundled in one message). |
| `camera_utils_ros` | ROS node package | `ament_python` | The `camera_publisher` node that grabs frames from the camera and publishes the topics. |

Workspace layout:

```
ros_ws/
└── src/
    ├── camera_utils/          # pip library (installed separately)
    ├── camera_utils_msgs/     # ament_cmake
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   └── msg/Frames.msg
    └── camera_utils_ros/      # ament_python
        ├── package.xml
        ├── setup.py
        ├── setup.cfg
        ├── resource/camera_utils_ros
        ├── camera_utils_ros/
        │   ├── __init__.py
        │   └── camera_publisher.py
        └── launch/
            └── camera_publisher.launch.py
```

---

## Dependencies

- **ROS 2 Humble** (Ubuntu 22.04)
- `cv_bridge` (`ros-humble-cv-bridge`)
- `image_transport` / `compressed_image_transport` (for compressed streams)
- `camera_utils` (pip library, see below)
- `numpy < 2` — **ABI constraint** required by `cv2` / `pyzed` / ROS on this stack
- Per camera:
  - **Intel RealSense** → `pyrealsense2`
  - **Stereolabs ZED** → **ZED SDK** + Python wrapper `pyzed`
  - **Webcam** → OpenCV only

---

## Installation

### 1. `camera_utils` library (pip)

This does **not** go through colcon. From its folder:

```bash
cd src/camera_utils
rm -rf build *.egg-info          # remove stale build artifacts (e.g. x86)
python3 -m pip install . --break-system-packages
```

### 2. ZED SDK (only if you use the ZED)

The `pyzed` wrapper cannot be installed via plain pip: it is provided by the ZED SDK installer, compiled against your CUDA/architecture. On x86 + CUDA 12 + Ubuntu 22.04:

```bash
wget -q -O ZED_SDK.run https://download.stereolabs.com/zedsdk/5.0/cu12/ubuntu22
chmod +x ZED_SDK.run
./ZED_SDK.run silent skip_cuda skip_drivers     # do NOT pass skip_python
rm -f ZED_SDK.run
```

Verify:

```bash
python3 -c "import pyzed.sl as sl; print('pyzed ok')"
```

### 3. Build the workspace

```bash
cd ros_ws
colcon build --packages-select camera_utils_msgs camera_utils_ros
source install/setup.bash
```

### 4. Pin NumPy

Run this **after** the ZED SDK and `camera_utils`, since both may pull in NumPy 2.x:

```bash
python3 -m pip install "numpy<2" --break-system-packages
```

### Final check

```bash
ros2 pkg executables camera_utils_ros          # -> camera_utils_ros camera_publisher
ros2 interface show camera_utils_msgs/msg/Frames
```

---

## Parameters

All parameters of the `camera_publisher` node, with node defaults:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `camera_type` | string | `""` | `intel`, `zed`, or `webcam`. **Required.** |
| `camera_resolution` | string | `HD` | Resolution (mapped onto `Camera.Resolution`). |
| `fps` | int | `30` | Frame rate requested from the camera. |
| `serial_number` | string | `""` | Camera serial (for multi-camera setups). |
| `device_idx` | int | `0` | Device index (`webcam` only). |
| `publish_rgb` | bool | `false` | Publish the RGB channel. |
| `publish_depth` | bool | `false` | Publish the depth channel. |
| `publish_camera_info` | bool | `false` | Publish `CameraInfo`. |
| `publish_separated_frames` | bool | `true` | `true` = separate topics; `false` = single `Frames` message. |
| `compressed_image` | bool | `false` | Publish RGB as `CompressedImage` (JPEG). |
| `rgb_topic` | string | `rgb_image_raw` | RGB topic name. |
| `depth_topic` | string | `depth_image_raw` | Depth topic name. |
| `camera_info_topic` | string | `camera_info` | CameraInfo topic name. |
| `frames_topic` | string | `camera_frames` | `Frames` topic name (when not separated). |
| `reliability` | string | `best_effort` | Image topics QoS: `best_effort` or `reliable`. |
| `qos_depth` | int | `5` | QoS queue depth. |

> The **launch file** sets some convenience defaults that differ (e.g. `camera_type:=zed`, `compressed_image:=true`, the three `publish_*` set to `true`). All are overridable from the CLI.

---

## Usage

### With `ros2 run`

```bash
ros2 run camera_utils_ros camera_publisher --ros-args \
  -p camera_type:=zed \
  -p publish_rgb:=true \
  -p publish_depth:=true \
  -p camera_resolution:=HD \
  -p fps:=15 \
  -p reliability:=best_effort
```

### With `ros2 launch`

```bash
ros2 launch camera_utils_ros camera_publisher.launch.py
```

With overrides:

```bash
ros2 launch camera_utils_ros camera_publisher.launch.py \
  camera_type:=intel \
  reliability:=reliable \
  fps:=30 \
  compressed_image:=false
```

### Per-camera examples

**ZED, RGB+depth compressed, best_effort:**

```bash
ros2 launch camera_utils_ros camera_publisher.launch.py \
  camera_type:=zed publish_rgb:=true publish_depth:=true \
  compressed_image:=true reliability:=best_effort
```

**Intel RealSense, depth only:**

```bash
ros2 run camera_utils_ros camera_publisher --ros-args \
  -p camera_type:=intel -p publish_depth:=true
```

**Webcam:**

```bash
ros2 run camera_utils_ros camera_publisher --ros-args \
  -p camera_type:=webcam -p publish_rgb:=true -p device_idx:=0
```

---

## Published topics

These depend on the `publish_*` flags and on `publish_separated_frames`.

| Topic (default) | Type | Condition |
|---|---|---|
| `rgb_image_raw` (or `.../compressed`) | `sensor_msgs/Image` or `CompressedImage` | `publish_rgb` + separated |
| `depth_image_raw` | `sensor_msgs/Image` | `publish_depth` + separated |
| `camera_info` | `sensor_msgs/CameraInfo` | `publish_camera_info` + separated |
| `camera_frames` | `camera_utils_msgs/Frames` | `publish_rgb` + `publish_depth` + **not** separated |

Depth encoding: `mono16` (Intel), `32FC1` (ZED, depth in meters as float32).

Runtime checks:

```bash
ros2 topic list
ros2 topic hz /rgb_image_raw
ros2 topic echo /camera_info --once
```

---

## QoS

Image topics use a `QoSProfile` configurable via the `reliability` and `qos_depth` parameters:

- `best_effort` — recommended for high frame-rate streams (dropped frames are not retransmitted). It is the sensor-data standard.
- `reliable` — guarantees delivery, but can add latency/backpressure on heavy streams.

**Publisher and subscriber must have compatible QoS.** If you publish `best_effort` and a downstream node (e.g. inference) subscribes as `reliable`, it **receives nothing**. Match both sides.

`CameraInfo` uses a dedicated, fixed QoS: `RELIABLE` + `TRANSIENT_LOCAL` (latched), so late-joining subscribers still get the intrinsics.

---

## Compressed images

With `compressed_image:=true`:

- The RGB topic becomes `<rgb_topic>/compressed` of type `CompressedImage` (JPEG).
- Depth stays uncompressed (JPEG compression makes no sense on 16-bit depth).

> **ZED note:** the ZED SDK returns **BGRA (4-channel)** frames, while JPEG encoding requires 3 channels (BGR). If compressed publishing fails or produces wrong images, make sure `camera_utils/cameras/Zed.py` converts to BGR (`cv2.cvtColor(..., cv2.COLOR_BGRA2BGR)`) before returning the frame.

---

## Docker

The node is meant to run in a dedicated container (ROS 2 Humble + ZED SDK + `numpy<2`), separate from the inference container, to avoid dependency conflicts (e.g. NumPy 1.x vs 2.x). The two containers communicate over ROS topics / DDS.

Typical camera-container launch (GPU and USB access for the ZED):

```bash
docker run -it --gpus all --privileged \
  -v /dev:/dev --network host \
  -e ROS_DOMAIN_ID=104 \
  husky_camera:humble

# inside the container
ros2 launch camera_utils_ros camera_publisher.launch.py reliability:=best_effort fps:=15
```

---

## What changed in the port

Main differences from the ROS Noetic version:

- **`rospy` → `rclpy`**: the node is a `Node` class; `rospy.init_node` replaced by `rclpy.init()` + node construction.
- **Parameters**: `rospy.get_param("~x", def)` → `declare_parameter` + `get_parameter().value`. The private `~` namespace no longer exists.
- **Publishers / QoS**: `queue_size` → `QoSProfile`. Added the `reliability` and `qos_depth` parameters.
- **`CameraInfo`**: the intrinsics matrix field is `k` (lowercase) in ROS 2, was `K` in ROS 1.
- **Time**: `rospy.Time.now()` → `self.get_clock().now().to_msg()`.
- **Loop**: `while not rospy.is_shutdown()` → `while rclpy.ok()` with `spin_once(timeout_sec=0)`.
- **Build**: `catkin` / `CMakeLists.txt` + `message_generation` → `ament_python` for the node and `ament_cmake` + `rosidl` for the messages.
- **Launch**: XML `.launch` → Python `.launch.py`. int/bool params must be wrapped with `ParameterValue(..., value_type=...)`.
- **Removed** the commented-out PointCloud block based on `ros_numpy` (not available in ROS 2; one would use `sensor_msgs_py.point_cloud2`).

---

## Troubleshooting

**`pyzed.sl module not found`**
The ZED SDK / Python wrapper is not installed. See [Installation → ZED SDK](#2-zed-sdk-only-if-you-use-the-zed). Do not pass `skip_python` to the installer.

**`numpy.core.multiarray failed to import` / `_ARRAY_API not found`**
NumPy 2.x conflicting with modules built for 1.x (cv2, pyzed). Fix:
```bash
python3 -m pip install "numpy<2" --break-system-packages
```

**`Findcatkin.cmake ... could not find catkin`**
The `CMakeLists.txt` / `package.xml` of `camera_utils_msgs` is still the catkin (ROS 1) version. They must be rewritten for `ament_cmake` + `rosidl`.

**`file 'camera_publisher.launch.py' was not found in the share directory`**
The launch file is not installed. Check that it lives in `launch/`, that it ends in `.launch.py`, and that `setup.py` has the `data_files` entry with `glob('launch/*.launch.py')`. Then rebuild.

**No data received downstream although the publisher is active**
QoS mismatch: align `reliability` between publisher and subscriber.

**`CAMERA NOT DETECTED` (ZED)**
Container started without GPU/USB access. Use `--gpus all --privileged -v /dev:/dev`.

---

## Credits

ROS 2 Humble port based on the original `camera_utils_ros` by Federico Rollo (IASRobolab — Leonardo Labs / IIT). License **GPLv3**.