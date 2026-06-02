---
name: daq-t1603
description: DAQ-T-1603 热电偶采集设备驱动开发指南。涵盖 ASCII 文本协议（@e3/@f0/@f1/@f3/@fd/@fe 命令体系）、ASCII 数据帧解析（192 字节固定宽度）、串口协议（46 字节二进制帧）、配置同步和生命周期管理。Use when writing or modifying DAQ-T-1603 driver code, debugging thermocouple data streams, adding DAQ-T-1603 features, or when user mentions DAQ-T-1603, T1603, 热电偶, thermocouple, temp model firmware, or 16通道温度采集.
---

# DAQ-T-1603 热电偶采集设备驱动开发

## 设备概述

DAQ-T-1603 是一款基于 TCP/IP 或串口的 16 通道热电偶温度采集设备：

| 通道 | 内容 | 精度 |
|------|------|------|
| CH1~CH16 | 热电偶温度值 | 取决于 thermocoupleType |

出厂默认 TCP 地址 `192.168.1.7:9000`，ASCII 文本协议（命令以 `\n` 结尾）。

部分设备固件（如 `temp` 型号，FW v1.01）的默认数据格式为 ASCII 文本（192 字节固定宽度帧），而非二进制格式。

## 快速开始：生命周期

```
// 1. 构造驱动（需传入 DeviceConfig + ThermocoupleChannelConfig[]）
const driver = new DAQT1603Device(deviceConfig, channels)

// 2. 注册数据回调
driver.onData((payload) => { /* payload: { deviceId, timestamp, channels[], channelIndices[] } */ })

// 3. 连接 → 自动执行 syncHardwareConfig（300ms 延迟后）
const ok = await driver.connect()

// 4. 启动采集 → 发送 @f0 FFFF 2
const started = await driver.startAcquisition()

// 5. 停止采集 → @f1
await driver.stopAcquisition()

// 6. 断开
await driver.disconnect()
```

## 核心工作流

### 连接流程（TCP）

1. 创建 TCP socket，连接 `host:port`，超时 5s
2. 注册 `socket.on('data')` 监听
3. 连接成功后启用 TCP keep-alive（间隔 10s）
4. 300ms 延迟后自动执行 `syncHardwareConfig()`：
   - `@e3` → 读取 16 通道热电偶类型
   - `@fd MCH/SPS/BIN/TIME/HEAD/AVG/TYPE/TRIG/TNUM` → 逐个读取配置参数

### 采集流程

```
步骤1: 连接设备
  └── TCP socket.connect(port, host)

步骤2: 启动采集
  └── @f0 <hexMask> 2    → 开始连续数据推送
      参数:
        hexMask = FFFF（16 进制通道掩码，1 bit/通道）

步骤3: 接收数据（自动解析）
  设备推送 ASCII 文本数据帧（非换行分隔，固定 192 字节/帧）：
  └── 每帧 = 16 × 12 字符 ASCII 浮点数，空格分隔
  └── 通道顺序：CH15 → CH0（需要 reverse）
  └── 使用 toUseCase 设置单位（℃/℉/K）

步骤4: 停止采集
  └── @f1                → 停止推送
```

### 断连处理

- Socket `close` 事件触发时自动清理
- 清空 `tcpRecvBuffer`、`pendingResponses` 队列
- 标记 `connected = false`
- 触发 `handleDisconnect()`（基类提供重连逻辑）

## ASCII 数据帧解析

设备在 ASCII 模式下推送的数据帧格式：

| 属性 | 值 |
|------|-----|
| 帧大小 | 192 字节（固定，无换行符） |
| 字段宽度 | 每个通道值 12 字符 ASCII |
| 字段分隔 | 空格（固定宽度左对齐填充） |
| 通道数 | 16 |
| 通道顺序 | CH15 → CH0（递减） |
| 单位 | ℃（默认） |

示例帧（16 个值为 0.000000，CH15 为 39.95℃）：
```
   0.000000    0.000000    0.000000    0.000000
   0.000000    0.000000    0.000000   39.952503
```

解析步骤：
1. 累积到 `tcpRecvBuffer`，每次检查是否 ≥ 192 字节
2. 截取 192 字节，`.toString('ascii')` → 字符串
3. `.trim().split(/\s+/)` → 16 个 token
4. `.map(Number).reverse()` → `number[]`（CH0→CH15 顺序）
5. 构造 `DataPayload` 发射

