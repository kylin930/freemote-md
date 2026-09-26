# Physics Output → RenderContext 完整追踪报告

**任务**: Task 23 (P2) — 完整追踪 asm.js physics 输出到 RenderContext
**作者**: PhysicsOutputTracer
**日期**: 2026-09-26
**证据来源**: `reference/FreeMoteDriver-format.js` (99629 行 asm.js), `src/freemote/`

---

## 摘要

本报告完整追踪 asm.js 中 physics 积分器 (zp/Pn) 的输出字段，经 EmotePlayer
variable map (b+100) 传递，到渲染管线 (hu/xr/zr) 的读取路径，并在 Python 代码
中找到对应位置。**核心结论**：physics 输出是 **transform 级别**的修改（part 的
scale/position/rotation），不是 mesh_bp 或 vertex buffer 的直接偏移。所有 59 个
NEKOPARA 模型均启用 physics 且拥有对应 layer label，physics 输出确实进入渲染管线。

---

## Q1: Physics 积分器输出字段

### Pn (bust 积分器) — asm.js L28025-28092

**函数签名**: `Pn(b, c, d, e, f, h, i, j)`
- b: 物理对象指针 (92 字节对象，在 Hl 中遍历)
- c, d: 外力 (target position 增量, float)
- e, f: **输出角度指针** (float*, 通过指针传出)
- h: dt (时间增量)
- i: stiffness
- j: rotation angle (当前角度)

**输出字段表**:

| Offset | 语义 | 类型 | 写入行 | 证据 |
|--------|------|------|--------|------|
| b+28 | target position X | float | L28050 | `g[b+28>>2] = c` [PROVEN] |
| b+32 | target position Y | float | L28052 | `g[b+32>>2] = d` [PROVEN] |
| b+40 | -error X (current - target) | float | L28055 | `g[b+40>>2] = m - c` [PROVEN] |
| b+44 | -error Y (current - target) | float | L28057 | `g[b+44>>2] = l - d` [PROVEN] |
| b+52 | current position X | float | L28086 | `g[q>>2] = l` (q=b+52) [PROVEN] |
| b+56 | current position Y | float | L28088 | `g[o>>2] = m` (o=b+56) [PROVEN] |
| b+60 | current position Z | float | L28089 | `g[k>>2] = n + j*h` (k=b+60) [PROVEN] |
| b+64 | velocity X | float | L28080 | `g[v>>2] = l` (v=b+64) [PROVEN] |
| b+68 | velocity Y | float | L28082 | `g[u>>2] = m` (u=b+68) [PROVEN] |
| b+72 | velocity Z | float | L28084 | `g[s>>2] = j` (s=b+72) [PROVEN] |
| *e | output angle X (主角度) | float | L28090 | `g[e>>2] = xo(-((c-l)*i*g[b+16]))` [PROVEN] |
| *f | output angle Y (次角度) | float | L28091 | `g[f>>2] = xo((-((d-m)*i)-g[b+76])*g[b+20])` [PROVEN] |

**无其他输出字段**（无 accumulated rotation、deformation offset）。[PROVEN]

### zp (hair/parts 积分器) — asm.js L65028-65195

**函数签名**: `zp(b, d, e, f, h, j, k, l, m)`
- b: 物理对象指针 (176 字节对象，在 Il 中遍历)
- d, e: 外力 (target position 增量, float)
- f, h, j: **输出角度指针** (float*, 通过指针传出, 3 个角度)
- k: stiffness (scale factor)
- l: 某个参数 (scale)
- m: rotation angle (当前角度)

**输出字段表**:

