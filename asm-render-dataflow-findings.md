# asm.js 渲染数据流完整分析

> 证据来源：`reference/FreeMoteDriver-format.js`（99629 行 Emscripten 产物）
> 证据等级：`[PROVEN]`（明确代码证据）、`[LIKELY]`（强推测）、`[UNVERIFIED]`（弱推测）、`[UNKNOWN]`

---

## 1. 完整调用链（从渲染入口到 draw call）

### 1.1 Update 链（每帧状态更新）`[PROVEN]`

```
_EmotePlayer_Update (Ci, L57514)
  → wd[vtable+272 & 7](a, b)           // L57522 虚函数分发
    → eg (L55402, wd[5])               // wrapper: 取 a+16 的子对象
      → Al(c[a+16>>2], b)              // L55405
        → Au(children[f], w)           // L78746 循环每个子对象
          → cu(b, d)                   // L87488 某个 update
          → du(b)                      // L87489 主 update（含 hu）
            → gu(b)  if flags&512      // L52560 stereovision parallax
            → hu(b)                    // L52561 mesh deform (xr/zr)
            → iu(b)                    // L52562 deform flag 设置
            → ju(b)  if flags&32       // L52563 bust scale
            → ku(b)                    // L52564 bounding box 计算
            → lu(b)  if flags&2        // L52567 particle 区域
            → mu(b)  if flags&8        // L52571 update+deform
            → nu(b)  if flags&64       // L52575 碰撞/关联
            → ou(b)  if flags&16       // L52578 update+deform
            → pu(b)  if flags&1024     // L52582 effect
          → xd[vtable+16 & 127](b)    // L87490 后处理回调
```

### 1.2 Draw 链（每帧渲染）`[PROVEN]`

```
_EmotePlayer_Draw (Dh, L56408)
  → Fh(a)                              // L56410 = _EmotePlayer_DrawToTexture
    → OpenGL 状态保存 (L56570-56607)   // viewport/blend/depth/stencil
    → xd[vtable+280 & 127](b)          // L56609 虚函数分发
      → Ji (L57580, xd[16])            // 主 draw 函数
        → Hv(b, ca)                    // L58857 设置 uniform + draw
          → Tw(a, e)                    // L6809
            → Uw(a)                     // L9605
              → Ox(...)                // L9651
                → Nx(5, ...)           // L12977
                  → glBindBuffer       // L12650-12653
                  → glBufferData       // 上传 vertex/index buffer
                  → glVertexAttribPointer // L12672-12674
                  → glDrawElements     // L12924/L12956 (Cb)
```

### 1.3 初始化链（加载 PSB 数据）`[PROVEN]`

```
Me (L54479, 构造函数)
  → cj(s, r, t, 1)                     // L54526/L54571
    → ej(b)                            // L60452/L60458
      → fj(b, n)                       // L60678 从 PSB 加载配置
        → Au(c[C>>2], 0.0)             // L60744 首次 update
```

---

## 2. xr/zr 分支条件确认

### 2.1 变量来源 `[PROVEN]`

在 hu 函数（L53301）的主循环（L53415-53946）中，对每个 layer 对象 `E + (I * 748)`：

| 变量 | 代码 | offset | 含义 |
|------|------|--------|------|
| `D` | `c[E + (I*748) + 704 >> 2]` | 704 | deform 控制对象指针（0 = 无 deform）`[PROVEN]` |
| `B` | `D ? c[D + 8 >> 2] : 0` | D+8 | deform 子对象（Bezier 参数/grid 分辨率）`[PROVEN]` |
| `A` | `E + (I*748) + 616` | 616 | mesh_bp vector（begin/end/cap = 616/620/624）`[PROVEN]` |
| `H` | `c[E + (I*748) + 712 >> 2]` | 712 | parent 链表指针`[PROVEN]` |
| `y` | `E + (I*748) + 700` | 700 | deform 模式标志（1 = Bezier 模式）`[PROVEN]` |
| `z` | `E + (I*748) + 24` | 24 | layer type`[PROVEN]` |
| `q` | `E + (I*748) + 132` | 132 | deform enable flag`[PROVEN]` |
| `o` | `E + (I*748) + 716` | 716 | 某个状态指针`[PROVEN]` |

### 2.2 分支条件 `[PROVEN]`

关键分支在 **L53679**：

```javascript
if ((B | 0) != 0 ? (c[A >> 2] | 0) != (c[A + 4 >> 2] | 0) : 0) {
    // xr 分支：双三次 Bezier
    xr(B, z, B + 52 | 0, W)
} else {
    // B == 0 或 A vector 为空
    if (!d) {  // d = c[H >> 2], H = offset 712
        // 简单 4 角点路径（不调用 xr/zr）
        // 直接写入 4 个角点到输出 vector
    } else {
        // zr 分支：双线性插值
        zr(W, wa, xa, z)
    }
}
```

