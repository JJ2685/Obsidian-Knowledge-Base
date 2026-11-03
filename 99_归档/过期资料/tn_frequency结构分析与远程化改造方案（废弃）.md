# tn_frequency 快速频率响应系统 —— 软件结构分析与远程化改造方案

> 分析对象（均来自 `E:\BaiduNetdiskDownload\服务器程序\任务\`）：
>
> - `tn_frequency\tn_frequency\` —— 原始系统源码（Qt Creator 2018，称"系统源码"）
> - `tn_frequency_change_qt5.12\tn_frequency_change_qt5.12\` —— 升级版（Qt5.12，已引入配置机制）
> - `Reqlian_frequency-fan-v2.1\Reqlian_frequency-fan-v2.1\` —— 数据/控制后台服务 + Web 前端  
>   部署目标系统：凝思安全操作系统 V6.0.80（Debian 8），Qt 5.12.7 运行时（见 `tn_frequency_deploy/check_env.sh`）



---

## 1. 一句话结论

本系统由**两个程序**组成：

- **`tn_frequency`（GUI 工作站程序）**：Qt5 + QWebEngine 桌面程序，内嵌浏览器展示 Web 前端（layui + echarts），通过 **QWebChannel over WebSocket** 与前端通信，并读取本地 `/DataOut` 的 CSV 录波文件做分析。
- **`Reqlian_frequency`（数据/控制后台服务）**：独立 Linux 守护进程（编译好的二进制，装到 `/usr/sbin/`），负责采集频率数据、**写 CSV 录波文件到 `/DataOut`**、并通过 **UDP** 把实时数据发给 GUI、接收 GUI 下发的控制命令。

当前两者都跑在**同一台机器（127.0.0.1）**。你要做的是：把"数据端"（`Reqlian_frequency`）挪到**远程服务器**，GUI 仍在工作站本地运行。需要改的只有 **UDP 数据通道** +（可选）**数据目录路径**，而 **WebSocket(LocalHost:2018) 是 GUI 内部的"界面桥"，必须保持本地回环不动**。

---

## 2. 三个目录的角色定位

| 目录                             | 角色                       | 关键内容                                                                                                                   |
| ------------------------------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `tn_frequency`                 | **GUI 原始源码**（2018）       | `mainwindow.cpp/.h`、`webengie.*`、`csvread.*`、`websocket/`、`index.html`（最简 qwebchannel 测试页）、`tn_frequency.pro`          |
| `tn_frequency_change_qt5.12`   | **GUI 升级版**（2023，Qt5.12） | 在原始版基础上**新增 `appconfig.cpp/.h` + `config.ini`**，把 UDP 端点参数化；`.pro` 加了 `core5compat`；`html/` 内含完整产品前端                   |
| **Reqlian_frequency-fan-v2.1** | **数据/控制后台 + Web 前端包**    | 二进制 `Reqlian_frequency`、`html/`（layui 产品 UI：`index.html/real.html/table.html/login.html` + `js/`）、`install.sh`、`user/` |

> 注：`Reqlian_frequency` 的 `html/` 与 `tn_frequency_change_qt5.12/html/` 是同一套前端（均通过 `ws://localhost:2018` 连 QWebChannel）。GUI 用 `ui->WebBrowser->load("../html/index.html")` 内嵌加载它。

---

## 3. 系统总体架构（双程序）

