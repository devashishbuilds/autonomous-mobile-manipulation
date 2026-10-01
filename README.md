# TIAGo Autonomous Mobile Manipulation – ROS 2

ROS 2 project developed for the TIAGo robot simulation using Gazebo, Nav2, SLAM, ArUco marker detection, and MoveIt 2.

The project is divided into three main tasks:

- **Task 1:** Autonomous exploration and map generation
- **Task 2:** Autonomous localization, environment coverage, ArUco marker detection, and storage of pick/place locations
- **Task 3:** Navigation to the stored locations and autonomous pick-and-place execution

The ROS 2 package is located at:

```text
tiago_ws/src/tiago_exam
```

## Repository Structure

```text
tiago_ws/
└── src/
    └── tiago_exam/
        ├── launch/
        │   ├── task1.launch.py
        │   ├── task2.launch.py
        │   ├── task3.launch.py
        │   └── tiago_exam.launch.py
        │
        ├── scripts/
        │   ├── task1 related scripts
        │   ├── task2 related scripts
        │   ├── task3 related scripts
        │   ├── auto_localize.py
        │   └── tuck_arm.py
        │
        ├── package.xml
        ├── CMakeLists.txt
        └── ...
```

> The exact content of `scripts/` depends on the final version of the project. Keep all Python nodes required by the three task launch files inside this directory.

## System

The project was developed with:

- Ubuntu
- ROS 2 Humble
- Gazebo 11
- TIAGo simulation packages
- Nav2
- SLAM
- MoveIt 2 / pymoveit2
- ArUco ROS
- Explore Lite

## Build

Clone the repository into a ROS 2 workspace or use the repository as the workspace itself.

```bash
cd ~/tiago_ws

source /opt/ros/humble/setup.bash

colcon build --symlink-install

source install/setup.bash
```

After rebuilding, source the workspace again:

```bash
source ~/tiago_ws/install/setup.bash
```

## Task 1 – Autonomous Exploration and Mapping

Task 1 launches the TIAGo simulation, controllers, topic relays, SLAM/navigation stack, RViz, and autonomous exploration.

The Task 1 implementation starts the Gazebo world and waits for the required controllers before enabling odometry, scan and navigation velocity relays. It then launches SLAM and `explore_lite` for autonomous map exploration.

Run:

```bash
cd ~/tiago_ws
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch tiago_exam task1.launch.py
```

Main launch file:

```text
tiago_ws/src/tiago_exam/launch/task1.launch.py
```

The Task 1 launch sequence includes:

1. Gazebo simulation
2. Mobile base controller
3. Arm controller
4. Torso controller
5. Head controller
6. Odometry relay
7. Laser scan relay
8. Navigation velocity relay
9. SLAM / Nav2
10. Explore Lite

Once exploration is complete, the map can be saved using:

```bash
ros2 run nav2_map_server map_saver_cli -f ~/tiago_ws/amr_exam/map
```

## Task 2 – Autonomous Marker Search and Location Registration

Task 2 uses the saved map and performs autonomous localization and navigation through the environment.

The robot searches for the two table ArUco markers:

```text
Pick table marker  : 26
Place table marker : 238
```

The task uses Nav2 waypoints and a 360-degree search rotation at each search position.

When a required marker is detected, the robot calculates a stand-off pose approximately **0.45 m in front of the marker**, faces the marker, and navigates to that pose.

The approach position is calculated using the marker orientation so that the robot approaches the printed marker face approximately perpendicular to it.

Run:

```bash
cd ~/tiago_ws
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch tiago_exam task2.launch.py
```

Main launch file:

```text
tiago_ws/src/tiago_exam/launch/task2.launch.py
```

Task 2 includes:

1. Gazebo / simulation startup
2. Required TIAGo controllers
3. Navigation and sensor topic relays
4. Arm tuck configuration
5. Saved-map navigation
6. AMCL localization
7. ArUco detection
8. Coverage waypoint navigation
9. 360-degree marker search
10. Marker approach using Nav2
11. Pick/place location storage

