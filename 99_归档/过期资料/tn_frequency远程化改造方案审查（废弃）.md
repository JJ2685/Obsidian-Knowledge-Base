# tn_frequency 远程化改造方案审查结论

> 审查对象：`tn_frequency_change_qt5.12` Qt GUI 工程及《tn_frequency 快速频率响应系统——软件结构分析与远程化改造方案》。
>
> 审查范围说明：当前已检查 GUI 工程与方案文档；未获得 `Reqlian_frequency-fan-v2.1` 后台服务源码及完整运行配置。因此，后台 UDP 行为、录波文件读写行为仍需在实际环境中验证，不能视为已经确认。

## 一、总体结论

**整体远程化思路可行，但不建议直接按原方案实施。**

建议以 `tn_frequency_change_qt5.12` 作为 GUI 改造基线，保留现有 Qt、QWebEngine、HTML 前端及 QWebChannel 结构，优先改造 UDP 网络配置和文件访问方式，不进行大规模架构重构。

当前方案的关键风险不在 GUI 是否能够配置远程 IP，而在于后台服务是否真正支持跨机器 UDP 通信，以及 GUI 和后台之间是否存在共享文件目录依赖。

## 二、审查结论汇总

| 审查项目 | 结论 | 优先级 |
|---|---|---|
| 使用 Qt 5.12 版作为改造基线 | 合理，建议采用 | 低 |
| GUI 通过 UDP 向远程服务器发送控制命令 | 方向正确，需核验后台监听地址及协议 | 高 |
| GUI 监听 UDP 11002 接收实时数据 | 技术上可行，需确认后台发送目标 | 高 |
| 后台服务向工作站发送实时数据 | 尚未从后台源码或抓包证实 | **最高** |
| WebSocket 2018 保持 localhost | 当前架构下合理，不需要跨机 | 低 |
| CSV 通过 NFS/SMB 共享 | 可行，但不应不加区分地共享所有目录 | 高 |
| `config.ini` 配置加载 | 有当前工作目录依赖及校验不足问题 | 高 |
| UDP 接收与错误处理 | 建议完善 | 中 |
| 部署与验收方案 | 需要补充断网、目录失效、服务重启等测试 | 高 |

## 三、主要问题与修改建议

### 1. 必须先确认后台 UDP 实时数据发送机制

GUI 当前具备配置 UDP 发送目标和监听地址/端口的能力，但这并不能证明后台服务已经支持远程化。

需要确认：

1. 后台服务是否监听 UDP `11001`，监听地址是 `127.0.0.1`、指定网卡 IP，还是 `0.0.0.0`。
2. 后台实时数据发往哪里：是否根据收到的 GUI 数据报源 IP/端口回传，还是固定发送到 `127.0.0.1:11002` 或其他配置地址。
3. 后台是否需要先收到 GUI 的特定命令，才开始发送实时数据。
4. 控制命令的数据格式、响应格式及执行确认机制。

**验证方法：**先在原先两程序同机运行的环境抓包：

```bash
sudo tcpdump -ni lo -vv -X 'udp port 11001 or udp port 11002'
```

分别操作实时监测和投运开关，记录数据报的源 IP/端口、目的 IP/端口及触发顺序。两机部署测试时再抓取实际网卡流量：

```bash
sudo tcpdump -ni any -vv -X 'udp port 11001 or udp port 11002'
```

如果后台固定向 `127.0.0.1:11002` 发送，则必须修改后台目标地址或提供其可用配置方式；仅修改 GUI 不够。

### 2. `config.ini` 和前端 HTML 路径依赖当前工作目录

当前配置文件路径使用：

```cpp
QString AppConfig::ConfigFile = "config.ini";
```

前端页面路径使用：

```cpp
QString url = QDir::currentPath()+"/../html/index.html";
```

这两处都依赖进程启动时的当前工作目录。通过桌面快捷方式、启动脚本或开机启动运行时，当前目录可能与预期不同，导致配置或 HTML 找不到。

**建议：**统一使用可执行文件所在目录定位程序资源，例如：

```cpp
#include <QCoreApplication>
#include <QDir>

QString appDir = QCoreApplication::applicationDirPath();
QString configPath = QDir(appDir).filePath("config.ini");
QString htmlPath = QDir(appDir).filePath("../html/index.html");
```

实际 HTML 路径应根据最终部署目录结构确定。

同时完善配置校验：

- `UdpServerIP` 必须是有效的远程服务器地址。
- `UdpServerPort`、`UdpListenPort` 必须在 `1~65535` 范围内。
- `UdpListenIP` 必须是有效的本地监听地址。
- 检查必填项是否缺失、端口是否转换失败。
- 关键网络配置错误时，明确报错并阻止进入运行状态，不要静默回退到 `127.0.0.1`。

### 3. 文件目录不应全部直接远程挂载

当前 GUI 涉及多个本地目录，但它们的生产者、消费者和访问方式不同，不能统一处理。