```
┌─────────────────────────── 工作站（本地回环 + 界面） ───────────────────────────┐
│                                                                                  │
│   tn_frequency (Qt GUI)                                                            │
│   ├─ QWebEngineView 内嵌浏览器 ──加载 ../html/index.html ──┐                        │
│   ├─ QWebSocketServer :2018 (LocalHost)  ◀── QWebChannel(WebSocket) ──┘ 界面桥    │
│   ├─ 读 /DataOut/*.csv → csvread 分析（录波历史）                                  │
│   └─ QUdpSocket                                                                   │
│        ├─ bind  UdpListenIP:11002  ◀── 实时数据（来自数据端）                       │
│        └─ send  UdpServerIP:11001  ──▶ 控制命令（投运开关等）                       │
└──────────────────────────────────────────────────────────────────────────────────┘
        │                                   ▲
        │ UDP 命令(11001)                     │ UDP 实时(11002)
        ▼                                   │
┌─────────────────────────── 远程服务器（数据端，本次新增） ─────────────────────────┐
│   Reqlian_frequency (守护进程 /usr/sbin/)                                            │
│   ├─ 采集频率数据（对接现场/RTU）                                                    │
│   ├─ 写 CSV 录波文件 → /DataOut/                                                    │
│   ├─ 监听 UDP :11001  ◀── 接收 GUI 命令                                            │
│   └─ 发送 UDP 实时数据 ──▶ GUI 的 11002                                            │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**关键判断：** `WebSocket :2018` 是"浏览器↔Qt后端"的**界面桥**，二者同在 GUI 进程内/同机，必须保持 `LocalHost`。真正跨机的是 **UDP 11001/11002** 与 **CSV 文件目录**——这正是"数据请求使用 127.0.0.1 本地回环"所指之处。

---

## 4. 数据流详情

### 4.1 实时数据 / 控制通道（UDP）

- GUI 接收实时数据：`MainWindow::udp_read_data()`（`mainwindow.cpp`）→ `WebEngieFun.QtSendReal()` → 前端 `RealChanged` 信号 → echarts 曲线。
- GUI 下发控制命令（投运开关）：前端 `WebSendMessage` → `OnReceiveFromWeb` 的 `cmd=120` → `udp_send_data()`。
- 端点硬编码位置（原始版 `tn_frequency`）：
  - `mainwindow.cpp:529` `m_WebSocketServer->listen(QHostAddress::LocalHost, 2018)` —— **界面桥，保持 LocalHost**
  - `mainwindow.cpp:657` `UdpSocket->bind(QHostAddress::LocalHost, 11002)` —— 接收实时数据，**需改为可配置**
  - `mainwindow.cpp:681` `UdpSocket->writeDatagram(..., QHostAddress::LocalHost, 11001)` —— 发送命令，**需改为可配置**

### 4.2 录波文件（CSV，本地文件 I/O）

- 目录宏（`mainwindow.cpp:16-28` / `csvread.cpp:8-16`）：
  | 宏                   | Linux 路径                      | 用途                           |
  | ------------------- | ----------------------------- | ---------------------------- |
  | `FILE_DIR`          | `/DataOut/`                   | 录波 CSV（历史记录，前端"查询记录"读取）      |
  | `TIG_FILE_DIR`      | `/LoBoTest/`                  | 手动触发写出的文件                    |
  | `TEMP_FILE_DIR`     | `/opt/Template/`              | 模板保存/查阅                      |
  | `CONFIG_FILE_PATCH` | `/PfreqCnf/config`            | 业务配置文件（csvread 读装机容量/门槛/调差率） |
  | `LOGO_FILE_PATCH`   | `/opt/logo/tn_frequency.logo` | 日志                           |
- 数据端 `Reqlian_frequency` 把录波 CSV 写到它**自己机器**的 `/DataOut`。一旦数据端搬到远程，GUI 在本地就**读不到**这些 CSV。需通过挂载或路径配置解决（见第 9 节）。

### 4.3 前端 ↔ Qt 通信（QWebChannel）

- 前端 `html/js/public.js:61,76` 与 `index.js:169`、`login.js:80` 均 `ws://localhost:2018` 连接。
- Qt 侧 `WebBrowserInit()` 建 `QWebSocketServer(LocalHost,2018)`，`registerObject("WebChannel", &WebEngieFun)`。
- 协议：`WebSendMessage(text)`（JS→Qt，JSON 命令）、`MessageChanged`/`RealChanged`（Qt→JS 信号）。命令字见 `OnReceiveFromWeb` 的 switch（100 列文件 / 101 按天 / 102 按时间 / 103 读指定 / 104 读最新 / 110 触发 / 111 存模板 / 112 查模板 / 120 投运开关 / 121 读配置）。

---

