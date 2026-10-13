# para、msg、srv 开发指引

## 先检查编辑器插件与生成器

检查 **RMVL CMake and IDL**，扩展 ID 为 `zhaoxi-scut.rmvl-vscode-extension`，官方仓库：[cv-rmvl/rmvl-vscode-extension](https://github.com/cv-rmvl/rmvl-vscode-extension)，[扩展商店](https://marketplace.visualstudio.com/items?itemName=zhaoxi-scut.rmvl-vscode-extension)。

```bash
code --list-extensions
```

若 CLI 不可用或结果有错误，检查 `~/.vscode/extensions/*/package.json` 或远程端 `~/.vscode-server/extensions/*/package.json` 的 name、publisher、displayName。目录中存在只说明有安装文件，不能据此保证当前 Profile 已启用，CLI 列表也只反映其所在环境与 Profile。两者不一致时说明证据范围，建议在实际编辑 IDL 的 VS Code 环境中查看扩展面板。

未安装时建议在扩展面板搜索 `RMVL CMake and IDL`，或让用户执行：

```bash
code --install-extension zhaoxi-scut.rmvl-vscode-extension
```

插件提供 IDL 语法、提示和 CMake 编辑支持，不替代生成器。外部工程 `find_package(RMVL REQUIRED ...)` 会加载安装包中的 `RMVLCodeGenerate.cmake` 及模板，先确认所需函数存在。缺函数或模板时核实安装版本和完整性，不直接混用另一版本源码的生成器。

## 定义与目录规则

路径以**调用生成函数的 CMakeLists.txt 所在目录**为基准。函数在 CMake 配置阶段生成文件，当前实现写入该目录下的 `include/` 和 `src/`，并非自动写到 build/。解释该行为，避免用户手改生成文件，版本控制规则由项目决定。

```text
demo/
├── CMakeLists.txt
├── src/main.cpp
└── interfaces/
    ├── CMakeLists.txt
    ├── param/settings.para
    ├── msg/Status.msg
    └── srv/SetEnabled.srv
```

### para：C++ 类型、默认值与运行时参数

`interfaces/param/settings.para`：

```text
int retries = 3 # 重试次数
double gain = 1.5 # 增益
string device # 运行时设置设备名
```

使用 C++ 风格类型，例如 `int`、`uint32_t`、`double`、`string`，不是 msg 的 `int32`、`float64`。`=` 设置默认值，尾置 `#` 写注释，支持 `enum ... endenum` 的有作用域枚举。OpenCV 点、向量、矩阵及复合类型按教程和构建条件选用。

`rmvl_generate_para(settings)` 生成 `rm::para::SettingsParam` 和 `rm::para::settings_param`，通过 `.read("config.yml")` / `.write(...)` 加载或保存 YAML，检查 bool 返回值。未出现在 YAML 中的字段保留原值。一般使用生成的参数对象，不另外手写加载器。

### msg：消息字段与序列化

`interfaces/msg/Status.msg`：

```text
int32 id
string label
bool active
```

生成 `rm::msg::Status`。基本类型包括 `bool`、`int8/uint8` 到 `int64/uint64`、`float32/float64`、`string`、`char`，动态数组写 `float64[] values`，定长数组写 `float64[3] position`。标准嵌套类型如 `Header header` 不必加 `std/`，其他组写 `geometry/Pose pose`、`sensor/Imu imu` 等，按本地内置 msg 和生成器核实。

`TYPE NAME = VALUE` 用于常量定义，不把 msg 的 `=` 当作 para 的普通字段默认值。引用自定义消息时先生成依赖的消息，确保对应头文件能找到。消息提供 serialize、deserialize、compact_size，可直接用于 LPSS 的模板端点，不需手写序列化。

### srv：恰好一个请求/响应分隔符

`interfaces/srv/SetEnabled.srv`：

```text
bool enabled
---
bool success
string message
```

字段与常量规则同 msg，独占一行的 `---` 必须恰好一个，上面是请求，下面是响应，两侧可为空。生成 `rm::srv::SetEnabled::Request` 与 `Response`，均具备消息序列化接口。服务调用见 LPSS 教程的 async Service/Client，不假定存在同步服务接口。

## 从定义到可链接目标

下面示例将三类生成文件编入同一个库，展示普通 CMake 接法。顶层 `demo/CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)
project(demo LANGUAGES CXX)
find_package(RMVL REQUIRED COMPONENTS core lpss)
add_subdirectory(interfaces)
add_executable(demo src/main.cpp)
target_link_libraries(demo PRIVATE demo_interfaces)
```

`interfaces/CMakeLists.txt`，先调用生成函数，再创建目标：

```cmake
rmvl_generate_para(settings)
rmvl_generate_msg(Status)
rmvl_generate_srv(SetEnabled)

add_library(demo_interfaces STATIC
    src/_rm_codegen_param.cpp
    src/_rm_codegen_msg_status.cpp
    src/_rm_codegen_srv_set_enabled.cpp
)
target_include_directories(demo_interfaces PUBLIC
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
    ${RMVL_INCLUDE_DIRS}
)
target_link_libraries(demo_interfaces PUBLIC ${RMVL_LIBS})
```

这是当前生成器输出路径的示例，写目标前检查本地生成结果。旧文档提到的 `src/para/<name>.cpp` 不一定适用，多个独立 para 调用还可能写入同一个 `_rm_codegen_param.cpp`，不要盲目复制或覆盖。

生成后，项目 `src/main.cpp` 可包含：

```cpp
#include <fmt/format.h>
#include <rmvlpara/settings.hpp>
#include <rmvlmsg/status.hpp>
#include <rmvlsrv/set_enabled.hpp>

int main() {
    rm::msg::Status status{};
    status.id = rm::para::settings_param.retries;
    status.label = "ready";
    status.active = true;
    rm::srv::SetEnabled::Request request{};
    request.enabled = true;
    fmt::println("{}: {}, enabled={}", status.label, status.id, request.enabled);
}
```

从项目根目录 `cmake -S . -B build`、`cmake --build build`，再运行 `./build/demo`。只有头文件而漏编生成的 cpp 会导致序列化或参数 read/write 的链接错误，必须把源文件加入目标。

官方 LPSS 自定义消息教程采用 RMVL 封装：接口子目录先 `rmvl_generate_msg(MyCustomMsg)` 再 `rmvl_add_module(custom_msg)`，顶层通过 `rmvl_add_exe(pub SOURCES src/pub.cpp DEPENDS lpss custom_msg)` 接入。该模块目标通常为 `rmvl_custom_msg`，DEPENDS 填短名 `custom_msg`。项目已使用这一风格时沿用，不必换成普通 CMake。

## MODULE、分组与多个参数文件

| 调用 | 当前实现的头文件 | 当前实现的源文件 |
| --- | --- | --- |
| `rmvl_generate_para(settings)` | `include/rmvlpara/settings.hpp` | `src/_rm_codegen_param.cpp` |
| `rmvl_generate_para(settings MODULE app)` | `include/rmvlpara/app/settings.h` | `src/settings/_rm_codegen_param.cpp` |
| `rmvl_generate_msg(Status)` | `include/rmvlmsg/status.hpp` | `src/_rm_codegen_msg_status.cpp` |
| `rmvl_generate_msg(Status MODULE interfaces)` | 同上 | `src/status/_rm_codegen_msg_status.cpp` |
| `rmvl_generate_srv(SetEnabled)` | `include/rmvlsrv/set_enabled.hpp` | `src/_rm_codegen_srv_set_enabled.cpp` |
| `rmvl_generate_srv(SetEnabled MODULE interfaces)` | 同上 | `src/set_enabled/_rm_codegen_srv_set_enabled.cpp` |

para 的 MODULE 控制头文件归组，多个参数文件可以分别 `rmvl_generate_para(a MODULE app)`、`rmvl_generate_para(b MODULE app)`，最后 `rmvl_generate_module_para(app)` 生成汇总头。当前汇总头用 `HAVE_RMVL_A/B` 宏控制包含，使用普通 CMake 时核实宏，或直接包含子头文件。生成源文件分别加入目标，避免多个无 MODULE 的 para 覆盖同一路径。

msg/srv 的 MODULE 不代表 `rm::msg::interfaces` 或 `rm::srv::interfaces` 命名空间，实际类型仍在 `rm::msg` / `rm::srv` 中。也不自动生成 CMake library 目标。`rmvl_generate_msg(geometry/Pose)` 读取 `msg/geometry/Pose.msg`，头文件保留路径并将名称转为 snake_case，srv 同理。

教程入口与右上角搜索词：

- `doc/tutorials/modules/tutorial_rmvlpara.md`，搜 `参数模块使用教程`、`rmvl_generate_para`、`rmvl_generate_module_para`。
- `doc/tutorials/modules/tutorial_rmvlmsg.md`，搜 `消息模块使用教程`、`rmvl_generate_msg`。
- `doc/tutorials/modules/tutorial_rmvlsrv.md`，搜 `服务模块使用教程`、`rmvl_generate_srv`。
- `doc/tutorials/modules/tools/lpss.md` 的自定义消息和 Service/Client 示例，搜 `LPSS`、`createService`、`createClient`。
