---
name: "five-hole-probe-interpolation"
description: "五孔探针PRB插值算法：基于Ka/Kb压力系数进行区域1-9分类与凸四边形双线性插值，从5个压力值反算气流角度(α,β)、总压Pt、静压Ps、速度V和马赫数Ma。实现五孔探针数据后处理时调用，或用户提到五孔探针、PRB校准、移位测试、流场测量时触发。"
---

# 五孔探针 PRB 插值算法

## 1. 概述

五孔探针是一种用于测量三维流场的气动探针，包含 5 个压力孔：
- **4 个周向孔**（P1、P3、P4、P5，间隔 90° 均匀分布）
- **1 个中心孔**（P2）

通过 5 个压力值反算来流的**偏航角 α**、**俯仰角 β**、**总压 P0**、**静压 Ps**、**速度 V** 和**马赫数 Ma**。

算法采用**Ka/Kb 压力系数 + 区域1-9分类 + 凸四边形双线性插值**的策略：
- **Ka/Kb 压力系数**：将 5 维压力空间降维到 2 维系数空间
- **区域 1-9 分类**：在 Ka/Kb 空间将待测点定位到 9 个互斥区域之一
- **凸四边形双线性插值**：在区域 9（中心栅格区域）内做参量化双线性插值

---

## 2. PRB 校准文件格式

### 2.1 文件结构

```
13 13                                  ← 网格维度（固定）
ka_00 kb_00 cpt_00 cps_00  α_00  β_00  ← 第1行校准数据
ka_01 kb_01 cpt_01 cps_01  α_01  β_01  ← 第2行校准数据
...
ka_168 kb_168 cpt_168 cps_168 α_168 β_168 ← 第169行校准数据
```

| 特征 | 值 |
|------|-----|
| 网格维度 | 13 × 13（固定不变） |
| 数据行数 | 169 行（= 13 × 13） |
| 每行列数 | 6 列 |
| 列含义 | `ka` `kb` `cpt` `cps` `alpha` `beta` |
| α 范围 | −30° ~ 30°，步长 5° |
| β 范围 | −30° ~ 30°，步长 5° |

**注意**：PRB 文件中的 `ka`、`kb`、`cpt`、`cps` 是在校准过程中**预计算好的**，插值算法直接读取使用，不再从原始压力推导。

---

## 3. 探针孔位布局与系数定义

### 3.1 孔位排布

```
         P1 (0°)
           ▲
           │
    P5 ────┼──── P3
   (270°)  │  (90°)
           │
           ▼
         P4 (180°)
         
         P2 = 中心孔
```

| 孔号 | 周向位置 | 说明 |
|------|----------|------|
| P1 | 0° | 对应 β 方向（俯仰角，P1 与 P3 的压差） |
| P2 | 中心 | 中心孔，总压参考 |
| P3 | 90° | 对应 β 方向（俯仰角） |
| P4 | 180° | 对应 α 方向（偏航角，P4 与 P5 的压差） |
| P5 | 270° | 对应 α 方向（偏航角） |

### 3.2 压力系数定义

**算法中使用的 Ka/Kb 系数为固定公式（不分区域）**：

```
P_avg = (P1 + P3 + P4 + P5) / 4
   δ = P2 − P_avg          ← 参考压差（中心孔与周向平均的差值）

  Ka = (P4 − P5) / δ       ← α 方向压力系数（偏航角）
  Kb = (P3 − P1) / δ       ← β 方向压力系数（俯仰角）
```

---

## 4. 模块架构

```
┌──────────────────────────────────────────────────────────────┐
│                    五孔探针 PRB 插值模块                       │
├──────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ 输入参数处理  │→│ 核心插值计算  │→│  结果输出    │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│        │                 │                 │                 │
│        ▼                 ▼                 ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ PRB 文件加载  │  │ Ka/Kb 计算   │  │ α/β 角度    │       │
│  │ 数据验证      │  │ 区域 1-9 分类│  │ P0/Ps 压力  │       │
│  │ 网格索引构建  │  │ 凸四边形插值  │  │ V/Ma 气动   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
```

### 4.1 输入接口

```
PressureInput {
    P1, P2, P3, P4, P5: number   // 五个孔压力 (Pa)
    Patm: number                   // 大气压 (Pa)
    Tatm: number                   // 大气温度 (°C)
}
```

