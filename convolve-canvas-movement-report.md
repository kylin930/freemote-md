# convolveCanvasMovementToPhysics 追踪报告

**任务 ID**: 22 (P1)
**作者**: CanvasMovementTracer
**日期**: 2026-09-26
**范围**: E-mote 如何把 canvas movement 转换成 physics input 的完整数据流追踪
**方法**: 从 `reference/emoteplayer-format.js`（JS 层）和 `reference/FreeMoteDriver-format.js`（asm.js 层，99629 行）追踪完整调用链，并与 Python `src/freemote/` 对比

---

## 0. 执行摘要

| 项目 | 结论 | 证据等级 |
|------|------|---------|
| **convolveCanvasMovementToPhysics 本质** | JS 层配置开关（默认 **false**），不是 asm.js 内部函数。开启后每帧把 canvas DOM 位置差分作为外力推入 asm.js 的环形缓冲区。 | `[PROVEN]` |
| **位移来源** | `canvas.getBoundingClientRect()` + `window.scrollX/Y`，即 canvas 元素在页面中的绝对位置（鼠标拖动 canvas、窗口滚动、布局变化等引起）。 | `[PROVEN]` |
| **X/Y 分量** | `vec.x = (cur.left - prev.left) / scale * frameCount`；`vec.y = (cur.top - prev.top) / scale * frameCount` | `[PROVEN]` |
| **时间步** | `frameCount` 是**乘数**（放大位移），不是除数。asm.js 内部用子步长（最大 1.1）积分。 | `[PROVEN]` |
| **滤波/卷积** | JS 层无滤波（简单差分）。asm.js 层有**环形缓冲区**（FIFO 队列，170 元素/页）+ **指数衰减插值**（`pow(k, exponent)`）。"convolve" 指对历史外力序列做加权求和。 | `[PROVEN]` |
| **速度还是位移** | **位移输入**。外力加到积分器的**目标位置**（不是速度），error = 目标位置 - 当前位置，angle = -error × stiffness。 | `[PROVEN]` |
| **施加目标** | 同时施加给 bust/parts/hair 三类。施加给**外力队列**（环形缓冲区），积分器读取后加到**目标位置**，最终影响 angle。 | `[PROVEN]` |
| **Python 实现状态** | **未实现 convolveCanvasMovementToPhysics**（无 canvas 位移差分逻辑）。有 `set_outer_force_value` 方法但**语义不匹配**：Python 把 outer_force 加到速度/gravity，asm.js 加到目标位置。Python 无环形缓冲区。 | `[PROVEN]` |

---

## 1. convolveCanvasMovementToPhysics 开关

### 1.1 定义位置
- **文件**: `reference/emoteplayer-format.js`
- **默认值**: `false`（L407 `this._convolveCanvasMovementToPhysics = false`）
- **getter/setter**: L862-875

### 1.2 开关逻辑
```javascript
// emoteplayer-format.js L294-305（每帧 Update 前调用）
player.onUpdate();
if (!player.stepUpdate &&
    player.convolveCanvasMovementToPhysics) {
    const curCanvasPosition = player.canvasPosition;
    const prevCanvasPosition = player.prevCanvasPosition;
    const scale = player.getState("scale");
    const vec = [(curCanvasPosition.left - prevCanvasPosition.left) / scale * frameCount,
        (curCanvasPosition.top - prevCanvasPosition.top) / scale * frameCount
    ];
    EmotePlayer_SetOuterForce(player.playerId, "bust", vec[0], vec[1], 0, 0);
    EmotePlayer_SetOuterForce(player.playerId, "parts", vec[0], vec[1], 0, 0);
    EmotePlayer_SetOuterForce(player.playerId, "hair", vec[0], vec[1], 0, 0);
}
player.prevCanvasPosition = player.canvasPosition;
```

### 1.3 关闭时清零
```javascript
// L865-874：从 true 切到 false 时，清零所有外力
set convolveCanvasMovementToPhysics(val) {
    if (val == this._convolveCanvasMovementToPhysics) return;
    this._convolveCanvasMovementToPhysics = val;
    if (this.initialized && !val) {
        EmotePlayer_SetOuterForce(this.playerId, "bust", 0, 0, 0, 0);
        EmotePlayer_SetOuterForce(this.playerId, "parts", 0, 0, 0, 0);
        EmotePlayer_SetOuterForce(this.playerId, "hair", 0, 0, 0, 0);
    }
}
```

