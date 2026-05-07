---
name: const811a-pressure-device
description: 康斯特 ConST811A / 通用SCPI 打压设备通信协议参考，涵盖IEEE 488.2标准指令、完整SCPI压力命令集、数字单位代码体系、命令回退机制和扩展开发模式。
---

# 康斯特 ConST 811A / 通用 SCPI 打压设备

## 设备概述

| 属性 | 值 |
|------|-----|
| 厂商 | 康斯特 (ConST) |
| 型号 | ConST 811A（通用 SCPI 协议设备） |
| 协议 | SCPI (Standard Commands for Programmable Instruments) |
| 适用范围 | ConST 811A 及任何遵循 IEEE 488.2 + SCPI 压力控制标准的设备 |
| 物理连接 | TCP/IP，默认端口 5025 |
| 命令分隔符 | 换行符 `\n` (LF) |
| 推荐超时 | 5000ms（排空 10000ms） |
| 参考标准 | IEEE 488.2 + SCPI V1.2.21 |

**核心定位**：ConST 811A 使用标准 SCPI 协议，没有专用的命令覆盖。它是所有打压设备通信的基线实现。ConST820、ConST860、SPC4000 等设备在此基础上覆盖特定命令。

---

## 一、架构设计

### 策略继承模式

```
抽象基类（定义命令构建 + 响应解析）
  └── 通用SCPI实现（注册全部 41 个标准命令）
        ├── ConST820（覆盖 11 个命令）
        ├── ConST860（覆盖 14 个命令）
        └── SPC4000（覆盖 12 个命令，非 SCPI 协议）
```

### 命令回退机制

当设备专用策略未覆盖某个命令时：

1. 检查专用策略是否实现了该命令
2. 如果未实现 → 记录回退日志（含设备型号、命令类型、回退原因）
3. 自动执行基类的通用 SCPI 实现
4. 通知所有注册的回退事件监听器

这个机制确保向前兼容：基类新增命令后，所有设备策略自动获得默认实现。

---

## 二、连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识（厂商、型号、序列号、软件版本）
3. 发送 PRESsure:MODule:UNIT? 1 → 读取控压模块当前单位
4. 以设备当前单位为准（若本地配置不同则采用设备单位）
5. 发送 PRESsure0? → 读取当前压力（响应格式: "值,单位"）
6. 通知上层：单位信息 + 当前压力值
```

---

## 三、完整命令参考

通用 SCPI 实现完整命令集，按功能分为 13 个分组。

### 3.1 IEEE 488.2 共同指令（4 个）

所有 SCPI 设备必须支持的通用命令。

| 命令 | 功能 | 类型 |
|------|------|------|
| `*IDN?` | 设备标识查询 | 查询 |
| `*RST` | 设备复位 | 设置 |
| `*CLS` | 清除状态寄存器和错误队列 | 设置 |
| `*TST?` | 设备自检 | 查询 |

**`*IDN?` 响应格式**：`厂商,型号,序列号,版本`

示例：`ADDITEL,,123456789,P25d&MPC V2.0.0.6`

解析：按逗号拆分为 4 段。

---

### 3.2 压力单位命令（3 个）

#### 设置压力单位 — `PRESsure:MODule<n>:UNIT <code>`

**格式**：`PRESsure:MODule<模块ID>:UNIT <数字代码>`

**示例**：`PRESsure:MODule1:UNIT 1133`（设为 kPa）

**说明**：设置类命令，发送后不等待响应。`<n>` 为模块编号，通常用 1（控制模块）。

#### 查询压力单位 — `PRESsure:MODule:UNIT? <模块ID>`

**格式**：`PRESsure:MODule:UNIT? <模块ID>`

**示例**：`PRESsure:MODule:UNIT? 1`

**响应**：数字代码字符串，如 `1132`

#### 查询单位列表 — `PRESsure:MODule:UNIT:LIST?`

**格式**：`PRESsure:MODule:UNIT:LIST?`

**响应**：设备支持的所有压力单位列表

#### 通用 SCPI 单位数字代码表（标准代码体系）

| 数字代码 | 单位 | 说明 |
|----------|------|------|
| 1130 | Pa | 帕斯卡 |
| 1133 | kPa | 千帕 |
| 1132 | MPa | 兆帕 |
| 1105 | bar | 巴 |
| 1104 | mbar | 毫巴 |
| 1141 | psi | 磅/平方英寸 |
| 1145 | kgf/cm² | 千克力/平方厘米 |
| 1134 | mmHg | 毫米汞柱 |
| 1135 | atm | 标准大气压 |

**跨设备对比**：

| 单位 | 通用SCPI | ConST820 | SPC4000 |
|------|----------|----------|---------|
| MPa | 1132 | 2 | 36 |
| kPa | 1133 | 1 | 22 |
| psi | 1141 | 3 | 1 |

**单位响应解析伪代码**：

```
function parseUnit(response):
    code = response.trim()

    // 数字代码映射
    codeMap = {
        "1130": PA, "1133": KPA, "1132": MPA,
        "1141": PSI, "1145": KGF_CM2  // 1145 实际为 kgf/cm2
    }
    if code in codeMap: return codeMap[code]

    // 兼容字符串格式
    upper = code.toUpperCase()
    switch upper:
        "PA", "1130"    → PA
        "KPA", "1133"   → KPA
        "MPA", "1132"   → MPA
        "BAR"           → BAR
        "PSI", "1141"   → PSI
        default         → MPA  // 默认
