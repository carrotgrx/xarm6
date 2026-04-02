# SKILLS.md

## 1. 用途

本文件用于指导 Codex 在本新项目中执行**日常高频任务**。  
若 `AGENTS.md` 规定“做什么”，则本文件规定“怎么做更稳、更快、更少返工”。

本项目固定运行环境：

- Windows + WSL2
- **WSL 发行版：`Ubuntu`**
- 发行版内部系统版本：**Ubuntu 24.04**
- VS Code Remote - WSL
- ROS 2 Jazzy
- Gazebo Harmonic
- RViz2

---

## 2. 开始前环境确认

每次开始前先确认当前环境属于 **WSL 的 `Ubuntu` 发行版**，且当前位于项目根目录。

推荐命令：

```bash
uname -a
pwd
echo $WSL_DISTRO_NAME
```

期望结果：

- `WSL_DISTRO_NAME=Ubuntu`
- 当前目录为项目根目录

若不是 `Ubuntu`，先停止操作并提示切回正确发行版。

---

## 3. 标准构建流程

优先使用系统 Python，避免 Conda 干扰。

```bash
cd <project-root>
conda deactivate 2>/dev/null || true
export PATH=/usr/bin:/bin:/usr/sbin:/sbin:$PATH
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --cmake-args -DPython3_EXECUTABLE=/usr/bin/python3
source install/setup.bash
```

如果只构建局部模块，可按需使用 `--packages-select`。

---

## 4. 新项目起步时的优先动作

新项目不是旧项目复制版。开始任务时优先做下面几件事：

1. 确认当前主 URDF 路径与主基准文件。
2. 确认末端参考点、末端参考坐标系的命名与定义。
3. 确认电机参数配置路径。
4. 确认 Gazebo 是不是唯一机械执行真源。
5. 确认 FOC / LQR 是否仍被误接到现成控制库。
6. 确认文档目录与代码目录是否已经建立一一对应关系。

---

## 5. Gazebo / RViz / ROS 启动前清理

若出现 DDS SHM 占用、旧节点残留、时间链异常等问题，先清理：

```bash
pkill -f rviz2
pkill -f ros2
pkill -f gz
pkill -f gazebo
pkill -f move_group
pkill -f robot_state_publisher
pkill -f controller_manager
pkill -f ros_gz
pkill -f can_bus_router
pkill -f can_bus_r
fastdds shm clean
```

再重新启动。

---

## 6. 推荐启动环境变量

从项目根目录启动前建议：

```bash
source /opt/ros/jazzy/setup.bash
source install/setup.bash
export GALLIUM_DRIVER=d3d12
export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA
export QT_QPA_PLATFORM=xcb
```

若项目使用 Gazebo 资源路径，再按项目文档设置 `GZ_SIM_RESOURCE_PATH`。

---

## 7. 控制问题优先排查顺序

遇到“机械臂不能正确到达目标点”时，必须按以下顺序排查。

### 第一步：状态链

检查：

```bash
ros2 control list_controllers
ros2 topic echo /joint_states --once
ros2 node info /robot_state_publisher
```

目标：

- `joint_state_broadcaster` active
- 轨迹控制器 active
- `/joint_states` 正常发布

### 第二步：时间链

检查：

```bash
ros2 topic info /clock
ros2 param get /rviz2 use_sim_time
ros2 param get /robot_state_publisher use_sim_time
```

目标：

- `/clock` 只有一个主要 publisher
- 关键节点统一 `use_sim_time=true`

若出现：

```text
Detected jump back in time
Resetting RViz
Clearing TF buffer
```

先修时间链，不要先改模型。

### 第三步：TF 链

检查：

```bash
ros2 topic echo /tf --once
ros2 run tf2_tools view_frames
```

目标：

- TF 树完整
- 基座到末端参考链可追踪
- 不存在末端 frame 定义混乱

### 第四步：几何结构与坐标系

重点检查：

- 关节 frame 是否落在真实关节轴心
- link frame 是否被误当成 joint frame
- 末端工具参考点是否清晰
- 末端参考坐标系是否清晰
- visual / collision / inertial 是否一致

### 第五步：规划层

检查：

- 是否存在正式轨迹规划层
- 是否仍在错误依赖“目标点 -> IK -> 跟踪”
- 规划器是否使用了正确模型、正确限位、正确碰撞场景
- 是否存在预抓取 / 接近 / 接触分层逻辑

### 第六步：LQR 跟踪层

检查：

- 状态定义是否清晰
- 参考轨迹是否正确
- Q / R 是否导致 wrist 被压制
- LQR 是否被错误拿来顶替规划器

### 第七步：FOC 电机层

检查：

- 电机参数是否正确
- 线间值 / 单相值是否明确
- 力矩常数、反电动势常数是否一致
- 电流环是否过快或过慢
- 电压 / 电流限幅是否合理

### 第八步：执行层

检查：

- Gazebo 是否为唯一机械执行真源
- 是否仍出现 `goal_time_tolerance`
- 是否仍出现 `state tolerance`
- 长轨迹是否出现收敛速度过慢

---

## 8. FOC 自研实现的高频检查项

本项目中 FOC 核心必须自研。排查时优先看：

1. Clarke 变换
2. Park 变换
3. 逆 Park 变换
4. dq 电流环
5. 力矩到电流目标映射
6. 电压限幅
7. PWM / 平均电压模型输出
8. 线间值到单相值转换是否只做一次

若控制异常，不要第一时间怀疑参数，先核查实现链路和单位。

---

## 9. LQR 自研实现的高频检查项

本项目中 LQR 核心必须自研。排查时优先看：

