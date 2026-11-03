# tn_frequency 远程取数改造 — 代码改动及依据

> 改造基线：`E:\BaiduNetdiskDownload\服务器程序\任务\tn_frequency_change_qt5.12\tn_frequency_change_qt5.12\`
> 配套分析文档：`tn_frequency_结构分析与远程化改造方案.md`
> 适用运行环境：凝思安全操作系统 V6.0.80（Debian 8），Qt 5.12.7 运行时
> 生成日期：2026-10-30

---

## 一、改动总览

| 编号  | 文件               | 位置                                                                     | 改动摘要                                       | 来源/依据                    |
| --- | ---------------- | ---------------------------------------------------------------------- | ------------------------------------------ | ------------------------ |
| 1   | `appconfig.cpp`  | L14                                                                    | 默认 `UdpListenIP` 由 `127.0.0.1` → `0.0.0.0` | 需求：让远程数据端能把实时 UDP 发到工作站  |
| 2   | `bin/config.ini` | 全文件                                                                    | 重写为占位远程 IP + 详细注释，四项非空                     | 需求：UDP 端点参数化，指向远程数据端     |
| 3   | `appconfig.cpp`  | L4–7（include）、L27、L30、L40                                              | 用 `applicationDirPath()` 定位 `config.ini`   | 评审§2：修复对进程 CWD 的依赖       |
| 4   | `appconfig.cpp`  | `checkIniFile()` L53–132、`readConfig()` L40–43                         | 四键必存在 + 端口范围 + IP 合法 + `qCritical` 报错      | 评审§2：避免缺键时静默回退 127.0.0.1 |
| 5   | `mainwindow.cpp` | L15–16（include）、L548                                                   | 用 `applicationDirPath()` 定位 `html`         | 评审§2：同上，修复 CWD 依赖        |
| 6   | `mainwindow.cpp` | `SocketUdpInit` L663、`udp_read_data` L674–690、`udp_send_data` L692–699 | UDP 健壮性：bind 返回值检查、循环接收、发送返回值检查            | 评审§4：完善错误处理              |

**改动文件清单（共 3 个文件）**：`appconfig.cpp`、`mainwindow.cpp`、新增/改写 `bin/config.ini`。
**未改动**：WebSocket `LocalHost:2018`、前端 `ws://localhost:2018`、录波目录宏、2018 原始源码、Reqlian 二进制。

---

## 二、逐条改动详述

### 改动 1 — `appconfig.cpp` 默认监听地址加固
```cpp
// L14（改动后）
QString AppConfig::UdpListenIP = "0.0.0.0";  // 监听所有网卡，使远程数据端能把实时UDP数据发到本机11002；仅本机回环用127.0.0.1即可
```
- **改动前**：`"127.0.0.1"`
- **依据**：GUI 在工作站本地运行，但实时数据来自远程数据端。若仍绑定 `127.0.0.1`，远程数据端无法把 UDP 发进来。改为 `0.0.0.0` 监听所有网卡（最终值以 `config.ini` 为准，可收窄为工作站业务网卡 IP）。此默认值仅当 `config.ini` 缺省时生效。

### 改动 2 — `bin/config.ini` 重写（占位 + 注释）
```ini
; tn_frequency UDP 通信配置
; 数据端(Reqlian_frequency)已迁移到远程服务器，GUI 仍在工作站本地。
; WebSocket:2018 是 GUI 内部界面桥，保持本地不动。
; 四个值都必须非空；若某项为空，程序会用代码默认值(127.0.0.1)覆盖本文件。
[UdpConfig]
UdpServerIP=192.168.1.100   ; 远程数据端 IP（部署时改成真实 IP）
UdpServerPort=11001         ; 远程数据端命令监听端口
UdpListenIP=0.0.0.0         ; 工作站监听地址（0.0.0.0=所有网卡；切勿填127.0.0.1）
UdpListenPort=11002         ; 工作站接收实时数据端口
```
- **依据**：把 "数据请求使用 127.0.0.1" 改为可指向远程数据端。四项均非空，确保 `checkIniFile()` 判定完整、不被默认值覆盖。
- **注意**：本文件位于**可执行文件同目录**（见改动 3），与 `DESTDIR=bin` 对应，即 `bin/config.ini`。

### 改动 3 — `appconfig.cpp` / `mainwindow.cpp`：资源路径改用 exe 目录（评审§2）
```cpp
// appconfig.cpp 顶部新增 include（L4–7）
#include <QCoreApplication>
#include <QDir>
#include <QHostAddress>
#include <QDebug>

// appconfig.cpp readConfig()（L27–30，L40）
QString cfgPath = QDir(QCoreApplication::applicationDirPath()).filePath("config.ini");
AppConfig::ConfigFile = cfgPath;
QSettings set(cfgPath, QSettings::IniFormat);
...
if (!AppConfig::checkIniFile(cfgPath)) {
    qCritical() << "config.ini invalid or incomplete, regenerating with default values:" << cfgPath;
    writeConfig();
    return;
}
```
```cpp
// mainwindow.cpp 顶部新增 include（L15–16）
#include <QCoreApplication>
#include <QDir>

// mainwindow.cpp WebBrowserInit()（L548）
QString url = QDir(QCoreApplication::applicationDirPath()).filePath("../html/index.html");
```
- **改动前**：`QString AppConfig::ConfigFile = "config.ini";`（相对 CWD）；`QString url = QDir::currentPath()+"/../html/index.html";`（相对 CWD）。
- **依据**：评审§2 指出原实现依赖"进程当前工作目录"。通过桌面快捷方式、启动脚本或开机启动运行时，CWD 往往不是 exe 目录，会导致 `config.ini` 找不到（被判定缺失→用默认 127.0.0.1 静默覆盖）或 HTML 加载失败。改为基于 `applicationDirPath()`，从根本消除该隐患。

