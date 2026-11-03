---
type: runbook
status: verified
created: 2026-11-03
updated: 2026-11-03
domain: deployment
applicable_system: Linux 程序集（Qt 等带大量符号链接的发行版）
applicable_version: tar.gz / zip 打包
last_verified: 2026-11-03
risk: low
tags:
  - work/runbook
---

# Linux 发布包符号链接自检与修复

## 0. 快速结论
- **解决什么问题**：发布包解压后程序报 `error while loading shared libraries: xxx.so: file too short`。
- **适用场景**：发布包在 Windows/NTFS/网盘环境转手后再部署到 Linux；Qt 发行版（libQt5Core.so 三层链接）尤其高发。
- **核心处理思路**：打包在 Linux 上做；落地 Windows 后自检 `find -type l` 数量；已损坏时按"同名最长非空文件"脚本重建软链。
- **是否经过实际验证**：是（2026-11-03 现场重建 116 个符号链接修复）。
- **预计影响与风险**：低——修复脚本只处理 0 字节 .so 文件。

## 1. 适用范围
- 适用软件/版本：Qt 5.x 运行库；任何含 .so 版本链接的程序集。
- 不适用的情况：纯 Windows 程序；单文件部署。

## 2. 故障特征
- 常见现象：`file too short`；`cannot open shared object file`。
- 可能原因：符号链接在 Windows 上被展开成 0 字节普通文件（不是断链，是空文件）。
- 哪些现象容易混淆：`file too short` ≈ 空文件——第一反应应查 `find -size 0`，**不要**去查 LD_LIBRARY_PATH、ldd、GLIBC 版本。

## 3. 操作前检查
- [x] 已确认发布包在哪一端打包（Windows 打包必出此问题）
- [x] 已备份发布包原件
- [x] 已确认目标机 Python 版本（凝思 6.0.80 自带 Python 3.4，不支持 f-string）

## 4. 标准排查流程
| 顺序 | 检查目标 | 操作/命令 | 正常结果 | 异常时下一步 |
|---|---|---|---|---|
| 1 | 直接跑 | 执行启动脚本 | 正常启动 | 报 file too short → 2 |
| 2 | 空文件分布 | `find <安装目录> -type f -size 0 \| sed 's\|/[^/]*$\|\|\|' \| sort \| uniq -c \| sort -rn` | 无（或仅 include） | 大量 .so → 3 |
| 3 | 链接残留 | `find <安装目录> -type l \| wc -l` | 一两百（Linux 原版 Qt） | 为 0 → 全丢，重建 |

## 5. 标准解决步骤
### 步骤 1：修复脚本（重建 0 字节 .so 的软链）
对每个 0 字节的 `libX.so[.N[.N]]`，在同目录找**名字最长且非空**的 `libX.so.Y.Z` 作目标重建软链：
```python
import os, re
ROOT = "/opt/tn-frequency-release"
pat = re.compile(r"^(?P<base>.+?)\.so(?P<ver>(\.\d+)*)$")

for dirpath, _, filenames in os.walk(ROOT):
    for fn in filenames:
        full = os.path.join(dirpath, fn)
        if os.path.islink(full) or os.path.getsize(full) != 0:
            continue
        m = pat.match(fn)
        if not m:
            continue
        base = m.group("base")
        cand = []
        for other in os.listdir(dirpath):
            om = pat.match(other)
            if not om or om.group("base") != base:
                continue
            op = os.path.join(dirpath, other)
            if os.path.islink(op) or not os.path.isfile(op) or os.path.getsize(op) == 0:
                continue
            cand.append((len(other), os.path.getsize(op), other))
        if not cand:
            print("SKIP %s" % full)
            continue
        cand.sort(key=lambda x: (-x[0], -x[1]))
        target = cand[0][2]
        os.remove(full)
        os.symlink(target, full)
```
> 老 Python（3.4）不支持 f-string，用 `%` 格式化，否则 SyntaxError。

### 步骤 2：验证
```bash
LD_LIBRARY_PATH=<qt-lib>:<qt-runtime> ldd <bin> | grep -c "not found"   # 应为 0
DISPLAY=:0 ./启动脚本
pgrep -a -x <程序名>；ss -tlnp | grep <端口>；截图确认界面
```

### 步骤 3：打包规范（预防）
- **打包必须在 Linux 上做**；Windows 上用 zip（原生 UTF-8 文件名）+ 英文名文件，且仍不保留链接。
- 打包后自检：`tar -tvzf pkg.tar.gz | grep -c "^l"` 应等于符号链接数量，为 0 说明包是坏的。
- 权限位同理：NTFS 不存 Unix 权限，`test -x` 在 Windows 上不可信，以 `tar -tvzf` 头信息为准。

## 6. 验证标准
- [x] `ldd` 无 not found
- [x] 程序启动、窗口正常
- [x] 功能三件套：进程在、端口监听、界面非白屏

## 7. 回滚方法
- 保留原 tar.gz；重打包即可回退。

## 8. 常见变体与特殊情况
| 情况 | 区别 | 应对方法 |
|---|---|---|
| SKIP 项（无真身） | 如 dri/swrast_dri.so | 系统自带同名文件则忽略（未设 LIBGL_DRIVERS_PATH 时目录不被使用） |
| `.so.*.new` 0 字节 | 升级残渣，无 soname | 不处理，加载器不会找 |
| 中文文件名乱码 | tar.gz UTF-8 被按 GBK 解 | 解压后手动 mv 回中文名；或改 zip/英文名 |

## 9. 注意事项与禁止操作
- 不要先删发布包再修复（原包是最后的回滚手段）。
- 不要在 Windows 上验证 Unix 可执行权限。

## 10. 来源与验证记录
### 来源案例
- [[INC-20261103-Linux-发布包解压后无法启动]]

### 验证环境
- 系统/版本：凝思 V6.0.8；tn-frequency-release-20261102（193MB）
- 验证日期：2026-11-03
- 验证结论：通过（116 个链接重建，程序正常）

## 11. 更新记录
| 日期 | 修改内容 | 修改原因 |
|---|---|---|
| 2026-11-03 | 从源库案例抽象整理 | 知识库重建 |