| 目录 | GUI 用途 | 建议处理方式 |
|---|---|---|
| `/DataOut/` | 遍历、读取录波 CSV | 第一阶段考虑 NFS 只读挂载 |
| `/LoBoTest/` | 手动触发时由 GUI 写入 CSV | 先确认后台是否读取；若需要，单独建立可写共享目录 |
| `/opt/Template/` | 模板保存和读取 | 暂时保留工作站本地 |
| `/PfreqCnf/config` | 读取装机容量、门槛、调差率等业务配置 | 默认保留本地，除非确认需要集中共享 |
| `/opt/logo/tn_frequency.logo` | GUI 日志相关路径 | 暂时保留工作站本地 |

特别注意：GUI 的 `case 110` 会调用 `WritTriggerFile()`，并直接向 `TIG_FILE_DIR` 创建日期 CSV 文件。这是文件 I/O 行为，不等同于 UDP 命令。

因此，若后台服务依赖 `/LoBoTest/` 中的文件，必须保证 GUI 写入的目录与后台读取的目录实际指向同一存储位置，并测试权限、文件完成状态和并发访问。

建议新增配置项时使用清晰名称，例如 `PfreqConfigPath`，避免与 GUI 的 `AppConfig::ConfigFile` 混淆。

### 4. UDP 接收和错误处理需要完善

当前接收逻辑只读取一次数据报，建议改为循环处理所有待处理数据报，并取得发送端地址用于来源校验。

示例：

```cpp
void MainWindow::udp_read_data()
{
    while (UdpSocket->hasPendingDatagrams())
    {
        QHostAddress sender;
        quint16 senderPort = 0;

        QByteArray data;
        data.resize(UdpSocket->pendingDatagramSize());

        qint64 len = UdpSocket->readDatagram(
            data.data(), data.size(), &sender, &senderPort);

        if (len < 0)
        {
            qWarning() << "UDP receive error:"
                       << UdpSocket->errorString();
            continue;
        }

        // 应按实际部署配置校验发送端 IP，不能盲目假设源端口。
        WebEngieFun.QtSendReal(QString::fromUtf8(data));
    }
}
```

UDP 绑定时必须检查返回值：

```cpp
if (!UdpSocket->bind(
        QHostAddress(AppConfig::UdpListenIP),
        AppConfig::UdpListenPort))
{
    qCritical() << "UDP bind failed:"
                << UdpSocket->errorString();
    return;
}
```

发送时也应检查 `writeDatagram()` 返回值并记录错误。需要明确：UDP 本地发送成功不等于远端收到，更不等于控制动作已经执行。若后台可修改，建议增加命令确认、超时和执行结果反馈；若后台二进制不能修改，则应在验收中明确这一限制。

### 5. 远程文件访问可能影响性能和稳定性

GUI 每 3 秒触发一次目录检查，`getUnreadFile()` 会递归遍历 `/DataOut/` 并检查文件状态。改为 NFS 后，目录元数据访问也会变成网络操作，文件数量较多时可能拖慢界面。

此外，`getNewFile()` 在目录为空时访问最后一个元素，可能造成越界问题。通过 `lastModified() == lastRead()` 判断文件是否已经读取，也不适合作为可靠的跨机器文件状态机制，可能受 Linux atime 策略和 NFS 属性缓存影响。

建议：

- 检查挂载点是否真实挂载，避免挂载失败后误读本地空目录。
- 对目录不存在、目录为空、文件打开失败增加异常处理。
- 确保只读取后台已经写入完成的 CSV 文件。
- 优化为增量扫描，避免每 3 秒递归遍历所有历史文件。
- 网络文件系统异常时，GUI 应给出明确提示，不应卡死或崩溃。

### 6. 网络安全和编译部署注意事项

**网络安全：**

- 优先绑定工作站实际业务网卡 IP；只有确有需要时才使用 `0.0.0.0`。
- 工作站防火墙仅允许远程数据服务器访问 UDP `11002`。
- 服务器仅允许指定工作站访问 UDP `11001`。
- 使用专用工业网络或 VPN，不要将控制 UDP 端口直接暴露到公网。
- 若后台协议没有认证和加密，应通过网络隔离、访问控制降低风险。

**编译部署：**

上传工程中存在不同平台/版本生成的构建文件：Linux 可执行文件显示 Qt 5.12.7 相关编译信息，而工程中的部分 Makefile 来自 Qt 6.4.3、Windows MSVC 环境。重新编译时不要直接复用旧 Makefile。

建议在目标 Linux 环境或兼容的构建环境中，使用 Qt 5.12.7 对应的 GCC Kit 重新运行 qmake 并编译，最终在凝思 V6.0.80 实机验证 Qt 动态库、WebEngine、网络库及运行环境。

## 四、推荐的目标架构

保留两个程序的职责划分：

**工作站（GUI）：**

