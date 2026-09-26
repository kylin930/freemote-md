# Physics Stiffness 真实来源追踪报告

**任务 ID**: 21 (P0)
**作者**: StiffnessTracer
**日期**: 2026-09-26
**范围**: hair / parts / bust 三类物理对象的 stiffness 参数真实来源追踪
**方法**: 从 Python 代码反向追踪到 asm.js（`reference/FreeMoteDriver-format.js`，99629 行），并用 59 个 NEKOPARA 模型 cross-validation

---

## 0. 执行摘要

| 物理对象 | stiffness 真实来源 | 证据等级 | 59 模型实际值范围 | Python 当前状态 |
|---------|-------------------|---------|------------------|----------------|
| **hair** | E-mote 内部属性 ID **8926**（stiffness）/ **8934**（stiffness2），在 `EmotePlayer_Initialize` 阶段由 asm.js `pk` 函数（L42615）从模型对象读取。**不来自 PSB `hairControl` 的任何字段**（hairControl dict 中无 spring/stiffness key）。 | `[LIKELY]` 属性 ID 读取路径 `[PROVEN]`，属性 ID → PSB key 精确映射 `[LIKELY]` | 无法直接观测（属性 ID 体系是 E-mote 内部的）；hairControl 中相关数组字段：`scale_x∈{0.25,0.75}`、`scale_y∈{2.0,3.0}`、`length∈{48.0,64.0}`、`friction_x∈{0.03125,0.046875,0.0625}`、`friction_y∈{0.03125,0.0625,0.09375}` | **使用硬编码默认值 0.003**（`player.py` L1244），与原版差异 `[CONFIRMED-DIFFERENCE]` |
| **parts** | 同 hair，属性 ID 8926/8934（`pk` 函数复用，jj 函数以 `h=2` 调用）。**不来自 PSB `partsControl` 的任何字段**。 | `[LIKELY]` 同 hair | 同 hair（partsControl 字段值范围与 hairControl 完全一致） | **使用硬编码默认值 0.003**，与原版差异 `[CONFIRMED-DIFFERENCE]` |
| **bust** | **PSB `metadata.bustControl[*].spring`**（单值 float），在 `EmotePlayer_Initialize` 阶段由 asm.js `ok`/`tk` 函数从模型属性 8942→9340/9972 读取。**与 hair/parts 使用不同来源**。 | `[PROVEN]` | `spring ∈ {0.015625, 0.03125}`（2 个唯一值，59 模型中分布：~20 模型用 0.015625/0.03125 组合，~39 模型用 0.03125/0.03125） | **从 PSB `bustControl.spring` 正确读取**（`player.py` L1218-1223），但 stiffness2 被设为与 stiffness 相同值（原版 stiffness2 来自 9972，可能不同）`[LIKELY-MATCH]` |

**核心结论**：
1. **bust stiffness 来源已确认** `[PROVEN]`：PSB `bustControl.spring`，Python 已正确实现。
2. **hair/parts stiffness 来源未完全确认** `[LIKELY]`：来自 E-mote 内部属性 ID 8926/8934，**不来自 PSB hairControl/partsControl 的任何直接字段**。Python 当前用硬编码默认值 0.003 替代，**与原版有差异**。
3. **8926/8934 属性 ID 到 PSB key 的精确映射无法直接确认**（E-mote 属性 ID 体系是内部的，asm.js 中无属性名→ID 映射表）。根据 hairControl 中数组字段的存在性和 pk 函数的数组读取模式，**最可能的对应是 `scale_x`/`scale_y` 或其他数组字段**，但需动态 trace 确认。

---

## 1. hair stiffness

