---
name: rmvl-navigation
description: 维护和实现 RMVL 的无 ROS 导航能力。处理 RMVL 导航架构、导航消息、时间与坐标系语义、Odometry、Path、OccupancyGrid、TransformBuffer、代价地图、路径规划、控制、视觉 SLAM 接入以及 Nav 相关 LPSS/KDT 多进程测试时使用；也用于在新会话中恢复 RMVL Nav 的设计背景和当前进度。不要为普通 LPSS、CLI、LViz、相机或机器人控制任务触发，除非任务涉及导航数据流或导航接口。
---

# RMVL Navigation

## 目标与边界

将 RMVL 扩展为不依赖 ROS/ROS 2 运行时及其组件的轻量导航库。允许参考 ROS 消息的成熟语义，但自行实现消息生成、发布订阅、坐标变换、地图、规划和控制。

优先支持能运行 Linux 的 A 核、x86 和其他通用 Linux 平台。不要为了首版导航引入 Cortex-M、Keil、RTOS 或裸机适配，也不要让基础导航消息依赖 OpenCV、Eigen、PCL 或具体 SLAM 后端。

按以下数据流理解系统：

```text
相机 / IMU / 轮速计
        |
        v
视觉里程计 / SLAM -----> Odometry + Transform
        |                       |
        v                       v
二维占据栅格 ---------> 全局/局部代价地图
                                |
                                v
                         全局规划 -> 路径跟踪 -> 底盘命令
```

## 先检查代码真实状态

开始设计或编码前，检查当前分支、工作树和下列文件。不要只根据 todo 的勾选状态判断功能是否存在：

- `modules/lpss/msg/`
- `modules/lpss/CMakeLists.txt`
- `modules/nav/`
- `modules/nav/CMakeLists.txt`
- `cmake/RMVLCodeGenerate.cmake`
- `modules/lpss/test/test_serialization.cpp`
- `doc/tutorials/modules/tutorial_rmvlmsg.md`
- 仓库外规划文件 `../todo.md`（仓库通常位于 `~/.rmvldev/rmvl`）

截至创建本 skill 时，代码状态如下：

- 已有 Odometry、Path、OccupancyGrid、OccupancyGridUpdate 及其依赖的基础 geometry/std 消息，并覆盖二进制与 JSON 序列化测试。
- `lpss::tf::Buffer` 已提供坐标树约束、静态/动态历史、按时间查询和插值；Listener、Broadcaster 与 StaticBroadcaster 已接入隔离的 `<name>/tf`、`<name>/tf_static` 话题。
- 导航算法与数据模型位于和 `modules/lpss` 同级的 `modules/nav`，公开 API 直接使用 `rm::nav` 命名空间；导航消息的 IDL 与生成流程仍由 LPSS 提供。
- `rm::nav::GridMap` 已提供地图校验、世界/栅格转换、概率更新、边界裁剪射线和自动脏矩形增量；`rm::nav::Costmap` 已提供静态层、局部障碍层、膨胀和 footprint 碰撞检测。
- `rm::nav::AStarPlanner` 已提供四/八邻域搜索、代价加权、夹角穿越策略、路径简化和平滑；`rm::nav::PurePursuit` 已提供基础路径跟踪，`rm::nav::CollisionStop` 已提供独立的短时轨迹预测与碰撞停车过滤。
- `Duration.msg` 使用单个 `int64 nanoseconds`；在其他消息中用小写内置类型 `duration`，生成 C++ 类型 `rm::msg::Duration`。
- `Time.msg` 已使用 `int64 sec + uint32 nsec`，并已有可注入的 SystemClock/ManualClock 时间模型。

## 固定设计约定

### 时间

- 将消息采集时间、地图版本时间和变换查询时间视为系统时间或可回放的逻辑时间。
- 将超时、心跳、重试和控制周期视为单调时间，继续使用 `std::chrono::steady_clock`。
- 将 `Duration` 仅作为可序列化线协议类型；在 C++ 算法内部优先使用 `std::chrono::duration`。
- 保留 `lpss::now()` 作为兼容入口；后续为 `Node` 或时间上下文注入 `Clock`，默认使用系统时钟，并提供可 `set/advance` 的手动时钟。
- 补齐时间运算时采用 `Time - Time -> std::chrono::nanoseconds`、`Time +/- std::chrono::duration -> Time`，不要用 `Time` 表示时间间隔。

### 坐标系

使用以下导航坐标系分工：

