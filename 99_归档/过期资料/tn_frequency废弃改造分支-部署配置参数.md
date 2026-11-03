# tn_frequency 远程取数改造 — 部署前配置参数清单

> 适用对象：`tn_frequency_change_qt5.12`（Qt5.12.7，运行于凝思 V6.0.80 工作站/服务器）
> 配套文档：`tn_frequency_结构分析与远程化改造方案.md`、`tn_frequency_代码改动与依据.md`
> 生成日期：2026-10-30

本文档列出**上线前必须配置/确认的参数**，每项标注「代码读取位置」与「服务器上的配置方法」。

---

## 一、GUI 侧「必须配置」的参数（写进 config.ini）

程序启动时由 `appconfig.cpp::readConfig()` 读取，文件位于**可执行文件同目录**下的 `config.ini`（`readConfig` 用 `applicationDirPath()` 定位，因此 config.ini 必须和 exe 放在一起；因 `DESTDIR=bin`，即 `bin/config.ini`）。

| 参数 | 默认值/占位 | 代码读取位置 | 含义 | 服务器配置方法 |
|---|---|---|---|---|
| `UdpServerIP` | `192.168.1.100`（占位） | `appconfig.cpp` L33、`checkIniFile` L92 | 远程数据端服务器 IP（GUI 投运开关等**命令**发往此处） | **必改**：改成真实数据端 IP。校验为合法 IPv4，否则程序 `qCritical` 报错 |
| `UdpServerPort` | `11001` | `appconfig.cpp` L34、`checkIniFile` L106 | 远程数据端**命令监听端口** | 一般保持 `11001`；须与数据端实际监听端口一致；范围 1–65535 |
| `UdpListenIP` | `0.0.0.0` | `appconfig.cpp` L35、`checkIniFile` L99 | 工作站本机 UDP **监听地址**（接收实时数据） | 默认 `0.0.0.0`（所有网卡）。若只允许特定网卡，改为工作站业务网卡 IP。**切勿填 127.0.0.1**（远程发不进来）。合法 IPv4 校验 |
| `UdpListenPort` | `11002` | `appconfig.cpp` L36、`checkIniFile` L114 | 工作站**接收实时数据端口** | 一般保持 `11002`；范围 1–65535 |

### config.ini 文件格式（示例）
```ini
[UdpConfig]
UdpServerIP=192.168.1.100   ; ← 改成真实远程数据端 IP
UdpServerPort=11001
UdpListenIP=0.0.0.0
UdpListenPort=11002
```
- **四个值都必须非空且合法**，否则 `checkIniFile()` 判定不完整 → 程序用代码默认值重写并 `qCritical` 报错（改造失效）。
- 注释行以 `;` 或 `#` 开头会被跳过，可放心加说明。

---

## 二、代码中「硬编码、部署无需改」的参数（仅标注位置，勿改）

| 参数 | 代码位置 | 说明 |
|---|---|---|
| WebSocket 界面桥端口 `2018` | `mainwindow.cpp` L535 `m_WebSocketServer->listen(QHostAddress::LocalHost, 2018)`（经 `WebBrowserInit(2018)` 传入） | **保持 `LocalHost`**。这是 GUI 内嵌浏览器↔Qt 后端通信，同机，不可跨机 |
| 前端连界面桥地址 | `html/js/public.js` L61/76、`index.js` L179、`login.js` L74 `ws://localhost:2018` | 与上面配套，保持 `localhost` |
| 录波/模板/业务配置目录 | `mainwindow.cpp` L18–28、`csvread.cpp` L8–16（`FILE_DIR=/DataOut/`、`TIG_FILE_DIR=/LoBoTest/`、`TEMP_FILE_DIR=/opt/Template/`、`CONFIG_FILE_PATCH=/PfreqCnf/config`、`LOGO_FILE_PATCH=/opt/logo/tn_frequency.logo`） | 不改成远程地址，靠 NFS/SMB 挂载解决（见第三节） |
| `UdpListenIP` 代码默认值 | `appconfig.cpp` L14 `"0.0.0.0"` | 仅当 config.ini 缺省时生效；最终以 config.ini 为准 |

