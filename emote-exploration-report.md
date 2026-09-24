# E-mote / FreeMote 逆向研究独立审查报告

> **审查日期**：2026-09-24
> **审查性质**：独立逆向审查 + 研究方向重新定位（非 bug 修复）
> **审查者立场**：以刚加入项目的逆向研究员视角，不默认相信任何历史结论，所有判断标注证据等级
> **证据等级**：`[PROVEN]` 已被证据证明 ｜ `[LIKELY]` 高可信推测 ｜ `[UNVERIFIED]` 尚未验证 ｜ `[UNKNOWN]` 完全未知

---

## 1. 当前项目概况

### 1.1 项目目标

将 E-mote / FreeMote 的模型、动画、变形和渲染机制逆向出来，迁移到 Python/OpenGL 实现中，使实际模型尽可能接近原版 E-mote 表现。重点不是"能显示 PSB"，而是忠实复现原版运行机制。

### 1.2 整体架构（已确认）

```
official asm.js (FreeMoteDriver-format.js, 99629 行, Emscripten 编译产物)
        │
        ├── 逆向分析 (analysis/, 40+ 文档)
        │
        ▼
Python E-mote Core (src/freemote/, ~13000 行)
        │
        ├── format/psb/      PSB/PURE-PSB 解析 + motion_painter (3518 行)
        ├── player/          EmotePlayer + Transform + State (5375 行)
        ├── timeline/        Timeline 推进 + Easing + Variable
        ├── physics/         Hair/Parts/Bust 积分器 + Wind (691 行)
        ├── graphics/        Renderer ABC + Shader + Mask + Texture
        ├── platform/        OpenGL + GLFW + EmoteApp 主循环
        └── core/            Device + Runtime + Enums
        │
        ▼
OpenGL 后端 → 桌面程序 (examples/viewer.py)
```

### 1.3 证据链

项目拥有四条实现链，是逆向的核心依据：
- **asm.js**：`reference/FreeMoteDriver-format.js`（99629 行，Emscripten 编译的 C++ 产物，"use asm" 在 L4423-99341）。这是**真正的运行时标准**。
- **C# reference**：`reference/FreeMote-master/`（FreeMote 开源工具）。`StaticMotionPainterCore.cs` 是**静态分析工具**，不做帧间插值/mesh_bp/parameterize，**不是完整运行时标准** `[PROVEN]`。
- **JS 上层**：`reference/emoteplayer-format.js`（1267 行，ES6 封装）。
- **Python**：`src/freemote/`（当前实现）。

### 1.4 声称的状态 vs 实际状态

README 声称 Phase 3-E2 完成，1127 测试通过，"完整渲染流程验证"。但同一份 README 和 `examples/README.md` 明确写：

> "Mesh 顶点数据：当前 Phase 3-E0 仅恢复了 mesh 元信息，顶点/索引字节尚未完全解码，因此渲染窗口中 draw_elements 调用为 0。"

`[PROVEN]` **这是关键矛盾**：测试通过验证的是"流程不抛异常"和"数据结构正确"，**不是"像素与原版一致"**。全项目只有 1 个截图 `debug_frame.tga`，无视觉回归测试。

**修正**：经代码核查，`draw_elements` 调用数为 0 指的是 **part mesh 路径**（`_draw_player_mesh` 中基于 PSB mesh 顶点的路径）。实际渲染走的是 **sprite quad + 9×9 Bezier grid 路径**（`_draw_drawable_resources` → `_draw_drawable_resource`），这条路径会调用 `draw_elements` 并产生像素。所以渲染并非全黑，而是用"程序生成的 quad/grid 几何 + PSB 的 UV/transform/mesh_bp"绘制，**不是原版的 PSB mesh 顶点**。

---

## 2. 当前已经确认的成果（有证据支持）

以下机制有 asm.js 逆向证据 + Python 实现 + 测试验证：

