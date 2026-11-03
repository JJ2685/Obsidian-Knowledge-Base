---
type: runbook
status: verified
created: 2026-11-02
updated: 2026-11-03
domain: qt
applicable_system: 无 Qt 商用许可的 Linux 环境 / 老发行版
applicable_version: Qt 5.12.x；Docker；Ubuntu 20.04
last_verified: 2026-11-02
risk: low
tags:
  - work/runbook
---

# 无 Qt 许可环境的 Linux Qt 程序编译方案

## 0. 快速结论
- **解决什么问题**：服务器/目标机无法编译 Qt 程序（qmake 报 License check failed；apt 无 WebEngine 包）。
- **适用场景**：只有 Qt 商用预编译运行库、无许可文件；需要从源码编译 Linux 二进制。
- **核心处理思路**：在任意带 Docker 的机器上跑 Ubuntu 20.04 容器，用发行版开源包（qtwebengine5-dev）编译，产物拷回目标机。
- **是否经过实际验证**：是（2026-11-02 编译 tn_frequency 成功并上线）。
- **预计影响与风险**：低——编译在容器内进行，不动目标机环境。

## 1. 适用范围
- 适用操作系统/版本：目标机凝思 V6.0.8 / Debian 8；编译机 CentOS 7 + Docker（或任何带 Docker 的机器）。
- 不适用的情况：已持有有效 Qt 商用许可（.qt-license / qtlicenses.ini）；目标机 apt 源有 qtwebengine 开源包。

## 2. 故障特征
- 常见现象：`qmake` 报 `Qt license file was not found! License check failed`；`apt install qt5-qtwebengine` 报包不存在。
- 可能原因：Qt 5.12+ 官方在线仓库的预编译二进制统一内置 licheck64 校验（与镜像来源无关，清华/阿里下载都一样）。
- 哪些现象容易混淆：上次"能装"成品 ≠ 这次"能编译"源码（运行不触发许可校验）。

## 3. 操作前检查
- [x] 已确认目标机 Qt 版本与编译目标（本项目：Qt 5.12.7，容器 Qt 5.12.8 匹配）
- [x] 已确认编译产物运行库已备好（目标机运行库无需许可）
- [x] 已备份目标机部署目录

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 目标机能否编译 | `qmake` | 通过 | 无需本方案 |
| 2 | 许可校验位置 | `strace -f qmake` | 看 licheck64 找什么文件 | 确认商用版 → 本方案 |
| 3 | 发行版有无包 | `apt-cache search qtwebengine` | 有 qtwebengine5-dev | 直接 apt 装 |
| 4 | 编译产物验证 | 目标机 `ldd` + 启动 | 无缺失、正常启动 | 补运行库 / 查 ABI |

## 5. 标准解决步骤
### 步骤 1：拉取镜像（国内加速）
```bash
docker pull docker.m.daocloud.io/library/ubuntu:20.04
```

### 步骤 2：搭建编译容器
```bash
docker run -d --name qt5-build \
  --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
  -v /root/tn_src:/src docker.m.daocloud.io/library/ubuntu:20.04 sleep infinity
docker exec -it qt5-build bash
# 容器内：apt 源切阿里云镜像后
DEBIAN_FRONTEND=noninteractive apt update && \
  debconf-set-selections <<< "tzdata tzdata/Area select Asia" && \
  debconf-set-selections <<< "tzdata tzdata/Zones/Zone select Shanghai" && \
  DEBIAN_FRONTEND=noninteractive apt install -y \
  qt5-qmake qtbase5-dev qtwebengine5-dev \
  libqt5webchannel5-dev libqt5websockets5-dev build-essential
```
**坑**：
- tzdata 交互卡死 → `DEBIAN_FRONTEND=noninteractive` + debconf 预置时区。
- 包名带 `lib` 前缀：`libqt5webchannel5-dev`、`libqt5websockets5-dev`（不是 qtwebchannel5-dev）。
- 诊断需要 strace/gdb 时容器要 `--cap-add=SYS_PTRACE --security-opt seccomp=unconfined`。
- 版本确认：Ubuntu 20.04 的 Qt 是 5.12.8（匹配目标 5.12.7）。

### 步骤 3：编译与绕坑
```bash
cd /src && qmake && make
```
- **uic 报 `libQt5Core.so.5: cannot open shared object file`**（SYSV ABI 加载失败，qmake/moc/rcc 正常）→ `.ui` 极简时手写 `ui_mainwindow.h`，并改 Makefile 把 uic 命令替换为空操作。
- 源码编译错误按日志修（如 `fabs` 未声明 → `#include <cmath>`）。
- 不要复用旧 Makefile（可能混有 Qt6/MSVC 生成物），每次重新 qmake。

### 步骤 4：拷回目标机 + 启动验证
```bash
scp tn_frequency root@目标机:/opt/app/bin/
```
- 启动脚本用已验证的 Qt 运行库（本项目用 5.15.2 + softpipe，见 [[RB-Qt-海光CPU图形渲染问题排查]]）。
- **SIOF 排查**：启动即 SIGSEGV 且 gdb 栈含 `_GLOBAL__sub_I_*.cpp` + `QtPrivate::RefCount::ref()` → 静态初始化顺序 fiasco：非字面量类型的 `const QString` 等静态成员定义必须排在引用它们的常量**之前**。
- **界面核对**：源码自带的 html 可能只是测试页，完整界面可能在旧版部署目录里（`ws://localhost:2018` 协议相同可直接复用）。

## 6. 验证标准
- [x] 编译产物为 Linux ELF，`ldd` 无缺失
- [x] 目标机启动无崩溃（无 SIOF 段错误）
- [x] 功能验证：端口监听、配置加载、界面正常
- [x] 与目标机 Qt 运行库 ABI 实测兼容

## 7. 回滚方法
- 产物替换前备份旧二进制与启动脚本。

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| 目标机可上网 | 直接下载 Qt 7z 包？ | 只能补**运行库**；编译仍卡许可 |
| 需在 Windows 编译 Linux 产物 | 交叉编译复杂 | 不推荐，优先 Docker |
| 老 glibc 目标机 | 容器 glibc 较新 | 用老容器镜像（如 Ubuntu 16.04）或静态链接 |

## 9. 注意事项与禁止操作
- 不要伪造/空写 `.qt-license` 文件（格式强校验，报 Invalid license key）。
- 编译容器里不要装 Qt 官方在线安装器（同样带许可校验）。
- 编译与运行分离：目标机缺的是编译环境，不缺运行库。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261102-Qt-服务器无法编译]]

### 验证环境
- 系统/版本：CentOS 7.7 VM + Docker 26.1.4 + Ubuntu 20.04 容器；目标机凝思 V6.0.8
- 验证日期：2026-11-02
- 验证结论：通过（tn_frequency 162KB 产物上线运行）

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库三篇编译笔记抽象整理 | 知识库重建 |
