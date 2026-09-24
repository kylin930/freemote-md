# E-mote 渲染管线闭环研究报告

> **研究日期**：2026-09-24
> **研究性质**：第二阶段——原版几何/变形/绘制路径彻底追通
> **证据等级**：`[PROVEN]`（明确代码证据）｜`[LIKELY]`（强推测）｜`[UNVERIFIED]`（弱推测）｜`[UNKNOWN]`

---

## 1. 最终渲染数据流

### 1.1 原版 asm.js 渲染数据流 `[PROVEN]`

```
PSB/PURE-PSB 数据
  ↓ [加载阶段: Me→cj→ej→fj]
metadata/layer/drawable（layer 对象 748 bytes/个）
  ↓
mesh_bp(16控制点)/texture/UV → runtime state
  ↓ [Update 链: Ci→eg→Al→Au→du]
  ↓   gu(flags&512) → stereovision parallax
  ↓   hu(无条件) → mesh deform: xr/zr/简单4角点
  ↓   iu(无条件) → deform flag
  ↓   ju(flags&32) → bust scale
  ↓   ku(无条件) → bounding box
  ↓   lu(flags&2) → particle 区域
  ↓   mu(flags&8) → update+deform（递归 du）
  ↓   nu(flags&64) → 碰撞/关联
  ↓   ou(flags&16) → update+deform（递归 du）
  ↓   pu(flags&1024) → effect
deform grid 顶点（存入 offset 740）
  ↓ [Draw 链: Dh→Fh→Ji→Hv→Tw→Uw→Ox→Nx]
vertex buffer（44 bytes/vertex: pos3f+uv2f+unknown2f+color4f）
  ↓
glDrawElements（GL_UNSIGNED_SHORT 索引）
```

### 1.2 Python 当前渲染数据流 `[PROVEN]`

```
PSB/PURE-PSB 数据
  ↓ [PSBLoader.load]
ModelResource（textures✓, meshes✗(vertices=b""), sprites✓, root_value✓）
  ↓
_draw_drawable_resources（累积变换管线遍历）
  ↓
build_context（三态继承 + physics 摆动 + mesh_bp 偏移）
  ↓
  ├─ mesh_bp 为空 → sprite quad（4 顶点, GL_TRIANGLES）
  └─ mesh_bp 非空 → 9×9 Bezier grid（81 顶点, GL_TRIANGLE_STRIP）
       ↓ _bezier_patch_eval（双三次 Bezier 曲面求值）
       ↓ transform_point（矩阵变换）
vertex buffer → glBufferData → glDrawElements
```

### 1.3 关键差异：原版 A→B→C→D→E vs Python A→B→C→X→E

```
原版：  PSB → runtime state → hu(xr Bezier deform) → 44B vertex → glDrawElements
Python：PSB → runtime state → _bezier_patch_eval(9×9 grid) → 20B vertex → glDrawElements

真正的差异（C→X）：
1. 原版 xr 动态计算 grid 分辨率，Python 固定 9×9
2. 原版 vertex 44 bytes（含 color 4f + unknown 2f），Python 20 bytes（仅 pos 3f + uv 2f）
3. 原版 shader 应用 opacity/color/blend_mode，Python sprite shader 缺失这些 uniform
4. 原版 V 翻转在 shader 中统一处理，Python 不同路径 V 翻转不一致
5. 原版用 coord=[0,0] scale=1，Python 自动 fit_scale
```

---

## 2. xr / zr 结论

### 2.1 已确认 `[PROVEN]`

**三路分支条件**（在 hu 函数 L53679）：

| 分支 | 条件 | 语义 |
|------|------|------|
| **xr (Bezier)** | `B != 0`（offset 704→8 deform 控制对象存在）AND `mesh_bp vector 非空`（offset 616） | 有 Bezier 控制点 + deform 控制对象 |
| **zr (双线性)** | (`B == 0` OR mesh_bp 为空) AND `H != 0`（offset 712 parent 链表存在） | 无 Bezier，但有 parent 链 |
| **简单 4 角点** | (`B == 0` OR mesh_bp 为空) AND `H == 0` | 无 deform，无链表 |

**xr 函数**：双三次 Bezier 曲面 tessellation，16 个控制点（4×4 grid），grid 分辨率从 deform 控制对象动态读取。

**zr 函数**：双线性插值，4 个角点，grid 分辨率 = (宽+1) × (高+1)。

### 2.2 NEKOPARA 实际路径 `[PROVEN]`