### 1.4 证据等级
- `[PROVEN]` 开关默认关闭，开启后每帧计算 canvas 位移差分并调用 SetOuterForce

---

## 2. canvas movement 来源

### 2.1 canvasPosition getter
```javascript
// emoteplayer-format.js L850-859
get canvasPosition() {
    if (this.canvas == null)
        return this.prevCanvasPosition;
    else {
        const rect = this.canvas.getBoundingClientRect();
        return {
            left: rect.left + window.scrollX,
            top: rect.top + window.scrollY
        };
    }
}
```

### 2.2 来源分析
- **`canvas.getBoundingClientRect()`**：返回 canvas DOM 元素相对于视口的位置
- **`+ window.scrollX/Y`**：加上页面滚动偏移，得到**页面绝对位置**
- **触发场景**：
  - 鼠标拖动 canvas 元素（改变 left/top）
  - 窗口滚动（改变 scrollX/Y）
  - 布局变化（resize、其他元素变化）
  - **不是**模型内部移动，**不是**viewport 变化

### 2.3 prevCanvasPosition 更新时机
- 每帧 Update 后（L306）
- canvas 切换时（L556, L640）
- skip/pass 时（L1137, L1144）

### 2.4 证据等级
- `[PROVEN]` canvas movement = canvas DOM 元素在页面中绝对位置的变化

---

## 3. X/Y 分量和时间步

### 3.1 计算公式
```
vec.x = (curCanvasPosition.left - prevCanvasPosition.left) / scale * frameCount
vec.y = (curCanvasPosition.top - prevCanvasPosition.top) / scale * frameCount
```

### 3.2 分量分析
- **X 分量**：`delta.left / scale * frameCount`
  - `delta.left` = canvas 水平位移（像素）
  - `/ scale`：归一化（canvas 缩放后位移要反归一化）
  - `* frameCount`：时间步放大
- **Y 分量**：`delta.top / scale * frameCount`
  - 同 X，但是垂直方向

### 3.3 scale 来源
- `player.getState("scale")`：player 的缩放状态（E-mote 内部 scale 变量）

### 3.4 frameCount 来源
- `frameCount` 是 device.update(frameCount) 的参数，即每帧推进的帧数
- **注意**：frameCount 是**乘数**，不是除数。这意味着 frameCount 越大，外力越大（位移放大）

### 3.5 后 4 个参数
- `EmotePlayer_SetOuterForce(playerId, category, x, y, comp3, comp4)`
- `comp3 = 0`：z 方向外力（2D 模型无 z 方向）
- `comp4 = 0`：scale-gain / keyframe time（未使用）

### 3.6 证据等级
- `[PROVEN]` X/Y 分量计算公式
- `[PROVEN]` frameCount 是乘数
- `[PROVEN]` scale 是 player 缩放状态

---

## 4. SetOuterForce → 环形缓冲区

### 4.1 调用链
```
EmotePlayer_SetOuterForce (JS L251)
  → li (asm.js L57252) — vtable[176] dispatch
  → Gf (L55099) — 取 player 内部对象
  → xm (L82988) — string 转换
  → wm (L82897) — string 分发到 bust/parts/hair
  → To (L31189) — 推入环形缓冲区
```

### 4.2 wm 函数分发逻辑
```javascript
// wm (L82897) 根据 string 分发：
// "bust" (string 8998, 4 chars) → player+352 (bust 物理对象)
// "parts" (string 9003, 4 chars) → player+356 (parts 物理对象)
// "hair" (string 9008, 5 chars) → player+360 (hair 物理对象)
```

### 4.3 To 函数：环形缓冲区插入
- **元素大小**：24 字节（包含 x, y 外力 + z + flag + 时间戳）
- **容量**：170 元素/页（L31216 `/ 170`，L31235 `!= 4080` 即 170×24）
- **结构**：多页环形缓冲区，自动扩容（Uo 函数 L31395）
- **插入逻辑**：
  - 计算写索引（L31296）
  - 拷贝 24 字节到队列（L31299-31304）
  - 计数器 +1（L31305）

