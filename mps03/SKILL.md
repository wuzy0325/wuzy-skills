---
name: mps03
description: MPS-03 多功能探针系统（风洞测量）设备驱动开发指南。涵盖 ASCII 文本协议（#SET/#GET/#START/#STOP/#SAVE 命令体系）、CSV 数据帧解析、温度传感器模式（TMODE/TCHO/TTYPE）、配置管理和生命周期。Use when writing or modifying MPS03 driver code, debugging probe data streams, adding MPS03 features, or when user mentions MPS-03, MPS03, 多功能探针, 风洞测量, 五孔探针数据, probe system, or 攻角侧滑角.
---

# MPS-03 多功能探针系统驱动开发

## 设备概述

MPS-03 是一款基于 TCP/IP 的风洞测量数据采集设备（**数据采集设备，非运动控制器**），可输出 16 通道气动数据：

| 索引 | 通道 | 名称 | 单位 | 量程 |
|------|------|------|------|------|
| 0 | CH1 | 攻角 α | ° | [-30, 30] |
| 1 | CH2 | 侧滑角 β | ° | [-30, 30] |
| 2 | CH3 | 马赫数 Ma | - | [0, 2] |
| 3 | CH4 | 速度 V | m/s | [-500, 500] |
| 4 | CH5 | X 方向速度 Vx | m/s | [-500, 500] |
| 5 | CH6 | Y 方向速度 Vy | m/s | [-500, 500] |
| 6 | CH7 | Z 方向速度 Vz | m/s | [-500, 500] |
| 7 | CH8 | 传感器 1 S1 | - | [-100, 100] |
| 8 | CH9 | 传感器 2 S2 | - | [-100, 100] |
| 9 | CH10 | 传感器 3 S3 | - | [-100, 100] |
| 10 | CH11 | 传感器 4 S4 | - | [-100, 100] |
| 11 | CH12 | 传感器 5 S5 | - | [-100, 100] |
| 12 | CH13 | 传感器 6 S6 | - | [-100, 100] |
| 13 | CH14 | 总压 P_total | kPa | [0, 200] |
| 14 | CH15 | 外部温度 T_ext | ℃ | [-50, 100] |
| 15 | CH16 | 内部温度 T_int | ℃ | [-50, 100] |

出厂默认地址 `192.168.1.9:9000`，ASCII 文本协议（命令以 `\r\n` 结尾）。

## 快速开始：生命周期

```
// 1. 构造驱动（需传入 DeviceConfig + ChannelConfig[]）
const driver = new MPS03Device(deviceConfig, channels)

// 2. 注册数据回调
driver.onData((payload) => { /* payload: { deviceId, timestamp, channels[], channelIndices[] } */ })

// 3. 连接 → 自动读取全部硬件配置（readAllConfigs）
const ok = await driver.connect()

// 4. 启动采集 → 先发送 #SET BIN 0，再发送 #START
const started = await driver.startAcquisition()

// 5. 停止采集 → #STOP
await driver.stopAcquisition()

// 6. 断开
await driver.disconnect()
```

## 核心工作流

### 连接流程

1. 创建 TCP socket，连接 `host:port`（默认 `192.168.1.9:9000`），超时 5s
2. 注册 `socket.on('data')` 监听
3. 连接成功后启用 TCP keep-alive（间隔 10s）
4. 连接后自动执行 `readAllConfigs()`：
   - 依次发送 `#GET AVG`、`#GET BIN`、`#GET DELAY`、`#GET CNUM`、`#GET HEAD`、`#GET CHA`、`#GET TMODE`、`#GET TTYPE`、`#GET TCHO`
   - 解析后通过 `onConfigSynced` 回调通知上层

### 采集流程

```
步骤1: 连接设备
  └── TCP socket.connect(9000, host)

步骤2: 启动采集
  ├── #SET BIN 0           → 确保字符串模式（逗号分隔 CSV）
  └── #START               → 设备开始推送数据（无响应帧）

步骤3: 接收数据（CSV 行解析）
  设备按 #SET DELAY 设定的间隔推送 CSV 行（\r\n 分隔）：
  └── HEAD=1: <seq>,<α>,<β>,<Ma>,<V>,<Vx>,<Vy>,<Vz>,<S1>-<S6>,<P_total>,<T_ext>,<T_int>
  └── HEAD=0: <α>,<β>,<Ma>,<V>,<Vx>,<Vy>,<Vz>,<S1>-<S6>,<P_total>,<T_ext>,<T_int>

步骤4: 停止采集
  └── #STOP                → 停止推送（无响应帧）
```

### 断连处理

- Socket `close` 事件触发时自动清理
- 重置 `recvBuffer`、`pendingResolve`
- 标记 `connected = false`
- 触发 `handleDisconnect()`（基类提供指数退避重连逻辑）

## 命令速查

所有命令以 `\r\n` 结尾，使用 ASCII 编码发送。

⚠️ **关键差异**：SET/GET/SAVE 命令的响应**不带换行符**，仅采集数据行以 `\r\n` 结尾。驱动必须使用超时机制（而非行解析器）接收命令响应。

