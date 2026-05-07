---
name: const820-pressure-device
description: >
  康斯特 ConST820 智能压力控制器通信协议参考，涵盖简化 SCPI 命令集、
  单位代码映射、响应解析规则和开发扩展指南。
  用于编写、调试、修改 ConST820 打压设备驱动代码。
---

# 康斯特 ConST820 打压设备

## 设备概述

| 属性 | 值 |
|------|-----|
| 厂商 | 康斯特 (ConST) |
| 型号 | ConST820 智能压力控制器 |
| 协议 | SCPI 简化命令集 |
| 物理连接 | TCP/IP，默认端口 5025 |
| 命令分隔符 | 换行符 `\n` (LF) |
| 推荐超时 | 3000ms（排空 6000ms） |

**核心特性**：ConST820 使用简化的命令格式，与标准 SCPI 树形层级命令有显著差异。主要用扁平命令名替代标准层级命令。

---

## 一、连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识
3. 发送 UNIT? → 读取设备当前压力单位
4. 对比本地配置单位与设备单位，以设备单位为准
5. 发送 MEASure:SCALar:PRESsure1? → 读取当前压力
6. 通知上层：单位信息 + 当前压力值
```

断开时先关闭输出再断开 TCP。

---

## 二、命令参考

### 2.1 设备标识查询

**命令**：`*IDN?`

**响应格式**：`厂商,型号,序列号,版本`

**响应示例**：`ADDITEL,,123456789,P25d&MPC V2.0.0.6`

**解析**：按逗号拆分为 4 段，依次为厂商、型号、序列号、固件版本。

---

### 2.2 单位命令

ConST820 使用简化的单位命令格式，与其他 SCPI 设备不同。

#### 查询当前单位 — `UNIT?`

**发送**：`UNIT?`

**响应**：数字代码 `0` ~ `4`

#### 设置单位 — `UNIT:PRESsure <code>`

**发送**：`UNIT:PRESsure <code>`

**说明**：设置类命令，发送后不等待响应。

#### 单位代码映射表

| 代码 | 单位 | 说明 |
|------|------|------|
| 0 | Pa | 帕斯卡 |
| 1 | kPa | 千帕 |
| 2 | MPa | 兆帕 |
| 3 | psi | 磅/平方英寸 |
| 4 | kgf/cm² | 千克力/平方厘米 |

> **限制**：ConST820 仅支持以上 5 种单位。不支持 bar、mbar、mmHg、atm。设置不支持的单位会失败。

**示例**：
- 设为 kPa → `UNIT:PRESsure 1`
- 设为 MPa → `UNIT:PRESsure 2`
- 设为 psi → `UNIT:PRESsure 3`

**单位响应解析逻辑**（伪代码）：

```
function parseUnit(response):
    // 优先按数字代码匹配
    codeMap = {"0": PA, "1": KPA, "2": MPA, "3": PSI, "4": KGF_CM2}
    if response in codeMap:
        return codeMap[response]

    // 回退到字符串匹配
    upper = response.toUpperCase()
    switch upper:
        "PA" → PA
        "KPA" → KPA
        "MPA" → MPA
        "PSI" → PSI
        "KGF/CM2", "KG/CM2" → KGF_CM2
        default → 默认 MPa
```

---

### 2.3 读取当前压力

**命令**：`MEASure:SCALar:PRESsure1?`

**响应**：纯数值字符串（不含单位）

**响应示例**：`0.566`、`100.5`、`-0.023`

> **注意**：与通用 SCPI 设备不同，ConST820 的压力响应不含单位。程序需使用之前通过 `UNIT?` 查询并保存的单位。

**解析伪代码**：

```
function parsePressure(response):
    value = parseFloat(response.trim())
    if isNaN(value): return null
    return { value, unit: currentDeviceUnit }