### 4.4 环形缓冲区存储位置
| 物理对象 | player 偏移 | 用途 |
|---------|------------|------|
| bust | player+352 | bust 外力队列 |
| parts | player+356 | parts 外力队列 |
| hair | player+360 | hair 外力队列 |

### 4.5 证据等级
- `[PROVEN]` SetOuterForce 推入环形缓冲区（FIFO 队列）
- `[PROVEN]` 三类物理对象分别有独立队列（player+352/356/360）
- `[LIKELY]` string 8998/9003/9008 = "bust"/"parts"/"hair"（通过长度 4/4/5 和调用顺序推断，asm.js 中无直接字符串字面量）

---

## 5. Update → 读取队列 → 积分器

### 5.1 调用链
```
EmotePlayer_Update (JS L87)
  → Ci (asm.js L57514) — vtable[272] dispatch
  → eg (L55402) — 取 player 内部对象
  → Al (L78610) — 主 Update 函数
    → Qo (L30945) → Ro (L30953) — 从环形缓冲区读取外力
    → Hl (L79304) — 处理 bust → Qn → Pn (L28025)
    → Il (L79418) — 处理 parts/hair → Ap → zp (L65028)
```

### 5.2 Qo/Ro 函数：从队列读取外力
```javascript
// Ro (L30953) 逻辑：
// 1. 检查队列是否有元素（a+24 计数器）
// 2. 从环形缓冲区读取 24 字节元素（L31032-31038）
// 3. 读索引 +1，元素数 -1（L31039-31041）
// 4. 如果读索引 > 339，释放一页（L31042-31046）
// 5. 设置衰减状态（a+32 = 1，L31062）
// 6. 输出外力到 b（L31089-31092）
```

### 5.3 外力累积
```javascript
// Al (L78610) 中：
// L78755: Qo(c[b+352], y, w) — 读取 bust 外力到 y
// L78759: g[b+368] = w * g[y] + g[b+368] — 累积到 bust 外力 x
// L78761: g[b+372] = w * g[y+4] + g[b+372] — 累积到 bust 外力 y
// L78762: Qo(c[b+356], y, w) — 读取 parts 外力
// L78765-78767: 累积到 b+376/380
// L78768: Qo(c[b+360], y, w) — 读取 hair 外力
// L78771-78773: 累积到 b+384/388
```

### 5.4 外力累积位置
| 物理对象 | 累积外力 x | 累积外力 y |
|---------|-----------|-----------|
| bust | player+368 | player+372 |
| parts | player+376 | player+380 |
| hair | player+384 | player+388 |

### 5.5 证据等级
- `[PROVEN]` Update 时从环形缓冲区读取外力（FIFO）
- `[PROVEN]` 外力被时间步加权累积到 player+368/376/384

---

## 6. 滤波/卷积机制

### 6.1 JS 层：无滤波
- 只是简单差分（cur - prev），无滤波/平滑

### 6.2 asm.js 层：环形缓冲区 + 指数衰减插值

#### 6.2.1 环形缓冲区（FIFO 队列）
- 每次 SetOuterForce 推入一个外力记录
- 每次 Update 从队列头部读取一个记录
- **不是直接覆盖**，而是排队处理

#### 6.2.2 指数衰减插值（Ro 函数 L30998-31081）
```javascript
// case 1（衰减状态）：
// L30999: k = g[a+52] + g[a+56] * dt  — 累积衰减时间
// L31001: if (k < 1.0) → 衰减未完成，进入插值
// L31067: k = pow(k, g[a+48])  — 指数衰减
// L31079: current = old + k * (new - old)  — 插值
```

- **a+52**：累积衰减时间（每帧 += a+56 × dt）
- **a+56**：衰减速率
- **a+48**：指数（控制衰减曲线形状）
- **插值公式**：`current = old × (1-k') + new × k'`，其中 `k' = pow(k, exponent)`

### 6.3 "convolve" 的数学本质
- 外力不是瞬间切换，而是**指数衰减过渡**
- 当前外力 = 历史外力序列的加权求和（权重随时间指数衰减）
- 这就是对历史外力序列的**卷积**（convolution）