- `map`：全局连续参考系；允许因回环或重定位修正 `map -> odom`。
- `odom`：局部连续参考系；短时平滑但会累积漂移。
- `base_link`：机器人本体坐标系。
- 相机、IMU、雷达等传感器坐标系：通过静态外参连接至 `base_link`。

保持坐标树单父节点且无环。`lpss::tf::Buffer` 支持静态/动态边、时间历史、平移线性插值和四元数 SLERP。`TransformStamped` 的父子边记法、存储的“子坐标系在父坐标系中的位姿”以及点变换 API 的输入输出方向必须保持无歧义。

### 协方差

统一使用 `float64[36]`，按 6x6 row-major 存储，变量顺序固定为：

```text
x, y, z, roll, pitch, yaw
```

不要针对二维导航缩短数组；二维实现可只填写 `x`、`y`、`yaw` 相关元素，其余元素按照文档约定表示未知或高不确定度。

### 占据栅格

- 使用 row-major 布局，索引为 `index = y * width + x`；`x` 先递增。
- 使用 `int8[] data`；`-1` 表示未知，`0` 表示空闲，`100` 表示占据，中间值表示概率或代价。
- `origin` 表示栅格 `(0, 0)` 在地图坐标系中的真实位姿；`resolution` 单位为米/格。
- 全局地图低频或按需发布；局部地图优先发布矩形增量，避免通过 LPSS 高频发送整张大地图。
- 为全量地图和增量更新保留单调递增 `revision`，用 `base_revision` 拒绝乱序、丢失前序或应用到错误基线的增量。

## 第一批：导航基础消息

先只实现消息、生成配置、文档和序列化测试，不在同一批改动中实现 TF 算法、地图算法或 SLAM。

建议采用下列 RMVL v1 定义；编码前检查生成器是否支持所用固定数组和嵌套数组语法。

### geometry/PoseStamped.msg

```text
Header header
geometry/Pose pose
```

`header.frame_id` 指定 `pose` 所在坐标系，`header.stamp` 指定观测或估计时刻。

### geometry/PoseWithCovariance.msg

```text
geometry/Pose pose
float64[36] covariance
```

### geometry/TwistWithCovariance.msg

```text
geometry/Twist twist
float64[36] covariance
```

### nav/Odometry.msg

```text
Header header
string child_frame_id
geometry/PoseWithCovariance pose
geometry/TwistWithCovariance twist
```

明确以下语义：

- `pose` 是 `child_frame_id` 在 `header.frame_id` 中的位姿。
- `twist` 是 `child_frame_id` 的速度；表达坐标系必须在消息注释和消费 API 中固定。默认采用 `child_frame_id` 表达，避免消费者猜测。
- 常见实例为 `header.frame_id = odom`、`child_frame_id = base_link`。

### nav/Path.msg

```text
Header header
geometry/PoseStamped[] poses
```

要求所有 pose 与路径的 `header.frame_id` 一致。路径头时间表示路径生成时刻；单点时间可用于轨迹时间参数化，没有时间参数化时允许保持零值。

### nav/MapMetaData.msg

```text
time map_load_time
float32 resolution
uint32 width
uint32 height
geometry/Pose origin
```

### nav/OccupancyGrid.msg

```text
Header header
nav/MapMetaData info
uint64 revision
int8[] data
```

要求 `data.size() == width * height`，并在反序列化后或地图对象构造时校验，防止溢出和畸形输入。

### nav/OccupancyGridUpdate.msg

```text
Header header
uint64 base_revision
uint64 revision
uint32 x
uint32 y
uint32 width
uint32 height
int8[] data
```

要求更新矩形完全落在目标地图内，`data.size() == width * height`，且 `base_revision` 等于本地当前版本；成功应用后本地版本变为 `revision`。失败时请求或等待下一张全量地图，不要猜测补丁顺序。

### 集成清单

实现消息批次时完成全部项目：

1. 在 `modules/lpss/msg/geometry` 与新建的 `modules/lpss/msg/nav` 中添加消息。
2. 在 `modules/lpss/CMakeLists.txt` 的消息分组中加入 `nav`。
3. 在 `tutorial_rmvlmsg.md` 中增加 `nav` 分组，并更新“主要分组”的数量和说明。
4. 为每种新消息增加二进制序列化/反序列化测试；为嵌套、固定数组、动态数组、负占据值和最大尺寸边界增加覆盖。
5. 增加 JSON 往返测试，检查 `int8` 的数值表现，不要让它被当作字符输出。
6. 按 `$rmvl-dev-tools` Skill 使用 `lpss interface` 检查生成后的接口，再构建 LPSS 并运行相关 C++ 测试；若完整测试因沙箱禁止创建网络 socket 而失败，单独报告网络测试限制，不要将其误归因于消息实现。
7. 保持消息改动不依赖 ROS，不复制 ROS 生成代码，只参考公开的数据语义。

