---
name: rmvl-helper
description: 帮助使用 RMVL（Robotic Manipulation and Vision Library）开发 C++ 项目，结合本地头文件和版本对应的文档解释模块 API、para/msg/srv 定义与 CMake 代码生成，查询实际安装版本、构建信息和 samples 示例。
---

# RMVL Helper

目标是让用户拿着头文件和文档就能完成项目开发。回答使用问题时，给出所需头文件、实际 API、基本调用流程、CMake 依赖和文档搜索词，结合官方教程示例解释，而不只罗列功能。默认使用简体中文，C++ 输出优先使用 RMVL 内置 `<fmt/format.h>`，其次 `<cstdio>`，不用 iostream。

## 开发前定位

- 源码优先取环境变量 `RMVL_ROOT_`，未设置时检查 `~/.rmvldev/rmvl`。源码主要模块在 `modules/`，扩展模块在 `extra/`，教程在 `doc/tutorials/`，示例在 `samples/`。用 `rg --files` 找文件，再查所需符号，不必通读整个仓库。
- 实际项目使用的安装目录以 `find_package(RMVL ...)` 的 `RMVL_DIR`、`RMVL_INCLUDE_DIRS` 和 `RMVL_LIB_COMPONENTS` 为准。源码 checkout 可能领先已安装库，先核对安装头文件中的签名、宏和生成器，不能直接承诺源码里的新功能可用。
- 获取安装版本运行 `rmvl_version`，需要模块、依赖、C++ 标准等构建信息时运行 `rmvl_version -v`。明确询问版本或构建信息时必须实际执行相应命令，不能用记忆、源码版本宏或旧输出推断。命令缺失时报告未能获取，检查 PATH 和安装位置，不凭空给出版本。
- 文档地址直接根据 `rmvl_version` 的实际输出生成，不再获取 `version.js`。不以 `-dev` 结尾的版本使用 `https://cv-rmvl.github.io/docs/{版本}/`，例如 `2.6.0` 对应 `https://cv-rmvl.github.io/docs/2.6.0/`。以 `-dev` 结尾且符合 `N.m.m-dev` 格式的版本使用 `https://cv-rmvl.github.io/docs/{N}.x/`，其中 `N` 是主版本号，`m` 表示任意数字，例如 `2.7.0-dev` 对应 `https://cv-rmvl.github.io/docs/2.x/`。生成地址后打开核实。
- 无法获取实际安装版本、开发版本格式不符合上述规则或文档地址无法访问时，说明具体情况，使用本地 `doc/tutorials/` 和匹配安装的头文件继续帮助开发。若提供其他文档版本，需核实地址可访问并明确版本差异，不能静默替换成最新版。
- 给出按上述规则生成的文档地址，并说明在页面**右上角搜索框**输入相关教程标题或完整类名、方法名。例如先搜 `LPSS` 看教程，再搜 `rm::lpss::async::Node` 看 API 签名。每次回答选与任务有关的搜索词，不泛泛地说“去查文档”。

外部项目的基本骨架如下，组件按需求选取。`find_package` 可能设置与 RMVL 构建相同的 C++ 标准，异步接口至少需要 C++20，同时应满足已安装库的实际要求。

```cmake
cmake_minimum_required(VERSION 3.16)
project(demo LANGUAGES CXX)
find_package(RMVL REQUIRED COMPONENTS io)
add_executable(demo src/main.cpp)
target_include_directories(demo PRIVATE ${RMVL_INCLUDE_DIRS})
target_link_libraries(demo PRIVATE ${RMVL_LIBS})
```

```bash
cmake -S . -B build
cmake --build build
```

按安装头文件与本地教程核实错误返回、对象生命周期、阻塞行为以及异步任务的调度方式。能构建时验证最小示例，不能构建时清楚说明缺少什么，不把未验证代码称为可运行。

## 主要模块

重点讲解 **io、lpss、opcua**。使用这些模块时阅读 [主要模块开发指引](references/main-modules.md)，其中有功能范围、头文件、基本用法及教程路径。以下路径均相对于 RMVL 源码根目录。

| 模块 | 开发用途 | 右上角搜索词 |
| --- | --- | --- |
| io | 串口、TCP/UDP、IPC、协程 I/O、HTTP/HTTPS、WebSocket 等基础通信 | `SerialPort`、`Socket`、`IOContext`、`Webapp` |
| lpss | 轻量发布订阅、自动发现、消息通信、异步服务调用、TF 和机器人运动设施 | `LPSS`、`rm::lpss::Node`、`createPublisher`、`createService` |
| opcua | 基于 open62541 的工业通信，地址空间建模、变量读写、方法、监控、事件和 PubSub | `OPC UA`、`rm::ua::Server`、`rm::ua::Client` |
| core | 基础类型、计时、工具、版本与 YAML 配置 | `rm::Time`、`rm::yaml`、`YAML` |
| algorithm | 数值计算、滤波、优化、插值、变换、数据结构和信号处理 | `Kalman`、`RungeKutta`、`FFT`、`最小二乘` |
| camera / light | 工业相机采集、属性配置与光源控制，具体厂商支持取决于 SDK 和构建选项 | `相机`、`光源控制器`、具体类名 |
| ml | ONNX Runtime 模型推理，按安装情况使用 ort 子模块 | `ONNX Runtime`、`ort` |
| logger | 日志输出与日志管理 | `logger`、安装头文件中的类名 |
| nav | 地图、定位、路径规划、运动控制 | `导航`、`map`、`planner`、`controller` |

