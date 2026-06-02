---
name: dsa3217
description: DSA3217 压力扫描阀设备驱动开发指南。涵盖 TCP/Telnet 文本协议、SET/LIST/SCAN 命令体系、二进制帧解析、单位系统、扫描配置和生命周期管理。Use when writing or modifying DSA3217 driver code, debugging pressure scanner data streams, adding DSA3217 features, or when user mentions DSA3217, DSA 3217, 压力扫描阀, 16通道压力扫描, or scan valve.
---

# DSA3217 压力扫描阀设备驱动开发

## 设备概述

DSA3217 是一款基于 TCP/IP（Telnet）的 16 通道压力扫描阀设备：

| 通道 | 内容 | 默认精度 | 默认量程 |
|------|------|----------|----------|
| CH1~CH16 | 压力值 | 3 位小数 | [-10, 10] |

出厂默认地址 `192.168.3.7:23`，文本协议（命令以 `\r\n` 结尾）。

## 快速开始：生命周期

```
// 1. 构造驱动（需传入 DeviceConfig + ChannelConfig[]）
const driver = new DSA3217Device(deviceConfig, channels)

// 2. 注册数据回调
driver.onData((payload) => {
  // payload: { deviceId, timestamp, channels[], channelIndices[] }
})

// 3. 连接 → 建立 TCP socket，启用 keep-alive
const ok = await driver.connect()

// 4. （可选）读取/配置扫描参数
const config = await driver.readScanConfig()   // 读取当前配置
await driver.setPressureUnit('kPa')            // 设置压力单位
await driver.setAvg(32)                        // 设置平均次数
await driver.setPeriod(500)                    // 设置扫描周期

// 5. 启动采集 → 发送 SCAN 命令，设备开始推送数据流
const started = await driver.startAcquisition()

// 6. 停止采集 → 发送 STOP 命令
await driver.stopAcquisition()

// 7. 断开
await driver.disconnect()
```

## 核心工作流

### 连接流程

1. 创建 TCP socket，连接 `host:port`（默认 `192.168.3.7:23`），超时 5s
2. 注册 `socket.on('data')` 监听
3. 连接成功后启用 TCP keep-alive（间隔 10s）
4. 无需发送初始化命令（与 DAQ-P-1604 不同，DSA3217 连接即可用）

### 采集流程

```
步骤1: 连接设备
  └── TCP socket.connect(port, host)

步骤2: （可选）配置扫描参数
  ├── SET UNITSCAN <单位>    → 设置压力单位
  ├── SET AVG <值>           → 设置平均次数
  └── SET PERIOD <值>        → 设置扫描周期

步骤3: 启动扫描
  └── SCAN                   → 设备开始推送数据帧

步骤4: 接收数据（自动解析）
  ├── 优先检测二进制帧（含 0x00 字节）
  │   ├── 识别帧头 packetType → 确定帧大小
  │   ├── 解析 16 通道 float/int16 值
  │   └── 发射 DataPayload
  └── 回退文本行解析
      ├── 按 \n 分割
      ├── 解析空格分隔的浮点数
      └── 发射 DataPayload

步骤5: 停止扫描
  └── STOP                   → 设备停止推送数据帧
```

### 断连处理

- Socket `close` 事件触发时自动清理状态
- 重置 `scanning`、`lineBuffer`、`binaryBuffer`
- 标记 `connected = false`
- 触发 `handleDisconnect()` 回调（由基类提供重连逻辑）

## 命令速查

### 配置命令（SET 系列）

| 命令 | 功能 | 参数范围 | 示例 |
|------|------|----------|------|
| `SET UNITSCAN <单位>` | 设置压力单位 | PSI/PA/KPA/MPA/KGF/CM2 | `SET UNITSCAN KPA` |
| `SET AVG <值>` | 设置平均次数 | 1 ~ 240 | `SET AVG 32` |
| `SET PERIOD <值>` | 设置扫描周期（μs） | 73 ~ 65535 | `SET PERIOD 500` |
| `SET FPS <值>` | 设置帧率 | 0+ | `SET FPS 5` |
| `SET CVTUNIT <系数>` | 设置单位转换系数 | 浮点数 | `SET CVTUNIT 1.000000` |
| `SET BIN <值>` | 设置二进制输出模式 | 0=文本, 1=二进制 | `SET BIN 1` |

