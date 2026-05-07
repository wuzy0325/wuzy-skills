---
name: const860-pressure-device
description: 康斯特 ConST860 打压设备通信协议参考，涵盖标准SCPI命令集、逗号分隔响应格式、单位字符串映射和开发扩展指南。
---

# 康斯特 ConST860 打压设备

## 设备概述

| 属性 | 值 |
|------|-----|
| 厂商 | 康斯特 (ConST) |
| 型号 | ConST860 智能压力控制器 |
| 协议 | SCPI 标准命令集 + 部分简化格式 |
| 物理连接 | TCP/IP，默认端口 5025 |
| 命令分隔符 | 换行符 `\n` (LF) |
| 推荐超时 | 3000ms（排空 6000ms） |
| 实测参考 | 固件 P25d&MPC V2.0.2.1，量程 0-7350 kPa |

**核心特性**：ConST860 主要使用标准 SCPI 命令格式，但在稳定状态查询和压力查询上使用简化格式。响应格式的特点是使用**逗号分隔**的 `"值,单位"` 格式，如 `"0.28,kPa"`。

---

## 一、连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识（厂商、型号、序列号、版本）
3. 发送 PRESsure:MODule:UNIT? 1 → 读取控压模块当前压力单位
4. 以设备当前单位为准（若本地配置单位不同则采用设备单位）
5. 发送 PRESsure? → 读取当前压力（响应为 "值,单位" 格式）
6. 通知上层：单位信息 + 当前压力值
```

---

## 二、命令参考

### 2.1 设备标识查询

**命令**：`*IDN?`

**响应格式**：`厂商,型号,序列号,版本`

**响应示例**：`ADDITEL,,020250010031,P25d&MPC V2.0.2.1`

---

### 2.2 单位命令

ConST860 在单位命令上混合使用了标准 SCPI 格式和简化格式。

#### 设置压力单位

**格式**：`PRESsure:MODule:UNIT <模块ID>,<单位字符串>`

**示例**：
- 设为 kPa → `PRESsure:MODule:UNIT 1,kPa`
- 设为 MPa → `PRESsure:MODule:UNIT 1,MPa`
- 设为 psi → `PRESsure:MODule:UNIT 1,psi`
- 设为 bar → `PRESsure:MODule:UNIT 1,bar`

**说明**：设置类命令，发送后不等待响应。参数中模块 ID 通常为 1（控制模块）。

#### 查询压力单位

**格式**：`PRESsure:MODule:UNIT? <模块ID>`

**示例**：`PRESsure:MODule:UNIT? 1`

**响应**：单位字符串，如 `kPa`、`MPa`、`bar`

#### 支持的单位字符串

| 发送字符串 | 含义 |
|-----------|------|
| `Pa` | 帕斯卡 |
| `kPa` | 千帕 |
| `MPa` | 兆帕 |
| `bar` | 巴 |
| `mbar` | 毫巴 |
| `psi` | 磅/平方英寸 |
| `kgf/cm2` | 千克力/平方厘米 |
| `mmHg` | 毫米汞柱 |
| `atm` | 标准大气压 |

> ConST860 支持全部 9 种单位，比 ConST820（仅 5 种）更多。

**单位响应解析伪代码**：

```
function parseUnit(response):
    upper = response.trim().toUpperCase()
    switch upper:
        "PA"      → PA
        "KPA"     → KPA
        "MPA"     → MPA
        "BAR"     → BAR
        "MBAR"    → MBAR
        "PSI"     → PSI
        "KG/CM2",
        "KGF/CM2" → KGF_CM2
        "MMHG",
        "MMHG0C"  → MMHG
        "ATM"     → ATM
        default   → 尝试通用解析，失败则返回默认单位 MPA
```

---

### 2.3 读取当前压力

**命令**：`PRESsure?`

> **注意**：与通用 SCPI 的 `PRESsure0?` 不同，ConST860 使用简化格式。

**响应格式**：`值,单位`（逗号分隔）

**响应示例**：
- `0.28,kPa`
- `-0.05,MPa`
- `100.5,psi`

**解析伪代码**：

```
function parsePressure(response):
    // 按逗号拆分
    parts = response.split(",")

    if parts.length >= 2:
        value = parseFloat(parts[0])
        unit = parseUnit(parts[1])  // 用上面的 parseUnit 解析单位字符串
        return { value, unit }

    // 纯数字回退
    value = parseFloat(response)
    if not isNaN(value):
        return { value, unit: getDefaultUnit() }  // 使用当前设备单位

    return null  // 解析失败