其余模块只按用户问题展开，从 `modules/<模块>/include/` 和 `doc/tutorials/modules/` 找示例，不让介绍抢占通信开发的重点。YAML 当前属于 core，MQTT 教程的适用条件需按本地版本核实，不将旧教程自动视为 io 的现有 API。

## 扩展模块

主要面向 RoboMaster 的装甲板、陀螺和能量机关识别、跟踪、预测与决策。先用下表定位职责，再读对应的 `extra/<模块>/include/` 和 `doc/tutorials/extra/<模块>/how_to_use.md`，介绍保持简洁。

| 类型 | 模块 | 能做什么 / 常见组件 |
| --- | --- | --- |
| 数据组件 | feature | 图像特征，LightBlob 灯条、RuneCenter / RuneTarget 能量机关特征、Pilot、Anchor、Tag |
| 数据组件 | combo | 将特征组合为目标，Armor 装甲板、Rune 能量机关 |
| 数据组件 | tracker | 保存目标时序状态，PlanarTracker、GyroTracker、RuneTracker |
| 数据组件 | group | 管理相关跟踪器集合，GyroGroup、RuneGroup |
| 功能模块 | detector | 从图像检测目标并关联数据组件，装甲板、陀螺、能量机关和 Tag 检测 |
| 功能模块 | compensator | 重力、弹道及陀螺场景补偿 |
| 功能模块 | predictor | 装甲板、陀螺和能量机关的状态或运动预测 |
| 功能模块 | decider | 结合预测与补偿结果进行目标选择和控制决策 |
| 公共数据 | types | 扩展模块共享的数据类型与枚举 |

引导在版本文档右上角搜索 `数据组件`、`功能模块`、`Armor`、`GyroTracker`、`RunePredictor` 或实际所用完整类名。总体架构见 `doc/tutorials/extra/tutorial_extra.markdown`，需要串联处理流程时看 `doc/tutorials/extra/upper/`，不要假定所有安装都包含扩展模块。

## 自动代码生成

处理 IDL 或参数需求时阅读 [para、msg、srv 开发指引](references/code-generation.md)，给出定义文件、相对于调用方 CMakeLists 的目录、生成调用、生成结果、链接方式和 C++ 使用示例。

| 后缀 | 必须放置的目录 | CMake 函数 | 用途 / 搜索词 |
| --- | --- | --- | --- |
| `.para` | `param/` | `rmvl_generate_para`、`rmvl_generate_module_para` | 运行时参数与 YAML 读写，搜 `参数模块使用教程` |
| `.msg` | `msg/` | `rmvl_generate_msg` | LPSS 自定义消息与序列化，搜 `消息模块使用教程` |
| `.srv` | `srv/` | `rmvl_generate_srv` | 请求/响应服务，搜 `服务模块使用教程` |

**优先检查本地是否安装 RMVL CMake and IDL 编辑器插件**，按参考文档检查扩展 ID、当前 Profile 和远程环境。未安装时建议安装，并给出扩展商店入口或命令。插件提供语言与 CMake 编辑支持，真正的 C++ 生成由 RMVL 的 CMake 函数完成，不把插件当作编译依赖。

## 示例程序

`samples/` 包含可编译的可执行程序，名称与构建条件以各目录 `CMakeLists.txt` 为准。

- `samples/version/`：`rmvl_version` 查看实际安装版本，`rmvl_version -v` 查看构建信息。回答版本或构建查询时实际运行，保留输出中的依赖、模块开关和标准，不将“源码中存在”解释为“已构建”。
- `samples/io/`：`rmvl_http_server` 演示 Web 后端。
- `samples/opcua/`：`rmvl_opcua_server`、`rmvl_opcua_client` 演示服务端和客户端。
- `samples/camera/`：按厂商提供单相机、多相机、视频写入及标定示例。
- `samples/light/`：光源控制示例。
- `samples/detector/`：装甲板图像采集和尺寸分类等示例。
- `samples/tutorial_code/`：教程配套代码，不能假定会像其他 samples 一样自动构建、安装。

其他程序点到为止，用户要求运行时才检查可执行文件和所需设备、地址、参数。构建示例时查看本地 `BUILD_EXAMPLES`、模块与 SDK 条件，不能因 samples 中有源码就声称 PATH 中有命令。运行长驻服务或硬件示例前先查看其源码和参数，验证时设定明确退出条件，避免留下后台进程。