## KDT 多进程通信测试

代码涉及进程间通信或需要同时运行多个 LPSS 节点时，优先参考并复用独立仓库 `~/.rmvldev/ci-workflow/kdt`。基础发布订阅、服务、节点重启、崩溃恢复和多发布者/多订阅者场景不要重新搭建临时测试框架。

KDT 的执行链固定为：

```text
plans/*.yml
  -> cases/<group>/*.yml
  -> components/keywords.yml
  -> components/**/*.py
  -> adapter/src/*.cpp
  -> RMVL LPSS 节点进程
```

先阅读最接近目标的现有用例：

- 基本发现顺序：`basic_pubsub_sub_first.yml`、`basic_pubsub_pub_first.yml`
- 拓扑组合：`one_pub_multi_sub.yml`、`multi_pub_one_sub.yml`、`multi_pub_multi_sub.yml`
- 生命周期：`node_restart.yml`、`node_crash_recovery.yml`
- 复合消息端到端传输：`test_types_pubsub.yml`
- 服务：`basic_service.yml`、`service_restart.yml`、`test_types_service.yml`

#### 直接复用现有关键字

仅需验证基础发布订阅发现、传输和多节点拓扑时，使用 `lpss_msg_strpub`、`lpss_msg_strsub`、`logic_wait`、`lpss_stop` 与 `fs_*` 断言组合。使用唯一的节点名、`/kdt/...` 主题名和窄范围 `tmp-lpss-*` 临时目录。

用例结构采用：

```yaml
info:
  name: 【发布订阅】场景名称
  input: 输入条件
  expected: 期望结果
steps:
  - name: fs_remove_recursive
    args:
      - - tmp-lpss-example
  - name: fs_create_dir
    dep: true
    args:
      - - tmp-lpss-example
  # 启动订阅者/发布者、等待发现、检查输出
  # 停止进程和删除临时目录时不要设置 dep，确保前序失败后仍能清理
```

使用 `dep: true` 表示当前步骤依赖紧邻前一步成功；任一 `KeywordSkipped` 还会让该用例后续所有依赖步骤跳过。KDT 会继续执行非依赖步骤，并在 plan 结束时聚合失败。用 `{{ stepN }}` 引用较早成功步骤的返回值；完整占位符保留原 Python 类型，嵌入字符串时只支持标量。

不要仅依靠后台 Adapter 的日志判定成功。后台进程意外退出只会被监控线程记入日志，必须通过输出文件、`fs_contains_lines`、`fs_unique_count`、`fs_same`、`fs_slice + assert_equal` 等关键字形成可失败的断言。

#### 增加 Nav 专用用例

需要验证 Odometry、Path、OccupancyGrid 等具体消息经过“生成、序列化、LPSS 传输、反序列化”后字段保持一致时，参考 `TestTypes` 链路增加强类型 Adapter，不要把 Nav 数据降级成字符串测试：

1. 在 `kdt/adapter/src` 增加发布端和订阅端 C++ 文件；`adapter/CMakeLists.txt` 会通过 `aux_source_directory` 自动创建同名可执行程序并输出到 `adapter/bin`。
2. 在 `components/func/lpss/msg.py` 增加参数校验和 `_start(...)` 包装。
3. 在 `components/func/lpss/base.py` 的 `LpssAdapter` 增加 Adapter 枚举。
4. 在 `components/keywords.yml` 注册公开关键字及全部参数。
5. 同步扩展 `lpss_stop` 的 Adapter 枚举数据。关键字框架按选项位置映射 Python Enum，因此 YAML 选项与 `LpssAdapter` 成员的数量和顺序必须完全一致。
6. 在 `cases/lpss` 增加端到端用例，并按门禁价值加入 `plans/ci_guardian.yml`；耗时或重复稳定性场景再加入 `stability_test.yml`。
7. 同时兼容 RMVL 当前支持的 C++17/C++20 节点路径；确实只支持特定环境时使用退出码 77 和 `KeywordSkipped`，不要伪装成通过。

