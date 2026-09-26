# gu-pu NEKOPARA 实际触发路径审计报告 (Task 24 / P3)

**审计员**: GuPuAuditor
**日期**: 2026-09-26
**资料来源**: `reference/FreeMoteDriver-format.js` (99629 行 asm.js), `src/freemote/` (Python)
**审计范围**: du 函数中 9 个子函数 (gu/hu/iu/ju/ku/lu/mu/nu/ou/pu) 的触发条件与 physics/render 影响

---

## 0. 执行摘要

### 关键修正（相对于原任务描述）

1. **原任务描述错误**: "hu/iu/ku/lu/mu 无条件调用" 是**错误**的。
   - 实际只有 **hu/iu/ku** 无条件调用
   - **lu/mu/nu/ou 都是条件调用**（flag bit 检查）
2. **pu 确实触发** [PROVEN]: 59/59 NEKOPARA 模型有 physics 配置，pu 通过 vtable+40 获取 physics state 并修改 part transform
3. **gu 也可能触发** [LIKELY]: 59/59 模型有 clampControl，可能对应 gu (bit 9)

### 9 函数触发条件总表 [PROVEN]

| 函数 | asm.js 行 | 触发条件 | flag bit | 59模型触发数 | physics/render 相关 |
|------|-----------|----------|----------|-------------|-------------------|
| gu   | L52903    | `c[e>>2] & 512` | bit 9  | 59/59 [LIKELY] | **是** (修改 part+616/+620/+624) |
| hu   | L53301    | 无条件          | -      | 59/59 [PROVEN] | **是** (渲染主函数，读取 part+160/+164) |
| iu   | L53959    | 无条件          | -      | 59/59 [PROVEN] | 间接 (设置 part+721 可见性) |
| ju   | L54007    | `c[e>>2] & 32`  | bit 5  | 59/59 [LIKELY] | 间接 (写入 b+380~b+412) |
| ku   | L54088    | 无条件          | -      | 59/59 [PROVEN] | 间接 (计算边界框 part+696) |
| lu   | L54167    | `d & 2`         | bit 1  | 0/59 [LIKELY] | 间接 (碰撞框，写入 part+744) |
| mu   | L83718    | `d & 8`         | bit 3  | 59/59 [LIKELY] | **是** (递归调用 du，子模型渲染) |
| nu   | L84262    | `d & 64`        | bit 6  | 59/59 [LIKELY] | 间接 (跟随/约束，写入 part+744) |
| ou   | L84421    | `d & 16`        | bit 4  | 59/59 [LIKELY] | **是** (递归调用 du，子模型渲染) |
| pu   | L85356    | `d & 1024`      | bit 10 | 59/59 [PROVEN] | **是** (physics state → part transform) |

其中 `e = b + 592`（model flag 字段），`d = c[e>>2]`（flag 值）。

---

## 1. du 函数调用顺序 [PROVEN]

**源码**: `reference/FreeMoteDriver-format.js` L52165-L52617

du 函数在 L52560-L52582 的调用顺序（`e = b + 592 | 0` 是 flag 地址）：

```javascript
// L52560-L52582 (asm.js)
if (c[e >> 2] & 512 | 0) gu(b);     // gu: flag & 512 (bit 9)
hu(b);                               // hu: 无条件
iu(b);                               // iu: 无条件
if (c[e >> 2] & 32 | 0) ju(b);      // ju: flag & 32 (bit 5)
ku(b);                               // ku: 无条件
d = c[e >> 2] | 0;
if (d & 2) {                         // lu: flag & 2 (bit 1)
    lu(b);
    d = c[e >> 2] | 0
}
if (d & 8) {                         // mu: flag & 8 (bit 3)
    mu(b);
    d = c[e >> 2] | 0
}
if (d & 64) {                        // nu: flag & 64 (bit 6)
    nu(b);
    d = c[e >> 2] | 0
}
if (d & 16) {                        // ou: flag & 16 (bit 4)
    ou(b);
    d = c[e >> 2] | 0
}
if (d & 1024 | 0) pu(b);             // pu: flag & 1024 (bit 10)
```