**三个分支的精确条件**：

| 分支 | 条件 | 语义 |
|------|------|------|
| **xr (Bezier)** | `B != 0` AND `c[A] != c[A+4]`（mesh_bp vector 非空） | 有 deform 控制对象且有 Bezier 控制点`[PROVEN]` |
| **zr (双线性)** | (`B == 0` OR mesh_bp vector 为空) AND `H != 0` | 无 Bezier deform，但有 parent 链表`[PROVEN]` |
| **简单 4 角点** | (`B == 0` OR mesh_bp vector 为空) AND `H == 0` | 无 deform，无链表，直接 4 角点`[PROVEN]` |

### 2.3 前置条件 `[PROVEN]`

分支之前还有两层条件检查：

1. **L53649**: `1 << c[z >> 2] & 5121` — type ∈ {0, 10, 12} 且 `offset 133 == 0`
   - 5121 = 2^0 + 2^10 + 2^12

2. **L53525**: `1 << c[z >> 2] & 7169` — type ∈ {0, 10, 11, 12} 且 `offset 132 != 0`
   - 7169 = 2^0 + 2^10 + 2^11 + 2^12

### 2.4 offset 700 == 1 的含义 `[PROVEN]`

当 `c[y >> 2] == 1`（offset 700 == 1）时（L53581）：
- 从 layer 的 width/height（offset 160/164）和 3 个方向向量（offset 92/96/100/104）构造 2×2 仿射矩阵
- 对 16 个 Bezier 控制点（从 A vector 读取）应用矩阵变换
- 将变换后的控制点存入 B+40 的 vector（128 bytes = 16 × 8）
- 计算 2×2 矩阵的逆，存入 B+68~B+80
- 平移向量存入 B+84~B+88

**语义**：offset 700 == 1 表示此 layer 使用 Bezier deform 模式，mesh_bp 数据是 4×4 Bezier 控制点网格 `[PROVEN]`

---

## 3. xr 函数详细分析（双三次 Bezier）

### 3.1 函数签名 `[PROVEN]`

```javascript
function xr(a, b, d, e)  // L14719
// a = B: deform 控制对象（含 grid 分辨率、控制点数组指针）
// b = z: 输出 vector 指针（t+12，t 是 44 字节 deform 结果对象）
// d = B+52: 2×2 变换矩阵 + 逆矩阵 + 平移（36 bytes）
// e = W: 平移向量 (j, k)（8 bytes）
```

### 3.2 数据结构 `[PROVEN]`

deform 控制对象 `a`（92 字节，由 Qr 初始化）：

| offset | 类型 | 含义 |
|--------|------|------|
| 0 | ptr | Bezier 基函数数组指针 `[LIKELY]` |
| 4 | int | 某个参数 `[UNKNOWN]` |
| 8 | ptr | 控制点数组指针 `[PROVEN]` |
| 16 | int | grid 宽 - 1 `[PROVEN]` |
| 20 | int | grid 高 - 1 `[PROVEN]` |
| 24 | ptr | 控制点 vector `[PROVEN]` |
| 28-32 | vector | 控制点数据（begin/end）`[PROVEN]` |
| 40-88 | float[12] | 变换矩阵 + 逆矩阵 + 平移 `[PROVEN]` |

变换矩阵 `d = B+52`：

| offset | 含义 |
|--------|------|
| d[0] (52) | 原始矩阵 m11 = h * width `[PROVEN]` |
| d[4] (56) | 原始矩阵 m12 = height * dir96 `[PROVEN]` |
| d[8] (60) | 原始矩阵 m21 = width * dir100 `[PROVEN]` |
| d[12] (64) | 原始矩阵 m22 = height * dir104 `[PROVEN]` |
| d[16] (68) | 逆矩阵 im11 `[PROVEN]` |
| d[20] (72) | 逆矩阵 im12 `[PROVEN]` |
| d[24] (76) | 逆矩阵 im21 `[PROVEN]` |
| d[28] (80) | 逆矩阵 im22 `[PROVEN]` |
| d[32] (84) | 平移 tx = -j `[PROVEN]` |
| d[36] (88) | 平移 ty = -k `[PROVEN]` |

### 3.3 执行流程 `[PROVEN]`

1. **步骤 1（L14794-14825）**：读取 16 个控制点（从 `c[c[a+24>>2]>>2]`），通过 2×2 矩阵变换后存入栈上临时空间 Z（128 bytes = 16 × 8）
   - 变换公式：`new_x = m11 * x + m12 * y + tx`，`new_y = m21 * x + m22 * y + ty`