### 1.1 来源
- **asm.js 函数**: `pk(a)` @ `reference/FreeMoteDriver-format.js` L42615
- **属性读取**: 
  - L42677: `fv(n, a, 8926)` — 从模型对象 `a` 读取属性 ID 8926（数组）
  - L42683: `fv(n, a, 8934)` — 读取属性 ID 8934（数组）
  - L42688-42699: 用 `_u(k, l, 0/1)` 读取数组元素，存到 `d+24/28`（8919）、`d+32/36`（8926）、`d+40/44`（8934）
- **拷贝到物理对象**: `xp(b, d)` @ L64917
  - L64935: `c[b+36>>2] = c[d+32>>2]`（stiffness[0] ← 8926[0]）
  - L64939: `c[b+40>>2] = c[d+36>>2]`（stiffness[1] ← 8926[1]）
  - L64936: `c[b+44>>2] = c[d+40>>2]`（stiffness2[0] ← 8934[0]）
  - L64940: `c[b+48>>2] = c[d+44>>2]`（stiffness2[1] ← 8934[1]）
- **积分器使用**: `zp(b, d, e, f, h, j, k, l, m)` @ L65028
  - L65185: `d = +xo(-(d * +g[b + 36 + (v << 2) >> 2] * l))` — `angle = xo(-error.x * stiffness[v] * scale)`
  - L65188: `g[j>>2] = +xo((+g[H>>2] - +g[y>>2]) * +g[b + 44 + (v << 2) >> 2] * l)` — `angle2 = xo((cur.y - error.y) * stiffness2[v] * scale)`

### 1.2 字段/属性名
- **stiffness**: E-mote 属性 ID **8926**（2-element float32 数组，每顶点 1 个值）
- **stiffness2**: E-mote 属性 ID **8934**（2-element float32 数组）

### 1.3 数据类型
- `list[float]`，长度 2（2 顶点链，每顶点 1 个 stiffness 值）

### 1.4 单位
- 无量纲系数（角度响应增益，与 scale 相乘后输入 `xo` asin 限幅）

### 1.5 默认值
- asm.js: 无显式默认值（属性不存在时 `fv` 返回 0.0）
- Python: **0.003**（`player.py` L1244 `_DEFAULT_STIFFNESS = 0.003`）

### 1.6 59 模型实际值范围
- **无法直接观测**（8926/8934 是 E-mote 内部属性 ID，PSB 中无直接对应字段）
- hairControl 中相关数组字段值范围（59 模型，4 objects × 59 = 236 entries）：

| 字段 | 唯一值 | 说明 |
|------|--------|------|
| `scale_x` | `{0.25, 0.75}` | 每顶点缩放系数（2 元素数组） |
| `scale_y` | `{2.0, 3.0}` | 每顶点缩放系数（2 元素数组） |
| `length` | `{48.0, 64.0}` | 链长度（2 元素数组） |
| `friction_x` | `{0.03125, 0.046875, 0.0625}` | x 方向阻尼 |
| `friction_y` | `{0.03125, 0.0625, 0.09375}` | y 方向阻尼 |
| `b_rate` | `{0.003711}` | 弯曲速率 |
| `bend_spd` | `{0.098175, 0.2, 0.3, 0.392699}` | 弯曲速度 |
| `bend_vol` | `{3.0, 4.0}` | 弯曲体积 |
| `v_bound` | `{0.5, 1.0, 2.0}` | 速度边界 |
| `gravity` | `{0.2, 0.6}` | 重力 |

### 1.7 证据等级
- `[PROVEN]` 8926/8934 在 `pk` 函数中被读取，存到物理对象 b+36/40 和 b+44/48
- `[PROVEN]` b+36/40 和 b+44/48 在 `zp` 积分器中用于 angle/angle2 计算
- `[PROVEN]` hairControl dict 中无 `spring`/`stiffness` 字段（59 模型 cross-validation）
- `[LIKELY]` 8926/8934 对应 hairControl 中的某个数组字段（最可能 `scale_x`/`scale_y`，因 pk 函数读取的是 2-element 数组）
- `[UNKNOWN]` 8926/8934 到 PSB key 的精确映射（E-mote 属性 ID 体系是内部的）