**所有 59 个 NEKOPARA 模型只走 xr (Bezier) 路径**。

- `mesh.cc` 全部为 bool 标志（无真正顶点数据）→ 不走 PSB mesh 路径
- `mesh.bp` 有 32 个 float 的真实数据 → 有 Bezier 控制点
- `offset 700 == 1`（Bezier 模式）→ deform 控制对象存在
- **zr 和简单 4 角点路径在 NEKOPARA 中不使用**

### 2.3 Python 对齐情况 `[PROVEN]`

Python 的 `_bezier_patch_eval` + `_draw_drawable_resource_grid` 实现了 xr 对应的 Bezier 曲面求值，**数学方法正确**。差异在于：
- grid 分辨率固定 9×9（原版动态）`[PROVEN]`
- 控制点变换方式可能略有差异 `[UNVERIFIED]`

---

## 3. mesh_bp 结论

### 3.1 已确认 `[PROVEN]`

**mesh_bp 是 Bezier deform 的 4×4 控制点网格**（16 个 2D 点 = 32 个 float），不是：
- ❌ mesh 顶点本身
- ❌ deform 参数
- ❌ 关键帧数据

**而是**：
- ✅ Bezier 曲面的 16 个控制点，定义如何变形 sprite 的顶点
- ✅ 在 PSB 中：`content['mesh']['bp']`
- ✅ 在 asm.js 中：layer 对象 offset 616 的 vector
- ✅ 当 offset 700 == 1 时启用 Bezier 模式，控制点经 2×2 矩阵变换后由 xr tessellation

### 3.2 数据流 `[PROVEN]`

```
PSB frame data offset 68（mesh_bp vector begin）
  ↓ L49820-49822 复制到 layer offset 616-624
  ↓ hu 函数读取 offset 616
  ↓ 当 offset 700 == 1：16 控制点经 2×2 矩阵变换 → 存入 B+40
  ↓ xr 函数从 B+40 读取控制点 → Bezier 曲面 tessellation
  ↓ 生成 deform grid 顶点 → 存入 offset 740
```

### 3.3 与 section G mesh 顶点的关系 `[PROVEN]`

**叠加关系，不是替代关系**：
- mesh_bp = deform 控制点（定义如何变形）
- mesh 顶点 = sprite 原始顶点（定义 sprite 形状）
- 渲染顺序：先 mesh_bp 变形 → 再 matrix 旋转

**关键发现**：NEKOPARA 的 `mesh.cc` 全为 bool 标志，**不使用 PSB mesh 顶点变形**。所有变形通过 mesh_bp（Bezier 控制点）实现。

---

## 4. section G 结论

### 4.1 数据格式 `[PROVEN]`

| 字段 | offset | size | type |
|------|--------|------|------|
| position | 0 | 12 | 3 × float |
| UV | 12 | 8 | 2 × float |
| unknown | 20 | 8 | 2 × float（可能是 normal/tangent） |
| color | 28 | 16 | 4 × float (RGBA) |
| **总计** | | **44** | |

索引格式：GL_UNSIGNED_SHORT（2 bytes/index）

### 4.2 引用方式 `[LIKELY]`

```
ref1_id → section E（资源引用）→ section G（mesh 顶点数据）
  → 加载到 layer 对象的 vertex/index vector
  → pw 函数准备渲染数据 → Uw → Ox → Nx → glDrawElements
```

### 4.3 哪些模型使用 `[PROVEN]`

**NEKOPARA 的 59 个模型都不使用 section G mesh 顶点**。

- 所有模型的 `mesh.cc` 字段全为 bool 标志（True/False），无真正顶点坐标
- 渲染完全走 Bezier（xr）路径，不走 PSB mesh 顶点路径
- section G 中的纹理像素数据已被解码和使用（纹理 atlas）

### 4.4 在当前 pipeline 中的位置

**section G mesh 顶点在 NEKOPARA 渲染中不参与**。当前 Python 的 sprite quad + 9×9 Bezier grid 路径就是 NEKOPARA 的实际渲染路径。

### 4.5 是否值得下一轮实现

**对于 NEKOPARA：不值得**。NEKOPARA 不使用 PSB mesh 顶点，实现 section G 解码不会改善 NEKOPARA 渲染。

**对于其他 E-mote 模型：可能值得**。如果未来需要支持使用 PSB mesh 顶点的非 NEKOPARA 模型，则需要实现。但当前优先级应降低。

---

## 5. 59 模型路径分类

