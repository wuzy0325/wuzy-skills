# DAQ-P-1604 协议与命令完整参考

16 通道压力扫描阀（16 压力 + 1 大气压力 + 1 温度 = 18 通道），TCP/IP 通信（默认 `192.168.3.101:9000`），UDP 发现（端口 7000/7001，命令 `psi9000`）。

---

## 通信基础

- 所有响应前 2 字节为大端长度前缀（**含自身 2 字节**），连接后必须发 `w1601` 启用
- 命令以纯 ASCII 发送（无换行符）
- 成功响应 `A`，失败响应 `Nxx`

## 命令响应码

| 响应 | 含义 |
|------|------|
| `A` | 成功（Acknowledge） |
| `N01` | 无效命令 |
| `N03` | 输入缓冲区溢出 |
| `N04` | 收到无效的 ASCII 字符 |
| `N05` | 数据字段错误 |
| `N07` | 设置的限定值无效 |
| `N08` | 扫描器错误，无效的参数 |
| `N0A` | 校准阀不在要求的阀位上 |

---

## 命令速查

### 基础控制

| 命令 | 格式 | 说明 |
|------|------|------|
| 测试通信 | `A` | → `A` |
| 复位 | `B` | → `A`，重设系数出厂值，清除数据流 |
| 读型号 | `q00` | → 如 `9116` |
| 读固件版本 | `q01` | → 版本号×100 (hex) |
| 读AD平均次数 | `q05` | → 十六进制 |
| 读TCP端口 | `q09` | → 十六进制端口号 |
| 启用长度前缀 | `w1601` | → `A`，**连接后必须首个执行** |

### 数据读取

| 命令 | 格式 | 说明 |
|------|------|------|
| 读压力 | `r<pppp><f>` | 通道16→1倒序，空格分隔 |
| 读温度 | `t<pppp><f>` | 同上 |
| 读全部压力 | `rFFFF0` | 轮询模式常用 |
| 读全部温度 | `tFFFF0` | 同上 |

通道位图 `pppp`（4位hex）：FFFF=全16通道, 000F=CH1-4, F000=CH13-16。数据格式 `f`：0=ASCII, 7=大端二进制, 8=小端二进制。

### 数据流控制

| 命令 | 格式 | 说明 |
|------|------|------|
| 配置流参数 | `c 00 <id> <mask> <sync> <per> <fmt> <mode>` | id=1/2/3, sync=1内部时钟, fmt=7大端BE, mode=0连续 |
| 启动流 | `c 01 <id>` | id=0启动全部 |
| 停止流 | `c 02 <id>` | id=0停止全部 |
| 清除流 | `c 03 <id>` | |
| 查询流配置 | `c 04 <id>` | |
| 设置流内容 | `c 05 <id> <content>` | 0810=0010\|0800=压力+大气压+温度 |
| 设置协议 | `c 06 <id> <pro>` | 0=TCP, 1=UDP（部分固件返回N05，不建议发送）|

**本系统标准配置**：
```
c 00 1 FFFF 1 <periodMs> 7 0    → 配置数据流
c 05 1 0810                      → 压力+大气压+温度
c 01 1                           → 启动
```

### 流配置参数详解

#### c 00 参数

| 参数 | 值 | 说明 |
|------|-----|------|
| st | 1 | 流 ID（固定为 1） |
| pppp | FFFF | 通道掩码，FFFF=全部 16 路压力 |
| sync | 1 | 时钟源，1=内部时钟 |
| per | periodMs | 采样周期（毫秒），最小 10ms |
| f | 7 | 数据格式，7=大端 IEEE754 float |
| num | 0 | 采集数量，0=连续 |

#### c 05 数据内容位图

| 位 | 十六进制 | 内容 |
|----|----------|------|
| 0 | 0001 | 阀位信息 |
| 1 | 0002 | 传感器温度状态信息 |
| 4 | 0010 | EU 压力数据 |
| 7 | 0080 | 传感器温度 EU 值（℃） |
| 10 | 0400 | 时间戳字段 |
| 11 | 0800 | 大气压力、大气温度 |