2. **步骤 2（L14826-14827）**：读取 grid 分辨率
   - `W = c[a + 16 >> 2] + 1` = grid 宽
   - `X = c[a + 20 >> 2] + 1` = grid 高

3. **步骤 3（L14836-14871）**：调整输出 vector b 的容量为 `W × X` 个 2D 点

4. **步骤 4（L14912-14994）**：双重循环 tessellation
   - 外层循环 X 次（高度方向）
   - 内层循环 W 次（宽度方向）
   - 对每个 (u, v) 参数，计算双三次 Bezier 曲面值：
     - 从 `c[a+8>>2]` 读取 Bezier 基函数值（4 个 float per axis）
     - 从 `c[a>>2]` 读取另一组控制点数据
     - 4×4 矩阵乘法：控制点 × 基函数
     - 结果写入输出 vector b

### 3.4 输出 `[PROVEN]`

- 输出：`W × X` 个 2D 顶点（每个 8 bytes），写入 vector b
- 用途：Bezier 曲面 tessellation，生成 deform grid 顶点

---

## 4. zr 函数详细分析（双线性插值）

### 4.1 函数签名 `[PROVEN]`

```javascript
function zr(a, b, d, e)  // L15055
// a = W: 4 个角点（32 bytes = 4 × 8）
// b = wa: grid 宽（已 +1）
// d = xa: grid 高（已 +1）
// e = z: 输出 vector 指针
```

### 4.2 4 个角点布局 `[PROVEN]`

角点存储在 `a`（32 bytes）中：

| offset | 含义 |
|--------|------|
| a[0], a[4] | 左上角 (x, y) `[PROVEN]` |
| a[8], a[12] | 右上角 (x, y) `[PROVEN]` |
| a[16], a[20] | 左下角 (x, y) `[PROVEN]` |
| a[24], a[28] | 右下角 (x, y) `[PROVEN]` |

### 4.3 执行流程 `[PROVEN]`

1. **L15088-15090**：计算 grid 尺寸
   - `y = b + 1` = grid 宽 + 1
   - `z = d + 1` = grid 高 + 1
   - `m = S(z, y)` = 总顶点数

2. **L15095-15131**：调整输出 vector e 的容量为 m 个 2D 点

3. **L15134-15135**：计算插值步长
   - `v = 1.0 / b`
   - `p = 1.0 / d`

4. **L15152-15176**：双重循环双线性插值
   - 外层循环 z 次（高度 + 1）
   - 内层循环 y 次（宽度 + 1）
   - 对每个 (u, v) 参数：
     - 先在 v 方向插值上下边：`s = (1-v) * topLeft + v * bottomLeft`
     - 再在 u 方向插值：`result = (1-u) * s + u * t`
   - 标准双线性插值公式：`P(u,v) = (1-u)(1-v)*P00 + u(1-v)*P10 + (1-u)v*P01 + uv*P11`

### 4.4 输出 `[PROVEN]`

- 输出：`(b+1) × (d+1)` 个 2D 顶点，写入 vector e
- 用途：双线性 deform tessellation，从 4 个角点生成 grid

### 4.5 grid 分辨率确定 `[PROVEN]`

grid 分辨率在 hu 的 else 分支（L53812-53840）中计算：

- 当 `c[y >> 2]`（offset 700）不为 0 时：
  - `d = ~~(scale * (c[D+4>>2] >>> 0))` — 基于 D 的 offset 4 和全局 scale
  - `o = o + r` — o 是某个 base 值

- 当 `c[y >> 2]` 为 0 时：
  - 遍历 parent 链找到第一个 active 的节点
  - `d = ~~(scale * (c[(c[d+704>>2])+4>>2] >>> 0))`

- 最终：`wa = e + 1`（宽 + 1），`xa = d + 1`（高 + 1）
- `zr(W, wa, xa, z)` — wa 和 xa 是 grid 分辨率

---

## 5. mesh_bp 的真实职责

### 5.1 在 asm.js 中的存储 `[PROVEN]`

mesh_bp（32 个 float = 16 个 2D 点）在 asm.js 中存储为 layer 对象的 **offset 616** 的 vector：

| offset | 字段 | 来源 |
|--------|------|------|
| 616 | vector begin | `c[e + 180 + (q*212) + 68 >> 2]`（frame data offset 68）`[PROVEN]` |
| 620 | vector end | `c[e + 180 + (q*212) + 72 >> 2]`（frame data offset 72）`[PROVEN]` |
| 624 | vector capacity | `c[e + 180 + (q*212) + 76 >> 2]`（frame data offset 76）`[PROVEN]` |

设置位置：L49820-49822（从 frame data 复制到 layer 对象）

### 5.2 在 hu 中的读取 `[PROVEN]`

