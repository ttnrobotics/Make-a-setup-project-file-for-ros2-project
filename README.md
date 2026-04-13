A `setup_project.sh` script helps you recreate your ROS environment on another computer with fewer mistakes.

For your ROS 2 Humble project, the script usually does these jobs:

1. checks Ubuntu/ROS environment
2. installs basic tools
3. installs dependencies with `rosdep`
4. builds the workspace
5. reminds you how to source it

---

# 1. Basic idea

Suppose your workspace is:

```bash
~/master_ros2_ws
```

and inside it you have:

```bash
~/master_ros2_ws/src
```

Then create a file:

```bash
~/master_ros2_ws/setup_project.sh
```

---

# 2. A good setup script template

Use this version first:

```bash
#!/bin/bash
set -e

echo "========== ROS 2 Project Setup Start =========="

# Workspace path
# WS=~/master_ros2_ws
WS="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ROS distro
ROS_DISTRO_NAME=humble

echo "Workspace: $WS"
echo "ROS distro: $ROS_DISTRO_NAME"

# 1) Check ROS installation
if [ ! -f /opt/ros/$ROS_DISTRO_NAME/setup.bash ]; then
    echo "ERROR: ROS 2 $ROS_DISTRO_NAME is not installed in /opt/ros/$ROS_DISTRO_NAME"
    exit 1
fi

# 2) Source ROS
source /opt/ros/$ROS_DISTRO_NAME/setup.bash

# 3) Install basic tools
echo "Installing basic tools..."
sudo apt update
sudo apt install -y \
    python3-pip \
    python3-rosdep \
    python3-colcon-common-extensions \
    python3-vcstool \
    build-essential \
    cmake \
    git

# 4) Initialize rosdep if needed
if [ ! -f /etc/ros/rosdep/sources.list.d/20-default.list ]; then
    echo "Initializing rosdep..."
    sudo rosdep init || true
fi

echo "Updating rosdep..."
rosdep update

# 5) Go to workspace
if [ ! -d "$WS/src" ]; then
    echo "ERROR: Workspace src folder not found: $WS/src"
    exit 1
fi

cd "$WS"

# 6) Install ROS dependencies from package.xml
echo "Installing ROS package dependencies from src..."
rosdep install --from-paths src --ignore-src -r -y

# 7) Optional: install Python requirements if file exists
if [ -f "$WS/requirements_python.txt" ]; then
    echo "Installing Python requirements..."
    python3 -m pip install -r requirements_python.txt
fi

# 8) Clean old build files if user wants
echo "Cleaning previous build/install/log folders..."
rm -rf build install log

# 9) Build workspace
echo "Building workspace..."
colcon build --symlink-install

# 10) Finish message
echo ""
echo "========== Setup Complete =========="
echo "Now run:"
echo "source /opt/ros/$ROS_DISTRO_NAME/setup.bash"
echo "source $WS/install/setup.bash"
```

---

# 3. Save it

Open terminal:

```bash
cd ~/master_ros2_ws
gedit setup_project.sh
```

Paste the script and save.

Then make it executable:

```bash
chmod +x ~/master_ros2_ws/setup_project.sh
```

Run it:

```bash
cd ~/master_ros2_ws
./setup_project.sh
```

---

# 4. What each part means

## `set -e`

```bash
set -e
```

This means:

* if one command fails, stop the script immediately

This is good because it prevents the script from continuing in a broken state.

---

## `WS=~/master_ros2_ws`

This sets your workspace path.

If your workspace name changes later, you only update one line.

---

## check ROS installation

```bash
if [ ! -f /opt/ros/$ROS_DISTRO_NAME/setup.bash ]; then
```

This checks whether ROS 2 Humble exists on that machine.

If ROS is not installed, the script stops.

---

## `source /opt/ros/humble/setup.bash`

This loads ROS 2 environment so commands like:

* `rosdep`
* `colcon`
* ROS package discovery

work correctly.

---

## install basic tools

This installs important development tools:

* `python3-rosdep`
* `python3-colcon-common-extensions`
* `git`
* `cmake`

Without these, many ROS builds fail.

---

## `rosdep install --from-paths src --ignore-src -r -y`

This is the most important line.

It reads your packages’ `package.xml` files and installs missing dependencies.

So instead of manually remembering:

* xacro
* robot_state_publisher
* joint_state_publisher
* controller_manager
* ros_gz_sim
* gazebo_ros

`rosdep` tries to install them automatically.