常用组合：

| cccc | 含义 |
|------|------|
| 0610 | 仅压力通道 + 传感器温度状态 |
| 0810 | 压力 + 大气压力 + 大气温度（18 路） |
| 0C10 | 全部（含保留通道） |

### 校准

| 命令 | 格式 | 超时 | 说明 |
|------|------|------|------|
| 零点校准 | `C 04 <pppp>` | 10s | 须在校准阀位 |
| 满量程校准 | `C 05 <pppp> <value>` | 10s | value≥90%量程，6位小数 |
| 启动多点校准 | `C 00 <pppp> <np> 1 <avg>` | 5s | np=点数, avg=平均次数 |
| 采集校准点 | `C 01 <idx> <pressure>` | 10s | idx从0开始 |
| 执行拟合 | `C 02` | 10s | 最小二乘拟合 |
| 结束校准 | `C 03` | 5s | |

### 阀门控制

| 命令 | 说明 |
|------|------|
| `@01  0` | 读阀门状态 → 1=校准位, 0/2/3=测量位 |
| `w0C01` | 设为校准位（阀打开） |
| `w0C00` | 设为测量位（阀关闭） |

### 系数与单位

| 命令 | 说明 |
|------|------|
| `u01101` | 读取全局EU压力转换系数 |
| `v01101 <value>` | 写入系数（6位小数）→ `A` |
| `w08` | 保存零点系数到非易失存储 |
| `w09` | 保存增益系数到非易失存储 |

**单位系数映射**（psi基准）：psi=1.0, Pa=6894.757, kPa=6.894757, MPa=0.006894757, kgf/cm²=0.070307

读取：`u01101` → 解析 float → 查表匹配（容差 1e-3）→ 得到 `PressureUnit`

### 系数索引（u/v 命令详细）

格式：`u<f><aa><cc>[-cc]` / `v<f><aa><cc>[-cc> <data>...`
- `f`: 0=十进制ASCII, 1=十六进制
- `aa`: 01-10=传感器1-16, 11=全局数组
- `cc`: 00=零位系数, 01=增益系数, 02-05=EU转换c0-c3, 0A=量程代码, 5F=当前EU值

---

## 帧协议

### 长度前缀格式（w1601 启用后）

```
| byte0  | byte1  | byte2..byte(N-1)     |
| lenHi  | lenLo  | payload (len-2 bytes) |
```

- 长度字段为 16 位大端无符号整数，**包含自身**（最小值为 2）
- 如果 `len < 2`，视为异常，清空缓冲区
- 数据不足时累积到 `recvBuffer`，等待下一个 TCP chunk

### 二进制数据帧结构（w1601 + c 05 1 0810 配置后）

不含 2 字节长度前缀的帧格式：

```
┌─ 长度前缀 2B (BE, 含自身) = 79 ─┐
│ 帧负载 77B                        │
│ ┌─ 头部 5B ─┬─ 通道数据 72B ────┐ │
│ │ byte0=0x01 │ CH16 (float32BE) │ │
│ │ seqNum(2B) │ CH15             │ │
│ │ reserved   │ ...              │ │
│ │            │ CH1              │ │
│ │            │ 大气压力 (CH17)    │ │
│ │            │ 温度 (CH18)       │ │
│ └────────────┴──────────────────┘ │
└────────────────────────────────────┘

| 偏移 | 大小 | 内容 |
|------|------|------|
| 0 | 1 | 固定 0x01 |
| 1 | 2 | 帧序号（uint16 BE） |
| 3 | 2 | 保留 |
| 5 | 4×18 | 18 个 float32 BE 通道值 |
```

通道值排列顺序（硬件返回）：

```
Index 0..15:  压力通道 CH16, CH15, CH14, ..., CH1  ← 逆序！
Index 16:     大气压力 CH17
Index 17:     大气温度 CH18
```

**必须在驱动层反转**：将 `values.slice(0, 16).reverse()` 后得到 CH1..CH16 正序数组。

### ASCII 帧 vs 二进制帧判定