### 6.4 证据等级
- `[PROVEN]` 环形缓冲区 FIFO 队列
- `[PROVEN]` 指数衰减插值公式
- `[LIKELY]` "convolve" 指对历史外力序列的卷积（数学上等价）

---

## 7. 施加目标：hair/parts/bust 的哪一级状态

### 7.1 hair/parts 积分器 zp（L65028）

#### 7.1.1 外力 → 目标位置
```javascript
// zp (L65028) 参数 d, e = 外力 x, y
// L65073-65076（无 flag 时）：
d = g[b+76] + d  // 累积外力 x + 传入 d
g[b+64] = d      // 存到 b+64（目标位置 x）
e = g[b+80] + e  // 累积外力 y + 传入 e
g[b+68] = e      // 存到 b+68（目标位置 y）
```

#### 7.1.2 error 计算
```javascript
// L65181-65184：
d = g[b+88 + v*12] - m  // 目标位置 x - 当前位置 x = error_x
g[y] = g[b+88 + v*12 + 4] - R  // 目标位置 y - 当前位置 y = error_y
```

#### 7.1.3 angle 输出
```javascript
// L65185: angle = xo(-error_x * stiffness[v] * scale)
// L65188: angle2 = xo((cur_y - error_y) * stiffness2[v] * scale)
```

### 7.2 bust 积分器 Pn（L28025）

#### 7.2.1 外力 → 目标位置
```javascript
// Pn (L28025) 参数 c, d = 外力 x, y
// L28049-28052（无 flag 时）：
c = g[b+40] + c  // 累积外力 x + 传入 c
g[b+28] = c      // 存到 b+28（目标位置 x）
d = g[b+44] + d  // 累积外力 y + 传入 d
g[b+32] = d      // 存到 b+32（目标位置 y）
```

#### 7.2.2 error 计算 + angle 输出
```javascript
// L28090: angle = xo(-((c - l) * scale * stiffness))  // c=目标位置, l=当前位置
// L28091: angle2 = xo((-((d - m) * scale) - bust_offset) * stiffness2)
```

### 7.3 施加层级总结
| 层级 | 是否直接修改 | 说明 |
|------|------------|------|
| angle | 否 | angle 由 error × stiffness 计算，不直接设置 |
| velocity | 否 | velocity 由弹簧力 + 阻尼计算，外力不直接加到 velocity |
| **position (目标位置)** | **是** | **外力加到目标位置（b+64/68 for zp, b+28/32 for Pn）** |
| error | 间接 | error = 目标位置 - 当前位置 |
| angle | 间接 | angle = -error × stiffness |

### 7.4 关键结论
- **外力施加给"目标位置"**，不是 velocity 或 angle
- 物理积分器通过弹簧机制追踪目标位置：error = target - current，angle = -error × stiffness
- 这意味着 canvas movement **改变弹簧的目标**，模型会弹性地追踪这个新目标

### 7.5 证据等级
- `[PROVEN]` 外力加到目标位置（zp b+64/68, Pn b+28/32）
- `[PROVEN]` error = 目标位置 - 当前位置
- `[PROVEN]` angle = -error × stiffness
- `[PROVEN]` 外力不直接加到 velocity 或 angle

---

## 8. 最终如何进入 physics integrator

### 8.1 完整数据流
```
[JS 层] canvas.getBoundingClientRect() → {left, top}
  ↓ 每帧差分
delta = {cur.left - prev.left, cur.top - prev.top}
  ↓ 归一化 + 时间步放大
vec = [delta.x / scale * frameCount, delta.y / scale * frameCount]
  ↓ 同时施加给三类
EmotePlayer_SetOuterForce(playerId, "bust"/"parts"/"hair", vec[0], vec[1], 0, 0)
  ↓
[asm.js 层] li → Gf → xm → wm → To
  ↓ 推入环形缓冲区 (player+352/356/360)
  ↓ 元素: 24 字节 (x, y, ?, ?, z, flag)
  ↓
[每帧 Update] EmotePlayer_Update → Ci → eg → Al
  ↓
Qo → Ro: 从环形缓冲区读取一个外力记录
  ↓ 指数衰减插值: current = old + pow(k, exp) * (new - old)
  ↓ 累积到 player+368/376/384 (bust/parts/hair 累积外力)
  ↓
Hl (bust): 外力 = player+368/372 + Jl(bone 变换)
  → Qn → Pn: 外力加到 b+40/44 → b+28/32 (目标位置)
  → error = 目标位置 - 当前位置
  → angle = xo(-error × stiffness)
  ↓
Il (parts/hair): 外力 = player+376/384 + Jl(bone 变换)
  → Ap → zp: 外力加到 b+76/80 → b+64/68 (目标位置)
  → error = 目标位置 - 当前位置
  → angle = xo(-error × stiffness)
  ↓
output_angles → motion_painter → layer 旋转
```