```

---

### 3.3 压力测量命令（3 个）

#### 读取控压模块当前压力 — `PRESsure0?`

**发送**：`PRESsure0?`

**响应格式**：`数值,单位,附加数据...`

**响应示例**：
- `0.566,MPa`
- `-0.05,kPa,other_data`

**解析伪代码**：

```
function parsePressure(response):
    if response is empty: return null

    parts = response.split(",")
    if parts.length >= 2:
        value = parseFloat(parts[0])
        unit = parts[1]  // 单位字符串
        if isNaN(value): return null
        return { value, unit }
    // 注意：忽略逗号第 3 段及之后的数据

    return null  // 格式不正确
```

#### 读取指定模块测量值 — `PRESsure:MODule<n>:MEASure?`

**格式**：`PRESsure:MODule<模块ID>:MEASure?`

**说明**：读取指定压力模块的测量值（区别于 `PRESsure0?` 读取控压模块输出）。

#### 读取压力稳定状态 — `PRESsure:MODule<n>:STABle?`

**格式**：`PRESsure:MODule<模块ID>:STABle?`

**默认模块**：模块 1（控制模块）

**响应**：`0` = 不稳定，`1` = 稳定

**子类覆盖情况**：
- ConST820 → 覆盖为 `OUTPut:PRESsure:STABle?`
- ConST860 → 覆盖为 `PRESsure:STABle?`
- SPC4000 → 无硬件稳定查询（软件判断）

---

### 3.4 压力输出控制命令（6 个）

#### 设置目标压力 — `PRESsure:TARGet <value>`

**发送**：`PRESsure:TARGet <压力值>`

**示例**：
- `PRESsure:TARGet 100.5`
- `PRESsure:TARGet -50`

**说明**：打压核心命令。设置类命令，发送后不等待响应。如果输出未启用需先启用。

**子类覆盖**：
- ConST820 → `SOURce:PRESsure <val>`
- SPC4000 → `GP <val>` / `GN <val>`

#### 查询目标压力 — `PRESsure:TARGet?`

#### 查询目标值范围 — `PRESsure:TARGet:RANGe?`

**响应格式**：`最小值,最大值,单位`

**示例**：`0,73.5,MPa`

**解析伪代码**：

```
function parseTargetRange(response):
    parts = response.split(",")
    if parts.length >= 3:
        low  = parseFloat(parts[0])
        high = parseFloat(parts[1])
        unit = parts[2]
        if not isNaN(low) and not isNaN(high):
            return { low, high, unit }
    return null
```

#### 设置压力控制模式 — `PRESsure:MODE <STATE>`

**发送**：`PRESsure:MODE <STATE>`

**参数**：`CONTROL`、`MEASURE`、`VENT`

**子类覆盖**：
- ConST820 → `OUTPut:PRESsure:MODE <STATE>`
- ConST860 → `PRESsure:MODule:CONTrol <STATE>`

#### 查询压力控制模式 — `PRESsure:MODE?`

**响应**：`CONTROL` / `MEASURE` / `VENT`

**解析**：`CONTROL` → 输出启用；其他 → 输出未启用

#### 模块控制状态 — `PRESsure:MODule:CONTrol` / `CONTrol?`

**说明**：功能上类似 `PRESsure:MODE`，但针对特定模块。ConST860 使用此格式。

---

### 3.5 输出控制（2 个）

输出控制复用 `PRESsure:MODE` 命令族：

```
启用输出 → PRESsure:MODE CONTROL
关闭输出 → PRESsure:MODE MEASURE
查询状态 → PRESsure:MODE?  → "CONTROL"=已启用
```

---

### 3.6 排空命令（3 个）

#### 执行排空 — `PRESsure:MODE VENT`

**发送**：`PRESsure:MODE VENT`

**超时**：2 倍默认超时（排空物理时间更长）

#### 配置排空压力值 — `PRESsure:Vent <value>`

**发送**：`PRESsure:Vent <压力值>`

**说明**：设置排空的目标压力值（默认为大气压）。

#### 查询排空压力值 — `PRESsure:Vent?`

---

### 3.7 设定点限制命令（4 个）

限制压力输出范围，防止超压损坏被测设备。

#### 设置限制范围 — `PRESsure:PLIMit <下限>,<上限>`

**示例**：`PRESsure:PLIMit 0,70`

#### 查询限制范围 — `PRESsure:PLIMit?`

**响应格式**：`下限,上限,单位`

**示例**：`0.005,70,MPa`

**解析伪代码**：

```
function parsePressureLimit(response):
    parts = response.split(",")
    if parts.length >= 3:
        lower = parseFloat(parts[0])
        upper = parseFloat(parts[1])
        unit  = parts[2]
        if not isNaN(lower) and not isNaN(upper):
            return { lower, upper, unit }
    return null