检查 payload 前 64 字节：若全部为可打印 ASCII（0x20-0x7E）或 CR/LF，则判定为 ASCII 帧。

- **ASCII 帧** → 放入 `pendingResponses` 队列（FIFO），匹配 `sendCommand` 返回的 Promise
- **二进制帧** → 调用 `handleStreamFrame` 解析

### 解析步骤

1. 读前2字节得 `frameLength`（BE uint16）
2. 等待缓冲区 ≥ frameLength
3. 提取 `frame[2..frameLength]` 为负载
4. 区分 ASCII/二进制帧
5. 对二进制帧：跳过5字节头部，每4字节读 float32BE，共18个
6. **前16个值反转**（设备发CH16..CH1 → 需转为CH1..CH16）
7. 第17=大气压力，第18=温度

### DataPayload 输出格式

```typescript
{
  deviceId: string,       // 设备 ID
  timestamp: number,       // 本地时间戳 (Date.now())
  channels: number[],      // 仅包含 enabled 通道的数值
  channelIndices: number[] // 对应 enabled 通道的索引
}
```

---

## 轮询命令（rFFFF0）备用模式

- 返回 ASCII 格式，空格分隔
- 顺序：CH16, CH15, ..., CH1（逆序，与流帧一致）
- 解析：`parts[i]` → 对应物理通道 `16 - i`
- 不返回 CH17/CH18（大气压/温度需要额外命令）
- 当前主采集路径使用数据流模式；`rFFFF0` 仅保留作调试用途

---

## 设备发现协议

### 发现流程

```
PC(UDP 7001 监听)  --[psi9000]-->  广播地址:7000
                                    ↓
设备:7000 收到   →  回复 (CSV) → PC:7001
```

### 响应 CSV 格式

```
<IP>,<MAC>,,<序列号>,<固件版本>,,<端口>,<子网掩码>,<网关>
```

解析映射（0-indexed）：

| 索引 | 字段 |
|------|------|
| 0 | IP 地址 |
| 1 | MAC 地址 |
| 2 | （保留/空） |
| 3 | 序列号 |
| 4 | 固件版本 |
| 5-6 | （保留/空） |
| 7 | TCP 端口 |
| 8 | 子网掩码 |
| 9 | 网关 |

### 广播地址计算

```
broadcast = address | (~netmask)
```

遍历所有非内部 IPv4 网络接口，对每个接口计算广播地址并发送 `psi9000`。

---

## 伪代码示例

### 连接与采集

```
function connect(host, port=9000):
    socket = tcp.connect(host, port, timeout=5000)
    resp = send_command("w1601", timeout=1000)
    if resp starts with "N":
        log.warn("Failed to enable length prefix")
    return true

function start_acquisition(period_ms=100):
    resp = send_command("c 00 1 FFFF 1 {period_ms} 7 0")
    assert resp == "A"
    resp = send_command("c 05 1 0810")
    assert resp == "A"
    resp = send_command("c 01 1")
    assert resp == "A"

function stop_acquisition():
    send_command("c 02 1")  // best effort

function read_frame(buffer):
    length = buffer.read_uint16_be(0)
    if buffer.size < length: return null
    payload = buffer[2..length]
    buffer.consume(length)
    if is_ascii(payload):
        return ascii_response(payload)
    else:
        return parse_binary_frame(payload)

function parse_binary_frame(payload):
    seq_num = payload.read_uint16_be(1)
    values = []
    for i in 0..18:
        values.append(payload.read_float32_be(5 + i*4))
    pressures = values[0..16].reverse()
    p_atm = values[16]
    t_atm = values[17]
    return { pressures, p_atm, t_atm, seq_num }
```

### 轮询单次读取

```
function poll_all_channels():
    resp = send_command("rFFFF0", timeout=2000)
    if resp starts with "N": return error
    parts = resp.split_whitespace()
    values = array[16]
    for i in 0..min(parts.length, 16):
        physical_ch = 16 - i
        values[physical_ch - 1] = float(parts[i])
    return values
```

### 校准流程

