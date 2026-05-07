---
name: const860-pressure-device
description: >
  康斯特 ConST860 智能压力控制器通信协议参考，涵盖标准 SCPI 命令集、
  逗号分隔响应格式、单位字符串映射和开发扩展指南。
  用于编写、调试、修改 ConST860 打压设备驱动代码。
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

**核心特性**：使用 `"值,单位"` 逗号分隔响应格式。与通用 SCPI 在稳定查询和压力查询上有差异。跨设备对比详见 [REFERENCE.md](../REFERENCE.md)。

---

## 连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识
3. 发送 PRESsure:MODule:UNIT? 1 → 读取单位
4. 以设备单位为准
5. 发送 PRESsure? → 读取当前压力（"值,单位" 格式）
6. 通知上层
```

---

## 命令参考

### 设备标识查询

**命令**：`*IDN?`

响应：`厂商,型号,序列号,版本`

---

### 单位命令

ConST860 使用**单位字符串**作为参数，与数字代码体系不同。

#### 设置 — `PRESsure:MODule:UNIT <模块ID>,<单位字符串>`

示例：
- `PRESsure:MODule:UNIT 1,kPa`
- `PRESsure:MODule:UNIT 1,MPa`
- `PRESsure:MODule:UNIT 1,psi`
- `PRESsure:MODule:UNIT 1,bar`

#### 查询 — `PRESsure:MODule:UNIT? <模块ID>`

响应示例：`kPa`、`MPa`、`bar`

#### 支持的 9 种单位

| 字符串 | 单位 | 字符串 | 单位 |
|--------|------|--------|------|
| `Pa` | 帕斯卡 | `psi` | psi |
| `kPa` | 千帕 | `kgf/cm2` | kgf/cm² |
| `MPa` | 兆帕 | `mmHg` | mmHg |
| `bar` | 巴 | `atm` | atm |
| `mbar` | 毫巴 | | |

**解析**：
```
function parseUnit(response):
    upper = response.trim().toUpperCase()
    switch upper:
        "PA" → PA; "KPA" → KPA; "MPA" → MPA
        "BAR" → BAR; "MBAR" → MBAR
        "PSI" → PSI
        "KG/CM2", "KGF/CM2" → KGF_CM2
        "MMHG", "MMHG0C" → MMHG
        "ATM" → ATM
        default → MPA
```

---

### 读取当前压力

**命令**：`PRESsure?`（与通用 SCPI 的 `PRESsure0?` 不同）

响应格式：`值,单位`，如 `0.28,kPa`

**解析**：
```
function parsePressure(response):
    parts = response.split(",")
    if parts.length >= 2:
        value = parseFloat(parts[0]); unit = parseUnit(parts[1])
        return { value, unit }

    value = parseFloat(response)
    if not isNaN(value): return { value, unit: getDefaultUnit() }
    return null
```

---

### 设置目标压力

**命令**：`PRESsure:TARGet <value>`（与通用 SCPI 格式相同）

#### 查询目标压力 — `PRESsure:TARGet?`

#### 查询目标范围 — `PRESsure:TARGet:RANGe?`

响应：`0,7350,kPa`

**解析**：
```
function parseTargetRange(response):
    parts = response.split(",")
    if parts.length >= 3:
        return { low: parseFloat(parts[0]), high: parseFloat(parts[1]), unit: parseUnit(parts[2]) }
    return null
```

---

### 查询稳定状态

**命令**：`PRESsure:STABle?`（简化格式，非标准 `PRESsure:MODule1:STABle?`）

响应：`0`=不稳定，`1`=稳定

---

### 输出控制

| 操作 | 命令 |
|------|------|
| 启用输出 | `PRESsure:MODule:CONTrol CONTROL` |
| 关闭输出 | `PRESsure:MODule:CONTrol MEASURE` |
| 排空 | `PRESsure:MODule:CONTrol VENT` |
| 查询模式 | `PRESsure:MODule:CONTrol?` |

---

### 系统错误查询

**命令**：`SYSTem:ERRor?`

响应：`错误码,错误信息`

---

### IEEE 488.2 通用命令

| 命令 | 功能 |
|------|------|
| `*IDN?` | 标识查询 |
| `*RST` | 设备复位 |
| `*CLS` | 清除状态 |

---

## 命令格式特殊处理

ConST860 有 4 个命令与通用 SCPI 格式不同：

| 功能 | 内部映射 | 实际发送 |
|------|---------|---------|
| 压力查询 | `PRESsure?` | `PRESsure?` |
| 稳定查询 | `PRESsure:STABle?` | `PRESsure:STABle?` |
| 查询单位 | `UNIT?` | `PRESsure:MODule:UNIT? <模块ID>` |
| 设置单位 | `UNIT` | `PRESsure:MODule:UNIT <模块ID>,<单位>` |

**构建逻辑**：
```
function buildCommandString(command):
    if command.type in SPECIAL_COMMAND_MAP:
        format = SPECIAL_COMMAND_MAP[command.type]
        if command is unit and has params: return 拼接完整 SCPI 格式
        return format
    return 通用Build(command)
```

---

## 开发扩展

### 添加新特殊命令

```
1. 发现命令格式与通用 SCPI 不同时，加入特殊格式映射表
2. 如需参数拼接，在构建逻辑中添加条件分支
```

### 响应格式注意事项

- 始终按逗号拆分解析，兼容纯数字回退
- `PRESsure?` → `"0.28,kPa"`
- `PRESsure:TARGet:RANGe?` → `"0,7350,kPa"`

---

## 注意事项

1. **单位参数为字符串**（如 `kPa`），不是数字代码
2. **压力响应是 `"值,单位"` 格式**
3. **稳定查询用 `PRESsure:STABle?`**，不是 `PRESsure:MODule1:STABle?`
4. **控制命令用 `PRESsure:MODule:CONTrol`**，不是 `PRESsure:MODE`
5. **支持 9 种压力单位**，比 ConST820（5 种）多
6. **命令串行执行**，设置类不等待响应
7. **排空超时加倍**