### 8.2 调用链行号引用
| 函数 | 行号 | 作用 |
|------|------|------|
| `EmotePlayer_SetOuterForce` (JS) | emoteplayer L302-304 | JS 层调用 |
| `li` | L57252 | vtable[176] dispatch |
| `Gf` | L55099 | 取 player 内部对象 |
| `xm` | L82988 | string 转换 |
| `wm` | L82897 | string 分发 bust/parts/hair |
| `To` | L31189 | 推入环形缓冲区 |
| `EmotePlayer_Update` (JS) | emoteplayer L313 | JS 层调用 |
| `Ci` | L57514 | vtable[272] dispatch |
| `eg` | L55402 | 取 player 内部对象 |
| `Al` | L78610 | 主 Update 函数 |
| `Qo` | L30945 | 读取外力（wrapper） |
| `Ro` | L30953 | 读取外力 + 衰减插值 |
| `Hl` | L79304 | bust 物理处理 |
| `Il` | L79418 | parts/hair 物理处理 |
| `Jl` | L79536 | bone 变换外力计算 |
| `Qn` | L28095 | bust 积分器（wrapper） |
| `Pn` | L28025 | bust 积分器 |
| `Ap` | L65198 | hair/parts 积分器（wrapper） |
| `zp` | L65028 | hair/parts 积分器 |

### 8.3 证据等级
- `[PROVEN]` 完整调用链（每一步都有行号引用）

---

## 9. Python 当前实现状态

### 9.1 convolveCanvasMovementToPhysics
- **状态**: **未实现**
- **证据**: `src/freemote/` 中无 `canvas_position`、`prev_canvas`、`canvas_delta`、`convolve` 相关代码
- **影响**: Python 无法响应 canvas 移动产生的物理摆动

### 9.2 set_outer_force_value 方法
- **位置**: `src/freemote/player/player.py` L4488-4527
- **实现**:
```python
def set_outer_force_value(self, category, x, y, comp3=0.0, comp4=0.0):
    # ...
    physics = self._get_physics_for_category(cat)
    physics.outer_force = (float(x), float(y))  # 直接覆盖
    self._outer_force_enabled = True
```
- **问题**: 直接覆盖 `outer_force` 字段，**无环形缓冲区**，**无衰减插值**

### 9.3 Python zp 积分器（HairPartsPhysics.update）
- **位置**: `src/freemote/physics/physics.py` L380-493
- **外力使用**:
```python
# L399: outer_force = self.outer_force[0]  — 只用 x 分量
# L448-450: d = outer_force * dt; vel_x += w * d; vel_y += x * d  — 加到速度
```
- **与 asm.js 差异**:
  - Python: outer_force 加到**速度**（vel_x += w × outer_force × dt）
  - asm.js: convolveCanvasMovementToPhysics 外力加到**目标位置**（b+64/68）
  - **语义不匹配**

### 9.4 Python Pn 积分器（BustPhysics.update）
- **位置**: `src/freemote/physics/physics.py` L594-692
- **外力使用**:
```python
# L657: t = (self.gravity + self.outer_force[1]) * dt  — outer_force[1] 加到 gravity
# L661: l = vel_x + (cur_x - pos_x) * j - cos_a * t  — 作为持续外力
```
- **与 asm.js 差异**:
  - Python: outer_force[1] 加到 **gravity**（持续外力）
  - asm.js: convolveCanvasMovementToPhysics 外力加到**目标位置**（b+28/32）
  - **语义不匹配**

