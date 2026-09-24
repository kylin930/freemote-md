# E-mote 渲染数据流完整文档

> **研究日期**：2026-09-24
> **证据来源**：asm.js 逆向（`reference/FreeMoteDriver-format.js`）+ Python 代码分析 + 59 模型扫描
> **证据等级**：`[PROVEN]`｜`[LIKELY]`｜`[UNVERIFIED]`｜`[UNKNOWN]`

---

## 1. 原版 asm.js 完整调用链

### 1.1 Update 链（每帧状态更新）`[PROVEN]`

```
_E motePlayer_Update (Ci, L57514)
  → wd[vtable+272 & 7](a, b)           // L57522 虚函数分发
    → eg (L55402, wd[5])               // wrapper: 取 a+16 子对象
      → Al(c[a+16>>2], b)              // L55405 时间步进
        → Au(children[f], w)           // L78746 循环每个子对象
          → cu(b, d)                   // L87488 某 update
          → du(b)                      // L87489 主 update（含 gu-pu）
            → gu(b)  if flags&512      // L52560 stereovision parallax
            → hu(b)                    // L52561 mesh deform (xr/zr) ←核心
            → iu(b)                    // L52562 deform flag 设置
            → ju(b)  if flags&32       // L52563 bust scale
            → ku(b)                    // L52564 bounding box
            → lu(b)  if flags&2        // L52567 particle 区域
            → mu(b)  if flags&8        // L52571 update+deform（递归 du）
            → nu(b)  if flags&64       // L52575 碰撞/关联
            → ou(b)  if flags&16       // L52578 update+deform（递归 du）
            → pu(b)  if flags&1024     // L52582 effect
          → xd[vtable+16 & 127](b)    // L87490 后处理回调
```

### 1.2 Draw 链（每帧渲染）`[PROVEN]`

```
_E motePlayer_Draw (Dh, L56408)
  → Fh(a)                              // L56410 = _EmotePlayer_DrawToTexture
    → OpenGL 状态保存 (L56570-56607)   // viewport/blend/depth/stencil
    → xd[vtable+280 & 127](b)          // L56609 虚函数分发
      → Ji (L57580, xd[16])            // 主 draw 函数
        → Hv(b, ca)                    // L58857 设置 uniform + draw
          → Tw(a, e)                    // L6809
            → Uw(a)                     // L9605 准备渲染数据
              → Ox(...)                // L9651
                → Nx(5, ...)           // L12977 OpenGL draw
                  → glBindBuffer       // L12650-12653
                  → glBufferData       // 上传 vertex/index buffer
                  → glVertexAttribPointer // L12672-12674
                  → glDrawElements     // L12924/L12956
```

### 1.3 初始化链 `[PROVEN]`

```
Me (L54479, 构造函数)
  → cj(s, r, t, 1)                     // L54526
    → ej(b)                            // L60452
      → fj(b, n)                       // L60678 从 PSB 加载配置
        → Au(c[C>>2], 0.0)             // L60744 首次 update
```

---

## 2. 每级数据流详细分析

### Level 1: PSB → metadata/layer/drawable `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | PSB/PURE-PSB 二进制数据 |
| **输出** | layer 对象数组（每个 748 bytes） |
| **关键函数** | Me → cj → ej → fj |
| **数据结构** | layer 748B：type(24)、parent(28)、position(120-128)、dir vectors(92-104)、frame data(180+)、mesh_bp(616-624)、deform flags(700-716)、bbox(696) |
| **坐标系** | sprite 局部坐标 |
| **插值** | 帧间插值在 Al 中通过 qp/Gn/nn/qo/Qo 实现 |
| **transform** | 是（从 PSB 读取初始 transform） |
| **deform** | 否 |
| **GPU buffer** | 否 |

### Level 2: metadata → runtime state `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | PSB frame data（每帧 212 bytes） |
| **输出** | layer 对象运行时字段 |
| **关键函数** | fj 中的 fv 调用 |
| **字段映射** | frame offset 68-76 → layer 616-624（mesh_bp）；frame 48-60 → layer 76-88（transform）；frame 204 → deform 控制对象 |
| **插值** | 是（帧间） |
| **transform** | 是 |
| **deform** | 否 |
| **GPU buffer** | 否 |

