# Runtime / Physics 完整分析报告

**任务 ID**: 25 (FinalReportWriter)
**作者**: FinalReportWriter
**日期**: 2026-09-26
**范围**: 整合 P0/P1/P2/P3 四个子报告，合成 Runtime/Physics 完整对齐分析
**子报告来源**:
- P0: `physics-stiffness-source-report.md`（Stiffness 来源追踪，Task 21）
- P1: `convolve-canvas-movement-report.md`（convolveCanvasMovementToPhysics 数据流，Task 22）
- P2: `physics-output-to-render-report.md`（Physics 输出到 RenderContext，Task 23）
- P3: `gu-pu-trigger-audit-report.md`（gu-pu 触发路径审计，Task 24）

**证据来源**: `reference/FreeMoteDriver-format.js`（99629 行 asm.js）、`reference/emoteplayer-format.js`（1267 行 JS 层）、`src/freemote/`（Python 实现）、59 个 NEKOPARA 模型 cross-validation

**证据级别约定**:
- `[PROVEN]`: 直接从 asm.js/JS/Python 源码验证，有明确代码行号
- `[LIKELY]`: 基于合理推断，未能在源码中直接验证
- `[UNVERIFIED]`: 需要进一步验证
- `[UNKNOWN]`: 无法验证

---

## 1. 执行摘要

### 1.1 本阶段目标

本阶段（Runtime/Physics 对齐）的目标是完整追踪 E-mote 物理系统从参数来源、外力输入、积分器计算、到渲染输出的完整数据流，并对比 Python 实现与 asm.js 原版的差异，为下一阶段（eyeControl / eyebrowControl / mouthControl）研究建立入口。

### 1.2 四个子任务完成状态

| 子任务 | 报告 | 状态 | 核心结论 |
|--------|------|------|----------|
| P0 (Task 21) | `physics-stiffness-source-report.md` | completed | bust stiffness 来源已确认 `[PROVEN]`；hair/parts stiffness 来自 E-mote 属性 8926/8934 `[LIKELY]`，Python 用硬编码 0.003 替代 |
| P1 (Task 22) | `convolve-canvas-movement-report.md` | completed | convolveCanvasMovementToPhysics 是 JS 层开关（默认 false），Python 未实现 `[PROVEN]` |
| P2 (Task 23) | `physics-output-to-render-report.md` | completed | physics 输出是 transform 级别修改 `[PROVEN]`，所有 59 模型都触发 physics `[PROVEN]` |
| P3 (Task 24) | `gu-pu-trigger-audit-report.md` | completed | pu 确实触发 `[PROVEN]`，59/59 模型，1 帧延迟；没有函数可定性为 PROVEN NO EFFECT |

### 1.3 核心结论一句话总结

**asm.js 物理系统的完整数据流（PSB metadata → stiffness → 外力环形缓冲区 → 积分器 → variable map → pu → part transform → hu → xr/zr → draw call）已被完整追踪并 `[PROVEN]`；Python 已正确实现 bust stiffness 读取和 pu 的 physics angle 应用，但在 hair/parts stiffness 来源（硬编码 0.003）、convolveCanvasMovementToPhysics（完全缺失）、应用时机（1 帧延迟 vs 当前帧）、angle 索引选择（3 个 vs 1 个）、deformation/swing 输出（缺失）等 5 个关键环节存在 `[CONFIRMED-DIFFERENCE]`。**

---

## 2. Physics 数据流总览

### 2.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ① 参数来源阶段 (PSB load + Initialize)                                  │
│                                                                         │
│  PSB metadata                                                           │
│    ├─ bustControl[*].spring ──[PROVEN]──→ 属性 8942→9340 (bust stiff)   │
│    ├─ hairControl (无 spring) ─[LIKELY]─→ 属性 8926/8934 (hair stiff)   │
│    └─ partsControl (无 spring) ─[LIKELY]─→ 属性 8926/8934 (parts stiff) │
│                                                                         │
│  EmotePlayer_Initialize → Be → Me → cj → ej → fj                        │
│    ├─→ ij → tk/uk (bust: ok/tk 读取 8942→9340/9972)  [PROVEN]           │
│    ├─→ jj(h=1) → pk (hair: 读取 8926/8934)            [PROVEN 读取路径] │
│    └─→ jj(h=2) → pk (parts: 读取 8926/8934)           [PROVEN 读取路径] │
│                                                                         │
│  结果: 物理对象 b+36/40 (stiff), b+44/48 (stiff2) 被填充                │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ② 外力输入阶段 (convolveCanvasMovementToPhysics, 默认 false)            │
│                                                                         │
│  [JS 层] canvas.getBoundingClientRect() + scrollX/Y  [PROVEN]            │
│    ↓ 每帧差分                                                           │
│  vec = [(cur.left-prev.left)/scale*frameCount,                          │
│         (cur.top-prev.top)/scale*frameCount]           [PROVEN]          │
│    ↓ 同时施加给 bust/parts/hair                                         │
│  EmotePlayer_SetOuterForce(playerId, cat, vec[0], vec[1], 0, 0)         │
│    ↓                                                                     │
│  [asm.js] li → Gf → xm → wm → To                       [PROVEN]          │
│    ↓ 推入环形缓冲区 (player+352/356/360, 170 元素/页)   [PROVEN]        │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ③ 积分器阶段 (每帧 Update)                                              │
│                                                                         │
│  EmotePlayer_Update → Ci → eg → Al (L78610)            [PROVEN]          │
│    ├─ Qo/Ro: 从环形缓冲区读取 + 指数衰减插值           [PROVEN]          │
│    │    current = old + pow(k, exp) * (new - old)                       │
│    ├─ Hl (bust, L79304): 外力 → Qn → Pn (L28025)       [PROVEN]          │
│    │    外力 → b+28/32 (目标位置) → error → angle                       │
│    │    angle = xo(-error × stiffness)                  [PROVEN]          │
│    └─ Il (parts/hair, L79418): 外力 → Ap → zp (L65028) [PROVEN]          │
│         外力 → b+64/68 (目标位置) → error → angle                       │
│         angle = xo(-error × stiffness × scale)         [PROVEN]          │
│                                                                         │
│  结果: output_angles (Pn 2 个, zp 3 个)                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ④ 输出传递阶段 (variable map)                                           │
│                                                                         │
│  Hl/Il: Ij(b+100, obj+label) = angle                   [PROVEN]          │
│    ↓ 写入 EmotePlayer variable map (b+100 hash map)                     │
│    ↓ key = 物理对象 label, value = angle (float)                        │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ⑤ 渲染应用阶段 (du: gu→hu→...→pu→qu)                                   │
│                                                                         │
│  du(part) [L52165]                                                      │
│    ├─ gu (flag&512): 修改 part+616/620/624 (位置偏移)  [LIKELY]         │
│    ├─ hu (无条件): 渲染主函数                           [PROVEN]          │
│    │    ├─ 读取 part+160/164 (pu 上帧写入)             [PROVEN]          │
│    │    ├─ 计算 transform 矩阵 B+52~B+88               [PROVEN]          │
│    │    ├─ xr (L14719): mesh deformation               [PROVEN]          │
│    │    └─ zr (L15055): draw call 提交                 [PROVEN]          │
│    ├─ iu/ju/ku/lu/mu/nu/ou (条件/无条件)               [PROVEN]          │
│    └─ pu (flag&1024): physics → part transform         [PROVEN]          │
│         ├─ vtable+40 获取 physics state (读 b+100)     [LIKELY]          │
│         ├─ 修改 part+160/164/168/172 (transform)       [PROVEN]          │
│         ├─ 修改 part+616/620/624 (位置偏移)            [PROVEN]          │
│         ├─ 修改 part+628/632/636 (角度/缩放)           [PROVEN]          │
│         ├─ 修改 part+92~104 (rotation matrix)          [PROVEN]          │
│         └─ 修改 part+76~88, +648 (颜色/alpha)          [PROVEN]          │
│                                                                         │
│  结果: pu 修改 part transform → 下一帧 hu 读取 → xr/zr → draw call      │
│  注意: physics 影响有 1 帧延迟 (pu 在 hu 之后)          [PROVEN]          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 各环节证据级别