| # | 机制 | 证据来源 | Python 实现 | 可信度 |
|---|------|---------|------------|--------|
| 1 | PSB/PURE-PSB 解析（header + section + body 类型系统 0x00-0x25，XOR 解密） | `psb-format.md` + loader.py 285+ 测试 | ✅ | `[PROVEN]` |
| 2 | Layer 树结构（children/layer 递归，exportSelf，frameList） | `binding-analysis.md` + asm.js `du` | ✅ | `[PROVEN]` |
| 3 | Transform 累积（calc_affine_matrix 与 asm.js `eu` 一致） | `asm-render-pipeline.md` §3 (L52619) | ✅ | `[PROVEN]` |
| 4 | build_context 三态继承模型（状态 1/2/3） | `asm-du-function-analysis.md` + `du` L52447-52523 | ✅ Task 40 | `[PROVEN]`（当前 59 模型全走状态 1，差异不触发） |
| 5 | 帧间插值 type 2 不插值 / type 3 插值 | `"Ft` L49564 + 29298 帧验证 | ✅ D1 修复 | `[PROVEN]` |
| 6 | angle 最短路径插值（360° 回绕） | `Ft` L49614-49617 + 8 单元测试 | ✅ D4 修复 | `[PROVEN]` |
| 7 | mesh_bp 帧间线性插值 | asm.js `au`/`bu` | ✅ | `[LIKELY]`（asm.js 中 mesh_bp 直接读取位置仍 `[UNKNOWN]`） |
| 8 | Easing 幂函数插值（convert_easing/apply_easing） | timeline §3.12 + 66 检查项 | ✅ | `[PROVEN]` |
| 9 | Timeline 推进（Al 三阶段算法） | timeline §3.11 + 13 检查项 | ✅ | `[PROVEN]` |
| 10 | Physics 积分器 zp（hair/parts 8步弹簧-阻尼-距离约束） | physics.md §6.2 + L65028-65195 | ✅ | `[PROVEN]` |
| 11 | Physics 积分器 Pn（bust 5步） | physics.md §6.2.2 + L28025-28092 | ✅ | `[PROVEN]` |
| 12 | 子步循环 + angle_clamp xo 角度限幅 | physics.md §6.2.1/§6.3 | ✅ | `[PROVEN]` |
| 13 | Wind 128 脉冲槽离散风场（go/io） | physics.md §5 + L28768-28889 | ✅ | `[PROVEN]`（RNG 用 random.random 占位） |
| 14 | Physics 参数从 PSB 读取（bustControl/hairControl/partsControl 的 friction/spring/gravity/param.p/pv/bp/length/var_lr/var_ud/baseLayer/scale） | player.py L961-1155 `_init_physics_from_metadata` | ✅ | `[PROVEN]`（**修正了 pure-psb-field-audit.md 的过时结论**） |
| 15 | Physics 应用到渲染（build_context 阶段绕 pivot 旋转，模拟 gu-pu 类型 3） | player.py L2081-2107 + `drawable-detach-root-cause.md` | ✅ Task 48 | `[LIKELY]`（asm.js gu-pu 有 9 个函数，Python 只实现旋转） |
| 16 | 渲染管线（DrawToTexture2 序列，Mask stencil kw/lw/mw/nw/ow） | rendering.md §3.10/§4.1.1 | ✅ | `[PROVEN]` |
| 17 | sprite quad 绘制 + 9×9 Bezier grid geometry warping | player.py `_draw_drawable_resource_grid` + `_bezier_patch_eval` | ✅ Task 106 | `[LIKELY]`（对应 asm.js `xr` 函数，但原版可能用 `zr`，见 §4） |
| 18 | 多纹理支持（tex/tex#NNN） | `root-cause.md` + 59 模型验证 | ✅ | `[PROVEN]` |
| 19 | inheritMask bitmask 解析 | `pure-psb-field-audit.md` + Task 33/40 | ✅ | `[PROVEN]` |
| 20 | blend mode（12 种）/ color（0xRRGGBBAA）/ mesh.cc | Task 40 + asm.js 逆向 | ✅ | `[PROVEN]` |

---

## 3. 当前高可信推测（有较强证据但未完全验证）

| # | 推测 | 证据 | 缺失的验证 |
|---|------|------|-----------|
| H1 | C# StaticMotionPainterCore 是静态渲染器，不是完整运行时标准 | `[PROVEN]` 它不做帧间插值/mesh_bp/parameterize | — |
| H2 | asm.js gu-pu 有 9 个函数（gu/hu/iu/ju/ku/lu/mu/nu/ou/pu），Python 只实现旋转（类型 3），其他 8 个（position/scale 修改等）未实现 | `asm-render-pipeline.md` §7.1 + physics-e2e-root-cause 根因 6 | 未确认其他 8 个函数在 NEKOPARA 模型中是否触发 |
| H3 | physics 输出写入 variable map（根因 3）可能未实现 | physics-e2e-root-cause.md 根因 3 | grep 未找到写入逻辑 |
| H4 | build_sprite_grid（sprite 回退路径）修复后可能未接入主渲染 | `_verify_render_fix_report.md`："渲染管线断层" | 主渲染走 DrawableResource 路径，sprite 路径是回退；两者都有 Bezier 实现 |
| H5 | hair/parts stiffness 用默认 0.003，不是模型实际值 | player.py L1140-1150 + TODO(UNKNOWN-A06) | runtime API 属性 8926/8934 未接上 |
| H6 | 所有 59 个 NEKOPARA 模型 inherit 全为默认 True，三态模型差异不触发 | master-conclusions.md + 59 模型扫描 | — |
| H7 | PSB 无 easing 表，asm.js 回退线性，当前 Python 线性插值与 asm.js 一致 | easing-survey.md（66 模型 0 个有非空 easing） | 未来遇到含 easing 表的模型时会错 |
| H8 | asm.js `xr`（双三次 Bezier）和 `zr`（4角双线性）是两个不同的 mesh 变形路径，Python 用 xr | player.py `_bezier_patch_eval` 注释 + asm-mesh-deform.md §3.2 | 未确认原版实际走 xr 还是 zr |