mesh_bp 在 hu 中被读取用于：

1. **L53502-53523**：当 `1 << type & 34`（type ∈ {1, 5}）时，从 parent 链累加 mesh_bp 偏移
   - 遍历 parent 链（offset 712），对每个 active parent 的 offset 704 子对象应用 2×2 变换
   - 累加结果到 offset 120/124/128（position）

2. **L53569-53580**：计算 j, k（平移向量）
   - `f = c[E + (I*748) + 616 >> 2]` — mesh_bp vector begin
   - `va = +g[offset 620] + +g[offset 624] * scale` — 某个插值值
   - j, k 是基于 mesh_bp 和方向向量的平移分量

3. **L53581-53647**：当 offset 700 == 1 时，16 个控制点从 A vector 读取，经 2×2 矩阵变换后存入 B+40

4. **L53679**：分支条件检查 `c[A >> 2] != c[A + 4 >> 2]`（mesh_bp vector 是否非空）

### 5.3 是否参与 xr/zr 计算 `[PROVEN]`

- **xr**：是的。mesh_bp 的 16 个控制点经变换后存入 B+40 的 vector，xr 从 `c[a+24>>2]` 读取这些控制点进行 Bezier 曲面求值
- **zr**：不直接参与。zr 使用 4 个角点（W），这些角点是从 mesh_bp 和方向向量计算得出的 j, k 平移向量构造的

### 5.4 mesh_bp 的真实职责 `[PROVEN]`

**mesh_bp 是 Bezier deform 的 4×4 控制点网格**，不是：
- ❌ mesh 顶点本身
- ❌ deform 参数
- ❌ 关键帧数据

**而是**：
- ✅ Bezier 曲面的 16 个控制点（4×4 grid），定义了如何变形 sprite 的顶点
- ✅ 在 PSB 中来自 `content['mesh']['bp']`（32 个 float）
- ✅ 在 asm.js 中存储为 layer 对象 offset 616 的 vector
- ✅ 当 offset 700 == 1 时启用 Bezier 模式，控制点经变换后由 xr 函数进行 tessellation

### 5.5 与 PSB section G mesh 顶点的关系 `[LIKELY]`

- mesh_bp 是 **deform 控制点**，定义如何变形
- mesh 顶点是 sprite 的 **原始顶点**，定义 sprite 形状
- 渲染时：先 mesh_bp 变形（改变顶点位置），再 matrix 旋转（整体变换）
- 叠加关系，不是替代关系 `[LIKELY]`（基于 Python 实现和 `_diag_mesh_bp_investigation.md` 的分析）

---

## 6. section G / mesh 顶点格式

### 6.1 Vertex Buffer 格式 `[PROVEN]`

从 Nx 函数（L12591）的 OpenGL 调用：

```javascript
// L12650-12653: buffer 上传
cb(34962, ...);  // glBindBuffer(GL_ARRAY_BUFFER, ...)
Lb(34962, S(k, j), h, 35040);  // glBufferData(GL_ARRAY_BUFFER, vertexCount * stride, data, GL_STATIC_DRAW)
cb(34963, ...);  // glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ...)
Lb(34963, m << 1, l, 35040);  // glBufferData(GL_ELEMENT_ARRAY_BUFFER, indexCount * 2, data, GL_STATIC_DRAW)

// L12672-12674: vertex attributes
yb(pos, 3, 5126, 0, j, 0);    // glVertexAttribPointer(pos, 3, GL_FLOAT, false, stride=44, offset=0)  — position
yb(uv,   2, 5126, 0, j, 12);   // glVertexAttribPointer(uv,   2, GL_FLOAT, false, stride=44, offset=12) — UV
yb(col,  4, 5126, 0, j, 28);   // glVertexAttribPointer(col,  4, GL_FLOAT, false, stride=44, offset=28) — color
```

### 6.2 Vertex Layout（stride = 44 bytes） `[PROVEN]`

| offset | size | type | 含义 |
|--------|------|------|------|
| 0 | 12 | 3 × float | position (x, y, z) `[PROVEN]` |
| 12 | 8 | 2 × float | UV (u, v) `[PROVEN]` |
| 20 | 8 | 2 × float | 未命名属性 `[LIKELY]`（可能是 normal 或 tangent） |
| 28 | 16 | 4 × float | color (r, g, b, a) `[PROVEN]` |
| **总计** | **44** | | |

### 6.3 Index Buffer 格式 `[PROVEN]`

- 索引类型：`GL_UNSIGNED_SHORT`（5123），每个索引 2 字节
- 索引数：`m`（从 Uw 传入）
- Draw call：`Cb(mode, count, 5123, 0)` = `glDrawElements(mode, count, GL_UNSIGNED_SHORT, 0)`（L12924/L12956）