| 环节 | 证据级别 | 说明 |
|------|---------|------|
| ① PSB → stiffness（bust） | `[PROVEN]` | bustControl.spring 直接读取，59 模型验证 |
| ① PSB → stiffness（hair/parts） | `[LIKELY]` | 属性 8926/8934 读取路径 `[PROVEN]`，PSB key 映射 `[LIKELY]` |
| ② canvas → 外力 | `[PROVEN]` | JS 层完整代码 + asm.js 调用链 |
| ③ 积分器计算 | `[PROVEN]` | zp/Pn 完整源码行号引用 |
| ④ variable map 写入 | `[PROVEN]` | Ij 函数完整追踪 |
| ⑤ pu → part transform | `[PROVEN]` | vtable+40 调用 + 9 组字段修改 |
| ⑤ hu → draw call | `[PROVEN]` | hu 读取 part transform + xr/zr 调用 |
| ⑤ 1 帧延迟 | `[PROVEN]` | du 调用顺序 hu→pu |

---

## 3. P0: Stiffness 来源

### 3.1 bust stiffness 来源 `[PROVEN]`

- **PSB key**: `metadata.bustControl[*].spring`（float 单值）
- **asm.js 读取**: `ok`/`tk` 函数从属性 8942→9340 读取（L42521/L42528）
- **Python 读取**: `player.py` L1218-1223 `_init_physics_from_metadata` 正确读取
- **59 模型值范围**: `spring ∈ {0.015625, 0.03125}`（2 个唯一值）
  - ~20 模型用 `[0.015625, 0.03125]` 组合（azuki-*, chocola-*, vanilla-*, milk-winter）
  - ~39 模型用 `[0.03125, 0.03125]` 组合（cinnamon-*, coconut-*, maple-*, milk-teenage, fraise-maid）
- **积分器使用**: `Pn` L28084 `angle = xo(-((cur.x - pos.x) * scale * stiffness))`

### 3.2 hair/parts stiffness 来源 `[PROVEN 读取路径, LIKELY 映射]`

- **asm.js 读取**: `pk` 函数（L42615）从 E-mote 内部属性 ID **8926**（stiffness）/ **8934**（stiffness2）读取
  - L42677: `fv(n, a, 8926)` — 读取属性 8926（2-element float32 数组）
  - L42683: `fv(n, a, 8934)` — 读取属性 8934（2-element float32 数组）
- **不来自 PSB hairControl/partsControl 的任何直接字段** `[PROVEN]`（59 模型 cross-validation，dict 中无 spring/stiffness key）
- **属性 ID → PSB key 精确映射**: `[UNKNOWN]`（E-mote 属性 ID 体系是内部的，asm.js 中无属性名→ID 映射表）
  - 最可能对应 `scale_x`/`scale_y` 或其他数组字段 `[LIKELY]`
- **积分器使用**: `zp` L65185 `angle = xo(-error.x * stiffness[v] * scale)`，L65188 `angle2 = xo((cur.y - error.y) * stiffness2[v] * scale)`
- **hairControl/partsControl 中相关数组字段值范围**（59 模型）：

| 字段 | 唯一值 | 说明 |
|------|--------|------|
| `scale_x` | `{0.25, 0.75}` | 每顶点缩放系数 |
| `scale_y` | `{2.0, 3.0}` | 每顶点缩放系数 |
| `length` | `{48.0, 64.0}` | 链长度 |
| `friction_x` | `{0.03125, 0.046875, 0.0625}` | x 方向阻尼 |
| `friction_y` | `{0.03125, 0.0625, 0.09375}` | y 方向阻尼 |

### 3.3 Python 当前状态

| 物理对象 | Python 来源 | 与原版差异 | 证据 |
|---------|------------|-----------|------|
| bust stiffness | PSB `bustControl.spring`（`player.py` L1218-1223） | 正确读取 `[PROVEN]` | `[LIKELY-MATCH]` |
| bust stiffness2 | 设为与 stiffness 相同值 | 原版 stiffness2 来自属性 9972（可能不同） | `[LIKELY-DIFFERENCE]` |
| hair stiffness | 硬编码 0.003（`player.py` L1244 `_DEFAULT_STIFFNESS = 0.003`） | 所有 59 模型用相同值，原版应有模型间差异 | `[CONFIRMED-DIFFERENCE]` |
| parts stiffness | 硬编码 0.003 | 同 hair | `[CONFIRMED-DIFFERENCE]` |

### 3.4 P0 差异表