### 4.2 输出接口

```
InterpolationResult {
    alpha: number            // 偏航角 (°)
    beta: number             // 俯仰角 (°)
    machNumber: number       // 马赫数
    velocity: number         // 速度 (m/s)
    dynamicPressure: number  // 动压 Pt−Ps (Pa)
    density: number          // 空气密度 (kg/m³)
    P0: number               // 总压表压 (Pa)
    Ps: number               // 静压表压 (Pa)
    isValid: boolean         // 结果有效标志
    warning?: string         // 警告信息
}
```

---

## 5. 核心算法

### 5.1 主流程

```
FUNCTION 五孔探针PRB插值(压力数据, PRB校准表):
    // === 第1步: 输入验证 ===
    验证 P1~P5, Patm, Tatm 均为有限数值
    
    // === 第2步: 计算 Ka/Kb 压力系数 ===
    P_avg = (P1 + P3 + P4 + P5) / 4
    δ = P2 − P_avg
    IF |δ| < 1e-4:
        δ = sign(δ) × 1e-4    ← 防止零除，clamp 到最小压差
    Ka = (P4 − P5) / δ
    Kb = (P3 − P1) / δ
    
    // === 第3步: 区域 1-8 分类 ===
    point = 区域1_8分类(Ka, Kb, PRB校准表)
    
    IF point 未解析(α,β为空):
        // === 第4步: 区域 9 凸四边形插值 ===
        point = 区域9凸四边形插值(point, PRB校准表)
    
    断言 α,β 已解析为非空值
    
    // === 第5步: CPT/CPS 插值 ===
    CPT = 输出值插值(point, PRB校准表, selector=cpt)
    CPS = 输出值插值(point, PRB校准表, selector=cps)
    
    // === 第6步: 计算 Pt, Ps ===
    P0 = P2 − CPT × δ           // 总压（表压）
    Ps = P_avg − CPS × δ        // 静压（表压）
    
    // === 第7步: 计算速度, 马赫数 ===
    dp = |P0 − Ps|               // 动压
    T_K = Tatm + 273.15          // 开尔文温度
    P_static_abs = Patm + Ps     // 绝对静压
    ρ = P_static_abs / (287.06 × T_K)
    V = √(2 × dp / ρ)
    Ma = √(5 × |((P0 + Patm) / P_static_abs)^(0.4/1.4) − 1|)
    
    // === 第8步: 有效性校验 ===
    IF α或β超出[−30°, 30°]:
        标记为无效
    IF dp ≤ 0:
        标记为无效（总压低于静压，物理上不可能）
    IF ρ ≤ 0:
        标记为无效
    
    RETURN 构建结果(α, β, Ma, V, dp, ρ, P0, Ps, 有效性, 警告)
```

### 5.2 Ka/Kb 压力系数计算

```
FUNCTION 计算压力系数(P1, P2, P3, P4, P5):
    P_avg = (P1 + P3 + P4 + P5) × 0.25

    δ = P2 − P_avg

    // 防零除：当压差接近零时 clamp 到最小值
    IF |δ| < MIN_PRESSURE_DELTA:
        IF δ == 0:
            δ = MIN_PRESSURE_DELTA
        ELSE:
            δ = sign(δ) × MIN_PRESSURE_DELTA

    Ka = (P4 − P5) / δ
    Kb = (P3 − P1) / δ

    RETURN { Ka, Kb, α: null, β: null }
```

**物理含义**：
- `Ka`：α 方向的压力不对称度（约 0° 时 Ka ≈ 0，大攻角时 |Ka| 增大）
- `Kb`：β 方向的压力不对称度（约 0° 时 Kb ≈ 0，大侧滑角时 |Kb| 增大）
- `δ`：来流动压的参考量（中心孔总压 − 周向平均静压）

### 5.3 区域 1-9 分类

#### 5.3.1 区域划分原理

PRB 校准数据在 Ka/Kb 空间中构成一个 13×13 的网格。当待测点的 (Ka, Kb) 落在网格范围内时（区域 9），进行标准双线性插值；落在网格外时（区域 1-8），通过邻近边界段的线性外推锚定到网格边缘的 α/β 值。