### 6.4 顶点数据来源 `[PROVEN]`

从 Uw 函数（L9616）：
- `a + 96` / `a + 100`：vertex vector（begin/end）
- `a + 108` / `a + 112`：index vector（begin/end）
- `a + 120`：texture ID
- `a + 128`：某个 float 参数

顶点数据在 `pw` 函数（L8537）中准备，根据 blend mode 选择不同的 shader 和顶点组装方式。

### 6.5 PSB section G 引用链 `[LIKELY]`

PSB 的 section G（mesh 数据）通过以下路径进入渲染管线：

```
PSB section G (mesh 顶点数据)
  → ref1_id → section E (texture/mesh'资源引用)
    → section G (mesh 顶点)
      → 加载到 layer 对象的 vertex/index vector
        → pw 函数准备渲染数据
          → Uw → Ox → Nx → glDrawElements
```

具体加载函数和 stride 字段的精确对应关系为 `[UNVERIFIED]`（需要进一步追踪 PSB 加载代码）。

---

## 7. 完整渲染数据流

### 7.1 数据流总览 `[PROVEN]`

```
PSB/PURE-PSB 数据
  ↓ [加载阶段]
metadata/layer/drawable
  ↓
mesh/mesh_bp/texture/UV
  ↓ [运行时状态]
runtime state (layer 对象 748 bytes)
  ↓ [Update 链]
transform/physics (gu-pu)
  ↓ [hu 函数]
mesh deformation (xr/zr)
  ↓ [deform grid 顶点]
最终 vertex buffer (44 bytes/vertex)
  ↓ [Draw 链]
draw call (glDrawElements)
```

### 7.2 每级详细分析

#### Level 1: PSB → metadata/layer/drawable `[PROVEN]`

- **输入**：PSB/PURE-PSB 二进制数据
- **输出**：layer 对象数组（每个 748 bytes）
- **关键函数**：Me (L54479) → cj (L60332) → ej (L60510) → fj (L60684)
- **数据结构**：layer 对象 748 bytes，包含 type(24)、parent(28)、position(120-128)、direction vectors(92-104)、frame data(180+)、mesh_bp(616-624)、deform flags(700-716)、bounding box(696) 等
- **坐标系**：sprite 局部坐标
- **插值**：帧间插值在 Al 函数中通过 qp/Gn/nn/qo/Qo 函数实现
- **transform**：是（从 PSB 读取初始 transform）
- **deform**：否
- **GPU buffer**：否

#### Level 2: metadata → runtime state `[PROVEN]`

- **输入**：PSB frame data（每帧 212 bytes）
- **输出**：layer 对象的运行时字段
- **关键函数**：fj (L60684) 中的 `fv` 调用
- **关键字段映射**：
  - frame offset 68-76 → layer offset 616-624（mesh_bp vector）
  - frame offset 48-60 → layer offset 76-88（transform）
  - frame offset 204 → deform 控制对象
- **插值**：是（帧间）
- **transform**：是
- **deform**：否
- **GPU buffer**：否

#### Level 3: runtime state → transform/physics `[PROVEN]`

- **输入**：layer 对象的当前状态
- **输出**：更新后的 position、bounding box、deform flags
- **关键函数**：gu-pu（见 §8）
- **条件分支**：基于 flags 的不同 bit
- **插值**：是（物理模拟）
- **transform**：是
- **deform**：是（gu 修改 mesh_bp）
- **GPU buffer**：否

#### Level 4: transform → mesh deformation `[PROVEN]`

- **输入**：layer 对象的 mesh_bp（offset 616）、deform 控制对象（offset 704）、方向向量（offset 92-104）
- **输出**：deform grid 顶点（存入 offset 740 的 deform 结果对象）
- **关键函数**：hu (L53301) → xr (L14719) / zr (L15055)
- **条件分支**：
  - xr：有 deform 控制对象 + mesh_bp 非空 → Bezier tessellation
  - zr：无 deform 控制对象 + 有 parent 链 → 双线性 tessellation
  - 简单：无 deform + 无链 → 4 角点
- **坐标系**：sprite 局部坐标 → deform 后的局部坐标
- **插值**：是（Bezier/双线性）
- **transform**：是（2×2 矩阵变换控制点）
- **deform**：是（核心 deform 步骤）
- **GPU buffer**：否

#### Level 5: deform grid → vertex buffer `[PROVEN]`

- **输入**：deform grid 顶点、texture UV、color
- **输出**：44 bytes/vertex 的 vertex buffer + 2 bytes/index 的 index buffer
- **关键函数**：Ji (L57580) → Hv (L6796) → Tw (L9597) → Uw (L9616) → pw (L8537)
- **数据结构**：
  - vertex: position(3f) + UV(2f) + unknown(2f) + color(4f) = 44 bytes
  - index: unsigned short