### 查询命令（LIST 系列）

| 命令 | 功能 | 响应格式 |
|------|------|----------|
| `LIST` | 查询全部配置 | 多行文本 |
| `LIST S` | 查询扫描相关配置 | 多行 `SET xxx` 文本 |
| `LIST EU` | 查询工程单位 | 单位字符串 |

### 直接查询命令

| 命令 | 功能 | 响应格式 |
|------|------|----------|
| `EU?` | 查询当前单位 | 单位字符串 |

### 控制命令

| 命令 | 功能 |
|------|------|
| `SCAN` | 启动连续扫描数据流 |
| `STOP` | 停止扫描数据流 |
| `SAVE` | 保存当前配置到设备非易失存储 |

### 命令协议细节

- 所有命令以 `\r\n` 结尾，使用 ASCII 编码发送
- SET 命令为"发送即忘"模式（writeCommand），不等待响应
- LIST 命令为"请求-响应"模式（sendMultilineCommand），需等待多行响应
- 多行响应使用"稳定定时器"判定结束：收到数据后 150ms 无新数据则视为响应完成
- 单行查询使用"换行判定"：收到 `\n` 即视为响应完成
- 命令超时默认 2000ms

## 单位系统

### DSA 命令用单位字符串

| PressureUnit | DSA 命令字符串 |
|--------------|---------------|
| psi | PSI |
| Pa | PA |
| kPa | KPA |
| MPa | MPA |
| kgf/cm² | KGF/CM2 |

### 单位别名映射

设备可能返回多种格式的单位字符串，驱动需做归一化处理：

```
输入别名 → 归一化结果
PSI / PSIA / PSIG → psi
PA / PASCAL / PASCALS → Pa
KPA → kPa
MPA → MPa
KGF_CM2 / KGFCM2 / KGF/CM2 / KGF/CM^2 → kgf/cm2
```

### 单位系数映射（用于数值系数识别）

| 单位 | 系数 |
|------|------|
| psi | 1.0 |
| Pa | 6894.757 |
| kPa | 6.894757 |
| MPa | 0.006894757 |
| kgf/cm² | 0.070307 |

当设备返回纯数值时，通过匹配系数（容差 1e-3）反查单位。

### 读取单位的多级回退策略

```
1. 尝试 LIST S → 解析 "SET UNITSCAN <单位>" 行
2. 失败 → 尝试 LIST EU → 解析单位字符串
3. 失败 → 尝试 EU? → 解析单位字符串
4. 失败 → 尝试 LIST → 解析完整配置中的单位信息
5. 全部失败 → 返回 null
```

## 二进制帧解析

### 帧类型与大小

| packetType (UInt16LE) | 帧大小 (字节) | 数据格式 |
|----------------------|-------------|----------|
| 0x04 | 72 | Raw（Int16LE × 16） |
| 0x05 | 104 | EU（FloatLE × 16） |
| 0x06 | 80 | Raw + 时间戳（Int16LE × 16） |
| 0x07 | 112 | EU + 时间戳（FloatLE × 16） |
| 0x09 | 136 | EU + 扩展（FloatLE × 16） |

### 帧结构

```
EU 帧（packetType = 0x05 或 0x07）:
| 偏移 0-1  | 偏移 2-7  | 偏移 8-71 (16×float32 LE) | 偏移 72+ (可选时间戳等) |
| packetType | 序号/保留 | CH1~CH16 压力值            | 扩展数据                |

Raw 帧（packetType = 0x04 或 0x06）:
| 偏移 0-1  | 偏移 2-7  | 偏移 8-39 (16×int16 LE) | 偏移 40+ (可选时间戳等) |
| packetType | 序号/保留 | CH1~CH16 原始ADC值       | 扩展数据                |
```

### 二进制检测与帧同步