---

## 4. 当前完全未知的部分

### 4.1 数据层未知

| # | 未知项 | 影响 | 证据 |
|---|--------|------|------|
| U1 | **PSB mesh 顶点数据解码**（vertices=b""，需 section G 解码） | part mesh 路径完全跳过，无法用原版 mesh 顶点绘制 | loader.py L1508 `[PROVEN]` |
| U2 | **hair/parts stiffness 真实值**（来自 runtime API 属性 8926/8934） | 所有模型物理强度相同（0.003） | player.py L913 TODO(UNKNOWN-A06) |
| U3 | **stiffness 自动提取**（PhysicsData.raw_params 布局） | 同 U2 | UNKNOWN-A06 |
| U4 | **Wind RNG 种子来源**（用 random.random 占位） | 风场随机性可能与原版不同 | UNKNOWN-WIND-RNG |
| U5 | **asm.js 中 coord/opacity/angle/zoom 的帧间插值位置** | Python 插值位置可能不对 | asm-frame-interpolation.md §5.2 `[UNKNOWN]` |
| U6 | **asm.js 中 mesh_bp（32 float）的直接读取位置** | 不确定原版是否用 mesh_bp | asm-mesh-deform.md §11 `[UNKNOWN]` |
| U7 | **asm.js 原版实际走 xr（Bezier）还是 zr（双线性）** | Python 的 Bezier 可能与原版不一致 | 两个函数都存在于 asm.js |

### 4.2 控制字段未解析（影响动画行为）

`[PROVEN]` 以下字段 Python 完全未解析（pure-psb-field-audit.md 确认，且代码核查确认未读取）：

| # | 字段 | 含义 | 影响 |
|---|------|------|------|
| U8 | `metadata.eyeControl`（11 子字段） | 眨眼控制（blinkEnabled/blinkFrameCount/blinkInterval...） | 眨眼动画完全不工作 |
| U9 | `metadata.eyebrowControl`（5 子字段） | 眉毛控制 | 眉毛动画不工作 |
| U10 | `metadata.mouthControl`（4 子字段） | 嘴部控制（talkLabel...） | 嘴部动画不工作 |
| U11 | `metadata.charaProfile.pixelMarker`（22 标记点） | 角色 bounds/bust/eye/mouth/headA/headB/pantA/pantB... | 物理基准点可能错误 |
| U12 | `metadata.clampControl`（7 子字段） | 变量钳制（enabled/label/max/min/type/var_lr/var_ud） | 变量无钳制，可能超范围 |
| U13 | `metadata.selectorControl` | 选择器控制 | 选择器不工作 |
| U14 | `metadata.transitionControl` | 过渡控制 | 过渡不工作 |
| U15 | `metadata.mirrorControl` / `metadata.mirror` | 镜像控制 | 镜像不工作 |
| U16 | `metadata.loopControl` | 循环控制 | 循环行为可能错误 |
| U17 | `metadata.timelineControl`（115 项，含 variableList） | 时间线控制 | diff timeline 变量定义缺失 |
| U18 | `metadata.variableList` | 变量定义列表 | 变量初始化缺失 |
| U19 | `metadata.instantVariableList` | 即时变量 | 即时变量不工作 |
| U20 | `stereovisionProfile` / `stereovisionControl` | 立体视觉 | 立体视觉不工作 |

### 4.3 motion/layer/frame 级未解析字段

| # | 字段集 | 影响 |
|---|--------|------|
| U21 | `motion.bounds/lastTime/loopTime/layerIndexMap/priority/variable/tag/type` | motion 级元数据（loopTime 影响 Python 总返回 True） |
| U22 | `layer.coordinate/groundCorrection/joinTarget/meshCombine/meshDivision/meshSyncChildMask/meshTransform/shape/stencilCompositeMaskLayerList/stencilType/type`（13 字段） | layer 级辅助字段（meshDivision 影响网格细分度） |
| U23 | `frame.content.motion` | 嵌套 motion 参数 |
| U24 | `parameter.discretization` / `parameter.enabled` | 离散化参数 / Python 假设全启用 |

### 4.4 机制层未知

| # | 未知机制 | 影响 |
|---|---------|------|
| U25 | **gu-pu 的其他 8 个函数**（position/scale 修改等） | 物理只旋转，不修改 position/scale |
| U26 | **convolveCanvasMovementToPhysics**（canvas 位移驱动物理） | 角色移动不驱动物理 |
| U27 | **asm.js 顶点颜色**（44 字节含 RGBA，Python 16 字节无 color） | 顶点颜色缺失 |
| U28 | **asm.js 可变网格大小**（ratio 控制，Python 固定 9×9） | mesh 细分度不可调 |
| U29 | **Catmull-Rom 缓动**（`Iq` L69048） | 未来有 easing 表时会错 |
| U30 | **bit 22 透明传递层** | 未确认 NEKOPARA 是否使用 |
| U31 | **coord 旋转矩阵插值**（`It` L50271） | 未来有 coord easing 表时会错 |