```

#### 启用/禁用限制 — `PRESsure:PLIMit:ENABle <0|1>`

**参数**：`1` = 启用，`0` = 禁用

#### 查询限制启用状态 — `PRESsure:PLIMit:ENABle?`

---

### 3.8 压力模块信息命令（7 个）

| 命令 | 功能 | 响应格式 |
|------|------|---------|
| `PRESsure:MODule<n>:ONLIne?` | 模块在线状态 | 0 或 1 |
| `PRESsure:MODule<n>:INFO?` | 模块详细信息 | `序列号,量程,压力类型,版本,精度` |
| `PRESsure:MODule<n>:PTYPe?` | 模块压力类型 | G（表压）/ A（绝压）/ D（差压） |
| `PRESsure:MODule<n>:RANGe?` | 模块量程 | 量程值 |
| `PRESsure:MODUle:VALUes?` | 所有模块压力值 | 多模块复合数据 |
| `PRESsure:MODule<n>:FILTer` | 设置/查询滤波 | 使能(0/1), 类型(0/1), 参数 |
| `PRESsure:MODule<n>:MULTirange?` | 是否多量程模块 | 0 或 1 |

### 3.9 量程管理命令（5 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:RANGe:LIST?` | 多量程列表 |
| `PRESsure:RANGe:INDEx` | 设置/查询当前量程索引 |
| `PRESsure:RANGe:MODE` | 设置/查询量程模式 |

### 3.10 压力类型命令（2 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:TYPE` | 设置当前压力类型（G/A/D） |
| `PRESsure:TYPE?` | 查询当前压力类型及是否支持表绝压切换 |

### 3.11 当前模块命令（2 个）

| 命令 | 功能 |
|------|------|
| `PRESsure:MODule` | 设置当前使用的控压模块 |
| `PRESsure:MODule?` | 查询当前控压模块编号 |

### 3.12 手动阶跃命令（3 个）

用于手动微调压力值：

| 命令 | 功能 |
|------|------|
| `PRESsure:STEP <step>` | 设置阶跃步进值 |
| `PRESsure:STEP?` | 查询当前步进值 |
| `PRESsure:STEP:UP` | 向上阶跃一个步进值 |
| `PRESsure:STEP:DOWN` | 向下阶跃一个步进值 |

### 3.13 系统错误查询（1 个）

**命令**：`SYSTem:ERRor?`

**响应格式**：`错误码,错误信息`

**示例**：
- `0,"No error"` → 无错误
- `-102,"Syntax error"` → 命令语法错误

每次查询从错误队列取出一个错误，循环查询直到返回 `0,"No error"` 可清空整个队列。

---

## 四、命令字符串构建

命令字符串构建的通用逻辑：

```
function buildCommandString(command):
    // 命令名直接使用命令类型枚举值
    cmdStr = command.type  // 如 "*IDN?"、"PRESsure:TARGet"

    if command 有参数:
        // 参数用逗号连接
        params = command.parameters.map(p => {
            if p 是字符串: return p
            if p 是数字:   return p.toString()
            if p 是布尔:   return p ? "ON" : "OFF"
        }).join(",")
        cmdStr += " " + params

    return cmdStr + 换行符
```

**注意**：
- 布尔值自动转换为 `ON`/`OFF`
- 参数之间用逗号分隔
- 子类（ConST860 等）可覆盖此方法实现设备特有的命令字符串格式

---

## 五、响应解析方法总览

