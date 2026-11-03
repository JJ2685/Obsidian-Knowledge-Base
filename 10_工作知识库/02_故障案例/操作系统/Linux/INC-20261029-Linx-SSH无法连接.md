---
type: work_case
id: INC-20261029-002
status: resolved
created: 2026-10-29
updated: 2026-11-03
project: 电网频率响应系统
domain: linux
system: 凝思安全操作系统 V6.0.8
software: OpenSSH（老版本）
version: "4.9.0-8-linx-security-amd64"
severity: high
tags:
  - work/incident
---

# 凝思服务器 SSH 无法连接（无 IP + 算法协商失败）

## 0. 结论摘要
- **问题现象**：sshd 服务正常监听 22 端口，但 Windows 端 `ssh root@192.168.0.200` 连不上：先是不通，配好 IP 后又报 `no matching MAC found`。
- **直接影响**：无法远程运维，所有操作需到现场本机。
- **根本原因**：已确认——两层问题：① 网卡无 IP、无路由（网络层不可达）；② 服务器老 OpenSSH 只提供 hmac-sha1 等旧算法，Windows 11 OpenSSH 9.x 默认禁用。
- **最终处理**：配置静态 IP + 客户端 SSH 算法兼容参数 + 服务器 `PermitRootLogin yes`。
- **当前状态**：已解决。
- **可复用方案**：[[RB-Linux-凝思工作站SSH远程连接配置]]

## 1. 问题背景
- **所属项目/设备**：凝思工作站服务器（电网频率响应系统）。
- **问题发现时间**：2026-10-29。
- **是否可以稳定复现**：是（新装系统默认无网卡配置）。

## 2. 环境信息
| 项目 | 信息 |
|---|---|
| 操作系统及版本 | 凝思 V6.0.8（Debian 系，ifupdown 管理网络，无 NetworkManager） |
| 软件及版本 | OpenSSH 旧版（仅 hmac-sha1 / ssh-rsa 等旧算法） |
| 客户端 | Windows 11 OpenSSH 9.x |
| 网络/部署环境 | 办公室网段 192.168.0.0/24，网关 192.168.0.1 |

## 3. 现象与排查过程
| 顺序 | 排查假设 | 执行操作/命令 | 观察结果 | 判断 |
|---|---|---|---|---|
| 1 | sshd 未运行 | `systemctl status ssh`、`ss -tnlp \| grep :22` | enabled + 监听 `*:22` | 排除 |
| 2 | 网卡无 IP | `ip addr`、`ip route` | eth0 DOWN 无 IPv4，路由为空 | 支持 |
| 3 | 物理链路问题 | `ip link set eth0 up`、`ethtool eth0` | LOWER_UP，Link detected: yes | 排除 |
| 4 | 网段/DHCP 探测 | `dhclient -v eth0` | DHCPACK from 192.168.0.1，bound to .117 | 确认同网段 |
| 5 | 静态 IP 配置 | 写 `/etc/network/interfaces` 后 `ifup eth0` | 报 `option with empty value`（行尾残留空选项） | 修正后成功 |
| 6 | 算法协商失败 | `ssh root@192.168.0.200` | `no matching MAC found. Their offer: hmac-sha1` | 支持 |
| 7 | root 登录被拒 | `PermitRootLogin yes` 后重启 ssh | 连接成功 | 支持 |

## 4. 根本原因分析
- **根因 1**：凝思 V6.0.8 默认 `/etc/network/interfaces` 只有 lo，新装系统必须手工补网卡配置。
- **根因 2**：老 OpenSSH 与新客户端算法集无交集；`PermitRootLogin` 只决定"允不允许 root 登录"，**不解决算法协商**。

## 5. 实际解决过程
详见 [[RB-Linux-凝思工作站SSH远程连接配置]]。

关键点：
1. `/etc/network/interfaces` 追加 eth0 静态配置（`auto eth0` 必须有）。
2. Windows `~/.ssh/config` 加算法兼容块：`MACs +hmac-sha1`、`HostKeyAlgorithms +ssh-rsa`、`KexAlgorithms +diffie-hellman-group14-sha1`、`Ciphers +aes128-cbc,aes256-cbc`。
3. 服务器 `PermitRootLogin yes`，`systemctl restart ssh`。

## 6. 验证结果
- [x] `ssh root@192.168.0.200` 可登录
- [x] 重启后配置持久化（后续案例曾暴露重启断网问题，见 [[INC-20261030-Linx-序列号弹窗与重启断网]]）
- [ ] 未做 SSH 加固（root+密码直连，安全偏低，留待后续）

## 7. 经验总结
- **这次最关键的判断依据**：排查顺序永远是"先网络、再服务、最后认证"——服务正常 ≠ 可以远程。
- **下次优先检查什么**：`ip -4 addr` + `ip route` 是否为空；`ethtool` 链路状态。
- **不建议重复尝试的操作**：在网卡无 IP 时反复改 sshd 配置。

## 8. 关联内容
- 所属项目：[[00_项目总览]]
- 可复用方案：[[RB-Linux-凝思工作站SSH远程连接配置]]
- 相关案例：[[INC-20261030-Linx-序列号弹窗与重启断网]]（加装 PCIe 网卡后接口改名与 IP 冲突）