### 设置命令（#SET）

| 命令 | 参数 | 值范围 | 说明 |
|------|------|--------|------|
| `#SET AVG <N>` | 平均次数 | 4, 8, 32, 64 | 每数据点的采样平均次数 |
| `#SET BIN <N>` | 输出格式 | 0=字符串, 1=十六进制小端 | 数据包格式 |
| `#SET DELAY <MS>` | 输出间隔 | 10~60000 ms | 数据推送周期 |
| `#SET CNUM <N>` | 采集次数 | 0=无限, >0=有限 | 达到次数后自动停止 |
| `#SET HEAD <N>` | 序号头 | 0=关闭, 1=开启 | 数据行首是否加序号 |
| `#SET CHA <MASK>` | 通道掩码 | 4 位十六进制（如 `FFFF`） | 按位使能通道 |
| `#SET TMODE <HH>` | 温度模式 | 十六进制输入（如 `11`=0x11） | 高4位=外部, 低4位=内部 |
| `#SET TTYPE <N>` | 热电偶类型 | 0=K,1=B,2=E,3=J,4=T,5=S,6=N,7=R,8=C | 外部热电偶型号 |
| `#SET TCHO <N>` | 温度选择 | 0=内部, 1=外部 | 用于计算的温度源 |

### 查询命令（#GET）

| 命令 | 响应示例 | 响应类型 | 说明 |
|------|---------|----------|------|
| `#GET AVG` | `8` | 十进制数字 | |
| `#GET BIN` | `0` | 十进制数字 | |
| `#GET DELAY` | `1000` | 十进制数字 | |
| `#GET CNUM` | `0` | 十进制数字 | |
| `#GET HEAD` | `1` | 十进制数字 | |
| `#GET CHA` | `FFFF` | 十六进制字符串 | |
| `#GET TMODE` | `17` | 十进制数字 | ⚠️ 输入为十六进制，返回为十进制 |
| `#GET TTYPE` | `2` | 十进制数字 | |
| `#GET TCHO` | `E` | 字母 | ⚠️ `E`=外部, `I`=内部（非数字） |

### 控制命令

| 命令 | 响应 | 说明 |
|------|------|------|
| `#START` | 无响应帧 | 开始推送数据流 |
| `#STOP` | 无响应帧 | 停止推送数据流 |
| `#SAVE` | `A`（无换行） | 持久化当前设置到非易失存储 |
| `#CAL<A> <B> <C> <D>` | - | 校准命令（v1 未实现） |

## 采集数据格式（BIN=0）

### HEAD=1（17 字段）

每行以 `\r\n` 结尾，逗号分隔：

```
<seq>,<α>,<β>,<Ma>,<V>,<Vx>,<Vy>,<Vz>,<S1>,<S2>,<S3>,<S4>,<S5>,<S6>,<P_total>,<T_ext>,<T_int>
```

示例：
```
0,14.50,-5.87,0.5700,182.99,-45.59,18.71,176.23,-11.64,0.00,-12.38,-0.00,-13.36,72.02,1.40,-13.12,0.00
```

### HEAD=0（16 字段）

```
<α>,<β>,<Ma>,<V>,<Vx>,<Vy>,<Vz>,<S1>,<S2>,<S3>,<S4>,<S5>,<S6>,<P_total>,<T_ext>,<T_int>
```

### 字段索引映射

| 索引 | 字段 | 单位 |
|------|------|------|
| 0 (HEAD=0) / 1 (HEAD=1) | α 攻角 | ° |
| 1 / 2 | β 侧滑角 | ° |
| 2 / 3 | 马赫数 Ma | - |
| 3 / 4 | 速度 V | m/s |
| 4 / 5 | X 方向速度 Vx | m/s |
| 5 / 6 | Y 方向速度 Vy | m/s |
| 6 / 7 | Z 方向速度 Vz | m/s |
| 7 / 8 | 传感器 1 | - |
| 8 / 9 | 传感器 2 | - |
| 9 / 10 | 传感器 3 | - |
| 10 / 11 | 传感器 4 | - |
| 11 / 12 | 传感器 5 | - |
| 12 / 13 | 传感器 6 | - |
| 13 / 14 | 总压 P_total | kPa |
| 14 / 15 | 外部温度 T_ext | ℃ |
| 15 / 16 | 内部温度 T_int | ℃ |

### 数据行识别

`handleData` 中的 `isDataLine` 逻辑排除以下行：
- 等于 `A`（SET/SAVE 响应）
- 以 `#` 开头（命令回显）
- 不包含 `,`（非 CSV 行）

## 命令协议细节

### 响应接收机制（超时模式）

驱动内部使用单槽位 `pendingResolve` 接收命令响应：

```
sendCommand(command, timeoutMs):
  1. 清空 recvBuffer
  2. 注册 pendingResolve 回调
  3. 设置超时定时器（默认 3000ms）
  4. socket.write(command + '\r\n')
  5. 等待 handleData 触发 → 任意数据即视为响应完成
  6. 超时则 reject
```