```
              Kb ↑
                 │
     ┌───────────┼───────────┐
     │ 区域7     │ 区域6     │ 区域5      ← 上方超出
     │ (角点)    │ (边界外推) │ (角点)      │
     ├───────────┼───────────┤
区域8 │           │           │ 区域4      ← α方向超出
(左)  │  区域9    │  区域9    │ (右)
     │ (网格内)  │ (网格内)  │
     ├───────────┼───────────┤
     │ 区域1     │ 区域2     │ 区域3      ← 下方超出
     │ (角点)    │ (边界外推) │ (角点)      │
     └───────────┼───────────┘
                 └────────────→ Ka
```

- **区域 1, 3, 5, 7**：四个角点 —— Ka 和 Kb 同时超出两个方向，直接锚定到网格角点
- **区域 2, 4, 6, 8**：四条边 —— 一个方向在范围内，另一个方向超出，沿边界线性外推
- **区域 9**：中心网格区域 —— Ka 和 Kb 均在网格凸包内，做凸四边形双线性插值

#### 5.3.2 区域 1-8 分类逻辑

```
FUNCTION 区域1_8分类(point, PRB校准表):
    // 获取四个角点（−30°, −30°/30°/30°/−30° + −30°/−30°/30°/30°）
    corner_NW = 表.get(−30, 30)   // 区域7: Ka < −30的Ka 且 Kb > 30的Kb
    corner_NE = 表.get(30, 30)    // 区域5: Ka > 30的Ka 且 Kb > 30的Kb
    corner_SW = 表.get(−30, −30)  // 区域1: Ka < −30的Ka 且 Kb < −30的Kb
    corner_SE = 表.get(30, −30)   // 区域3: Ka > 30的Ka 且 Kb < −30的Kb

    // ---- 角点判断 ----
    IF Ka < corner_SW.Ka AND Kb < corner_SW.Kb:
        RETURN {..., α: −30, β: −30}    // 区域1：直接锚定

    IF Ka > corner_SE.Ka AND Kb < corner_SE.Kb:
        RETURN {..., α: 30, β: −30}     // 区域3

    IF Ka > corner_NE.Ka AND Kb > corner_NE.Kb:
        RETURN {..., α: 30, β: 30}      // 区域5

    IF Ka < corner_NW.Ka AND Kb > corner_NW.Kb:
        RETURN {..., α: −30, β: 30}     // 区域7

    // ---- 边界外推 ----
    // 区域2: β=−30° 边界 (下方)，沿 α 方向外推
    boundary = 构建边界点集(校准表, β=−30° 且 Kb<0)
    IF Kb < boundary 范围内:
        FOR EACH 相邻边界段:
            IF Ka 在段的 Ka 范围内:
                factor = (Ka − 段起点.Ka) / (段终点.Ka − 段起点.Ka)
                α = 段起点.α + factor × (段终点.α − 段起点.α)
                RETURN {..., α, β: −30}

    // 区域4: α=30° 边界 (右侧)，沿 β 方向外推
    boundary = 构建边界点集(校准表, α=30° 且 Ka>0)
    IF Ka > boundary 范围内:
        FOR EACH 相邻边界段:
            IF Kb 在段的 Kb 范围内:
                factor = (Kb − 段起点.Kb) / (段终点.Kb − 段起点.Kb)
                β = 段起点.β + factor × (段终点.β − 段起点.β)
                RETURN {..., α: 30, β}

    // 区域6: β=30° 边界 (上方)，沿 α 方向外推
    boundary = 构建边界点集(校准表, β=30° 且 Kb>0)
    IF Kb > boundary 范围内:
        FOR EACH 相邻边界段:
            IF Ka 在段的 Ka 范围内:
                factor = (Ka − 段起点.Ka) / (段终点.Ka − 段起点.Ka)
                α = 段起点.α + factor × (段终点.α − 段起点.α)
                RETURN {..., α, β: 30}

    // 区域8: α=−30° 边界 (左侧)，沿 β 方向外推
    boundary = 构建边界点集(校准表, α=−30° 且 Ka<0)
    IF Ka < boundary 范围内:
        FOR EACH 相邻边界段:
            IF Kb 在段的 Kb 范围内:
                factor = (Kb − 段起点.Kb) / (段终点.Kb − 段起点.Kb)
                β = 段起点.β + factor × (段终点.β − 段起点.β)
                RETURN {..., α: −30, β}

    // 未解析（进入区域9）
    RETURN point（α,β保持null）
```