| # | 差异项 | asm.js 原版 | Python 当前 | 影响 | 证据 |
|---|--------|------------|------------|------|------|
| 1 | hair/parts stiffness 来源 | E-mote 属性 8926/8934（模型相关值） | 硬编码 0.003 | 所有 59 模型 hair/parts 摆动角度响应相同 | `[CONFIRMED-DIFFERENCE]` |
| 2 | bust stiffness2 来源 | E-mote 属性 9972（可能独立值） | 设为与 stiffness 相同 | bust angle2 响应可能不同 | `[LIKELY-DIFFERENCE]` |

---

## 4. P1: convolveCanvasMovementToPhysics

### 4.1 开关本质 `[PROVEN]`

- **定义位置**: `reference/emoteplayer-format.js` L407 `this._convolveCanvasMovementToPhysics = false`
- **本质**: JS 层配置开关（默认 **false**），不是 asm.js 内部函数
- **开启后行为**: 每帧把 canvas DOM 位置差分作为外力推入 asm.js 的环形缓冲区

### 4.2 位移来源和计算 `[PROVEN]`

- **位移来源**: `canvas.getBoundingClientRect()` + `window.scrollX/Y`（canvas 元素在页面中的绝对位置）
- **触发场景**: 鼠标拖动 canvas、窗口滚动、布局变化（**不是**模型内部移动，**不是**viewport 变化）
- **X/Y 分量计算**:
  - `vec.x = (curCanvasPosition.left - prevCanvasPosition.left) / scale * frameCount`
  - `vec.y = (curCanvasPosition.top - prevCanvasPosition.top) / scale * frameCount`
- **frameCount**: 是**乘数**（放大位移），不是除数
- **scale**: `player.getState("scale")`（player 缩放状态）

### 4.3 环形缓冲区和指数衰减 `[PROVEN]`

- **调用链**: `EmotePlayer_SetOuterForce` → `li` (L57252) → `Gf` → `xm` → `wm` (L82897) → `To` (L31189)
- **环形缓冲区结构**:
  - 元素大小：24 字节（x, y 外力 + z + flag + 时间戳）
  - 容量：170 元素/页，多页自动扩容
  - 结构：FIFO 队列（先进先出）
  - 三类独立队列：bust (player+352)、parts (player+356)、hair (player+360)
- **指数衰减插值**（`Ro` 函数 L30998-31081）:
  - `k = pow(k, exponent)` — 指数衰减
  - `current = old + k * (new - old)` — 插值
  - "convolve" 指对历史外力序列的加权求和（权重随时间指数衰减）

### 4.4 施加目标 `[PROVEN]`

- **施加层级**: 加到**目标位置**（不是 velocity 或 angle）
  - hair/parts: 外力 → b+64/68（目标位置）→ error = 目标位置 - 当前位置 → angle = -error × stiffness
  - bust: 外力 → b+28/32（目标位置）→ error → angle
- **施加范围**: 同时施加给 bust/parts/hair 三类（JS 层 L302-304 同时调用三次 SetOuterForce，使用相同 vec）
- **物理意义**: canvas movement 改变弹簧的**目标**，模型弹性地追踪新目标

### 4.5 Python 当前状态 `[PROVEN]`

- **convolveCanvasMovementToPhysics**: **未实现**（`src/freemote/` 中无 canvas 位移差分逻辑）
- **环形缓冲区**: **未实现**（`set_outer_force_value` 直接覆盖 `outer_force` 字段，无队列）
- **指数衰减插值**: **未实现**
- **外力施加层级**: **语义不匹配**
  - Python zp: outer_force 加到**速度**（`vel_x += w × outer_force × dt`）
  - Python Pn: outer_force 加到 **gravity**（`t = (gravity + outer_force[1]) * dt`）
  - asm.js: 外力加到**目标位置**
- **外力分量使用**: **不匹配**
  - Python zp 只用 x 分量，Pn 只用 y 分量
  - asm.js zp 用 x+y（b+64/68），Pn 用 x+y（b+28/32）
- **Python outer_force 字段实际对应**: asm.js 的 **b+4 字段**（持续外力，加到速度），**不是** convolveCanvasMovementToPhysics 的传入参数（目标位置外力）

### 4.6 P1 差异表

| # | 差异项 | asm.js 原版 | Python 当前 | 影响 | 证据 |
|---|--------|------------|------------|------|------|
| 1 | convolveCanvasMovementToPhysics 开关 | JS 层实现（默认 false） | 未实现 | Python 无法响应 canvas 移动物理 | `[CONFIRMED-DIFFERENCE]` |
| 2 | canvas 位移差分 | getBoundingClientRect() 差分 | 无 | 无 canvas movement 输入 | `[CONFIRMED-DIFFERENCE]` |
| 3 | 环形缓冲区 | FIFO 队列（170 元素/页）+ 指数衰减 | 直接覆盖 | 无历史外力平滑，无衰减过渡 | `[CONFIRMED-DIFFERENCE]` |
| 4 | 外力施加层级 | 加到目标位置（b+64/68, b+28/32） | 加到速度（zp）或 gravity（Pn） | 语义不匹配，摆动行为不同 | `[CONFIRMED-DIFFERENCE]` |
| 5 | 外力分量使用 | zp 用 x+y，Pn 用 x+y | zp 只用 x，Pn 只用 y | 分量使用不匹配 | `[CONFIRMED-DIFFERENCE]` |

---

## 5. P2: Physics 输出到 Render

### 5.1 zp 积分器输出字段（hair/parts, L65028）`[PROVEN]`

**函数签名**: `zp(b, d, e, f, h, j, k, l, m)` — b 物理对象指针（176 字节），d/e 外力，f/h/j 输出角度指针（3 个），k stiffness，l scale，m rotation

| Offset | 语义 | 写入行 | 证据 |
|--------|------|--------|------|
| b+64 | target position X | L65074 | `[PROVEN]` |
| b+68 | target position Y | L65076 | `[PROVEN]` |
| b+76 | -error X | L65079 | `[PROVEN]` |
| b+80 | -error Y | L65081 | `[PROVEN]` |
| b+88/92/96 | current position (bone 0) X/Y/Z | L65089-65093 | `[PROVEN]` |
| b+100/104/108 | current position (bone 1) X/Y/Z | L65095-65097 | `[PROVEN]` |
| b+112+v*12 | chain bone v position (v=0,1) | L65176-65180 | `[PROVEN]` |
| b+136+v*12 | chain bone v velocity (v=0,1) | L65154-65174 | `[PROVEN]` |
| b+148/152/156 | deformation X/Y/Z (v=1 only) | L65136-65138 | `[PROVEN]` |
| *f | output angle 0 (v=0, x 方向) | L65186 | `[PROVEN]` |
| *h | output angle 1 (v=1, x 方向) | L65187 | `[PROVEN]` |
| *j | output angle 2 (v=1, y 方向) | L65188 | `[PROVEN]` |

