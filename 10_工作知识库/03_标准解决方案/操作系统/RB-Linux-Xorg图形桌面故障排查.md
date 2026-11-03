---
type: runbook
status: verified
created: 2026-10-29
updated: 2026-11-03
domain: linux
applicable_system: 凝思安全操作系统 V6.0.8（Xorg 1.16.4 + lightdm）
applicable_version: Xorg 1.16.x / lightdm / nouveau 或 modesetting
last_verified: 2026-10-30
risk: low
tags:
  - work/runbook
---

# Xorg 图形桌面故障排查（凝思 / Debian 系）

## 0. 快速结论
- **解决什么问题**：图形桌面无法启动、只能文本终端登录、lightdm 起不来。
- **适用场景**：修改 Xorg 配置后桌面起不来；加装 PCIe 设备后 BusID 失效；显示驱动/配置冲突。
- **核心处理思路**：以 `/var/log/Xorg.0.log` 的**首个 `(EE)`** 为准，一次只改一项，改完看日志。
- **是否经过实际验证**：是（2026-10-29 配置语法错误、2026-10-30 驱动切换均按此流程解决）。
- **预计影响与风险**：低；`systemctl restart lightdm` 会结束当前图形会话（需本机重新登录）。

## 1. 适用范围
- 适用操作系统/版本：凝思 V6.0.8（Xorg 1.16.4）；Debian 系 Xorg 通用。
- 不适用的情况：Wayland 会话；显示管理器非 lightdm 时替换对应服务名。

## 2. 故障特征
- 常见现象：开机停在 `linx login:` 文本界面；`systemctl restart display-manager` 无输出。
- 典型报错：`no screens found`、`No devices detected`、`".Identifier" is not a valid keyword`、`start-limit`。
- 哪些现象容易混淆：`systemctl restart` 无输出是正常现象（必须 status 确认）；`no screens found` 是最终失败提示，真正原因要看更早的日志。

## 3. 操作前检查
- [x] 已确认当前显卡 PCI 位置（`lspci`，加装设备后会变）
- [x] 已备份要改的配置文件
- [x] 已确认有 SSH/文本终端可用（重启 lightdm 会结束图形会话）

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 显示管理器状态 | `systemctl status lightdm --no-pager -l` | active (running) | failed → 看日志；start-limit → `reset-failed` |
| 2 | Xorg 首个错误 | `tail -n 100 /var/log/Xorg.0.log` | 无 (EE) 或已知项 | 按首个 (EE) 对症处理 |
| 3 | 显卡与驱动 | `lspci -nnk \| grep -A4 -Ei 'VGA\|3D'` | 驱动已绑定（nouveau/modesetting） | 驱动缺失则补驱动 |
| 4 | 配置冲突 | `ls /etc/X11/xorg.conf* /etc/X11/xorg.conf.d/` | 无并存冲突段 | xorg.conf 与 xorg.conf.d 二选一 |
| 5 | 驱动模块 | `ls /usr/lib/xorg/modules/drivers/` | 目标驱动 .so 存在 | 缺驱动先装包 |

## 5. 标准解决步骤
### 步骤 1：修正配置文件语法（常见根因）
```bash
cp -a /etc/X11/xorg.conf.d/20-xxx.conf{,.bak}
cat -n /etc/X11/xorg.conf.d/20-xxx.conf   # 逐行核对
cat -A /etc/X11/xorg.conf.d/20-xxx.conf   # 查异常字符
```
常见语法错误：`Identifier` 前多了点（`.Identifier`）、多条指令粘在一行、复制粘贴产生异常字符。建议用 heredoc 重建文件而不是手改。

### 步骤 2：重建并验证
```bash
systemctl reset-failed lightdm
systemctl restart lightdm
systemctl status lightdm --no-pager -l
tail -n 100 /var/log/Xorg.0.log
```

### 步骤 3：nouveau 硬件加速配置（GT 系列显卡）
```
Section "Device"
    Identifier "NVIDIA-GT730"
    Driver "nouveau"
    BusID "PCI:5:0:0"      # 以 lspci 实测为准！
EndSection
```
验证标志：`(II) AIGLX: Loaded and initialized nouveau` + `GLX: Initialized DRI2 GL provider`。

## 6. 验证标准
- [x] lightdm active (running)，显示器出现登录界面
- [x] Xorg 日志无新的 (EE)
- [x] 重启后仍然正常

## 7. 回滚方法
```bash
cp /etc/X11/xorg.conf.d/20-xxx.conf.bak /etc/X11/xorg.conf.d/20-xxx.conf
systemctl reset-failed lightdm && systemctl restart lightdm
```

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| 加装 PCIe 设备后桌面起不来 | 显卡 BusID 变了 | `lspci` 重查，更新 BusID |
| `No devices detected` | BusID 指向不存在的位置 | 同上 |
| xorg.conf 与 xorg.conf.d 并存 | 两份 Device/Screen 段冲突 | 清空/移除其一 |
| modesetting 无硬件 GL | 老驱动无 DRI2/glamor | 切 nouveau，见 [[RB-Qt-海光CPU图形渲染问题排查]] |

## 9. 注意事项与禁止操作
- 不要立即重装系统或装闭源 NVIDIA 驱动。
- 不要同时修改多个配置文件。
- 失败后以**新生成**的日志为准，不要沿用旧日志结论。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261029-Linx-图形桌面无法启动]]
- [[INC-20261030-Qt-图形界面卡顿高延迟]]

### 验证环境
- 系统/版本：凝思 V6.0.8，Xorg 1.16.4，GT 730
- 验证日期：2026-10-29 / 2026-10-30
- 验证结论：通过

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库两篇案例的排查步骤抽象整理 | 知识库重建 |