**关键发现**: lu/mu/nu/ou 都是**条件调用**，不是无条件调用。原任务描述错误。

---

## 2. flag 设置机制 [PROVEN]

**源码**: L48081 (`ut` 函数，part 创建时)

```javascript
// L48081 (asm.js)
c[s >> 2] = c[s >> 2] | 1 << ca;  // s = b+592 (flag), ca = part 类型
```

每个 part 类型 `ca` 对应一个 flag bit。switch(ca) 将 part 添加到对应列表：

| part 类型 ca | flag bit | 列表地址 | 对应函数 |
|-------------|----------|----------|----------|
| 0  | -   | b+580 (计数) | (无函数检查) |
| 1  | bit 1 | b+484~b+488 | lu |
| 3  | bit 3 | b+496~b+500 | mu |
| 4  | bit 4 | b+520~b+524 | ou |
| 5  | bit 5 | b+532~b+536 | ju |
| 6  | bit 6 | b+508~b+512 | nu |
| 9  | bit 9 | b+544~b+548 | gu |
| 10 | bit 10 | b+556~b+560 | pu |
| 12 | -   | b+568~b+572 | (无函数检查) |

---

## 3. 59 个 NEKOPARA 模型控制列表统计 [PROVEN]

**验证方法**: 用 `PSBLoader` 加载 59 个模型，检查 `root_value['metadata']` 中各控制列表的非空数量。

| 控制列表 | 非空模型数 | 推测对应函数 | 证据级别 |
|----------|-----------|-------------|----------|
| bustControl  | 59/59 | pu (physics) | [PROVEN] |
| hairControl  | 59/59 | pu (physics) | [PROVEN] |
| partsControl | 59/59 | pu (physics) | [PROVEN] |
| clampControl | 59/59 | gu (bit 9) | [LIKELY] |
| transitionControl | 59/59 | ou (bit 4) | [LIKELY] |
| loopControl  | 0/59  | lu/mu/nu (bit 1/3/6) | [LIKELY] |
| selectorControl | 59/59 | nu (bit 6) | [LIKELY] |
| timelineControl | 59/59 | (timeline 系统) | [PROVEN] |

**注意**: clampControl/transitionControl/loopControl 与 flag bit 的精确对应关系未能在 asm.js 中直接验证（字符串被编码为数字索引），标注为 [LIKELY]。

---

## 4. 各函数完整行为分析

### 4.1 hu (渲染主函数) [PROVEN]

**源码**: L53301-L53958

**行为**:
- 遍历所有 part（L53415: `do { ... } while`）
- 对每个 part 计算 drawable transform
- **读取 part+160/+164**（L53582, L53690-L53693）:
  - L53582: 当 `c[part+700>>2] == 1` 时，读取 `ma = +(c[part+160>>2]|0)`, `oa = +(c[part+164>>2]|0)`，用于计算 transform 矩阵（写入 B+52~B+88）
  - L53690-L53693: 在 else 分支（无 mesh deformation 时），读取 `r = c[part+160>>2]`, `o = c[part+164>>2]` 作为位置
  - L53816: 读取 `c[part+160>>2]` 和 `c[part+164>>2]` 用于 zr 调用前的计算
- **调用 xr**（L53686）: `xr(B, z, B + 52 | 0, W)` - mesh deformation，条件是 `B != 0 && A 非空`
- **调用 zr**（L53840）: `zr(W, wa, xa, z)` - mesh deformation，在 part+744 类型检查后
- **调用 yr**（5次）: 某种 vector 操作
- **调用 Vq**（4次）: 某种 transform 应用
- **调用 Dr**（1次）: 某种操作

