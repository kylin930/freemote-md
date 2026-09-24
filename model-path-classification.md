# NEKOPARA 59 模型路径分类表 + gu-pu 触发条件扫描报告

> 扫描时间：2026-09-24
> 扫描对象：`reference/nekopara-model/*.pure.psb`（59 个模型）
> 扫描脚本：`tools/scan_model_paths.py`
> 原始数据：`analysis/.model-path-scan.json`

---

## 1. 扫描方法

### 1.1 脚本

**路径**：`tools/scan_model_paths.py`

**扫描逻辑**：

1. 用 `src/freemote/format/psb/loader.py::PSBLoader` 加载每个 `.pure.psb` 文件
2. 从 `root_value['metadata']` 读取 physics 配置（`hairControl` / `bustControl` / `partsControl`）
3. 递归遍历 `root_value['object'][chara]['motion'][motion]['layer']` 树，对每个 layer 收集：
   - `layer.type`（layer type，存入 part+24）
   - `frameList[*].content.mesh`：mesh 字段
   - `mesh.bp`：Bezier 控制点（32 个 float = 4×4 网格 UV，或 bool 标志）
   - `mesh.cc`：mesh 顶点坐标（list 或 bool 标志）
   - `content.src`：texture / nested motion 引用
   - `content.icon`：drawable icon 类型
4. 计算 **model flags** = `OR(1 << layer_type)` for all layer types
   - 依据：asm.js `ut` 函数 L52441 `c[motion+592] |= 1 << layer_type`
   - gu-pu 的 flags = `c[b+592>>2]`（b 是 motion 结构，见 `analysis/asm-render-pipeline.md` §7.1）
5. 根据 flags 判断 gu-pu 触发（见 §5）
6. 五类分类（见 §3）

### 1.2 关键判定规则

| 字段 | 判定 |
|------|------|
| `drawable_type` | `mesh.cc` 非空 → mesh；`mesh.bp` 非空 → bezier；有 src → sprite；否则 none |
| `deform_path` | 有 `mesh.bp` → xr (Bezier)；有 `mesh.cc` → mesh (PSB)；否则 simple/zr |
| `has_section_G` | PSB 总有 section G（ref1 目标数据）；vf>3 时还有 H-K |
| `has_physics` | `hairControl`/`bustControl`/`partsControl` 任一非空 |

### 1.3 mesh.bp / mesh.cc 的两种形态

扫描中发现 PSB 的 `mesh.bp` 和 `mesh.cc` 字段有**两种形态**：

- **bool 形态**（`True`/`False`）：标志位，表示"使用默认网格"或"不使用"
- **list 形态**（`bp` = 32 个 float）：真正的 Bezier 控制点数据

以 `azuki-casual` 为例（全量 801 个 mesh，depth 限制下采样）：
- `bp`: 414 个 bool + 387 个 list(32)
- `cc`: **全部 801 个都是 bool**（无真正的顶点坐标数据）

**结论**：NEKOPARA 模型**不使用 PSB mesh 顶点变形**（mesh.cc 全是标志），只使用 Bezier 网格变形（mesh.bp）。

---

## 2. 59 模型分类表

> 完整 59 模型扫描结果。所有模型**完全同构**：flags=`0x0000100F`，layer_types=`[0,1,2,3,12]`，physics=`4/2/2`，类别=`A/D/E`。
>
> 列说明：drawable=drawable 类型；mesh.cc=是否有真正 mesh 顶点；mesh.bp=是否有 Bezier 控制点；sec G=section G；deform=deform path；physics(h/b/p)=hairControl/bustControl/partsControl 数量；flags=model flags；类别=A/B/C/D/E；vf=version_field

