---
name: spc4000-pressure-device
description: Scanivalve SPC4000 打压设备通信协议参考，涵盖 Mensor 兼容命令集、正负压分流控制、软件稳定判断算法和单位代码映射。
---

# Scanivalve SPC4000 打压设备

## 设备概述

| 属性 | 值 |
|------|-----|
| 厂商 | Scanivalve |
| 型号 | SPC4000 压力校准器 |
| 协议 | Mensor 兼容命令集（非 SCPI 协议） |
| 物理连接 | TCP/IP，默认端口 5025 |
| 命令分隔符 | 换行符 `\n` (LF) |
| 推荐超时 | 5000ms（排空 10000ms） |

**核心特性**：SPC4000 **完全不遵循 SCPI 协议**，使用 Mensor 兼容的扁平命令集。与其它打压设备有 3 个根本性架构差异：

1. **没有硬件稳定状态查询命令** — 必须通过软件循环读取压力并计算偏差来判断稳定
2. **正负压分流控制** — 正压用 `GP`，负压用 `GN`，没有统一的设压命令
3. **没有独立的输出启用命令** — 发送 GP/GN 自动进入控制模式

---

## 一、连接流程

```
1. 建立 TCP 连接
2. 发送 ID? → 获取设备标识
3. 发送 Units? → 读取当前压力单位
4. 以设备单位为准
5. 发送 RP → 读取当前压力
6. 通知上层：单位信息 + 当前压力值
```

---

## 二、命令参考

### 2.1 ID? — 设备标识查询

**发送**：`ID?`

**响应**：设备标识字符串

> **注意**：SPC4000 使用 `ID?`，不是 SCPI 标准的 `*IDN?`。

---

### 2.2 单位命令

#### 查询当前单位 — `Units?`

**发送**：`Units?`

**响应**：数字代码

#### 设置单位 — `Units <code>`

**发送**：`Units <code>`

**说明**：设置类命令，发送后不等待响应。

#### SPC4000 单位代码映射表

| 代码 | 单位 | 说明 |
|------|------|------|
| 23 | Pa | 帕斯卡 |
| 22 | kPa | 千帕 |
| 36 | MPa | 兆帕 |
| 1 | psi | 磅/平方英寸 |
| 26 | kgf/cm² | 千克力/平方厘米 |
| 14 | bar | 巴 |
| 15 | mbar | 毫巴 |
| 19 | mmHg | 毫米汞柱 |
| 13 | atm | 标准大气压 |

> **关键差异**：SPC4000 的单位数字代码体系与所有其他设备都不同。例如 MPa 为 36（通用 SCPI 为 1132，ConST820 为 2）。

**跨设备单位代码对比**：

| 单位 | 通用SCPI | ConST820 | SPC4000 |
|------|----------|----------|---------|
| MPa | 1132 | 2 | **36** |
| kPa | 1133 | 1 | **22** |
| psi | 1141 | 3 | **1** |
| Pa | 1130 | 0 | **23** |

**示例**：
- 设为 kPa → `Units 22`
- 设为 MPa → `Units 36`
- 设为 psi → `Units 1`

**单位响应解析伪代码**：

```
function parseUnit(response):
    code = parseInt(response.trim())

    // 优先按数字代码查表
    codeToUnit = {
        1:  PSI,   13: ATM,  14: BAR,  15: MBAR,
        19: MMHG,  22: KPA,  23: PA,   26: KGF_CM2,
        36: MPA
    }
    if code in codeToUnit:
        return codeToUnit[code]

    // 回退到字符串匹配
    return normalizeUnitToken(response)
```

---

### 2.3 RP — 读取当前压力

**发送**：`RP`

> **注意**：`RP` 不带问号，是一个隐式查询命令。发送后设备返回当前压力。

**响应格式**：`数值` 或 `数值,单位`

**响应示例**：
- `100.5`
- `-0.05,kPa`
- `0.28,MPa`

**解析伪代码**：

```
function parsePressure(response):
    parts = response.trim().split(",").filter(非空)

    value = parseFloat(parts[0])
    if isNaN(value): return null

    if parts.length > 1:
        unit = normalizeUnitToken(parts[1])
    else:
        unit = 当前设备单位  // 使用之前 Units? 查询到的单位

    return { value, unit }
```

---

### 2.4 GP / GN — 正负压分流控制

SPC4000 没有统一的目标压力命令，正压和负压走不同的命令。

