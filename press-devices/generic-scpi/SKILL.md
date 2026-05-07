---
name: generic-scpi-pressure-device
description: >
  通用 SCPI 打压设备通信协议参考，涵盖 IEEE 488.2 标准指令、完整 SCPI 压力命令集、
  数字单位代码体系、命令回退机制和扩展开发模式。
  用于编写、调试、修改通用 SCPI 打压设备驱动代码。
---

# 通用 SCPI 打压设备

## 设备概述

| 属性 | 值 |
|------|-----|
| 协议 | SCPI (Standard Commands for Programmable Instruments) |
| 适用设备 | ConST 811A 及任何遵循 IEEE 488.2 + SCPI 压力控制标准的设备 |
| 物理连接 | TCP/IP，默认端口 5025 |
| 命令分隔符 | 换行符 `\n` (LF) |
| 推荐超时 | 5000ms（排空 10000ms） |
| 参考标准 | IEEE 488.2 + SCPI V1.2.21 |

**核心定位**：通用 SCPI 策略是所有打压设备通信的基础实现。ConST820、ConST860、SPC4000 等设备在此基础上覆盖特定命令。详见 [REFERENCE.md](../REFERENCE.md) 了解架构全景和跨设备对比。

---

## 连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识（厂商、型号、序列号、软件版本）
3. 发送 PRESsure:MODule:UNIT? 1 → 读取控压模块当前单位
4. 以设备当前单位为准（若本地配置不同则采用设备单位）
5. 发送 PRESsure0? → 读取当前压力（响应格式: "值,单位"）
6. 通知上层：单位信息 + 当前压力值
```

---

## 完整命令参考

### IEEE 488.2 共同指令（4 个）

| 命令 | 功能 | 类型 |
|------|------|------|
| `*IDN?` | 设备标识查询 | 查询 |
| `*RST` | 设备复位 | 设置 |
| `*CLS` | 清除状态寄存器和错误队列 | 设置 |
| `*TST?` | 设备自检 | 查询 |

**`*IDN?` 响应**：`厂商,型号,序列号,版本`

---

### 压力单位命令（3 个）

#### 设置 — `PRESsure:MODule<n>:UNIT <code>`

示例：`PRESsure:MODule1:UNIT 1133`（设为 kPa）

设置类命令，发送后不等待响应。

#### 查询 — `PRESsure:MODule:UNIT? <模块ID>`

响应示例：`1132`

#### 查询单位列表 — `PRESsure:MODule:UNIT:LIST?`

#### 单位数字代码表

| 代码 | 单位 | 代码 | 单位 |
|------|------|------|------|
| 1130 | Pa | 1105 | bar |
| 1133 | kPa | 1104 | mbar |
| 1132 | MPa | 1134 | mmHg |
| 1141 | psi | 1135 | atm |
| 1145 | kgf/cm² | | |

**单位响应解析伪代码**：

```
function parseUnit(response):
    code = response.trim()
    codeMap = {
        "1130": PA, "1133": KPA, "1132": MPA,
        "1141": PSI, "1145": KGF_CM2
    }
    if code in codeMap: return codeMap[code]

    upper = code.toUpperCase()
    switch upper:
        "PA", "1130"   → PA
        "KPA", "1133"  → KPA
        "MPA", "1132"  → MPA
        "BAR"          → BAR
        "PSI", "1141"  → PSI
        default        → MPA
```

---

### 压力测量命令（3 个）

#### 读取控压模块压力 — `PRESsure0?`

响应格式：`数值,单位,附加数据...`

**解析**：
```
function parsePressure(response):
    if response is empty: return null
    parts = response.split(",")
    if parts.length >= 2:
        value = parseFloat(parts[0])
        unit = parts[1]
        if isNaN(value): return null
        return { value, unit }
    return null
```

#### 读取指定模块测量值 — `PRESsure:MODule<n>:MEASure?`

#### 读取压力稳定状态 — `PRESsure:MODule<n>:STABle?`

默认模块 1，响应 `0`=不稳定，`1`=稳定。

子类覆盖：ConST820 → `OUTPut:PRESsure:STABle?`，ConST860 → `PRESsure:STABle?`，SPC4000 → 无（软件判断）。

---

### 压力输出控制（6 个）

#### 设置目标压力 — `PRESsure:TARGet <value>`

打压核心命令，设置类不等待响应。子类覆盖：ConST820 → `SOURce:PRESsure`，SPC4000 → `GP`/`GN`。

#### 查询目标压力 — `PRESsure:TARGet?`

#### 查询目标值范围 — `PRESsure:TARGet:RANGe?`

响应格式：`最小值,最大值,单位`

**解析**：
```
function parseTargetRange(response):
    parts = response.split(",")
    if parts.length >= 3:
        low = parseFloat(parts[0]); high = parseFloat(parts[1])
        unit = parts[2]
        if not isNaN(low) and not isNaN(high): return { low, high, unit }
    return null