```

---

### 2.4 设置目标压力（打压）

**命令**：`SOURce:PRESsure <value>`

**发送示例**：
- 打压到 100.5 → `SOURce:PRESsure 100.5`
- 打压到 -50 → `SOURce:PRESsure -50`

**说明**：
- 设置类命令，发送后不等待设备响应
- 如果输出未启用，需先启用输出再发此命令
- 打压到负压时直接传负数值

> **与通用 SCPI 的差异**：通用 SCPI 用 `PRESsure:TARGet`，ConST820 用 `SOURce:PRESsure`。

---

### 2.5 查询压力稳定状态

**命令**：`OUTPut:PRESsure:STABle?`

**响应**：`0` = 不稳定，`1` = 稳定

**用途**：采集流程中轮询此状态，直到设备报告稳定后再等待 2000ms 确认持续稳定。

> **与通用 SCPI 的差异**：通用 SCPI 用 `PRESsure:MODule1:STABle?`。

---

### 2.6 输出控制

ConST820 输出控制统一使用 `OUTPut:PRESsure:MODE` 命令族。

#### 设置输出模式 — `OUTPut:PRESsure:MODE <mode>`

| 模式值 | 含义 | 用途 |
|--------|------|------|
| `CONTROL` | 控制模式 | 启用输出，设备主动调节压力 |
| `MEASURE` | 测量模式 | 关闭输出，仅监测压力 |
| `VENT` | 排空模式 | 释放压力到大气压 |

**示例**：
- 启用输出：`OUTPut:PRESsure:MODE CONTROL`
- 关闭输出：`OUTPut:PRESsure:MODE MEASURE`
- 排空：`OUTPut:PRESsure:MODE VENT`

#### 查询输出模式 — `OUTPut:PRESsure:MODE?`

**响应**：`CONTROL`、`MEASURE` 或 `VENT`

**解析**：`CONTROL` 表示输出已启用，其他状态表示输出未启用。

---

### 2.7 系统错误查询

**命令**：`SYSTem:ERRor?`

**响应格式**：`错误码,错误信息`

**示例**：`0,"No error"` 表示无错误。

每次查询从错误队列取出一个错误，循环查询直到返回 `0,"No error"` 可清空整个队列。

---

### 2.8 IEEE 488.2 通用命令

| 命令 | 功能 | 说明 |
|------|------|------|
| `*IDN?` | 设备标识查询 | 返回厂商、型号、序列号、版本 |
| `*RST` | 设备复位 | 恢复到初始上电状态 |
| `*CLS` | 清除状态 | 清除状态寄存器和错误队列 |

---

## 三、ConST820 与其它设备的命令对比

| 功能 | 通用SCPI | ConST820 | ConST860 | SPC4000 |
|------|----------|----------|----------|---------|
| 标识查询 | `*IDN?` | `*IDN?` | `*IDN?` | `ID?` |
| 查询单位 | `PRESsure:MODule:UNIT? 1` | **`UNIT?`** | `PRESsure:MODule:UNIT? 1` | `Units?` |
| 设置单位 | `PRESsure:MODule1:UNIT <1132>` | **`UNIT:PRESsure 2`** | `PRESsure:MODule:UNIT 1,MPa` | `Units 36` |
| 读取压力 | `PRESsure0?` | **`MEASure:SCALar:PRESsure1?`** | `PRESsure?` | `RP` |
| 设置目标 | `PRESsure:TARGet <v>` | **`SOURce:PRESsure <v>`** | `PRESsure:TARGet <v>` | `GP <v>` / `GN <v>` |
| 稳定查询 | `PRESsure:MODule1:STABle?` | **`OUTPut:PRESsure:STABle?`** | `PRESsure:STABle?` | 无 |
| 启用输出 | `PRESsure:MODE CONTROL` | **`OUTPut:PRESsure:MODE CONTROL`** | `PRESsure:MODule:CONTrol CONTROL` | GP/GN 自动 |
| 关闭输出 | `PRESsure:MODE MEASURE` | **`OUTPut:PRESsure:MODE MEASURE`** | `PRESsure:MODule:CONTrol MEASURE` | `Measure` |
| 排空 | `PRESsure:MODE VENT` | **`OUTPut:PRESsure:MODE VENT`** | `PRESsure:MODule:CONTrol VENT` | `Vent` |

---

## 四、采集工作流

```
1. 设置目标压力
   ├── 若输出未启用 → OUTPut:PRESsure:MODE CONTROL
   └── SOURce:PRESsure <目标值>

2. 稳定等待循环（每 200ms 轮询）
   ├── OUTPut:PRESsure:STABle?
   ├── 响应 "0" → 继续等待
   └── 响应 "1" → 设备报告稳定 → 额外等待 2000ms 确认持续稳定

3. 稳定确认 → 计量设备进行数据采集

4. 下一个压力点 或 排空
   └── OUTPut:PRESsure:MODE VENT（排空超时需翻倍）
```

---

## 五、开发扩展指南

### 5.1 添加新命令支持

当需要让 ConST820 支持一个新的操作时：

```
1. 确认 ConST820 的固件是否支持该命令，以及命令格式是否与通用 SCPI 不同
2. 如果格式相同 → 直接使用通用 SCPI 的命令格式
3. 如果格式不同 → 实现 ConST820 专用版本:
   a. 确定命令发送字符串（如 "CUSTOM:COMMAND?"）
   b. 确定响应格式和解析规则
   c. 如果是设置类命令，标记为不等待响应
```

### 5.2 添加新单位支持

如果 ConST820 固件更新后支持了新的压力单位：

```
1. 在单位代码映射表中添加新的代码→单位映射项
2. 在响应解析中处理新代码
3. 同时更新代码→单位和单位→代码的正反向映射
```

### 5.3 调试建议

- 抓取 TCP 通信日志，确认发送的命令字符串和接收的原始响应
- 故障排查时先用手动 `*IDN?` 确认设备在线
- 单位不匹配时先用 `UNIT?` 确认设备当前单位
- 压力读数异常时确认单位是否一致

---

## 六、注意事项

1. **仅支持 5 种单位**：Pa、kPa、MPa、psi、kgf/cm²。bar/mbar/mmHg/atm 不支持
2. **压力响应不含单位**：`MEASure:SCALar:PRESsure1?` 返回纯数值，需要自行关联单位
3. **设置类命令不等待响应**：`UNIT:PRESsure`、`SOURce:PRESsure`、`OUTPut:PRESsure:MODE` 等设置命令发送后不阻塞等待
4. **命令串行执行**：同一连接上不能并发发送命令，需要互斥锁保护
5. **排空超时需加长**：排空操作物理时间较长，超时设置应为普通命令的 2 倍
6. **控制模块默认为模块 1**：`PRESsure:MODule1:STABle?` 等带模块 ID 的命令默认使用模块 1