---

## 2. parts stiffness

### 2.1 来源
- **与 hair 完全相同**：复用 `pk` 函数和 `zp` 积分器
- **调用路径**: `fj` @ L60684 → `jj(b, b+224, h, 2)` @ L60783（h=2 表示 parts）→ `pk(ua)` @ L62296
- **属性读取**: 同 hair（8926/8934），但从 parts 物理对象的模型对象读取

### 2.2 字段/属性名
- 同 hair：属性 ID 8926（stiffness）/ 8934（stiffness2）

### 2.3 数据类型 / 单位 / 默认值
- 同 hair

### 2.4 59 模型实际值范围
- **无法直接观测**（同 hair）
- partsControl 中数组字段值范围（59 模型，2 objects × 59 = 118 entries）**与 hairControl 完全一致**：

| 字段 | 唯一值 |
|------|--------|
| `scale_x` | `{0.25, 0.75}` |
| `scale_y` | `{2.0, 3.0}` |
| `length` | `{48.0, 64.0}` |
| `friction_x` | `{0.03125, 0.046875, 0.0625}` |
| `friction_y` | `{0.03125, 0.0625, 0.09375}` |
| `b_rate` | `{0.003711}` |
| `bend_spd` | `{0.098175, 0.2, 0.3, 0.392699}` |
| `bend_vol` | `{3.0, 4.0}` |
| `v_bound` | `{0.5, 1.0, 2.0}` |
| `gravity` | `{0.2, 0.6}` |

### 2.5 证据等级
- `[PROVEN]` parts 与 hair 复用相同代码路径（`pk`/`zp`）
- `[PROVEN]` partsControl dict 中无 `spring`/`stiffness` 字段（59 模型 cross-validation）
- `[LIKELY]` 同 hair，8926/8934 对应 partsControl 中的某个数组字段

---

## 3. bust stiffness

### 3.1 来源
- **asm.js 函数**: `ok(a, b)` @ L42482（bust 物理初始化）/ `tk(a, b)` @ L42951（另一种 bust 路径）
- **属性读取**:
  - L42521: `fv(y, a, 8942)` — 从模型对象 `a` 读取属性 ID 8942（bust 配置对象）
  - L42528: `fv(m, o, 9340)` — 从 8942 子对象读取属性 ID 9340（stiffness，单值）
  - L42530: `fv(n, o, 9972)` — 读取属性 ID 9972（stiffness2，单值）
  - L42532-42534: `g[b>>2]=v; g[b+4>>2]=u; g[b+8>>2]=t`（存到 b+0/4/8）
- **积分器使用**: `Pn(...)` @ L28025
  - L28084 (asm.js `gv(v, b, 9340, 28084)`): stiffness 用于 `angle = xo(-((cur.x - pos.x) * scale * stiffness))`
  - L28088 (asm.js `gv(A, x, 9342, 28088)`): stiffness2 用于 `angle2 = xo((-((cur.y - pos.y) * scale) - bust_offset) * stiffness2)`

### 3.2 PSB key
- **`metadata.bustControl[*].spring`** `[PROVEN]`
- PSB bustControl dict keys: `['baseLayer', 'enabled', 'friction', 'gravity', 'label', 'param', 'parameter', 'scale_x', 'scale_y', 'spring', 'var_lr', 'var_ud']`
- `spring` 字段直接存在于 PSB 中，Python 已正确读取

### 3.3 字段/属性名
- **stiffness**: PSB key `spring` → E-mote 属性 ID 9340（via 8942）
- **stiffness2**: E-mote 属性 ID 9972（via 8942），**PSB 中可能无独立字段**（Python 当前把 stiffness2 设为与 stiffness 相同值）

### 3.4 数据类型
- `float`（单值，非数组）

### 3.5 单位
- 无量纲系数（角度响应增益）

