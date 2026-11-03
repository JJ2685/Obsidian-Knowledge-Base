---
type: runbook
status: verified
created: 2026-10-30
updated: 2026-11-03
domain: linux
applicable_system: 凝思安全操作系统 V6.0.8（Debian 系，ifupdown）
applicable_version: 内核 4.9.0-8；客户端 Windows 10/11 OpenSSH
last_verified: 2026-11-02
risk: low
tags:
  - work/runbook
---

# 凝思工作站 SSH 远程连接配置

## 0. 快速结论
- **解决什么问题**：让办公电脑通过 SSH 远程登录凝思 V6.0.8 工作站（新装系统默认无网卡配置）。
- **适用场景**：服务器首次开通远程、重装后恢复、换客户端电脑。
- **核心处理思路**：**先网络，再服务，最后认证**——静态 IP → sshd 服务 → root 登录许可 → 客户端算法兼容。
- **是否经过实际验证**：是（2026-10-29 首配，多次复用）。
- **预计影响与风险**：低；改 sshd 配置前先 `sshd -t` 语法检查。

## 1. 适用范围
- 适用操作系统/版本：凝思 V6.0.8（网络由 ifupdown 管理，无 NetworkManager）；Debian 系可参考。
- 不适用的情况：NetworkManager 管理的系统（用 nmtui/nmcli，不要改 interfaces）。

## 2. 故障特征
- 常见现象：`ssh` 超时 / `Connection refused` / `no matching MAC found` / `Permission denied`。
- 典型报错：`Unable to negotiate with 192.168.0.250 port 22: no matching MAC found. Their offer: hmac-sha1`。
- 哪些现象容易混淆：sshd 监听正常 ≠ 可以远程（网卡没 IP 照样连不上）；`PermitRootLogin` 不解决算法协商。

## 3. 操作前检查
- [x] 已确认客户端与服务器同一网段
- [x] 已备份 `/etc/network/interfaces` 与 `/etc/ssh/sshd_config`
- [x] 已确认服务器有本机终端可用（远程失联时的恢复通道）

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 物理链路 | `ip link`、`ethtool eth0 \| grep -E 'Link detected\|Speed'` | LOWER_UP + Link detected: yes | 查网线/交换机端口/VLAN |
| 2 | 地址与路由 | `ip -4 addr`、`ip route` | 有静态 IP 和默认路由 | 写 interfaces 后 `ifup eth0` |
| 3 | 地址管理方式 | `cat /etc/network/interfaces`、`systemctl is-active NetworkManager` | ifupdown（NM 返回 unknown） | NM active 则用 nmcli |
| 4 | sshd 服务 | `systemctl status ssh`、`ss -tnlp \| grep :22`、`systemctl is-enabled ssh` | active + 监听 *:22 + enabled | start/enable；未装则 `apt install openssh-server` |
| 5 | root 登录 | `grep -i '^PermitRootLogin' /etc/ssh/sshd_config` | yes 或 prohibit-password | 修改后 `sshd -t && systemctl restart ssh` |
| 6 | 防火墙 | `iptables -L INPUT -n --line-numbers` | 放行 22 | `iptables -I INPUT -p tcp --dport 22 -j ACCEPT` |
| 7 | 客户端算法 | 连接报错信息 | 无算法类报错 | 追加旧算法参数（见下） |

## 5. 标准解决步骤
### 步骤 1：配置静态 IP（ifupdown）
```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
vi /etc/network/interfaces
```
追加（接口名以 `ip link` 实际输出为准）：
```
auto eth0
iface eth0 inet static
    address 192.168.0.250
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 114.114.114.114 180.76.76.76
```
- 分配地址前先冲突检测：`ping -c 3 <候选IP>` + `arp -a | grep <候选IP>`。
- `auto eth0` 必须有，否则开机不自动配置；选项后必须有值（否则 ifup 报 `option with empty value`）。
- 应用：`ifup eth0 && ip -4 addr show eth0 && ping -c 3 192.168.0.1`。

### 步骤 2：Windows 客户端算法兼容（凝思老 OpenSSH 必需）
`C:\Users\<用户名>\.ssh\config` 追加：
```
Host linx 192.168.0.250
  HostName 192.168.0.250
  Port 22
  User root
  IdentityFile ~/.ssh/id_ed25519
  MACs +hmac-sha1
  HostKeyAlgorithms +ssh-rsa
  PubkeyAcceptedAlgorithms +ssh-rsa
  KexAlgorithms +diffie-hellman-group14-sha1
  Ciphers +aes128-cbc,aes256-cbc
```
`+` 表示在默认算法之外追加，不影响连接其他服务器。临时测试可 `ssh -o MACs=+hmac-sha1 root@<IP>`。

### 步骤 3：服务器放行 root 登录
```
PermitRootLogin yes
```
```bash
sshd -t && systemctl restart ssh
```

## 6. 验证标准
- [x] `ssh linx` 可直接登录
- [x] `reboot` 后配置持久（auto eth0 存在、ssh enabled）
- [x] 网关 ping 通 0% 丢包

## 7. 回滚方法
1. `cp /etc/network/interfaces.bak /etc/network/interfaces`，`ifdown eth0; ifup eth0`。
2. sshd_config 备份恢复 + `systemctl restart ssh`。

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| 加装 PCIe 网卡后接口改名 | eth0/eth1 名字互换 | 以 `ip link` 为准；序列号绑定 eth0 MAC，见 [[RB-Linux-凝思开机网络自动恢复]] |
| IP 被占用 | ping 通但连不上 | `arp -a` 比对 MAC，换地址 |
| 免密登录 | 需先部署公钥 | `type %USERPROFILE%\.ssh\id_ed25519.pub \| ssh root@<IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"` |

## 9. 注意事项与禁止操作
- 不要把 22 端口映射公网；只在内网/VPN 使用。
- 不要远程执行 `systemctl restart lightdm`（结束现场图形会话）。
- 不要用 `pkill -f tn_frequency`（杀掉远程 shell 自己），用 `pkill -x tn_frequency`。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261029-Linx-SSH无法连接]]

### 验证环境
- 系统/版本：凝思 V6.0.8 / Windows 11 OpenSSH 9.x
- 验证日期：2026-10-29 首配，2026-11-02 复用于新会话
- 验证结论：通过

### 参考资料
- 项目部署文档：[[02_环境与部署]]

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库操作指南迁入并整理 | 知识库重建 |
