# Bot-ticelli | Autonomous Drawing Robot

**From visual perception to physical drawing: a TurtleBot3 that maps its surroundings, finds a safe workspace, recognizes a shape, and draws it autonomously.**

Bot-ticelli is a university robotics project built in **C++ with ROS Noetic, OpenCV, and a TurtleBot3 Waffle**. It integrates mapping and localization, occupancy-grid analysis, computer vision, path planning, navigation, and motion control in a modular system.

The robot can explore an unknown environment using SLAM or localize itself on a saved map using AMCL. Once it identifies a suitable obstacle-free area, it extracts a shape from a camera image and transforms its contour into a trajectory that the robot follows to reproduce the drawing on the floor.

## Project demonstrations

See Bot-ticelli in action in the three demonstration videos below. Click a thumbnail to watch the corresponding video on YouTube.

| Demo 1 | Demo 2 | Demo 3 |
| :---: | :---: | :---: |
| [![Bot-ticelli demonstration 1](https://img.youtube.com/vi/RLudQn7i9ZI/hqdefault.jpg)](https://www.youtube.com/watch?v=RLudQn7i9ZI) | [![Bot-ticelli demonstration 2](https://img.youtube.com/vi/Dro_8_x7c0A/hqdefault.jpg)](https://www.youtube.com/watch?v=Dro_8_x7c0A) | [![Bot-ticelli demonstration 3](https://img.youtube.com/vi/gTl-xQZ3lTM/hqdefault.jpg)](https://www.youtube.com/watch?v=gTl-xQZ3lTM) |
| [Watch on YouTube](https://www.youtube.com/watch?v=RLudQn7i9ZI) | [Watch on YouTube](https://www.youtube.com/watch?v=Dro_8_x7c0A) | [Watch on YouTube](https://www.youtube.com/watch?v=gTl-xQZ3lTM) |

*The demo labels are intentionally generic until their video content and titles can be verified.*

## Key capabilities

- **Mapping and localization:** Build 2D occupancy maps with SLAM or localize on pre-existing maps with AMCL.
- **Free-space selection:** Find a region large enough for the drawing without overlapping obstacles or unexplored cells.
- **Computer vision:** Segment images using Otsu's thresholding, morphological filtering, and contour extraction.
- **Trajectory planning:** Simplify and normalize contours, scale them to physical dimensions, and publish a ROS `nav_msgs/Path`.
- **Autonomous navigation:** Reach the drawing's starting point and align the robot before execution.
- **Motion control and safety:** Follow the path with lookahead-based control, handle sharp turns, and stop when unexpected obstacles are detected.

## System workflow

```text
LiDAR + map ──> Mapping / localization ──> Free-space selection ──┐
                                                                ├──> Path planning
Camera image ──> Segmentation ──> Contour extraction ───────────┘         │
                                                                         v
                                                            Navigation to start
                                                                         │
                                                                         v
                                                            Drawing path follower
```

## Architecture

The implementation is split into six cooperating C++ ROS nodes.

| Node | Role | Main interfaces |
| --- | --- | --- |
| `boticelli_interface_node` | Coordinates the application through a command-line interface, launches child ROS processes, and manages workflow transitions. | ROS commands and subprocesses |
| `free_space_explorer_node` | Finds a suitable free region in the occupancy grid and chooses a drawing center with obstacle clearance. | Subscribes to `/map`, `/scan`; publishes `/drawing_center` |
| `vision_node` | Segments the camera image, extracts the largest contour, simplifies it, and normalizes its coordinates. | Subscribes to `/camera/image/compressed`; publishes `/contour_points` |
| `drawing_planner_node` | Scales the detected contour to the requested dimensions and generates a global drawing trajectory. | Uses `/contour_points`, `/drawing_center`, `/contour_info`; publishes `/drawing_path` |
| `drawing_navigator_node` | Drives to the beginning of the planned path and aligns the robot's heading. | Uses `/cmd_vel`; signals `/drawing_ready` |
| `drawing_follower_node` | Executes the trajectory with lookahead-based control, sharp-corner handling, and emergency obstacle stops. | Uses `/scan` and `/cmd_vel` |

### Mapping and workspace selection

The robot uses **GMapping** for SLAM and **AMCL** when localizing against a previously created map. The workspace-selection node analyzes the occupancy grid to determine where the drawing can fit. It uses an **integral image** to test candidate regions efficiently, **connected-component analysis** to group valid locations, and a **multi-source BFS** distance calculation to prefer positions farther from walls and obstacles.

### Computer vision

The vision pipeline applies the following OpenCV operations:

1. Gaussian blur to reduce image noise.
2. Inverted Otsu thresholding for automatic binarization.
3. Morphological closing and opening to clean the mask.
4. Largest-contour extraction.
5. Douglas–Peucker simplification (`approxPolyDP`).
6. Normalization of contour coordinates to the range `[0, 1]`.

### Path planning and drawing

The planner transforms the normalized contour into a physical trajectory, preserving the shape's aspect ratio. The navigator uses a **finite-state machine** to reach and align with the first point. The follower then tracks the path using a **lookahead-based controller**, pausing and turning at sharp corners as needed. LiDAR-based obstacle checks provide an emergency stop during execution.

## Repository structure

After moving the ROS project out of its current `bot-ticelli/` wrapper folder, the intended repository layout is:

```text
.
├── README.md
└── botticelli_pkg/
    ├── CMakeLists.txt
    ├── package.xml
    ├── config/
    │   ├── boticelli.rviz
    │   ├── drawing_follower_params.yaml
    │   ├── drawing_navigator_params.yaml
    │   ├── drawing_planner_params.yaml
    │   └── vision_params.yaml
    ├── launch/
    │   ├── boticelli_draw.launch
    │   ├── boticelli_slam.launch
    │   ├── drawing_follower.launch
    │   ├── drawing_navigator.launch
    │   ├── drawing_planner.launch
    │   ├── drawing.launch
    │   ├── localization.launch
    │   └── vision.launch
    ├── maps/
    └── src/
        ├── boticelli_interface_node.cpp
        ├── drawing_follower_node.cpp
        ├── drawing_navigator_node.cpp
        ├── drawing_planner_node.cpp
        ├── free_space_explorer_node.cpp
        └── vision_node.cpp
```

> **Note:** The current private repository places `botticelli_pkg/` and its original README under `bot-ticelli/`. The tree above is the proposed public-facing layout, not a claim that the files have already been moved.

## Technology stack

| Area | Tools and methods |
| --- | --- |
| Language | C++ |
| Robot platform | TurtleBot3 Waffle, 360° LiDAR, Raspberry Pi Camera |
| Robotics middleware | ROS Noetic, Catkin, RViz |
| Mapping and localization | GMapping / SLAM, AMCL, occupancy grids |
| Computer vision | OpenCV, `cv_bridge`, Otsu thresholding, morphological operations, contour approximation |
| Planning and control | ROS paths, connected components, BFS, finite-state machines, lookahead-based tracking |

## Requirements

- Ubuntu 20.04 LTS with **ROS Noetic** and a working **Catkin** environment.
- TurtleBot3 packages and bringup configured for your robot.
- ROS dependencies including `geometry_msgs`, `nav_msgs`, `sensor_msgs`, `std_msgs`, `visualization_msgs`, `tf`, `gmapping`, `amcl`, and `map_server`.
- OpenCV and `cv_bridge`.
- A TurtleBot3 compatible with the project's configuration, including LiDAR and camera, for physical execution.

> ROS Noetic is an older ROS distribution. This project documents the original development environment; compatibility with newer ROS versions has not been established.

## Build

These instructions assume the proposed repository layout shown above. Clone the repository **outside** `catkin_ws/src`, then link the actual ROS package into the Catkin workspace:

```bash
mkdir -p ~/catkin_ws/src
cd ~
git clone https://github.com/olgaobiols/bot-ticelli-autonomous-robot.git
ln -s ~/bot-ticelli-autonomous-robot/botticelli_pkg ~/catkin_ws/src/botticelli_pkg
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

If the repository has not yet been renamed, replace the clone URL and local directory name with the current repository name. If you retain the original nested `bot-ticelli/` directory, point the symlink to `bot-ticelli/botticelli_pkg` instead.

## Run on TurtleBot3

The robot and the remote workstation must be connected to the same network and configured with the correct ROS master and host addresses. Replace the example placeholders below with the actual machine IP addresses.

### 1. Start the ROS master on the workstation

Configure the environment on the workstation:

```bash
export ROS_MASTER_URI=http://WORKSTATION_IP:11311
export ROS_IP=WORKSTATION_IP
roscore
```

### 2. Start TurtleBot3 bringup on the robot

Connect to the robot and configure its ROS environment:

```bash
ssh ubuntu@ROBOT_IP
export ROS_MASTER_URI=http://WORKSTATION_IP:11311
export ROS_IP=ROBOT_IP
roslaunch turtlebot3_bringup turtlebot3_robot.launch
```

### 3. Start the robot camera

In another terminal connected to the robot:

```bash
roslaunch turtlebot3_bringup turtlebot3_rpicamera.launch
```

### 4. Start remote bringup on the workstation

In another workstation terminal:

```bash
roslaunch turtlebot3_bringup turtlebot3_remote.launch
```

### 5. Launch Bot-ticelli

From a terminal where the Catkin workspace is sourced:

```bash
source ~/catkin_ws/devel/setup.bash
rosrun botticelli_pkg boticelli_interface_node
```

Follow the command-line interface prompts to coordinate the mapping, vision, planning, and drawing stages.

> **Validation status:** The commands above are adapted from the original project documentation and repository layout. They have not been re-tested on the physical robot for this README update. ROS network settings, launch files, executable targets, and package dependencies should be verified before labeling this a reproducible installation guide.

## Demonstration and results

The most useful additions to this README would be:

- A short video or GIF showing the robot recognizing and drawing a shape.
- A screenshot of the occupancy map in RViz.
- An example camera input alongside its segmented and simplified contour.
- A visualization of the planned path and the resulting physical drawing.

Only add quantitative performance claims (such as accuracy, completion time, or tracking error) when supported by measured results.

## Authors

This project was developed collaboratively as a university robotics project at the **Universitat Politècnica de Catalunya (UPC)** by:

- Olga Obiols Fuentes
- Pablo Abella Escudero
- Raul Castillo Moraz
- Blanca Mira Fradera
- Laida Queral Llopart