---

## 5. 已解决问题与对应机制（真正的机制修复）

以下修复有明确证据表明**找到了底层机制**并正确实现：

| # | 修复 | 机制 | 证据 |
|---|------|------|------|
| M1 | **多纹理 icon 表**（Task 6） | `_build_icon_table` 遍历所有 tex 键 | 3 个模型 drawable 0→34/37/33，59 模型无回归 `[PROVEN]` |
| M2 | **type 2 帧不插值**（D1） | asm.js `Ft` 只对 type 3 插值 | 29298 帧验证 `[PROVEN]` |
| M3 | **angle 最短路径**（D4） | asm.js `Ft` L49614-49617 的 360° 回绕 | 8 单元测试 `[PROVEN]` |
| M4 | **build_context 总是 parent*local**（Task 84） | asm.js `du` 总是执行矩阵乘法 | `asm-render-pipeline.md` §4 `[PROVEN]` |
| M5 | **inheritMask bitmask 解析 + 三态模型**（Task 33/40） | asm.js `du` 三态继承 | `asm-du-function-analysis.md` `[PROVEN]` |
| M6 | **Physics 角度索引**（根因 1） | output_angles[2]（y方向）vs [1]（x方向恒零） | physics-e2e-root-cause.md `[PROVEN]` |
| M7 | **Physics 单位转换**（根因 2） | E-mote 内部单位 π/80 → 弧度 | physics-e2e-root-cause.md `[PROVEN]` |
| M8 | **Physics 变量注册**（根因 5） | var_lr/var_ud 注册到 variable map | physics-e2e-root-cause.md `[PROVEN]` |
| M9 | **Physics 参数从 PSB 读取**（Task 22） | bustControl/hairControl/partsControl → friction/spring/gravity/param/baseLayer/var_lr/var_ud | player.py L961-1155 `[PROVEN]` |
| M10 | **blend mode / color / mesh.cc**（Task 40） | asm.js 12 种 blend mode + 0xRRGGBBAA + ColorControl | Task 40 `[PROVEN]` |
| M11 | **Easing table 解析**（Task 33） | cubic spline（非 Catmull-Rom），12 字节 stride | Task 31-34 `[PROVEN]` |
| M12 | **9×9 Bezier geometry warping**（Task 106） | asm.js `xr` 函数双三次 Bezier 曲面 | player.py `_bezier_patch_eval` `[LIKELY]`（原版可能用 `zr`） |

---

## 6. 可能只是补丁的修复（针对症状而非根因）

以下修复**可能没有触及底层机制**，需谨慎对待：

| # | 修复 | 怀疑理由 | 证据 |
|---|------|---------|------|
| P1 | **mouth3~mouth26**（24 个版本迭代） | 24 次反复调查同一问题，若找到机制不需要如此多迭代 | `_diag_mouth3.py`~`_diag_mouth26.py` 文件名 `[LIKELY]` |
| P2 | **coconut-casual 耳朵脱离** | 诊断结论是"头部 sprite 是包含头部和身体的大 sprite，origin 在身体中心"——这是特定模型结构描述，非通用机制修复 | `_diag_ear_detachment_report.md` `[LIKELY]` |
| P3 | **build_sprite_grid 修复但未接入主管线** | `_verify_render_fix_report.md` 明确写"渲染管线断层" | `[PROVEN]` 存在断层（但主路径 DrawableResource 有独立 Bezier 实现） |
| P4 | **hair/parts stiffness 默认 0.003** | 用占位值而非模型实际值，在此基础上修物理应用方式，不同模型差异不大 | player.py L1140 `[PROVEN]` |
| P5 | **Physics 初始扰动移除**（"之前设 velocity.x=1.5 导致偏移"） | 注释说移除了初始扰动，但这可能是为了消除可见偏移而非修复机制 | player.py L1040-1042 `[UNVERIFIED]` |
| P6 | **fit_scale 用 bounds 计算但不用 bounds center 居中** | 注释说"bounds 包含极端元素如髪揺れ，center 偏移会导致角色整体往右偏移"——这是为避免可见偏移而做的调整 | player.py L2196-2199 `[LIKELY]` |

**关键模式**：`_diag_bust1~4`、`_diag_ear_*`、`_diag_mouth3~26`、`_diag_head_rotate`、`_diag_transform_compare` 等大量针对特定模型/部件的诊断脚本，暗示历史上存在"某模型出现问题 → 修改参数/条件 → 该模型改善 → 其他模型无明显改善"的循环。

---

## 7. 原版 vs Python 差异表