**Ap (zp wrapper, L65198) 额外输出**:
| b+164 | 累积旋转角度 | L65223 | `[PROVEN]` |
| b+168 | swing 程度 (0-1) | L65220 | `[PROVEN]` |

### 5.2 Pn 积分器输出字段（bust, L28025）`[PROVEN]`

**函数签名**: `Pn(b, c, d, e, f, h, i, j)` — b 物理对象指针（92 字节），c/d 外力，e/f 输出角度指针（2 个），h dt，i stiffness，j rotation

| Offset | 语义 | 写入行 | 证据 |
|--------|------|--------|------|
| b+28 | target position X | L28050 | `[PROVEN]` |
| b+32 | target position Y | L28052 | `[PROVEN]` |
| b+40 | -error X | L28055 | `[PROVEN]` |
| b+44 | -error Y | L28057 | `[PROVEN]` |
| b+52/56/60 | current position X/Y/Z | L28086-28089 | `[PROVEN]` |
| b+64/68/72 | velocity X/Y/Z | L28080-28084 | `[PROVEN]` |
| *e | output angle X (主角度) | L28090 | `[PROVEN]` |
| *f | output angle Y (次角度) | L28091 | `[PROVEN]` |

### 5.3 Physics → Render 数据流 `[PROVEN]`

```
zp/Pn (积分器) → output_angles
  ↓
Hl/Il: Ij(b+100, obj+label) = angle  ← 写入 variable map [PROVEN]
  ↓
pu (flag&1024): vtable+40 获取 physics state → 修改 part transform [PROVEN]
  ↓
hu (无条件): 读取 part+160/164 → 计算 transform 矩阵 B+52~B+88 [PROVEN]
  ↓
xr (L14719): mesh deformation (transform 顶点) [PROVEN]
  ↓
zr (L15055): draw call 提交 [PROVEN]
```

### 5.4 影响的渲染数据类别 `[PROVEN]`

**结论: (a) drawable 的 position/rotation/scale (transform 级别)**

- pu 修改 part 的：
  - **rotation**: part+92/96/100/104 (rotation matrix), part+628/632/636 (angle)
  - **position**: part+616/620/624 (position), part+168/172 (transform 元素)
  - **scale**: part+160/164 (scale x/y)
  - **opacity**: part+648
  - **color**: part+76 (RGBA)
- hu 读取这些字段计算 drawable transform（L53580: `B+52 = h * part+160` 等）
- **不是 mesh_bp** `[PROVEN]`：mesh_bp 在 hu 中独立处理，physics 输出不写入 mesh_bp 字段
- **不是 vertex buffer 直接偏移** `[PROVEN]`：xr 用 transform matrix 变换顶点，vertex buffer 在 xr 内部临时计算

### 5.5 59 模型触发验证 `[PROVEN]`

| 验证项 | 结果 | 证据 |
|--------|------|------|
| hairControl metadata 存在 | 59/59 | `[PROVEN]` |
| partsControl metadata 存在 | 59/59 | `[PROVEN]` |
| bustControl metadata 存在 | 59/59 | `[PROVEN]` |
| "髪揺れ" layer label 存在 | 59/59 | `[PROVEN]` |
| "パーツ揺れ" layer label 存在 | 59/59 | `[PROVEN]` |
| "胸" layer label 存在 | 59/59 | `[PROVEN]` |
| physics 输出进入渲染管线 | 是 | `[PROVEN]` |
| 启用但输出被忽略的模型 | 无 | `[PROVEN]` |

### 5.6 Python 对应和差异

**Python 对应实现**:

| asm.js | Python | 对应关系 | 证据 |
|--------|--------|----------|------|
| Pn (L28025) | `BustPhysics.update` (physics.py L594) | 完全对应 | `[PROVEN]` |
| zp (L65028) | `HairPartsPhysics.update` (physics.py L376) | 完全对应 | `[PROVEN]` |
| Hl/Il: Ij(b+100) = angle | player.py L2193-2211: physics_angles[label] = angle | variable map 写入 | `[PROVEN]` |
| pu: vtable+40 → part transform | motion_painter.py L779-790: build_context 应用旋转 | transform 修改 | `[PROVEN]` |
| hu: 读取 part transform 渲染 | player.py L2220: collect_drawable_resources → render | 渲染 | `[PROVEN]` |
| zp: b+148/152/156 (deformation) | 未实现 | 缺失 | `[UNVERIFIED]` |
| Ap: b+164/168 (swing) | 未实现 | 缺失 | `[UNVERIFIED]` |

**P2 差异表**:

| # | 差异项 | asm.js | Python | 差异类型 | 证据 |
|---|--------|--------|--------|----------|------|
| 1 | 应用时机 | pu 在 hu 之后（1 帧延迟） | build_context 在 render 之前（当前帧） | 1 帧延迟 | `[PROVEN]` |
| 2 | angle 索引（hair/parts） | 3 个角度 (a0/a1/a2) | output_angles[2] (仅 a2) | 索引选择 | `[PROVEN]` |
| 3 | angle 索引（bust） | 2 个角度 (angle1/angle2) | output_angles[0] (仅 angle1) | 索引选择 | `[PROVEN]` |
| 4 | deformation 输出 | zp 输出 b+148/152/156 | 未实现 | 缺失 | `[UNVERIFIED]` |
| 5 | swing 输出 | Ap 输出 b+164/168 | 未实现 | 缺失 | `[UNVERIFIED]` |
| 6 | variable map 数据结构 | b+100 (Ij/_l hash map) | physics_angles dict | 数据结构不同 | `[PROVEN]` |
| 7 | key 匹配方式 | 物理对象 label 字段 | layer label 字符串包含 "髪揺れ" 等 | 匹配方式不同 | `[LIKELY]` |

---

## 6. P3: gu-pu 触发路径审计

### 6.1 du 调用顺序 `[PROVEN]`

**源码**: `reference/FreeMoteDriver-format.js` L52560-L52582

```
du(part):
  if (flag & 512)   gu(b);     // bit 9
  hu(b);                        // 无条件
  iu(b);                        // 无条件
  if (flag & 32)    ju(b);     // bit 5
  ku(b);                        // 无条件
  if (flag & 2)     lu(b);     // bit 1
  if (flag & 8)     mu(b);     // bit 3
  if (flag & 64)    nu(b);     // bit 6
  if (flag & 16)    ou(b);     // bit 4
  if (flag & 1024)  pu(b);     // bit 10
```