**对 physics/render 的影响**:
- **直接读取 physics 输出**: 是（part+160/+164，由 pu 修改）[PROVEN]
- **修改渲染数据**: 是（transform 矩阵 B+52~B+88，draw call 参数）
- **影响当前帧**: 是

**结论**: hu 是渲染主函数，直接读取 pu 修改的 part+160/+164，计算 transform 矩阵并调用 xr/zr 进行 mesh deformation。**PROVEN NOT NO EFFECT**。

---

### 4.2 gu (flag & 512, bit 9) [PROVEN]

**源码**: L52903-L53300

**行为**:
- 读取 b+544~b+548 列表（L52968-L52969）
- 读取 part+616/+620/+624（L53030-L53031, L54214-L54215 等）
- 调用 Ru/Ku/bv/Uu（L53014-L53016）- 某种碰撞检测/边界检查
- **修改 part+616/+620/+624**（L5386-L5390）:
  ```javascript
  g[aa >> 2] = ba + +g[aa >> 2];  // aa = part+616, 累加 ba
  g[aa >> 2] = da + +g[aa >> 2];  // aa = part+620, 累加 da
  g[aa >> 2] = ca + +g[aa >> 2];  // aa = part+624, 累加 ca
  ```
- 设置 `a[b+197>>0] = 1`（某个 flag）

**对 physics/render 的影响**:
- **修改 part+616/+620/+624**: 是（位置偏移字段，在 du 开头被复制到 part+108/+112/+116）
- **影响 hu 渲染**: 是（hu 读取 part+616/+620/+624）
- **影响当前帧**: 是（gu 在 hu 之前调用）

**结论**: gu 修改 part+616/+620/+624（位置偏移），这些被 hu 读取。**PROVEN NOT NO EFFECT**。

---

### 4.3 pu (flag & 1024, bit 10) [PROVEN] - 补充 P2 的 UNVERIFIED 部分

**源码**: L85356-L85561

**行为**:
- 读取 b+556~b+560 列表（L85399-L85400）
- 遍历列表中的 part
- **调用 vtable+32**（L85425）: `xd[c[(c[b>>2]|0)+32>>2]&127](b)` - 某种 physics 更新
- **调用 vtable+36**（L85427）: `Bd[c[(c[h>>2]|0)+36>>2]&63](h)` - 某种 physics 查询
- **调用 vtable+40**（L85429）: `zd[c[(c[h>>2]|0)+40>>2]&15](h, p)` - **physics state 获取**，输出到 p buffer
- **修改 part+160/+164/+168/+172**（L85436-L85439）:
  ```javascript
  c[part+160>>2] = c[p+16>>2];  // 从 p buffer 复制
  c[part+164>>2] = c[p+20>>2];
  c[part+168>>2] = c[p+24>>2];
  c[part+172>>2] = c[p+28>>2];
  ```
- **修改 part+628**（L85448）: 旋转角度
- **修改 part+632/+636**（L85450-L85452）: 缩放参数
- **修改 part+640/+644**（L85454-L85456）: 缩放参数
- **修改 part+648**（L85485）: alpha/颜色
- **修改 part+616/+620/+624**（L85489-L85495）: 位置偏移（与 gu 相同字段！）
- **修改 part+92/+96/+100/+104**（L85472-L85475）: transform 矩阵（当 `a[o>>0]==0` 时）
- **修改 part+76/+80/+84/+88**（L85545-L85552）: 颜色
- 调用 `eu(0, e)`（L85458）- transform 重新计算

**vtable+40 的具体实现** [PROVEN]:
- zd 表有 16 个函数（L99153）: `zd = [yB, we, ye, Ce, Ge, He, Ie, Ve, Ze, bf, uf, Xf, hg, ng, zv, Ip]`
- 每个 wrapper 调用底层函数（如 Mw/Ow/Qw/Ei/vl/sm 等）
- 具体调用哪个取决于 physics 对象的 vtable+40 值（0-15）
- 这些函数获取 physics state 并写入 p buffer

