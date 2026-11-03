---
type: runbook
status: verified
created: 2026-10-30
updated: 2026-11-03
domain: linux
applicable_system: 凝思安全操作系统 V6.0.8（序列号绑定 eth0 MAC）
applicable_version: 内核 4.9.0-8；systemd
last_verified: 2026-10-30（多次重启验证）
risk: medium
tags:
  - work/runbook
---

# 凝思开机网络自动恢复（initramfs 时序 + 兜底服务）

## 0. 快速结论
- **解决什么问题**：重启后网络不自动恢复 / 弹出"凝思产品授权管理"窗口要求输入序列号。
- **适用场景**：凝思工作站加装/更换网卡后；开机 ifup 被 linx_sec 拒绝（`Operation not permitted`）。
- **核心处理思路**：三层——① initramfs 预加载板载网卡驱动 + 拷贝 udev 命名规则（根治时序）；② udev 接口出现时自动 ifup；③ systemd 兜底服务重试。
- **是否经过实际验证**：是（修复后重启 ~1 分钟网络全自动恢复，校验开机即通过）。
- **预计影响与风险**：中——涉及 initramfs 重建与 grub，操作前备份 initrd/vmlinuz。

## 1. 适用范围
- 适用操作系统/版本：凝思 V6.0.8（序列号校验 eth0 MAC 的机制）；通用 Linux 网卡"迟到"问题可参考第 3 层。
- 不适用的情况：网卡硬件故障、纯物理链路问题。

## 2. 故障特征
- 常见现象：重启后 `ssh` 连不上；本机 `ip addr` 有地址但接口 DOWN、无默认路由；登录桌面弹授权窗口。
- 典型报错：`networking: RTNETLINK answers: Operation not permitted`、`Failed to bring up eth0.`、`IPwatchD is unable to work with device "eth0"`。
- 哪些现象容易混淆：`networking.service` 显示 SUCCESS ≠ 配置成功；弹授权窗是 checksn 校验失败，不是序列号失效。

## 3. 操作前检查
- [x] `get_linxsninfo -s <序列号>` 确认绑定 MAC 与 eth0 实际 MAC 一致
- [x] 已备份 /boot/initrd.img-*、vmlinuz-*、/etc/network/interfaces
- [x] 已确认 udev 持久命名规则内容（`70-persistent-net.rules`）

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 时序还原 | `dmesg \| grep -iE 'eth\|link'`、`grep networking /var/log/syslog` | 网卡注册早于 networking | 注册晚 → 走第 3 层兜底 |
| 2 | 序列号校验 | `cat /proc/cmdline`、`get_linxsninfo -s` | sn_text = eth0 MAC | 不一致 → 命名规则没生效 |
| 3 | 安全模块拦截 | syslog 中 ifup 报错 | 无 Operation not permitted | 有 → 修 initramfs 时序 |
| 4 | 兜底服务状态 | `systemctl status ensure-eth0` | active (exited) | 失败 → 看服务日志 |

## 5. 标准解决步骤
### 步骤 1：initramfs 时序根治（凝思序列号场景）
```bash
# 1. 板载网卡驱动最早加载
echo yt6801 >> /etc/initramfs-tools/modules

# 2. hook：把按 MAC 固定命名的 udev 规则拷进 initramfs
cat > /etc/initramfs-tools/hooks/copy-net-rules <<'EOF'
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case "$1" in prereqs) prereqs; exit 0;; esac
. /usr/share/initramfs-tools/hook-functions
cp /etc/udev/rules.d/70-persistent-net.rules "$DESTDIR/etc/udev/rules.d/" 2>/dev/null || true
exit 0
EOF
chmod +x /etc/initramfs-tools/hooks/copy-net-rules

# 3. 重建并验证
update-initramfs -u
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'yt6801|70-persistent'
```
> 凝思的 hook-functions 是精简版，没有 copy_file 函数，要用 `$DESTDIR` + `cp`。

