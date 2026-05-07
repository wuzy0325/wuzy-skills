# 打压设备 — 跨设备通用参考

所有打压设备共享相同的 TCP/IP 通信基础（端口 5025，分隔符 `\n`），在驱动层遵循策略继承模式。

---

## 架构设计：策略继承模式

```
抽象基类 PressDeviceStrategy（定义命令构建 + 响应解析接口）
  └── 通用 SCPI 实现 GenericScpiStrategy（注册全部标准 SCPI 命令集）
        ├── ConST820（覆盖 11 个命令，简化 SCPI）
        ├── ConST860（覆盖 14 个命令，标准 SCPI + 简化格式）
        └── SPC4000（覆盖 12 个命令，非 SCPI / Mensor 协议）
```

### 命令回退机制

当设备专用策略未覆盖某命令时：
1. 检查专用策略是否实现了该命令
2. 如果未实现 → 记录回退日志（含设备型号、命令类型、原因）
3. 自动执行父类的通用 SCPI 实现
4. 通知所有注册的回退事件监听器

---

## 跨设备命令对比

| 功能 | 通用 SCPI | ConST820 | ConST860 | SPC4000 |
|------|-----------|----------|----------|---------|
| 标识查询 | `*IDN?` | `*IDN?` | `*IDN?` | `ID?` |
| 查询单位 | `PRESsure:MODule:UNIT? 1` | `UNIT?` | `PRESsure:MODule:UNIT? 1` | `Units?` |
| 设置单位 | `PRESsure:MODule1:UNIT <1132>` | `UNIT:PRESsure 2` | `PRESsure:MODule:UNIT 1,kPa` | `Units 36` |
| 单位参数类型 | 数字代码 1130~1145 | 数字代码 0~4 | 单位字符串 kPa/MPa... | 数字代码 1~36 |
| 读取压力 | `PRESsure0?` | `MEASure:SCALar:PRESsure1?` | `PRESsure?` | `RP` |
| 压力响应格式 | `"值,单位"` | 纯数值 | `"值,单位"` | `"值"` 或 `"值,单位"` |
| 设置目标压力 | `PRESsure:TARGet <v>` | `SOURce:PRESsure <v>` | `PRESsure:TARGet <v>` | `GP <v>` / `GN <v>` |
| 稳定查询 | `PRESsure:MODule1:STABle?` | `OUTPut:PRESsure:STABle?` | `PRESsure:STABle?` | **无（软件判断）** |
| 启用输出 | `PRESsure:MODE CONTROL` | `OUTPut:PRESsure:MODE CONTROL` | `PRESsure:MODule:CONTrol CONTROL` | GP/GN 自动 |
| 关闭输出 | `PRESsure:MODE MEASURE` | `OUTPut:PRESsure:MODE MEASURE` | `PRESsure:MODule:CONTrol MEASURE` | `Measure` |
| 排空 | `PRESsure:MODE VENT` | `OUTPut:PRESsure:MODE VENT` | `PRESsure:MODule:CONTrol VENT` | `Vent` |
| 查询模式 | `PRESsure:MODE?` | `OUTPut:PRESsure:MODE?` | `PRESsure:MODule:CONTrol?` | `Mode?` |

---

## 跨设备单位代码对比

| 单位 | 通用 SCPI | ConST820 | ConST860 | SPC4000 |
|------|-----------|----------|----------|---------|
| Pa | 1130 | 0 | Pa (字符串) | 23 |
| kPa | 1133 | 1 | kPa | 22 |
| MPa | 1132 | 2 | MPa | 36 |
| psi | 1141 | 3 | psi | 1 |
| kgf/cm² | 1145 | 4 | kgf/cm2 | 26 |
| bar | 1105 | ✗ | bar | 14 |
| mbar | 1104 | ✗ | mbar | 15 |
| mmHg | 1134 | ✗ | mmHg | 19 |
| atm | 1135 | ✗ | atm | 13 |

**ConST860** 使用单位字符串（如 `kPa`），其余设备使用数字代码，且各设备的代码体系完全不同，不可混用。

---

## 公共采集工作流

所有打压设备的采集流程遵循同一模式，仅在具体命令和稳定判断方式上不同：

```
1. 设置目标压力（单位以设备实际单位为准）
   ├── 若输出未启用 → 发送启用输出命令
   └── 发送设置目标压力命令

2. 稳定等待循环（每 200ms 轮询）
   ├── 有硬件稳定查询 → 轮询状态直到 "1"
   │    稳定后再额外等 2000ms 确认持续稳定
   └── 无硬件稳定查询 → 软件计算偏差
        └── 偏差 ≤ 阈值 且 持续 ≥ 2000ms → 判定稳定

3. 稳定确认 → 计量设备采集数据

4. 下一个压力点 或 排空（排空超时为普通超时的 2 倍）
```

---

## 通用注意事项

| 条目 | 说明 |
|------|------|
| 命令串行执行 | 同一 TCP 连接上的命令需通过互斥锁串行化，不能并发发送 |
| 设置类命令不等待响应 | `PRESsure:TARGet`、`MODE`、`UNIT` 等设置命令发送后不阻塞等待设备响应 |
| 排空超时翻倍 | 排空操作物理时间较长，超时应为普通命令的 2 倍 |
| 连接后先读标识 | 先发 `*IDN?`（或对应标识命令）确认设备在线和型号 |
| 单位以设备为准 | 连接后先读取设备当前单位，若与本地配置不同则以设备单位为准 |
| 命令分隔符 | TCP 发送时自动追加 `\n`，不要在命令字符串中手动添加 |