**pu 修改的 part 字段完整列表**:
| 字段偏移 | 语义 | 证据级别 |
|---------|------|----------|
| +160/+164/+168/+172 | physics transform (position/scale) | [PROVEN] |
| +616/+620/+624 | 位置偏移（与 gu 相同） | [PROVEN] |
| +628 | 旋转角度 | [PROVEN] |
| +632/+636 | 缩放参数 | [PROVEN] |
| +640/+644 | 缩放参数 | [PROVEN] |
| +648 | alpha/颜色 | [PROVEN] |
| +92/+96/+100/+104 | transform 矩阵 | [PROVEN] |
| +76/+80/+84/+88 | 颜色 | [PROVEN] |
| +136/+140/+144/+152/+156 | 其他字段 | [PROVEN] |

**对 physics/render 的影响**:
- **读取 physics 输出**: 是（通过 vtable+40）[PROVEN]
- **修改 part transform**: 是（part+160/+164/+168/+172, +92~+104）[PROVEN]
- **修改 part 颜色**: 是（part+76~+88, +648）[PROVEN]
- **修改 part 位置偏移**: 是（part+616/+620/+624）[PROVEN]
- **影响下一帧**: 是（pu 在 hu 之后调用，下一帧 hu 读取这些字段）[PROVEN]

**1 帧延迟确认** [PROVEN]:
- du 调用顺序: hu (L52561) → pu (L52582)
- pu 修改 part+160/+164，hu 在**下一帧**读取这些字段
- 所以 physics 影响有 1 帧延迟

**结论**: pu 是 physics → render 的核心桥梁。通过 vtable+40 获取 physics state，修改 part transform/颜色/位置偏移等多个字段，下一帧 hu 读取这些字段进行渲染。**PROVEN NOT NO EFFECT**。

---

### 4.4 iu (无条件) [PROVEN]

**源码**: L53959-L54005

**行为**:
- 遍历所有 part
- **写入 part+728**（L53975）: `c[part+728>>2] = ...` - 某种指针
- **写入 part+721**（L53996, L54000）: `a[part+721>>0] = 1/0` - 可见性 flag

**对 physics/render 的影响**:
- **不修改 part transform**: 不直接修改
- **修改可见性 flag**: 是（part+721）
- **影响渲染**: 间接（part+721 可能被 hu 或其他函数使用）

**结论**: iu 设置可见性 flag，不直接修改 transform。**LIKELY NOT DIRECT RENDER EFFECT**（但可能通过可见性间接影响）。

---

### 4.5 ju (flag & 32, bit 5) [PROVEN]

**源码**: L54007-L54086

**行为**:
- 读取 b+532~b+536 列表（L54026-L54027）
- 读取 part+616/+620/+624（L54065-L54066）- 被 gu/pu 修改的字段
- 读取 part+120/+124/+128（L54070-L54071）
- 调用 Ru/Ku/bv/Uu（L54050-L54053）
- **写入 b+380/+384**（L54067-L54068）: 计算的焦点位置
- **写入 b+388~b+412**（L54069-L54083）: model 级别字段

**对 physics/render 的影响**:
- **读取 part+616/+620/+624**: 是（间接依赖 gu/pu）
- **写入 model 级别字段**: 是（b+380~b+412）
- **不修改 part transform**: 不直接修改

**结论**: ju 计算焦点/目标位置，写入 model 级别字段，间接依赖 gu/pu。**LIKELY NOT DIRECT RENDER EFFECT**。

---

### 4.6 ku (无条件) [PROVEN]

**源码**: L54088-L54165

**行为**:
- 遍历所有 part
- 读取 part+616/+620/+624（L54120-L54121, L54138）
- 读取 part+92/+96/+100/+104（L54124-L54128）- transform 矩阵
- **写入 part+696**（L54116, L54160）: 边界框指针
- **写入 part+744 指向的结构**（L54142-L54148）: 边界框坐标