#### GP — 输出正压并进入控制模式

**发送**：`GP <绝对值>`

#### GN — 输出负压并进入控制模式

**发送**：`GN <绝对值>`

**命令选择逻辑**（伪代码）：

```
function buildSetPressureCommand(targetPressure):
    if targetPressure >= 0:
        command = "GP"
    else:
        command = "GN"

    absValue = abs(targetPressure)
    send(command + " " + absValue)
```

**示例**：
- 打压到 100.5 kPa → `GP 100.5`
- 打压到 -50 kPa → `GN 50`
- 打压到 0 → `GP 0`（0 视为正压）

**重要特性**：
- 发送 GP/GN **自动进入控制模式**，无需也不能先调用输出启用命令
- 如果尝试单独启用输出（不设置目标压力），应该抛出错误而非发送无效命令
- 设置类命令，发送后不等待响应

---

### 2.5 Measure — 测量模式控制

#### 进入测量模式 — `Measure`

**发送**：`Measure`

**用途**：关闭输出控制，设备仅监测压力不再主动调节。这是关闭输出的唯一方式。

#### 查询是否处于测量模式 — `Measure?`

**发送**：`Measure?`

**响应**：`YES` / `NO`，或 `ON` / `OFF`，或 `1` / `0`

---

### 2.6 Mode? — 查询当前模式

**发送**：`Mode?`

**响应**：`CONTROL`、`MEASURE`、`VENT`、`STANDBY` 等

**输出状态解析伪代码**：

```
function parseOutputState(response):
    upper = response.trim().toUpperCase()

    // 这些状态表示输出未启用
    if upper in ["YES", "ON", "1", "MEASURE", "VENT", "STANDBY"]:
        return false

    // 这些状态表示输出已启用
    if upper in ["NO", "OFF", "0", "CONTROL"]:
        return true

    // 回退默认
    return false
```

---

### 2.7 Vent — 排空

**发送**：`Vent`

**超时建议**：10000ms（2 倍普通超时）

---

### 2.8 SYSTem:ERRor? — 系统错误查询

**发送**：`SYSTem:ERRor?`

**响应格式**：`错误码,错误信息`

SPC4000 支持此标准 SCPI 格式的错误查询命令。

---

## 三、软件稳定判断算法

SPC4000 **没有硬件稳定状态查询命令**，必须通过软件计算偏差来判断。这是 SPC4000 与其他所有设备最关键的差异。

### 算法流程

```
function waitForStable(targetPressure, thresholdPercent):
    stableStartTime = null

    loop every 200ms:
        // 1. 读取当前压力
        currentPressure = sendAndParse("RP")

        // 2. 计算偏差
        deviation = abs(currentPressure - targetPressure)

        // 3. 计算动态阈值（最小 0.001 防止除零）
        threshold = max(abs(targetPressure) * thresholdPercent / 100, 0.001)

        // 4. 判断稳定
        if deviation <= threshold:
            if stableStartTime == null:
                stableStartTime = now()       // 首次进入阈值，记录时间
            else if now() - stableStartTime >= 2000ms:
                return STABLE                  // 持续稳定 2 秒，判定成功
            // 否则继续等待
        else:
            stableStartTime = null             // 超出阈值，重置计时

        sleep(200ms)  // 等待后重新循环
```

### 参数说明

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `thresholdPercent` | 稳定阈值百分比 | 取决于配置（如 0.1%） |
| `稳定持续时间` | 需要在阈值内的持续时间 | 2000ms |
| `轮询间隔` | 读取压力的间隔 | 200ms |
| `最小阈值` | 防止目标为 0 时阈值为 0 | 0.001 |

---

## 四、控制状态设置的限制

SPC4000 的控制状态切换与其他设备不同：

```
MEASURE 状态 → 发送 "Measure"        ✓ 支持
VENT 状态    → 发送 "Vent"           ✓ 支持
CONTROL 状态 → 不支持单独设置         ✗ 必须通过 GP/GN 附带进入
```

这意味着：
- **不能**先切换到 CONTROL 模式再设定目标压力
- **不能**只启用输出而不指定目标值
- 打压时必须直接在 `GP`/`GN` 命令中携带目标值

---

## 五、SPC4000 与其他设备命令对比