But this only works well if your `package.xml` files are correct.

---

## optional Python requirements

If your Python nodes use packages like:

* `numpy`
* `scipy`
* `transforms3d`

you can create:

```bash
~/master_ros2_ws/requirements_python.txt
```

Example:

```txt
numpy
scipy
transforms3d
pyyaml
```

Then the setup script installs them too.

---

## clean build

```bash
rm -rf build install log
```

This removes old machine-specific build files.

Very useful when moving project from laptop to desktop.

---

## `colcon build --symlink-install`

This builds the workspace.

`--symlink-install` is useful for development because:

* Python/scripts changes reflect faster
* easier to debug and edit

---

# 5. Even better version for your ROS robot projects

Since your projects often use:

* URDF/Xacro
* RViz
* ros2_control
* joint trajectory controller
* Gazebo / ros_gz

you can make a stronger version.

Example:

```bash
#!/bin/bash
set -e

echo "========== ROS 2 Robot Project Setup =========="

WS=~/master_ros2_ws
ROS_DISTRO_NAME=humble

source /opt/ros/$ROS_DISTRO_NAME/setup.bash

sudo apt update
sudo apt install -y \
    python3-pip \
    python3-rosdep \
    python3-colcon-common-extensions \
    python3-vcstool \
    build-essential \
    cmake \
    git \
    ros-humble-xacro \
    ros-humble-joint-state-publisher \
    ros-humble-joint-state-publisher-gui \
    ros-humble-robot-state-publisher \
    ros-humble-rviz2 \
    ros-humble-controller-manager \
    ros-humble-joint-trajectory-controller \
    ros-humble-joint-state-broadcaster \
    ros-humble-ros2-control \
    ros-humble-ros2-controllers \
    ros-humble-urdf \
    ros-humble-urdf-tutorial \
    ros-humble-tf2-tools

if [ ! -f /etc/ros/rosdep/sources.list.d/20-default.list ]; then
    sudo rosdep init || true
fi

rosdep update

cd "$WS"

if [ -f requirements_python.txt ]; then
    python3 -m pip install -r requirements_python.txt
fi

rosdep install --from-paths src --ignore-src -r -y

rm -rf build install log
colcon build --symlink-install

echo "Done."
echo "Run these commands next:"
echo "source /opt/ros/$ROS_DISTRO_NAME/setup.bash"
echo "source $WS/install/setup.bash"
```

This version is more practical for manipulator/robot simulation projects.

---

# 6. Add auto-source helper

You can also make another small file:

```bash
~/master_ros2_ws/source_me.sh
```

Content:

```bash
#!/bin/bash
source /opt/ros/humble/setup.bash
source ~/master_ros2_ws/install/setup.bash
```

Make executable:

```bash
chmod +x ~/master_ros2_ws/source_me.sh
```

Then whenever you open terminal:

```bash
source ~/master_ros2_ws/source_me.sh
```

This is easier than typing both lines every time.

---

# 7. Recommended file structure

A clean project can look like this:

```bash
master_ros2_ws/
├── src/
│   ├── master_ros2_interface/
│   ├── master_ros2_pkg/
│   └── my_robot1/
├── requirements_python.txt
├── setup_project.sh
└── source_me.sh
```

---

# 8. Important limitation

This script can only install what is properly declared.

So if your code uses something but you forgot to declare it in:

* `package.xml`
* `CMakeLists.txt`
* `requirements_python.txt`

then the desktop may still fail.

That is why the setup script is powerful, but only if your dependency declarations are clean.

---

# 9. Best habit for you

For every project, keep these 3 things:

## `package.xml`

for ROS dependencies

## `requirements_python.txt`

for pip Python dependencies

## `setup_project.sh`

for one-command setup

This is the best simple workflow before moving to Docker.

---

# 10. How to use on desktop

After copying your workspace to desktop:

```bash
cd ~/master_ros2_ws
chmod +x setup_project.sh
./setup_project.sh
source install/setup.bash
```

---

# 11. My recommendation for your current ROS work

Because your projects often involve:

* custom packages
* robot description
* ros2_control
* trajectory controllers
* Gazebo / RViz

I recommend you create:

1. `setup_project.sh`
2. `requirements_python.txt`
3. `source_me.sh`

That will already make your laptop-to-desktop workflow much better.

---

# 12. Small extra tip

If you use Git, add this `.gitignore`:

```gitignore
build/
install/
log/
*.pyc
__pycache__/
.vscode/
```

So you only move the real source files.