### Level 3: runtime state → transform/physics `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | layer 对象当前状态 |
| **输出** | 更新后的 position、bbox、deform flags |
| **关键函数** | gu-pu（10 个子步骤） |
| **条件分支** | 基于 flags 的不同 bit |
| **插值** | 是（物理模拟） |
| **transform** | 是 |
| **deform** | 是（gu 修改 mesh_bp） |
| **GPU buffer** | 否 |

### Level 4: transform → mesh deformation `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | layer 的 mesh_bp(616)、deform 控制对象(704)、方向向量(92-104) |
| **输出** | deform grid 顶点（存入 offset 740） |
| **关键函数** | hu → xr / zr / 简单4角点 |
| **条件分支** | xr：B≠0 且 mesh_bp 非空；zr：B=0 或 mesh_bp 空，且 H≠0；简单：B=0 或 mesh_bp 空，且 H=0 |
| **坐标系** | sprite 局部 → deform 后局部 |
| **插值** | 是（Bezier/双线性） |
| **transform** | 是（2×2 矩阵变换控制点） |
| **deform** | 是（核心 deform 步骤） |
| **GPU buffer** | 否 |

### Level 5: deform grid → vertex buffer `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | deform grid 顶点、texture UV、color |
| **输出** | 44B/vertex buffer + 2B/index buffer |
| **关键函数** | Ji → Hv → Tw → Uw → pw |
| **vertex 格式** | pos(3f) + UV(2f) + unknown(2f) + color(4f) = 44B |
| **坐标系** | sprite 局部 → screen/NDC |
| **插值** | 否（直接组装） |
| **transform** | 是（最终 transform 到 screen） |
| **deform** | 否（已完成） |
| **GPU buffer** | 是（glBufferData 上传） |

### Level 6: vertex buffer → draw call `[PROVEN]`

| 属性 | 值 |
|------|-----|
| **输入** | vertex buffer + index buffer + texture + shader uniforms |
| **输出** | glDrawElements 调用 |
| **关键函数** | Ox → Nx |
| **OpenGL 调用** | glBindBuffer, glBufferData, glVertexAttribPointer, glDrawElements |
| **条件分支** | blend mode 选择 shader |
| **GPU buffer** | 是 |

---

## 3. xr 函数详细分析

### 3.1 签名与参数 `[PROVEN]`

```javascript
function xr(a, b, d, e)  // L14719
// a = B: deform 控制对象（92B，含 grid 分辨率、控制点数组指针）
// b = z: 输出 vector 指针
// d = B+52: 2×2 变换矩阵 + 逆矩阵 + 平移（36B）
// e = W: 平移向量 (j, k)（8B）
```

### 3.2 执行流程 `[PROVEN]`

1. **步骤 1**：读取 16 个控制点，经 2×2 矩阵变换后存入栈上临时空间（128B = 16×8）
2. **步骤 2**：读取 grid 分辨率 W = a[16]+1, X = a[20]+1
3. **步骤 3**：调整输出 vector 容量为 W×X 个 2D 点
4. **步骤 4**：双重循环 tessellation，对每个 (u,v) 计算双三次 Bezier 曲面值

### 3.3 输出 `[PROVEN]`

W×X 个 2D 顶点（每个 8B），写入输出 vector。Bezier 曲面 tessellation。

---

## 4. zr 函数详细分析

### 4.1 签名与参数 `[PROVEN]`

```javascript
function zr(a, b, d, e)  // L15055
// a = W: 4 个角点（32B = 4×8）
// b = wa: grid 宽（已 +1）
// d = xa: grid 高（已 +1）
// e = z: 输出 vector 指针
```

### 4.2 执行流程 `[PROVEN]`

标准双线性插值：`P(u,v) = (1-u)(1-v)*P00 + u(1-v)*P10 + (1-u)v*P01 + uv*P11`

输出 (b+1)×(d+1) 个 2D 顶点。

---

## 5. mesh_bp 数据流