- **坐标系**：sprite 局部 → screen/NDC
- **插值**：否（直接组装）
- **transform**：是（最终 transform 到 screen 坐标）
- **deform**：否（已完成）
- **GPU buffer**：是（glBufferData 上传）

#### Level 6: vertex buffer → draw call `[PROVEN]`

- **输入**：vertex buffer + index buffer + texture + shader uniforms
- **输出**：glDrawElements 调用
- **关键函数**：Ox (L12965) → Nx (L12591)
- **OpenGL 调用**：
  - glBindBuffer(GL_ARRAY_BUFFER / GL_ELEMENT_ARRAY_BUFFER)
  - glBufferData(GL_STATIC_DRAW)
  - glVertexAttribPointer (position/UV/color)
  - glDrawElements(GL_UNSIGNED_SHORT)
- **条件分支**：blend mode 选择 shader
- **插值**：否
- **transform**：否（在 shader 中）
- **deform**：否
- **GPU buffer**：是

---

## 8. gu-pu 触发条件

### 8.1 触发条件总表 `[PROVEN]`

所有函数在 `du(b)`（L52165）中调用，flags = `c[b + 592 >> 2]`：

| 函数 | 行号 | 触发条件 | 功能 |
|------|------|----------|------|
| **gu** | L52560 | `flags & 512` (bit 9) | stereovision parallax：对所有 layer 的 mesh_bp 应用 parallax 偏移 `[PROVEN]` |
| **hu** | L52561 | 无条件 | mesh deform：xr (Bezier) / zr (双线性) / 简单 4 角点 `[PROVEN]` |
| **iu** | L52562 | 无条件 | deform flag 设置：检查 layer 是否需要 deform，设置 offset 721 `[PROVEN]` |
| **ju** | L52563 | `flags & 32` (bit 5) | bust scale：计算 bust 位置和大小 `[LIKELY]` |
| **ku** | L52564 | 无条件 | bounding box：对 type==7 的 layer 计算 bounding box（offset 696） `[PROVEN]` |
| **lu** | L52567 | `flags & 2` (bit 1) | particle/effect 区域：根据 type 设置区域参数 `[LIKELY]` |
| **mu** | L52571 | `flags & 8` (bit 3) | update + deform：调用 cu 和 du `[PROVEN]` |
| **nu** | L52575 | `flags & 64` (bit 6) | 碰撞/关联：计算 mesh_bp 差异 `[LIKELY]` |
| **ou** | L52578 | `flags & 16` (bit 4) | update + deform：调用 cu 和 du `[PROVEN]` |
| **pu** | L52582 | `flags & 1024` (bit 10) | effect：调用 eu `[PROVEN]` |

### 8.2 各函数详细分析

#### gu (L52903-53299) `[PROVEN]`

- **触发**：`flags & 512`（stereovision parallax ratio 设置时）
- **功能**：
  1. 遍历 offset 544/548 的列表（parallax 参数）
  2. 计算 parallax 偏移量 (ba, da, ca)
  3. 对所有 layer 的 offset 616/620/624（mesh_bp 前 3 个 float）加上偏移
- **修改**：mesh_bp 数据（offset 616/620/624）
- **渲染管线阶段**：Update 阶段，hu 之前

#### hu (L53301-53957) `[PROVEN]`

- **触发**：无条件
- **功能**：mesh deform（见 §2-4 详细分析）
- **修改**：offset 740 的 deform 结果对象
- **渲染管线阶段**：Update 阶段，核心 deform 步骤

#### iu (L53959-54005) `[PROVEN]`

- **触发**：无条件
- **功能**：设置 deform flag（offset 721）
- **修改**：offset 721（是否需要 deform）
- **渲染管线阶段**：Update 阶段，hu 之后

#### ju (L54007-54086) `[LIKELY]`

- **触发**：`flags & 32`
- **功能**：bust scale 相关，计算 bust 位置和大小
- **修改**：offset 380/384（bust 位置）、offset 388-408（bust 区域）
- **渲染管线阶段**：Update 阶段

#### ku (L54088-54165) `[PROVEN]`

- **触发**：无条件
- **功能**：对 type==7 的 layer 计算 bounding box
- **修改**：offset 696（bounding box 指针）
- **渲染管线阶段**：Update 阶段

#### lu (L54167-?) `[LIKELY]`

- **触发**：`flags & 2`
- **功能**：particle/effect 区域参数设置
- **修改**：offset 744 子对象的区域参数
- **渲染管线阶段**：Update 阶段

#### mu (L83718-84261) `[PROVEN]`

