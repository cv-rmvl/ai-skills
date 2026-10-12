---
name: rmvl-lviz
description: 开发和维护 RMVL 的 LViz 可视化前后端。处理 Display、图标与属性面板、LPSS 话题接入、TF/RobotModel/Map 显示、前端构建、后端路由以及从本地调试仓库同步到 RMVL Dev Tools 正式包时使用。普通 rdt/lpss CLI 操作使用 rmvl-dev-tools。
---

# RMVL LViz

## 两套目录的职责

不要混淆本地开发工程和正式发布工程：

| 用途 | 路径 | 内容 |
| --- | --- | --- |
| 本地开发后端 | `/home/zhaoxi/桌面/ws/lviz/lviz_backend` | 独立 CMake 工程，便于调试后端 |
| 本地前端源码 | `/home/zhaoxi/桌面/ws/lviz/lviz_backend/deps/lviz_frontend` | Vue 前端源码与构建配置 |
| 正式 LViz 包 | `~/.rmvldev/rmvl-dev-tools/src/viz` | Dev Tools 后端源码与构建后的 `dist` |

本地路径只用于开发调试。最终用户通过 RMVL Dev Tools 获得 LViz，因此正式交付必须把前端构建产物同步到 `src/viz/dist`，并把后端改动按正式工程的结构适配到 `src/viz`。

开始修改前分别检查 LViz 开发仓库、前端子仓库或子模块、rmvl-dev-tools 的分支和工作树。三处可能同时存在用户改动，不要用整目录覆盖掩盖差异。

## 开发流程

1. 在本地前端源码中实现 Vue 组件、Display 状态、属性编辑、图标和 API 适配。
2. 在本地后端独立工程中实现 LPSS 订阅、数据缓存、HTTP/WebSocket 路由和资源清理。
3. 运行前端构建与后端编译，使用本地工程完成联调。
4. 对比正式后端的当前实现，逐项迁移代码，不要直接复制本地 `CMakeLists.txt`、路径配置或整个源目录。
5. 重新构建前端，将生成的 `dist` 同步至 `~/.rmvldev/rmvl-dev-tools/src/viz/dist`。
6. 从 rmvl-dev-tools 的 `src/CMakeLists.txt` 构建正式 `lpss_viz`，最后通过 `lpss viz` 冒烟验证。

使用 `lpss` 命令时遵循 `$rmvl-dev-tools` Skill；涉及导航消息与语义时同时遵循 `$rmvl-navigation` Skill。

## 后端适配差异

本地后端和正式后端的写法不同，迁移时保持以下约定：

| 项目 | 本地调试工程 | RMVL Dev Tools 正式工程 |
| --- | --- | --- |
| CMake 入口 | 独立 `project(LpssVisualizationBackend ...)` | `src/CMakeLists.txt` 中的 `RMVLDevTools` 工程 |
| 目标 | 本地独立目标 | `rmvl_add_exe(lpss_viz ...)` |
| RMVL 依赖 | 按本地工程声明 | 在 `DEPENDS` 中列出真实模块，例如地图支持需要 `lpss nav` |
| 路径头文件 | `#include "path_config.hpp"` | `#include "viz/path_config.hpp"` |
| 静态资源路径 | `path::deps + "lviz_frontend/dist/"` | `path::frontend` |
| 路径配置来源 | `@PROJECT_SOURCE_DIR@/deps/` | `@CMAKE_CURRENT_LIST_DIR@/viz/dist/` |
| 版本宏 | 本地工程版本 | `target_compile_definitions(lpss_viz PRIVATE LVIZ_VERSION="...")` |

正式 `backend_node.cpp` 应使用：

```cpp
#include "viz/path_config.hpp"

app.use(statics("/", path::frontend));
```

不要把本地的 `path::deps` 写法带入正式包，也不要在正式包中依赖桌面开发路径。

## 新增 Display 的检查清单

一个 Display 通常同时涉及：

- 前端类型定义、默认属性、组件注册、图标、创建菜单与属性面板。
- 订阅生命周期，包括话题变更、启用/禁用、组件卸载和连接重建。
- 后端订阅对象、缓存或序列化、路由注册、错误响应和删除 Display 时的资源释放。
- 正式 CMake 的 RMVL 模块依赖以及 `backend_node.hpp/.cpp`、`backend_http.cpp` 的一致更新。
- 空数据、非法消息、话题消失、后启动前端和重复创建/删除 Display 的行为。

Map Display 还要核对 `OccupancyGrid` 的 row-major、`-1/0/100` 语义、origin、resolution、revision 与增量补丁顺序。TF、RobotModel 和 Map 使用同一 Fixed Frame 时，应统一坐标变换约定，不在前端各自实现互相矛盾的方向。

## 构建与同步

前端源码目录执行：

```bash
npm run build
```

先检查构建输出目录和正式 `dist` 的现有差异，再同步完整产物，包括入口 HTML、带哈希资源和已删除的旧哈希文件。删除旧产物属于覆盖发布内容，应确保目标严格限定在 `rmvl-dev-tools/src/viz/dist`。

正式后端由 `~/.rmvldev/rmvl-dev-tools/src/CMakeLists.txt` 统一配置。不要把本地后端 CMake 覆盖过去。构建成功后仍需检查网页资源可加载、LPSS 订阅能退出、Display 删除不会残留订阅，以及 `lpss viz` 启动的是新安装的 `_autogen_lpss_viz` 而不是旧二进制。

提交时按仓库边界拆分：前端源码、本地后端和 rmvl-dev-tools 正式包可能属于不同 Git 仓库。每个仓库分别检查 diff 和提交，不要将构建产物来源说明遗漏在正式包提交中。