#### 5.3.3 边界点集构建

```
FUNCTION 构建边界点集(校准表, 筛选条件, 排序键):
    按排序键收集满足筛选条件的所有网格点
    按排序键升序排列（步长5°）
    RETURN [边界点1, 边界点2, ..., 边界点13]

    示例（区域2：β=−30°边界）:
        筛选条件: row.β == −30° AND row.Kb < 0
        排序键: row.α
        结果: 13个点，从 α=−30° 到 α=30°，步长5°
```

### 5.4 区域 9：凸四边形双线性插值

#### 5.4.1 四边形定位

```
FUNCTION 区域9凸四边形插值(point, PRB校准表):
    构建所有12×12=144个凸四边形
    // 每个四边形由4个相邻网格点定义:
    //   x1=(α,β), x2=(α+5°,β), x3=(α+5°,β+5°), x4=(α,β+5°)
    
    FOR EACH quad IN 144个四边形:
        // 将四边形顶点从(α,β)空间映射到(Ka,Kb)空间
        vertices = [x1→(Ka,Kb), x2→(Ka,Kb), x3→(Ka,Kb), x4→(Ka,Kb)]
        
        IF 点在凸四边形内(point, vertices):
            // 构造四边形的四条边线
            line12 = 通过vertices[0], vertices[1]的直线
            line23 = 通过vertices[1], vertices[2]的直线
            line34 = 通过vertices[2], vertices[3]的直线
            line41 = 通过vertices[3], vertices[0]的直线
            
            // β 方向插值：点到下边(line12)和上边(line34)的距离比
            d12 = 点到直线距离(point, line12)
            d34 = 点到直线距离(point, line34)
            β = 对边距离插值(x1.β, x4.β, d12, d34)
            
            // α 方向插值：点到左边(line41)和右边(line23)的距离比
            d41 = 点到直线距离(point, line41)
            d23 = 点到直线距离(point, line23)
            α = 对边距离插值(x1.α, x2.α, d41, d23)
            
            RETURN {..., α, β}

    // 未找到匹配四边形（点在网格外部，理论上不应到此）
    RETURN point（α,β仍为null）
```

#### 5.4.2 对边距离比插值公式

```
FUNCTION 对边距离插值(value_start, value_end, dist_to_start_edge, dist_to_end_edge):
    total = dist_to_start_edge + dist_to_end_edge
    IF total == 0:
        RETURN value_start
    RETURN value_start + (value_end - value_start) × dist_to_start_edge / total
```

**几何直觉**：在 (Ka, Kb) 空间中，PRB 标定网格的四边形通常近似为梯形或平行四边形。通过计算待测点到一对平行边的距离比，可以在角度空间做线性插值：

```
如果点离下边（β = −30° 对应边）更近 → β 更接近该边的 β 值
如果点离左边（α = −30° 对应边）更近 → α 更接近该边的 α 值
```

#### 5.4.3 点在凸四边形内判断

```
FUNCTION 点在凸四边形内(point, vertices[4]):
    FOR i = 0 TO 3:
        start = vertices[i]
        end = vertices[(i+1) % 4]
        
        // 构造边线
        line = 通过两点构造直线(start, end)
        
        // 取对边顶点作为参考点，确定"内侧"方向
        ref = vertices[(i+2) % 4]   // 对边顶点
        ref_sign = 有号距离(ref, line)
        
        point_sign = 有号距离(point, line)
        
        // 如果参考点和测试点在有号距离上的符号不同，则点在四边形外
        IF ref_sign > 0 AND point_sign < 0: RETURN FALSE
        IF ref_sign < 0 AND point_sign > 0: RETURN FALSE
    
    RETURN TRUE
```

### 5.5 输出值插值（CPT/CPS 获取）

在 α、β 确定后，对 CPT/CPS 做 α−β 空间的双线性插值：