| # | 模型 | drawable | mesh.cc | mesh.bp | sec G | deform | physics(h/b/p) | flags | 类别 | vf | mesh层 | layer数 |
|---|------|----------|---------|---------|-------|--------|----------------|------|------|----|--------|---------|
| 1 | azuki-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1380 | 459 |
| 2 | azuki-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1184 | 411 |
| 3 | azuki-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1423 | 471 |
| 4 | azuki-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1476 | 482 |
| 5 | azuki-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1500 | 491 |
| 6 | azuki-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1543 | 503 |
| 7 | azuki-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1434 | 474 |
| 8 | azuki-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1378 | 457 |
| 9 | chocola-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 3 | 1249 | 419 |
| 10 | chocola-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1321 | 439 |
| 11 | chocola-koneko | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1491 | 489 |
| 12 | chocola-lolita | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 3 | 1293 | 431 |
| 13 | chocola-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1417 | 469 |
| 14 | chocola-pajama | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1138 | 387 |
| 15 | chocola-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1722 | 558 |
| 16 | chocola-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1395 | 463 |
| 17 | chocola-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1429 | 473 |
| 18 | chocola-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1469 | 484 |
| 19 | chocola-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1314 | 437 |
| 20 | cinnamon-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1357 | 453 |
| 21 | cinnamon-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1122 | 385 |
| 22 | cinnamon-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1404 | 466 |
| 23 | cinnamon-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1530 | 499 |
| 24 | cinnamon-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1378 | 457 |
| 25 | cinnamon-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1378 | 457 |
| 26 | cinnamon-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1415 | 468 |
| 27 | cinnamon-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1483 | 486 |
| 28 | coconut-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 3 | 1326 | 440 |
| 29 | coconut-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1097 | 377 |
| 30 | coconut-koneko | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1477 | 485 |
| 31 | coconut-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1395 | 463 |
| 32 | coconut-pajama | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1043 | 339 |
| 33 | coconut-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1473 | 484 |
| 34 | coconut-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1133 | 389 |
| 35 | coconut-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1231 | 413 |
| 36 | coconut-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1406 | 466 |
| 37 | coconut-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1224 | 411 |
| 38 | fraise-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 2722 | 907 |
| 39 | maple-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1081 | 352 |
| 40 | maple-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1034 | 336 |
| 41 | maple-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1362 | 453 |
| 42 | maple-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1449 | 477 |
| 43 | maple-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1543 | 523 |
| 44 | maple-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1457 | 480 |
| 45 | maple-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1437 | 474 |
| 46 | maple-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1222 | 410 |
| 47 | milk-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1078 | 351 |
| 48 | milk-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 2276 | 750 |
| 49 | vanilla-casual | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 3 | 1289 | 429 |
| 50 | vanilla-dress | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1263 | 421 |
| 51 | vanilla-koneko | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1395 | 463 |
| 52 | vanilla-lolita | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 3 | 1335 | 443 |
| 53 | vanilla-maid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1448 | 478 |
| 54 | vanilla-pajama | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1138 | 387 |
| 55 | vanilla-santa | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1599 | 516 |
| 56 | vanilla-teenage | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1395 | 463 |
| 57 | vanilla-winter | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1400 | 464 |
| 58 | vanilla-wintermaid | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1459 | 481 |
| 59 | vanilla-yukata | bezier | N | Y | Y | xr (Bezier) | 4/2/2 | 0x0000100F | A/D/E | 4 | 1401 | 464 |

### 2.1 按角色分组统计

| 角色 | 模型数 | mesh 层范围 | 备注 |
|------|--------|-------------|------|
| azuki | 8 | 1184–1543 | |
| chocola | 11 | 1138–1722 | 含 2 个 vf=3 (casual/lolita) |
| cinnamon | 8 | 1122–1530 | |
| coconut | 10 | 1043–1477 | 含 1 个 vf=3 (casual) |
| fraise | 1 | 2722 | 最大单模型 |
| maple | 8 | 1034–1543 | |
| milk | 2 | 1078–2276 | milk-winter 异常大 |
| vanilla | 11 | 1138–1599 | 含 2 个 vf=3 (casual/lolita) |

### 2.2 version_field 分布

| version_field | header_size | 模型数 | 模型 |
|---------------|-------------|--------|------|
| 4 | 56 | 54 | 绝大多数（含 section H-K） |
| 3 | 44 | 5 | chocola-casual, chocola-lolita, coconut-casual, vanilla-casual, vanilla-lolita |

所有模型 `is_encrypted=False`（`.pure.psb` 是明文）。

---

## 3. 五类分类

### 3.1 分类定义

| 类别 | 定义 | 判定条件 |
|------|------|----------|
| **A** | 完全依赖 sprite / Bezier（无 mesh.cc，有 mesh.bp，走 xr 路径） | `mesh_bp > 0 and mesh_cc == 0` |
| **B** | sprite + mesh deform（有 mesh.bp 且有部分 mesh.cc） | `mesh_bp > 0 and mesh_cc > 0` |
| **C** | 真正使用 PSB mesh（有完整 mesh 顶点数据 mesh.cc） | `mesh_cc > 0` |
| **D** | physics 明显参与（hairControl/bustControl/partsControl 任一非空） | `has_physics` |
| **E** | 存在特殊 flags / 特殊路径（gu/ju/lu/nu/ou/pu 任一触发） | gu-pu 特殊触发 |