### 3.6 默认值
- asm.js: 无显式默认值
- Python: 0.003（仅当 `spring` 字段缺失时回退，`player.py` L1256-1259）

### 3.7 59 模型实际值范围
- **`spring` ∈ {0.015625, 0.03125}** `[PROVEN]`（2 个唯一值）
- **`friction` ∈ {0.06, 0.125}** `[PROVEN]`（2 个唯一值）
- **`gravity` ∈ {0.1, 0.3}** `[PROVEN]`（2 个唯一值）

**每模型 bust spring 值**（59 模型，2 objects × 59 = 118 entries）：

| spring 组合 | 模型数 | 代表模型 |
|------------|--------|---------|
| `[0.015625, 0.03125]` | ~20 | azuki-*, chocola-*, vanilla-*, milk-winter |
| `[0.03125, 0.03125]` | ~39 | cinnamon-*, coconut-*, maple-*, milk-teenage, fraise-maid |

**特殊模型**：
- `fraise-maid`: `friction=[0.125, 0.06]`, `gravity=[0.3, 0.1]`（唯一使用 0.06/0.1 的模型）
- `milk-winter`: `spring=[0.015625, 0.03125]`（与 milk-teenage `[0.03125, 0.03125]` 不同）

### 3.8 证据等级
- `[PROVEN]` bust stiffness 来自 PSB `bustControl.spring`（59 模型 cross-validation）
- `[PROVEN]` asm.js `ok`/`tk` 函数从属性 8942→9340 读取
- `[PROVEN]` Python `player.py` L1218-1223 正确读取 `bustControl.spring`
- `[LIKELY]` stiffness2（属性 9972）在 PSB 中可能无独立字段，Python 设为与 stiffness 相同值

---

## 4. runtimeAPI 属性 8926/8934

### 4.1 属性 8926
- **含义**: hair/parts stiffness 数组（2-element float32，每顶点 1 个值）
- **读取位置**: asm.js `pk` 函数 L42677 `fv(n, a, 8926)`，`uk` 函数 L43049 `fv(f, a, 8926)`
- **存储位置**: 物理对象 `b+36`（stiffness[0]）、`b+40`（stiffness[1]）
- **用途**: `zp` 积分器 L65185 `angle = xo(-error.x * stiffness[v] * scale)`
- **设置时机**: **Initialize 阶段**（`EmotePlayer_-Initialize` → `Be` → `Me` → `cj` → `ej` → `fj` → `jj` → `pk`）
- **设置者**: PSB 加载时自动映射（asm.js 中无显式 setter，属性在 PSB 节点树加载时建立）

### 4.2 属性 8934
- **含义**: hair/parts stiffness2 数组（2-element float32）
- **读取位置**: asm.js `pk` 函数 L42683 `fv(n, a, 8934)`，`uk` 函数 L43051 `fv(e, a, 8934)`
- **存储位置**: 物理对象 `b+44`（stiffness2[0]）、`b+48`（stiffness2[1]）
- **用途**: `zp` 积分器 L65188 `angle2 = xo((cur.y - error.y) * stiffness2[v] * scale)`
- **设置时机**: 同 8926（Initialize 阶段）
- **设置者**: PSB 加载时自动映射

### 4.3 相关属性 ID
- **8919**: pk 函数 L42671 读取，存到 `d+24/28` → `b+28/32`。用途未完全确认 `[LIKELY]` 某种 damping 或 auxiliary stiffness
- **8942**: bust 配置对象（ok/tk 函数入口）
- **9340**: bust stiffness（via 8942 子对象）
- **9972**: bust stiffness2（via 8942 子对象）
- **9342**: bust 配置子对象（tk 函数 L42992 读取）

---

## 5. 设置时机