- **触发**：`flags & 8`
- **功能**：某个 update + mesh deform
- **调用**：cu(w, j) + du(w)
- **渲染管线阶段**：Update 阶段，递归调用 du

#### nu (L84262-84420) `[LIKELY]`

- **触发**：`flags & 64`
- **功能**：碰撞/关联检测
- **修改**：计算 mesh_bp 差异（offset 616/620/624 的差值）
- **渲染管线阶段**：Update 阶段

#### ou (L84421-85355) `[PROVEN]`

- **触发**：`flags & 16`
- **功能**：某个 update + mesh deform
- **调用**：cu(m, e) + du(m)
- **渲染管线阶段**：Update 阶段，递归调用 du

#### pu (L85356-87456) `[PROVEN]`

- **触发**：`flags & 1024`
- **功能**：effect 处理
- **调用**：eu(0, e)
- **渲染管线阶段**：Update 阶段

---

## 9. 最重要的 3 个发现

### 发现 1：xr/zr 分支的精确条件 `[PROVEN]`

**分支条件不是简单的 "有 mesh_bp → xr，无 mesh_bp → zr"，而是三路分支**：

1. **xr (Bezier)**：需要 `B != 0`（offset 704 → offset 8 的 deform 控制对象存在）**AND** `mesh_bp vector 非空`（offset 616 的 vector 有元素）
2. **zr (双线性)**：`B == 0` 或 mesh_bp 为空，**但** `H != 0`（offset 712 的 parent 链表存在）
3. **简单 4 角点**：`B == 0` 或 mesh_bp 为空，**且** `H == 0`

**关键含义**：
- offset 704（deform 控制对象）的存在性决定 xr vs zr/简单
- offset 616（mesh_bp vector）的非空性也参与决定 xr vs zr/简单
- offset 712（parent 链表）的存在性决定 zr vs 简单
- offset 700 == 1 是 Bezier 模式的标志，但**不是** xr 的直接触发条件（xr 的触发是 B 和 A 的组合）

### 发现 2：mesh_bp 是 Bezier 控制点，不是 mesh 顶点 `[PROVEN]`

mesh_bp（32 个 float = 16 个 2D 点）是 **4×4 Bezier 曲面的控制点网格**，定义了如何变形 sprite 的顶点：

- 在 PSB 中：`content['mesh']['bp']`
- 在 asm.js 中：layer 对象 offset 616 的 vector
- 当 offset 700 == 1 时：16 个控制点经 2×2 矩阵变换后由 xr 函数进行双三次 Bezier tessellation
- 渲染顺序：**先 mesh_bp 变形（改变顶点在 sprite 局部坐标中的位置），再 matrix 旋转（整体变换）**

这与 Python 实现中的 `_draw_drawable_resource_grid` 逻辑一致：先 `_bezier_patch_eval` 求 Bezier 曲面值，再 `transform_point` 应用 matrix。

### 发现 3：完整渲染管线是 Update + Draw 双阶段 `[PROVEN]`

asm.js 的渲染管线分为两个独立阶段：

**Update 阶段**（`_EmotePlayer_Update` → Ci → eg → Al → Au → du）：
- 每帧调用，更新所有 layer 的运行时状态
- 包含 gu-pu 共 10 个子步骤，按固定顺序执行
- hu 是核心 deform 步骤，生成 deform grid 顶点
- 结果存入 layer 对象的各个 offset（120-128 position、696 bounding box、740 deform result）

**Draw 阶段**（`_EmotePlayer_Draw` → Dh → Fh → xd[16] → Ji → Hv → Tw → Uw → Ox → Nx）：
- 每帧调用，将 Update 阶段的结果渲染到 texture
- Ji 是主 draw 函数，遍历所有可见 layer
- Uw/Nx 组装 vertex buffer 并调用 glDrawElements
- vertex 格式：position(3f) + UV(2f) + unknown(2f) + color(4f) = 44 bytes/vertex

**两个阶段共享 layer 对象数组**（offset 284 的 748 字节对象数组），Update 写入，Draw 读取。

---

## 附录 A：关键 offset 速查表

### Layer 对象（748 bytes）