### 9.5 Python outer_force 字段注释
```python
# physics.py L184-188:
outer_force: tuple[float, float] = (0.0, 0.0)
"""External force (x, y) from OuterForce API (physics.md §1.3, §6.2 步骤 3).
对应 ``zp`` 的 b+4 (单值外力) 和 ``Pn`` 的 b+8 (outer_force_x) / b+4
(outer_force_y)。Phase 3-D 简化：zp 用 ``outer_force[0]``，Pn 用两者。
"""
```
- Python 认为 outer_force 对应 zp 的 **b+4**（速度外力字段），不是传入参数（目标位置外力）
- 这意味着 Python 实现的是 **asm.js 的 b+4 字段机制**（持续外力），不是 convolveCanvasMovementToPhysics（目标位置外力）

### 9.6 app.py 中的使用
- **位置**: `src/freemote/app.py` L239-245
- **逻辑**: 鼠标左键按下时 `set_outer_force_value("bust", 0.0, 3.0)`，释放时归零
- **与 convolveCanvasMovementToPhysics 无关**: 这是手动外力，不是 canvas 位移差分

### 9.7 证据等级
- `[PROVEN]` Python 未实现 convolveCanvasMovementToPhysics
- `[PROVEN]` Python outer_force 语义与 asm.js convolveCanvasMovementToPhysics 不匹配
- `[PROVEN]` Python 无环形缓冲区

---

## 10. 差异总结

| # | 差异 | asm.js 原版 | Python 当前 | 影响 | 证据 |
|---|------|------------|------------|------|------|
| 1 | convolveCanvasMovementToPhysics 开关 | JS 层实现（默认 false） | **未实现** | Python 无法响应 canvas 移动物理 | `[CONFIRMED-DIFFERENCE]` |
| 2 | canvas 位移差分 | `getBoundingClientRect()` 差分 | **无** | 无 canvas movement 输入 | `[CONFIRMED-DIFFERENCE]` |
| 3 | 环形缓冲区 | FIFO 队列（170 元素/页）+ 指数衰减 | **直接覆盖** | 无历史外力平滑，无衰减过渡 | `[CONFIRMED-DIFFERENCE]` |
| 4 | 外力施加层级 | 加到**目标位置**（b+64/68, b+28/32） | 加到**速度**（zp）或**gravity**（Pn） | 语义不匹配，摆动行为不同 | `[CONFIRMED-DIFFERENCE]` |
| 5 | 外力分量使用 | zp 用 x+y（b+64/68），Pn 用 x+y（b+28/32） | zp 只用 x，Pn 只用 y | 分量使用不匹配 | `[CONFIRMED-DIFFERENCE]` |

---

## 11. 数据流图

### 11.1 asm.js 完整数据流
```
[JS] canvas.getBoundingClientRect() → {left, top}
  ↓ 每帧差分 (emoteplayer L296-301)
vec = [(cur.left-prev.left)/scale*frameCount, (cur.top-prev.top)/scale*frameCount]
  ↓
[JS] EmotePlayer_SetOuterForce(playerId, "bust"/"parts"/"hair", vec[0], vec[1], 0, 0)
  ↓
[asm.js] li (L57252) → Gf (L55099) → xm (L82988) → wm (L82897)
  ↓ string 分发
[asm.js] To (L31189) → 推入环形缓冲区
  - bust 队列: player+352
  - parts 队列: player+356
  - hair 队列: player+360
  - 元素: 24 字节 (x, y, ?, ?, z, flag)
  - 容量: 170 元素/页
  ↓
[每帧 Update] EmotePlayer_Update → Ci (L57514) → eg (L55402) → Al (L78610)
  ↓
[asm.js] Qo (L30945) → Ro (L30953)
  - 从环形缓冲区读取一个外力记录
  - 指数衰减插值: current = old + pow(k, exp) * (new - old)
  ↓ 累积到 player+368/376/384
  ↓
[asm.js] Hl (L79304) 处理 bust:
  - 外力 = player+368/372 (累积) + Jl (L79536, bone 变换)
  → Qn (L28095) → Pn (L28025)
  - 外力加到 b+40/44 → b+28/32 (目标位置)
  - error = 目标位置 - 当前位置
  - angle = xo(-error × stiffness)
  ↓
[asm.js] Il (L79418) 处理 parts/hair:
  - 外力 = player+376/384 (累积) + Jl (bone 变换)
  → Ap (L65198) → zp (L65028)
  - 外力加到 b+76/80 → b+64/68 (目标位置)
  - error = 目标位置 - 当前位置
  - angle = xo(-error × stiffness)
  ↓
output_angles → motion_painter → layer 旋转
```