The detected navigation poses are stored in:

```text
~/tiago_ws/amr_exam/locations.yaml
```

These stored poses are then reused by Task 3.

## Autonomous Localization

The project contains an autonomous localization helper.

It calls global localization and moves the robot using alternating rotational and forward motion while monitoring AMCL covariance.

Localization is considered converged when the covariance values fall below the configured threshold.

Relevant script:

```text
tiago_exam/scripts/auto_localize.py
```

The localization helper uses:

```text
/amcl_pose
/reinitialize_global_localization
/mobile_base_controller/cmd_vel_unstamped
```

## Arm Tuck Configuration

Before navigation, the manipulator is placed in a compact configuration to avoid collisions and improve robot mobility.

Relevant script:

```text
tiago_exam/scripts/tuck_arm.py
```

MoveIt 2 is used with the `arm_torso` planning group and the `RRTConnectkConfigDefault` planner.

## Task 3 – Navigation and Pick-and-Place

Task 3 uses the locations recorded during Task 2.

Run:

```bash
cd ~/tiago_ws
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch tiago_exam task3.launch.py
```

Main launch file:

```text
tiago_ws/src/tiago_exam/launch/task3.launch.py
```

The Task 3 startup sequence contains:

1. Gazebo
2. TIAGo controllers
3. Odometry and laser scan relays
4. Arm tuck motion
5. Navigation with the saved map
6. Autonomous localization
7. ArUco detectors
8. Navigation to the required table
9. Pick-and-place manipulation sequence

The implementation uses the following ArUco markers:

```text
Pick table  : 26
Place table : 238
Cube        : 63
Cube        : 582
```

The robot loads the previously stored pose from:

```text
~/tiago_ws/amr_exam/locations.yaml
```

and sends a Nav2 `NavigateToPose` goal.

The robot verifies position and orientation tolerances together with current ArUco detections before continuing with manipulation.

## Navigation

The project uses the ROS 2 Nav2 stack.

Important navigation functionality includes:

- AMCL localization
- `NavigateToPose` action client
- goal cancellation
- saved-map navigation
- waypoint coverage
- distance-based early goal completion
- yaw tolerance checking
- marker-based stand-off poses

Typical navigation data includes:

```text
/map
/amcl_pose
/odom
/scan
/key_vel
navigate_to_pose
```

## ArUco Detection

ArUco markers are used to detect tables and cubes.

The TIAGo head RGB camera is used as the image source:

```text
/head_front_camera/rgb/image_raw
/head_front_camera/rgb/camera_info
```

Separate marker detection nodes are used for the required Task 3 markers to avoid topic conflicts.

## Manipulation

The manipulation part of the project uses MoveIt 2 / pymoveit2.

The robot uses the TIAGo arm and torso joints through the `arm_torso` planning group.

Motion planning is performed using:

```text
RRTConnectkConfigDefault
```

The manipulation sequence includes arm positioning, safe clearance motions, object interaction, transport, and placement.

## Useful Checks

Check active nodes:

```bash
ros2 node list
```

Check topics:

```bash
ros2 topic list
```

Check controllers:

```bash
ros2 control list_controllers
```

Check AMCL pose:

```bash
ros2 topic echo /amcl_pose
```

Check detected markers:

```bash
ros2 topic list | grep aruco
```

Check Nav2 action:

```bash
ros2 action list
```

## Git Repository Notes

Generated ROS 2 workspace directories should **not** be uploaded to GitHub:

```text
build/
install/
log/
```

Python cache and editor files should also be ignored.

A suitable `.gitignore` is:

```gitignore
# ROS 2 / colcon
build/
install/
log/

# Python
__pycache__/
*.py[cod]
*.pyo

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Temporary files
*.log
*.tmp
*.swp
*~

# Gazebo / ROS temporary data
.gazebo/
.ros/
```

Do **not** commit passwords, GitHub personal access tokens, private SSH keys, or other credentials.

## Author

Group 18

ROS 2 TIAGo Autonomous Mobile Robotics Project