| offset | 类型 | 含义 | 证据 |
|--------|------|------|------|
| 24 | int | layer type | `[PROVEN]` |
| 28 | int | parent index | `[PROVEN]` |
| 37 | byte | active flag | `[PROVEN]` |
| 92 | float | direction vector x | `[PROVEN]` |
| 96 | float | direction vector y | `[PROVEN]` |
| 100 | float | direction vector z | `[PROVEN]` |
| 104 | float | direction vector w | `[PROVEN]` |
| 120-128 | float[3] | position (x, y, z) | `[PROVEN]` |
| 132 | byte | deform enable flag | `[PROVEN]` |
| 133 | byte | 简单 4 角点 flag | `[PROVEN]` |
| 160 | int | width | `[PROVEN]` |
| 164 | int | height | `[PROVEN]` |
| 180+ | frame data | 每帧 212 bytes | `[PROVEN]` |
| 604 | int | current frame index | `[PROVEN]` |
| 608 | byte | 某个 flag | `[PROVEN]` |
| 609 | byte | visible flag | `[PROVEN]` |
| 616-624 | vector | mesh_bp (16 × 2D points) | `[PROVEN]` |
| 628-648 | int[6] | 某些参数 | `[LIKELY]` |
| 696 | ptr | bounding box | `[PROVEN]` |
| 700 | int | deform mode (1 = Bezier) | `[PROVEN]` |
| 704 | ptr | deform 控制对象 | `[PROVEN]` |
| 708 | byte | deform active flag | `[PROVEN]` |
| 709 | byte | 某个 flag | `[PROVEN]` |
| 710 | byte | 某个 flag | `[PROVEN]` |
| 712 | ptr | parent 链表指针 | `[PROVEN]` |
| 716 | ptr | 某个状态指针 | `[PROVEN]` |
| 721 | byte | deform needed flag | `[PROVEN]` |
| 728 | int | 某个参数 | `[PROVEN]` |
| 740 | ptr | deform 结果对象 (44 bytes) | `[PROVEN]` |
| 744 | ptr | type-specific 数据 | `[PROVEN]` |

### Frame data（212 bytes，从 offset 180 开始）

| offset | 类型 | 含义 | 证据 |
|--------|------|------|------|
| 18-19 | byte[2] | 某些 flags | `[PROVEN]` |
| 40-44 | float[2] | 某个 offset | `[PROVEN]` |
| 48-60 | int[4] | transform 参数 | `[PROVEN]` |
| 64 | int | 某个参数 | `[PROVEN]` |
| 68-76 | vector | mesh_bp 数据 | `[PROVEN]` |
| 80-81 | byte[2] | 某些 flags | `[PROVEN]` |
| 84-100 | int[5] | 某些参数 | `[PROVEN]` |
| 204 | ptr | deform 控制数据 | `[PROVEN]` |
| 208 | ptr | type-specific 数据 | `[PROVEN]` |

### Deform 控制对象（92 bytes，由 Qr 初始化）

| offset | 类型 | 含义 | 证据 |
|--------|------|------|------|
| 0 | ptr | Bezier 基函数数组 | `[LIKELY]` |
| 4 | int | 某个参数 | `[UNKNOWN]` |
| 8 | ptr | 控制点数组 | `[PROVEN]` |
| 16 | int | grid 宽 - 1 | `[PROVEN]` |
| 20 | int | grid 高 - 1 | `[PROVEN]` |
| 24 | ptr | 控制点 vector | `[PROVEN]` |
| 28-32 | vector | 控制点数据 | `[PROVEN]` |
| 40-88 | float[12] | 变换矩阵 + 逆矩阵 + 平移 | `[PROVEN]` |

---

## 附录 B：函数表

### wd 表（8 元素，Update 相关）`[PROVEN]`

```
wd = [vB, $e, df, ff, hf, eg, jg, lg]
```
- wd[5] = eg = Update wrapper（通过 `_EmotePlayer_Update` 调用）

### xd 表（128 元素，Draw/生命周期相关）`[PROVEN]`

关键元素：
- xd[1] = re = 析构函数
- xd[5] = Qe = 析构函数
- xd[10] = bg = gj = 清理函数
- xd[16] = Ji = **主 draw 函数**
- xd[44] = Xo = 析构函数
- xd[45] = Yo = 析构函数

### Bd 表（64 元素，分配/构造相关）`[PROVEN]`

- Bd[c[513] & 63] = 内存分配函数（malloc）
- xd[c[514] & 127] = 内存释放函数（free）

---

## 附录 C：OpenGL 函数映射

| asm.js 函数 | OpenGL 函数 | 证据 |
|-------------|-------------|------|
| `cb(target, buffer)` | `glBindBuffer` | `[PROVEN]` |
| `Lb(target, size, data, usage)` | `glBufferData` | `[PROVEN]` |
| `yb(index, size, type, norm, stride, ptr)` | `glVertexAttribPointer` | `[PROVEN]` |
| `Cb(mode, count, type, indices)` | `glDrawElements` | `[PROVEN]` |
| `Qa(texture)` | `glBindTexture` | `[LIKELY]` |
| `Hc(location)` | `glEnableVertexAttribArray` | `[LIKELY]` |
| `hb(location)` | `glDisableVertexAttribArray` | `[LIKELY]` |

---

*文档生成时间：2026-09-24*
*分析基于：reference/FreeMoteDriver-format.js（99629 行）*