| 模块 | 原版推测/确认行为 | Python 当前行为 | 差异 | 影响 |
|------|---------|-----------|------|------|
| **PSB 解析** | header + section + body 类型系统 | ✅ 完整实现 0x00-0x25 | 无（明文文件） | 无 |
| **PURE-PSB** | XOR 解密 + section 偏移 | ✅ 实现明文，加密文件 TODO | 加密文件不支持 | 低（NEKOPARA 用明文） |
| **hierarchy** | children/layer 递归 + exportSelf | ✅ 一致 | 无 | 无 |
| **transform** | calc_affine_matrix + build_context 三态 | ✅ 三态实现，当前全走状态 1 | 无（59 模型不触发） | 无 |
| **motion** | 嵌套 motion via src+icon | ✅ 一致 | 无 | 无 |
| **interpolation** | type 2 不插值，type 3 插值，angle 最短路径 | ✅ 一致 | Catmull-Rom 缓动缺失 | 低（PSB 无 easing 表） |
| **mesh 变形** | asm.js `zr`（4角双线性）或 `xr`（16控制点双三次 Bezier） | 9×9 Bezier geometry warping（`xr`） | 数学方法可能不同；网格固定 9×9 vs 可变；顶点无颜色 | `[UNKNOWN]` 原版走哪个 |
| **mesh 顶点** | PSB section G 解码的顶点数据 | ❌ vertices=b""（未解码） | part mesh 路径跳过 | **高**（无法用原版 mesh 绘制） |
| **physics 积分** | zp/Pn 弹簧-阻尼 | ✅ 一致 | 无 | 无 |
| **physics 参数** | bustControl/hairControl/partsControl 从 PSB 读取 | ✅ 读取（friction/spring/gravity/param/baseLayer/var_lr/var_ud） | hair/parts stiffness 用默认 0.003 | **中**（所有模型强度相同） |
| **physics 应用** | gu-pu 9 个函数（旋转+position+scale 修改） | 仅旋转（类型 3），在 build_context 阶段 | 其他 8 个函数未实现 | `[UNVERIFIED]`（未确认是否触发） |
| **physics 输出** | 写入 variable map + part transform | 仅 part transform（build_context 旋转） | variable map 未写入 | 低（不影响渲染） |
| **timing** | Al 三阶段 + 子步循环 | ✅ 一致 | 无 | 无 |
| **render traversal** | Ji 按 z 排序遍历 | ✅ 按 z 排序 | 无 | 无 |
| **texture** | 多纹理 tex/tex#NNN | ✅ 一致 | 无 | 无 |
| **UV** | kx 线性插值 u/v | ✅ 均匀映射到 uv_coords | 无 | 无 |
| **alpha** | 预乘 alpha + 近全透 discard | ✅ shader 实现 | 无 | 无 |
| **mask** | kw/lw/mw/nw/ow stencil 操作 | ✅ 一致 | 无 | 无 |
| **blend mode** | 12 种 GL blend 函数 | ✅ 一致 | 无 | 无 |
| **color** | 0xRRGGBBAA / 255.0 归一化 | ✅ 一致 | 无 | 无 |
| **clipping** | bit 22 透明传递层 | ❌ 不检查 | `[UNKNOWN]` 是否触发 | 低 |
| **runtime state** | eyeControl/eyebrowControl/mouthControl/clampControl/selectorControl/transitionControl/mirrorControl/loopControl/timelineControl/variableList/instantVariableList | ❌ 全部未解析 | **大量控制机制缺失** | **高** |
| **convolveCanvasMovement** | canvas 位移驱动物理 | ❌ 未实现 | 角色移动不驱动物理 | 中 |
| **stereovision** | stereovisionProfile/Control | ❌ 未实现 | 立体视觉不工作 | 低 |

---

## 8. 当前瓶颈分析

**核心问题**：为什么出现"修了很多东西，但整体效果提升不明显"？

### 8.1 渲染几何来源错误（最可能根因）

`[PROVEN]` Python 渲染用的是**程序生成的 quad/grid 几何**，不是原版 PSB mesh 顶点。PSB mesh 顶点数据未解码（`vertices=b""`），part mesh 路径跳过。当前绘制的是 sprite quad（4 顶点）和 9×9 Bezier grid（81 顶点），用 PSB 的 UV/transform/mesh_bp，但**不是原版的 mesh 顶点数据**。

在这个基础上调整 transform/interpolation/physics，效果有限——因为几何本身就不是原版的。

### 8.2 物理参数占位（次要根因）

`[PROVEN]` hair/parts stiffness 用默认 0.003，不是模型实际值。所有模型的头发/部件摆动强度相同。在这个基础上修物理应用方式，不同模型表现差异不大。

### 8.3 大量控制机制缺失（深层根因）