## 5. 模块与文件清单（精确路径 + 职责）

### 5.1 GUI 程序（`tn_frequency` / `tn_frequency_change_qt5.12` 共有）

| 文件                                                                    | 职责                                                                  |
| --------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `main.cpp`                                                            | 入口；qt5.12 版在此调用 `AppConfig::readConfig()`（第12行）                     |
| `mainwindow.h/.cpp`                                                   | 主窗口：Web 初始化、UDP 初始化、文件/模板/配置读写、命令分发                                 |
| `webengie.h/.cpp`                                                     | `WebEngie` QObject，QWebChannel 暴露的"WebChannel"对象，桥接 JS↔Qt           |
| `csvread.h/.cpp`                                                      | CSV 录波文件读取与指标分析（Thx/T09/Ts/Ps/Pe/P/Qs/Qe…）                          |
| `websocket/websocketclientwrapper.*` `websocket/websockettransport.*` | QWebChannel 的 WebSocket 传输封装                                        |
| `mainwindow.ui`                                                       | Qt Designer 界面（含 `WebBrowser` QWebEngineView）                       |
| `html/`                                                               | 产品前端（layui + echarts），由 GUI 内嵌加载                                    |
| `appconfig.h/.cpp`                                                    | **【仅 qt5.12 版有】** UDP 端点配置读写（config.ini）                            |
| `config.ini`                                                          | **【仅 qt5.12 版有，在 bin/】** 实际配置：`UdpServerIP/Port`、`UdpListenIP/Port` |
| `tn_frequency.pro`                                                    | qmake 工程；qt5.12 版多出 `core5compat` 与 `DESTDIR=bin`                   |

### 5.2 数据/控制后台（`Reqlian_frequency-fan-v2.1`）

| 文件                                    | 职责                                                                     |
| ------------------------------------- | ---------------------------------------------------------------------- |
| `Reqlian_frequency/Reqlian_frequency` | 编译好的后台二进制（2MB），装到 `/usr/sbin/`                                         |
| `html/`                               | 同一套 Web 前端（含 `js/qwebchannel.js`、`real.js`、`tn-line.js`、`public.js` 等） |
| `user/info`                           | 用户/授权信息                                                                |
| `install.sh`                          | 安装脚本：`cp` 二进制到 `/usr/sbin`、`html`→`/opt/html`、`user`→`/opt/user`       |

---

## 6. 当前"本地回环"硬编码点汇总

| 位置                                             | 端点                               | 性质          | 是否需改                    |
| ---------------------------------------------- | -------------------------------- | ----------- | ----------------------- |
| `mainwindow.cpp:529`                           | `WebSocketServer LocalHost:2018` | 界面桥（浏览器↔Qt） | **否**（保持本地）             |
| `index.html / public.js / index.js / login.js` | `ws://localhost:2018`            | 前端连界面桥      | **否**（保持本地）             |
| `mainwindow.cpp:657`                           | `UDP bind LocalHost:11002`       | 接收实时数据      | **是**（改监听地址为可路由地址）      |
| `mainwindow.cpp:681`                           | `UDP send LocalHost:11001`       | 发控制命令       | **是**（改为远程数据端 IP）       |
| `mainwindow.cpp:16-28` `csvread.cpp:8-16`      | `/DataOut` 等目录                   | 录波文件 I/O    | **视部署方式**（远程数据端时需挂载或配置） |

> qt5.12 版已把 657/681 两处的 `LocalHost` 替换为 `AppConfig::UdpListenIP/Port` 与 `AppConfig::UdpServerIP/Port`，并加了 `config.ini`。**唯独没有把文件目录（`FILE_DIR` 等）参数化**，也未对"远程数据端"给出部署说明。

---

## 7. 远程化改造目标架构