```
PSB content['mesh']['bp']（32 float = 16 个 2D 点）
  ↓ PSBLoader 解析
  ↓ 存入 SpriteInfo.mesh_bp
  ↓ motion_painter.build_context 读取
  ↓ 帧间插值（type 3 线性插值）
  ↓ 存入 layer offset 616-624（asm.js）
  ↓ hu 函数读取
  ↓ 当 offset 700 == 1：16 控制点经 2×2 矩阵变换
  ↓ xr 函数 Bezier tessellation
  ↓ 生成 deform grid 顶点
  ↓ 存入 offset 740
  ↓ Draw 链读取，组装进 vertex buffer
  ↓ glDrawElements
```

**mesh_bp = Bezier 控制点（4×4 grid），不是 mesh 顶点** `[PROVEN]`

---

## 6. gu-pu 触发条件与 NEKOPARA 实际触发

| 函数 | 触发条件 | 功能 | NEKOPARA 触发 | 原因 |
|------|----------|------|--------------|------|
| gu | flags & 512 (bit 9) | stereovision parallax | ❌ 0/59 | 无 layer type 9 |
| hu | 无条件 | mesh deform (xr/zr) | ✅ 59/59 | — |
| iu | 无条件 | deform flag 设置 | ✅ 59/59 | — |
| ju | flags & 32 (bit 5) | bust scale | ❌ 0/59 | 无 layer type 5 |
| ku | 无条件 | bounding box | ✅ 59/59 | — |
| lu | flags & 2 (bit 1) | particle 区域 | ✅ 59/59 | 有 layer type 1 |
| mu | flags & 8 (bit 3) | update+deform | ✅ 59/59 | 有 layer type 3 |
| nu | flags & 64 (bit 6) | 碰撞/关联 | ❌ 0/59 | 无 layer type 6 |
| ou | flags & 16 (bit 4) | update+deform | ❌ 0/59 | 无 layer type 4 |
| pu | flags & 1024 (bit 10) | effect | ❌ 0/59 | 无 layer type 10 |

**flags = OR(1 << layer_type) for all layer types = 0x100F = bits {0,1,2,3,12}** `[PROVEN]`

---

## 7. Layer 对象关键 offset 速查

| offset | 类型 | 含义 | 证据 |
|--------|------|------|------|
| 24 | int | layer type | `[PROVEN]` |
| 28 | int | parent index | `[PROVEN]` |
| 92-104 | float[4] | 方向向量 | `[PROVEN]` |
| 120-128 | float[3] | position | `[PROVEN]` |
| 132 | byte | deform enable flag | `[PROVEN]` |
| 160/164 | int | width/height | `[PROVEN]` |
| 180+ | frame data | 每帧 212B | `[PROVEN]` |
| 616-624 | vector | mesh_bp (16×2D points) | `[PROVEN]` |
| 696 | ptr | bounding box | `[PROVEN]` |
| 700 | int | deform mode (1=Bezier) | `[PROVEN]` |
| 704 | ptr | deform 控制对象 | `[PROVEN]` |
| 712 | ptr | parent 链表指针 | `[PROVEN]` |
| 740 | ptr | deform 结果对象 (44B) | `[PROVEN]` |

---

## 8. Vertex 格式

### 原版 `[PROVEN]`

| offset | size | type | 含义 |
|--------|------|------|------|
| 0 | 12 | 3×float | position (x,y,z) |
| 12 | 8 | 2×float | UV (u,v) |
| 20 | 8 | 2×float | unknown（normal/tangent?） |
| 28 | 16 | 4×float | color (r,g,b,a) |
| **总计** | **44** | | |

索引：GL_UNSIGNED_SHORT（2B/index）

### Python 当前 `[PROVEN]`

| offset | size | type | 含义 |
|--------|------|------|------|
| 0 | 12 | 3×float | position (x,y,z) |
| 12 | 8 | 2×float | UV (u,v) |
| **总计** | **20** | | |

**差异**：Python 缺少 unknown(8B) 和 color(16B) 字段 `[PROVEN]`

---

*文档生成时间：2026-09-24*