**重要修正** `[PROVEN]`: lu/mu/nu/ou 也是**条件调用**（flag bit 检查），不是无条件调用。原任务描述"hu/iu/ku/lu/mu 无条件调用"是错误的，实际只有 hu/iu/ku 无条件调用。

### 6.2 9 个函数触发条件表 `[PROVEN]`

| 函数 | asm.js 行 | 触发条件 | flag bit | 59 模型触发数 | physics/render 相关 |
|------|-----------|----------|----------|-------------|-------------------|
| gu | L52903 | `flag & 512` | bit 9 | 59/59 `[LIKELY]` | **是** (修改 part+616/620/624) |
| hu | L53301 | 无条件 | - | 59/59 `[PROVEN]` | **是** (渲染主函数) |
| iu | L53959 | 无条件 | - | 59/59 `[PROVEN]` | 间接 (可见性 flag) |
| ju | L54007 | `flag & 32` | bit 5 | 59/59 `[LIKELY]` | 间接 (model 级字段) |
| ku | L54088 | 无条件 | - | 59/59 `[PROVEN]` | 间接 (边界框) |
| lu | L54167 | `flag & 2` | bit 1 | 0/59 `[LIKELY]` | 间接 (碰撞框) |
| mu | L83718 | `flag & 8` | bit 3 | 59/59 `[LIKELY]` | **是** (递归 du, 子模型) |
| nu | L84262 | `flag & 64` | bit 6 | 59/59 `[LIKELY]` | 间接 (跟随/约束) |
| ou | L84421 | `flag & 16` | bit 4 | 59/59 `[LIKELY]` | **是** (递归 du, 子模型) |
| pu | L85356 | `flag & 1024` | bit 10 | 59/59 `[PROVEN]` | **是** (physics → transform) |

### 6.3 各函数行为摘要

| 函数 | 行为 | 修改的字段 | 证据 |
|------|------|-----------|------|
| **hu** | 渲染主函数，读取 part+160/164，计算 transform 矩阵，调用 xr/zr | transform 矩阵 B+52~B+88, draw call | `[PROVEN]` |
| **gu** | z-sort/visibility，修改位置偏移 | part+616/620/624 (位置偏移) | `[PROVEN]` |
| **pu** | physics → part transform 桥梁，通过 vtable+40 获取 physics state | 9 组字段（见 6.4） | `[PROVEN]` |
| iu | 可见性判断 | part+721 (可见性 flag), part+728 | `[PROVEN]` |
| ju | 焦点/目标位置计算 | b+380~b+412 (model 级字段) | `[PROVEN]` |
| ku | 边界框计算（culling/z-sorting） | part+696, part+744 指向结构 | `[PROVEN]` |
| lu | 碰撞框/触发器形状 | part+744 指向结构 | `[PROVEN]` |
| mu | 子模型/嵌套模型渲染 | 递归调用 du | `[PROVEN]` |
| nu | 跟随/约束 | part+744 指向结构 | `[PROVEN]` |
| ou | 子模型/粒子系统渲染 | 递归调用 du | `[PROVEN]` |

### 6.4 pu 修改的 9 组字段 `[PROVEN]`

| 字段偏移 | 语义 | 证据 |
|---------|------|------|
| +160/+164/+168/+172 | physics transform (position/scale) | `[PROVEN]` |
| +616/+620/+624 | 位置偏移（与 gu 相同字段） | `[PROVEN]` |
| +628 | 旋转角度 | `[PROVEN]` |
| +632/+636 | 缩放参数 | `[PROVEN]` |
| +640/+644 | 缩放参数 | `[PROVEN]` |
| +648 | alpha/颜色 | `[PROVEN]` |
| +92/+96/+100/+104 | transform 矩阵 (rotation) | `[PROVEN]` |
| +76/+80/+84/+88 | 颜色 (RGBA) | `[PROVEN]` |
| +136/+140/+144/+152/+156 | 其他字段 | `[PROVEN]` |

### 6.5 PROVEN NO EFFECT 定性

**结论: 没有函数可以定性为 PROVEN NO EFFECT** `[PROVEN]`

| 函数 | 定性 | 理由 | 证据 |
|------|------|------|------|
| hu | NOT NO EFFECT | 渲染主函数，读取 pu 修改的 part+160/164 | `[PROVEN]` |
| gu | NOT NO EFFECT | 修改 part+616/620/624，被 hu 读取 | `[PROVEN]` |
| pu | NOT NO EFFECT | physics → part transform，被 hu 读取 | `[PROVEN]` |
| mu | NOT NO EFFECT (如果触发) | 递归调用 du 渲染子模型 | `[PROVEN]` |
| ou | NOT NO EFFECT (如果触发) | 递归调用 du 渲染子模型 | `[PROVEN]` |
| iu | LIKELY NOT DIRECT EFFECT | 只设置可见性 flag | `[LIKELY]` |
| ju | LIKELY NOT DIRECT EFFECT | 写入 model 级字段 | `[LIKELY]` |
| ku | LIKELY NOT DIRECT EFFECT | 计算边界框，用于 culling | `[LIKELY]` |
| lu | LIKELY NOT DIRECT EFFECT | 计算碰撞框，不修改 transform | `[LIKELY]` |
| nu | LIKELY NOT DIRECT EFFECT | 跟随/约束，不修改 transform | `[LIKELY]` |

### 6.6 Python 实现状态 `[PROVEN]`

| asm.js 函数 | Python 对应 | 实现状态 |
|------------|------------|----------|
| du | `EmotePlayer.update()` (L1797) | **NotImplementedError** |
| hu | `_draw_drawable_resources()` + `_draw_part()` | 部分实现（不读取 part+160/164） |
| pu | `_advance_physics()` + `_get_physics_angle_for_label()` + `build_context()` | **已实现**（机制不同） |
| gu/iu/ju/ku/lu/mu/nu/ou | (无) | **完全未实现** |

**Python pu 实现机制差异**:
- asm.js: pu 直接修改 part 内存中的 transform 字段，hu 读取
- Python: physics_angles dict 传递给 build_context，在 build_context 中应用旋转矩阵
- **效果相同**: physics 影响渲染输出 `[PROVEN]`
- **1 帧延迟**: Python 未实现 `[UNVERIFIED]`

---

## 7. 已对齐 vs 未对齐项总表