1. **检测条件**：数据块包含 `0x00` 字节，或正在扫描模式中
2. **帧同步**：读取前 2 字节为 `packetType`，查表得到帧大小
3. **帧大小不足**：缓存数据，等待后续数据到达
4. **无效 packetType**：从偏移 1 开始扫描，寻找下一个有效帧头
5. **帧完整**：按帧大小截取，解析通道数据，剩余数据继续处理

### 数据输出

解析后的 16 通道值经过通道过滤：
- 只输出 `enabled === true` 的通道
- 输出格式为 `DataPayload: { deviceId, timestamp, channels[], channelIndices[] }`

## 扫描配置

### DSA3217ScanConfig 结构

```
interface DSA3217ScanConfig {
  avg: number     // 平均次数，1~240，默认 32
  period: number  // 扫描周期（μs），73~65535，默认 500
  fps: number     // 帧率（Hz），根据 avg/period 自动换算，默认 0
  unit: string    // 压力单位，默认 'psi'
}
```

### 参数约束

| 参数 | 最小值 | 最大值 | 说明 |
|------|--------|--------|------|
| AVG | 1 | 240 | 平均次数，越大数据越稳定但帧率越低 |
| PERIOD | 73 | 65535 | 扫描周期（μs），影响采样频率 |

### 读取配置

```
发送: LIST S
响应示例:
  SET PERIOD 625
  SET AVG 8
  SET FPS 5
  SET UNITSCAN PSI
  SET CVTUNIT 1.000000

解析: 逐行匹配 "SET <KEY> <VALUE>" 模式
```

## 通道热更新

运行期间调用 `driver.updateChannels(newChannels)` 更新通道配置，无需重连。下一帧数据将使用新的通道过滤规则。

## 通信架构要点

### 命令发送模式

驱动内部维护一个 `dataHandler` 槽位，同一时刻只能有一个命令在等待响应：

```
// 写命令（SET/SCAN/STOP/SAVE）— 不等待响应
writeCommand(command)
  → socket.write(command + '\r\n')
  → 立即返回

// 单行查询（EU?）— 等待第一行响应
sendCommand(command, timeoutMs)
  → 注册临时 dataHandler
  → socket.write(command + '\r\n')
  → 等待 \n 出现或超时
  → 返回第一行文本

// 多行查询（LIST/LIST S）— 等待完整响应
sendMultilineCommand(command, timeoutMs)
  → 注册临时 dataHandler
  → socket.write(command + '\r\n')
  → 收集数据，150ms 无新数据视为结束
  → 超时兜底
  → 返回完整文本
```

### 数据接收分流

```
socket.on('data', chunk) {
  if (正在采集 && 正在扫描) {
    if (尝试二进制解析(chunk)) return  // 二进制帧优先
  }
  // 回退到文本行解析
  lineBuffer += chunk.toString('ascii')
  按 \n 分割 → 逐行解析浮点数 → 发射 DataPayload
}
```

### 并发安全

- **命令互斥**：`dataHandler` 槽位确保同一时刻只有一个命令在等待响应
- **采集期间避免查询**：SCAN 模式下数据流持续推送，发送查询命令会导致响应与数据流混杂
- **如需查询**：先 STOP 停止扫描，执行查询，再重新 SCAN

## 开发注意事项

| 条目 | 说明 |
|------|------|
| 连接后无需初始化命令 | 与 DAQ-P-1604 不同，DSA3217 连接即可用 |
| 文本协议编码 | 所有命令和数据使用 ASCII 编码 |
| 命令分隔符 | TCP 发送时自动追加 `\r\n`，不要在命令字符串中手动添加 |
| 二进制与文本自动切换 | 数据接收时自动检测帧类型，无需手动切换模式 |
| 保存配置 | 修改 SET 参数后需发送 `SAVE` 命令持久化，否则断电丢失 |
| 单位以设备为准 | 连接后先读取设备当前单位，若与本地配置不同则以设备单位为准 |
| 采集期间禁止查询 | SCAN 模式下数据流与查询响应会混杂，需先 STOP 再查询 |
| 通道数为 16 | 只有压力通道，无大气压和温度通道（与 DAQ-P-1604 的 18 通道不同） |
| Socket keep-alive | 连接成功后自动启用，间隔 10s，防止长时间空闲断连 |