### 二进制格式模式

当 `@fd BIN` 返回 `1` 时，设备推送 64 字节二进制帧：
- 16 通道 × 4 字节 float32 LE
- 通道顺序：CH15 → CH0（需 reverse）
- 不支持 `showSequence` / `showTimestamp`

## 串口协议

### 命令

| 命令字节 | 说明 |
|----------|------|
| `55 AA 03 F0 00 00` | 开始采集 |
| `55 AA 03 F1 00 00` | 停止采集 |

串口模式下不支持配置命令（@e3/@fd/@fe），配置需在 TCP 模式下完成。

### 数据帧（46 字节）

| 偏移 | 长度 | 内容 |
|------|------|------|
| 0-1 | 2B | 帧头 |
| 2 | 1B | 长度 |
| 3 | 1B | 帧计数 |
| 4 | 1B | 反帧计数 |
| 5 | 1B | 状态 |
| 6-7 | 2B | 版本号 |
| **8-39** | **32B** | **温度数据（16 通道 × 2B）** |
| 40-43 | 4B | 备用 |
| 44 | 1B | 校验和 |

温度数据解析：
- 偏移 8 开始，每通道 2 字节，int16BE
- 值 × 0.1 = 摄氏度（0.1℃/LSB）

## 命令速查

所有命令以 `\n` 结尾，使用 ASCII 编码发送。响应 `E` 表示错误，`N` 开头表示不支持。

### 配置查询命令（@fd）

| 命令 | 功能 | 响应长度 | 示例响应 |
|------|------|----------|----------|
| `@fd MCH` | 读通道掩码 | 4 字符 | `FFFF` |
| `@fd SPS` | 读采样频率 (Hz) | 可变 | `10` |
| `@fd BIN` | 读二进制格式标志 | 1 字符 | `0` 或 `1` |
| `@fd TIME` | 读时间戳标志 | 1 字符 | `0` 或 `1` |
| `@fd HEAD` | 读序号标志 | 1 字符 | `0` 或 `1` |
| `@fd AVG` | 读平均次数 | 可变 | `4` |
| `@fd TYPE` | 读触发方式 | 1 字符 | `0` 或 `2` |
| `@fd TRIG` | 读触发沿 | 1 字符 | `0`、`1`、`2` |
| `@fd TNUM` | 读触发次数 | 可变 | `1` |
| `@fd CHECK` | 读开路检测 | 4 字符 | `0000` |

### 配置设置命令（@fe）

| 命令 | 功能 | 说明 |
|------|------|------|
| `@fe SPS <n>` | 设置采样频率 | n = 正整数 |
| `@fe TIME <0\|1>` | 时间戳显示 | 0=不显示，1=显示 |
| `@fe AVG <n>` | 设置平均次数 | n = 1~100 |
| `@fe BIN <0\|1>` | 设置二进制格式 | 0=ASCII文本，1=二进制float32 LE |
| `@fe HEAD <0\|1>` | 设置序号显示 | 0=不显示，1=显示 |
| `@fe TYPE <0\|2>` | 设置触发方式 | 0=软件，2=硬件 |
| `@fe TRIG <0\|1\|2>` | 设置触发沿 | 0=上升，1=下降，2=变化 |
| `@fe TNUM <n>` | 设置触发次数 | n = 正整数 |

### 其他命令

| 命令 | 功能 | 响应格式 |
|------|------|----------|
| `@f0 <mask> 2` | 开始连续采集（发送即忘） | 无/可能 `A` + `\n` |
| `@f1` | 停止采集（发送即忘） | 无 |
| `@e3` | 读热电偶类型 | 16 字符 ASCII，如 `KKKKKKKKKKKKKKKK` |
| `@f3 <typeString>` | 写热电偶类型 | 格式：`0<16 chars>0` |

### 热电偶类型映射

| 类型字符串 | DAQ-T-1603 码 |
|------------|---------------|
| K | `K` |
| B | `B` |
| E | `E` |
| J | `J` |
| T | `T` |
| S | `S` |
| N | `N` |
| R | `R` |
| C | `C` |
| WRE325 | `1` |
| WRE526 | `2` |
| WRE520 | `3` |

`@f3` 命令格式示例（16 通道全为 K 型）：
```
@f3 0KKKKKKKKKKKKKKKK0
```

### 命令协议细节