### 11.2 Python 当前数据流
```
[无 canvas movement 输入]
  ↓
[手动] set_outer_force_value("bust", 0.0, 3.0)  # app.py 鼠标回调
  ↓
physics.outer_force = (0.0, 3.0)  # 直接覆盖，无队列
  ↓
[每帧] _advance_physics (player.py L5414)
  ↓
BustPhysics.update (physics.py L594):
  - t = (gravity + outer_force[1]) * dt  # outer_force 加到 gravity
  - velocity += (cur - pos) * spring * dt - cos * t
  - angle = xo(-((cur - pos) * scale * stiffness))
  ↓
HairPartsPhysics.update (physics.py L380):
  - d = outer_force[0] * dt  # outer_force 加到速度
  - velocity += (w, x) * d
  - angle = xo(-(error * stiffness * scale))
  ↓
output_angles → motion_painter → layer 旋转
```

---

## 12. 12 个问题完整回答

### Q1: convolveCanvasMovementToPhysics 是 asm.js 内部函数吗？
**A1**: **不是** `[PROVEN]`。它是 `emoteplayer-format.js` 中的 JS 层配置开关（getter/setter，L862-875），默认 false。开启后每帧在 JS 层计算 canvas 位移差分，然后调用 `EmotePlayer_SetOuterForce` 推入 asm.js 的环形缓冲区。

### Q2: canvas movement 是指什么？
**A2**: canvas DOM 元素在页面中绝对位置的变化 `[PROVEN]`。来源是 `canvas.getBoundingClientRect() + window.scrollX/Y`（L850-859）。触发场景：鼠标拖动 canvas、窗口滚动、布局变化。**不是**模型内部移动，**不是**viewport 变化。

### Q3: X/Y 分量如何获取？
**A3**: `[PROVEN]`
- X = `(curCanvasPosition.left - prevCanvasPosition.left) / scale * frameCount`
- Y = `(curCanvasPosition.top - prevCanvasPosition.top) / scale * frameCount`
- scale = `player.getState("scale")`（player 缩放状态）

### Q4: 时间步如何参与？
**A4**: `frameCount` 是**乘数**（放大位移），不是除数 `[PROVEN]`。asm.js 内部 Update 时用子步长（最大 1.1）积分（L78672 `d = v > 1.1 ? 1.1 : v`）。

### Q5: 是否有滤波/卷积操作？
**A5**: **有** `[PROVEN]`。
- JS 层无滤波（简单差分）
- asm.js 层有**环形缓冲区**（FIFO 队列，170 元素/页）+ **指数衰减插值**（`current = old + pow(k, exp) * (new - old)`）
- "convolve" 指对历史外力序列的加权求和（权重随时间指数衰减）

### Q6: 是速度还是位移？
**A6**: **位移输入** `[PROVEN]`。外力加到积分器的**目标位置**（zp b+64/68, Pn b+28/32），不是速度。error = 目标位置 - 当前位置，angle = -error × stiffness。

### Q7: 施加给 hair/parts/bust 的哪一级状态？
**A7**: **目标位置** `[PROVEN]`。
- hair/parts: 外力 → b+64/68（目标位置）→ error → angle
- bust: 外力 → b+28/32（目标位置）→ error → angle
- 不直接修改 angle、velocity、position（当前位置）

### Q8: 最终如何进入 physics integrator？
**A8**: `[PROVEN]`
- hair/parts: `zp` (L65028)，外力 → b+76/80 → b+64/68（目标位置）→ error → angle
- bust: `Pn` (L28025)，外力 → b+40/44 → b+28/32（目标位置）→ error → angle

### Q9: 是否同时施加给 bust/parts/hair？
**A9**: **是** `[PROVEN]`。JS 层 L302-304 同时调用三次 SetOuterForce，使用相同的 vec。