`[PROVEN]` eyeControl/mouthControl/eyebrowControl/clampControl/selectorControl/transitionControl/mirrorControl/loopControl/timelineControl/variableList/instantVariableList 全部未解析。这些控制眨眼、嘴部、眉毛、变量钳制、选择器、过渡、镜像、循环等**动态行为**。修了 transform/interpolation/physics，但这些控制层缺失，整体动画行为仍与原版有差异。

### 8.4 测试通过 ≠ 渲染正确（验证缺口）

`[PROVEN]` 1127 测试验证的是"流程不抛异常"和"数据结构正确"，不是"像素与原版一致"。全项目只有 1 个截图，无视觉回归测试。没有与原版 E-mote 运行时的截图对比，无法量化"接近程度"。

### 8.5 文档与代码不一致（认知偏差）

`[PROVEN]` 多份关键分析文档已过时：
- `asm-mesh-deform.md` 描述 Python 为"4×4 texture warping"，实际代码是"9×9 Bezier geometry warping"（Task 106）。
- `pure-psb-field-audit.md` 标记 hairControl/bustControl/partsControl 为 ❌，实际已读取（`_init_physics_from_metadata`）。
- `physics-e2e-root-cause.md` 的根因 1/2/5 已修复，但文档仍以"修复方案"形式存在。

**影响**：基于过时文档的修复方向可能错误。新加入的研究员若先读文档会被误导。

### 8.6 asm.js 关键插值位置未确认

`[UNKNOWN]` asm.js 中 coord/opacity/angle/zoom 的帧间插值位置未确认（分散在 vu/au/bu/du/Hl/Il 中）。Python 的插值实现可能位置不对，但无法验证。

### 8.7 历史修复模式

`[LIKELY]` 历史上存在大量针对特定模型/部件的反复调查（mouth 24 次迭代、bust 4 次、ear 多次）。这种模式暗示部分修复是"针对症状调参"而非"找到机制"。`_verify_render_fix_report.md` 明确记录了"修了函数但没接入主管线"的情况。

---

## 9. 最大研究盲区（我们不知道自己不知道什么）

### 盲区 1：PSB mesh 顶点数据解码

```
未知问题：PSB section G 的 mesh 顶点数据如何解码？
当前证据：loader.py L1508 vertices=b""，注释"需 section G 解码"
为什么重要：这是原版 mesh 绘制的数据基础，不解码则永远用程序生成几何
可能影响：所有 mesh-based 部件的渲染几何与原版不一致
怎么验证：逆向 asm.js 中 mesh 顶点加载函数，对比 PSB section G 字节布局
```

### 盲区 2：原版用 xr 还是 zr

```
未知问题：asm.js 运行时实际走 xr（双三次 Bezier）还是 zr（4角双线性）？
当前证据：两个函数都存在于 asm.js；Python 用 xr；asm-mesh-deform.md 分析的是 zr
为什么重要：若原版用 zr，Python 的 Bezier 实现虽然更精细但与原版不一致
可能影响：mesh 变形的形状与原版有差异
怎么验证：在 asm.js 中追踪 hu 函数调用 xr 还是 zr 的条件分支
```

### 盲区 3：gu-pu 其他 8 个函数的触发条件

```
未知问题：asm.js gu-pu 的 9 个函数中，Python 只实现旋转（类型 3），其他 8 个何时触发？
当前证据：asm-render-pipeline.md §7.1 列出 9 个函数及 flags 条件
为什么重要：若其他函数在 NEKOPARA 中触发，Python 缺少 position/scale 物理修改
可能影响：物理摆动只旋转不位移/缩放，与原版不符
怎么验证：扫描 59 个模型的 part flags，确认 gu/hu/iu/ju/ku/lu/mu/nu/ou/pu 的触发条件
```

### 盲区 4：eyeControl/mouthControl 的运行时机制

```
未知问题：眨眼/嘴部控制如何在运行时驱动动画？
当前证据：pure-psb-field-audit.md 确认字段存在且 asm.js 读取，Python 不读取
为什么重要：这是角色"活起来"的关键动态机制
可能影响：角色不会眨眼/说话，表情呆滞
怎么验证：逆向 asm.js 中 eyeControl/mouthControl 的使用路径
```

### 盲区 5：coord/opacity/angle/zoom 帧间插值的真实位置

```
未知问题：asm.js 中这些参数的帧间插值发生在哪个函数？
当前证据：asm-frame-interpolation.md §5.2 标记 [UNKNOWN]
为什么重要：Python 的插值位置可能不对
可能影响：动画曲线形状与原版有差异
怎么验证：在 asm.js vu/au/bu/du 中追踪这些字段的读写
```

### 盲区 6：hair/parts stiffness 的真实来源

```
未知问题：stiffness 来自 runtime API 属性 8926/8934，这些属性何时被设置？
当前证据：physics.py 注释说来自 runtime API，player.py 用默认 0.003
为什么重要：所有模型物理强度相同，无法匹配原版
可能影响：头发/部件摆动幅度与原版不符
怎么验证：逆向 asm.js 中属性 8926/8934 的设置时机（Initialize？SetVariable？）
```