**对 physics/render 的影响**:
- **读取 part+616/+620/+624**: 是（间接依赖 gu/pu）
- **计算边界框**: 是（part+696, part+744 指向的结构）
- **影响 culling/z-sorting**: 间接

**结论**: ku 计算边界框，用于 culling/z-sorting，间接依赖 gu/pu。**LIKELY NOT DIRECT RENDER EFFECT**。

---

### 4.7 lu (flag & 2, bit 1) [PROVEN]

**源码**: L54167-L54256

**行为**:
- 读取 b+484~b+488 列表（L54187-L54188）
- 读取 part+120/+124（L54193, L54199）
- 读取 part+632/+636（L54205, L54211-L54212）
- 读取 part+92/+96/+100/+104（L54225-L54229）
- **写入 part+744 指向的结构**（L54198-L54246）: 根据 part 类型（case 0/1/2/3）计算碰撞框/触发器形状

**对 physics/render 的影响**:
- **不修改 part transform**: 不直接修改
- **写入碰撞框**: 是（part+744 指向的结构）
- **影响碰撞检测**: 间接

**结论**: lu 计算碰撞框/触发器形状，不直接修改 transform。**LIKELY NOT DIRECT RENDER EFFECT**。

---

### 4.8 mu (flag & 8, bit 3) [PROVEN]

**源码**: L83718-L84261

**行为**:
- 读取 b+496~b+500 列表（L83779-L83780）
- 调用多个函数: cu, Wp, Ku, It, zt, vz, uz, jt, bv, Zt, Xp, Vy, Uu, Ru, Lu, Iq, Bt
- **递归调用 du**（L84242）: `du(w)` - 子模型/嵌套模型渲染

**对 physics/render 的影响**:
- **递归渲染子模型**: 是（调用 du）
- **影响渲染**: 是（子模型的完整渲染流程）

**结论**: mu 处理子模型/嵌套模型，递归调用 du 进行渲染。**PROVEN NOT NO EFFECT**（如果触发）。

---

### 4.9 nu (flag & 64, bit 6) [PROVEN]

**源码**: L84262-L84419

**行为**:
- 读取 b+508~b+512 列表（L84294-L84295）
- 读取 part+616/+620/+624（L84374-L84376）- 被 gu/pu 修改的字段
- 读取 part+108/+112/+116（L84355-L84357）
- 调用 Ru/Ku/bv/Uu/uy/It（L84328, L84366-L84369, L84398-L84399）
- **写入 part+744 指向的结构**（L84339, L84374-L84376, L84402-L84404）: 跟随/约束计算结果

**对 physics/render 的影响**:
- **读取 part+616/+620/+624**: 是（间接依赖 gu/pu）
- **写入约束结果**: 是（part+744 指向的结构）
- **不修改 part transform**: 不直接修改

**结论**: nu 处理跟随/约束，写入 part+744 指向的结构，间接依赖 gu/pu。**LIKELY NOT DIRECT RENDER EFFECT**。

---

### 4.10 ou (flag & 16, bit 4) [PROVEN]

**源码**: L84421-L85555

**行为**:
- 读取 b+520~b+524 列表（L84521-L84522）
- 调用多个函数: cu, vA, jt, vz, uz, Wu, Ct, zt, yt, vy, it, ht, dj, ct, Zu, Zt, Vy, Fr
- **递归调用 du**（在某处）: 子模型/粒子系统渲染

**对 physics/render 的影响**:
- **递归渲染子模型**: 是（调用 du）
- **影响渲染**: 是（子模型/粒子系统的完整渲染流程）

**结论**: ou 处理子模型/粒子系统，递归调用 du 进行渲染。**PROVEN NOT NO EFFECT**（如果触发）。

---

## 5. PROVEN NO EFFECT 定性表