Adapter 由框架统一跟踪。`lpss_stop` 对 SIGINT/SIGTERM 要求 1 秒内以退出码 0 优雅结束，对 SIGKILL 等异常信号校验真实信号退出；KDT 退出时还会通过 `atexit` 清理剩余后台进程。仍须为每个用例显式停止节点，使生命周期问题能够被测试发现。

当前用例主要用 0.2 秒等待发现、0.5 至 0.7 秒等待消息、1 秒等待崩溃节点心跳清理。新增用例先沿用邻近场景；若出现时序抖动，优先增加有界条件等待关键字，而不是无限加长固定 sleep。

#### 运行 KDT

从 KDT 根目录运行，入口会先重新配置并构建全部 Adapter，再执行一个或多个 plan：

```bash
cd ~/.rmvldev/ci-workflow/kdt
python3 test.py ci_guardian --rmvl-dir "$RMVL_ROOT_/build/config-install"
python3 test.py stability_test --rmvl-dir "$RMVL_ROOT_/build/config-install"
python3 test.py /absolute/path/to/focused-plan.yml --rmvl-dir "$RMVL_ROOT_/build/config-install"
```

`--rmvl-dir` 接受安装前缀或包含 `RMVLConfig.cmake` 的目录，并递归定位包配置。日志写入 `kdt/tmp/summary.log`，旧日志会按时间归档。退出码 0 表示所有执行的 plan 通过，1 表示至少一个 plan 失败，2 表示 plan 或 RMVL 路径解析失败；Adapter 构建失败会阻止 plan 执行。

KDT 的 LPSS 用例需要本机 socket、多播和多个进程；若沙箱禁止网络系统调用，应申请在沙箱外运行。不要把 `Operation not permitted`、socket 创建失败或发现超时直接归因于导航代码。

`ci-workflow` 与 RMVL 是两个独立 Git 仓库。修改 KDT 用例前分别检查两边的工作树；提交或推送时分别使用各自分支和提交，不要把两个仓库的改动混为一个提交。

## 后续实现顺序

完成基础消息后按以下顺序推进，避免一开始同时实现完整 SLAM 和导航栈：

1. [已完成] 实现 `lpss::tf::Buffer`，覆盖树约束、静态变换、最新查询和时间插值。
2. [已完成] 在 `modules/nav` 实现 `rm::nav::GridMap`，覆盖 OccupancyGrid 内存模型、世界/栅格坐标转换、矩形增量合并和边界检查。
3. [已完成] 在 `modules/nav` 实现 `rm::nav::Costmap`，覆盖静态层、局部障碍层、机器人 footprint 碰撞检测和障碍膨胀。
4. [已完成] 使用人工地图和离线测试实现 A*，并增加路径简化和平滑。
5. [已完成] 实现 Pure Pursuit 基础路径跟踪，并加入独立的短时预测碰撞停车策略。
6. 接入 RGB-D 或双目视觉里程计，随后融合轮速计和可选 IMU。
7. 将深度图或点云按高度投影为二维占据栅格；区分用于定位的稀疏特征地图与用于导航的障碍地图。
8. 最后再增加关键帧、局部 BA、回环检测、位姿图优化、地图保存和重定位。

## SLAM 后端边界

定义可替换的 Odometry、SLAM 和 Mapper 后端接口，不让 RMVL 核心依赖某个第三方实现。优先使用公开数据集离线验证时间同步、尺度、轨迹误差和地图投影，再接真实硬件。

单目视觉 SLAM 可以建稀疏特征地图，但通常缺乏稳定绝对尺度，也不能直接提供可靠的二维障碍栅格。若目标是尽快完成简单导航，优先 RGB-D 或双目；融合轮速计和 IMU 以增强低纹理、运动模糊和短时遮挡下的稳定性。

## 工作约定

- 默认使用简体中文沟通。
- 面向熟悉现代 C++ 的开发者，直接说明数据语义、接口约束和权衡。
- C++ I/O 优先使用 RMVL 内置 fmt，其次使用 cstdio；不要使用 iostream。
- 修改代码时保留用户已有改动，先看 `git status` 和当前 diff。
- 用户只要求讨论、预研或评审时保持只读；只有明确要求实现时才改代码。
- 涉及新技术、第三方 SLAM 状态或当前 API 时，查阅其官方文档或原始论文，不依赖记忆给出易过时结论。
- 通用 rdt/lpss 命令与 Git 流程使用 `$rmvl-dev-tools` Skill；LViz 前后端和正式包同步使用 `$rmvl-lviz` Skill，Navigation Skill 只保留导航语义与 Nav 专用测试约定。