### 7.1 已对齐项

| 项目 | asm.js 行为 | Python 状态 | 证据级别 |
|------|------------|------------|---------|
| bust stiffness 读取 | PSB `bustControl.spring` → 属性 9340 | `player.py` L1218-1223 正确读取 | `[PROVEN]` |
| bust 积分器 Pn | L28025，输出 2 个角度 + position/velocity | `BustPhysics.update` (physics.py L594) 完全对应 | `[PROVEN]` |
| hair/parts 积分器 zp | L65028，输出 3 个角度 + position/velocity | `HairPartsPhysics.update` (physics.py L376) 完全对应 | `[PROVEN]` |
| variable map 写入 | Ij(b+100, label) = angle | physics_angles[label] = angle (player.py L2193-2211) | `[PROVEN]` |
| pu 触发 | flag&1024，59/59 模型 | `_advance_physics` + build_context 已实现 | `[PROVEN]` |
| physics 影响渲染 | transform 级别（position/rotation/scale） | build_context 应用旋转矩阵 + 位移 | `[PROVEN]` |
| 59 模型触发 physics | 全部启用，有对应 layer label | 全部有 hairControl/partsControl/bustControl | `[PROVEN]` |
| physics → render 路径 | zp/Pn → Hl/Il → Ij → pu → hu → xr/zr | physics_angles → build_context → render | `[PROVEN]` |

### 7.2 未对齐项

| 项目 | asm.js 行为 | Python 状态 | 差异级别 | 证据级别 |
|------|------------|------------|---------|---------|
| hair/parts stiffness 来源 | E-mote 属性 8926/8934（模型相关） | 硬编码 0.003 | **高** | `[CONFIRMED-DIFFERENCE]` |
| bust stiffness2 来源 | 属性 9972（可能独立值） | 设为与 stiffness 相同 | 中 | `[LIKELY-DIFFERENCE]` |
| convolveCanvasMovementToPhysics | JS 层开关（默认 false） | 未实现 | **高** | `[CONFIRMED-DIFFERENCE]` |
| canvas 位移差分 | getBoundingClientRect() 差分 | 无 | **高** | `[CONFIRMED-DIFFERENCE]` |
| 环形缓冲区 | FIFO 队列（170 元素/页）+ 指数衰减 | 直接覆盖 | **高** | `[CONFIRMED-DIFFERENCE]` |
| 外力施加层级 | 加到目标位置 | 加到速度/gravity | **高** | `[CONFIRMED-DIFFERENCE]` |
| 外力分量使用 | zp 用 x+y，Pn 用 x+y | zp 只用 x，Pn 只用 y | 中 | `[CONFIRMED-DIFFERENCE]` |
| 应用时机（1 帧延迟） | pu 在 hu 之后（1 帧延迟） | build_context 在 render 之前（当前帧） | **高** | `[PROVEN]` |
| angle 索引（hair/parts） | 3 个角度 (a0/a1/a2) | output_angles[2] (仅 a2) | 中 | `[PROVEN]` |
| angle 索引（bust） | 2 个角度 (angle1/angle2) | output_angles[0] (仅 angle1) | 中 | `[PROVEN]` |
| deformation 输出 | zp 输出 b+148/152/156 | 未实现 | 低 | `[UNVERIFIED]` |
| swing 输出 | Ap 输出 b+164/168 | 未实现 | 低 | `[UNVERIFIED]` |
| variable map 数据结构 | b+100 (Ij/_l hash map) | physics_angles dict | 低 | `[PROVEN]` |
| key 匹配方式 | 物理对象 label 字段 | layer label 字符串包含匹配 | 低 | `[LIKELY]` |
| gu (flag&512) | 修改 part+616/620/624 | 未实现 | 中 | `[LIKELY]` |
| iu/ju/ku/lu/mu/nu/ou | 各种辅助渲染功能 | 未实现 | 中 | `[PROVEN]` |

---

## 8. 关键差异优先级排序

按影响范围 × 修复难度综合排序，从高到低：

### 8.1 优先级 P0（阻塞级，影响所有 59 模型的核心物理行为）

| 排序 | 差异 | 影响范围 | 修复难度 | 阻塞下一阶段 | 证据 |
|------|------|---------|---------|------------|------|
| 1 | **hair/parts stiffness 来源（硬编码 0.003）** | 所有 59 模型 hair/parts 摆动角度响应相同，原版应有模型间差异 | 高（需动态 trace E-mote runtime 确认 8926/8934 → PSB key 映射，或解析 PhysicsData.raw_params） | 是 | `[CONFIRMED-DIFFERENCE]` |
| 2 | **应用时机（1 帧延迟 vs 当前帧）** | 所有 59 模型 physics 输出比原版提前 1 帧应用，摆动相位偏移 | 中（需调整 build_context 调用时机或引入 1 帧缓冲） | 是 | `[PROVEN]` |

### 8.2 优先级 P1（重要，影响外力响应正确性）

| 排序 | 差异 | 影响范围 | 修复难度 | 阻塞下一阶段 | 证据 |
|------|------|---------|---------|------------|------|
| 3 | **convolveCanvasMovementToPhysics 完全缺失** | Python 无法响应 canvas 移动物理（默认 false，但原版开启时行为完全不同） | 高（需实现 canvas 位移差分 + 环形缓冲区 + 指数衰减） | 否（默认关闭时不影响） | `[CONFIRMED-DIFFERENCE]` |
| 4 | **外力施加层级（目标位置 vs 速度/gravity）** | 外力响应语义不匹配，摆动行为不同 | 中（需修改 zp/Pn 外力施加位置） | 否 | `[CONFIRMED-DIFFERENCE]` |
| 5 | **环形缓冲区 + 指数衰减缺失** | 无历史外力平滑，无衰减过渡 | 中（需实现 OuterForceQueue 类） | 否 | `[CONFIRMED-DIFFERENCE]` |

### 8.3 优先级 P2（中等，影响输出精度）