### 步骤 2：udev 自动配网（可选，接口出现即 ifup）
```bash
cat > /etc/udev/rules.d/99-auto-ifup-eth0.rules <<'EOF'
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", RUN+="/bin/sh -c '(sleep 3; /sbin/ifup eth0 || (sleep 60; /sbin/ifup eth0)) &'"
EOF
```

### 步骤 3：systemd 兜底服务（通用，网卡"迟到"场景必配）
`/usr/local/sbin/ensure-eth0.sh`：
```bash
#!/bin/bash
IFACE=eth0
TAG=ensure-eth0
MAX_TRY=60          # 60 次 x 10 秒 = 最多 10 分钟

have_addr() { ip -4 addr show dev "$IFACE" 2>/dev/null | grep -q 'inet '; }
have_route() { ip route show default 2>/dev/null | grep -q "dev $IFACE"; }

for i in $(seq 1 $MAX_TRY); do
    if ip link show dev "$IFACE" >/dev/null 2>&1; then
        ip link set dev "$IFACE" up 2>/dev/null || true
        if ! have_addr || ! have_route; then
            /sbin/ifup "$IFACE" >/dev/null 2>&1 || true
        fi
        if have_addr && have_route; then
            logger -t "$TAG" "eth0 配置成功(第 $i 次检查,约 $((i*10)) 秒)"
            exit 0
        fi
    fi
    sleep 10
done
logger -t "$TAG" "警告:10 分钟内 eth0 仍未配置成功,请人工检查"
exit 1
```

`/etc/systemd/system/ensure-eth0.service`：
```ini
[Unit]
Description=Ensure eth0 is up and configured (retry until interface appears)
After=networking.service systemd-modules-load.service systemd-udevd.service
Wants=networking.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/ensure-eth0.sh
TimeoutStartSec=660

[Install]
WantedBy=multi-user.target
```
```bash
chmod +x /usr/local/sbin/ensure-eth0.sh
systemctl enable ensure-eth0
```

**关键点**：重试逻辑必须放 systemd 服务里（进程生命周期受管）。udev RUN 里的后台 shell 会被 udevd 超时杀掉；rc.local 的后台循环会被 oneshot 服务清理。

## 6. 验证标准
- [x] 重启后 ~1 分钟 `ip route` 有默认路由、`ip -4 addr` 有地址
- [x] 序列号校验开机即通过，不弹授权窗
- [x] `systemctl status ensure-eth0` 成功退出
- [x] 多次重启稳定

## 7. 回滚方法
1. initrd 回退：`cp /boot/initrd.img-*.bak-YYYYMMDD /boot/initrd.img-$(uname -r)`（备份按当日惯例命名）。
2. 兜底服务回退：`systemctl disable ensure-eth0`。

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| 只有网卡迟到、无序列号机制 | 通用 Linux | 只做步骤 3 |
| 序列号换绑新网卡 | 授权 MAC 变更 | 与售后确认绑定板载卡 MAC，再改 grub linx_serial + update-grub |
| 弹窗但网络正常 | checksn 误判 | 查 `check_old_sn.py` 返回与 eth0 MAC |

## 9. 注意事项与禁止操作
- 禁止自行修改 `/etc/default/grub` 中的 linx_serial（需凝思售后授权）。
- 不要反复输入序列号（不解决时序问题）。
- systemd 单元 `Documentation=` 字段不要写中文（被按 URL 解析报 Invalid URL）。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261030-Linx-序列号弹窗与重启断网]]
- [[INC-20261030-Linx-重启后网络不自动恢复]]

### 验证环境
- 系统/版本：凝思 V6.0.8，yt6801 + r8169 双网卡
- 验证日期：2026-10-30（两次重启验证）
- 验证结论：通过（~1 分钟自动恢复，不再弹窗）

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库两篇案例抽象为三层方案 | 知识库重建 |