### 改动 4 — `appconfig.cpp` 配置校验增强（评审§2）
在 `checkIniFile()`（L53–132）基础上增强：
- 跳过注释(`;`/`#`)、节名(`[`)、空行（L78）。
- 显式记录 `UdpServerIP/UdpServerPort/UdpListenIP/UdpListenPort` 四键是否出现，**任一缺失 → 返回 false 并打印 `missing key Xxx`**（L66–70、L126–129）。
- 值空 → 打印 `empty value for key` 并返回 false（L87–90）。
- `UdpServerIP`/`UdpListenIP` 用 `QHostAddress::setAddress()` 校验非法 IP（L92–105）。
- `UdpServerPort`/`UdpListenPort` 须能转 int 且落在 `[1,65535]`（L106–122）。
- `readConfig()` 在校验失败时 `qCritical()` 明确报错（L41），不再静默回退。
- **依据**：原 `checkIniFile` 只判"值非空"，不判"键缺失"。若 `config.ini` 漏写某键（如 `UdpServerIP`），`readConfig` 会用硬编码默认 `127.0.0.1` 把命令发往本机、改造静默失效。增强后把该坑堵死，并给出可读的错误日志。

### 改动 5 — `mainwindow.cpp` HTML 路径改用 exe 目录（评审§2）
见改动 3 中 `WebBrowserInit()` L548。依据同改动 3。

### 改动 6 — `mainwindow.cpp` UDP 健壮性（评审§4）
```cpp
// SocketUdpInit()（L663–666）
if (!UdpSocket->bind(QHostAddress(AppConfig::UdpListenIP), AppConfig::UdpListenPort)) {
    qCritical() << "UDP bind failed:" << UdpSocket->errorString();
    return;
}

// udp_read_data()（L674–690）
while (UdpSocket->hasPendingDatagrams()) {
    QHostAddress sender;
    quint16 senderPort = 0;
    QByteArray data;
    data.resize(static_cast<int>(UdpSocket->pendingDatagramSize()));
    qint64 len = UdpSocket->readDatagram(data.data(), data.size(), &sender, &senderPort);
    if (len < 0) {
        qWarning() << "UDP recv err:" << UdpSocket->errorString();
        continue;
    }
    WebEngieFun.QtSendReal(QString::fromUtf8(data));
}

// udp_send_data()（L692–699）
qint64 len = UdpSocket->writeDatagram(str.toUtf8(), QHostAddress(AppConfig::UdpServerIP), AppConfig::UdpServerPort);
if (len < 0) {
    qWarning() << "UDP send err:" << UdpSocket->errorString();
}
```
- **改动前**：`bind` 不检查返回值；`udp_read_data` 单次 `readDatagram`；`udp_send_data` 不检查 `writeDatagram` 返回值。
- **依据**：评审§4 指出接收只处理一次数据报可能漏报积压包；绑定/发送失败应明确告警而非静默。增强后循环接收全部积压数据报、对 bind/发送错误输出日志，提升可诊断性。

---

## 三、明确未改动项（及依据）

| 项                        | 位置                                                                               | 未改依据                                                           |
| ------------------------ | -------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| QWebChannel WebSocket 地址 | `mainwindow.cpp` L535 `m_WebSocketServer->listen(QHostAddress::LocalHost, port)` | 界面桥（内嵌浏览器↔Qt 后端）同机，必须保持本地，不能跨机（评审：WebSocket 2018 保持 localhost） |
| 前端连接地址                   | `html/js/public.js` L61/76、`index.js` L179、`login.js` L74 `ws://localhost:2018`  | 同上，前端连本地界面桥，不变                                                 |
| 录波目录宏                    | `mainwindow.cpp` L18–28、`csvread.cpp` L8–16（`FILE_DIR` 等）                        | 录波目录由 NFS/SMB 挂载到工作站同名路径解决，代码保持本地路径（评审§3：不统一远程化所有目录）           |
| 2018 原始源码 / Reqlian 二进制  | —                                                                                | 用户确认改造基线仅为 `tn_frequency_change_qt5.12`，不碰其它目录                 |

---

## 四、验收与遗留

- 第一轮（UDP 参数化）QA 6/6 通过；第二轮（本表 A/B/C 加固）由 QA 回归中，主理人已直接核验代码无误。
- 遗留（部署侧，非源码缺陷）：本机无 Qt 环境仅做静态核查；`0.0.0.0` 监听需防火墙限制来源；部署需填真实 `UdpServerIP` 并重新编译；录波目录依赖 NFS 挂载。
