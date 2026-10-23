# robotms_ros2

The ROS 2 integration layer for robot-assisted TMS: tracking, hand-eye
calibration, head registration and robot control. It sits behind
[SlicerRoboTMS](https://github.com/m9h/SlicerRoboTMS) (3D Slicer front end)
and [NaviNIBS](https://github.com/PrecisionNeuroLab/NaviNIBS), talking to them
over OpenIGTLink.

Part of [open-neuronav](https://github.com/m9h/open-neuronav). Research
software, not a medical device. **Phantom heads only** until the safety layer
below exists and has been reviewed.

## Why this exists

SlicerRoboTMS (Bai et al., EMBC 2026, MIT) is the closest open project to an
open robotic TMS navigator. But it is only the application layer. Its
integration layer is unpublished, it was written for ROS 1 Noetic (end of
life May 2025), and its reference hardware is a Franka FR3. This project
provides:

* **a maintained ROS 2 OpenIGTLink bridge**:
  [m9h/ros2_igtl_bridge](https://github.com/m9h/ros2_igtl_bridge), forked from
  `openigtlink/ros2_igtl_bridge`. It builds and is tested on Humble, Jazzy,
  Lyrical and Rolling, with a crash on shutdown fixed and POLYDATA ported;
* **a SlicerRoboTMS branch for ROS 2**:
  [m9h/SlicerRoboTMS](https://github.com/m9h/SlicerRoboTMS) `ros2`, with its
  OpenIGTLink contract written down in `docs/IGTL-CONTRACT.md`;
* **the integration layer itself**, for ROS 2 and any arm with a ROS 2 driver,
  starting with the UFACTORY Lite 6.

## Layout (planned)

| package | does | status |
|---|---|---|
| `ros2_igtl_bridge` | OpenIGTLink ⇄ ROS 2 topics | ✅ fork; 12 launch tests; CI on 4 distros |
| `robotms_sim` | a simulated integration layer implementing the whole SlicerRoboTMS contract (status flags, joint transforms, calibration and registration results), so the Slicer side runs end to end with no hardware | ⏳ next |
| `robotms_tracking` | marker poses from a RealSense or webcam (AprilTag/ArUco), or from PlusServer (NDI Polaris) | ⏳ |
| `robotms_calibration` | hand-eye calibration (OpenCV `calibrateHandEye`) | ⏳ |
| `robotms_registration` | landmark registration (Kabsch/Horn) against `landmark_pts`; residuals reported | ⏳ |
| `robotms_bringup` | launch files; MoveIt 2 scene (head, chair, camera stand as collision objects); `moveit_servo` hold-on-target | ⏳ |
| `robotms_safety` | independent supervisor: speed and force limits, workspace box, stale-pose stop, e-stop and dead-man, no motion during a pulse | ⏳ **required before any human use** |

Robots come in through their own ROS 2 drivers (`xarm_ros2` for the Lite 6,
`ur_robot_driver`, `franka_ros2`, `kortex`), so supporting another arm is a
MoveIt configuration, not new control code.

## Development container

ROS 2 has no Fedora packages, so the ROS side runs in a container and talks
to Slicer, NaviNIBS or PlusServer on the host over TCP 18944.

    podman build -t robotms-ros2-dev:jazzy -f docker/Containerfile docker
    podman build -t robotms-ros2-dev:humble --build-arg ROS_DISTRO=humble -f docker/Containerfile docker

The image has ROS 2, OpenIGTLink (pinned to the commit Fedora's `openigtlink`
package and SlicerRoboTMS's submodule use) and `pyigtl` for tests.

## Licence

BSD-3-Clause for code written here (to match the bridge and ROS 2
convention). Forked components keep their own licences: the bridge is
BSD-3-Clause, SlicerRoboTMS is MIT.