- **响应匹配机制**：驱动维护 `pendingResponses` 队列，按 FIFO 顺序匹配响应
- **换行判定**：收到 `\n`（0x0a）即视为一行响应完成，优先于长度判定
- **固定长度判定**：对于 `@e3`（16 字节）/ `@fd BIN`（1 字节）等已知长度命令，缓冲区达到期望长度即分发
- **可变长度静默窗口**：`@fd SPS` / `@fd AVG` / `@fd TNUM` 等长度不固定的命令，使用 30ms 静默窗口：收到数据后 30ms 内无新数据则视为响应完成
- **命令互斥**：`pendingResponses` 队列确保 FIFO 顺序，同一时刻不会交叉处理多个命令
- **超时**：默认 5000ms，超时后 reject 并移出队列
- **`@f0`/`@f1`**：走 `socket.write` 直接发送（不通过 `sendCommand`），不进入 pending 队列

## 配置系统

### DaqT1603HardwareConfig 结构

```typescript
interface DaqT1603HardwareConfig {
  thermocoupleTypes: ThermocoupleType[]  // 16 通道热电偶类型
  channelMask: string                    // 通道掩码 hex（0000-FFFF）
  samplingRate: number                   // 采样频率 (Hz)
  binaryFormat: boolean                  // true=二进制，false=ASCII 文本
  averageCount: number                   // 平均次数 1-100
  triggerMode: number                    // 0=软件触发，2=硬件触发
  triggerEdge: number                    // 0=上升沿，1=下降沿，2=变化沿
  triggerCount: number                   // 触发次数
  showTimestamp: boolean                 // 是否显示时间戳
  showSequence: boolean                  // 是否显示序号
  openCircuitCheck: string               // 开路检测掩码
}
```

### 配置同步流程

连接成功后自动执行（300ms 延迟）：

1. `@e3` → 读取 16 通道热电偶类型
2. `@fd MCH` → 读取通道掩码
3. `@fd SPS` → 读取采样频率
4. `@fd BIN` → 读取二进制格式标志
5. `@fd TIME` → 读取时间戳标志
6. `@fd HEAD` → 读取序号标志
7. `@fd AVG` → 读取平均次数
8. `@fd TYPE` → 读取触发方式
9. `@fd TRIG` → 读取触发沿
10. `@fd TNUM` → 读取触发次数

配置通过 `onConfigSynced` 回调通知上层 UI。

## 单位系统

设备支持通过 `toUseCase` 命令设置温度单位：

| 单位 | 说明 |
|------|------|
| ℃ | 摄氏度（默认） |
| ℉ | 华氏度 |
| K | 开尔文 |

## 发现协议（UDP 广播）

1. 发送 UDP 广播 `T1603` 到端口 7000
2. 在相同端口监听响应
3. 响应格式：
   - JSON：`{"ip":"...","mac":"...","serialNumber":"...","model":"...","firmwareVersion":"...","port":9000,...}`
   - CSV 回退：`<IP>,<MAC>,<序列号>,<型号>,<固件版本>,<tcpConnected>,<ipAssigned>,<端口>,<子网掩码>,...`
4. 解析为 `DiscoveredDevice` 返回

注意：部分设备固件可能返回型号 `temp`（而非 `T1603`），不影响设备类型识别。

## 通道热更新

运行期间调用 `driver.updateChannels(newChannels)` 更新通道配置，无需重连。下一帧数据将使用新的通道过滤规则。

## 开发注意事项

| 条目 | 说明 |
|------|------|
| ASCII 文本帧无换行 | 192 字节固定帧，不依赖 `\n` 分隔，需按长度截取 |
| 通道顺序需反转 | 设备发送 CH15→CH0 顺序，解析后需 `.reverse()` |
| @f0/@f1 不走 pending 队列 | 采集开始/停止命令直接 `socket.write`，不等待响应 |
| 采集期间避免查询命令 | 数据流与查询响应会混杂，需先停止采集再查询 |
| binaryFormat 决定数据格式 | BIN=0 时 192B ASCII，BIN=1 时 64B float32 LE |
| 部分固件版本差异 | `temp` 型号（FW v1.01）使用纯 ASCII 文本格式，无 `A` 确认前缀 |
| 串口不支持配置命令 | 串口模式下只能采集，配置需在 TCP 模式下完成 |
| 连接后自动同步配置 | 300ms 延迟后自动执行，无需手动触发 |