```
FUNCTION 输出值插值(point, PRB校准表, selector):
    // 确定(α,β)所在的5°网格单元
    [α1, α2] = 确定α单元(point.α)
    [β1, β2] = 确定β单元(point.β)
    
    // 获取四个角点的值
    v11 = selector(表.get(α1, β1))   // 左下
    v21 = selector(表.get(α2, β1))   // 右下
    v12 = selector(表.get(α1, β2))   // 左上
    v22 = selector(表.get(α2, β2))   // 右上
    
    // α 方向的插值因子
    t_α = (point.α − α1) / (α2 − α1)
    
    // 先沿下边做 α 方向插值
    v_lower = v11 + t_α × (v21 − v11)
    // 再沿上边做 α 方向插值
    v_upper = v12 + t_α × (v22 − v12)
    
    // 再沿 β 方向插值
    t_β = (point.β − β1) / (β2 − β1)
    value = v_lower + t_β × (v_upper − v_lower)
    
    RETURN value
```

### 5.6 总压/静压/速度/马赫数计算

```
FUNCTION 计算气动参数(P1,P2,P3,P4,P5, CPT,CPS, Tatm, Patm):
    // === 计算压差 ===
    P_avg = (P1 + P3 + P4 + P5) / 4
    δ = P2 − P_avg
    
    // === 总压和静压（表压） ===
    P0 = P2 − CPT × δ      // 总压表压
    Ps = P_avg − CPS × δ   // 静压表压
    
    // === 绝对压力和温度 ===
    P0_abs = Patm + P0      // 绝对总压
    Ps_abs = Patm + Ps      // 绝对静压
    T_K = Tatm + 273.15     // 开尔文温度
    
    dp = P0 − Ps            // 动压
    
    // === 空气密度（理想气体状态方程） ===
    IF Ps_abs > 0 AND T_K > 0:
        ρ = Ps_abs / (287.06 × T_K)
    ELSE:
        ρ = NaN
    
    // === 速度（伯努利方程） ===
    IF dp > 0 AND ρ > 0:
        V = √(2 × dp / ρ)
    ELSE:
        V = 0
    
    // === 马赫数（等熵关系，γ=1.4） ===
    IF Ps_abs > 0:
        ratio = P0_abs / Ps_abs
        Ma = √(5 × |ratio^(0.4/1.4) − 1|)
    ELSE:
        Ma = 0
    
    RETURN { P0, Ps, dp, ρ, V, Ma }
```

### 5.7 有效性校验

```
FUNCTION 校验结果(α, β, P0, Ps, dp, ρ, V, Ma, 有效范围, Tatm, Patm):
    warnings = []
    isValid = TRUE
    
    // 输出非有限值检查
    IF α或β或V或Ma不是有限数:
        push警告("Interpolation returned non-finite outputs")
        isValid = FALSE
    
    // 动压检查
    IF dp ≤ 0:
        push警告("Resolved total pressure is below static pressure (pt < ps)")
        isValid = FALSE
    
    // 角度范围检查
    IF α 超出[有效范围.α_min, 有效范围.α_max]:
        push警告("Resolved alpha is outside the PRB table range")
        isValid = FALSE
    IF β 超出[有效范围.β_min, 有效范围.β_max]:
        push警告("Resolved beta is outside the PRB table range")
        isValid = FALSE
    
    // 物理参数检查
    IF T_K ≤ 0:
        push警告("Atmospheric temperature must be above absolute zero")
        isValid = FALSE
    IF Ps_abs ≤ 0:
        push警告("Resolved static pressure is not physically valid")
        isValid = FALSE
    IF ρ ≤ 0:
        push警告("Resolved air density is not physically valid")
        isValid = FALSE
    
    RETURN { isValid, warning }
```

---

## 6. 异常与边界处理

### 6.1 PRB 文件加载异常

| 异常情况 | 处理方式 |
|----------|----------|
| 文件不存在 | 抛出异常：PRB文件路径无效 |
| 文件为空 | 抛出异常：PRB file is empty |
| 首行维度不是 `13 13` | 抛出异常：PRB file header must be 13 13 |
| 数据行数 ≠ 169 | 抛出异常：PRB file must contain exactly 169 rows |
| 某行列数 ≠ 6 | 抛出异常：PRB row X must contain exactly 6 numeric columns |
| 包含非数值 | 抛出异常：PRB row X contains a non-numeric value |
| 包含非有限数值（NaN/Inf） | 抛出异常：Calibration row X field "xxx" must be a finite number |
| 网格点重复 | 抛出异常：Calibration table … duplicate entry |
| 网格点缺失 | 抛出异常：Calibration table is missing grid point (xx, xx) |
| 不支持的网格点坐标 | 抛出异常：Calibration row X has unsupported grid point |