### 盲区 7：convolveCanvasMovementToPhysics

```
未知问题：canvas 位移如何驱动物理？
当前证据：physics.md §1.3 描述了机制，Python 未实现
为什么重要：角色移动时头发/胸部应有惯性摆动
可能影响：移动角色时物理不响应
怎么验证：在 emoteplayer-format.js L862-875 确认机制，对比 Python update_animation
```

### 盲区 8：stiffness 自动提取（raw_params 布局）

```
未知问题：PhysicsData.raw_params 的字节布局？
当前证据：UNKNOWN-A06，model.py raw_params: bytes = b""
为什么重要：bust stiffness 从 spring 读取，但 hair/parts stiffness 可能在 raw_params 中
可能影响：同盲区 6
怎么验证：逆向 asm.js 中 raw_params 的解析路径
```

---

## 10. 下一阶段研究方向

### 方向 1：PSB mesh 顶点数据解码（最高优先级）

- **研究目标**：实现 PSB section G 的 mesh 顶点/索引解码，使 part mesh 路径能使用原版几何
- **当前证据**：loader.py L1508 留空 + 注释"需 section G 解码"；ref1_id → section E → G 间接寻址未实现
- **入口**：`src/freemote/format/psb/loader.py` 的 mesh 解析部分 + asm.js 中 mesh 顶点加载函数
- **验证方法**：解码后对比 C# FreeMote Viewer 的 mesh 顶点数；渲染截图对比
- **如果确认**：part mesh 路径可用原版几何绘制，渲染保真度大幅提升

### 方向 2：确认原版 mesh 变形路径（xr vs zr）

- **研究目标**：确认 asm.js 运行时实际走 xr（Bezier）还是 zr（双线性），以及触发条件
- **当前证据**：两个函数都存在；Python 用 xr；asm-mesh-deform.md 分析 zr
- **入口**：asm.js `hu` 函数（L53301）中调用 xr/zr 的分支条件
- **验证方法**：在 asm.js 中追踪 hu → xr/zr 的调用路径，确认条件
- **如果确认**：若原版用 zr，需将 Python 改为 4 角双线性；若用 xr，当前 Bezier 实现正确

### 方向 3：eyeControl / mouthControl / eyebrowControl 运行时机制

- **研究目标**：实现眨眼/嘴部/眉毛控制，使角色有基础表情动态
- **当前证据**：pure-psb-field-audit.md 确认字段存在 + asm.js 读取；Python 完全未解析
- **入口**：asm.js 中 eyeControl/mouthControl 的使用路径 + PSB metadata 解析
- **验证方法**：实现后观察角色是否眨眼/说话
- **如果确认**：角色表情动态恢复，"活起来"

### 方向 4：hair/parts stiffness 真实值获取

- **研究目标**：从正确来源获取 hair/parts stiffness，替代默认 0.003
- **当前证据**：physics.py 注释说来自 runtime API 属性 8926/8934；UNKNOWN-A06 raw_params 布局未解析
- **入口**：asm.js 中属性 8926/8934 的设置时机 + PhysicsData.raw_params 布局逆向
- **验证方法**：获取后对比不同模型摆动幅度差异
- **如果确认**：不同模型物理强度匹配原版

### 方向 5：gu-pu 其他 8 个函数的触发条件与实现

- **研究目标**：确认 gu/hu/iu/ju/ku/lu/mu/nu/ou/pu 中哪些在 NEKOPARA 触发，实现缺失的
- **当前证据**：asm-render-pipeline.md §7.1 列出 9 个函数及 flags 条件
- **入口**：扫描 59 个模型的 part flags（`flags & 512/32/2/8/64/16/1024`）
- **验证方法**：确认触发条件后实现，对比物理摆动行为
- **如果确认**：物理应用更完整（position/scale 修改），匹配原版

### 方向 6：建立视觉回归测试

- **研究目标**：建立与原版 E-mote 运行时的截图对比机制
- **当前证据**：全项目只有 1 个截图，无视觉回归
- **入口**：用原版 E-mote WebGL 运行 NEKOPARA 模型截图，Python 渲染同模型截图，像素对比
- **验证方法**：SSIM / 像素差异量化
- **如果确认**：能量化"接近程度"，避免"修了和没修差不多"无法判断

### 方向 7：coord/opacity/angle/zoom 帧间插值位置确认

- **研究目标**：确认 asm.js 中这些参数的帧间插值发生在哪个函数
- **当前证据**：asm-frame-interpolation.md §5.2 [UNKNOWN]
- **入口**：asm.js vu/au/bu/du 中追踪这些字段的读写
- **验证方法**：动态 trace asm.js 运行时，观察插值发生位置
- **如果确认**：修正 Python 插值位置，动画曲线形状匹配原版

---

## 11. Agent Team 拆分建议