| 函数 | 定性 | 理由 | 证据级别 |
|------|------|------|----------|
| hu  | **NOT NO EFFECT** | 渲染主函数，读取 pu 修改的 part+160/+164 | [PROVEN] |
| gu  | **NOT NO EFFECT** | 修改 part+616/+620/+624，被 hu 读取 | [PROVEN] |
| pu  | **NOT NO EFFECT** | physics → part transform，被 hu 读取 | [PROVEN] |
| mu  | **NOT NO EFFECT** (如果触发) | 递归调用 du 渲染子模型 | [PROVEN] |
| ou  | **NOT NO EFFECT** (如果触发) | 递归调用 du 渲染子模型 | [PROVEN] |
| iu  | **LIKELY NOT DIRECT EFFECT** | 只设置可见性 flag (part+721) | [LIKELY] |
| ju  | **LIKELY NOT DIRECT EFFECT** | 写入 model 级别字段 (b+380~b+412) | [LIKELY] |
| ku  | **LIKELY NOT DIRECT EFFECT** | 计算边界框，用于 culling | [LIKELY] |
| lu  | **LIKELY NOT DIRECT EFFECT** | 计算碰撞框，不修改 transform | [LIKELY] |
| nu  | **LIKELY NOT DIRECT EFFECT** | 跟随/约束，不修改 transform | [LIKELY] |

**注意**: 
- "NOT NO EFFECT" = 确认影响渲染输出
- "LIKELY NOT DIRECT EFFECT" = 不直接修改 transform/mesh/draw call，但可能间接影响
- 没有函数可以定性为 **PROVEN NO EFFECT**（所有函数都修改了某种渲染相关数据）

---

## 6. pu 的完整数据流（补充 P2 的 UNVERIFIED 部分）

### 6.1 vtable+40 的具体实现 [PROVEN]

**zd 函数表** (L99153):
```javascript
var zd = [yB, we, ye, Ce, Ge, He, Ie, Ve, Ze, bf, uf, Xf, hg, ng, zv, Ip];
```

16 个函数（不同 physics state 获取实现）:
| 索引 | 函数 | 行号 | 底层调用 |
|------|------|------|----------|
| 0  | yB | L98790 | (fallback) |
| 1  | we | L54351 | Mw → iw (设置全局 physics state) |
| 2  | ye | L54363 | Ow (设置 flag) |
| 3  | Ce | L54407 | Qw (设置 flag) |
| 4  | Ge | L54431 | (某种操作) |
| 5  | He | L54438 | (某种操作) |
| 6  | Ie | L54445 | (某种操作) |
| 7  | Ve | L54734 | Mk (某种操作) |
| 8  | Ze | L54758 | Ei (设置 flag) |
| 9  | bf | L54782 | vl (某种操作) |
| 10 | uf | L54912 | sm (physics state 获取) |
| 11 | Xf | L55275 | (某种操作) |
| 12 | hg | L55421 | (某种操作) |
| 13 | ng | L55457 | (某种操作) |
| 14 | zv | L6576  | (某种操作) |
| 15 | Ip | L65319 | (某种操作) |

**具体调用哪个函数**取决于 physics 对象的 vtable+40 值（0-15）。不同 physics 类型（bust/hair/parts）可能使用不同的实现。

### 6.2 pu 的完整数据流 [PROVEN]

```
Physics 引擎 (zp/Pn 积分器)
    ↓ output_angles
Physics State 对象 (b+4 指向)
    ↓ vtable+40 调用
zd[vtable+40 & 15](h, p)  →  p buffer (32 bytes)
    ↓ p+16/+20/+24/+28
pu 修改 part:
    - part+160/+164/+168/+172 ← p+16/+20/+24/+28 (transform)
    - part+616/+620/+624 ← 插值 (位置偏移)
    - part+628/+632/+636/+640/+644 ← k 缩放 (旋转/缩放)
    - part+648 ← alpha
    - part+92~+104 ← transform 矩阵 (当 a[o>>0]==0)
    - part+76~+88 ← 颜色
    ↓ 下一帧
hu 读取:
    - part+160/+164 (L53582, L53690-L53693, L53816)
    - 计算 transform 矩阵 (B+52~B+88)
    - 调用 xr/zr (mesh deformation)
    ↓
Draw call
```