1. 状态向量定义
2. 参考状态构造
3. 误差状态构造
4. Q / R 权重矩阵
5. 增益矩阵使用逻辑
6. 前馈与反馈合成方式
7. 是否被错误地拿来替代全局轨迹规划

---

## 10. 电机参数使用规则

本项目约定：

- `phase_resistance`
- `phase_inductance`

在配置中填写的是**线间值**。

因此必须遵守：

1. 方程若需要单相等效值，则在代码内部做一次显式转换。
2. 文档中必须写清楚：
   - 配置层是线间值
   - 实现层如何转换
3. 不允许在不同模块里各自重复转换。
4. 不允许有人为“这个模块按线间值、那个模块按单相值”但不文档化。

---

## 11. 机械惯量重复计入检查

若 URDF 已经手动调整过关节对应 link 的惯量（例如 `izz`），则：

1. 先确认 URDF 是否已经把“电机 + 关节臂整体惯量”并入机械侧。
2. 再确认电机物理仿真层是否仍在额外叠加同一部分机械惯量。
3. 若存在双重计入，必须优先修正。
4. 不要在未确认惯量归属前继续调 LQR / FOC。

---

## 12. Drake 与 Gazebo 协同使用的排查规则

若项目同时使用 Drake 与 Gazebo：

### 正确分工
- Drake：规划、轨迹优化、线性化、LQR 设计
- Gazebo：机械执行、碰撞、接触、状态反馈

### 必查项
1. 是否出现两个物理真源
2. 是否出现两套 joint states
3. 是否出现两套 `/clock`
4. 是否出现 TF 抖动
5. 是否出现规划结果与执行结果不一致

如有上述问题，优先收敛主从关系，而不是继续叠补丁。

---

## 13. 仿真提速的优先手段（不降难度）

如果需要提速，优先采用以下方法：

1. 多速率仿真
2. 平均电压模型优先
3. 适合刚性系统的积分方法
4. 缓存不随时间变化的量
5. 分离 visual mesh 与 collision mesh
6. 降低日志与可视化频率
7. Warm start 轨迹优化
8. 避免 Drake / Gazebo / 电机层重复解算同一物理量

**不要**用粗暴删物理、删碰撞、删约束来换速度。

---

## 14. 终端输出规范

控制相关终端输出优先保持**结构化、对齐、稳定格式**，并使用**简体中文**。

默认输出尽量包含：

- 当前规划状态
- 当前目标点位置
- 当前末端位置
- 关节目标角度
- 关键控制状态
- 解析后的 CAN 报文内容

建议格式：

```text
[规划状态]
状态      : 成功
原因      : 已生成轨迹

[目标信息]
目标点    : x=0.320 y=0.100 z=0.240
末端位置  : x=0.312 y=0.098 z=0.236

[目标关节角]
joint1    : +0.123 rad
joint2    : -0.456 rad
joint3    : +0.210 rad
joint4    : +1.047 rad
joint5    : -0.382 rad
joint6    : +0.000 rad

[CAN报文解析]
报文ID    : 0x201
方向      : 发送
字段1     : ...
字段2     : ...
```

---

## 15. 文档同步要求

每次对以下内容进行修改后，必须同步更新文档：

- 主 URDF 结构
- 末端参考点 / 末端参考坐标系
- 轨迹规划逻辑
- LQR 核心控制逻辑
- FOC 核心控制逻辑
- 电机参数
- 线间值/单相值转换规则
- 通信协议
- Gazebo / Drake 协同方式
- 启动方式
- 调试方法
- 已知限制

优先更新：

- `docs/architecture/system_overview.md`
- `docs/architecture/control_hierarchy.md`
- `docs/control/foc_design.md`
- `docs/control/lqr_design.md`
- `docs/control/motor_physics.md`
- `docs/planning/trajectory_planning_design.md`
- `docs/communication/can_protocol.md`
- `docs/simulation/gazebo_execution_design.md`
- `docs/development/build_and_run.md`

**禁止只改代码不改文档。**

---

## 16. 避免返工的执行策略

Codex 执行修复时必须遵循：

1. 先确认当前问题属于哪一层：
   - 环境
   - 时间链
   - 状态链
   - TF 链
   - 模型与坐标系
   - 规划层
   - LQR 层
   - FOC 层
   - 通信层
   - 执行层
2. 一次只修一层。
3. 每修完一层立即验证。
4. 不要在 TF 断链时调 LQR。
5. 不要在模型/坐标系错误未修复时判断规划器或控制器本身有问题。
6. 不要在惯量归属未确认时就调电机参数。
7. 不要在 FOC / LQR 核心尚未自研落地时就引入现成控制库顶替。

---

## 17. 常用最小验证命令

```bash
ros2 control list_controllers
ros2 topic echo /joint_states --once
ros2 topic echo /tf --once
ros2 topic info /clock
ros2 node info /robot_state_publisher
ros2 run tf2_tools view_frames
```

如果要验证控制器是否真的加载：

```bash
ros2 run controller_manager spawner joint_state_broadcaster -c /controller_manager
```

若项目已有具体轨迹控制器，再按实际控制器名称做 spawner。

---

## 18. 最终目标

Codex 在本项目中的目标不是“先拼出一套能跑的旧项目变体”，而是：

- 在 **WSL 的 `Ubuntu` 发行版** 中
- 基于**当前新项目主 URDF**
- 建立**可维护、可复现、可解释**
- 具备**轨迹规划 + 自研 LQR + 自研 FOC + CAN + Gazebo + RViz**
- 并能体现**真实电机电气方程与力矩方程**的完整工程系统
