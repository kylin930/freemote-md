# Renderer 收尾审计最终报告

> 本报告综合本轮 Renderer 收尾审计（P0–P6 七项）的全部结果，作为 Runtime/Physics 阶段开始前的最后验证基线。
>
> 证据等级：`[PROVEN]`（已证实，有代码/数据锚点） / `[LIKELY]`（高度可信但未完全闭环） / `[UNVERIFIED]`（未验证） / `[UNKNOWN]`（未知）。

---

## 1. 概述

- **目标**：Renderer 收尾审计 / Runtime 前最后验证。
- **方法**：P0–P6 七项审计，每项结论标注 `[PROVEN]` / `[LIKELY]` / `[UNVERIFIED]` / `[UNKNOWN]`。
- **审计范围**：
  - P0 baseline 时间与版本校正
  - P1 shader uniform 语义复核
  - P2 meshDivision 数据来源
  - P3 dynamic grid 最终验证
  - P4 44B vertex 最终定性
  - P5 59 模型 multi-texture 最终验证
  - P6 renderer 剩余差异表 + 封版结论
- **结论**：**✅ Renderer 可以封版** `[PROVEN]`
  - 无 `NEEDS-FIX` 项。
  - 剩余差异均为 `DIFFERENCE-BUT-IRRELEVANT`（对 NEKOPARA 无视觉影响）或 `DIFFERENCE-BUT-VIEWER-LEVEL`（viewer 层差异，保留）。
  - 无 renderer 回归。

---

## 2. P0: baseline 时间与版本校正

- `baseline_summary.json` 在多纹理修复期间生成（23:38），`azuki-casual=35` 是**修复后值** `[PROVEN]`。
- 三阶段命名已建立，消除报告中数据冲突 `[PROVEN]`：
  - `baseline_pre_renderer_cleanup` — Renderer 清理前基线
  - `renderer_cleanup_after_p5` — P5 多纹理修复后基线
  - `current` — 当前最新基线
- 报告数据冲突已修正 `[PROVEN]`：明确区分"修复前 0 draw call"与"修复后 35 draw call"，不再混用同一基线名。

---

## 3. P1: shader uniform 语义复核

四个 uniform 的语义审计结果：

| uniform (Python) | asm.js 名称 | asm.js 语义 | Python 实现 | 语义一致？ | NEKOPARA 影响 | 证据等级 |
|---|---|---|---|---|---|---|
| `u_spriteOpacity` | `u_testAlpha` | alpha test threshold（discard if alpha <= threshold） | opacity 乘法（`alpha *= opacity`） | ❌ 不一致 | 无（行为接近） | `[PROVEN]` |
| `u_filterColor` | `u_filterColor` | 设置了但 shader 不消费 | 用 `.a` 做 alpha 调制 | ❌ Python 错误消费 | 无（`.a=1.0`, `colorMod=1.0`） | `[PROVEN]` |
| `u_preFilterColor` | `u_preFilterColor` | `.a` 用作灰度混合因子 | 用 `.a` 做 alpha 调制 | ❌ 不一致 | 无（NEKOPARA 走简单 FS 变体） | `[PROVEN]` |
| `u_meshColorControl` | 不存在 | asm.js 中无此 uniform | color 调制开关 | ❌ Python 发明 | 无（`mesh_cc=True → 1.0`） | `[PROVEN]` |

**结论**：4 个 uniform 语义与 asm.js 不一致，但在 NEKOPARA 中均无视觉影响 `[PROVEN]`。**不需要修改**，标记为 `DIFFERENCE-BUT-IRRELEVANT`。

---

## 4. P2: meshDivision 数据来源

- **PSB key**：`"meshDivision"` `[PROVEN]`
- **asm.js 解析**：L48566 `gv(x, f, 10157, 28304); c[z+4>>2] = Uu(x)` `[PROVEN]`
- **实际值**：`16`（4138 occurrences）和 `20`（16363 occurrences）— **不是 10！** `[PROVEN]`
- **层级**：layer 级字段，只在 `meshTransform=1` 的 layer 中存在 `[PROVEN]`
- **Python 之前硬编码** `DEFAULT_MESH_DIVISION=10` 是**错误的** `[PROVEN]`
- **已修复**：从 PSB 解析 `meshDivision`，传递到渲染管线 `[PROVEN]`
- **修改文件**：`model.py`, `motion_painter.py`, `loader.py`, `player.py`

---

## 5. P3: dynamic grid 最终验证

- **18/18 测试矩阵全部一致** `[PROVEN]`
- 修复了 `total=0` 和 `denom=0` 边界情况 `[PROVEN]`
- 逐项确认（与 asm.js `bu` 完全对齐）`[PROVEN]`：
  - truncation（截断）
  - integer conversion（整数转换）
  - width-height 来源
  - scale
  - cols-rows 方向
  - min（最小值）
  - 边界值

---

## 6. P4: 44B vertex 最终定性