### 6.2 运行时计算异常

| 异常情况 | 处理方式 |
|----------|----------|
| `|δ| < 1e-4` (压差过小) | clamp 到 `sign(δ) × 1e-4`，warning: "Reference pressure delta is near zero" |
| `δ == 0` 精确为零 | clamp 到 `1e-4`（正值） |
| 输入包含 NaN/Inf | 抛出异常：Pressure input "xxx" must be a finite number |
| (Ka, Kb) 落在网格外且无法匹配边界 | 抛出异常：Unable to resolve alpha/beta (ka=xxx, kb=xxx) |
| 四边形退化为直线（面积≈0） | 跳过该四边形继续搜索下一个 |
| PRB 未加载即调用 calculate | 抛出异常：PRB file not loaded |

### 6.3 计算结果异常

| 异常情况 | 处理方式 |
|----------|----------|
| α/β 非有限值 | isValid=false, warning |
| 动压 ≤ 0 | isValid=false, warning: "pt < ps" |
| α/β 超范围 | isValid=false, warning: "outside the PRB table range" |
| T ≤ 0K | isValid=false, warning |
| Ps_abs ≤ 0 | isValid=false, warning |
| ρ ≤ 0 | isValid=false, warning |
| 所有异常 | 返回所有字段=0, isValid=false, warning 包含异常信息 |

---

## 7. 完整伪代码汇总

```
常量:
    GRID_MIN = −30, GRID_MAX = 30, GRID_STEP = 5, GRID_SIZE = 13
    GRID_POINTS = 169
    MIN_PRESSURE_DELTA = 1e-4
    EPSILON = 1e-3

// ==================== 主入口 ====================

FUNCTION 插值(P1, P2, P3, P4, P5, Patm, Tatm, PRB文件路径):
    // 加载PRB
    rows = 加载PRB文件(PRB文件路径)
    indexed = 构建索引表(rows)
    
    // 计算系数
    point = 计算压力系数(P1, P2, P3, P4, P5)
    
    // 区域分类
    point = 分类区域1_8(point, indexed)
    IF point.α == null:
        point = 分类区域9(point, indexed)
    断言 point.α != null AND point.β != null
    
    // 输出值插值
    CPT = 插值输出值(point, indexed, selector=取CPT)
    CPS = 插值输出值(point, indexed, selector=取CPS)
    
    // 气动参数
    (P0, Ps, dp, ρ, V, Ma) = 计算气动参数(P1,P2,P3,P4,P5, CPT,CPS, Tatm, Patm)
    
    // 校验
    (isValid, warning) = 校验结果(point.α, point.β, P0, Ps, dp, ρ, V, Ma, ...)
    
    RETURN { α: point.α, β: point.β, Ma, V, dp, ρ, P0, Ps, isValid, warning }
```

---

## 8. 关键常数汇总

| 常数 | 值 | 含义 |
|------|-----|------|
| `GRID_SIZE` | 13 | 网格维度（α × β） |
| `GRID_STEP` | 5° | 网格步长 |
| `MIN_PRESSURE_DELTA` | 1e-4 | 最小压差阈值（Pa），防止零除 |
| `EPSILON` | 1e-3 | 浮点比较容差 |
| `GAS_CONSTANT_AIR` | 287.06 | 空气气体常数 J/(kg·K) |
| `GRID_POINTS` | 169 | 13×13 校准点数 |

---

## 9. 新旧算法对照（参考）

| 对比维度 | 本算法（PRB插值） | 新算法（公式+网格） |
|----------|-------------------|---------------------|
| 系数公式 | 固定 Ka=(P4−P5)/δ, Kb=(P3−P1)/δ | AA1/AA2/AA3 三套公式 |
| 区域数量 | 9 个（1-8边界 + 9网格内） | 3 个（corner/edge/center） |
| 网格数量 | 1 个（Ka/Kb + α/β 叠加） | 3 个（AA1/AA2/AA3 独立网格） |
| 校准文件 | .prb（预计算 ka/kb/cpt/cps） | .csv（原始压力 P1~P5） |
| 扩展外推 | 无（超网格标记无效） | 有（扩展网格线性外推） |
| CPT/CPS 来源 | PRB 表双线性插值 | 硬编码固定系数（或未实现） |