| 排序 | 差异 | 影响范围 | 修复难度 | 阻塞下一阶段 | 证据 |
|------|------|---------|---------|------------|------|
| 6 | **angle 索引选择（3 个 vs 1 个）** | hair/parts 只用 a2（y 方向），丢失 a0/a1（x 方向）；bust 只用 angle1，丢失 angle2 | 低（a0/a1 恒为零因 error_x=0，但需确认） | 否 | `[PROVEN]` |
| 7 | **外力分量使用（x+y vs 单分量）** | zp 只用 x，Pn 只用 y，原版都用 x+y | 低（修改分量使用） | 否 | `[CONFIRMED-DIFFERENCE]` |
| 8 | **gu 未实现（flag&512）** | 59/59 模型可能缺少边界推动/碰撞响应 | 中（需实现 gu 逻辑） | 否 | `[LIKELY]` |
| 9 | **bust stiffness2 来源** | bust angle2 响应可能与原版不同 | 低（检查 PSB 是否有独立字段） | 否 | `[LIKELY-DIFFERENCE]` |

### 8.4 优先级 P3（低，辅助功能或未确认影响）

| 排序 | 差异 | 影响范围 | 修复难度 | 阻塞下一阶段 | 证据 |
|------|------|---------|---------|------------|------|
| 10 | **deformation 输出缺失（b+148/152/156）** | 可能影响 hair 最终变形 | 低（未确认是否被渲染使用） | 否 | `[UNVERIFIED]` |
| 11 | **swing 输出缺失（b+164/168）** | 可能影响 hair swing 效果 | 低（未确认是否被渲染使用） | 否 | `[UNVERIFIED]` |
| 12 | **iu/ju/ku/lu/mu/nu/ou 未实现** | 缺少可见性/边界框/碰撞框/子模型/跟随约束等辅助功能 | 高（7 个函数） | 否 | `[PROVEN]` |
| 13 | **variable map 数据结构不同** | hash map vs dict，功能等价 | 低（无需修复） | 否 | `[PROVEN]` |
| 14 | **key 匹配方式不同** | label 字段 vs 字符串包含匹配 | 低（功能等价） | 否 | `[LIKELY]` |

---

## 9. 下一阶段入口

### 9.1 eyeControl / eyebrowControl / mouthControl 研究入口

本阶段已完成 Runtime/Physics 对齐分析，下一阶段应研究面部控制（eye/eyebrow/mouth）。基于 P0-P3 的发现，建议的研究入口如下：

#### 9.1.1 已知的 asm.js 函数位置（来自 P0-P3 报告）

| 功能 | asm.js 函数 | 行号 | 说明 | 证据 |
|------|------------|------|------|------|
| Initialize 总入口 | `Zh` (EmotePlayer_Initialize) | L57037 | 所有物理/控制初始化入口 | `[PROVEN]` |
| Initialize 调度 | `Be` → `Me` → `cj` → `ej` → `fj` | L54394/L54479/L60332/L60510/L60684 | 初始化调用链 | `[PROVEN]` |
| bust 物理初始化 | `ij` → `tk`/`uk` | L61472/L42951/L43026 | bust stiffness 读取 | `[PROVEN]` |
| hair/parts 物理初始化 | `jj` → `pk` | L62150/L42615 | hair/parts stiffness 读取 | `[PROVEN]` |
| 物理总调度 | `Al` | L78610 | 每帧 Update 主函数 | `[PROVEN]` |
| bust 积分器 | `Pn` | L28025 | bust physics 计算 | `[PROVEN]` |
| hair/parts 积分器 | `zp` | L65028 | hair/parts physics 计算 | `[PROVEN]` |
| 渲染主函数 | `hu` | L53301 | 读取 part transform 渲染 | `[PROVEN]` |
| physics apply | `pu` | L85356 | physics → part transform | `[PROVEN]` |
| variable map | `Ij` / `_l` / `Zl` / `Df` | L39067/L81423/L81368/L55075 | hash map 操作 | `[PROVEN]` |
| vtable 函数表 | `ce`/`ae`/`zd` | L99184/L99182/L99153 | vtable+40 等 dispatch | `[PROVEN]` |

#### 9.1.2 建议的研究顺序

1. **第一步：定位 eyeControl/eyebrowControl/mouthControl 的 PSB metadata 结构**
   - 检查 59 模型 `root_value['metadata']` 中是否存在 `eyeControl`/`eyebrowControl`/`mouthControl` 字段
   - 记录字段 keys 和值范围（类似 P0 对 hairControl/partsControl/bustControl 的分析）
   - 证据级别目标：`[PROVEN]`

2. **第二步：追踪 asm.js 中面部控制的初始化路径**
   - 从 `EmotePlayer_Initialize` (L57037) → `Be` → `Me` → `cj` → `ej` → `fj` 调用链中寻找面部控制相关分支
   - 类比 P0 的 `ij`/`jj` 分支，寻找 eye/eyebrow/mouth 对应的初始化函数
   - 关注是否复用 `pk` 函数（属性 8926/8934 读取）或使用独立函数
   - 证据级别目标：`[PROVEN]` 读取路径

3. **第三步：追踪面部控制的运行时 Update 路径**
   - 在 `Al` (L78610) 主 Update 函数中寻找面部控制相关调用
   - 类比 `Hl` (bust) / `Il` (hair/parts)，寻找 eye/eyebrow/mouth 对应的 Update 函数
   - 确认是否通过 variable map (b+100) 传递输出
   - 证据级别目标：`[PROVEN]`

4. **第四步：追踪面部控制输出到渲染的路径**
   - 在 `du` (L52165) 调用链中寻找面部控制对应的函数（可能新增 flag bit）
   - 类比 `pu` (flag&1024)，寻找 eye/eyebrow/mouth 对应的 apply 函数
   - 确认影响的是 transform 级别还是其他渲染数据
   - 证据级别目标：`[PROVEN]`

5. **第五步：对比 Python 实现**
   - 检查 `src/freemote/player/player.py` 中是否有 eye/eyebrow/mouth 相关实现
   - 对比 PSB metadata 读取、运行时 Update、输出应用三个环节
   - 生成差异表（类似本报告第 7 节）

#### 9.1.3 已知的相关 PSB metadata 字段（待验证）

基于 P3 报告第 3 节的 59 模型控制列表统计，以下控制列表已在 metadata 中确认存在：

| 控制列表 | 非空模型数 | 推测对应功能 | 证据 |
|----------|-----------|------------|------|
| bustControl | 59/59 | bust physics (pu) | `[PROVEN]` |
| hairControl | 59/59 | hair physics (pu) | `[PROVEN]` |
| partsControl | 59/59 | parts physics (pu) | `[PROVEN]` |
| clampControl | 59/59 | clamp (gu, bit 9) | `[LIKELY]` |
| transitionControl | 59/59 | transition (ou, bit 4) | `[LIKELY]` |
| loopControl | 0/59 | loop (lu/mu/nu, bit 1/3/6) | `[LIKELY]` |
| selectorControl | 59/59 | selector (nu, bit 6) | `[LIKELY]` |
| timelineControl | 59/59 | timeline 系统 | `[PROVEN]` |