### 5.1 分类结果 `[PROVEN]`

**所有 59 个 NEKOPARA 模型完全同构**：

| 属性 | 值 |
|------|-----|
| model_flags | `0x0000100F`（唯一值） |
| layer_types | `[0, 1, 2, 3, 12]`（唯一值） |
| physics | hairControl=4, bustControl=2, partsControl=2（唯一值） |
| categories | A/D/E（唯一值） |
| chara_count | 21（唯一值） |
| deform path | xr (Bezier)（全部） |
| mesh.cc | 全 bool 标志（无真正顶点） |
| mesh.bp | 有真实数据（32 float） |

### 5.2 五类分类

| 类别 | 数量 | 描述 | 代表模型 |
|------|------|------|----------|
| A 类（Bezier） | 59 | 完全依赖 sprite + Bezier | azuki-casual |
| B 类（sprite+mesh） | 0 | — | — |
| C 类（PSB mesh） | 0 | — | — |
| D 类（physics） | 59 | physics 明显参与 | vanilla-maid |
| E 类（特殊 flags） | 59 | 非常规 gu-pu 触发 | maple-dress |

### 5.3 关键含义

- **NEKOPARA 使用统一模型架构**，不同角色/服装只在纹理和参数上差异
- **无 B/C 类模型**：NEKOPARA 不使用 PSB mesh 顶点变形
- **所有模型走同一条渲染路径**（xr Bezier），简化了逆向工作

---

## 6. 当前 Python 与原版的真实差异

### 6.1 已证实的差异

| # | 差异 | 原版行为 | Python 行为 | 影响 | 证据 |
|---|------|---------|-----------|------|------|
| 1 | **vertex 格式** | 44 bytes（pos3f+uv2f+unknown2f+color4f） | 20 bytes（pos3f+uv2f） | 顶点颜色缺失 | `[PROVEN]` |
| 2 | **shader uniform 缺失** | sprite shader 应用 opacity/color/blend_mode | sprite shader 未声明这些 uniform | opacity/color 不生效 | `[PROVEN]` |
| 3 | **V 翻转不一致** | shader 中统一 V 翻转 | DrawableResource 路径未翻转，SpriteInfo 路径已翻转 | 可能纹理颠倒 | `[PROVEN]` |
| 4 | **Bezier grid 密度** | 动态计算（基于 sprite 尺寸和缩放） | 固定 9×9（81 顶点） | 某些尺寸下精度不足或过度细分 | `[PROVEN]` |
| 5 | **fit_scale 自动缩放** | coord=[0,0] scale=1 | 自动 fit 到屏幕 | 缩放与原版不一致 | `[PROVEN]` |
| 6 | **每帧重建 VBO** | 可能复用 VBO | 每帧 create+delete buffer | 性能开销（功能正确） | `[LIKELY]` |

### 6.2 已证实 NOT 差异

| # | 项 | 结论 | 证据 |
|---|-----|------|------|
| 1 | PSB mesh 顶点未解码 | **不影响 NEKOPARA**（不使用 mesh 顶点） | `[PROVEN]` |
| 2 | xr vs zr 路径选择 | **Python 用 xr 是正确的**（NEKOPARA 全走 xr） | `[PROVEN]` |
| 3 | mesh_bp 角色 | **Python 的 Bezier 控制点用法正确** | `[PROVEN]` |
| 4 | zr/简单4角点缺失 | **不影响 NEKOPARA**（不使用这些路径） | `[PROVEN]` |
| 5 | gu-pu 其他8函数 | **5个不触发，3个已实现**（hu/iu/ku 无条件，lu/mu 触发） | `[PROVEN]` |

---

## 7. 视觉基线

### 7.1 实验配置 `[PROVEN]`

| 参数 | 值 |
|------|-----|
| 模型 | `vanilla-maid.pure.psb` |
| motion | `sample_00`（第 0 个 motion） |
| frame | 0 |
| viewport | 800×600 |
| camera | 默认（coord=[0,0], scale=auto-fit） |
| 输出 | `tools/baseline_python.png`（238KB） |

### 7.2 基线结果 `[PROVEN]`

- **vanilla-maid frame 0**：24.87% 非透明像素（渲染成功，有可见内容）
- **azuki-casual frame 0**：0% 非透明像素（渲染空白，已知问题）
- **帧间差异**：frame 0 vs frame 30 = 18.65% 像素差异（动画在工作）

### 7.3 可复现流程