基于实际观察到的项目结构和瓶颈，建议以下拆分（非机械套用用户建议）：

```
Agent A：PSB mesh 顶点解码
  - 目标：实现 section G 解码
  - 入口：loader.py mesh 解析 + asm.js mesh 加载函数
  - 产出：可用的 mesh 顶点数据

Agent B：asm.js mesh 变形路径确认（xr vs zr）
  - 目标：确认原版实际走 xr 还是 zr
  - 入口：asm.js hu 函数分支条件
  - 产出：明确的变形路径 + 触发条件

Agent C：控制字段运行时机制（eye/mouth/eyebrow/clamp/selector）
  - 目标：逆向 asm.js 中控制字段使用路径
  - 入口：asm.js eyeControl/mouthControl 使用点
  - 产出：控制机制文档 + Python 实现方案

Agent D：physics 完整性（stiffness 来源 + gu-pu 其他函数 + convolveCanvas）
  - 目标：补全 physics 缺失机制
  - 入口：asm.js 属性 8926/8934 + gu-pu 9 函数 + convolveCanvasMovement
  - 产出：完整 physics 数据流

Agent E：视觉回归测试基础设施
  - 目标：建立原版 vs Python 截图对比
  - 入口：原版 E-mote WebGL + Python viewer
  - 产出：量化差异基线

Agent F：文档同步与证据链审计
  - 目标：修正过时文档（asm-mesh-deform.md / pure-psb-field-audit.md / physics-e2e-root-cause.md）
  - 入口：对比文档结论与当前代码
  - 产出：准确的现状文档
```

**关键依赖**：Agent A（mesh 解码）是其他所有渲染相关工作的基础。Agent E（视觉回归）是验证所有修复效果的前提。建议优先启动 A + E。

---

## 12. 给下一位 AI 的交接摘要

我们在做 E-mote/FreeMote 的逆向工程，目标是把原版运行机制（PSB 数据、hierarchy、transform、animation、interpolation、mesh deform、physics、render pipeline）忠实迁移到 Python/OpenGL。

**已经逆向到的程度**：数据解析（PSB/PURE-PSB）完整；layer 树 + transform 累积（含三态继承）完整；帧间插值（type 2/3 + angle 最短路径）完整；timeline 推进（Al 三阶段）完整；physics 积分器（zp/Pn 弹簧-阻尼 + Wind）完整；physics 参数从 PSB 读取（bustControl/hairControl/partsControl）已实现；渲染管线（DrawToTexture2 + Mask + sprite quad + 9×9 Bezier grid）完整。asm.js 逆向覆盖了 Ft/du/Ji/Iq/It/Rq/Hl/Il/Jl/Ij/Zl 等核心函数。

**比较可靠的结论**：calc_affine_matrix 与 asm.js eu 完全一致；build_context 三态模型已实现（当前 59 模型全走状态 1）；C# StaticMotionPainterCore 是静态渲染器不是运行时标准；physics 积分算法与 asm.js 一致；mesh 变形已改为 9×9 Bezier geometry warping（对应 asm.js xr 函数）。

**可能是错的结论**：pure-psb-field-audit.md 标记 hairControl 等为 ❌，实际已读取（文档过时）；asm-mesh-deform.md 说 Python 是 texture warping，实际已是 geometry warping（文档过时）；physics-e2e-root-cause.md 的根因 1/2/5 已修复但文档仍以"方案"形式存在。原版 mesh 变形走 xr 还是 zr 未确认——Python 用 xr，但 asm-mesh-deform.md 分析的是 zr。

**当前最大的瓶颈**：(1) PSB mesh 顶点数据未解码（vertices=b""），渲染用程序生成几何而非原版 mesh；(2) hair/parts stiffness 用默认 0.003，所有模型物理强度相同；(3) eyeControl/mouthControl/eyebrowControl 等大量控制字段未解析，角色不会眨眼/说话；(4) 无视觉回归测试，无法量化"接近程度"；(5) 多份关键文档过时，基于过时文档的修复方向可能错误。

**当前最大的未知机制**：PSB section G 的 mesh 顶点解码（这是原版 mesh 绘制的数据基础）；asm.js 原版走 xr 还是 zr（决定 mesh 变形数学方法）；gu-pu 的其他 8 个物理函数何时触发（决定物理应用完整性）。

**下一步最值得调查**：(1) 实现 PSB mesh 顶点解码——这是所有渲染工作的基础；(2) 确认原版 mesh 变形路径 xr vs zr；(3) 建立视觉回归测试——否则永远无法判断"修了有没有用"；(4) 逆向 eyeControl/mouthControl 运行时机制——这是角色"活起来"的关键。

**重要原则**：不要默认相信历史分析文档的结论——多份已过时。不要为了"让当前模型看起来更像"而加 patch——需要找到缺失机制。1127 测试通过 ≠ 渲染正确——测试验证的是流程不抛异常，不是像素一致。