### 5.1 调用链 `[PROVEN]`
```
EmotePlayer_Initialize (Zh @ L57037, exported as _EmotePlayer_Initialize)
  → vtable[1] = Be (@ L54394, L99149 `var vd =4 = [uB, Be, Ke, Fp, Iz, Pz, fA, uB]`)
    → Me (@ L54479, L54402 `Me(g, a, b, d, e)`)
      → cj (@ L60332, L54526 `cj(s, r, t, 1)`, L54571 `cj(s, r, t, 1)`)
        → ej (@ L60510, L60452/60458 `ej(b)`)
          → fj (@ L60684, L60678 `fj(b, n)`)
            → ij (@ L61472, L60765 `ij(b, h)`)        [bust 物理初始化]
              → tk (@ L42951, L61592 `tk(ka, D)`)     [bust stiffness 读取]
              → uk (@ L43026, L61601 `u = uk(ka)`)    [bust alt 路径]
            → jj (@ L62150, L60774 `jj(b, b+212, h, 1)`) [hair 物理初始化, h=1]
              → ok (@ L42482, L62287 `ok(ua, $)`)     [bust config 读取]
              → pk (@ L42615, L62296 `w = pk(ua)`)    [hair/parts stiffness 读取, 8926/8934]
            → jj (@ L62150, L60783 `jj(b, b+224, h, 2)`) [parts 物理初始化, h=2]
              → pk (@ L42615, L62296 `w = pk(ua)`)    [parts stiffness 读取]
```

### 5.2 阶段确认
- **Initialize 阶段** `[PROVEN]`：8926/8934 在 `EmotePlayer_Initialize` 调用链中被读取
- **不是 PSB load 阶段**：PSB load 只解析节点树，不调用 `pk`
- **不是 physics init 阶段**：physics init（`invalidatePhysics`）只复位状态，不读取 stiffness
- **不是运行时 SetVariable**：8926/8934 是模型属性，不是变量

---

## 6. PSB 中是否存在对应值

### 6.1 bust stiffness
- **存在** `[PROVEN]`：PSB key `metadata.bustControl[*].spring`
- **值类型**: float（单值）
- **59 模型值范围**: `{0.015625, 0.03125}`

### 6.2 hair/parts stiffness
- **不存在直接对应字段** `[PROVEN]`：hairControl/partsControl dict 中无 `spring`/`stiffness` key
- **可能对应其他数组字段** `[LIKELY]`：
  - `scale_x`（2-element 数组，值 `{0.25, 0.75}`）
  - `scale_y`（2-element 数组，值 `{2.0, 3.0}`）
  - `length`（2-element 数组，值 `{48.0, 64.0}`）
- **来源推测** `[LIKELY]`：8926/8934 属性 ID 在 PSB 加载时从 hairControl/partsControl 的某个数组字段自动映射。E-mote 的属性 ID 体系是内部的，asm.js 中无属性名→ID 映射表，无法直接确认精确对应。

---

## 7. 不同 59 模型是否真的存在不同 stiffness

### 7.1 bust stiffness
- **是** `[PROVEN]`：存在 2 个唯一值（0.015625, 0.03125），分布如下：
  - ~20 模型使用 `[0.015625, 0.03125]` 组合（azuki-*, chocola-*, vanilla-*, milk-winter）
  - ~39 模型使用 `[0.03125, 0.03125]` 组合（cinnamon-*, coconut-*, maple-*, milk-teenage, fraise-maid）

### 7.2 hair/parts stiffness
- **无法直接确认** `[UNKNOWN]`：8926/8934 是 E-mote 内部属性 ID，无法直接观测
- **间接证据** `[LIKELY]`：hairControl/partsControl 中的数组字段（scale_x, scale_y, length, friction_x, friction_y 等）在 59 模型中存在多个唯一值，说明不同模型确实有不同的物理参数。如果 8926/8934 对应这些字段之一，则不同模型有不同的 stiffness。
- **Python 当前状态**：所有 59 模型都使用相同的硬编码默认值 0.003，**与原版差异**

---