### 3.2 分类结果

| 类别 | 模型数 | 模型列表 |
|------|--------|----------|
| **A** | **59** | 全部 59 个模型 |
| **B** | **0** | — |
| **C** | **0** | — |
| **D** | **59** | 全部 59 个模型 |
| **E** | **59** | 全部 59 个模型 |

**关键结论**：

- **所有 59 个模型同属 A/D/E 三类**，无 B/C 类模型
- NEKOPARA **不使用 PSB mesh 顶点变形**（mesh.cc 全是 bool 标志，无真正顶点数据）
- NEKOPARA **统一使用 Bezier 网格变形**（mesh.bp，走 xr 路径）
- 所有模型都有完整的 physics 配置（hair=4, bust=2, parts=2）
- 所有模型都触发特殊 gu-pu 路径（lu/mu 因 layer type 1/3 存在而触发）

### 3.3 layer type 分布（跨所有 59 模型）

| layer type | 含义 | 总数 | 对应 gu-pu |
|-----------|------|------|-----------|
| 0 | 普通 layer（有 sprite 引用） | 20501 | — |
| 1 | shape layer | 463 | **lu** (flags & 2) |
| 2 | 未文档化 layer（可能是中间层/容器） | 3877 | — |
| 3 | 嵌套 motion | 2005 | **mu** (flags & 8) |
| 12 | 某种 layer（有 sprite 引用） | 354 | — |

**注意**：layer type 2 在 `analysis/asm-children-layer-src-semantics.md` §9.1 中未明确列出，但每个模型平均 65 个。它不对应任何 gu-pu 触发位（bit 2 不在 gu-pu 条件中），可能是某种容器/分组层。

---

## 4. 代表模型

由于所有 59 个模型完全同构（flags/physics/类别一致），代表模型按**复杂度梯度**选择，覆盖最小/最大/中等规模：

| 类别 | 代表模型 | 选择理由 | 特征 |
|------|----------|----------|------|
| **A**（最小） | `maple-dress` | mesh 层最少（1034），layer 最少（336） | 最简单的 Bezier 模型，快速回归测试 |
| **A**（最大） | `fraise-maid` | mesh 层最多（2722），layer 最多（907） | 最复杂的 Bezier 模型，压力测试 |
| **A**（中等） | `azuki-casual` | mesh=1380，layer=459，接近均值 | 典型规模，主回归测试 |
| **D** | `chocola-maid` | physics=4/2/2（与所有模型一致） | physics 路径代表 |
| **E**（vf=3） | `chocola-casual` | version_field=3，header=44 | 旧版本格式代表（无 section H-K） |
| **E**（vf=4） | `milk-winter` | version_field=4，mesh=2276（第二大） | 新版本 + 大规模 + winter 特殊路径 |
| **E**（特殊） | `vanilla-santa` | santa 服装，mesh=1599 | 特殊服装 + particle 潜在路径 |

### 4.1 推荐视觉回归测试集

**最小集（3 个）**：`maple-dress`（最小）、`azuki-casual`（典型）、`fraise-maid`（最大）

**标准集（6 个）**：上述 3 个 + `chocola-casual`（vf=3）、`milk-winter`（vf=4 大）、`vanilla-santa`（特殊服装）

**完整集**：全部 59 个模型（由于同构性，边际收益递减）

---

## 5. gu-pu 触发统计

### 5.1 触发条件映射

> flags = `c[b+592>>2]`，b 是 motion 结构。flags 的每一位由 asm.js `ut` 函数设置：`c[motion+592] |= 1 << layer_type`。
> 因此 `flags & (1<<N)` 触发 ⟺ 模型中存在 layer type N。

| 函数 | 行号 | 触发条件 | 对应 layer type | 功能 |
|------|------|----------|---------------|------|
| `gu` | L52903 | `flags & 512` (bit 9) | type 9 | stereovision parallax |
| `hu` | L53301 | 无条件 | — | mesh deform (xr/zr) |
| `iu` | L53959 | 无条件 | — | deform flag 设置 |
| `ju` | L54007 | `flags & 32` (bit 5) | type 5 | bust scale |
| `ku` | L54088 | 无条件 | — | bounding box |
| `lu` | L54167 | `flags & 2` (bit 1) | type 1 (shape) | particle/effect |
| `mu` | L83718 | `flags & 8` (bit 3) | type 3 (nested motion) | update+deform |
| `nu` | L84262 | `flags & 64` (bit 6) | type 6 | 碰撞/关联 |
| `ou` | L84421 | `flags & 16` (bit 4) | type 4 (particle) | update+deform |
| `pu` | L85356 | `flags & 1024` (bit 10) | type 10 | effect |