```
工作站(GUI)                              远程服务器(数据端 Reqlian_frequency)
┌────────────────────┐                   ┌──────────────────────────┐
│ tn_frequency       │                   │ Reqlian_frequency       │
│ bind 0.0.0.0:11002 │◀── UDP 实时数据 ──│ 写 /DataOut/*.csv        │
│ send <远程IP>:11001│── UDP 命令 ──────▶│ 监听 :11001              │
│ 读 /DataOut (挂载) │   ← NFS/SMB 挂载 →│ /DataOut 在此机上         │
│ WebSocket LocalHost:2018 (界面桥,不变) │                          │
└────────────────────┘                   └──────────────────────────┘
```

---

## 8. 改造方案（代码改动点）

### 8.1 必改：UDP 端点参数化（qt5.12 版已基本完成，原始版需移植）

- 复用/新增 `appconfig`：`UdpServerIP`(命令目标=远程数据端IP)、`UdpServerPort`(11001)、`UdpListenIP`(GUI 监听=工作站可路由地址，建议 `0.0.0.0`)、`UdpListenPort`(11002)。
- `main.cpp` 最早调用 `AppConfig::readConfig()`（qt5.12 已做）。
- `SocketUdpInit()` 用 `AppConfig::UdpListenIP/Port` 绑定；`udp_send_data()` 用 `AppConfig::UdpServerIP/Port` 发送（qt5.12 已做）。
- `config.ini` 默认值改为远程数据端 IP + `UdpListenIP=0.0.0.0`。

### 8.2 建议改：数据目录参数化（解决远程 CSV 读不到）

- 在 `AppConfig` 增加 `FileDir`/`TigFileDir`/`TempFileDir`/`ConfigFile`/`LogoFile`，从 `config.ini` 读取，替换 `mainwindow.cpp` 与 `csvread.cpp` 中的硬编码宏。
- 这样无论远程 `/DataOut` 是通过 NFS 挂载到同名路径，还是挂到别的路径，都能不改编译地适配。

### 8.3 不动：界面桥 WebSocket

- `WebSocketServer` 继续 `listen(QHostAddress::LocalHost, 2018)`；前端 `ws://localhost:2018` 不变。GUI 与内嵌浏览器同机，符合"在工作站上运行界面"。

---

## 9. 部署与验证建议

1. **远程数据端**：在远程服务器部署 `Reqlian_frequency`（`install.sh`），确认其监听 UDP `:11001` 并向工作站 `:11002` 发实时数据；录波仍写远程 `/DataOut`。
2. **工作站 GUI**：
   - `config.ini`：`UdpServerIP=<远程数据端IP>`、`UdpServerPort=11001`、`UdpListenIP=0.0.0.0`、`UdpListenPort=11002`。
   - 录波目录：推荐用 **NFS/SMB 把远程 `/DataOut`、`/LoBoTest`、`/opt/Template`、`/PfreqCnf` 挂载到工作站同名路径**（零代码改动）；或把 8.2 的目录做成可配置。
3. **防火墙/网络**：工作站需放行入向 UDP `11002`；远程需放行入向 UDP `11001`；两机路由可达。
4. **验证**：  
   |  |  |  
   | --- | --- |
   - `netstat -ulnp | grep 11002` 应看到 `0.0.0.0:11002`（而非 `127.0.0.1`）。
   - 前端"实时监测"曲线有数据 → UDP 实时通。
   - 投运开关动作在远程数据端生效 → UDP 命令通。
   - "查询记录"能列出远程录波文件 → 目录可达。

---

## 10. 风险与待确认

- **R1 编译环境**：原始 `tn_frequency`(2018) 配套 Qt 较旧，部署目标为 Qt5.12.7；建议以 `tn_frequency_change_qt5.12` 为改造基线（已含 appconfig 且匹配运行时）。
- **R2 CSV 可达性**：数据端上云/上远程后，`/DataOut` 必须能被工作站读到（挂载 or 配置），否则"查询记录/分析"功能失效。
- **R3 网络安全**：`0.0.0.0:11002` 暴露在所有网卡，建议在局域网/专线内，或由防火墙仅放行数据端 IP。
- **待确认**：改造基线选哪个目录、远程数据端 IP/端口、CSV 目录采用"挂载"还是"配置化"。

---

*生成于 2026-10-30。本文档基于源码静态分析，改造代码前请先确认第 10 节待确认项。*