| 功能 | 通用SCPI | ConST820 | ConST860 | SPC4000 |
|------|----------|----------|----------|---------|
| 标识查询 | `*IDN?` | `*IDN?` | `*IDN?` | **`ID?`** |
| 查询单位 | `PRESsure:MODule:UNIT? 1` | `UNIT?` | `PRESsure:MODule:UNIT? 1` | **`Units?`** |
| 设置单位 | `PRESsure:MODule1:UNIT <1132>` | `UNIT:PRESsure 2` | `PRESsure:MODule:UNIT 1,MPa` | **`Units 36`** |
| 读取压力 | `PRESsure0?` | `MEASure:SCALar:PRESsure1?` | `PRESsure?` | **`RP`** |
| 正压设目标 | `PRESsure:TARGet <v>` | `SOURce:PRESsure <v>` | `PRESsure:TARGet <v>` | **`GP <v>`** |
| 负压设目标 | `PRESsure:TARGet <-v>` | `SOURce:PRESsure <-v>` | `PRESsure:TARGet <-v>` | **`GN <v>`** |
| 稳定查询 | 硬件查询 | 硬件查询 | 硬件查询 | **软件计算** |
| 启用输出 | 独立命令 | 独立命令 | 独立命令 | **GP/GN 自动** |
| 关闭输出 | `PRESsure:MODE MEASURE` | `OUTPut:PRESsure:MODE MEASURE` | `PRESsure:MODule:CONTrol MEASURE` | **`Measure`** |
| 排空 | `PRESsure:MODE VENT` | `OUTPut:PRESsure:MODE VENT` | `PRESsure:MODule:CONTrol VENT` | **`Vent`** |
| 查询模式 | `PRESsure:MODE?` | `OUTPut:PRESsure:MODE?` | `PRESsure:MODule:CONTrol?` | **`Mode?`** |

---

## 六、采集工作流

```
1. 设置目标压力
   └── pressure >= 0 → GP <|pressure|>
       pressure < 0  → GN <|pressure|>
       (自动进入控制模式，无需单独启用输出)

2. 软件稳定等待循环（每 200ms）
   ├── RP 读取当前压力
   ├── 计算偏差 = |current - target|
   ├── 偏差 > 阈值 → 重置稳定计时器，继续
   └── 偏差 ≤ 阈值 且 持续时间 ≥ 2000ms → 判定稳定

3. 稳定确认 → 计量设备采集数据

4. 下一个压力点 或 Vent 排空
```

---

## 七、开发扩展指南

### 7.1 添加新命令

SPC4000 使用 Mensor 兼容命令，添加新命令时：

```
1. 查阅 SPC4000 的 Remote Operations 文档确认命令格式
2. 命令名使用扁平格式（如 "NewCmd"），非 SCPI 树形格式
3. 确定是查询命令（?后缀）还是设置命令（无?后缀）
4. 实现对应的发送和响应解析逻辑
```

### 7.2 固件升级后改用硬件稳定判断

如果 SPC4000 新固件支持硬件稳定状态查询：

```
1. 添加新的查询命令（如 "STABle?"）
2. 在采集流程中将稳定判断从软件计算切换为硬件查询
3. 保留软件判断作为回退方案（兼容旧固件）
```

### 7.3 添加新单位

```
1. 查询 SPC4000 文档确认新单位的数字代码
2. 在代码→单位和单位→代码两个映射表中添加对应项
3. 同时更新 normalizeUnitToken 中的字符串匹配
```

---

## 八、注意事项

1. **没有硬件稳定查询** — 稳定判断完全依赖软件循环读取 RP 并计算偏差。阈值百分比和持续时间需合理配置
2. **正负压命令不同** — `GP` 用于 ≥0，`GN` 用于 <0，参数始终传绝对值
3. **不能单独启用输出** — 尝试在无目标压力时启用输出应抛出错误
4. **RP 不是查询命令** — 命令本身不带问号，但设备会返回当前压力
5. **标识查询用 `ID?`** — 不是 `*IDN?`
6. **单位代码体系独立** — 与其他所有设备都不同，不能混用代码映射表
7. **响应格式不统一** — RP 可能返回纯数字或 `"值,单位"` 格式，解析必须兼容两种情况
8. **超时时间更长** — 推荐 5000ms（其他设备 3000ms），因为 SPC4000 响应较慢
9. **Mode? 响应值多样** — 可能是 CONTROL / MEASURE / VENT / STANDBY 等多种值，解析需覆盖所有情况
10. **命令串行执行** — TCP 连接上的命令需要互斥保护
11. **排空超时翻倍** — 10000ms