```
function calibrate_zero_fullscale(target_pressure):
    send_command("w0C01")                          // valve → calibration
    send_command("C 04 FFFF", timeout=10000)       // zero calibrate
    send_command("C 05 FFFF {target_pressure:.6f}", timeout=10000)
    send_command("w08")                            // save zero
    send_command("w09")                            // save gain
    send_command("w0C00")                          // valve → measurement

function calibrate_multipoint(pressure_points, avg_per_point=10):
    n = pressure_points.length
    send_command("C 00 FFFF {n} 1 {avg_per_point}", timeout=5000)
    for i, target in enumerate(pressure_points):
        send_command("C 01 {i} {target:.2f}", timeout=10000)
    send_command("C 02", timeout=10000)            // fitting
    send_command("C 03", timeout=5000)             // end
    send_command("w08")
    send_command("w09")
```

### 单位读写

```
function read_unit():
    resp = send_command("u01101")
    coeff = float(resp.trim())
    for unit, expected in UNIT_MAP:
        if abs(coeff - expected) < 1e-3:
            return unit
    return "unknown"

function set_unit(unit):
    coeff = UNIT_MAP[unit]
    resp = send_command("v01101 {coeff:.6f}")
    assert resp.trim() == "A"
```

### UDP 设备发现

```
function scan_devices(timeout_ms=3000):
    udp.bind(port=7001)
    udp.set_broadcast(true)
    broadcast_targets = get_broadcast_addresses()
    for addr in broadcast_targets:
        udp.send("psi9000", addr, port=7000)
    devices = []
    wait until timeout:
        msg = udp.receive()
        parts = msg.split(",")
        if parts.length >= 9:
            devices.append({
                ip: parts[0], mac: parts[1], serial: parts[3],
                firmware: parts[4], port: int(parts[7]),
                mask: parts[8], gateway: parts[9]
            })
    return devices
```

---

## 开发注意事项

1. **连接后必须 `w1601`**：启用长度前缀才能正确拆包，且必须作为第一条命令
2. **通道倒序**：`r` 命令和数据流均为CH16..CH1，必须反转为CH1..CH16
3. **校准前切阀**：`w0C01`→校准位，完成后`w0C00`→测量位
4. **校准后保存**：`w08` + `w09` 写入非易失存储，否则断电丢失
5. **不建议发`c 06`**：部分固件返回N05
6. **帧长度含自身**：length - 2 = 实际负载数据长度
7. **所有响应以N开头为错误**

## 错误处理约定

- 所有命令发送返回 Promise，超时抛出 Error
- 响应以 `Nxx` 开头视为逻辑错误，需根据错误码判断
- Socket 异常关闭 → 清空 recvBuffer 和 pendingResponses → 触发重连
- 数据流帧间隔超过 `STREAM_GAP_WARN_MS`(1000ms) 发出警告日志
- 单位同步失败不阻塞连接流程（catch 后仅 warn）

## 重连机制

- 底层依赖 `BaseDevice.handleDisconnect()`
- 指数退避：`min(2^attempt * baseDelay, maxDelay)`
- 默认最多重试 `MAX_RECONNECT_ATTEMPTS` 次
- 重连成功重置计数器；达到上限后停止

## 数据结构类型

### DeviceConfig

```
id: string            设备唯一 ID
name: string          设备名称
type: 'DAQ-P-1604'    设备类型标识
address: string       IP 地址，默认 "192.168.3.101"
port: number          TCP 端口，默认 9000
samplingRate: number  采样率 Hz，默认 10
```

### ChannelConfig

```
index: number         通道索引 (0-17)
name: string          通道名称
enabled: boolean      是否启用
unit?: string         物理单位
precision?: number    显示精度（小数位数）
rangeMin?: number     量程下限
rangeMax?: number     量程上限
```

## 架构约束

- DAQP1604Device 继承自 BaseDevice，仅负责硬件协议实现
- 不走 IPC 直连（layering: drivers ← runtime ← modules ← ipc ← renderer）
- 通过 DriverFactory 统一创建
- 原始命令发送通过 `sendRawCommand()` 暴露给上层服务
- 数据输出通过 `onDataCallback` 推送 DataPayload，不直接操作 UI