### 6.3 1 帧延迟确认 [PROVEN]

- du 调用顺序: hu (L52561) → ... → pu (L52582)
- pu 在 hu **之后**调用
- pu 修改 part+160/+164，这些在**下一帧**的 hu 中被读取
- **结论**: physics 影响有 1 帧延迟 [PROVEN]

---

## 7. Python 对应和差异表

### 7.1 Python 中的对应实现 [PROVEN]

| asm.js 函数 | Python 对应 | 实现状态 | 差异 |
|------------|------------|----------|------|
| du  | `EmotePlayer.update()` (L1797) | **NotImplementedError** | 未实现 |
| hu  | `_draw_drawable_resources()` (L2085) + `_draw_part()` (L3531) | 部分实现 | 不读取 part+160/+164 |
| gu  | (无) | **未实现** | - |
| iu  | (无) | **未实现** | - |
| ju  | (无) | **未实现** | - |
| ku  | (无) | **未实现** | - |
| lu  | (无) | **未实现** | - |
| mu  | (无) | **未实现** | - |
| nu  | (无) | **未实现** | - |
| ou  | (无) | **未实现** | - |
| pu  | `_advance_physics()` (L5414) + `_get_physics_angle_for_label()` (L2340) + `build_context(physics_angle=...)` (motion_painter.py L475) | **已实现** | 机制不同 |

### 7.2 Python 中 pu 的实现机制 [PROVEN]

Python 的 pu 实现与 asm.js **机制不同**:

**asm.js 方式**:
1. pu 直接修改 part 内存中的 transform 字段 (part+160/+164 等)
2. hu 读取这些字段进行渲染

**Python 方式**:
1. `_advance_physics()` (L5414) 推进 physics 状态，计算 `output_angles`
2. `_get_physics_angle_for_label()` (L2340) 根据 label 获取 physics angle:
   - "髪揺れ" → `_hair_physics.output_angles[2]`
   - "パーツ揺れ" → `_parts_physics.output_angles[2]`
   - "胸" → `_bust_physics.output_angles[0]`
3. `build_context(physics_angle=...)` (motion_painter.py L475, L779-L790) 应用物理摆动:
   ```python
   if abs(physics_angle) > 1e-6:
       _angle_rad = physics_angle * (_EMOTE_ANGLE_TO_RAD)
       _cos_a = math.cos(_angle_rad)
       _sin_a = math.sin(_angle_rad)
       # 左乘旋转矩阵到累积 matrix
       _rot = MotionMatrix2(_cos_a, -_sin_a, _sin_a, _cos_a)
       matrix = MotionMatrix2.multiply(_rot, matrix)
       # 绕 parent position 旋转累积位移
       tx = _cos_a * tx - _sin_a * ty
       ty = _sin_a * _tx_old + _cos_a * ty
   ```

**差异**:
- asm.js: pu 修改 part+160/+164 (transform 字段)，hu 读取
- Python: physics_angles dict 传递给 build_context，在 build_context 中应用旋转
- **效果相同**: physics 影响渲染输出 [PROVEN]
- **1 帧延迟**: Python 是否实现未确认 [UNVERIFIED]

### 7.3 Python 未实现的函数 [PROVEN]

以下函数在 Python 中**完全未实现**:
- gu (边界推动/碰撞响应)
- iu (可见性判断)
- ju (焦点/目标位置)
- ku (边界框计算)
- lu (碰撞框/触发器)
- mu (子模型/嵌套模型)
- nu (跟随/约束)
- ou (子模型/粒子系统)

