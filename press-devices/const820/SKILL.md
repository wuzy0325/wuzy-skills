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

**核心特性**：ConST820 使用简化的命令格式，用扁平命令名替代标准 SCPI 层级命令。跨设备对比详见 [REFERENCE.md](../REFERENCE.md)。

---

## 连接流程

```
1. 建立 TCP 连接
2. 发送 *IDN? → 获取设备标识
3. 发送 UNIT? → 读取设备当前压力单位
4. 以设备单位为准
5. 发送 MEASure:SCALar:PRESsure1? → 读取当前压力
6. 通知上层
```

断开时先关闭输出再断开 TCP。

---

## 命令参考

### 设备标识查询

**命令**：`*IDN?`

响应：`厂商,型号,序列号,版本`

---

### 单位命令

#### 查询 — `UNIT?`

响应：数字代码 `0` ~ `4`

#### 设置 — `UNIT:PRESsure <code>`

设置类命令，发送后不等待响应。

#### 单位代码映射

| 代码 | 单位 | 代码 | 单位 |
|------|------|------|------|
| 0 | Pa | 3 | psi |
| 1 | kPa | 4 | kgf/cm² |
| 2 | MPa | | |

> ConST820 **仅支持以上 5 种单位**，不支持 bar/mbar/mmHg/atm。

**示例**：
- `UNIT:PRESsure 1` → kPa
- `UNIT:PRESsure 2` → MPa
- `UNIT:PRESsure 3` → psi

**解析**：
```
function parseUnit(response):
    codeMap = {"0": PA, "1": KPA, "2": MPA, "3": PSI, "4": KGF_CM2}
    if response in codeMap: return codeMap[response]

    upper = response.toUpperCase()
    switch upper:
        "PA" → PA; "KPA" → KPA; "MPA" → MPA
        "PSI" → PSI; "KGF/CM2", "KG/CM2" → KGF_CM2
        default → MPA
```

---

### 读取当前压力

**命令**：`MEASure:SCALar:PRESsure1?`

**响应**：纯数值字符串（不含单位），如 `0.566`

> 与通用 SCPI 不同，ConST820 压力响应不含单位，需自行关联已保存的单位。

**解析**：
```
function parsePressure(response):
    value = parseFloat(response.trim())
    if isNaN(value): return null
    return { value, unit: currentDeviceUnit }
```

---

### 设置目标压力

**命令**：`SOURce:PRESsure <value>`

设置类命令，发送后不等待。输出未启用时需先启用。

---

### 查询稳定状态

**命令**：`OUTPut:PRESsure:STABle?`

响应：`0`=不稳定，`1`=稳定

---

### 输出控制

| 操作 | 命令 |
|------|------|
| 启用输出 | `OUTPut:PRESsure:MODE CONTROL` |
| 关闭输出 | `OUTPut:PRESsure:MODE MEASURE` |
| 排空 | `OUTPut:PRESsure:MODE VENT` |
| 查询模式 | `OUTPut:PRESsure:MODE?` |

`CONTROL`=启用，其他=未启用。

---

### 系统错误查询

**命令**：`SYSTem:ERRor?`

响应：`错误码,错误信息`。循环查询到 `0,"No error"` 清空队列。

---

### IEEE 488.2 通用命令

| 命令 | 功能 |
|------|------|
| `*IDN?` | 标识查询 |
| `*RST` | 设备复位 |
| `*CLS` | 清除状态 |

---

## 开发扩展

### 添加新命令

```
1. 确认固件是否支持，格式是否与通用 SCPI 不同
2. 格式相同 → 直接使用通用 SCPI 命令格式
3. 格式不同 → 实现 ConST820 专用版本
```

### 调试建议

- 抓 TCP 通信日志，确认收发命令字符串
- 先发 `*IDN?` 确认设备在线
- 单位不匹配时用 `UNIT?` 确认
- 压力读数异常时确认单位一致性

---

## 注意事项

1. **仅支持 5 种单位**：不支持 bar/mbar/mmHg/atm
2. **压力响应不含单位**，需自行关联
3. **设置类命令不等待响应**
4. **命令串行执行**，需互斥锁
5. **排空超时加长**（普通命令 2 倍）
