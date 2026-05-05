---
name: daq-p1604
description: DAQ-P-1604 压力采集设备驱动开发指南。涵盖 TCP 协议、数据流解析、通道映射、单位系统、设备发现和生命周期管理。Use when writing or modifying DAQ-P-1604 driver code, debugging pressure data streams, adding DAQ-P-1604 features, or when user mentions DAQ-P-1604, DAQP1604, 16通道压力采集, atm pressure, or p1604 unit conversion.
---

# DAQ-P-1604 设备驱动开发

## 设备概述

DAQ-P-1604 是一款基于 TCP/IP 的 18 通道压力采集设备：

| 通道 | 内容 | 默认单位 | 默认精度 |
|------|------|----------|----------|
| CH1~CH16 | 压力值 | 由硬件 EU 系数决定 | 3 位小数 |
| CH17 | 大气压力 | Pa | 0 位小数 |
| CH18 | 大气温度 | ℃ | 2 位小数 |

出厂默认地址 `192.168.3.101:9000`，自定义二进制协议（非 Modbus）。

## 快速开始：生命周期

```
// 1. 构造驱动（需传入 DeviceConfig + ChannelConfig[]）
const driver = new DAQP1604Device(deviceConfig, channels)

// 2. 注册数据回调
driver.onData((payload) => { /* payload: { deviceId, timestamp, channels[], channelIndices[] } */ })

// 3. 连接 → 自动发送 w1601 开启长度前缀模式
const ok = await driver.connect()

// 4. 启动采集 → 内部执行 c 00(参数), c 05(返回内容), c 01(启动数据流)
const started = await driver.startAcquisition()

// 5. 停止采集 → c 02(停止数据流)
await driver.stopAcquisition()

// 6. 断开
await driver.disconnect()
```

## 核心工作流

### 连接流程

1. 创建 TCP socket，连接 `host:port`，超时 5s
2. 注册 `socket.on('data')` 监听
3. 连接成功后立即发送 `w1601` → 启用"2 字节大端长度前缀"模式

**注意**：`w1601` 必须在任何其他命令之前发送，确保后续所有响应都带长度前缀。

### 采集流程（数据流模式）

步骤 | 命令 | 说明
---|---|---
配置参数 | `c 00 <st> FFFF 1 <per> 7 0` | st=1, per=周期(ms), 7=大端float, 0=连续
配置返回内容 | `c 05 <st> 0810` | 0810(0010|0800)=压力+大气压+大气温度
启动数据流 | `c 01 <st>` | 开始推送二进制帧
停止数据流 | `c 02 <st>` | 停止推送

### 数据流帧解析

传入 socket data 的二进制帧（2 字节长度前缀已剥离）：

```
| byte0 | byte1-2 | byte3-4 | byte5-76 (18×float32 BE) |
| 0x01  | seq(16) | 保留    | 通道数据                  |
```

**关键细节**：
- 前 16 个 float 按 CH16→CH1 顺序排列，**必须反转**为 CH1→CH16
- CH17（大气压）和 CH18（温度）正常顺序
- 每帧解析后只输出 `enabled === true` 的通道到 `DataPayload`

### 通道热更新

运行期间调用 `driver.updateChannels(newChannels)` 更新通道配置，无需重连。

## 命令速查

所有命令响应以 `A` 开头表示成功，`Nxx` 表示错误。

| 命令 | 功能 |
|------|------|
| `w1601` | 启用 2 字节大端长度前缀（连接后必须首个执行） |
| `c 00 <st> FFFF 1 <per> 7 0` | 配置流参数 |
| `c 05 <st> 0810` | 配置流返回内容（压力+大气压力+大气温度） |
| `c 01 <st>` | 启动流 |
| `c 02 <st>` | 停止流 |
| `rFFFF0` | 轮询一次 16 通道压力值（调试用，逆序返回，不含 CH17/18） |
| `u01101` | 读取 EU 系数（单位换算用） |
| `v01101 <coeff>` | 写入 EU 系数 |
| `psi9000` | UDP 广播发现命令（通过 7000 端口发送） |

## 单位系统

设备内部使用 EU 系数（psi 基准），通过相对系数映射到目标单位：

| 单位 | 系数 |
|------|------|
| psi | 1.0 |
| Pa | 6894.757 |
| kPa | 6.894757 |
| MPa | 0.006894757 |
| kgf/cm² | 0.070307 |

读取：`u01101` → 解析 float → 查表匹配（容差 1e-3）→ 得到 `PressureUnit`
写入：`v01101 <系数>` → 设备返回 `A` 即成功

## 设备发现（UDP 广播）

1. 局域网内通过 UDP 向各网段广播地址端口 7000 发送 `psi9000`
2. 在 7001 端口监听响应
3. 响应 CSV 格式：`<IP>,<MAC>,,<序列号>,<固件版本>,,<端口>,<子网掩码>,<网关>`
4. 解析为 `DiscoveredDevice` 返回

详见 [REFERENCE.md](REFERENCE.md)
