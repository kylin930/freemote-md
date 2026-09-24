# Python 渲染管线完整数据流分析

> 本文档记录 E-mote/FreeMote Python 实现（`src/freemote/`）中从 PSB 文件加载到
> OpenGL draw call 的完整渲染数据流。每一级记录输入/输出/数据结构/条件分支/
> 坐标系/是否插值/是否 transform/是否 deform/是否进入 GPU buffer/最终由什么
> draw call 消费。
>
> 涉及文件：
> - `src/freemote/format/psb/loader.py`（3119 行）— PSB 加载与 mesh 解码
> - `src/freemote/format/psb/motion_painter.py`（3518 行）— DrawableResource + build_context
> - `src/freemote/player/player.py`（5375 行）— 渲染管线
> - `src/freemote/graphics/renderer.py`（302 行）— Renderer ABC
> - `src/freemote/platform/opengl.py`（1230 行）— OpenGL 后端
> - `src/freemote/graphics/shader.py` — shader 源码
> - `src/freemote/format/model.py`（695 行）— 数据结构

---

## 目录

1. [PSB 加载数据流](#1-psb-加载数据流)
2. [mesh 解码现状](#2-mesh-解码现状)
3. [mesh_bp 处理](#3-mesh_bp-处理)
4. [build_context 数据流](#4-build_context-数据流)
5. [渲染管线总览](#5-渲染管线总览)
6. [sprite quad 路径（4 顶点）](#6-sprite-quad-路径4-顶点)
7. [9×9 Bezier grid 路径（81 顶点）](#7-9×9-bezier-grid-路径81-顶点)
8. [part mesh 路径（draw_elements 调用为 0 的原因）](#8-part-mesh-路径draw_elements-调用为-0-的原因)
9. [GPU buffer 和 draw call](#9-gpu-buffer-和-draw-call)
10. [与原版的疑似差异](#10-与原版的疑似差异)

---

## 1. PSB 加载数据流

### 1.1 文件 → 内存（loader.py 顶层）

**输入**：`.psb` 或 `.psb.xor` 文件字节流。

**解析阶段**（`PSBLoader.load`）：

| 阶段 | 偏移 | 内容 | 输出 |
|------|------|------|------|
| magic | 0-3 | `b"PSB\x00"` | 校验 |
| versionField | 4-5 | uint16 LE | 决定头大小（40/44/56）与 version |
| subVersion | 6-7 | uint16 LE | 仅 versionField ≥ 3 |
| section 偏移表 | 8-43 | 7×uint32 | section A-G 起始偏移 |
| 扩展 section 偏移 | 44-55 | 3×uint32 | section H-K 偏移（仅 vf>3） |
| body | header_size..end | type-tagged values | 节点树 root_value |

**XOR 解密**（`.psb.xor`，version bit0==1）：
- xorshift128 PRNG（左移 11、右移 8、右移 19，Marsaglia 变体）
- 三段解密：section 偏移表（36B）→ 扩展 section（12B）→ body
- **PRNG 种子来源未确认**（TODO(UNKNOWN-A01)）

**section 结构**（psb-types.md §3）：

| Section | 语义 | 是否解析 |
|---------|------|----------|
| A/B | 键偏移数组 | ✅ |
| C | 字符串偏移表 | ✅ |
| D | 字符串数据（null 结尾 UTF-8） | ✅ |
| E/F | ref1 标记 | ✅ |
| G | ref1 目标数据（chunk data，含纹理像素） | ✅（pixel_data 解码） |
| H | 扩展（vf>3） | 部分 |
| I/J/K | ref2（vf>3） | 部分 |

**body 类型系统**：每个值以单字节 type tag 开头，tag%4 决定 varint 字节宽度。
- tag 0x15-0x18：strref → section C → D 字符串
- tag 0x19-0x1c：ref1 → section E → G chunk data
- tag 0x22-0x25：ref2 → section I → K
- tag 0x20/0x21：type12/type13，**语义未确认**（TODO(UNKNOWN-type12/13)）

**输出**：`ModelResource`（model.py L574），含：
- `textures: list[TextureData]` — 纹理 atlas（pixel_data 非空）
- `meshes: list[MeshData]` — **vertices=b""、indices=b""、vertex_count=0**（未解码）
- `sprites: tuple[SpriteInfo, ...]` — sprite 引用（含 mesh_bp）
- `root_value: dict` — PSB 节点树根（供累积变换管线遍历）
- `base_chara` / `base_motion` / `screen_width` / `screen_height`
- `icon_table: tuple[IconRegion, ...]` — atlas icon 区域
- `easing_table` — cubic spline 控制点（当前所有 NEKOPARA 模型为空）

### 1.2 纹理提取（`_extract_emt_textures`，loader.py L1518）

**输入**：`root_value['source']['tex']['texture']` 字典。
**算法**：递归查找所有含 `"pixel"` 键的字典（对应 C# FindMotionResources）。
**字段提取**：

| 字段 | 来源 | 类型 | 含义 |
|------|------|------|------|
| pixel_data | `d['pixel'].data` | bytes | RGBA8 像素字节（从 section G chunk data 解码） |
| width / height | `d['width']` / `d['height']` | int | 纹理尺寸 |
| pixel_format | `d.get('type', 'RGBA8')` | str | 像素格式 |
| chunk_index | `d['pixel'].index` | int | chunk data 索引 |
| truncated_width/height | `d.get('truncated_*', 0)` | int | 截断尺寸 |

**输出**：`list[TextureData]`，pixel_data 非空。**进入 GPU**：通过 `renderer.create_texture` + `renderer.upload_texture_data` 上传为 GL 纹理（GL_RGBA/GL_UNSIGNED_BYTE）。

### 1.3 sprite 提取（`_extract_sprites`，loader.py L1604）

**输入**：`root_value['object'][chara]['motion'][motion]['layer']` 层次结构。
**遍历**：沿 layer 树递归（`_walk_layer`），对每个 frame content 含 `src`+`icon` 的 frame 构建 SpriteInfo。
**字段提取**：

| 字段 | 来源 | 类型 | 含义 |
|------|------|------|------|
| src | `content['src']` | str | part 名 |
| icon | `content['icon']` | str | 维度字符串 'W:H:W/2:H/2' |
| mesh_bp | `content['mesh']['bp']` | tuple[32 float] | 4×4 Bezier 控制点（空元组=无变形） |
| coord | `content['coord']` | tuple[3 float] | 帧坐标 (x,y,z) |
| opacity | `content['opa']` | int | 不透明度 0-255 |
| angle | `content['angle']` | float | 旋转角度 |
| blend_mode | `content['bm']` | int | 混合模式 0-11 |
| uv_coords | icon 表匹配计算 | tuple[4 float] | atlas UV (u0,v0,u1,v1)，**V 已翻转** |
| icon_width/height | 维度字符串解析 | int | icon 尺寸 |
| layer_path | layer 标签路径 | tuple[str,...] | 层次路径 |

**UV 计算**（L1733-1738）：
```python
u0 = left / atlas_width
v0 = top / atlas_height
u1 = (left + w) / atlas_width
v1 = (top + h) / atlas_height
uv_coords = (u0, 1.0 - v1, u1, 1.0 - v0)  # V 翻转适配 OpenGL bottom-up
```

**输出**：`tuple[SpriteInfo, ...]`。**不直接进入 GPU**，供 player.py 回退路径使用。

---

## 2. mesh 解码现状

### 2.1 已解码

| 数据 | 来源 | 状态 |
|------|------|------|
| 纹理像素数据 | section G chunk data（ref1 → E → G） | ✅ pixel_data 非空 |
| mesh_bp（16 个 Bezier 控制点） | frame content['mesh']['bp']（32 float） | ✅ 完整解码 |
| icon 表（atlas 区域） | `source['tex']['icon']` | ✅ |
| layer 树 + frameList | `object[chara][motion][layer]` | ✅ |
| 变量 / timeline / marker / profile | metadata 节点 | ✅ |
| easing table | `root['easing']` | ✅（当前模型为空） |

### 2.2 未解码

**PSB mesh 顶点未解码**（loader.py L1507-1514）：

```python
meshes.append(MeshData(
    vertices=b"",  # 顶点数据留空，需 section G 解码
    indices=b"",   # 索引数据留空，需 section G 解码
    vertex_count=0,
    index_count=0,
    name=f"mesh_{i}",
    vertex_layout="pos3_uv2",
))
```

**影响**：
- `MeshData.vertices` / `indices` / `vertex_count` / `index_count` 全为空/零
- `_draw_part`（player.py L3373）因 `mesh.vertex_count == 0` 跳过所有 part
- **part mesh 渲染路径完全不发出 draw call**（详见 §8）

**未确认项**：
- `index_format` uint16 vs uint32（TODO(UNKNOWN-A04)）
- type12 / type13 tag 语义（TODO(UNKNOWN-type12/13)）
- PRNG 种子来源（TODO(UNKNOWN-A01)）

### 2.3 渲染走哪条路径？

由于 part mesh 未解码，渲染实际走的路径：

```
_draw_player_mesh
  ├─ 优先：_draw_drawable_resources（累积变换管线，root_value 可用 + 纹理 pixel_data 非空）
  │    ├─ collect_drawable_resources → list[DrawableResource]
  │    └─ 对每个 DrawableResource：
  │         ├─ mesh_bp 含 32 float → _draw_drawable_resource_grid（9×9 Bezier grid）
  │         └─ 否则 → _draw_drawable_resource（4 顶点 quad）
  ├─ 回退 1：_draw_sprite_quads（sprite quad，model.sprites 非空）
  └─ 回退 2：_draw_part（part mesh，因 vertex_count==0 跳过）
```

**当前实际路径**：`_draw_drawable_resources`（累积变换管线）。

---

## 3. mesh_bp 处理

### 3.1 mesh_bp 的含义

`mesh_bp` 是 **16 个 Bezier 控制点**（4×4 控制网格，32 个 float，row-major），
每个控制点是 `(u, v)` 归一化坐标（0-1 范围），表示该控制点在 sprite 局部归一化
坐标系中的位置。

- **空元组 `()`**：无 mesh 形变（`content['mesh']['bp']` 是 bool True 或无 mesh 字段）
- **32 个 float**：有形变，走 9×9 Bezier grid 路径

### 3.2 从 PSB 到渲染的完整流程

```
PSB frame content['mesh']['bp'] (list of 32 float)
  ↓ read_frame_content (motion_painter.py L937)
FrameContent.mesh_bp (tuple[32 float] 或 ())
  ↓ get_complete_frame_content (L1221) — 帧间线性插值
FrameContent.mesh_bp (插值后)
  ↓ build_context (L464) — 加法偏移组合
RenderContext.mesh_bp (累积)
  ↓ _make_drawable_resource (L2541)
DrawableResource.mesh_bp (= ctx.mesh_bp)
  ↓ _draw_drawable_resource (player.py L2285)
  │   ├─ len(mesh_bp)==32 → _draw_drawable_resource_grid (9×9 Bezier grid)
  │   └─ 否则 → 4 顶点 quad
  ↓ _bezier_patch_eval (L442) — 双三次 Bezier 曲面求值
81 顶点 (x, y) → NDC → GPU VBO
```

### 3.3 帧间插值（`_interpolate_mesh_bp`，motion_painter.py L1062）

**触发条件**：frame type==3（tween）且存在下一帧且下一帧 type!=0。
**算法**：线性插值 `result[i] = bp1[i] + (bp2[i] - bp1[i]) * t`。
**缺失帧处理**：`bp=True` 或无 mesh 字段时视为默认均匀网格（`_default_mesh_bp`）。
**优化**：插值结果接近默认网格（偏差 < 1e-6）时返回空元组，走简单 quad 路径。

### 3.4 加法偏移组合（`_compose_mesh_bp`，motion_painter.py L1133）

**对应 asm.js hu 函数 L53540-53566**：
```
result = parent + (frame - default)
```
等价于递归展开：`result = default + sum(layer - default for all layers)`

| 条件 | 结果 |
|------|------|
| frame 和 parent 都有值 | `result[i] = parent[i] + frame[i] - default[i]` |
| 只有 frame 有值 | `result = frame` |
| 只有 parent 有值 | `result = parent` |
| 都无值 | `result = ()`（空元组，无变形） |

**默认网格**（`_default_mesh_bp`，L1046）：均匀分布 16 顶点，`u = i%4/3`，`v = i//4/3`。

### 3.5 数据结构

```python
# FrameContent.mesh_bp (motion_painter.py L236)
mesh_bp: tuple[float, ...] = ()  # 32 float 或空

# RenderContext.mesh_bp (motion_painter.py L279)
mesh_bp: tuple[float, ...] = ()  # 累积后

# DrawableResource.mesh_bp (motion_painter.py L347)
mesh_bp: tuple[float, ...] = ()  # 传给渲染
```

**坐标系**：sprite 局部归一化坐标（0-1 范围）。
**是否插值**：是（帧间线性插值）。
**是否 transform**：是（加法偏移组合，累积父层变形）。
**是否 deform**：是（Bezier 曲面变形）。
**是否进入 GPU**：是（通过 9×9 grid 顶点位置进入 VBO）。

---

## 4. build_context 数据流

### 4.1 输入/输出

**输入**：
- `layer: dict` — PSB layer 字典（含 transformOrder / inherit* 标志）
- `frame: FrameContent` — 当前帧内容
- `parent: RenderContext` — 父层渲染上下文
- `root: RenderContext | None` — 根层渲染上下文（三态模型用）
- `physics_angle: float` — 物理摆动角度（E-mote 内部单位，1 unit = π/80 rad）

**输出**：`RenderContext`（子层渲染上下文）。

### 4.2 RenderContext 数据结构（motion_painter.py L247）

```python
@dataclass(frozen=True, slots=True)
class RenderContext:
    x: float = 0.0          # 累积世界 X
    y: float = 0.0          # 累积世界 Y
    z: float = 0.0          # 累积世界 Z（深度排序用）
    opacity: int = 255      # 累积不透明度 0-255
    visible: bool = True    # 可见性
    matrix: MotionMatrix2 = identity  # 累积 2×2 仿射矩阵
    mesh_bp: tuple[float, ...] = ()   # 累积 mesh_bp（加法偏移）
    flip_x/flip_y: bool = False       # 累积翻转
    angle: float = 0.0                # 累积旋转角度
    zoom_x/zoom_y: float = 1.0        # 累积缩放
    slant_x/slant_y: float = 0.0      # 累积倾斜
```

### 4.3 三态模型（对应 asm.js du 函数）

**inherit_all_affine** = 所有 7 个 transform inherit 标志全为 True。
**motion_independent** = `layer.get('motionIndependentLayerInherit', False)`。

| 状态 | 条件 | matrix 计算 | local 分量 |
|------|------|-------------|------------|
| 状态 1 | `root is None` 或 `inherit_all_affine=True` | `parent.matrix * local` | `local = frame` |
| 状态 2 | `inherit_all_affine=False` 且 `motion_independent=False` | `root.matrix * local_adjusted` | `local = frame + parent - root`（inherit=true 分量） |
| 状态 3 | `inherit_all_affine=False` 且 `motion_independent=True` | `local`（不乘任何矩阵） | `local = frame + parent`（inherit=true 分量） |

**当前所有 59 个 NEKOPARA 模型全用默认 inherit=True**，始终走状态 1。

### 4.4 变换累积流程（状态 1）

```python
# 1. local 矩阵 = calc_affine_matrix(transform_order, frame 各分量)
local = calc_affine_matrix(transform_order, frame.flip_x, frame.flip_y,
                           frame.angle, frame.zoom_x, frame.zoom_y,
                           frame.slant_x, frame.slant_y)

# 2. 透明度累积
opacity = round(parent.opacity * frame.opacity / 255.0)  # inherit_opacity=true

# 3. matrix = parent.matrix * local
matrix = MotionMatrix2.multiply(parent.matrix, local)

# 4. child 分量累积（供孙层使用）
child_flip_x = frame.flip_x != parent.flip_x  # XOR
child_angle = frame.angle + parent.angle      # 加
child_zoom_x = frame.zoom_x * parent.zoom_x   # 乘
child_slant_x = frame.slant_x + parent.slant_x  # 加

# 5. 位移：parent.matrix 变换 frame.coord
tx, ty = parent.matrix.transform(frame.coord_x, frame.coord_y)

# 6. 物理摆动（physics_angle 非零时）
if abs(physics_angle) > 1e-6:
    rot = MotionMatrix2(cos, -sin, sin, cos)
    matrix = rot @ matrix                    # 左乘旋转矩阵
    tx, ty = rot @ (tx, ty)                  # 绕 parent position 旋转位移

# 7. mesh_bp 加法偏移组合
mesh_bp = _compose_mesh_bp(parent.mesh_bp, frame.mesh_bp)

# 8. 输出 RenderContext
return RenderContext(
    x=parent.x + tx, y=parent.y + ty, z=parent.z + frame.coord_z,
    opacity=opacity, visible=parent.visible, matrix=matrix,
    mesh_bp=mesh_bp, flip_x=child_flip_x, ...
)
```

### 4.5 calc_affine_matrix（motion_painter.py L397）

从单位矩阵开始，按 `transform_order` 顺序依次应用 flip / zoom / angle / slant。
**默认顺序**：`[flip, slant, zoom, angle]`。

| 变换 | 公式 |
|------|------|
| flip | `m11,m12 = -m11,-m12`（flip_x）；`m21,m22 = -m21,-m22`（flip_y） |
| zoom | `m11*=zoom_x; m12*=zoom_x; m21*=zoom_y; m22*=zoom_y` |
| angle | 旋转矩阵乘法（角度→弧度：`rad = angle * π * 2 / 360`） |
| slant | `m11 += slant_x*m21; m21 = slant_y*m11 + m21` 等 |

### 4.6 物理摆动应用

**触发条件**：`physics_angle` 非零（由 `_get_physics_angle_for_label` 按 layer label 查询）。
**应用方式**（对应 asm.js gu-pu 物理摆动类型 3）：
1. **左乘旋转矩阵**到累积 matrix：`matrix = rot @ matrix`
2. **绕 parent position 旋转累积位移**：`(tx, ty) → rot @ (tx, ty)`

**角度单位转换**：`_EMOTE_ANGLE_TO_RAD = π / 80`（E-mote 内部单位 → 弧度）。

**物理类别匹配**（`_get_physics_angle_for_label`，player.py L2236）：
- label 含 `'髪揺れ'` → `_hair_physics.output_angles[2]`（y 方向响应）
- label 含 `'パーツ揺れ'` → `_parts_physics.output_angles[2]`
- label 含 `'胸'` → `_bust_physics.output_angles[0]`（x 方向，主角度）
- 其他 → 0.0

**坐标系**：模型坐标系（累积世界坐标）。
**是否插值**：否（build_context 不插值，插值在 get_complete_frame_content 完成）。
**是否 transform**：是（matrix 累积 + physics 旋转）。
**是否 deform**：是（mesh_bp 加法偏移组合）。
**是否修改 position**：是（`x = parent.x + tx`，physics 旋转修改 tx/ty）。
**是否进入 GPU**：否（RenderContext 是中间态，后续转 DrawableResource 再转 GPU）。

---

## 5. 渲染管线总览

### 5.1 渲染入口（`draw`，player.py L1821）

```
EmotePlayer.draw(render_texture_id)
  ↓
renderer.begin_render_target(handle)    # glBindFramebuffer + viewport/scissor
renderer.clear_color(0, 0, 0, 0)        # glClearColor
renderer.clear(color, depth, stencil)   # glClear
  ↓
mask.enable(renderer)                   # 若 mask 启用
  ↓
self._draw_player_mesh(renderer)        # 核心渲染
  ↓
mask.disable(renderer)
renderer.end_render_target()            # glBindFramebuffer(0)
```

### 5.2 _draw_player_mesh（player.py L1907）

**路径选择**（优先级从高到低）：

| 优先级 | 条件 | 路径 | 实际触发 |
|--------|------|------|----------|
| 1 | `root_value` 是 dict + 有 valid textures | `_draw_drawable_resources` | ✅ 当前实际路径 |
| 2 | `model.sprites` 非空 + atlas pixel_data 非空 | `_draw_sprite_quads` | 回退 |
| 3 | `model.parts` 非空 | `_draw_part`（逐 part） | 因 mesh 未解码跳过 |

### 5.3 _draw_drawable_resources（player.py L1981）

**输入**：
- `root_value`（PSB 节点树根）
- `model.base_chara` / `model.base_motion`（基础角色/motion）
- `variable_values`（变量名→值，主值+差分叠加值）
- `physics_angles`（layer label→物理摆动角度）

**流程**：

```
1. 获取 motion / chara / time
   motion = model.base_motion
   time = timeline_runner 中匹配 motion 的 current_frame

2. 收集变量值
   variable_values[label] = main_value + diff_value

3. 计算物理摆动角度 dict
   _layer_positions = collect_all_layer_positions(root_value, chara, motion, time)
   for label in _layer_positions:
       angle = _get_physics_angle_for_label(label)
       if abs(angle) > 1e-6: physics_angles[label] = angle

4. 收集主 timeline 资源
   all_resources = collect_drawable_resources(
       root_value, chara, motion, time,
       variable_values=variable_values,
       physics_angles=physics_angles,
   )

5. 收集差分 timeline 资源（带 DIFFERENCE flag）
   for state in playing_states:
       if state.flags & DIFFERENCE:
           diff_res = collect_drawable_resources(...)
           # 应用 blend_ratio 到 opacity
           all_resources.extend(diff_res)

6. 按 Z 排序
   all_resources.sort(key=lambda r: r.z)

7. 上传纹理 atlas（缓存）
   atlas_handles = _get_or_upload_atlas_multi(renderer, textures)

8. 编译 sprite shader program（缓存）
   program = _get_or_compile_sprite_program(renderer)

9. 计算屏幕尺寸 + fit_scale + offset
   screen_w, screen_h = _get_screen_size(model)
   bounds = compute_bounds(all_resources)
   fit_scale = min(screen_w/content_w, screen_h/content_h) * 0.9
   offset_x = screen_w / 2  # PSB (0,0) 映射到屏幕中心
   offset_y = screen_h / 2

10. 逐资源渲染（已按 Z 从后向前排序）
    for res in all_resources:
        _draw_drawable_resource(renderer, program, res, atlas_handles,
                                 screen_w, screen_h, fit_scale, offset_x, offset_y)
```

**输出**：`bool`（是否成功渲染）+ 渲染统计（draw_calls/vertices/triangles/textures）。

### 5.4 collect_drawable_resources（motion_painter.py L1469）

**输入**：root_value / chara / motion / time / variable_values / physics_angles / easing_table。
**算法**：递归遍历 layer 树（`_travel`），对每个含 frameList 的 layer：
1. `get_complete_frame_content(frame_list, layer_time, easing_table)` — 找帧 + 插值
2. 查询 `physics_angles[layer.label]` 获取物理摆动角度
3. `build_context(layer, frame, parent_ctx, root_ctx, physics_angle)` — 累积变换
4. `_try_add_resource(...)` — 根据 frame.source 前缀分发：
   - `'src/'` → 直接 sprite 引用，查 icon 表 → `_make_drawable_resource`
   - `'motion/'` → 嵌套 motion 引用，递归 `_travel_motion`
   - frame.icon 非空 + source 在 source 表 → icon 引用
   - frame.icon 非空 + source 在 object 表 → 嵌套 motion

**输出**：`list[DrawableResource]`，按 Z 排序（从小到大，从后向前）。

### 5.5 DrawableResource 数据结构（motion_painter.py L294）

```python
@dataclass(frozen=True, slots=True)
class DrawableResource:
    x: float                    # 累积世界 X
    y: float                    # 累积世界 Y
    z: float                    # 累积世界 Z（深度排序）
    matrix: MotionMatrix2       # 累积 2×2 仿射矩阵
    origin_x: float             # 原点 X（frame.ox + icon_origin_x）
    origin_y: float             # 原点 Y（frame.oy + icon_origin_y）
    opacity: int                # 累积不透明度 0-255
    visible: bool               # 可见性
    label: str                  # layer 标签名
    motion_name: str            # motion 名
    icon_width: float           # icon 宽度（像素）
    icon_height: float          # icon 高度（像素）
    uv_coords: tuple[4 float]   # atlas UV (u0, v0, u1, v1)
    texture_index: int          # 纹理索引
    source: str                 # frame content src
    icon_name: str              # 匹配的 icon 表条目名
    mesh_bp: tuple[float, ...] = ()  # 4×4 Bezier 控制点（32 float 或空）
    color: int | None = None    # 顶点颜色 0xRRGGBBAA
    blend_mode: int = -1        # 混合模式 0-11
    mesh_cc: bool = True        # mesh ColorControl 标志
```

**关键方法**：

```python
def transform_point(self, x, y, x_offset=0, y_offset=0):
    px, py = self.matrix.transform(x - self.origin_x, y - self.origin_y)
    return (self.x + px + x_offset, self.y + py + y_offset)

def get_corners(self):
    return (
        self.transform_point(0.0, 0.0),       # 左上
        self.transform_point(w, 0.0),         # 右上
        self.transform_point(w, h),           # 右下
        self.transform_point(0.0, h),         # 左下
    )
```

### 5.6 _make_drawable_resource（motion_painter.py L2541）

**UV 计算**（**注意：此处未做 V 翻转**）：
```python
u0 = left / atlas_w
v0 = top / atlas_h
u1 = (left + icon_w) / atlas_w
v1 = (top + icon_h) / atlas_h
# 直接传入，未做 V 翻转！
uv_coords = (u0, v0, u1, v1)
```

**与 loader.py `_extract_sprites` 的差异**：loader.py 的 SpriteInfo.uv_coords 做了
`V 翻转 (u0, 1-v1, u1, 1-v0)`，但 motion_painter.py 的 DrawableResource.uv_coords
**未做 V 翻转**。这是一个潜在差异点（详见 §10）。

---

## 6. sprite quad 路径（4 顶点）

### 6.1 触发条件

`DrawableResource.mesh_bp` 不含 32 个 float（空元组或长度非 32）。

### 6.2 _draw_drawable_resource（player.py L2285）

**输入**：DrawableResource + 屏幕尺寸 + fit_scale + offset。
**流程**：

```
1. 跳过条件：not res.visible or res.opacity <= 0
2. 选择纹理句柄：atlas_handles[res.texture_index]（越界回退 0）
3. mesh_bp 含 32 float → 转走 _draw_drawable_resource_grid（§7）

4. 获取 4 角屏幕坐标
   corners = res.get_corners()  # 左上/右上/右下/左下

5. UV 坐标
   u0, v0, u1, v1 = res.uv_coords
   corner_uvs = ((u0,v0), (u1,v0), (u1,v1), (u0,v1))

6. 打包顶点数据（pos2_uv2 布局，16 字节/顶点）
   for (sx, sy), (u, v) in zip(corners, corner_uvs):
       scaled_x = sx * fit_scale + offset_x
       scaled_y = sy * fit_scale + offset_y
       x_ndc = (scaled_x - half_w) / half_w
       y_ndc = -(scaled_y - half_h) / half_h    # Y 翻转
       vertex_bytes += struct.pack("4f", x_ndc, y_ndc, u, v)

7. 索引：6 个 uint16 (0,1,2, 0,2,3)

8. 上传 VBO/IBO
   vbo = renderer.create_buffer(GL_ARRAY_BUFFER, vertex_bytes, GL_STATIC_DRAW)
   ibo = renderer.create_buffer(GL_ELEMENT_ARRAY_BUFFER, index_bytes, GL_STATIC_DRAW)

9. 构建 uniforms
   uniforms = _build_drawable_uniforms(program, res, selected_handle)

10. 绘制
    renderer.draw_elements(program.handle, vbo, ibo, uniforms)

11. 清理 VBO/IBO（每帧重建）
```

### 6.3 数据流总结

| 阶段 | 数据 | 坐标系 | transform | deform |
|------|------|--------|-----------|--------|
| get_corners | 4 角 (x,y) | 屏幕坐标（Y 向下） | matrix.transform + origin 偏移 | 否 |
| 缩放+偏移 | scaled_x/y | 屏幕坐标 | fit_scale + offset | 否 |
| NDC 转换 | x_ndc/y_ndc | NDC（[-1,1]，Y 翻转） | 线性映射 | 否 |
| UV | (u,v) | 纹理坐标 [0,1] | 无 | 否 |
| 打包 | 4×(pos2+uv2) | NDC + 纹理 | 无 | 否 |

**顶点数**：4。**三角形数**：2。**索引数**：6（GL_TRIANGLES）。
**绘制模式**：`GL_TRIANGLES`（默认，未设置 `_draw_mode`）。
**最终 draw call**：`glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, None)`。

---

## 7. 9×9 Bezier grid 路径（81 顶点）

### 7.1 触发条件

`DrawableResource.mesh_bp` 含 32 个 float（`isinstance(res.mesh_bp, tuple) and len(res.mesh_bp) == 32`）。

### 7.2 _draw_drawable_resource_grid（player.py L2402）

**输入**：DrawableResource（mesh_bp 含 32 float）+ 屏幕尺寸 + fit_scale + offset。
**流程**：

```
1. 屏幕尺寸半值 half_w, half_h
2. 选择纹理句柄
3. UV 区域 u0, v0, u1, v1 = res.uv_coords

4. 构建 81 顶点（9×9 网格）
   for i in range(81):
       row = i // 9
       col = i % 9
       u = col / 8  # 列参数 0=左, 1=右
       v = row / 8  # 行参数 0=顶, 1=底

       # 双三次 Bezier 曲面求值 → 归一化坐标
       bp_u, bp_v = _bezier_patch_eval(mesh_bp, u, v)

       # 归一化 → sprite 局部像素坐标
       local_x = bp_u * res.icon_width
       local_y = bp_v * res.icon_height

       # 变换到屏幕坐标（应用累积变换）
       sx, sy = res.transform_point(local_x, local_y)

       # 缩放 + 偏移
       scaled_x = sx * fit_scale + offset_x
       scaled_y = sy * fit_scale + offset_y

       # 转 NDC（Y 翻转）
       x_ndc = (scaled_x - half_w) / half_w
       y_ndc = -(scaled_y - half_h) / half_h

       # UV：均匀 9×9 网格映射到 uv_coords
       uv_u = u0 + u * (u1 - u0)
       uv_v = v0 + v * (v1 - v0)

       vertex_bytes += struct.pack("4f", x_ndc, y_ndc, uv_u, uv_v)

5. 索引：158 个 uint16（GL_TRIANGLE_STRIP + 退化顶点）
   index_bytes = struct.pack("158H", *BEZIER_GRID_INDICES_STRIP)

6. 上传 VBO/IBO
7. 构建 uniforms（复用 _build_drawable_uniforms）
   uniforms["_draw_mode"] = GL_TRIANGLE_STRIP  # 0x0005
8. 绘制
   renderer.draw_elements(program.handle, vbo, ibo, uniforms)
9. 清理 VBO/IBO
```

### 7.3 _bezier_patch_eval（player.py L442）

**数学实现**：双三次 Bezier 曲面求值。

$$S(u,v) = \sum_{i=0}^{3} \sum_{j=0}^{3} B_i^3(u) \cdot B_j^3(v) \cdot P[i][j]$$

其中：
- $B_i^3(t)$ 是三次 Bernstein 基函数（`_bernstein_cubic`，L424）：
  - $B_0(t) = (1-t)^3$
  - $B_1(t) = 3t(1-t)^2$
  - $B_2(t) = 3t^2(1-t)$
  - $B_3(t) = t^3$
- $P[i][j] = \text{mesh\_bp}[(i \cdot 4 + j) \cdot 2 : (i \cdot 4 + j) \cdot 2 + 2]$ 是 4×4 控制点

**代码**：
```python
def _bezier_patch_eval(mesh_bp, u, v):
    bu = _bernstein_cubic(u)  # (B0,B1,B2,B3)
    bv = _bernstein_cubic(v)
    x = 0.0
    y = 0.0
    for row in range(4):
        bw = bv[row]
        for col in range(4):
            w = bw * bu[col]
            idx = (row * 4 + col) * 2
            x += w * mesh_bp[idx]
            y += w * mesh_bp[idx + 1]
    return (x, y)
```

**对应 asm.js xr 函数 L14719-14996**（张量积插值）。

### 7.4 索引生成（`_build_triangle_strip_indices`，player.py L295）

**对应 asm.js kx 函数 L10530-10579**。
**拓扑**：GL_TRIANGLE_STRIP + 退化顶点。

```
行0: 0,9, 1,10, 2,11, ..., 8,17
退化: 17,9
行1: 9,18, 10,19, ..., 17,26
退化: 26,18
...
行8: 72,81, ..., 80
```

**总索引数**：`rows × (cols+2) × 2 - 2 = 8 × 10 × 2 - 2 = 158`。

### 7.5 常量

```python
BEZIER_GRID_DIVISION = 8          # 每维分割数
BEZIER_GRID_VERTEX_COUNT = 81     # (8+1)²
BEZIER_GRID_INDEX_COUNT_STRIP = 158  # 8×10×2-2
```

### 7.6 数据流总结

| 阶段 | 数据 | 坐标系 | transform | deform |
|------|------|--------|-----------|--------|
| Bezier 求值 | (bp_u, bp_v) | sprite 局部归一化 [0,1] | 无 | **是**（Bezier 曲面变形） |
| 归一化→像素 | (local_x, local_y) | sprite 局部像素 | ×icon_width/height | 否 |
| transform_point | (sx, sy) | 屏幕坐标（Y 向下） | matrix.transform + origin | 否 |
| 缩放+偏移 | scaled_x/y | 屏幕坐标 | fit_scale + offset | 否 |
| NDC 转换 | x_ndc/y_ndc | NDC（Y 翻转） | 线性映射 | 否 |
| UV | (uv_u, uv_v) | 纹理坐标 [0,1] | 均匀网格→uv_coords | 否 |

**顶点数**：81。**三角形数**：128（8×8×2）。**索引数**：158（GL_TRIANGLE_STRIP）。
**绘制模式**：`GL_TRIANGLE_STRIP`（0x0005，通过 `uniforms["_draw_mode"]` 指定）。
**最终 draw call**：`glDrawElements(GL_TRIANGLE_STRIP, 158, GL_UNSIGNED_SHORT, None)`。

### 7.7 与 build_sprite_grid 的关系

`build_sprite_grid`（player.py L3252）是**静态方法**，用于回退路径
（`_draw_sprite_grid`，L2897）。逻辑与 `_draw_drawable_resource_grid` 类似，
但顶点位置映射到 NDC 的方式不同：

| 方法 | 顶点 NDC 计算 | 用途 |
|------|---------------|------|
| `_draw_drawable_resource_grid` | `transform_point(local_x, local_y)` → 缩放/偏移 → NDC | 累积变换管线（当前主路径） |
| `build_sprite_grid` | `bp_u * 2 - 1`, `1 - bp_v * 2`（直接归一化→NDC） | 回退路径（SpriteInfo，无累积变换） |

---

## 8. part mesh 路径（draw_elements 调用为 0 的原因）

### 8.1 _draw_part（player.py L3373）

```python
def _draw_part(self, renderer, program, part):
    model = self._model_resource
    mesh = self._find_mesh(model, part.mesh_ref)
    if mesh is None or mesh.vertex_count == 0 or mesh.index_count == 0:
        return  # skip parts without mesh data
    # ... 后续 draw_elements 永远不会执行
```

### 8.2 为什么 draw_elements 调用为 0

**根因**：PSB mesh 顶点未解码（loader.py L1508）。

```python
# loader.py L1507-1514
meshes.append(MeshData(
    vertices=b"",  # 顶点数据留空
    indices=b"",   # 索引数据留空
    vertex_count=0,
    index_count=0,
    ...
))
```

**链式影响**：
1. `MeshData.vertex_count == 0` → `_draw_part` 在 L3401 提前 return
2. `glDrawElements` 永远不被调用
3. **part mesh 渲染路径完全不发出 draw call**

### 8.3 实际渲染路径

由于 part mesh 未解码，渲染实际走 `_draw_drawable_resources`（累积变换管线），
该路径**不依赖 MeshData**，而是从 `root_value`（PSB 节点树）实时计算顶点：
- 简单 sprite → 4 顶点 quad（`_draw_drawable_resource`）
- mesh 变形 → 81 顶点 Bezier grid（`_draw_drawable_resource_grid`）

**part mesh 路径的存在意义**：对应 asm.js Nx 序列（rendering.md §3.10），
未来若解码 PSB mesh 顶点，可走原生 part mesh 渲染（性能更高，无需实时 Bezier 求值）。

### 8.4 part mesh 路径的完整逻辑（若 mesh 已解码）

```
_draw_part(renderer, program, part):
    mesh = _find_mesh(model, part.mesh_ref)
    # 若 mesh.vertex_count > 0：
    vbo = renderer.create_buffer(GL_ARRAY_BUFFER, mesh.vertices, GL_STATIC_DRAW)
    ibo = renderer.create_buffer(GL_ELEMENT_ARRAY_BUFFER, mesh.indices, GL_STATIC_DRAW)
    uniforms = _build_draw_uniforms(program)
    renderer.draw_elements(program.handle, vbo, ibo, uniforms)
    renderer.delete_buffer(vbo)
    renderer.delete_buffer(ibo)
```

**顶点布局**：core shader（`pos4 + uv2 + color4` = 40 字节/顶点），与 sprite shader
（`pos2 + uv2` = 16 字节/顶点）不同。`draw_elements` 中默认布局为
`vec3 pos @ loc0 + vec2 uv @ loc1, stride=20B`（L751-755）。

---

## 9. GPU buffer 和 draw call

### 9.1 Renderer ABC（renderer.py L46）

**接口方法**：

| 方法 | 功能 | OpenGL 对应 |
|------|------|-------------|
| `create_texture(w, h)` | 创建纹理 | glGenTextures |
| `upload_texture_data(handle, w, h, data, fmt)` | 上传像素数据 | glTexSubImage2D（GL_RGBA/GL_UNSIGNED_BYTE） |
| `create_render_texture(w, h)` | 创建 RTT | glGenFramebuffers + glGenTextures + glGenRenderbuffers |
| `begin_render_target(handle)` | 绑定 RTT | glBindFramebuffer + glViewport + glScissor |
| `end_render_target()` | 解绑 RTT | glBindFramebuffer(0) |
| `clear_color(r,g,b,a)` | 设置清色 | glClearColor |
| `clear(color,depth,stencil)` | 清缓冲 | glClear |
| `draw_elements(program, vbo, ibo, uniforms)` | 绘制 | glDrawElements |
| `compile_shader(vs, fs)` | 编译链接 | glCreateShader + glCompileShader + glLinkProgram |
| `create_buffer(target, data, usage)` | 创建缓冲 | glGenBuffers + glBufferData |
| `delete_buffer(handle)` | 删除缓冲 | glDeleteBuffers |
| `blit(src, dst, ...)` | 位块传送 | glBlitFramebuffer 或类似 |

### 9.2 draw_elements 完整序列（opengl.py L675）

```
1. glUseProgram(program)

2. set_blend_mode(uniforms.get("_blend_mode", 0))
   → glEnable/glDisable(GL_BLEND) + glBlendEquationSeparate + glBlendFuncSeparate

3. glBindVertexArray(default_vao)  # Core Profile 要求

4. glBindBuffer(GL_ARRAY_BUFFER, vbo)
   根据 uniforms["_vertex_layout"]：
   - "pos2_uv2"（sprite quad/grid）:
       glEnableVertexAttribArray(0)
       glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 16, None)       # pos @ loc0
       glEnableVertexAttribArray(1)
       glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 16, offset(8))  # uv @ loc1
   - 默认（core mesh）:
       glEnableVertexAttribArray(0)
       glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 20, None)       # pos @ loc0
       glEnableVertexAttribArray(1)
       glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 20, offset(12)) # uv @ loc1

5. glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo)

6. 若 uniforms["_texture_handle"] 存在:
       glActiveTexture(GL_TEXTURE0)
       glBindTexture(GL_TEXTURE_2D, texture_handle)

7. 设置 uniforms（遍历 uniforms dict，跳过 _ 前缀元数据键）
   根据 value 类型分派：
   - int → glUniform1i
   - float → glUniform1f
   - tuple/list 长度 2/3/4 → glUniform{2,3,4}f
   - 长度 16 → glUniformMatrix4fv
   - location == -1（uniform 不存在/被优化掉）→ 静默跳过

8. count = ibo_size // 2  # GL_UNSIGNED_SHORT = 2 bytes
   draw_mode = uniforms.get("_draw_mode", GL_TRIANGLES)
   glDrawElements(draw_mode, count, GL_UNSIGNED_SHORT, None)
```

### 9.3 顶点缓冲布局

| 路径 | 布局 | stride | 顶点数 | 索引数 | 绘制模式 |
|------|------|--------|--------|--------|----------|
| sprite quad | pos2 + uv2 | 16B | 4 | 6 | GL_TRIANGLES |
| 9×9 Bezier grid | pos2 + uv2 | 16B | 81 | 158 | GL_TRIANGLE_STRIP |
| part mesh（未启用） | pos3 + uv2 | 20B | ? | ? | GL_TRIANGLES |

### 9.4 纹理上传

```python
# opengl.py L427 upload_texture_data
glBindTexture(GL_TEXTURE_2D, handle)
glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height,
                GL_RGBA, GL_UNSIGNED_BYTE, data)
glBindTexture(GL_TEXTURE_2D, 0)
```

- 像素格式：RGBA8（PSB RGBA 字节顺序，无需 BGRA 转换）
- 纹理参数：GL_LINEAR 过滤 + GL_CLAMP_TO_EDGE 包裹（在 create_render_texture 中设置）

### 9.5 blend mode（opengl.py L796）

12 种 blend mode 映射（对应 asm.js Nx 函数 L12741-12894）：

| bm | blendEquation | blendFunc | 含义 |
|----|---------------|-----------|------|
| 0 | FUNC_ADD | (SRC_ALPHA, ONE_MINUS_SRC_ALPHA, ONE, ONE_MINUS_SRC_ALPHA) | 标准 alpha over |
| 1 | FUNC_ADD | (SRC_ALPHA, ONE, ZERO, ONE) | premultiplied alpha over |
| 2/5 | FUNC_REVERSE_SUBTRACT | (SRC_ALPHA, ONE, ZERO, ONE) | reverse subtract |
| 3 | FUNC_ADD | (DST_ALPHA, ONE_MINUS_SRC_ALPHA, ZERO, ONE) | dst alpha mask |
| 4 | FUNC_ADD | (ONE_MINUS_DST_ALPHA, ONE, ZERO, ONE) | (1-dstA) mask |
| 6 | FUNC_ADD | (ONE_MINUS_DST_COLOR, DST_COLOR, ZERO, ONE) | multiply |
| 7 | FUNC_ADD | (SRC_ALPHA, ONE_MINUS_SRC_ALPHA, ZERO, ONE) | alpha over (alpha=zero) |
| 8 | disable(BLEND) | — | 无混合 |
| 9 | FUNC_ADD | (ONE, ONE_MINUS_SRC_COLOR, ZERO, ONE) | screen/dodge |
| 10 | FUNC_ADD | (ZERO, ONE_MINUS_SRC_COLOR, ZERO, ONE) | darken |
| 11 | FUNC_ADD | (ONE, ZERO, ZERO, ONE) | replace/copy |

### 9.6 shader 源码（shader.py）

**sprite shader**（当前主路径使用）：

```glsl
// vertex
layout(location = 0) in vec2 a_pos;
layout(location = 1) in vec2 a_texCoord;
uniform mat4 u_mvpMat;
uniform vec2 u_spriteOffset;
uniform vec2 u_spriteScale;
out vec2 v_texCoord;
void main() {
    vec2 pos = a_pos * u_spriteScale + u_spriteOffset;
    gl_Position = u_mvpMat * vec4(pos, 0.0, 1.0);
    v_texCoord = a_texCoord;
}

// fragment
in vec2 v_texCoord;
uniform sampler2D u_texUnitId;
out vec4 fragColor;
void main() {
    vec4 tex = texture(u_texUnitId, v_texCoord);
    if (tex.a <= 0.01) discard;
    fragColor = tex;
}
```

**注意**：sprite shader **未使用** `u_spriteOpacity`、`u_filterColor`、
`u_preFilterColor`、`u_meshColorControl`、`u_spriteCoord`、`u_spriteAngle` 等
uniform（虽然在 `_build_drawable_uniforms` / `_build_sprite_uniforms` 中被设置，
但 shader 中未声明，GLSL 编译器会优化掉，`glGetUniformLocation` 返回 -1 时静默跳过）。

**core shader**（part mesh 路径，当前未启用）：

```glsl
// vertex
in vec4 a_pos;
uniform mat4 u_mvpMat;
in vec2 a_texCoord;
uniform vec2 u_texSize;
out vec2 v_texCoord;
in vec4 a_color;
out vec4 v_color;
void main() {
    gl_Position = u_mvpMat * a_pos;
    v_texCoord.x = a_texCoord.x / u_texSize.x;
    v_texCoord.y = (u_texSize.y - a_texCoord.y) / u_texSize.y;  // V 翻转
    v_color = a_color / 255.0;
}

// fragment
in vec2 v_texCoord;
uniform sampler2D u_texUnitId;
uniform float u_testAlpha;
out vec4 fragColor;
void main() {
    vec4 tmp = texture(u_texUnitId, v_texCoord);
    if (tmp.a <= u_testAlpha) discard;
    else fragColor = tmp;
}
```

---

## 10. 与原版的疑似差异

基于代码分析，以下地方可能与 asm.js 原版不一致：

### 10.1 V 翻转不一致（高优先级）

**问题**：
- `loader.py` 的 `_extract_sprites`（L1738）计算 SpriteInfo.uv_coords 时**做了 V 翻转**：
  `uv_coords = (u0, 1.0 - v1, u1, 1.0 - v0)`
- `motion_painter.py` 的 `_make_drawable_resource`（L2573-2576）计算 DrawableResource.uv_coords 时**未做 V 翻转**：
  `uv_coords = (u0, v0, u1, v1)`

**影响**：当前主路径走 `_draw_drawable_resources`（DrawableResource），UV 未翻转，
可能导致纹理上下颠倒。回退路径走 `_draw_sprite_quads`（SpriteInfo），UV 已翻转。
两条路径的 UV 翻转行为不一致。

**与原版关系**：asm.js 原版在 shader 中做 V 翻转（core shader 的
`v_texCoord.y = (u_texSize.y - a_texCoord.y) / u_texSize.y`），但 sprite shader
未做。Python 的 sprite shader 也未做 V 翻转，依赖 CPU 端预翻转。
DrawableResource 路径缺少 CPU 端 V 翻转，可能导致纹理颠倒。

### 10.2 shader 未应用 opacity / color / blend_mode（高优先级）

**问题**：sprite shader 源码中**未声明**以下 uniform：
- `u_spriteOpacity`（不透明度）
- `u_filterColor` / `u_preFilterColor`（顶点颜色调制）
- `u_meshColorControl`（mesh ColorControl 标志）
- `u_spriteCoord`（sprite 坐标）
- `u_spriteAngle`（sprite 旋转角度）

虽然在 `_build_drawable_uniforms`（player.py L2566）和 `_build_sprite_uniforms`
（L2982）中设置了这些 uniform，但 shader 中未使用，`glGetUniformLocation` 返回 -1，
`_set_uniform` 静默跳过。

**影响**：
- **opacity 不生效**：所有 sprite 以全不透明渲染（除非 blend mode 间接影响）
- **color 调制不生效**：`u_filterColor` / `u_preFilterColor` 未应用
- **mesh_cc 不生效**：per-vertex 颜色调制未应用

**与原版关系**：asm.js 原版 shader 完整应用 opacity / color / blend_mode。
Python sprite shader 是简化版，缺失这些功能。

### 10.3 PSB mesh 顶点未解码（已知，中优先级）

**问题**：`MeshData.vertices = b""`、`vertex_count = 0`（loader.py L1508）。
**影响**：part mesh 渲染路径完全不发出 draw call，渲染走实时 Bezier 求值路径。
**与原版关系**：asm.js 原版从 PSB section G 解码 mesh 顶点，走原生 part mesh 渲染
（性能更高）。Python 未解码，走实时计算路径（功能等价但性能较低）。

### 10.4 Bezier 网格密度固定（中优先级）

**问题**：Python 用固定 `BEZIER_GRID_DIVISION = 8`（9×9 = 81 顶点）。
**与原版关系**：asm.js `hu` 函数 L53816-53840 根据 sprite 尺寸和缩放因子**动态**
计算网格密度。Python 用固定值，可能在某些尺寸下精度不足或过度细分。

### 10.5 build_context 三态模型（低优先级，当前不触发）

**问题**：当 `inherit_all_affine=False` 时，Python 实现的状态 2/3 分支可能与 asm.js
不完全一致（代码注释中有多处 TODO(task=12)）。
**当前状态**：所有 59 个 NEKOPARA 模型全用默认 inherit=True，始终走状态 1，差异不触发。

### 10.6 UV 坐标系约定不一致

**问题**：
- `DrawableResource.uv_coords`（motion_painter.py）：V 未翻转（top-down 约定）
- `SpriteInfo.uv_coords`（loader.py）：V 已翻转（bottom-up 约定）
- sprite shader：直接使用 `a_texCoord`，未做 V 翻转
- core shader：做 V 翻转 `v_texCoord.y = (u_texSize.y - a_texCoord.y) / u_texSize.y`

**约定混乱**：不同路径对 V 翻转的处理不一致，可能导致纹理颠倒。

### 10.7 fit_scale 计算可能偏移

**问题**：`_draw_drawable_resources` 中用 `compute_bounds` 计算 fit_scale，
但 `offset_x = screen_w / 2`、`offset_y = screen_h / 2` 固定将 PSB (0,0) 映射到屏幕中心。
**与原版关系**：asm.js 原版用 `coord=[0,0], scale=1`（emoteplayer-format.js L397-399），
不自动 fit。Python 的自动 fit 可能导致缩放与原版不一致。

### 10.8 每帧重建 VBO/IBO

**问题**：`_draw_drawable_resource` 和 `_draw_drawable_resource_grid` 每帧
`create_buffer` + `delete_buffer`，无 VBO 缓存。
**与原版关系**：asm.js 原版可能复用 VBO（待确认）。Python 的每帧重建有性能开销，
但功能正确。

### 10.9 物理摆动角度单位

**问题**：`_EMOTE_ANGLE_TO_RAD = π / 80`（E-mote 内部单位 → 弧度）。
**与原版关系**：已对齐 asm.js gu-pu 物理摆动类型 3（analysis/drawable-detach-root-cause.md §5.3）。
**当前状态**：已修复，与原版一致。

### 10.10 easing table 集成

**问题**：当前所有 NEKOPARA 模型 `root['easing']` 为空列表，easing_table 为空元组，
保持线性插值。part→easing 引用方式未确认（TODO in motion_painter.py L1368-1376）。
**与原版关系**：asm.js `Iq` 函数用 cubic spline 缓动。Python 在 easing_table 非空时
用第一条曲线作为占位，待 part→easing 引用方式确认后改为按 part 属性查表。
**当前状态**：空 easing 时与原版一致（asm.js null easing 指针回退到线性）。

---

## 附录 A：完整数据流图

```
PSB 文件 (.psb / .psb.xor)
  ↓ PSBLoader.load
  ↓ 解析 header + section 偏移表 + body 类型系统
  ↓ XOR 解密（若加密）
  ↓ 节点树 root_value (dict)
  ↓
  ├─→ _extract_emt_textures → list[TextureData]（pixel_data 非空）
  │     ↓ renderer.create_texture + upload_texture_data
  │     → GL 纹理句柄（atlas_handles）
  │
  ├─→ _extract_sprites → tuple[SpriteInfo]（含 mesh_bp、uv_coords V 翻转）
  │     → 供回退路径 _draw_sprite_quads 使用
  │
  └─→ root_value → ModelResource.root_value
        ↓
        EmotePlayer.draw(render_texture_id)
          ↓ renderer.begin_render_target + clear
          ↓ _draw_player_mesh
          ↓   ├─ _draw_drawable_resources（主路径）
          ↓   │    ↓ collect_drawable_resources
          ↓   │    │    ↓ _travel（递归遍历 layer 树）
          ↓   │    │    │    ↓ get_complete_frame_content（找帧 + 帧间插值）
          ↓   │    │    │    │    ↓ read_frame_content → FrameContent（含 mesh_bp）
          ↓   │    │    │    │    ↓ _interpolate_mesh_bp（线性插值 mesh_bp）
          ↓   │    │    │    │    ↓ _interpolate_angle_shortest_path（角度最短路径）
          ↓   │    │    │    ↓ build_context（累积变换 + physics 旋转 + mesh_bp 组合）
          ↓   │    │    │    │    → RenderContext（x/y/z/matrix/mesh_bp/...）
          ↓   │    │    │    ↓ _try_add_resource → _make_drawable_resource
          ↓   │    │    │    │    → DrawableResource（x/y/z/matrix/uv_coords/mesh_bp/...）
          ↓   │    │    ↓ 按 Z 排序
          ↓   │    ↓ 逐资源渲染：
          ↓   │      ├─ mesh_bp 含 32 float → _draw_drawable_resource_grid
          ↓   │      │    ↓ _bezier_patch_eval（双三次 Bezier 曲面求值，81 顶点）
          ↓   │      │    ↓ transform_point（应用累积变换）
          ↓   │      │    ↓ NDC 转换（Y 翻转）
          ↓   │      │    ↓ 打包 pos2_uv2 顶点（16B/顶点）
          ↓   │      │    ↓ create_buffer（VBO + IBO）
          ↓   │      │    ↓ draw_elements（GL_TRIANGLE_STRIP, 158 索引）
          ↓   │      └─ 否则 → _draw_drawable_resource
          ↓   │           ↓ get_corners（4 角屏幕坐标）
          ↓   │           ↓ NDC 转换（Y 翻转）
          ↓   │           ↓ 打包 pos2_uv2 顶点（16B/顶点）
          ↓   │           ↓ create_buffer（VBO + IBO）
          ↓   │           ↓ draw_elements（GL_TRIANGLES, 6 索引）
          ↓   │
          ↓   ├─ _draw_sprite_quads（回退 1，SpriteInfo 路径）
          ↓   └─ _draw_part（回退 2，因 mesh 未解码跳过）
          ↓
          ↓ renderer.end_render_target
```

## 附录 B：坐标系转换汇总

| 阶段 | 坐标系 | X 范围 | Y 范围 | Y 方向 |
|------|--------|--------|--------|--------|
| PSB frame coord | 模型局部 | 任意 | 任意 | 向下 |
| build_context 累积 | 世界（模型） | 任意 | 任意 | 向下 |
| transform_point | 屏幕（像素） | [0, screen_w] | [0, screen_h] | 向下 |
| fit_scale + offset | 屏幕（像素） | [0, screen_w] | [0, screen_h] | 向下 |
| NDC 转换 | NDC | [-1, 1] | [-1, 1] | **向上**（Y 翻转） |
| UV | 纹理 | [0, 1] | [0, 1] | 取决于 V 翻转处理 |
| Bezier 控制点 | sprite 局部归一化 | [0, 1] | [0, 1] | 向下 |

## 附录 C：数据结构字段填充状态

### MeshData（model.py L126）

| 字段 | 当前值 | 状态 |
|------|--------|------|
| vertices | `b""` | ❌ 未解码 |
| indices | `b""` | ❌ 未解码 |
| vertex_count | 0 | ❌ |
| index_count | 0 | ❌ |
| name | `f"mesh_{i}"` | ✅ |
| vertex_layout | `"pos3_uv2"` | ✅ |
| index_format | `""` | ❌ 未确认 |
| division_ratio | None | ❌ |
| ref1_id | -1 | ❌ |

### TextureData（model.py L72）

| 字段 | 当前值 | 状态 |
|------|--------|------|
| width / height | 从 PSB 解析 | ✅ |
| pixel_data | section G chunk data | ✅ |
| pixel_format | `"RGBA8"` | ✅ |
| chunk_index | 从 PSB 解析 | ✅ |
| ref1_id | chunk_index | ✅ |
| truncated_width/height | 从 PSB 解析 | ✅ |

### DrawableResource（motion_painter.py L294）

| 字段 | 来源 | 状态 |
|------|------|------|
| x/y/z | RenderContext 累积 | ✅ |
| matrix | RenderContext 累积 | ✅ |
| origin_x/y | frame.ox + icon_origin_x/y | ✅ |
| opacity | RenderContext 累积 | ✅ |
| icon_width/height | icon 表条目 | ✅ |
| uv_coords | icon 表计算（**V 未翻转**） | ⚠️ |
| texture_index | icon 表条目 | ✅ |
| mesh_bp | RenderContext 累积 | ✅ |
| color | frame content['color'] | ✅ |
| blend_mode | frame content['bm'] | ✅ |
| mesh_cc | frame content['mesh']['cc'] | ✅ |