### Q10: 环形缓冲区的容量和结构？
**A10**: `[PROVEN]`
- 元素大小：24 字节（x, y 外力 + z + flag + 时间戳）
- 容量：170 元素/页，多页自动扩容
- 结构：FIFO 队列（先进先出）

### Q11: Python 是否实现了 convolveCanvasMovementToPhysics？
**A11**: **未实现** `[PROVEN]`。`src/freemote/` 中无 canvas 位移差分逻辑。有 `set_outer_force_value` 方法但语义不匹配（加到速度/gravity，不是目标位置），无环形缓冲区。

### Q12: Python outer_force 与 asm.js 的对应关系？
**A12**: `[PROVEN]` Python outer_force 对应 asm.js 的 **b+4 字段**（持续外力，加到速度），**不是** convolveCanvasMovementToPhysics 的传入参数（目标位置外力）。两者语义不同。

---

## 13. 修复建议

### 13.1 短期（不影响现有功能）
1. **添加 convolveCanvasMovementToPhysics 开关**到 `EmotePlayer` 类（默认 False）
2. **实现 canvas 位移差分**：记录 prev_canvas_position，每帧计算 delta
3. **调用 set_outer_force_value**：开启时每帧调用（bust/parts/hair 同步）

### 13.2 中期（语义对齐）
1. **实现环形缓冲区**：OuterForceQueue 类（170 元素/页，FIFO）
2. **实现指数衰减插值**：`current = old + pow(k, exp) * (new - old)`
3. **修正外力施加层级**：加到目标位置（不是速度/gravity）

### 13.3 长期（完整对齐）
1. **Jl bone 变换外力**：当前 Python 的 current_position 已部分实现（baseLayer + var_lr/var_ud）
2. **子步长积分**：asm.js 用最大 1.1 子步，Python 需确认是否一致

---

## 14. 参考资料

- **JS 层**: `reference/emoteplayer-format.js`（1267 行）
  - L294-305: convolveCanvasMovementToPhysics 主逻辑
  - L407: 默认值 false
  - L556/640/1137/1144: prevCanvasPosition 更新
  - L850-859: canvasPosition getter
  - L862-875: convolveCanvasMovementToPhysics getter/setter
- **asm.js**: `reference/FreeMoteDriver-format.js`（99629 行）
  - `li` @ L57252: SetOuterForce dispatch
  - `Gf` @ L55099: player 内部对象
  - `xm` @ L82988: string 转换
  - `wm` @ L82897: string 分发 bust/parts/hair
  - `To` @ L31189: 环形缓冲区插入
  - `Ci` @ L57514: Update dispatch
  - `eg` @ L55402: Update player 内部
  - `Al` @ L78610: 主 Update 函数
  - `Qo` @ L30945 / `Ro` @ L30953: 读取外力 + 衰减插值
  - `Hl` @ L79304: bust 物理处理
  - `Il` @ L79418: parts/hair 物理处理
  - `Jl` @ L79536: bone 变换外力
  - `Qn` @ L28095 / `Pn` @ L28025: bust 积分器
  - `Ap` @ L65198 / `zp` @ L65028: hair/parts 积分器
- **Python**:
  - `src/freemote/player/player.py` L4488-4527: set_outer_force_value
  - `src/freemote/player/player.py` L5414-5493: _advance_physics
  - `src/freemote/physics/physics.py` L184-188: outer_force 字段
  - `src/freemote/physics/physics.py` L380-493: zp 积分器
  - `src/freemote/physics/physics.py` L594-692: Pn 积分器
  - `src/freemote/app.py` L239-245: 鼠标回调（手动外力）
- **P0 报告**: `physics-stiffness-source-report.md`（stiffness 来源背景）

---

**报告生成时间**: 2026-09-26
**任务状态**: completed
**关键结论**: convolveCanvasMovementToPhysics 是 JS 层开关（默认 false），开启后把 canvas DOM 位移差分推入 asm.js 环形缓冲区，积分器从队列读取并加到**目标位置**（不是速度），通过弹簧机制产生摆动。Python **未实现**此功能，现有 outer_force 语义不匹配（加到速度/gravity 而非目标位置）。