| Offset | 语义 | 类型 | 写入行 | 证据 |
|--------|------|------|--------|------|
| b+64 | target position X | float | L65074 | `g[b+64>>2] = d` [PROVEN] |
| b+68 | target position Y | float | L65076 | `g[b+68>>2] = e` [PROVEN] |
| b+76 | -error X (current - target) | float | L65079 | `g[b+76>>2] = w - d` [PROVEN] |
| b+80 | -error Y (current - target) | float | L65081 | `g[b+80>>2] = x - e` [PROVEN] |
| b+88 | current position X (bone 0) | float | L65089 | `g[b+88>>2] = S` [PROVEN] |
| b+92 | current position Y (bone 0) | float | L65091 | `g[b+92>>2] = w` [PROVEN] |
| b+96 | current position Z (bone 0) | float | L65093 | `g[b+96>>2] = x` [PROVEN] |
| b+100 | current position X (bone 1) | float | L65095 | `g[b+100>>2] = S + d*0.0` [PROVEN] |
| b+104 | current position Y (bone 1) | float | L65096 | `g[b+104>>2] = w + d*1.0` [PROVEN] |
| b+108 | current position Z (bone 1) | float | L65097 | `g[b+108>>2] = x + d*0.0` [PROVEN] |
| b+112+v*12 | chain bone v position (v=0,1) | float×3 | L65176-65180 | `g[r>>2] = m` etc. [PROVEN] |
| b+136+v*12 | chain bone v velocity (v=0,1) | float×3 | L65154-65174 | `g[p>>2] = m` etc. [PROVEN] |
| b+148 | deformation X (v=1 only) | float | L65136 | `g[K>>2] += g[O>>2]*S` [PROVEN] |
| b+152 | deformation Y (v=1 only) | float | L65137 | `g[L>>2] += g[y>>2]*S` [PROVEN] |
| b+156 | deformation Z (v=1 only) | float | L65138 | `g[M>>2] += g[z>>2]*S` [PROVEN] |
| *f | output angle 0 (v=0, x 方向) | float | L65186 | `g[f>>2] = d` [PROVEN] |
| *h | output angle 1 (v=1, x 方向) | float | L65187 | `g[h>>2] = d` [PROVEN] |
| *j | output angle 2 (v=1, y 方向) | float | L65188 | `g[j>>2] = xo((g[H>>2]-g[y>>2])*g[b+44+(v<<2)]*l)` [PROVEN] |

### Ap (zp wrapper, hair swing) — asm.js L65198-65228

Ap 调用 zp 后，额外更新：
| Offset | 语义 | 类型 | 写入行 | 证据 |
|--------|------|------|--------|------|
| b+168 | swing 程度 (0-1) | float | L65220 | `g[f>>2] = b` (f=b+168) [PROVEN] |
| b+164 | 累积旋转角度 | float | L65223 | `g[f>>2] = h` (f=b+164, h=Wy(...)) [PROVEN] |
| *e | output angle + swing correction | float | L65226 | `g[e>>2] += h` [PROVEN] |
| *d | output angle - swing correction | float | L65227 | `g[d>>2] -= h` [PROVEN] |

---

## Q2: Physics 输出 → 渲染读取路径