**关键**：SET/GET/SAVE 的响应**没有换行符**，因此不能依赖 `\n` 分割来接收命令响应。

### TMODE 十六进制陷阱

- `#SET TMODE 11` 中的 `11` 被设备解释为十六进制 `0x11` (= 十进制 17)
- `#GET TMODE` 返回十进制 `17`
- 驱动内部使用十进制存储，SET 时 `.toString(16)` 转为十六进制字符串

TMODE 位说明：

| 位 | 含义 | 值 |
|----|------|-----|
| 高 4 位 | 外部传感器类型 | 1=热电偶, 0=PT100 |
| 低 4 位 | 内部传感器类型 | 1=DS18B20（固定） |

常用值：

| 十进制 | 十六进制输入 | 外部 | 内部 |
|--------|-------------|------|------|
| 1 | `01` | PT100 | DS18B20 |
| 17 | `11` | 热电偶 | DS18B20 |

### TCHO 字母映射

`#GET TCHO` 返回字母，但 `#SET TCHO` 使用数字：

| GET 返回 | SET 输入 | 含义 |
|----------|---------|------|
| `I` | `0` | 内部温度 |
| `E` | `1` | 外部温度 |

## 配置系统

### MPS03HardwareConfig 结构

```typescript
interface MPS03HardwareConfig {
  avg: number      // 平均次数 (4/8/32/64)
  bin: number      // 输出格式 (0=字符串, 1=十六进制)
  delay: number    // 输出间隔 ms (10~60000)
  cnum: number     // 采集次数 (0=无限)
  head: number     // 序号头 (0=关闭, 1=开启)
  cha: string      // 通道掩码 hex (FFFF)
  tmode: number    // 温度模式（内部十进制存储）
  ttype: number    // 热电偶类型 (0=K...8=C)
  tcho: string     // 温度选择 ('E'/外部, 'I'/内部)
}
```

### 配置同步流程

连接成功后自动执行：
1. `#GET AVG` → 读取平均次数
2. `#GET BIN` → 读取输出格式
3. `#GET DELAY` → 读取输出间隔
4. `#GET CNUM` → 读取采集次数
5. `#GET HEAD` → 读取序号头开关
6. `#GET CHA` → 读取通道掩码
7. `#GET TMODE` → 读取温度模式
8. `#GET TTYPE` → 读取热电偶类型
9. `#GET TCHO` → 读取温度选择

配置通过 `onConfigSynced` 回调通知上层 UI。

### 配置持久化

`applyHardwareConfig()` 执行完所有 `#SET` 命令后，**自动调用 `#SAVE`** 将参数持久化到设备非易失存储，否则断电丢失。

### DELAY 与 samplingRate 转换

MPS03 使用 `DELAY`（毫秒间隔），UI 层使用 `samplingRate`（赫兹）：
- `samplingRate = Math.round(1000 / delay)`
- `delay = Math.round(1000 / samplingRate)`

## 通道热更新

运行期间调用 `driver.updateChannels(newChannels)` 更新通道配置，无需重连。下一帧数据将使用新的通道过滤规则。

## 通信架构要点

### 命令与数据分离

```
handleData(chunk):
  1. 若有 pendingResolve → 视为命令响应（清空 buffer → resolve）
  2. 若 acquiring → 按 \r\n 分割 → 逐行识别 → 解析 CSV → 发射 DataPayload
  3. 否则（空闲状态）→ 清空 buffer
```

### 并发安全

- **单槽位命令**：`pendingResolve` 确保同一时刻只有一个命令在等待响应
- **采集期间避免查询**：数据流与查询响应会混杂，需先停止采集再查询
- **数据行识别**：通过 `isDataLine` 过滤命令回显和 SET 响应

## 开发注意事项

| 条目 | 说明 |
|------|------|
| 命令响应无换行 | SET/GET/SAVE 响应不带 `\n`，必须用超时机制接收 |
| 采集数据有换行 | 数据行以 `\r\n` 结尾，可用行分割解析 |
| TMODE 十六进制输入 | SET 时需 `.toString(16)`，GET 返回十进制 |
| TCHO 字母返回 | `#GET TCHO` 返回 `E`/`I`，不是 `0`/`1` |
| #START/#STOP 无响应 | 无确认帧，发送后直接开始/停止数据流 |
| #SET BIN 0 在 startAcquisition | 每次采集前都设置字符串模式，确保数据格式正确 |
| #SAVE 自动执行 | applyHardwareConfig 末尾自动调用 #SAVE |
| 连接后自动读配置 | connect → readAllConfigs → onConfigSynced |
| 默认端口 9000 | ⚠️ 原始文档误标为 900，实测为 9000 |
| 非运动控制器 | MPS03 是 DAQ 设备（DeviceType 中的 `'MPS03'`），非运动控制器 |
| 重连机制 | 继承 BaseDevice 的指数退避自动重连，重连后恢复采集 |
