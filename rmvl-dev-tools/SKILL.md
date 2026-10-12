---
name: rmvl-dev-tools
description: 使用和维护 RMVL Dev Tools 的 rdt、lpss、lgraph 与 lviz 命令行工具。处理 LPSS 项目脚手架、接口与运行时诊断、Dev Tools 更新、CLI 实现排查或相关文档时使用。Git 分支、提交、推送和 GitHub Pull Request 使用 rmvl-git；导航设计与 LViz 前后端实现分别使用 rmvl-navigation 与 rmvl-lviz。
---

# RMVL Dev Tools

## 目标与信息源

以已安装命令和 `~/.rmvldev/rmvl-dev-tools` 中的脚本为准，不凭旧文档猜测参数。开始工作时按需检查：

```bash
command -v rdt
command -v lpss
rdt version
rdt help
lpss help
```

命令能力会随 Dev Tools 更新。需要确认行为时阅读：

- `scripts/bash/rdt.bash` 与 `scripts/bash/.rdt/*.bash`
- `scripts/bash/lpss.bash` 与 `scripts/bash/.lpss/*.bash`
- `setup/bash/install.bash`
- `src/CMakeLists.txt`

不要把 `help` 参数当作所有子命令的通用约定。尤其 `lpss create help` 会把 `help` 当作项目名并创建目录；查看它的参数应阅读 `scripts/bash/.lpss/create.bash`。

## 创建 LPSS 项目

当前脚手架用法为：

```bash
lpss create <project_name> [--deps <list>] [--exts <list>] [--cpp <17|20|23>]
```

- 在目标项目的父目录执行，`project_name` 只能是单层目录名，不能包含 `/` 或空格。
- `--deps` 传 RMVL 模块依赖，例如导航程序使用 `--deps nav`；生成的 CMake 会自动包含 `lpss`。
- `--exts` 传外部依赖列表。
- 默认 C++20；需要兼容其他标准时显式传 `--cpp`。
- 创建前检查同名目录，避免覆盖已有项目。创建后检查生成的 `CMakeLists.txt`、`src/main.cpp`、`README.md` 与 `.gitignore`。

示例：

```bash
cd /path/to/examples
lpss create map_demo --deps nav --cpp 20
cmake -S map_demo -B map_demo/build
cmake --build map_demo/build
```

## LPSS 接口与运行时诊断

检查消息定义：

```bash
lpss interface groups
lpss interface group nav
lpss interface list
lpss interface show nav/OccupancyGrid
```

检查运行中的节点、话题与服务：

```bash
lpss node list
lpss node info <node_name>
lpss topic list
lpss topic info <topic_name>
lpss topic type <topic_name>
lpss topic echo <topic_name>
lpss topic hz <topic_name>
lpss topic bw <topic_name>
lpss service list
lpss service info <service_name>
lpss service type <service_name>
lpss service find <service_type>
lpss service call <service_name> '<json_request>'
```

`lpss graph [instance_name]` 与 `lpss viz [instance_name]` 会启动已安装的原生工具；未安装 `_autogen_lpss_graph` 或 `_autogen_lpss_viz` 时，先重新构建并安装 Dev Tools。实例名省略时脚本会生成短随机名。

截至 2026-08-16，`lpss topic pub` 的 Bash 函数仍为空，不要把它作为可靠验证步骤。使用前再次检查当前脚本是否已经实现。

CLI 适合接口检查、发现、联调和冒烟测试；C++ 单元测试、序列化测试与回归测试仍使用项目的 CMake/CTest。交付时区分 CLI 联调和单元测试结果。

## 修改 Dev Tools 本身

修改脚本或内置程序前先检查 `~/.rmvldev/rmvl-dev-tools` 的分支、暂存区和未暂存改动，保留用户已有工作。Bash 子命令的帮助文本、参数校验和实际分派必须同步修改。

`setup/bash/install.bash` 会构建 `src` 下的原生工具，并把产物安装为 `.lpss/_autogen_*`。仅修改源码不会自动更新当前 shell 正在调用的已安装二进制；验证时明确区分源码构建产物和已安装产物。