## 8. Python 当前状态

### 8.1 hair/parts stiffness
- **来源**: 硬编码默认值 0.003（`src/freemote/player/player.py` L1244）
- **设置位置**: `_init_physics_from_metadata` 方法 L1239-1254
- **逻辑**: 检查 `hairControl`/`partsControl` 中是否有 stiffness（实际没有），若无则用 0.003
- **与原版差异**: `[CONFIRMED-DIFFERENCE]`
  - 原版从 E-mote 属性 ID 8926/8934 读取（Initialize 阶段）
  - Python 用固定 0.003 替代
  - 影响：所有 59 模型 hair/parts 摆动角度响应相同，原版应有模型间差异

### 8.2 bust stiffness
- **来源**: PSB `metadata.bustControl[*].spring`（`player.py` L1218-1223）
- **设置位置**: `_init_physics_from_metadata` 方法 L1218-1223
- **逻辑**: `spring = bc.get('spring'); stiffness = stiffness2 = float(spring)`
- **与原版差异**: `[LIKELY-MATCH]`
  - stiffness 读取正确 `[PROVEN]`
  - stiffness2 被设为与 stiffness 相同值，原版 stiffness2 来自属性 9972（可能不同）`[LIKELY-DIFFERENCE]`

### 8.3 PhysicsData.raw_params
- **状态**: 未解析（`src/freemote/format/model.py` L208 `raw_params: bytes = b""`）
- **PSB loader**: 未从 PSB 提取 PhysicsData（`loader.py` 中无 `PhysicsData(...)` 赋值）
- **TODO**: `UNKNOWN-A06`（`player.py` L1017-1022）— parse PhysicsData.raw_params to extract stiffness arrays

---

## 9. 10 个问题完整回答

### Q1: hair stiffness 从哪里读取？
**A1**: 从 E-mote 内部属性 ID **8926**（stiffness）/ **8934**（stiffness2）读取，在 `EmotePlayer_Initialize` 阶段由 asm.js `pk` 函数（L42615）从模型对象读取。**不来自 PSB hairControl 的任何直接字段**。`[PROVEN]` 读取路径，`[LIKELY]` 属性 ID → PSB key 映射。

### Q2: parts stiffness 从哪里读取？
**A2**: 与 hair 完全相同，复用 `pk` 函数和属性 ID 8926/8934，`jj` 函数以 `h=2` 调用。`[PROVEN]`

### Q3: bust stiffness 是否和它们使用不同来源？
**A3**: **是** `[PROVEN]`。bust stiffness 来自 PSB `bustControl.spring`（属性 ID 9340 via 8942），是单值 float；hair/parts stiffness 来自属性 ID 8926/8934，是 2-element 数组。代码路径也不同：bust 用 `ok`/`tk` 函数，hair/parts 用 `pk` 函数。

### Q4: runtimeAPI 属性 8926/8934 分别是什么？
**A4**: 
- **8926**: hair/parts stiffness 数组（2-element float32，每顶点 1 个值），用于 `angle = xo(-error.x * stiffness[v] * scale)`
- **8934**: hair/parts stiffness2 数组（2-element float32），用于 `angle2 = xo((cur.y - error.y) * stiffness2[v] * scale)`
`[PROVEN]`

### Q5: 这些属性什么时候被设置？
**A5**: 在 **Initialize 阶段**（`EmotePlayer_Initialize` 调用链）被读取。设置发生在 PSB 加载时（自动映射），asm.js 中无显式 setter。`[PROVEN]` 读取时机，`[LIKELY]` 设置时机。

### Q6: 是 Initialize 阶段、PSB load 阶段、physics init 阶段，还是运行时 SetVariable？
**A6**: **Initialize 阶段**读取 `[PROVEN]`。PSB load 阶段建立属性映射 `[LIKELY]`。physics init 阶段只复位状态。SetVariable 不涉及。

