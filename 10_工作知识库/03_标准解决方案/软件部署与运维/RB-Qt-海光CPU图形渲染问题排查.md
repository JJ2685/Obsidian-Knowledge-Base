---
type: runbook
status: verified
created: 2026-10-30
updated: 2026-11-03
domain: qt
applicable_system: 凝思 / 国产 x86 平台（海光、兆芯等 Zen 架构）
applicable_version: Qt 5.12/5.15 WebEngine；Mesa 10.x（老发行版）
last_verified: 2026-10-30
risk: medium
tags:
  - work/runbook
---

# 海光 CPU 上 Qt WebEngine 图形渲染问题排查

## 0. 快速结论
- **解决什么问题**：国产 x86 CPU 上 Qt/Chromium 程序渲染崩溃（LLVM ERROR）、白屏、界面卡顿。
- **适用场景**：海光/兆芯等 Zen 架构 CPU + 老发行版（Mesa 10.x / Qt 5.12 内置 SwiftShader）。
- **核心处理思路**：按决策树逐层判断——先确认 Xorg 是否有硬件 GL，再看崩溃/卡顿层级，最后选渲染路径（nouveau 硬件 GL + `--disable-gpu` 或 softpipe 兜底）。
- **是否经过实际验证**：是（2026-10-30 两次事件完整验证）。
- **预计影响与风险**：中——切换 Xorg 驱动需重启 lightdm（结束图形会话）。

## 1. 适用范围
- 适用操作系统/版本：凝思 V6.0.8（Xorg 1.16.4 + Mesa 10.3）；Debian 8/9 老发行版同理。
- 适用硬件/架构：海光/兆芯 x86（Zen）；普通 Intel/AMD 机器一般无此问题。
- 不适用的情况：Mesa 版本较新（≥18）的系统——llvmpipe 已支持 Zen，优先排查其他原因。

## 2. 故障特征
- 常见现象：程序启动即崩溃（LLVM ERROR / SIGSEGV）；页面白屏数分钟；CPU 单核满载卡顿。
- 典型报错：
  ```
  LLVM ERROR: Cannot select: intrinsic %llvm.x86.sse41.pblendvb
  QXcbIntegration: Cannot create platform OpenGL context, neither GLX nor EGL are enabled
  ERROR:gles2_cmd_decoder.cc ... Could not allocate offscreen buffer storage
  ```
- 哪些现象容易混淆：截图全黑 ≠ 渲染失败（可能是锁屏）；白屏也可能是端口冲突（WebSocket 2018 被占）。

## 3. 操作前检查
- [x] 已确认 CPU 型号（`lscpu`，是否国产 Zen 架构）
- [x] 已确认 Xorg 驱动与 GL 状态（`grep -iE 'GLX|DRI2|AIGLX' /var/log/Xorg.0.log`）
- [x] 已确认显卡当前 PCI 位置（`lspci`）
- [x] 已备份 xorg 配置与启动脚本

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 崩溃还是卡顿 | 启动程序观察 | — | 崩溃→步骤 5.1；卡顿→步骤 5.2 |
| 2 | X 是否有硬件 GL | Xorg 日志 AIGLX/DRI2 段 | DRI2 capable | 软件渲染 → 切 nouveau |
| 3 | qt.conf 指向 | `strace -f` 看插件加载路径 | 版本一致 | Prefix 指错 → 修正 |
| 4 | 基线量化 | `ps -eo pcpu` | 渲染进程 CPU 合理 | 单核满载 → 软件光栅瓶颈 |