| 方法 | 输入示例 | 输出 |
|------|---------|------|
| 压力解析 | `"0.566,MPa"` | `{ value: 0.566, unit: "MPa" }` |
| 标识解析 | `"厂商,型号,序列号,版本"` | `{ manufacturer, model, serialNumber, version }` |
| 目标范围解析 | `"0,73.5,MPa"` | `{ low: 0, high: 73.5, unit: "MPa" }` |
| 限幅解析 | `"0.005,70,MPa"` | `{ lower: 0.005, upper: 70, unit: "MPa" }` |
| 输出状态解析 | `"CONTROL"` / `"MEASURE"` | `true` / `false` |
| 单位解析 | `"MPa"` / `"1132"` | `PressureUnit.MPA` |

---

## 六、采集工作流

```
1. 设置目标压力
   ├── 若输出未启用 → PRESsure:MODE CONTROL
   └── PRESsure:TARGet <目标值>

2. 稳定等待循环（每 200ms）
   ├── PRESsure:MODule1:STABle?
   ├── "0" → 继续等待
   └── "1" → 设备报告稳定 → 额外等待 2000ms 确认持续稳定

3. 稳定确认 → 计量设备采集数据

4. 下一个压力点 或 PRESsure:MODE VENT（排空）
```

---

## 七、开发扩展指南

### 7.1 添加新的通用命令

```
1. 定义命令类型枚举值（如 "PRESSURE_NEW_FEATURE"）
2. 实现命令构建方法:
   buildNewFeatureCommand():
       return {
           type: NEW_FEATURE,
           isQuery: true,
           timeoutMs: 5000
       }
3. 在通用实现的已注册命令集合中添加此命令类型
4. (可选) 添加响应解析方法
```

### 7.2 创建新设备型号的策略

基于通用 SCPI 创建设备专用策略的模式：

```
1. 创建新策略类，继承通用 SCPI 实现类
2. 在构造函数中设置设备型号
3. 定义该设备已实现的命令集合
4. 覆盖需要定制的命令构建方法：
   - 如果命令格式与通用 SCPI 不同 → 实现设备专用格式
   - 如果命令不支持 → 覆盖并抛出明确错误

5. 在策略工厂中添加设备型号到策略的映射
```

**示例模式**（伪代码）：

```
class NewDeviceStrategy extends GenericScpiStrategy:
    constructor():
        super()
        this.deviceModel = "NEW_DEVICE"

    // 覆盖：设备用不同的单位命令格式
    buildSetUnitCommand(unit):
        return { type: "CUSTOM_UNIT", parameters: [unitMap[unit]] }

    // 覆盖：设备不支持限幅功能
    buildSetPressureLimitCommand(lower, upper):
        throw Error("此设备不支持压力限幅")
```

### 7.3 覆盖命令字符串构建

在子类中完全控制命令字符串：

```
buildCommandString(command):
    // 先检查是否有设备专用的命令格式映射
    if command.type in DEVICE_SPECIAL_FORMATS:
        format = DEVICE_SPECIAL_FORMATS[command.type]
        if command 有参数:
            return format + " " + 参数.join(",")
        return format

    // 回退到通用构建
    return super.buildCommandString(command)
```

### 7.4 调试未实现命令的回退

当设备策略调用未覆盖命令时，日志中会出现类似：

```
[命令回退] 设备 XXX 的命令 MODULE_FILTER 未实现
[命令回退] 原因: 命令未在子类命令集合中注册
[命令回退] 将使用父类通用命令
```

处理方式：
- 如果命令在设备上正常工作 → 将命令类型注册到子类的已实现集合中
- 如果设备不支持该命令 → 覆盖对应方法，返回错误或默认值

---

## 八、注意事项

1. **默认控压模块是模块 1（控制模块）**
2. **设置类命令不等待响应**：`PRESsure:TARGet`、`PRESsure:MODE VENT`、`PRESsure:MODule1:UNIT` 等
3. **命令串行执行**：同一 TCP 连接上的命令需通过互斥锁串行化，不能并发
4. **排空超时翻倍**：10000ms（2 × 5000ms）
5. **压力响应只取前两段**：`"0.566,MPa,extra"` → 解析时取 `0.566` 和 `MPa`，忽略后续
6. **单位代码为 4 位数字**：与 ConST820（0~4）和 SPC4000（1~36）体系完全不同
7. **命令名字符串直接使用枚举值**：`*IDN?` 就是发送的字符串，无需额外转换
8. **布尔参数自动转换为 ON/OFF**：`true` → `"ON"`，`false` → `"OFF"`
9. **输出状态解析**：只有 `CONTROL` 返回启用，`MEASURE` 和 `VENT` 都视为未启用
10. **此策略同时是回退策略**：所有未实现专用策略的设备型号都会回退到此通用实现
