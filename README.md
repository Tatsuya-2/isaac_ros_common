# isaac_ros_common (vendored, lightly patched)

This directory is a **vendored copy of NVIDIA's upstream
[`isaac_ros_common`](https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common)
repository**, on the `release-3.2` line. It is not maintained by this
project except for a small set of local patches listed below.

It provides:

- The Dockerfiles and helper scripts used to build the development /
  production container for the Jetson Orin Nano (see
  `docker/Dockerfile.*`, `scripts/build_image_layers.sh`,
  `scripts/run_dev.sh`).
- A collection of upstream interface packages used by Isaac ROS
  components in this workspace (notably `isaac_ros_apriltag_interfaces`,
  `isaac_ros_nitros_bridge_interfaces`, `isaac_ros_tensor_list_interfaces`,
  `isaac_ros_nova_interfaces`, `isaac_ros_pointcloud_interfaces`,
  `isaac_ros_bi3d_interfaces`).
- Common test / launch utilities (`isaac_ros_test`,
  `isaac_ros_test_cmake`, `isaac_ros_launch_utils`,
  `isaac_ros_rosbag_utils`, `isaac_ros_r2b_galileo`).
- Python / C++ shared helpers (`isaac_common_py`, `isaac_common`).

## Upstream Source and Version

- Origin: `https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common`
  (Isaac ROS suite).
- Tracked via the private mirror
  `ssh://git@github.com/Tatsuya-2/isaac_ros_common`, branch
  `v3.2-orin-nano`, which sits on top of upstream `release-3.2`.
- License: `LICENSE` in this directory (Apache 2.0, NVIDIA).

## Local Modifications

Three functional local commits sit on top of upstream `release-3.2`
(plus one documentation commit that keeps this README in sync):

1. **`fix: allow release info change during apt-get update in Dockerfiles`**
   Adds `--allow-releaseinfo-change` to the apt update calls in
   `docker/Dockerfile.base` and `scripts/deploy/_Dockerfile.deploy` so
   builds do not fail when ROS / Ubuntu apt repos update their release
   info between rebuilds.

2. **`feat: add size field to AprilTagDetection message for pose
   estimation`**
   Adds a `float64 size` field to
   `isaac_ros_apriltag_interfaces/msg/AprilTagDetection.msg`:

   ```
   string family
   int32 id
   float64 size                    # tag size in meters used for pose estimation
   geometry_msgs/Point center
   geometry_msgs/Point[4] corners
   geometry_msgs/PoseWithCovarianceStamped pose
   ```

   The field is **populated on the published `AprilTagDetectionArray`**
   by the AprilTag detector
   (`isaac_ros_apriltag/.../apriltag_node.cpp`, `msg_detection.size =
   tag_size`) and by its simulator counterpart
   (`drone_simulator/.../drone_simulator_node.py`,
   `detection.size = ...`). It carries the physical tag size (meters)
   alongside each detection so downstream consumers can read tag size
   directly off the message. Current consumers in this workspace
   (`tagslam`, `flight_planner`, `gimbal_controller`, `drone_monitor`)
   read other fields of the detection (`id`, `pose`, `center`,
   `corners`); the `size` field is part of the wire contract but is not
   yet consumed by them.

3. **`feat: allow optional override of Docker image name in run_dev.sh`**
   Extends `scripts/run_dev.sh` so a `CONFIG_IMAGE_NAME` value in
   `.isaac_ros_common-config` overrides the default
   `isaac_ros_dev-<platform>` container/image name. This is what lets
   the project's bringup scripts launch the
   `dji-drone-production-container` directly.
   This commit also changes the aarch64 GPU/PVA option from
   `nvidia.com/gpu=all,nvidia.com/pva=all` to `nvidia.com/gpu=all`
   (PVA is not used by this project).

All other content matches the upstream `release-3.2` snapshot at the
time of vendoring.

## How This Project Uses It

The repository-root scripts under `jetson_prod/scripts/` drive these
files:

- `build-isaac-ros-image.sh` calls `scripts/docker_deploy.sh` here
  (which in turn drives `scripts/build_image_layers.sh`) to build the
  Isaac ROS base image, then layers `jetson_prod/docker/Dockerfile.jetson`
  on top via a plain `docker build`. It reads the config files in
  `jetson_prod/config/isaac/.isaac_ros_common-config` and
  `.isaac_ros_dev-dockerargs`.
- `run-isaac-ros-container.sh` calls `scripts/run_dev.sh` here, relying
  on the `CONFIG_IMAGE_NAME` override (local patch #3) to launch the
  container. With `CONFIG_IMAGE_NAME=dji-drone-production` in
  `.isaac_ros_common-config`, `run_dev.sh` names the container
  `dji-drone-production-container` (`<CONFIG_IMAGE_NAME>-container`).

## What We Depend On From the Interface Packages

Producers and consumers in this workspace include:

- `isaac_ros_apriltag` — **produces** `AprilTagDetectionArray` and sets
  the patched `size` field on each `AprilTagDetection`.
- `drone_simulator` — mock **producer** of `AprilTagDetectionArray`,
  also setting the `size` field.
- `tagslam`, `flight_planner`, `gimbal_controller`, `drone_monitor`,
  `system_tests` — **consume** `AprilTagDetection` /
  `AprilTagDetectionArray` (reading fields such as `id`, `pose`,
  `center`, `corners`, or detection count).

## Upstream Documentation

For everything not specific to this fork, see the upstream docs:
[Isaac ROS Documentation - isaac_ros_common](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_common/index.html).
For Isaac ROS troubleshooting, see
[Troubleshooting](https://nvidia-isaac-ros.github.io/troubleshooting/index.html).
