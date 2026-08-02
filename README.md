# ROS2 机械臂项目（URDF + MoveIt）

一个基于 ROS2 的机械臂项目，分两个阶段完成：

1. **阶段一**：URDF/Xacro 模型描述与可视化（RViz）
2. **阶段二**：基于 MoveIt2 的运动规划仿真

---

## 项目结构

```
ros2_ws/
├── src/
│   ├── my_robot_description/        # 阶段一：机器人描述包
│   │   ├── urdf/
│   │   │   ├── arm.xacro            # 机械臂 xacro 宏
│   │   │   ├── common_properties.xacro
│   │   │   └── my_robot.urdf.xacro  # 顶层 URDF
│   │   ├── launch/
│   │   │   └── display.launch.xml   # 阶段一启动文件
│   │   ├── rviz/
│   │   │   └── urdf_config.rviz
│   │   └── package.xml
│   └── my_robot_moveit_config/      # 阶段二：MoveIt 配置包
│       ├── config/                  # SRDF、kinematics、joint_limits 等
│       ├── launch/
│       │   ├── demo.launch.py       # 阶段二启动文件
│       │   ├── move_group.launch.py
│       │   ├── moveit_rviz.launch.py
│       │   ├── rsp.launch.py
│       │   ├── spawn_controllers.launch.py
│       │   └── ...
│       └── package.xml
└── README.md
```

---

## 环境要求

- ROS2（建议 Humble / Iron / Jazzy 任一 LTS 版本）
- `colcon` 构建工具
- `xacro`
- `joint_state_publisher_gui`
- `robot_state_publisher`
- `rviz2`
- `moveit2`（`moveit_ros_move_group`、`moveit_ros_planning_interface` 等）

> 若使用 apt 安装 ROS2，可参考官方文档安装对应的 `ros-<distro>-desktop` 与 `ros-<distro>-moveit` 包。

---

## 构建项目

进入工作空间根目录，编译并加载环境：

```bash
cd ~/projects/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

> `--symlink-install` 表示对未编译的资源（如 launch 文件、urdf、rviz 配置）使用软链接，便于本地修改后无需重新编译即可生效。

如果只想编译单个包：

```bash
colcon build --symlink-install --packages-select my_robot_description
colcon build --symlink-install --packages-select my_robot_moveit_config
```

---

## 阶段一：URDF 可视化

通过 `robot_state_publisher` + `joint_state_publisher_gui` 在 RViz 中查看机器人模型，并可使用 GUI 滑块手动调整关节角度。

```bash
# 加载环境（每个新终端都需要）
source ~/projects/ros2_ws/install/setup.bash

# 阶段1 启动 URDF 可视化
ros2 launch my_robot_description display.launch.xml

# 阶段2启动 my_robot_bringup
ros2 launch my_robot_bringup my_robot.launch.xml

# 阶段3 调用moveit2规划运动
ros2 run my_robot_commander_cpp test_moveit  
ros2 run my_robot_commander_cpp commander  

#1)发送话题 /open_gripper 观察夹爪是否关闭
yishan@LAPTOP-1DTG0HVO:~/projects/ros2_ws$ ros2 topic pub -1 /open_gripper  example_interfaces/msg/Bool "{data: false}"
publisher: beginning loop
publishing #1: example_interfaces.msg.Bool(data=False)

2)发送话题 /joint_command 观察关节角度变化
yishan@LAPTOP-1DTG0HVO:~/projects/ros2_ws$ ros2 topic pub -1 /joint_command  example_interfaces/msg/Float64MultiArray "{data: [0.5,0.5,0.5,0.5,0.5,0.5]}"

3)发送话题 /pose_command 观察机械臂位姿变化
yishan@LAPTOP-1DTG0HVO:~/projects/ros2_ws$ ros2 topic pub -1 /pose_command  my_robot_interfaces/msg/PoseCommand "{x: 0.7,y: 0.0,z: 0.4,roll: 3.14,pitch: 0.0,yaw: 0.0,cartesian_path: false}"
# 开启笛卡尔路径规划
yishan@LAPTOP-1DTG0HVO:~/projects/ros2_ws$ ros2 topic pub -1 /pose_command  my_robot_interfaces/msg/PoseCommand "{x: 0.7,y: 0.0,z: 0.2,roll: 3.14,pitch: 0.0,yaw: 0.0,cartesian_path: true}"

```

启动后会出现：

- `joint_state_publisher_gui` 窗口：拖动滑块控制各关节
- RViz 窗口：实时显示机械臂模型，可切换不同视角查看

常见问题：

- 提示找不到包：确认已经 `source install/setup.bash`
- 模型显示不正确：检查 `urdf/my_robot.urdf.xacro` 引用的 xacro 文件路径

---

## 阶段二：MoveIt 仿真

在阶段一 URDF 基础上，配置 SRDF、kinematics、规划器、控制器等，使用 MoveIt2 进行运动规划与仿真。

```bash
# 加载环境（每个新终端都需要）
source ~/projects/ros2_ws/install/setup.bash

# 启动 MoveIt demo（启动 robot_state_publisher、move_group、RViz 等）
ros2 launch my_robot_moveit_config demo.launch.py
```

如果 demo 已在运行需要重启：

```bash
# 先关闭之前启动的节点
Ctrl + C

# 重新运行 demo
ros2 launch my_robot_moveit_config demo.launch.py
```

启动后可在 RViz 中使用 **MotionPlanning** 插件：

- **Planning** 选项卡：设置目标位姿，点击 `Plan` 规划路径，点击 `Execute` 执行
- **Scene Objects**：添加/移除障碍物，验证避障规划
- **Planned Path**：可视化规划轨迹

也可以通过命令行发送目标点：

```bash
# 查看可用的 action / topic
ros2 action list
ros2 topic list
```

---

## 常用命令

```bash
# 检查包是否被识别
ros2 pkg list | grep my_robot

# 查看 URDF 是否能正确解析
ros2 run xacro xacro src/my_robot_description/urdf/my_robot.urdf.xacro

# 查看当前 TF 树
ros2 run tf2_tools view_frames

# 查看话题
ros2 topic list
ros2 topic echo /joint_states
```

---

## 后续可能的工作

- 接入真实硬件或 Gazebo 仿真（需要 `gazebo_ros_control` 与 `joint_trajectory_controller`）
- 添加夹爪（gripper）并配置 MoveIt 抓取规划
- 配置 OMPL / Pilz 规划器参数
- 编写运动学插件（IKFast / MoveIt 插件）提升求解速度与稳定性