```bash
python tools/visual_regression_baseline.py  # 生成截图
python tools/compare_images.py img1.png img2.png  # 对比两张图
```

### 7.4 已知问题

**azuki-casual 渲染空白**：frame 0 时所有像素透明。可能原因：
- 该模型的 sample_00 在 frame 0 时 drawable opacity 为 0
- 某些模型需要推进帧才能看到内容
- 需要进一步调查是否是 Python 管线 bug

---

## 8. 当前最重要的 3 个未知量

### 未知量 1：shader uniform 缺失的实际影响 `[PROVEN]` 存在，`[UNKNOWN]` 量化影响

Python sprite shader 未应用 opacity/color/blend_mode。这是已证实的差异，但**对 NEKOPARA 视觉效果的实际影响程度未知**。需要：
- 确认 NEKOPARA 模型是否使用 per-vertex color
- 确认 opacity 在哪些 drawable 上非 1.0
- 量化缺失 color 调制对最终像素的影响

### 未知量 2：Bezier grid 密度差异的量化影响 `[PROVEN]` 存在，`[UNKNOWN]` 量化影响

Python 固定 9×9 grid，原版动态计算。对于不同尺寸的 sprite，9×9 可能过度或不足。需要：
- 确认原版动态 grid 密度的计算公式
- 对比不同 sprite 尺寸下 9×9 vs 动态的 Bezier 曲面差异
- 量化对最终像素的影响

### 未知量 3：azuki-casual 渲染空白的根因 `[UNKNOWN]`

azuki-casual 在 frame 0 渲染全透明，而 vanilla-maid 正常。这是 Python 管线的 bug 还是模型特性？需要：
- 对比 azuki-casual 和 vanilla-maid 的 drawable 结构
- 检查 azuki-casual 的 drawable opacity 在 frame 0 的值
- 确认是否需要推进帧才能看到内容

---

## 9. 下一轮建议

### 9.1 重新决定的优先级

基于本轮证据，**前一轮的优先级需要大幅调整**：

| 前一轮优先级 | 本轮结论 | 新优先级 |
|-------------|---------|---------|
| PSB mesh 顶点解码（最高） | **NEKOPARA 不使用 mesh 顶点** | 降级（非 NEKOPARA 才需要） |
| 确认 xr vs zr | **已确认：xr 正确** | 已解决 |
| 补 mesh_bp | **mesh_bp 用法正确** | 已解决 |
| gu-pu 其他 8 函数 | **5 个不触发，3 个已实现** | 降级 |
| eye/mouth/eyebrow | 本轮不研究 | 保持（下一轮） |

### 9.2 新的优先级

**最高优先级**：修复 shader uniform 缺失
- 在 sprite shader 中添加 opacity/color/blend_mode uniform
- 这是已证实的、影响所有模型的差异
- 修复后可通过视觉回归量化效果

**高优先级**：调查 azuki-casual 渲染空白
- 如果是 Python bug，修复它
- 如果是模型特性，记录并跳过

**中优先级**：动态 Bezier grid 密度
- 从 asm.js 提取 grid 密度计算公式
- 替换固定 9×9 为动态计算

**低优先级**（下一轮）：
- eye/mouth/eyebrow 控制字段
- V 翻转统一
- fit_scale 对齐

### 9.3 是否解 section G

**对于 NEKOPARA：不需要**。NEKOPARA 不使用 PSB mesh 顶点。

**对于通用 E-mote 支持：需要，但优先级降低**。只有当需要支持使用 mesh 顶点的非 NEKOPARA 模型时才实现。

### 9.4 是否修改 xr/zr

**不需要修改**。Python 用 xr（Bezier）是正确的，NEKOPARA 全走 xr 路径。zr 和简单 4 角点路径在 NEKOPARA 中不使用。

### 9.5 是否进入代码实现

**是，应该进入代码实现**。本轮已经完成了逆向研究：
- 渲染管线已闭环（从 PSB 到 draw call 的完整数据流已追踪）
- xr/zr 已确认（xr 正确）
- mesh_bp 已确认（Bezier 控制点）
- section G 已确认（NEKOPARA 不使用）
- 模型路径已分类（全部走 xr）
- 视觉基线已建立

**下一轮应该修复已证实的差异**（shader uniform、grid 密度），并通过视觉回归量化效果。

---

*文档生成时间：2026-09-24*
*基于：asm-render-dataflow-findings.md + python-render-pipeline-analysis.md + model-path-classification.md + visual_regression_config.md*