# 主要模块开发指引

路径相对于 RMVL 源码根目录。下面是阅读官方示例后的开发起点，API 以当前项目实际使用的安装头文件为准。回答时选取相关片段，并补上返回值处理，不必把整个文件发给用户。

## io：按通信层选择 API

| 功能 | 头文件 / API | 本地教程 | 右上角搜索词 |
| --- | --- | --- | --- |
| 串口 | `rmvl/io/serial.hpp`，`rm::SerialPort`、`rm::async::SerialPort` | `doc/tutorials/modules/tools/serial.md` | `串口`、`SerialPort`、`SerialReadMode` |
| TCP / UDP | `rmvl/io/socket.hpp`，`Acceptor` / `Connector`、`StreamSocket`、`Sender` / `Listener`、`DgramSocket`，及 async 版本 | `doc/tutorials/modules/tools/socket.md` | `Socket`、`Endpoint`、`Acceptor`、`Connector` |
| IPC | `rmvl/io/ipc.hpp`，`PipeServer` / `PipeClient`、`MqServer` / `MqClient`、`SHMBase`、`RingBufferSlotSHM` | `doc/tutorials/modules/tools/ipc.md` | `IPC`、`PipeServer`、`SHMBase` |
| 协程调度 | `rmvl/io/async.hpp`，`rm::async::Task`、`IOContext`、`co_spawn` | `doc/tutorials/modules/tools/coro.md` | `协程`、`IOContext`、`co_spawn` |
| Web | `rmvl/io/netapp.hpp`，`Request` / `Response`、`requests`、`async::Webapp`、`HttpServer` / `HttpsServer`、`WebSocket` | `doc/tutorials/modules/tools/netapp.md` | `网络应用层`、`Webapp`、`HttpServer`、`WebSocket` |
| TLS | `rmvl/io/ssl.hpp`，`SSLContext` 与 SSL Socket | `netapp.md` 的 HTTPS 示例及该头文件 | `SSLContext`、`HttpsServer` |

### 串口：基础用法与协议边界

根据 `serial.md`，同步接口可处理字符串或支持的聚合体，选设备、波特率与阻塞模式后读写：

```cpp
#include <fmt/format.h>
#include <rmvl/io/serial.hpp>

int main() {
    rm::SerialPort port("/dev/ttyUSB0", rm::BaudRate::BR_115200,
                        rm::SerialReadMode::NONBLOCK);
    std::string received;
    if (port.read(received))
        fmt::println("{}", received);
    return port.write("ping") ? 0 : 1;
}
```

这是需要串口设备的调用示例。开发二进制协议时先阅读 `read(head, tail, data)` 重载及聚合体限制，约定字段、长度、端序和帧头帧尾。教程中的同步 `write` 不自动加帧头帧尾，不假定结构体原始内存就是线路编码。非阻塞调用需要业务安排重试或调度，不能只调用一次便保证收到数据。

异步串口配合 `IOContext`，用 `co_await port.read()` / `write()`，本地教程当前只展示字符串接口。先创建上下文，再创建 I/O 对象，注册任务，最后 `run()`，对象和上下文要覆盖任务执行期间。

### TCP / UDP：连接与异步任务

`socket.md` 的 TCP 最小流程是服务端 `Acceptor(Endpoint(ip::tcp::v4(), port)) → accept() → read()/write()`，客户端 `Connector(Endpoint(...), address) → connect() → write()/read()`。例如：

```cpp
#include <fmt/format.h>
#include <rmvl/io/socket.hpp>

int main() {
    rm::Connector connector(rm::Endpoint(rm::ip::tcp::v4(), 8080), "127.0.0.1");
    auto socket = connector.connect();
    if (!socket.write("request"))
        return 1;
    auto response = socket.read();
    if (response.empty())
        return 1;
    fmt::println("{}", response);
}
```

先运行教程中的服务端，再运行客户端。TCP 是字节流，项目协议需要自行确定消息边界，不能把一次 read 等同于一个完整业务包。UDP 使用 `Sender` / `Listener`，需要多播时查控制选项示例。教程建议流式客户端可从同步阻塞开始，高并发服务端选择 async 接口，根据用户线程模型决定。

协程服务端在 `async::Task<>` 中 `co_await accept/read/write`，通过 `co_spawn` 注册到 `IOContext` 后 `run()`，不是调用 Task 函数就会自动运行。查头文件确认构造函数如何关联上下文，并处理断开、空读、写失败及任务退出。

### IPC / Web：选与业务匹配的上层设施

IPC 教程有同步和异步命名管道，POSIX 消息队列仅 Linux 可用，共享内存适合本机高效数据交换。共享内存的同步、数据布局和读写范围由项目约定，不将其当作自动线程安全消息总线。

Web 教程从 `requests::get/post`、Request / Response 讲到路由、中间件、JSON、静态文件与 WebSocket。需要给 Vue 3 前端提供后端时，读 `netapp.md` 和 `samples/io/http_server.cpp`：用 `async::Webapp` 注册路由，交给 `async::HttpServer` 监听，HTTPS 提供 `SSLContext` 并使用 `HttpsServer`。WebSocket 的路由回调是协程，通过 recv/send 通信。按当前头文件确认参数与返回类型后给完整工程，避免猜测类似 Express 的函数签名。