- `tn_frequency` 继续运行 Qt GUI 和内嵌 HTML 前端。
- QWebChannel WebSocket 保持 `localhost:2018`，不改为远程通信。
- UDP 向远程数据服务器发送控制命令。
- UDP 在工作站业务网卡接收后台实时数据。
- 在本地读取录波文件或读取远程同步/挂载的录波目录。
- 模板、业务配置等非必要共享目录先保留本地。

**远程服务器（数据端）：**

- 运行 `Reqlian_frequency`。
- 采集现场频率数据。
- 监听 GUI 控制命令。
- 向工作站发送实时数据。
- 将录波 CSV 写入服务器本地目录。

**文件通道：**

- 第一阶段优先尝试将远程 `/DataOut` 以只读方式挂载到工作站。
- 只有确认后台依赖 `/LoBoTest` 后，才建立对应的可写共享目录。
- 如果 NFS 导致 GUI 卡顿或网络中断时不稳定，再考虑 SFTP/rsync 同步到工作站本地缓存。

## 五、建议的配置文件示例

以下仅为示意，IP 和路径必须替换为实际环境值：

```ini
[UdpConfig]
UdpServerIP=192.168.1.100
UdpServerPort=11001
UdpListenIP=192.168.1.20
UdpListenPort=11002

[PathConfig]
FileDir=/mnt/frequency/DataOut/
TigFileDir=/mnt/frequency/LoBoTest/
TempFileDir=/opt/Template/
PfreqConfigPath=/PfreqCnf/config
LogoFilePath=/opt/logo/tn_frequency.logo
```

`UdpListenIP` 示例使用工作站实际业务网卡 IP。若采用 `0.0.0.0`，必须同时设置严格的防火墙来源限制。

## 六、推荐实施顺序

### 阶段一：确认后台真实行为（不改代码）

1. 在原同机环境抓取 UDP 11001、11002 数据包。
2. 确认后台监听地址、实时数据发送目标及触发时机。
3. 确认控制命令格式、响应情况及执行结果如何判断。
4. 查清 `/DataOut`、`/LoBoTest` 的后台读写关系。

### 阶段二：改造 GUI 网络配置

1. 以 Qt 5.12 版为基线。
2. 修正 `config.ini` 和 HTML 路径定位方式。
3. 增加配置项完整性、IP 和端口校验。
4. 完善 UDP 绑定、循环接收、发送错误处理和日志。
5. 暂时不动前端 WebSocket 和 CSV 目录结构。
6. 在原同机环境回归，确认原功能不受影响。

### 阶段三：完成跨机器 UDP 测试

1. 将后台部署到远程服务器。
2. 配置 GUI 的远程服务器 IP。
3. 按后台实际能力配置实时数据发送目标为工作站 IP/11002。
4. 双端抓包，确认数据报真正跨机器传输。
5. 验证实时曲线、控制命令及后台响应。

### 阶段四：处理录波文件访问

1. 建立 `/DataOut` 只读挂载。
2. 测试全部文件、按日期、按时间、指定文件和最新文件查询。
3. 单独测试手动触发功能及 `/LoBoTest` 的读写关系。
4. 检查挂载失效、空目录、文件写入中等异常情况。

### 阶段五：异常、安全和稳定性验收

至少测试：

- 远程服务器断网及恢复。
- 后台进程停止、重启。
- UDP 端口被占用或绑定失败。
- 配置文件缺失、错误 IP、非法端口。
- NFS 挂载失败、目录为空、文件权限不足。
- CSV 尚未写入完成时 GUI 查询。
- 非授权 IP 发送 UDP 数据。
- GUI 重启及工作站网卡地址变化。

## 七、最终建议

1. **不推翻现有架构。** 保留 Qt GUI、HTML、QWebChannel 和 UDP 的总体设计。
2. **先查后台，再改 GUI。** 后台实时数据目的地址是当前最大的未知点。
3. **网络配置和文件访问分开改造。** 先让双机 UDP 通信稳定，再解决 CSV 访问。
4. **文件目录按实际读写关系共享。** 不建议把所有目录一次性挂载。
5. **优先实现可诊断性。** 对配置错误、绑定失败、网络中断、目录不可用等情况增加日志和提示。

**当前结论：技术路线可行，但尚不具备直接上线条件。** 在后台 UDP 行为、`/LoBoTest` 依赖和部署网络地址确认之前，不应认定只修改 GUI 配置即可完成远程化。

## 八、待补充材料

若要完成最终定案，还需要检查以下后台材料：

- `Reqlian_frequency` 后台可执行文件及其版本信息。
- `install.sh` 安装脚本。
- 后台配置文件、启动脚本及服务管理方式。
- 如有条件，提供后台源码或同机运行时 UDP 抓包结果。

这些材料用于确认后台监听地址、实时数据发送目标、控制命令处理方式，以及 `/DataOut`、`/LoBoTest` 的实际文件依赖。
