---
name: spc4000-pressure-device
description: >
  Scanivalve SPC4000 压力校准器通信协议参考，涵盖 Mensor 兼容命令集、
  正负压分流控制、软件稳定判断算法和单位代码映射。
  用于编写、调试、修改 SPC4000 打压设备驱动代码。
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

**核心特性**：SPC4000 **完全不遵循 SCPI 协议**，与其它打压设备有 3 个根本性差异：

1. **没有硬件稳定查询** — 必须软件循环判断
2. **正负压分流** — 正压 `GP`，负压 `GN`
3. **没有独立输出启用** — GP/GN 自动进入控制模式

跨设备对比详见 [REFERENCE.md](../REFERENCE.md)。

---

## 连接流程

```
1. 建立 TCP 连接
2. 发送 ID? → 获取设备标识
3. 发送 Units? → 读取当前单位
4. 以设备单位为准
5. 发送 RP → 读取当前压力
6. 通知上层
```

---

## 命令参考

### ID? — 设备标识查询

**命令**：`ID?`（不是 SCPI 的 `*IDN?`）

---

### 单位命令

#### 查询 — `Units?`

响应：数字代码

#### 设置 — `Units <code>`

设置类命令，不等待响应。

#### 单位代码映射表

| 代码 | 单位 | 代码 | 单位 |
|------|------|------|------|
| 23 | Pa | 14 | bar |
| 22 | kPa | 15 | mbar |
| 36 | MPa | 19 | mmHg |
| 1 | psi | 13 | atm |
| 26 | kgf/cm² | | |

> **关键**：SPC4000 的数字代码体系与所有其他设备都不同，不可混用。

**示例**：
- `Units 22` → kPa
- `Units 36` → MPa
- `Units 1` → psi

**解析**：
```
function parseUnit(response):
    code = parseInt(response.trim())
    codeToUnit = {
        1: PSI, 13: ATM, 14: BAR, 15: MBAR,
        19: MMHG, 22: KPA, 23: PA, 26: KGF_CM2, 36: MPA
    }
    if code in codeToUnit: return codeToUnit[code]
    return normalizeUnitToken(response)
```

---

### RP — 读取当前压力

**命令**：`RP`（不带问号的隐式查询）

响应：`数值` 或 `数值,单位`

**解析**：
```
function parsePressure(response):
    parts = response.trim().split(",").filter(非空)
    value = parseFloat(parts[0])
    if isNaN(value): return null
    unit = parts.length > 1 ? normalizeUnitToken(parts[1]) : 当前设备单位
    return { value, unit }
```

---

### GP / GN — 正负压分流控制

SPC4000 没有统一设压命令，正负压不同：

| 条件 | 命令 | 示例 |
|------|------|------|
| ≥0 | `GP <绝对值>` | `GP 100.5` |
| <0 | `GN <绝对值>` | `GN 50` |

**关键特性**：
- GP/GN **自动进入控制模式**，无需先启用输出
- 尝试单独启用输出应抛出错误
- 设置类命令，不等待响应

**命令选择逻辑**：
```
function buildSetPressureCommand(targetPressure):
    command = targetPressure >= 0 ? "GP" : "GN"
    send(command + " " + abs(targetPressure))
```

---

### Measure — 测量模式

| 操作 | 命令 |
|------|------|
| 进入测量模式 | `Measure` |
| 查询测量模式 | `Measure?` |

响应：`YES`/`NO`、`ON`/`OFF`、`1`/`0`

---

### Mode? — 查询模式

**命令**：`Mode?`

响应：`CONTROL`、`MEASURE`、`VENT`、`STANDBY` 等

**输出状态解析**：
```
function parseOutputState(response):
    upper = response.trim().toUpperCase()
    if upper in ["YES","ON","1","MEASURE","VENT","STANDBY"]: return false
    if upper in ["NO","OFF","0","CONTROL"]: return true
    return false
```

---

### Vent — 排空

**命令**：`Vent`（超时建议 10000ms）

---

### 系统错误查询

**命令**：`SYSTem:ERRor?`

响应：`错误码,错误信息`

---

## 软件稳定判断算法

SPC4000 **没有硬件稳定查询**，必须软件计算：

```
function waitForStable(targetPressure, thresholdPercent):
    stableStartTime = null
    loop every 200ms:
        currentPressure = sendAndParse("RP")
        deviation = abs(currentPressure - targetPressure)
        threshold = max(abs(targetPressure) * thresholdPercent / 100, 0.001)

        if deviation <= threshold:
            if stableStartTime == null:
                stableStartTime = now()
            else if now() - stableStartTime >= 2000ms:
                return STABLE
        else:
            stableStartTime = null
        sleep(200ms)
```

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `thresholdPercent` | 稳定阈值百分比 | 0.1% |
| 稳定持续时间 | 需在阈值内持续 | 2000ms |
| 轮询间隔 | 读取压力间隔 | 200ms |
| 最小阈值 | 防止目标为 0 时除零 | 0.001 |

---

## 控制状态限制

```
MEASURE → "Measure"           ✓
VENT    → "Vent"              ✓
CONTROL → 不可单独设置         ✗ 必须通过 GP/GN 附带进入
```

---

## 开发扩展

### 添加新命令
- 使用扁平格式（非 SCPI 树形）
- 确定 `?` 后缀（查询）或无（设置）

### 固件升级改用硬件稳定
- 添加 `STABle?` 等查询命令
- 保留软件判断为回退方案

### 添加新单位
- 查文档获取数字代码
- 更新代码→单位和单位→代码映射表

---

## 注意事项

1. **没有硬件稳定查询** — 完全依赖软件判断
2. **正负压命令不同** — `GP`/`GN`，参数始终传绝对值
3. **不能单独启用输出** — GP/GN 自动进入控制模式
4. **RP 不是查询命令** — 不带问号但返回数据
5. **标识查询用 `ID?`** — 不是 `*IDN?`
6. **单位代码体系独立** — 不能与其他设备混用
7. **响应格式不统一** — RP 可能返回纯数字或 `"值,单位"`
8. **超时 5000ms** — 比其他设备长
9. **Mode? 响应值多样** — 需覆盖 CONTROL/MEASURE/VENT/STANDBY
10. **命令串行执行**，排空超时翻倍