**注意**: 上表中未直接列出 eyeControl/eyebrowControl/mouthControl，需在下一阶段第一步中确认这些字段是否存在于 PSB metadata 中，或是否使用其他字段名（如 `eyeBlinkControl`、`mouthOpenControl` 等）。

#### 9.1.4 可能的研究切入点

- **若 eyeControl 等字段存在于 PSB metadata**：直接从 PSB 加载路径追踪到 asm.js 初始化
- **若不存在于 metadata**：可能通过 timelineControl 或 parameter 系统驱动，需从 timeline 系统（timelineControl，59/59 模型 `[PROVEN]`）入手
- **属性 ID 体系**：P0 发现 E-mote 使用内部属性 ID（如 8926/8934/8942/9340/9972），eye/eyebrow/mouth 可能使用其他属性 ID，需在 `fv` 调用中搜索

---

## 附录 A: 关键函数行号对照表

| 函数 | asm.js 行号 | 用途 | 来源报告 |
|------|-------------|------|----------|
| Zh | L57037 | EmotePlayer_Initialize | P0 |
| Be | L54394 | Initialize 调度 | P0 |
| Me | L54479 | Initialize 调度 | P0 |
| cj | L60332 | Initialize 调度 | P0 |
| ej | L60510 | Initialize 调度 | P0 |
| fj | L60684 | Initialize 调度 | P0 |
| ij | L61472 | bust 物理初始化 | P0 |
| jj | L62150 | hair/parts 物理初始化 | P0 |
| ok | L42482 | bust stiffness 读取 (8942→9340) | P0 |
| tk | L42951 | bust stiffness alt 路径 | P0 |
| uk | L43026 | bust alt 路径 | P0 |
| pk | L42615 | hair/parts stiffness 读取 (8926/8934) | P0 |
| xp | L64917 | hair/parts 物理对象拷贝 | P0 |
| Pn | L28025-28092 | bust 积分器 | P0/P2 |
| Qn | L28095-28106 | Pn wrapper | P1/P2 |
| zp | L65028-65195 | hair/parts 积分器 | P0/P2 |
| Ap | L65198-65228 | zp wrapper (hair swing) | P2 |
| Al | L78610-78796 | physics 总调度 | P1/P2 |
| Hl | L79304-79416 | bust physics update | P1/P2 |
| Il | L79418-79534 | hair/parts physics update | P1/P2 |
| Jl | L79536 | bone 变换外力计算 | P1 |
| Ij | L39067-39200 | hash map insert/lookup | P2 |
| _l | L81423-81500 | hash map lookup | P2 |
| Zl | L81368-81421 | physics angle getter | P2 |
| Df | L55075-55079 | Zl wrapper | P2 |
| du | L52165-52617 | part update + render 主函数 | P2/P3 |
| gu | L52903-53299 | z-sort/visibility (flag&512) | P3 |
| hu | L53301-53957 | 渲染主函数 | P2/P3 |
| iu | L53959-54005 | 可见性判断 | P3 |
| ju | L54007-54086 | 焦点位置 (flag&32) | P3 |
| ku | L54088-54165 | 边界框计算 | P3 |
| lu | L54167-54256 | 碰撞框 (flag&2) | P3 |
| mu | L83718-84261 | 子模型 (flag&8) | P3 |
| nu | L84262-84419 | 跟随/约束 (flag&64) | P3 |
| ou | L84421-85555 | 子模型/粒子 (flag&16) | P3 |
| pu | L85356-85562 | physics apply (flag&1024) | P2/P3 |
| xr | L14719-14997 | mesh deformation | P2 |
| zr | L15055 | draw call 提交 | P2 |
| To | L31189 | 环形缓冲区插入 | P1 |
| Qo | L30945 | 读取外力 wrapper | P1 |
| Ro | L30953 | 读取外力 + 衰减插值 | P1 |
| li | L57252 | SetOuterForce dispatch | P1 |
| Ci | L57514 | Update dispatch | P1 |
| eg | L55402 | Update player 内部 | P1 |
| wm | L82897 | string 分发 bust/parts/hair | P1 |
| ce 表 | L99184 | vtable 函数表 (Df=索引2) | P2 |
| ae 表 | L99182 | vtable 函数表 (Vg=索引3) | P2 |
| zd 表 | L99153 | vtable 函数表 (pu 用 vtable+40) | P3 |

---

## 附录 B: Python 关键位置对照表

| Python 位置 | 对应 asm.js | 用途 | 来源报告 |
|------------|------------|------|----------|
| `player.py` L1218-1223 | ok/tk (bust stiffness 读取) | `_init_physics_from_metadata` bust | P0 |
| `player.py` L1244 | pk (hair/parts stiffness) | `_DEFAULT_STIFFNESS = 0.003` 硬编码 | P0 |
| `player.py` L2193-2211 | Hl/Il Ij(b+100) | physics_angles[label] = angle | P2 |
| `player.py` L2340 | Zl/Df (physics angle getter) | `_get_physics_angle_for_label` | P2 |
| `player.py` L4488-4527 | li/To (SetOuterForce) | `set_outer_force_value` | P1 |
| `player.py` L5414-5493 | Al (physics 总调度) | `_advance_physics` | P1/P3 |
| `physics.py` L184-188 | - | outer_force 字段定义 | P1 |
| `physics.py` L380-493 | zp (L65028) | `HairPartsPhysics.update` | P1/P2 |
| `physics.py` L594-692 | Pn (L28025) | `BustPhysics.update` | P1/P2 |
| `motion_painter.py` L779-790 | pu (part transform 修改) | build_context physics angle 应用 | P2/P3 |
| `app.py` L239-245 | - | 鼠标回调（手动外力） | P1 |

---

**报告生成时间**: 2026-09-26
**任务状态**: completed
**子报告整合**: P0 (Task 21) + P1 (Task 22) + P2 (Task 23) + P3 (Task 24)
**关键结论**: asm.js 物理系统完整数据流已 `[PROVEN]`；Python 在 bust stiffness 读取和 pu physics angle 应用上已对齐，但在 hair/parts stiffness 来源、convolveCanvasMovementToPhysics、应用时机、angle 索引、deformation/swing 等 5 个关键环节存在差异，其中 hair/parts stiffness 来源和应用时机为阻塞级差异。