## lpss：先用内置消息打通通信

核心头文件 `rmvl/lpss/node.hpp`，汇总头 `rmvl/lpss.hpp`，内置消息例如 `rmvlmsg/std/string.hpp`。教程 `doc/tutorials/modules/tools/lpss.md` 同时提供同步、异步发布订阅和异步 Service/Client 示例。

LPSS 提供自动节点/端点发现与话题通信，底层涉及 UDP 和本机共享内存。按教程和头文件查网络配置，排查时核对话题、消息类型、端点生命周期、网络接口和发现过程。

### 同步与异步接口不可混用

同步 `rm::lpss::Node` 的 `createPublisher<T>` 返回对象，使用 `.publish()`，订阅回调由内部线程执行。保持节点、发布者与订阅者存活，回调中的共享状态需要同步。

异步 `rm::lpss::async::Node` 的端点和定时器返回智能指针，使用 `->publish()`，持有创建返回值并用 `spin()` 驱动。以下例子源于官方异步发布示例，创建节点、内置 String 发布者和定时器即可：

```cpp
#include <fmt/format.h>
#include <rmvl/lpss/node.hpp>
#include <rmvlmsg/std/string.hpp>

int main() {
    using namespace std::chrono_literals;
    rm::lpss::async::Node node("demo_pub");
    auto pub = node.createPublisher<rm::msg::String>("/topic");
    auto timer = node.createTimer(100ms, [pub] {
        rm::msg::String message;
        message.data = "hello";
        pub->publish(message);
    });
    node.spin();
}
```

订阅者使用相同话题与消息类型的 `createSubscriber<rm::msg::String>("/topic", callback)` 并保存返回值，回调参数为 `const rm::msg::String &`。CMake 选择 `lpss` 组件，最小发布订阅打通后，再引入自定义 msg 和业务逻辑。

### Service、TF 和运动设施

- Service/Client 当前由 C++20 异步接口提供。用 `rmvlsrv/std/set_bool.hpp`、`createService<rm::srv::SetBool>`、`createClient` 和 `co_await client->call(request)` 开始，请求任务通过节点 `create_task` 注册，再 `spin()`。失败返回 `std::nullopt`，每个 Client 同时只允许一个未完成调用，服务回调用于短时操作。自定义 srv 见代码生成指引。
- TF 位于 `rmvl/lpss/transform.hpp`，`tf::Broadcaster`、`StaticBroadcaster`、`Listener` 配合 `Buffer` 查询变换。Node 比通信组件存活更久，Buffer 比 Listener 存活更久。无持久化 QoS 时，后启动监听器收不到过去只发一次的静态 TF，按业务安排重发。
- 机器人模型、状态发布、轨迹与控制等按任务读 `doc/tutorials/modules/tools/lpss/robotpln.md`、`robotctl.md`、`robotdemo.md`，不在普通通信问答中展开。

右上角先搜 `LPSS` 找教程，再搜 `rm::lpss::async::Node`、`createTimer`、`createSubscriber`、`createService`、`rm::lpss::tf::Buffer` 查 API。

## opcua：地址空间与客户端操作

依赖 open62541 和实际启用的 opcua 模块。服务端头文件 `rmvl/opcua/server.hpp`，客户端 `rmvl/opcua/client.hpp`，其他类型见 `modules/opcua/include/rmvl/opcua/`，教程 `doc/tutorials/modules/tools/opcua.md`，完整程序 `samples/opcua/`。

基本顺序是创建服务器 → 定义变量/对象/方法 → 添加到地址空间 → spin，客户端连接 endpoint → 按浏览名查 NodeId → 读写或监控。

```cpp
// server.cpp
#include <rmvl/opcua/server.hpp>
int main() {
    rm::ua::Server server(4840);
    rm::ua::Variable value = 3.14;
    value.browse_name = "number";
    value.display_name = "Number";
    auto id = server.addVariableNode(value);
    if (id.empty())
        return 1;
    server.spin();
}
```

客户端创建 `rm::ua::Client("opc.tcp://127.0.0.1:4840")`，用 `client.find("number")` 找节点，再 `client.read(id)` / `write(...)`。核实连接状态和返回类型，读出的 `Variable` 如何转换为业务数据应查它的类型接口。区分 BrowseName、DisplayName、命名空间索引与 NodeId，不能把显示名当作唯一节点地址。

功能包括变量/变量类型、数据源变量、对象/对象类型、方法调用、客户端监控订阅、事件及 PubSub。高级功能依赖 open62541 编译选项，按用户需求检查相关头文件宏和构建信息。服务端可 `spin()` 长驻或在业务循环中 `spinOnce()`，退出时处理 shutdown 和线程回收。不要把 OPC UA 的 PubSub 与 LPSS 的话题接口混为一谈。

右上角搜 `OPC UA`、`rm::ua::Server`、`rm::ua::Client`、`addVariableNode`、`DataSourceVariable`、`Method`、`monitor`，先读教程对应小节，再核对 API 重载。