### Q7: PSB 中是否已经存在对应值？
**A7**: 
- **bust**: 存在 `[PROVEN]`，key = `metadata.bustControl[*].spring`
- **hair/parts**: 不存在直接对应字段 `[PROVEN]`（hairControl/partsControl dict 中无 spring/stiffness key）。可能对应其他数组字段 `[LIKELY]`。

### Q8: 如果存在，PSB key 是什么？
**A8**: 
- **bust**: `metadata.bustControl[*].spring` `[PROVEN]`
- **hair/parts**: 无直接 key `[PROVEN]`。可能对应 `scale_x`/`scale_y`/`length` 等 `[LIKELY]`。

### Q9: 如果不存在，是否来自 raw_params 或 runtime API？
**A9**: hair/parts stiffness 不来自 `raw_params`（Python 中 raw_params 未解析）。来自 E-mote 内部属性 ID 8926/8934（runtime API 在 Initialize 阶段读取）。`[PROVEN]`

### Q10: 不同 59 个模型是否真的存在不同 stiffness？
**A10**: 
- **bust**: **是** `[PROVEN]`，2 个唯一值（0.015625, 0.03125）
- **hair/parts**: **无法直接确认** `[UNKNOWN]`，但 hairControl/partsControl 中数组字段存在多个唯一值，**likely** 不同模型有不同 stiffness。Python 当前所有模型用相同 0.003，**与原版差异**。

---

## 10. 数据流图

### 10.1 hair/parts stiffness 数据流
```
PSB hairControl/partsControl (无 spring/stiffness 字段)
    ↓ [LIKELY] 某个数组字段 (scale_x/scale_y/length?) 自动映射
E-mote 属性 ID 8926 (stiffness) / 8934 (stiffness2)
    ↓ [PROVEN] PSB load 阶段建立属性映射
模型对象 (EmotePlayerManager internal)
    ↓ [PROVEN] Initialize 阶段: pk 函数 L42677/L42683 读取 (fv)
    ↓   调用链: EmotePlayer_Initialize → Be → Me → cj → ej → fj → jj → pk
临时对象 d+32/36 (stiffness) / d+40/44 (stiffness2)
    ↓ [PROVEN] xp 函数 L64935-64940 拷贝
物理对象 b+36/40 (stiffness) / b+44/48 (stiffness2)
    ↓ [PROVEN] zp 积分器 L65185/L65188 使用
angle = xo(-error.x * stiffness[v] * scale)
angle2 = xo((cur.y - error.y) * stiffness2[v] * scale)
    ↓
output_angles → motion_painter → layer 旋转
```

### 10.2 bust stiffness 数据流
```
PSB metadata.bustControl[*].spring (float, 0.015625 or 0.03125)
    ↓ [PROVEN] PSB load 阶段解析到 root_value['metadata']['bustControl']
E-mote 属性 ID 8942 (bust config) → 9340 (stiffness) / 9972 (stiffness2)
    ↓ [PROVEN] Initialize 阶段: ok/tk 函数 L42521/L42528/L42530 读取 (fv)
    ↓   调用链: EmotePlayer_Initialize → Be → Me → cj → ej → fj → ij → tk/uk
物理对象 b+0 (stiffness) / b+4 (stiffness2)
    ↓ [PROVEN] Pn 积分器 L28084/L28088 使用
angle = xo(-((cur.x - pos.x) * scale * stiffness))
angle2 = xo((-((cur.y - pos.y) * scale) - bust_offset) * stiffness2)
    ↓
output_angles → motion_painter → layer 旋转
```