## 5. 标准解决步骤
### 步骤 1：崩溃（LLVM ERROR）→ softpipe 兜底
根因：老 Mesa 的 llvmpipe（LLVM 3.5）与 Qt 5.12 SwiftShader（LLVM 3.9）都不认识 Zen 架构。
```sh
export GALLIUM_DRIVER=softpipe    # Mesa 纯 C 软件光栅器，无 LLVM，任何 CPU 可跑
```
配合：Qt 5.15.2 运行库（WebEngine 83）+ 修正 qt.conf（`Prefix = /opt/Qt5.15.2`）+ 补齐运行库（libxkbcommon、libdbus、libxcb-util、libxcb-xinerama，glibc 2.19 兼容构建放 `/opt/qt-runtime-lib/`）。

### 步骤 2：卡顿 → 硬件 GL + --disable-gpu
1. 切 nouveau 硬件驱动（GT 系列显卡）：
```
Section "Device"
    Identifier "NVIDIA-GT730"
    Driver "nouveau"
    BusID "PCI:5:0:0"    # lspci 实测，加装设备后会变
EndSection
```
同时**清空或移除** `/etc/X11/xorg.conf`（与 xorg.conf.d 并存会产生冲突段，历史上"改了没生效"的元凶）。
验证：`(II) AIGLX: Loaded and initialized nouveau` + `GLX: Initialized DRI2 GL provider`。
2. 启动脚本加 `--disable-gpu`：
```sh
export QTWEBENGINE_CHROMIUM_FLAGS="--no-sandbox --disable-gpu"
```
原理：Chromium 83 与老 Mesa 不兼容（offscreen FBO 失败反复重试白屏）→ 完全跳过 GPU 进程，改 Skia 多线程 CPU 光栅 + X 位图拷贝，16 核机器上 45 秒完整渲染 1080p 页面。
3. **保留** `GALLIUM_DRIVER=softpipe` 作为硬件 GL 失效时的兜底（避免 llvmpipe 崩溃）。

### 步骤 3：参数实测法（通用）
每次只改一个 `QTWEBENGINE_CHROMIUM_FLAGS` 参数并截图留证。实测结论：`--use-gl=desktop/egl` 无效；`--disable-gpu-compositing` 白屏 5~7 分钟；`--disable-gpu` 最优。

## 6. 验证标准
- [x] 程序主进程 + 4 个 WebEngine 子进程存活，无崩溃
- [x] 页面在可接受时间内完整渲染（本次基线：45 秒）
- [x] 渲染进程 CPU 从单核满载降至 ~1%
- [x] 重启后配置持久生效

## 7. 回滚方法
1. Xorg：恢复 `.bak-softpipe` 备份 + `systemctl restart lightdm`。
2. 启动脚本：恢复 `*.bak-softpipe` 脚本。

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| Qt 5.12 时代 `--disable-gpu` 也崩 | SwiftShader LLVM 3.9 不认 Zen | 换 Qt 5.15.2 运行库 |
| 硬件 GL 生效但白屏 | Chromium offscreen FBO 失败 | `--disable-gpu` |
| 卡顿但 CPU 不高 | 可能不是渲染问题 | 查 WebSocket 2018 端口冲突（两程序互斥） |
| 普通 Intel/AMD 机器 | 无 LLVM 兼容问题 | 删除 `GALLIUM_DRIVER=softpipe` |

## 9. 注意事项与禁止操作
- 重启 lightdm 前与现场确认（结束图形会话）。
- SSH 里 `pkill -f tn_frequency` 会杀掉远程 shell 自己，用 `pkill -x`。
- 截屏前先 `mate-screensaver-command --deactivate`（锁屏截出黑屏误判渲染失败）。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261030-Qt-监控程序渲染崩溃]]
- [[INC-20261030-Qt-图形界面卡顿高延迟]]

### 验证环境
- 系统/版本：凝思 V6.0.8 / 海光 C86-3350 / GT 730 / Qt 5.15.2
- 验证日期：2026-10-30
- 验证结论：通过（CPU 43.4% → ~1%）

### 参考资料
- 官方文档：Qt WebEngine Chromium 参数列表（QTWEBENGINE_CHROMIUM_FLAGS）

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库两篇案例抽象为决策树 | 知识库重建 |