```

**扩展解析**：除 `{ value, unit }` 外，还可以附加：
- `unitStr`：原始单位字符串
- `timestamp`：解析时间戳
- `pressure`：压力值别名

---

### 2.4 设置目标压力（打压）

**命令**：`PRESsure:TARGet <value>`

**示例**：
- `PRESsure:TARGet 100.5`
- `PRESsure:TARGet -50`

**说明**：
- 设置类命令，发送后不等待响应
- 如果输出未启用，需先启用输出
- 此命令与通用 SCPI 格式相同

#### 查询目标压力

**命令**：`PRESsure:TARGet?`

#### 查询目标值范围

**命令**：`PRESsure:TARGet:RANGe?`

**响应格式**：`最小值,最大值,单位`

**响应示例**：`0,7350,kPa`

**解析伪代码**：

```
function parseTargetRange(response):
    parts = response.split(",")
    if parts.length >= 3:
        low  = parseFloat(parts[0])
        high = parseFloat(parts[1])
        unit = parseUnit(parts[2])
        return { low, high, unit }
    return null
```

---

### 2.5 查询压力稳定状态

**命令**：`PRESsure:STABle?`

**响应**：`0` = 不稳定，`1` = 稳定

> **注意**：实测确认 ConST860 使用简化格式 `PRESsure:STABle?`，而非标准的 `PRESsure:MODule:STABle?` 或 `PRESsure:MODule1:STABle?`。这是 ConST860 与通用 SCPI 的重要差异。

---

### 2.6 输出控制

ConST860 使用 `PRESsure:MODule:CONTrol` 命令族控制输出模式。

#### 设置控制状态

**格式**：`PRESsure:MODule:CONTrol <状态>`

| 状态值 | 含义 |
|--------|------|
| `CONTROL` | 控制模式，设备主动调节压力输出 |
| `MEASURE` | 测量模式，仅监测压力不主动调节 |
| `VENT` | 排空模式，释放压力 |

**示例**：
- 启用输出：`PRESsure:MODule:CONTrol CONTROL`
- 关闭输出：`PRESsure:MODule:CONTrol MEASURE`
- 排空：`PRESsure:MODule:CONTrol VENT`

#### 查询控制状态

**命令**：`PRESsure:MODule:CONTrol?`

**响应**：`CONTROL`、`MEASURE` 或 `VENT`

> **与通用 SCPI 的差异**：通用 SCPI 使用 `PRESsure:MODE`，ConST860 使用 `PRESsure:MODule:CONTrol`。功能等价但命令字符串不同。

---

### 2.7 系统错误查询

**命令**：`SYSTem:ERRor?`

**响应格式**：`错误码,错误信息`

---

### 2.8 IEEE 488.2 通用命令

| 命令 | 功能 |
|------|------|
| `*IDN?` | 设备标识查询 |
| `*RST` | 设备复位 |
| `*CLS` | 清除状态寄存器和错误队列 |

---

## 三、ConST860 命令格式特殊处理

ConST860 内部维护一个命令格式映射表，包含 4 个需要特殊处理的命令：

| 命令功能 | 内部映射 | 实际发送 |
|----------|---------|---------|
| 压力查询 | `PRESsure?` | `PRESsure?` |
| 稳定查询 | `PRESsure:STABle?` | `PRESsure:STABle?` |
| 查询单位 | `UNIT?` | `PRESsure:MODule:UNIT? <模块ID>`（带参数时拼接完整格式） |
| 设置单位 | `UNIT` | `PRESsure:MODule:UNIT <模块ID>,<单位>`（带参数时拼接完整格式） |

**命令字符串构建逻辑**（伪代码）：

```
function buildCommandString(command):
    // 先检查是否有特殊格式
    if command.type in SPECIAL_COMMAND_MAP:
        specialFormat = SPECIAL_COMMAND_MAP[command.type]

        // 单位命令如果有参数，拼接完整 SCPI 格式
        if command.type == "查询单位" and command 有参数:
            return "PRESsure:MODule:UNIT? " + 参数
        if command.type == "设置单位" and command 有参数:
            return "PRESsure:MODule:UNIT " + 参数

        return specialFormat  // 直接返回简化格式

    // 不在特殊映射中，使用通用构建逻辑
    return 通用Build(command)