---

## 三、服务器侧基础设施配置方法

### 1. 网络 / 防火墙（必须）
- **工作站（运行 GUI 的服务器）**：放行 **UDP 11002 入方向**（接收远程实时数据）。因 `UdpListenIP=0.0.0.0` 监听所有网卡，建议用防火墙把来源限制为**仅数据端 IP**。
- **远程数据端服务器**：放行 **UDP 11001 入方向**（接收 GUI 命令）；且数据端必须能把实时数据发送到**工作站 IP:11002**。
- 使用专用工业网络/VPN，勿将控制 UDP 端口暴露公网。

### 2. 录波目录共享（必须，靠挂载而非改代码）
GUI 仍按本地路径读 `/DataOut`、`/LoBoTest`、`/opt/Template`、`/PfreqCnf`、`/opt/logo`。把远程数据端对应目录以**只读（除 `/LoBoTest` 按需可写）**方式挂载到工作站同名路径：
```bash
# 工作站上（示例，NFS）
mount -t nfs <数据端IP>:/DataOut        /DataOut
mount -t nfs <数据端IP>:/LoBoTest       /LoBoTest
mount -t nfs <数据端IP>:/opt/Template   /opt/Template
mount -t nfs <数据端IP>:/PfreqCnf       /PfreqCnf
mount -t nfs <数据端IP>:/opt/logo       /opt/logo
```
- 写入 `/etc/fstab` 实现开机自动挂载；挂载点须真实存在，避免挂载失败后 GUI 误读本地空目录。
- 建议：`/DataOut` 只读；仅当确认后台依赖 `/LoBoTest` 时再建立可写共享。
- 网络文件系统异常时 GUI 应给出明确提示（当前代码未做挂载失效检测，属遗留项，需在验收中关注）。

### 3. 编译 / 运行（必须）
- 在目标机（或兼容环境）用 **Qt 5.12.7 + GCC Kit** 重新 `qmake && make`，**不要复用旧 Makefile**（工程中混入过 Qt6.4.3/Windows MSVC 的生成物）。
- 产物放到 `bin/`，确保 `bin/config.ini` 与可执行文件同目录。
- 启动建议直接运行 exe（已用 `applicationDirPath()`，不再依赖 CWD）；若用启动脚本/开机启动，CWD 也不再影响资源定位。

### 4. 启动顺序与权限
- 先确认数据端服务已在远程启动并监听 `11001`、向工作站 `11002` 发实时数据。
- GUI 以具备读 `/DataOut` 等目录权限的用户运行；日志写到 `/opt/logo/tn_frequency.logo`。

---

## 四、部署验证清单（建议逐项确认）

- [ ] `bin/config.ini` 四项填写正确（真实 `UdpServerIP`、端口、监听地址），且非空合法。
- [ ] 工作站防火墙放行 UDP 11002（来源限数据端 IP）；数据端放行 UDP 11001。
- [ ] 远程 `/DataOut` 等已挂载到工作站同名路径，`df`/`ls` 确认可读。
- [ ] 用 Qt5.12.7 重新编译成功，二进制与源码一致。
- [ ] 启动 GUI，日志无 `config.ini invalid …` / `UDP bind failed` 等 `qCritical`。
- [ ] 抓包确认数据报真正跨机：`sudo tcpdump -ni any 'udp port 11001 or udp port 11002'`，操作实时监测/投运开关，确认源/目的 IP 为双机真实地址。
- [ ] 前端"实时监测"曲线有数据、投运开关在数据端生效、"查询记录"能列出远程录波文件。
- [ ] 异常用例：数据端断网/重启、UDP 端口被占用、config.ini 错 IP/非法端口、NFS 挂载失败——GUI 应报错或提示，不卡死/不崩溃。