### 5.2 触发频率

| 函数 | 触发数 | 触发率 | 条件 | 在 NEKOPARA 中 |
|------|--------|--------|------|---------------|
| `gu` | 0/59 | 0% | flags & 512 (layer type 9) | **不触发** |
| `hu` | 59/59 | 100% | 无条件 | 总是触发 |
| `iu` | 59/59 | 100% | 无条件 | 总是触发 |
| `ju` | 0/59 | 0% | flags & 32 (layer type 5) | **不触发** |
| `ku` | 59/59 | 100% | 无条件 | 总是触发 |
| `lu` | 59/59 | 100% | flags & 2 (layer type 1) | 总是触发（shape layer 存在） |
| `mu` | 59/59 | 100% | flags & 8 (layer type 3) | 总是触发（nested motion 存在） |
| `nu` | 0/59 | 0% | flags & 64 (layer type 6) | **不触发** |
| `ou` | 0/59 | 0% | flags & 16 (layer type 4) | **不触发** |
| `pu` | 0/59 | 0% | flags & 1024 (layer type 10) | **不触发** |

### 5.3 不触发的函数

以下 5 个函数在**所有 59 个 NEKOPARA 模型中都不触发**：

| 函数 | 缺失的 layer type | 推测功能 | 影响 |
|------|------------------|----------|------|
| `gu` | type 9 | stereovision parallax | NEKOPARA 有 stereovisionControl 配置但**无 layer type 9**，gu 不触发。stereovision 可能在其他路径处理 |
| `ju` | type 5 | bust scale | NEKOPARA 无 layer type 5，bust 物理通过 bustControl 配置 + Hl 函数处理（见 `asm-physics-update-loop.md`） |
| `nu` | type 6 | 碰撞/关联 | NEKOPARA 无 layer type 6 |
| `ou` | type 4 (particle) | update+deform | NEKOPARA **无 particle layer**（type 4），所有效果通过 timeline + mesh.bp 实现 |
| `pu` | type 10 | effect | NEKOPARA 无 layer type 10 |

### 5.4 总是触发的函数

以下 5 个函数在**所有 59 个模型中总是触发**：

| 函数 | 条件 | 原因 |
|------|------|------|
| `hu` | 无条件 | mesh deform 核心（xr/zr 分支） |
| `iu` | 无条件 | deform flag 设置 |
| `ku` | 无条件 | bounding box 计算 |
| `lu` | flags & 2 | 每个模型有 6–10 个 shape layer（type 1），共 463 个 |
| `mu` | flags & 8 | 每个模型有 28–40 个 nested motion（type 3），共 2005 个 |

---

## 6. 关键发现

### 6.1 模型完全同构

**所有 59 个 NEKOPARA 模型在结构上完全同构**：

- `model_flags` = `0x0000100F`（唯一值）
- `layer_types` = `[0, 1, 2, 3, 12]`（唯一值）
- `physics` = `hairControl=4, bustControl=2, partsControl=2`（唯一值）
- `categories` = `A/D/E`（唯一值）
- `chara_count` = 21（唯一值）
- `motion_count` = 48 或 49（fraise-maid 和 milk-winter 是 49，其余 48）

这意味着 NEKOPARA 使用**统一的模型架构**，不同角色/服装只在纹理和具体 layer 参数上差异，骨架完全一致。

### 6.2 NEKOPARA 不使用 PSB mesh 顶点变形

扫描发现所有模型的 `mesh.cc` 字段**全是 bool 标志**（True/False），没有真正的顶点坐标数据。NEKOPARA 的 mesh 变形完全依赖：

1. **Bezier 网格**（`mesh.bp` = 32 个 float，4×4 UV 控制点）→ 走 **xr (Bezier) 路径**
2. **timeline 动画**驱动 bp 控制点插值
3. **physics**（hair/bust/parts）额外摆动

这解释了为什么 Python 实现的 `xr` 路径（Bezier 渲染）是 NEKOPARA 的**唯一渲染路径**，`zr`（双线性）和简单 4 角点路径在 NEKOPARA 中不使用。

### 6.3 stereovision 配置存在但 gu 不触发