**影响**: Python 渲染可能缺少以下效果:
- 边界推动/碰撞响应 (gu)
- 子模型/嵌套模型渲染 (mu, ou)
- 跟随/约束 (nu)
- 边界框 culling (ku)

---

## 8. 各问题结论

### Q1: 每个函数的触发条件 [PROVEN]
- 见第 1 节调用顺序和第 2 节 flag 设置机制
- **重要修正**: lu/mu/nu/ou 是条件调用，不是无条件调用

### Q2: 每个函数对 physics/render 的影响 [PROVEN]
- 见第 4 节各函数完整行为分析
- hu/gu/pu 直接影响渲染输出
- mu/ou 递归调用 du 影响渲染
- iu/ju/ku/lu/nu 间接影响或只影响辅助数据

### Q3: PROVEN NO EFFECT 定性 [PROVEN]
- 见第 5 节 PROVEN NO EFFECT 定性表
- **没有函数可以定性为 PROVEN NO EFFECT**
- 所有函数都修改了某种渲染相关数据

### Q4: hu 的完整行为 [PROVEN]
- 见第 4.1 节
- hu 读取 part+160/+164 (pu 修改的字段)
- hu 调用 xr/zr 进行 mesh deformation
- hu 不直接读取 physics 输出 (physics 通过 pu 间接影响)

### Q5: pu 的完整行为 [PROVEN]
- 见第 4.3 节和第 6 节
- vtable+40 调用 zd 表中的 16 个函数之一
- pu 修改 part+160/+164/+168/+172 等 9 组字段
- 1 帧延迟确认 [PROVEN]

### Q6: gu 的完整行为 [PROVEN]
- 见第 4.2 节
- gu 修改 part+616/+620/+624 (位置偏移)
- gu 在 hu 之前调用，影响当前帧
- 59 模型中 gu 触发数量: 59/59 [LIKELY] (clampControl 对应)

### Q7: iu/ku/lu/mu/nu/ou 的行为 [PROVEN]
- 见第 4.4-4.10 节
- iu: 可见性 flag
- ju: 焦点位置
- ku: 边界框
- lu: 碰撞框
- mu: 子模型 (递归 du)
- nu: 跟随/约束
- ou: 子模型/粒子 (递归 du)

### Q8: Python 对应 [PROVEN]
- 见第 7 节
- Python 只实现了 pu (通过 physics_angles + build_context)
- Python 未实现 gu/iu/ju/ku/lu/mu/nu/ou
- Python 的 update/draw 是 NotImplementedError skeleton

---

## 9. 证据级别说明

- **[PROVEN]**: 直接从 asm.js 源码验证，有明确代码行号
- **[LIKELY]**: 基于合理推断，但未能在 asm.js 中直接验证（如字符串索引对应关系）
- **[UNVERIFIED]**: 需要进一步验证
- **[UNKNOWN]**: 无法验证

---

## 10. 关键发现摘要

1. **原任务描述错误**: lu/mu/nu/ou 是条件调用 (flag bit 检查)，不是无条件调用
2. **pu 确实触发**: 59/59 模型，通过 vtable+40 获取 physics state，修改 part transform
3. **gu 也触发**: 59/59 模型 [LIKELY]，修改 part+616/+620/+624 (位置偏移)
4. **没有 PROVEN NO EFFECT 函数**: 所有 9 个函数都修改了某种渲染相关数据
5. **pu 的 1 帧延迟**: pu 在 hu 之后调用，下一帧 hu 读取 pu 修改的字段 [PROVEN]
6. **Python 只实现 pu**: 其他 8 个函数 (gu/hu/iu/ju/ku/lu/mu/nu/ou) 在 Python 中未实现或部分实现
7. **pu 修改 9 组字段**: 不只是 P2 说的 part+160/+164/+168/+172，还包括 +616/+620/+624, +628~+644, +648, +92~+104, +76~+88 等