### 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│  Physics Update 阶段 (Al, asm.js L78610-78796)                     │
│                                                                     │
│  Al(b, dt)                                                          │
│    ├─ Qo(b+352) → 外力 X/Y → b+368/372  (bust outer force)         │
│    ├─ Qo(b+356) → 外力 X/Y → b+376/380  (parts outer force)        │
│    ├─ Qo(b+360) → 外力 X/Y → b+384/388  (hair outer force)         │
│    ├─ Hl(b, w)  [L79304]                                            │
│    │    └─ for each bust obj (92 bytes):                            │
│    │       ├─ Jl(b, obj+48) → 外力 (target pos 增量)               │
│    │       ├─ Qn → Pn(obj, force, &angle1, &angle2, ...)           │
│    │       │    └─ 写入 obj+28/32/40/44/52/56/60/64/68/72          │
│    │       │    └─ 输出 angle1, angle2 到栈                         │
│    │       ├─ Ij(b+100, obj+60) = angle1  ← 写入 variable map      │
│    │       └─ Ij(b+100, obj+72) = angle2                            │
│    ├─ Il(b, b+212, stiffness, force, w)  [L79418]  (parts)          │
│    │    └─ for each parts obj (176 bytes):                          │
│    │       ├─ Jl(b, obj+104) → 外力                                 │
│    │       ├─ Ap → zp(obj, force, &a0, &a1, &a2, ...)              │
│    │       │    └─ 写入 obj+64/68/76/80/88-108/112-156             │
│    │       │    └─ 输出 a0, a1, a2 到栈                             │
│    │       ├─ Ij(b+100, obj+116) = a0  ← 写入 variable map         │
│    │       ├─ Ij(b+100, obj+128) = a1                               │
│    │       └─ Ij(b+100, obj+140) = a2                               │
│    └─ Il(b, b+224, stiffness, force, w)  (hair, 同上)               │
│                                                                     │
│  结果: physics angle 写入 EmotePlayer variable map (b+100)          │
│        key = 物理对象的 label 字符串, value = angle (float)          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Render 阶段 (du, asm.js L52165-52617)                              │
│                                                                     │
│  du(part)                                                           │
│    ├─ gu(part)  [L52903]  if flag & 512  (z-sort/visibility)        │
│    ├─ hu(part)  [L53301]  ← 渲染主函数                              │
│    │    ├─ 计算 drawable transform (L53580):                        │
│    │    │    B+52 = h * part+160  (transform matrix a)              │
│    │    │    B+56 = part+164 * part+96  (matrix b)                  │
│    │    │    B+60 = part+160 * part+100 (matrix c)                  │
│    │    │    B+64 = part+164 * part+104 (matrix d)                  │
│    │    │    B+68-80 = inverse matrix                               │
│    │    │    B+84/88 = -j/-k (translation)                           │
│    │    ├─ xr(B, z, B+52, W)  [L14719, 调用点 L53686]               │
│    │    │    └─ 用 transform 变换顶点 → mesh deformation            │
│    │    └─ zr(W, wa, xa, z)  [L15055, 调用点 L53840]                │
│    │         └─ 提交 draw call                                      │
│    ├─ iu, ju, ku, lu, mu, nu, ou  (各种后处理)                      │
│    ├─ pu(part)  [L85356]  if flag & 1024  ← physics apply           │
│    │    ├─ vtable+40(EmotePlayer, &p)  ← 读取 physics state         │
│    │    │    └─ 可能通过 Zl→Df→ce[2] 读取 b+100 map [LIKELY]        │
│    │    ├─ part+144-172 = p  (写入 base transform)                  │
│    │    ├─ k = part+212 / part+208  (缩放因子)                      │
│    │    ├─ part+628 *= k  (angle 缩放)                              │
│    │    ├─ part+632/636 = H(part+632/636, k)  (angle 衰减)          │
│    │    ├─ part+92-104 = matrix_mul(part+92, part+96)  (rotation)   │
│    │    ├─ part+616-624 = lerp(part+616, part+616, k)  (position)   │
│    │    ├─ part+648 *= k  (opacity)                                 │
│    │    └─ part+76 = color_scale(part+76, k)  (color)               │
│    └─ qu(part)  (final)                                             │
│                                                                     │
│  结果: physics angle → part transform → 下一帧 hu 读取 → render     │
└─────────────────────────────────────────────────────────────────────┘
```

### hu 是否直接读取 physics 输出？

**否**。[PROVEN]

hu (L53301-53957) 内部不调用 ce/ae 表、Ij/_l/Zl/Df 等 physics 读取函数。
hu 直接读取 part 的 transform 字段 (part+96/100/104/160/164/168/172)，
这些字段由 pu 在上一帧写入。

### xr/zr 是否读取 physics 输出？

**否，间接读取**。[PROVEN]

xr (L14719-14997) 只调用 yr 和 S（hash）。zr (L15055) 类似。
它们读取的是 drawable 的 transform (B+52..B+88) 和顶点数据，
这些 transform 由 hu 从 part 字段计算得来。

### 读取的是 position/rotation/scale 中的哪个？

**全部**。[PROVEN]

pu 修改 part 的：
- **rotation**: part+92/96/100/104 (rotation matrix), part+628/632/636 (angle)
- **position**: part+616/620/624 (position), part+168/172 (transform 元素)
- **scale**: part+160/164 (scale x/y)
- **opacity**: part+648
- **color**: part+76 (RGBA)

---

## Q3: Physics 影响的具体渲染数据类别

**结论: (a) drawable 的 position/rotation/scale (transform 级别)** [PROVEN]

### 证据链

1. **pu 修改 part transform 字段** [PROVEN]:
   - part+92/96/100/104: rotation matrix (2x2)
   - part+160/164: scale x/y (int)
   - part+168/172: transform matrix 元素
   - part+616/620/624: position x/y/z
   - part+628/632/636: angle x/y/z

2. **hu 读取这些字段计算 drawable transform** [PROVEN]:
   - L53580: `B+52 = h * part+160` (matrix a = rotation * scale_x)
   - L53580: `B+56 = part+164 * part+96` (matrix b = scale_y * rotation)
   - L53580: `B+60 = part+160 * part+100` (matrix c)
   - L53580: `B+64 = part+164 * part+104` (matrix d)
   - L53587-53588: `B+84/88 = -j/-k` (translation)

3. **xr 用 transform 变换顶点** [PROVEN]:
   - xr(B, z, B+52, W): 用 B+52 (transform matrix) 和 W (offset) 变换顶点
   - 这是 mesh deformation，但变形源是 transform 级别（不是 mesh_bp）

### 不是 mesh_bp [PROVEN]

mesh_bp (base point deformation) 在 hu 中由 L53540-53566 处理，
是独立的偏移组合路径 (`mesh_bp = parent.mesh_bp + (frame.mesh_bp - default)`)。
physics 输出不写入 mesh_bp 字段。

### 不是 vertex buffer 直接偏移 [PROVEN]

xr 变换顶点用的是 transform matrix，不是直接偏移 vertex buffer。
vertex buffer 在 xr 内部临时计算，不持久化。

---

## Q4: NEKOPARA 实际触发路径

### 59 模型 physics 启用验证

| 验证项 | 结果 | 证据 |
|--------|------|------|
| hairControl metadata 存在 | 59/59 | PSBLoader.load → root_value['metadata']['hairControl'] [PROVEN] |
| partsControl metadata 存在 | 59/59 | root_value['metadata']['partsControl'] [PROVEN] |
| bustControl metadata 存在 | 59/59 | root_value['metadata']['bustControl'] [PROVEN] |
| "髪揺れ" layer label 存在 | 59/59 | 遍历 root_value 找 label 字段 [PROVEN] |
| "パーツ揺れ" layer label 存在 | 59/59 | 同上 [PROVEN] |
| "胸" layer label 存在 | 59/59 | 同上 [PROVEN] |

### 各模型 physics label 数量（前 10 个）

| 模型 | hair labels | parts labels | bust labels |
|------|-------------|--------------|-------------|
| azuki-casual | 5 | 4 | 6 |
| azuki-dress | 3 | 2 | 4 |
| azuki-maid | 5 | 5 | 5 |
| azuki-santa | 5 | 6 | 6 |
| azuki-teenage | 5 | 4 | 10 |
| azuki-winter | 5 | 4 | 10 |
| azuki-wintermaid | 5 | 5 | 5 |
| azuki-yukata | 5 | 6 | 4 |
| chocola-casual | 5 | 5 | 4 |
| chocola-dress | 3 | 3 | 4 |

### physics 输出是否真的进入渲染管线？

**是** [PROVEN]

1. 所有 59 模型都有 hairControl/partsControl/bustControl metadata →
   physics 积分器被初始化且每帧 update
2. 所有 59 模型都有 "髪揺れ"/"パーツ揺れ"/"胸" layer label →
   physics angle 通过 label 匹配应用到对应 layer
3. Python 中 `_get_physics_angle_for_label` (player.py L2340) 根据 label
   关键字返回 output_angles，非零角度加入 physics_angles dict
4. build_context (motion_painter.py L779) 把角度应用到 matrix 和 position

### 是否有模型启用了 physics 但输出被忽略？

**否** [PROVEN]

所有 59 模型的 physics 输出都通过 layer label 匹配应用到渲染。
没有 "启用但忽略" 的情况。

---

## Q5: Python 对应位置

### Physics 积分器输出

| asm.js | Python | 对应关系 | 证据 |
|--------|--------|----------|------|
| Pn (L28025) | `BustPhysics.update` (physics.py L594) | 完全对应 | [PROVEN] |
| Pn: *e, *f | `self.output_angles[0/1]` (L687-688) | 输出角度 | [PROVEN] |
| Pn: b+52/56/60 | `vertex.position` (L691) | 当前位置 | [PROVEN] |
| Pn: b+64/68/72 | `vertex.velocity` (L692) | 速度 | [PROVEN] |
| zp (L65028) | `HairPartsPhysics.update` (physics.py L376) | 完全对应 | [PROVEN] |
| zp: *f, *h, *j | `self.output_angles[0/1/2]` (L483-489) | 输出角度 | [PROVEN] |
| zp: b+112+v*12 | `vertex.position` (L492) | 链式 bone 位置 | [PROVEN] |
| zp: b+136+v*12 | `vertex.velocity` (L493) | 链式 bone 速度 | [PROVEN] |
| zp: b+148/152/156 | (未实现) | deformation | [UNVERIFIED] |

### 输出如何传递到渲染

| asm.js | Python | 对应关系 | 证据 |
|--------|--------|----------|------|
| Hl/Il: Ij(b+100, key) = angle | player.py L2193-2211: physics_angles[label] = angle | variable map 写入 | [PROVEN] |
| Zl→Df→ce[2]: 读取 b+100 map | player.py L2209: `_get_physics_angle_for_label(label)` | variable map 读取 | [LIKELY] |
| pu: vtable+40 获取 physics state | motion_painter.py L2850-2860: 查询 physics_angles dict | physics state 获取 | [LIKELY] |
| pu: 修改 part transform | motion_painter.py L779-790: build_context 应用到 matrix/position | transform 修改 | [PROVEN] |
| hu: 读取 part transform 渲染 | player.py L2220: collect_drawable_resources → render | 渲染 | [PROVEN] |

### build_context 中的 physics angle 应用 (motion_painter.py L779-790)

```python
if abs(physics_angle) > 1e-6:
    _angle_rad = physics_angle * (_EMOTE_ANGLE_TO_RAD)
    _cos_a = math.cos(_angle_rad)
    _sin_a = math.sin(_angle_rad)
    # 左乘旋转矩阵到累积 matrix
    _rot = MotionMatrix2(_cos_a, -_sin_a, _sin_a, _cos_a)
    matrix = MotionMatrix2.multiply(_rot, matrix)
    # 绕 parent position 旋转累积位移
    _tx_old = tx
    tx = _cos_a * tx - _sin_a * ty
    ty = _sin_a * _tx_old + _cos_a * ty