所有 59 个模型的 metadata 都含 `stereovisionControl`，但 `gu`（flags & 512，layer type 9）**不触发**。这表明：

- NEKOPARA 的 stereovision 是**配置层面**的（通过 `stereovisionControl` metadata）
- 而非**layer 层面**的（无 layer type 9）
- gu 函数的 stereovision parallax 可能在 NEKOPARA 中通过其他机制实现，或该功能未启用

### 6.4 version_field 双版本

5 个模型使用 `version_field=3`（header=44，无 section H-K）：
- `chocola-casual`, `chocola-lolita`, `coconut-casual`, `vanilla-casual`, `vanilla-lolita`

54 个模型使用 `version_field=4`（header=56，有 section H-K）。

这 5 个 vf=3 模型都是 casual/lolita 服装，可能是**早期版本**制作的模型，后续模型升级到 vf=4。但两者在 layer 结构和 flags 上完全一致，说明 vf 差异不影响渲染管线。

### 6.5 layer type 2 未文档化但大量存在

`layer type 2` 在 `analysis/asm-children-layer-src-semantics.md` §9.1 的 layer type 分类表中**未明确列出**，但每个模型平均 65 个（总共 3877 个，占 17%）。

推测：layer type 2 可能是某种**容器/分组层**或**中间层**，不对应 src/icon 处理（不在 `{0,3,6,11,12}` 集合中），也不对应任何 gu-pu 触发位。需要进一步逆向 `xt` 函数 case 2 确认。

### 6.6 physics 配置完全一致

所有 59 个模型的 physics 配置数量完全相同：
- `hairControl`: 4（头发物理）
- `bustControl`: 2（胸部物理）
- `partsControl`: 2（部件物理）

这表明 NEKOPARA 角色共享**相同的物理骨架**（4 段头发 + 2 段胸部 + 2 段部件），不同角色/服装只在参数值上差异。

### 6.7 milk-winter 和 fraise-maid 异常大

| 模型 | mesh 层 | layer 数 | 与均值比 |
|------|---------|---------|---------|
| `fraise-maid` | 2722 | 907 | 1.97× / 1.97× |
| `milk-winter` | 2276 | 750 | 1.65× / 1.63× |
| 均值 | 1383 | 461 | 1.0× |

这两个模型异常大，可能是：
- `fraise-maid`：fraise 是特殊角色（只有 1 个模型），可能含更多装饰
- `milk-winter`：winter 服装 + milk 角色，可能含冬季特效层

建议视觉回归测试重点关注这两个模型的性能和渲染正确性。

---

## 7. 附录

### 7.1 扫描脚本输出

脚本运行输出（摘要）：
```
Found 59 models
成功: 59  失败: 0

五类分类统计:
  A 类: 59 个模型
  B 类: 0 个模型
  C 类: 0 个模型
  D 类: 59 个模型
  E 类: 59 个模型

gu-pu 触发统计:
  gu: 0/59 触发  [flags & 512 (bit 9, layer type 9)]
  hu: 59/59 触发  [无条件]
  iu: 59/59 触发  [无条件]
  ju: 0/59 触发  [flags & 32 (bit 5, layer type 5)]
  ku: 59/59 触发  [无条件]
  lu: 59/59 触发  [flags & 2 (bit 1, layer type 1)]
  mu: 59/59 触发  [flags & 8 (bit 3, layer type 3)]
  nu: 0/59 触发  [flags & 64 (bit 6, layer type 6)]
  ou: 0/59 触发  [flags & 16 (bit 4, layer type 4)]
  pu: 0/59 触发  [flags & 1024 (bit 10, layer type 10)]
```

### 7.2 数据文件

- 扫描脚本：`tools/scan_model_paths.py`
- 原始 JSON 数据：`analysis/.model-path-scan.json`
- 本报告：`model-path-classification.md`

### 7.3 依据文档

- `analysis/asm-render-pipeline.md` §7：gu-pu 调用位置与触发条件
- `analysis/asm-children-layer-src-semantics.md` §3.1/§9.1：layer type 分类 + `ut` 函数 flags 设置（`c[motion+592] |= 1 << layer_type`）
- `analysis/asm-du-function-analysis.md`：du 函数（build_context）中 b+592 的使用
- `analysis/asm-physics-update-loop.md`：physics 推进（Hl/Il 函数）
- `src/freemote/format/psb/loader.py`：PSBLoader 实现
- `src/freemote/player/player.py` L1000-1089：hairControl/partsControl/bustControl 读取