```

---

## 四、ConST860 与其他设备命令对比

| 功能 | 通用SCPI | ConST820 | ConST860 | SPC4000 |
|------|----------|----------|----------|---------|
| 标识查询 | `*IDN?` | `*IDN?` | `*IDN?` | `ID?` |
| 查询单位 | `PRESsure:MODule:UNIT? 1` | `UNIT?` | `PRESsure:MODule:UNIT? 1` | `Units?` |
| 设置单位 | `PRESsure:MODule1:UNIT <1132>` | `UNIT:PRESsure 2` | **`PRESsure:MODule:UNIT 1,kPa`** | `Units 22` |
| 单位参数 | 数字代码(1130~1145) | 数字代码(0~4) | **字符串(kPa/MPa...)** | 数字代码(1~36) |
| 读取压力 | `PRESsure0?` | `MEASure:SCALar:PRESsure1?` | **`PRESsure?`** | `RP` |
| 压力响应 | `"值,单位"` | 纯数值 | **`"值,单位"`** | `"值"` 或 `"值,单位"` |
| 设置目标 | `PRESsure:TARGet <v>` | `SOURce:PRESsure <v>` | `PRESsure:TARGet <v>` | `GP`/`GN` |
| 稳定查询 | `PRESsure:MODule1:STABle?` | `OUTPut:PRESsure:STABle?` | **`PRESsure:STABle?`** | 无 |
| 启用输出 | `PRESsure:MODE CONTROL` | `OUTPut:PRESsure:MODE CONTROL` | **`PRESsure:MODule:CONTrol CONTROL`** | GP/GN 自动 |
| 关闭输出 | `PRESsure:MODE MEASURE` | `OUTPut:PRESsure:MODE MEASURE` | **`PRESsure:MODule:CONTrol MEASURE`** | `Measure` |
| 排空 | `PRESsure:MODE VENT` | `OUTPut:PRESsure:MODE VENT` | **`PRESsure:MODule:CONTrol VENT`** | `Vent` |

---

## 五、采集工作流

```
1. 设置目标压力
   ├── 若输出未启用 → PRESsure:MODule:CONTrol CONTROL
   └── PRESsure:TARGet <目标值>

2. 稳定等待循环（每 200ms 轮询）
   ├── PRESsure:STABle?
   ├── "0" → 继续等待
   └── "1" → 设备报告稳定 → 额外等待 2000ms 确认稳定

3. 稳定确认 → 计量设备采集数据

4. 下一个压力点 或 排空
   └── PRESsure:MODule:CONTrol VENT（排空超时翻倍）
```

---

## 六、开发扩展指南

### 6.1 命令格式选择策略

ConST860 的命令构建遵循以下优先级：

```
1. 检查命令是否在特殊格式映射表中
   → 是：使用特殊格式（但单位命令需检查参数决定是否拼接完整格式）
   → 否：使用通用 SCPI 格式

2. 特殊格式表中仅有 4 个命令需要特殊处理：
   - PRESsure? （压力查询）
   - PRESsure:STABle? （稳定查询）
   - UNIT? / UNIT （单位查询/设置）
```

### 6.2 添加新的特殊命令

当发现 ConST860 的某个命令格式与通用 SCPI 不同时：

```
1. 将命令添加到特殊格式映射表
2. 如果命令需要参数拼接，在构建逻辑中添加对应条件分支
3. 文档记录该命令的特殊格式
```

### 6.3 响应格式注意事项

- ConST860 的响应以逗号分隔的 `"值,单位"` 格式为主
- `PRESsure?` 返回 `"0.28,kPa"`，不是纯数值
- `PRESsure:TARGet:RANGe?` 返回 `"0,7350,kPa"`
- 解析时始终按逗号拆分处理，并处理纯数字的回退情况

---

## 七、注意事项

1. **单位参数为字符串**：与通用 SCPI 的数字代码（如 1132）不同，ConST860 发送 `kPa` 这样的字符串
2. **压力响应是 `"值,单位"` 格式**：需按逗号拆分解析，不可直接当纯数值处理
3. **稳定查询用简化格式 `PRESsure:STABle?`**：不是 `PRESsure:MODule1:STABle?`
4. **控制命令用 `PRESsure:MODule:CONTrol`**：不是 `PRESsure:MODE`
5. **命令串行执行**：同一 TCP 连接上的命令需要通过互斥机制串行化
6. **排空超时需加倍**：排空操作物理时间更长
7. **支持 9 种压力单位**：比 ConST820 的 5 种多出 bar、mbar、mmHg、atm
8. **设置类命令不等待响应**：避免因设备无响应导致阻塞