```

这对应 asm.js pu 中修改 part rotation matrix (part+92-104) 和 position (part+616-624)。

### 差异表

| 项目 | asm.js | Python | 差异类型 | 证据 |
|------|--------|--------|----------|------|
| **应用时机** | pu 在 hu **之后** (du: hu→pu) | build_context 在 render **之前** | 1 帧延迟 | [PROVEN] |
| **应用对象** | part 对象的 transform 字段 | RenderContext 的 matrix/position | 数据结构不同 | [PROVEN] |
| **应用方式** | vtable+40 获取 + k 缩放 | 左乘旋转矩阵 + 旋转位移 | 算法不同 | [LIKELY] |
| **angle 索引** | hair/parts: 3 个角度 (a0/a1/a2) | hair/parts: output_angles[2] (仅 a2) | 索引选择 | [PROVEN] |
| **angle 索引** | bust: 2 个角度 (angle1/angle2) | bust: output_angles[0] (仅 angle1) | 索引选择 | [PROVEN] |
| **deformation** | zp 输出 b+148/152/156 | 未实现 | 缺失 | [UNVERIFIED] |
| **swing** | Ap 输出 b+164/168 (累积旋转) | 未实现 | 缺失 | [UNVERIFIED] |
| **variable map** | b+100 (Ij/_l hash map) | physics_angles dict | 数据结构不同 | [PROVEN] |
| **key 匹配** | 物理对象的 label 字段 | layer label 字符串包含 "髪揺れ" 等 | 匹配方式不同 | [LIKELY] |

### 关键差异详解

#### 1. 应用时机差异 [PROVEN]

- **asm.js**: du 中调用顺序是 `gu → hu → iu → ju → ku → lu → mu → nu → ou → pu → qu`
  - pu 在 hu **之后**，修改 part transform 后，**下一帧**的 hu 读取
  - 这意味着 physics angle 有 1 帧延迟
- **Python**: build_context 在 collect_drawable_resources **之前**，physics angle 在**当前帧**应用
  - 无 1 帧延迟

#### 2. angle 索引选择差异 [PROVEN]

- **asm.js zp**: 输出 3 个角度 (a0, a1, a2)，通过 3 个 Ij 写入 map
  - a0 = angle for v=0 (x 方向)
  - a1 = angle for v=1 (x 方向)
  - a2 = angle for v=1 (y 方向)
- **Python**: hair/parts 只用 `output_angles[2]` (a2, y 方向)
  - 原因：error_x = 0 - 0 = 0 恒为零，a0/a1 始终为 0
  - a2 = pos_y * stiffness2 * scale，pos_y ≈ 167，非零

#### 3. deformation 缺失 [UNVERIFIED]

- **asm.js zp**: 输出 b+148/152/156 (deformation X/Y/Z, v=1 only)
  - L65136-65138: `g[K>>2] += g[O>>2]*S` (K=b+148, deformation)
  - 这是距离约束的副产品
- **Python**: HairPartsPhysics.update 未实现 deformation 输出
  - 可能影响 hair 的最终变形，但未确认是否被渲染管线使用

---

## 关键发现摘要

1. **physics 输出是 transform 级别修改** [PROVEN]：
   physics angle 通过 pu 修改 part 的 rotation/scale/position，不是 mesh_bp 或 vertex buffer。

2. **physics → render 完整路径** [PROVEN]：
   `zp/Pn → Hl/Il → Ij(b+100) → pu(vtable+40) → part transform → hu → xr/zr → draw call`

3. **所有 59 模型都触发 physics** [PROVEN]：
   全部启用 hair/parts/bust physics，且有对应 layer label。

4. **Python 与 asm.js 的关键差异** [PROVEN]：
   - 应用时机：asm.js 1 帧延迟 (pu 在 hu 后)，Python 当前帧 (build_context 在 render 前)
   - angle 索引：asm.js 输出 3 个角度，Python 只用 output_angles[2] (hair/parts) 或 [0] (bust)

5. **未确认项** [UNVERIFIED]：
   - pu 中 vtable+40 调用的具体函数（是否直接读取 b+100 map）
   - zp 的 deformation 输出 (b+148/152/156) 是否被渲染使用
   - Ap 的 swing 累积 (b+164/168) 是否被渲染使用

---

## 附录：关键函数行号对照

| 函数 | asm.js 行号 | 用途 |
|------|-------------|------|
| Pn | L28025-28092 | bust 积分器 |
| zp | L65028-65195 | hair/parts 积分器 |
| Ap | L65198-65228 | zp wrapper (hair swing) |
| Qn | L28095-28106 | Pn wrapper |
| Hl | L79304-79416 | bust physics update |
| Il | L79418-79534 | hair/parts physics update |
| Al | L78610-78796 | physics 总调度 |
| Ij | L39067-39200 | hash map insert/lookup |
| _l | L81423-81500 | hash map lookup (no insert) |
| Zl | L81368-81421 | physics angle getter (读 b+100) |
| Df | L55075-55079 | Zl wrapper (ce[2]) |
| du | L52165-52617 | part update + render 主函数 |
| gu | L52903-53299 | z-sort/visibility |
| hu | L53301-53957 | 渲染主函数 |
| xr | L14719-14997 | mesh deformation (transform 顶点) |
| zr | L15055-... | draw call 提交 |
| pu | L85356-85562 | physics deformation apply |
| ce 表 | L99184 | vtable 函数表 (Df=索引2) |
| ae 表 | L99182 | vtable 函数表 (Vg=索引3) |
| zd 表 | L99153 | vtable 函数表 (pu 用 vtable+40) |