```

#### 设置压力控制模式 — `PRESsure:MODE <STATE>`

参数：`CONTROL` / `MEASURE` / `VENT`

子类覆盖：ConST820 → `OUTPut:PRESsure:MODE`，ConST860 → `PRESsure:MODule:CONTrol`

#### 查询压力控制模式 — `PRESsure:MODE?`

响应：`CONTROL` → 输出启用；`MEASURE` / `VENT` → 未启用

#### 模块控制状态 — `PRESsure:MODule:CONTrol` / `CONTrol?`

---

### 输出控制（2 个）

```
启用输出 → PRESsure:MODE CONTROL
关闭输出 → PRESsure:MODE MEASURE
查询状态 → PRESsure:MODE? → "CONTROL"=已启用
```

---

### 排空命令（3 个）

| 命令 | 说明 |
|------|------|
| `PRESsure:MODE VENT` | 执行排空（超时 2 倍默认） |
| `PRESsure:Vent <value>` | 设置排空目标压力 |
| `PRESsure:Vent?` | 查询排空目标压力 |

---

### 设定点限制（4 个）

| 命令 | 说明 |
|------|------|
| `PRESsure:PLIMit <下限>,<上限>` | 设置压力输出范围 |
| `PRESsure:PLIMit?` | 查询限制范围 |
| `PRESsure:PLIMit:ENABle <0\|1>` | 启用/禁用限制 |
| `PRESsure:PLIMit:ENABle?` | 查询限制启用状态 |

**限制查询解析**：`response.split(",")` → `{ lower, upper, unit }`

---

### 压力模块信息（7 个）

| 命令 | 说明 |
|------|------|
| `PRESsure:MODule<n>:ONLIne?` | 模块在线状态 |
| `PRESsure:MODule<n>:INFO?` | 详细信息 |
| `PRESsure:MODule<n>:PTYPe?` | 压力类型 G/A/D |
| `PRESsure:MODule<n>:RANGe?` | 量程 |
| `PRESsure:MODUle:VALUes?` | 所有模块压力值 |
| `PRESsure:MODule<n>:FILTer` | 滤波设置/查询 |
| `PRESsure:MODule<n>:MULTirange?` | 是否多量程 |

### 量程管理（5 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:RANGe:LIST?` | 多量程列表 |
| `PRESsure:RANGe:INDEx` | 当前量程索引 |
| `PRESsure:RANGe:MODE` | 量程模式 |

### 压力类型（2 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:TYPE` | 设置压力类型 G/A/D |
| `PRESsure:TYPE?` | 查询当前类型 |

### 当前模块（2 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:MODule` | 设置当前控压模块 |
| `PRESsure:MODule?` | 查询当前模块编号 |

### 手动阶跃（4 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:STEP <step>` | 设置步进值 |
| `PRESsure:STEP?` | 查询步进值 |
| `PRESsure:STEP:UP` | 向上阶跃 |
| `PRESsure:STEP:DOWN` | 向下阶跃 |

### 系统错误查询

**命令**：`SYSTem:ERRor?`

响应：`错误码,错误信息`。循环查询直到 `0,"No error"` 可清空队列。

---

## 命令字符串构建

```
function buildCommandString(command):
    cmdStr = command.type
    if command 有参数:
        params = command.parameters.map(p => {
            if p 是字符串: return p
            if p 是数字:   return p.toString()
            if p 是布尔:   return p ? "ON" : "OFF"
        }).join(",")
        cmdStr += " " + params
    return cmdStr + 换行符
```

布尔值自动转换为 `ON`/`OFF`，参数逗号分隔。

---

## 响应解析方法总览

| 方法 | 输入 | 输出 |
|------|------|------|
| 压力解析 | `"0.566,MPa"` | `{ value, unit }` |
| 标识解析 | `"厂商,型号,序列号,版本"` | `{ manufacturer, model, serialNumber, version }` |
| 目标范围解析 | `"0,73.5,MPa"` | `{ low, high, unit }` |
| 限幅解析 | `"0.005,70,MPa"` | `{ lower, upper, unit }` |
| 输出状态解析 | `"CONTROL"` | `true` / `false` |
| 单位解析 | `"MPa"` / `"1132"` | `PressureUnit.MPA` |

---

## 注意事项

1. **默认控压模块是模块 1（控制模块）**
2. **设置类命令不等待响应**
3. **命令串行执行**，需互斥锁保护
4. **排空超时翻倍**：10000ms
5. **压力响应只取前两段**：`"0.566,MPa,extra"` → 忽略 `extra`
6. **单位代码为 4 位数字**（与 ConST820 0~4、SPC4000 1~36 不同）
7. **此策略同时是回退策略**：所有未实现专用策略的设备型号都会回退到此通用实现
