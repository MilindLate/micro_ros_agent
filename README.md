# micro_ros_agent

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](#license)
[![ROS2](https://img.shields.io/badge/ROS2-Humble%20%7C%20Iron%20%7C%20Jazzy-22314E?logo=ros)](#requirements)
[![Platform](https://img.shields.io/badge/platform-Linux-orange?logo=linux)](#requirements)

> Bridge between microcontroller-based micro-ROS clients (ESP32, STM32, Pico, etc.) and the full ROS 2 / DDS network — the ground-station side of the stack that lets an MCU "speak ROS 2."

📺 **[Watch the live demo on YouTube ↓](#live-demo)**

---

## 📖 Repository Analysis

This repo currently ships as a single packaged source drop: **`micro-ROS-Agent.zip`**, containing the canonical [eProsima micro-ROS-Agent](https://github.com/micro-ROS/micro-ROS-Agent) package (v3.0.6), wrapped for use as a ROS 2 (`ament_cmake`) package named `micro_ros_agent`.

### What's inside the package

| Path | Purpose |
|---|---|
| `micro_ros_agent/src/main.cpp` | Entry point — parses CLI/launch args and starts the Agent |
| `micro_ros_agent/src/agent/Agent.cpp` | Core agent logic, wraps the Micro XRCE-DDS Agent |
| `micro_ros_agent/src/agent/graph_manager/` | Tracks micro-ROS client nodes and exposes them on the ROS 2 graph |
| `micro_ros_agent/src/agent/utils/demangle.cpp` | Type/name demangling utilities |
| `micro_ros_agent/include/agent/` | Public headers for the above |
| `micro_ros_agent/launch/micro_ros_agent_launch.py` | Ready-made ROS 2 launch file (defaults to UDP4, port 8888) |
| `micro_ros_agent/resource/` | Default FastDDS/rclcpp XML profiles |
| `micro_ros_agent/bin/*.py` | Build-time scripts that auto-generate XML interface profiles from your workspace's `.msg` files |
| `micro_ros_agent/CMakeLists.txt` | Superbuild that pulls in `microxrcedds_agent`, FastDDS, FastCDR, and `rosidl` typesupport |
| `snap/` | Snapcraft packaging (for distributing as a Snap) |
| `LICENSE`, `NOTICE`, `3rd-party-licenses.txt` | Apache-2.0 licensing + attribution |

### What the Agent actually does

The Agent is a **ROS 2 node that acts as a server between the DDS/ROS 2 network and micro-ROS nodes running on a microcontroller**. It:

- Receives and forwards messages between MCU-side micro-ROS clients and the ROS 2 graph
- Tracks connected micro-ROS nodes and exposes them as if they were native ROS 2 nodes
- Talks to clients over **UDP/TCP (IPv4 or IPv6)** or **Serial**
- Avoids re-sending full XML type definitions on every message by referencing pre-generated profiles (built at compile time from your workspace's message definitions)

In short: your MCU firmware (built with `micro_ros_arduino`, `micro_ros_esp_idf_component`, etc.) opens a transport (serial/UDP) to this Agent, and the Agent makes that MCU's publishers/subscribers/services visible to `ros2 topic list`, `rviz2`, `rqt`, and the rest of your ROS 2 stack.

### Dependencies (from `package.xml`)

```
rmw, rcutils, rmw_fastrtps_shared_cpp, rmw_dds_common, micro_ros_msgs
rosidl_typesupport_fastrtps_cpp, rosidl_runtime_cpp, rosidl_typesupport_cpp
fastcdr, fastrtps, fastrtps_cmake_module, microxrcedds_agent
```

> ⚠️ Note: the repo's root currently holds the package as a `.zip`, not as an unpacked, buildable colcon package. You'll need to unzip it into a workspace before building — covered in [Setup](#-full-setup-guide) below.

---

## 🎥 Live Demo

<a href="https://youtu.be/uLmXcxwyctk?si=302_CZiRScACyx9d">
  <img src="https://img.youtube.com/vi/uLmXcxwyctk/maxresdefault.jpg" alt="micro_ros_agent live demo" width="100%">
</a>

▶️ **[https://youtu.be/uLmXcxwyctk](https://youtu.be/uLmXcxwyctk?si=302_CZiRScACyx9d)**

---

## ⚙️ Requirements

- **OS:** Ubuntu 22.04 (Humble) or 24.04 (Jazzy) — Linux recommended; Agent also builds on Windows/macOS but this guide targets Linux
- **ROS 2:** Humble, Iron, or Jazzy (any distro with `micro_ros_setup` support)
- **Build tools:** `colcon`, `cmake` ≥ 3.5, a C++14-capable compiler (GCC/Clang)
- **Python 3** + `pip` (used by the XML profile generator)
- **Git**, **unzip**
- A micro-ROS **client/firmware** target (ESP32, STM32, Raspberry Pi Pico, Linux app, etc.) to actually talk to once the Agent is running

---

## 🚀 Full Setup Guide

### 1. Source ROS 2 and create a workspace

```bash
source /opt/ros/$ROS_DISTRO/setup.bash      # e.g. humble, iron, jazzy
mkdir -p ~/uros_ws/src
cd ~/uros_ws
```

### 2. Get this repo and unpack the Agent package into `src/`

```bash
git clone https://github.com/MilindLate/micro_ros_agent.git
cd micro_ros_agent
unzip micro-ROS-Agent.zip -d unpacked

# move the actual ament package into your colcon workspace
mv unpacked/micro-ROS-Agent/micro_ros_agent ~/uros_ws/src/
cd ~/uros_ws
```

### 3. Install dependencies with rosdep

```bash
sudo apt update
sudo rosdep init        # skip if already initialized
rosdep update
rosdep install --from-paths src --ignore-src -y
```

### 4. Build

The package uses a CMake **superbuild** (`MICROROSAGENT_SUPERBUILD=ON` by default), which automatically fetches and builds `Micro-XRCE-DDS-Agent`, FastDDS, and FastCDR for you — no need to install them system-wide first.

```bash
colcon build --packages-select micro_ros_agent
source install/local_setup.bash
```

> First build will take a while since the superbuild compiles FastDDS/FastCDR from source.

### 5. Run the Agent

**Direct CLI (recommended for first run):**

```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

Other transport options:

```bash
# Serial (USB-connected MCU, e.g. ESP32 on /dev/ttyUSB0)
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -b 115200

# TCP
ros2 run micro_ros_agent micro_ros_agent tcp4 --port 8888
```

**Using the included launch file:**

```bash
ros2 launch micro_ros_agent micro_ros_agent_launch.py
```

This defaults to `udp4 -p 8888 -v6` (verbosity level 6, useful for debugging).

> If you write your own launch file and pass multiple CLI args, pass them as a **single space-separated string**, not a list — the Agent's own CLI parser expects one token after `--ros-args`. E.g. `arguments: ["udp4 -p 8888"]`, not `["udp4", "-p", "8888"]`.

### 6. Connect your micro-ROS client

On the MCU side (flashed with `micro_ros_arduino`, ESP-IDF micro-ROS component, etc.), point your client's transport configuration at this Agent's IP and port (for UDP) or the correct serial device (for serial transport). Once connected, you'll see log lines confirming a new client session, and:

```bash
ros2 topic list
ros2 node list
```

...should show the MCU's topics/nodes appearing live.

### 7. (Optional) Verify with RViz/rqt

```bash
rviz2        # visualize sensor/pose topics published by the MCU
rqt_graph    # see the live node/topic graph including the micro-ROS client
```

---

## 🐳 Alternative: Run via Docker (no local ROS 2 install needed)

```bash
docker run -it --rm --net=host microros/micro-ros-agent:humble udp4 --port 8888
```

Swap `humble` for your target distro tag (`iron`, `jazzy`, etc.).

---

## 🛠️ Troubleshooting

| Issue | Fix |
|---|---|
| `find_package(microxrcedds_agent)` fails | Let the superbuild run (don't disable `MICROROSAGENT_SUPERBUILD`), or install Micro-XRCE-DDS-Agent system-wide first |
| Serial port packet loss inside Docker | Known upstream issue — prefer running the Agent natively or via the Snap, not Docker, for serial transport |
| Serial issues on the Snap build | Use the Dockerized or source-built version instead |
| `ros2 launch` arg parsing error | Combine all agent flags into one string argument (see step 5 note above) |
| MCU never shows up in `ros2 node list` | Double-check IP/port match between MCU firmware config and the Agent's bind address; confirm both are on the same network/subnet for UDP |

---

## 📜 License

Apache License 2.0 — see [`LICENSE`](https://github.com/MilindLate/micro_ros_agent/blob/main/LICENSE) and [`3rd-party-licenses.txt`](https://github.com/MilindLate/micro_ros_agent/blob/main/3rd-party-licenses.txt) inside the packaged source for full attribution. Original package by **eProsima**, as part of the [micro-ROS](https://micro.ros.org) project.

---

## ⚠️ Project Status

Per the upstream package: *this software is not production-hardened.* It hasn't been developed or tested for any single specific use case — adapt it to your needs, and validate thoroughly (e.g. against ISO 26262) before deploying in any safety-relevant setting.