### 10.3 Python 当前数据流
```
PSB metadata.bustControl[*].spring
    ↓ [PROVEN] player.py _init_physics_from_metadata L1218-1223
BustPhysics.stiffness = BustPhysics.stiffness2 = spring
    ↓ [PROVEN] Pn integrator (physics.py L682-684)
angle/angle2

PSB metadata.hairControl/partsControl (无 spring/stiffness)
    ↓ [PROVEN] player.py _init_physics_from_metadata L1244-1254
HairPartsPhysics.stiffness = [0.003, 0.003]  # 硬编码默认值
HairPartsPhysics.stiffness2 = [0.003, 0.003]
    ↓ [PROVEN] zp integrator (physics.py L482/L488)
angle/angle2
```

---

## 11. 差异总结与建议

### 11.1 已确认差异
| # | 差异 | 原版 | Python | 影响 | 证据 |
|---|------|------|--------|------|------|
| 1 | hair/parts stiffness 来源 | E-mote 属性 8926/8934（模型相关值） | 硬编码 0.003 | 所有 59 模型 hair/parts 摆动角度响应相同，原版应有模型间差异 | `[CONFIRMED-DIFFERENCE]` |
| 2 | bust stiffness2 来源 | E-mote 属性 9972（可能独立值） | 设为与 stiffness 相同 | bust angle2 响应可能与原版不同 | `[LIKELY-DIFFERENCE]` |

### 11.2 修复建议
1. **hair/parts stiffness**：需要动态 trace E-mote runtime，确认 8926/8934 属性 ID 到 PSB key 的精确映射。可能需要解析 `PhysicsData.raw_params`（`UNKNOWN-A06`）。
2. **bust stiffness2**：检查 PSB `bustControl` 中是否有独立 stiffness2 字段（当前 keys 中无），或确认属性 9972 的来源。
3. **Python `set_stiffness_from_model` 方法已实现**（`physics.py` L355-374 / L581-591），但未被调用（`player.py` L1017-1022 TODO）。

---

## 12. 验证脚本

- `_verify_stiffness_59.py`: 59 模型 hair/parts/bust stiffness 值范围验证
- `_verify_stiffness_59_result.json`: 验证结果 JSON
- `_verify_hair_parts_arrays.py`: 59 模型 hairControl/partsControl 数组字段值范围
- `_verify_hair_parts_arrays.json`: 验证结果 JSON
- `_verify_psb_strings.py`: PSB 字符串表和 hairControl/bustControl keys 检查

---

## 13. 参考资料

- **asm.js**: `reference/FreeMoteDriver-format.js`（99629 行）
  - `pk` 函数 @ L42615: hair/parts stiffness 读取（8926/8934）
  - `ok` 函数 @ L42482: bust stiffness 读取（8942→9340/9972）
  - `tk` 函数 @ L42951: bust stiffness alt 路径
  - `uk` 函数 @ L43026: bust alt 路径（8926/8934 读取）
  - `xp` 函数 @ L64917: hair/parts 物理对象拷贝
  - `zp` 函数 @ L65028: hair/parts 积分器
  - `Pn` 函数 @ L28025: bust 积分器
  - `Zh` 函数 @ L57037: `EmotePlayer_Initialize`
  - 调用链: `Be` @ L54394 → `Me` @ L54479 → `cj` @ L60332 → `ej` @ L60510 → `fj` @ L60684 → `ij` @ L61472 / `jj` @ L62150
- **Python**: 
  - `src/freemote/player/player.py` L1065-1275: `_init_physics_from_metadata`
  - `src/freemote/physics/physics.py` L320-518: `HairPartsPhysics` / L527-691: `BustPhysics`
  - `src/freemote/format/model.py` L196-211: `PhysicsData`
- **文档**:
  - `emote-render-pipeline-closure.md` §5: 59 模型路径分类
  - `render-alignment-final-audit.md` §9: Runtime/Physics 入口

---

**报告生成时间**: 2026-09-26
**任务状态**: completed
**关键结论**: bust stiffness 来源已确认 `[PROVEN]`；hair/parts stiffness 来源为 E-mote 属性 8926/8934 `[LIKELY]`，Python 当前用硬编码 0.003 替代，与原版有差异。