- **unknown 2f (offset 20)**：无 `glVertexAttribPointer` 绑定 `[PROVEN]`
- **color 4f (offset 28)**：Python SPRITE shader 不声明 `a_color` / `v_color` `[PROVEN]`
- **NEKOPARA**：`color=0x808080FF`, `mesh_cc=True` `[PROVEN]`
- **正式标记**：**"已确认差异，但对当前 NEKOPARA renderer 无影响"** `[PROVEN]`
  - 分类：`DIFFERENCE-BUT-IRRELEVANT`
  - unknown 2f 为占位字段，color 4f 在主路径不被消费。

---

## 7. P5: 59 模型 multi-texture 最终验证

- **59/59 renderable** `[PROVEN]`
- **0 个 0 draw call 模型** `[PROVEN]`
- **3 个多纹理模型** `[PROVEN]`：
  - `azuki-casual` = 35 draw calls
  - `azuki-teenage` = 38 draw calls
  - `vanilla-koneko` = 34 draw calls
- **draw calls 范围**：26–45 `[PROVEN]`
- **非透明像素比**：17%–43% `[PROVEN]`

---

## 8. P6: renderer 剩余差异表 + 封版结论

| 项目 | 状态 | 说明 |
|------|------|------|
| PSB texture extraction | `MATCH` | 单纹理 + 多纹理都正确加载 |
| multi-texture | `MATCH` | 59/59 renderable `[PROVEN]` |
| V convention | `MATCH` | 统一 top-left convention |
| opacity | `DIFFERENCE-BUT-IRRELEVANT` | `u_spriteOpacity` vs `u_testAlpha` 语义偏差，NEKOPARA 无影响 |
| color | `DIFFERENCE-BUT-IRRELEVANT` | `u_filterColor` / `u_preFilterColor` / `u_meshColorControl` 语义偏差，NEKOPARA 无影响 |
| blend mode | `MATCH` | 3 个错误已修正（`bm=3/4/6`） |
| xr path | `MATCH` | NEKOPARA 全走 xr Bezier |
| mesh_bp | `MATCH` | 4×4 Bezier 控制点 |
| grid resolution | `MATCH` | 动态 grid 完全对齐 asm.js `bu`（18/18 一致） |
| meshDivision source | `MATCH` | 从 PSB 解析，值 16/20 |
| 44B vertex | `DIFFERENCE-BUT-IRRELEVANT` | unknown 2f 占位，color 4f 不消费 |
| fit_scale | `DIFFERENCE-BUT-VIEWER-LEVEL` | viewer 层差异，保留 |
| 59-model coverage | `MATCH` | 59/59 renderable |

### 封版结论

**✅ Renderer 可以封版** `[PROVEN]`

- 无 `NEEDS-FIX` 项。
- 剩余差异均为 `DIFFERENCE-BUT-IRRELEVANT` 或 `DIFFERENCE-BUT-VIEWER-LEVEL`。
- 无 renderer 回归。

---

## 9. 下一阶段 Runtime/Physics 明确入口

Runtime/Physics 阶段的明确切入点（按优先级）：

1. **hair/parts stiffness 真实值** — 物理刚度参数的真实来源与数值范围。
2. **`convolveCanvasMovementToPhysics`** — canvas movement 到 physics 的卷积传递机制。
3. **physics 其他应用机制** — 除 hair/parts 外的 physics 应用路径。
4. **`eyeControl`** — 眼球控制（注视/眨眼/瞳孔）。
5. **`eyebrowControl`** — 眉毛控制。
6. **`mouthControl`** — 嘴部控制（开合/表情）。

---

## 10. 本轮修改的文件列表

| 文件 | 审计项 | 修改内容 |
|------|--------|----------|
| `src/freemote/player/player.py` | P3 | grid 边界修复 + meshDivision 使用 |
| `src/freemote/format/psb/loader.py` | P2 | meshDivision 解析 |
| `src/freemote/format/psb/motion_painter.py` | P2 | RenderContext + DrawableResource `mesh_division` 字段 |
| `src/freemote/format/model.py` | P2 | SpriteInfo `mesh_division` 字段 |
| `tools/baseline_audit.py` | P0 | 基线审计脚本 |
| `tools/audit_all_models.py` | P5 | 59 模型审计脚本 |
| `render-alignment-implementation.md` | P0 | 数据冲突修正 |
| `tests/test_mesh_grid.py` | P3 | 测试更新 |

---

## 附录：审计任务追溯

| 审计项 | 任务 ID | 负责 agent | 状态 |
|--------|---------|-----------|------|
| P0 baseline 时间与版本校正 | 15 | BaselineAuditor | completed |
| P1+P4 shader uniform + 44B vertex | 16 | ShaderVertexAuditor | completed |
| P2+P3 meshDivision + dynamic grid | 17 | MeshDivGridAuditor | completed |
| P5 59 模型 multi-texture | 18 | ModelCoverageAuditor | completed |
| P6 renderer 封版检查 | 19 | SealCheckAgent | completed |
| 最终审计报告（本文件） | 20 | FinalAuditReport | in_progress → completed |

---

**报告生成时间**：2026-09-26
**审计阶段**：Renderer 收尾 → Runtime/Physics 启动前
**最终结论**：✅ Renderer 可以封版 `[PROVEN]`