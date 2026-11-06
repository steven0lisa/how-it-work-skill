# Kit API

所有内容都在 `kit/iso-kit.ts` 中，`kit/iso-kit.mjs` 提供相同的 API。世界坐标为 `[x, y, z]`，x 与 y 在地面上，z 向上，单位约为 1 px。屏幕坐标为 SVG viewBox 中的 `[sx, sy]`。路径以 SVG `d` 字符串返回，取整到 0.1。

## 目录
1. 类型
2. 相机
3. 实体
4. 面上的线
5. 零件库
6. 移动物体
7. SVG 字符串（不依赖框架）
8. 卡片与页面
9. React 组件
10. 任意轴上的圆形零件（`kit/lathe.mjs`）
11. 管道、管件与线缆（`kit/tube.mjs`）
12. 审计（`kit/audit.mjs`）
13. WebGL 层（`kit/gl.mjs`）
14. 工具
15. 旋转组（`kit/turn.mjs`、`turn-build.mjs`、`turn-audit.mjs`）

## 1. 类型
```ts
type Flat = [number, number];            // screen point
type Point3 = [number, number, number];  // world point
type Plane = [number, number];           // floor point (x, y)
interface Plan { x: number; y: number; w: number; d: number; r: number }   // rounded rectangle on the floor
interface Projection { origin: Flat; scale: number; azimuth?: number; elevation?: number }
interface SolidPaths { fill; outline; crease; top; shades: string[4]; bevel?; seamless? }
type Tone = "hi" | "mid" | "lo" | "faint";
type LineTone = Tone | "lit" | "red" | "green" | "blue";
```

## 2. 相机
| 函数 | 作用 |
|---|---|
| `fitProjection(points: Point3[], width, height, { pad = 24, headroom = 0, azimuth = 45, elevation = 30 })` | 一个把这组点居中并适配进 viewBox 的投影（projection）。传入每个极值点，例如 `boxCorners(plan, z0, z1)` 与最高的尖端。 |
| `iso(point, P)` | 世界坐标转屏幕坐标。 |
| `depthOf(point, P)` | 数值越大表示离观察者越近。升序排序即可从后往前绘制。 |
| `towardViewer(P)` | 指向观察者的地面方向 `[cos a, sin a]`。法线朝此方向的面可见。 |
| `screenRise(h, P)` | 高度 `h` 覆盖多少屏幕 px。 |
| `cameraOf(P)` | 原始的正弦与余弦值。 |

## 3. 实体
所有实体（solid）都是棱柱（prism）：把地面轮廓竖直向上拉伸（extrude）而成。

| 函数 | 作用 |
|---|---|
| `slabOf(plan, z, h, P, steps = 8, bevel = 0)` | 一个圆角盒子。`steps` 是每个圆角处的点数（小零件用 2–4，大圆角件用 8–10）。`bevel`（倒角，0.3–1.6）会在顶面沿可见边缘内缩加一条高光线，一直延伸到轮廓（silhouette）角点。 |
| `cylinder(cx, cy, r, z, h, P, steps = 32, bevel = 0)` | 圆柱形的立柱、圆饼、支脚、旋钮或垫块。小件用 16–20 个 steps，大圆盘用 56–72。 |
| `extrude(ring: Plane[], z, h, P, { convex = true, bevel })` | 任意地面轮廓。凹轮廓传 `convex: false`（填充变为逐面而非凸包）。 |
| `ringOf(plan, steps)` | 以点表示的圆角矩形轮廓。 |
| `roundedPlan(plan, steps)` | 同一功能。 |
| `circleRing(cx, cy, r, steps)` | 以点表示的圆形轮廓。 |
| `insetPlan(plan, by)` | 缩小一个 plan（传负 `by` 则放大）。圆角半径保持合理。 |
| `corners(plan, inset)` | 从四角内缩得到的四个地面点，用于支脚与螺钉。 |
| `boxCorners(plan, z0, z1)` | 八个角点，供 `fitProjection` 使用。 |
| `haloOf(plan, z, h, P)` | 柔影（halo，软阴影）的填充路径：`<path class="iso-halo" d=…>`。 |

`SolidPaths` 包含的内容：
- `fill`：纸色，遮住其后方的一切。
- `shades[0..3]`：可见侧面，按其朝光的程度分桶。
- `top`：顶面，更亮。
- `crease`：顶面与可见侧面相接之处（折痕，crease）。
- `bevel`：可选。
- `outline`：轮廓。

`solidSvg` 与 `<Solid>` 按该顺序绘制它们。

## 4. 面上的线
| 函数 | 绘制内容 |
|---|---|
| `onTop(points: Plane[], z, P, closed?)` | 水平面上的折线。 |
| `lineOnTop(a, b, z, P)` | 水平面上的一条线段。 |
| `planOutline(plan, z, P, steps)` | 面上的圆角矩形轮廓，用于凹槽与镶嵌（`insetPlan`）。 |
| `ring(cx, cy, r, z, P, steps)` | 面上的圆，用于螺钉圈、表圈（bezel）与落座（seats）。 |
| `segment(a, b, P)` | 任意三维线段。 |
| `rise(at, z0, z1, P)` | 竖直线段。 |
| `pointsAt(points, P)` | 投影一组点，用于螺钉与铆钉圆点。 |
| `dotGrid(x, y, cols, rows, gap, z, P)` | 投影后的穿孔点阵。 |
| `topMatrix(at, z, P, along = "x")` | 把文字平铺在顶面上的变换，供 `faceTextSvg`/`FaceText` 使用。 |
| `sideMatrix(at, z, P, "left" \| "right")` | 同上，用于竖直面。 |

## 5. 零件库
| 函数 | 返回 |
|---|---|
| `knurl(cx, cy, r, z0, z1, count, P, inset = 0.4)` | 圆柱可见半侧上的竖直刻线（滚花，knurl）。 |
| `sideArc(cx, cy, r, z, P, steps)` | 圆柱侧面上水平圆的可见前半弧：接缝（seam）与环带。 |
| `sideSeam(plan, z, P, steps = 8)` | 盒子的对应版本：高度 `z` 处圆角 plan 的可见近半圈，包住各圆角并恰好终止于板件的竖直轮廓边上。传入板件自己的 `steps`，并把它作为板件的 `inner` 绘制。这是在盒子上画焊道、环带、盖缝或饰条的唯一正确方式：绝不要用从角点内缩的 `segment`。 |
| `ringSeam(ring, z, P)` | 任意地面轮廓版的 `sideSeam`（即传给 `extrude` 的 ring）。 |
| `sideRing(center: Point3, r, "x" \| "y", P)` | 竖直面上的圆：孔口、杆端、透镜。 |
| `coil(from, to, [cy, cz], r, turns, "x" \| "y", P)` | `{ back, front }` 弹簧（spring）两半，可按 后半 → 杆 → 前半 的顺序绘制。 |
| `topTicks(from, to, step, every, edge, z, [minor, major], P, axis)` | 顶面上的 `{ minor, major }` 刻度线，从线 `edge` 起沿 +y（或 +x）生长。 |
| `sideTicks(from, to, step, every, face, top, [minor, major], P, axis)` | 沿竖直面向下垂挂的刻度线。 |
| `radialTicks(cx, cy, r, count, every, z, [minor, major], P, from = 0, span = 2π)` | 表盘与转盘刻度。角度 0 指向 −y，俯视时顺时针增大。 |

## 6. 移动物体
| 函数 | 作用 |
|---|---|
| `translateAlong(P, [dx, dy, dz])` | 一个按世界偏移移动预构建组的 `"translate(x y)"` 字符串。用于滑架、抽屉、柱塞与升降件。 |
| `holdStill(element, [dx, dy])` | 给 `element` 内每个 `[data-hold]` 元素施加相反的位移，使其所在组移动 `[dx, dy]` 屏幕单位时，裁剪（clip）保持原位。移动持有 `G.rodOut` 裁剪的条目时，用相同偏移调用它。 |
| `axisVector(P, [1, 0, 0])` | 一个世界单位对应的屏幕向量。用于把指针移动换算成世界移动。 |

绕 z 轴的旋转无法用屏幕变换完成：每一帧或每一步都要重建旋转后的路径。对表针、指针或小零件来说这很便宜。

## 7. SVG 字符串（不依赖框架）
| 函数 | 说明 |
|---|---|
| `solidSvg(paths, { tone = "hi", crease = "faint", halo, lit, flat, className, inner })` | 设置了 `paths.seamless`（平滑回转着色，`sphereOf`）时，组会带上 `data-seamless`，`ISO_CSS` 会以各区域自己的颜色、0.4 宽度描每个色调区域，众多相接的色调片之间因此不显发丝缝。`inner` 是紧放在轮廓（`iso-edge`）之前的额外线条标记，用于必须压在轮廓描边之下的接缝、焊线与环带线；React 的 `Solid` 以 prop 接收它。 |
| `lineSvg(d, { tone = "lo", dotted, dashed, flow, className, free })` | `flow` 让虚线动起来；用于沿路径移动的东西。`free: true`（或 `"start"`、`"end"`）会写入 `data-free`，告诉 `scripts/lines.mjs` 某一端就是要停在裸面上的：代替印制文字的刻度线、表针尾、指针尖。其余所有线端必须落在某个物体上。 |
| `dotsSvg(points, { size = 0.5, tone = "mid", pulse })` | 点可以携带第三个值——不透明度（`[x, y, 0.4]`），`dotsOf(..., { fade })` 就是这样让螺栓圆渐隐而非中途截断的。 |
| `fadedSvg(list, style)` | 绘制一组各自带不透明度的线，`[{ d, alpha }]` 或 `[d, alpha]`，即 `ribsOf(..., { fade })` 的返回。线按 0.05 步长依不透明度分组，每组置于一个 `<g opacity>` 中，低于 0.02 的线丢弃。用于任何成员朝侧缘（limb）逐条渐隐的纹理：肋条、散热片、滚花、刻度线。 |
| `fadeLineSvg(points, { fade = [start, end], tone = "lo", className, stops = 6 })` | 一条线（屏幕点），在首端 `start` 与尾端 `end` px 内渐隐而不是戛然而止（每个渐变段的渐隐端带 `data-free`，因此线条检查会放行）。每个渐变段用沿自身弦的 `userSpaceOnUse` 线性渐变描边（`stops` 步的 smoothstep，颜色取自 `var(--anatomy-<tone>)`，两套主题都适用），中段是普通 `lineSvg`，因此没有重叠的端帽、珠点或接缝。用于终止在面中央的纹理线：走尽的凹槽、在角落前止步的焊线、曲面板上的刻度。渐变段保持短于线转弯的四分之一，因为线性渐变沿弦展开。渐变 id 是路径的哈希，同一条线画两次共用一个。 |
| `wireSvg(from, to, { via = [], tone = "lo", dotted, dashed, flow, ends: "both" \| "from" \| "to" \| "none", size = 1.2 })` | 带端点（terminal）圆点的连接线。 |
| `faceTextSvg(transform, text, { size = 3, tone = "mid", anchor })` | |
| `groupSvg(children, { class, transform, "data-part": … })` | |
| `figureSvg({ width, height, label, body, standalone: false \| "dark" \| "light", className })` | `<svg>` 本体。`label` 是无障碍描述：说明这个物体是什么、展示什么。`className` 会加进它的 class，把图叠在 canvas 之下时有用（`rt-back`、`rp-back`）。 |

## 8. 卡片与页面
| 函数 | 说明 |
|---|---|
| `plateHtml({ fig, title, hint, readout, keys: [{ mark, label }], caption, body })` | 卡片。`mark` 取 `raised`、`flat`、`lit`、`edge`、`dim`、`dotted`、`dashed`、`red`、`green`、`blue` 之一。读出（readout）元素带 `data-readout`，动态代码直接写入它。 |
| `pageHtml({ title, theme = "dark", body, script, width = 650 })` | 内联了 `ISO_CSS` 的独立文档。 |
| `kitScript()` | 来自 `scripts/inline-kit.mjs`。前置到 `script`，页面在动画中就能调用 `lineOnTop`、`translateAlong` 这类 kit 函数。 |
| `glScript()` | 来自 `scripts/inline-kit.mjs`。`kit/gl.mjs` 的源码（`GLSL_ISO`、`glCamera`、`glLayer`、`smooth`、`burst`、`settle`），在带着色器的页面上内联到 `kitScript()` 之后：`script: kitScript() + glScript() + live`。它不依赖 kit，动态代码不调用任何 kit 函数的页面可以单独内联它（Raptor 用的是 `glScript() + LIVE`）。所有内容是同一个模块脚本，动态代码不得重复声明 `smooth`、`burst` 或 `settle`：第二个 `const smooth` 是会杀死页面的 SyntaxError。 |
| `turnScript()` | 来自 `scripts/inline-kit.mjs`。3D 运行时（`TURN`、`TURN_CSS`），已剥离 `export`，用于带动态零件的页面：`script: turnScript() + glScript() + live`。其内部标识符以 `t3`/`T3_` 为前缀，因此可与 kit 和 WebGL 层并排内联。 |
| `ISO_CSS` | 全部样式。`kit/iso.css` 是同样的文本。 |

## 9. React 组件
来自 `kit/react/draw.tsx`，服务端安全：

- `<Solid paths tone crease halo lit flat className inner />`（`inner` 是放在轮廓之下的线条标记，同 `solidSvg`）
- `<Line d tone dotted dashed flow free />`
- `<Wire from to via tone dotted dashed flow ends size />`
- `<Dots points size tone pulse />`（点可携带不透明度 `[x, y, alpha]`，同 `dotsSvg`）
- `<FaceText transform size tone anchor>text</FaceText>`
- `<IsoFigure width height label>` …svg 子元素… `</IsoFigure>`
- `<Plate fig title hint readout keys caption>` …图形… `</Plate>`

来自 `kit/react/live.tsx`，标记 `"use client"`：

- `useReadout()` 返回 `{ anchor, say }`。把 `anchor` 放在根元素上；`say(text)` 写入卡片读出区。
- `useLoop(tick: (dt, calm) => keepGoing, running)` 在 `running` 为真且 `tick` 返回真时运行帧循环。返回 `kick()` 可重新启动。
- `follow(current, target, dt, seconds, calm)` 是指数缓动。
- `spring(state, target, dt, stiffness, damping, calm)` 是分步子步的弹簧。安定后返回 true。
- `useEased(target, seconds)` 以 React state 返回缓动后的数值。小图够用。
- `useInView(ref)`, `useStillness()`, `asksForStillness()`.
- `Stage` 是 `IsoFigure` 所用的包装组件。它为 CSS 动画添加 `data-inview`。

## 10. 任意轴上的圆形零件（`kit/lathe.mjs`）
`kit/iso-kit.mjs` 构建立在地面上的棱柱。`kit/lathe.mjs` 把轮廓剖面绕**任意**轴回转，用于钟形件、腔室、法兰（flange）、阀体、泵蜗壳（volute）、管接头、斜置的执行器以及竖直面上的连接器。它返回与 `slabOf` 相同的 `SolidPaths`，因此 `solidSvg` 可以直接绘制。以 `import * as G from "../../kit/lathe.mjs"` 导入。

**标架（frame）。** 标架是 `{ o, a, u, v }`：一个原点、单位轴，以及两个单位径向方向。零件上的一个点表示为 `(s, r, angle)`：`s` 沿轴从 `o` 起算，`r` 是到轴的距离，`angle` 从 `u` 转向 `v`。

| 函数 | 作用 |
|---|---|
| `frameAlong(o, a)` | 过 `o`、以 `a` 为轴的标架，`u` 与 `v` 替你选好。用于任意角度的管道、凸台（boss）与法兰。 |
| `frameOf(o, a, u, v)` | 径向方向显式给定的标架，角度因此有确切含义：`G.frameOf([0, 0, z], [0, 0, 1], [1, 0, 0], [0, 1, 0])` 使角度 0 指向 +x。 |
| `pointOf(F, s, r, angle)` | 对应的世界点。 |
| `radialOf(F, angle)`, `tangentOf(F, angle)`, `angleToward(F, direction)` | 标架内的方向，以及面向某方向的角度。 |

**实体。** 子午线（meridian）是沿轴排列的 `[s, r]` 点列表。加上第三个元素（`[s, r, 1]`）可把一个环标记为平滑环，该处就不会画折痕线。钟形或穹顶这类弯曲轮廓用它。任一顶点平滑的子午线会自动平滑着色（`smooth` 默认 `"auto"`，等价于 `softOf(meridian)`）；传 `smooth: false` 退出，`true` 或 `{ rows }` 可对无平滑顶点的轮廓强制开启，`{ slope }` 或 `{ slope, radius }` 则用于解析曲线。

**平滑着色。** 没有它时每个面片行只有单一平直色调，弯曲轮廓会呈现色调块的阶梯（最早的 Raptor 钟形件与喉部就是这样，评审一眼就看出来了）。有了 `smooth`，每段子午线段的侧面按插值法线着色：色调边界成为真实的等照度曲线，按 `rows` 层采样、每个色调区域输出为一个多边形，因此它们以干净的曲线贯穿整个零件。
- `smooth: true` 用单调三次 Hermite 曲线（Fritsch–Carlson 斜率：轮廓转折处为零，其余为加权调和平均）插值每一段平滑顶点（`[s, r, 1]`），并把这一条曲线同时用于法线、色调片、轮廓与凸包，重采样直到与弦相差 0.2 px 以内。着色与轮廓由此描述同一形状，法线只能在轮廓转折处变号。任何带平滑顶点的轮廓都用它：穹顶、喉部、锥面、阀体鼓包、火焰锥。（自然样条在峰值附近会过冲；在阀体鼓包上它曾让色调带在身体上来回折弯。）
- `smooth: { slope }` 从解析的 `dr/ds` 取法线，切成分段的同一物体共享一个法线场，色调曲线无缝穿过分段接缝。只给 `slope` 时轮廓点直线相连，因此要密集采样解析的 `r(s)`（喉部 20–30 个节点）；收腰物体的轮廓同样使用该 `slope`，因此外轮廓也贴合真实曲线。
- `smooth: { slope, radius }` 连曲面也取自解析的 `r(s)`：色调片、凸包与轮廓在节点之间都跟随 `radius`，因此一个环带只需两个节点。`bandsOf` 传给每个分段的正是这个。
- **手打节点不构成曲线。** 单调 Hermite 只有 C1 连续：其 `dr/ds` 是分段二次函数，在每个节点处走平、在节点之间过冲，色调带因此先停顿再猛跳，呈扇贝状阶梯。Raptor 喉部（`[128, 31, 1], [134, 27.6, 1], …`）在 5× 放大下就露了馅。任何大于一个接头的弯曲体都配解析轮廓加 `smooth: { slope }`：喉部是 `r = √(24.2² + (c·(s − 146))²)`，火焰锥是 `r = 48·(1 − x)^0.8` 并在尖端闭合到 `r = 0`，阀体鼓包是椭圆，万向节圆饼是 `18.4 − a·|s − 297.5|⁴`。审计的 `wobble` 会列出每个在无 `slope` 情况下平滑、且 `dr/ds` 在两端之间变向的轮廓。
- `rows`（默认 6）是每段沿轴的初始采样数。行随后继续细分：直到每条色调边界与直弦相差不超过 `error`（0.35 px），并且在环带数量变化之处一直细分到 `depth`（8）层或屏幕上的 `fineness`（0.1 px），以先到者为准，于是出现或消失的环带在细条中以零宽度诞生，而不是终结于一条直线切口。只要自身区域数量保持不变，每个色调就作为一个多边形输出，贯穿物体的每段子午线段与每个平滑节点；数量变化处仅对该色调做桥接，忽隐忽现的环带不再把其他色调切成行。这让一个小回转球体积缩小约一半，并消除了行缝。可见性按绘制面片的法线判断，因此色调总能延伸到轮廓；每个色调多边形按同一方向绕行，同一色调的重叠片绝不会相互抵消成洞。
- 结果会被标记 `seamless`（见 `solidSvg`）。平端盖（`s` 不变的环）保留其面片着色。

| 函数 | 作用 |
|---|---|
| `lathe(meridian, F, P, { steps = stepsFor(rMax, 32, 160, scaleOf(P)), fill = "auto", hole = null, bevel = 0, rims = "all", smooth = "auto", waist = 0.5 })` | 通用的回转体。面片按 kit 的光照以四色调着色，分桶方式与 `extrude` 相同。`fill: "auto"` 使用各环的凸包，除非真实轮廓比凸包内缩超过 `waist` px（喉部、线轴类），此时改用两侧的 `silhouettePoints` 并以可见端弧相连来填充和勾画物体（仅当轮廓沿整条侧边无断口时才这样：表面在某处正对你直视的物体——例如以直角离开颈部的阀体鼓包——会退回凸包，因为拼接断开的片段曾在其上画出杂散三角形）；`"surface"` 强制前者，`"hull"` 强制凸包；`fill: "facets"` 只填充可见面片且不画轮廓，从内部观看凹形或开口形状时需要它。`hole: [s, r]` 在填充与轮廓上切出圆孔，用于管的端部。`bevel` 从闭合端盖（`r = 0` 处的子午线端）内缩加一条高光。`rims` 决定端环上是否保留轮廓：`"all"`、`"none"`、`"first"`、`"last"` 或 `"ends"`。只有沿首环或末环的凸包边会被丢弃；沿中间环的边始终保留，因此侧面轮廓保持完整。 |
| `solidOf(profile, F, P, options)` | 把轮廓在两端闭合到轴上，并按最大半径选 `steps`。这是制作回转件的常规方式：`G.solidOf([[0, 12], [6, 12], [8, 9, 1], [20, 9]], F, P)`。 |
| `bandOfProfile(profile, F, P, options)` | 无端盖的开口环带，用于被切成深度排序（depth-sort）片段的长物体中的一段。优先用一次切分整个物体的 `bandsOf`。 |
| `bandsOf(profile, cuts, F, P, { slope, radius, steps, rows = 6, rims = "none", fill = "auto", samples = 72 })` | 一条弯曲物体（`s` 沿 `profile` 递增），在 `cuts` 给出的 `s` 值处切成可深度排序的环带，各段共享同一法线场、同一曲面与同一面片数。传入解析的 `slope` 与 `radius`，或两者都不传：此时 `profileCurve(profile)` 从轮廓推导（每段平滑节点走单调 Hermite，其余直线），因此任何分段都不会有自己的端部斜率。返回 `{ bands, sidesOf, radius, slope, steps, s0, s1 }`。每个分段是 `{ s0, s1, profile, paths, outline, seam }`：`paths` 已清空轮廓、可直接交给 `solidSvg`；`outline` 是分段自己的弦轮廓，需要时可取用（Raptor 裙部最后一个分段保留了它）；`seam` 是它与下一段相接处的可见弧（放进 `inner`）；`profile` 是供审计记录的内容（`A.solid(F, band.profile, { radius })`）。`sidesOf(from = s0, to = s1, { ends = false \| true \| "first" \| "last", count = samples })` 返回 `{ d, points }`：该跨度上物体的两条解析侧边合为一条路径，可选地在其两端带上可见轮缘（rim）弧，并附上供记录的三维点（`points.map((list) => A.tube(list, 0.4 / scale))`）。把 `d` 作为一个条目、加少量深度偏置画在分段之上，轮廓便是一条连续曲线。 |
| `profileCurve(profile, { slope, radius })` | 针对沿 `s` 递增轮廓的 `{ s0, s1, radius(s), slope(s) }`：`smooth` 绘制时穿过它的曲线。`s` 不递增时抛错，因为轮廓中的台阶是一道折痕（crease），应属于另一个独立实体。 |
| `outlineOf(meridian, F, P, { steps, fill, hole, rims, smooth, waist })` | 只有 `lathe` 的 `fill` 与 `outline`，不带着色，用于轮廓另行绘制的物体。 |
| `softOf(meridian)` | 任一顶点被标记为平滑时为 true。 |
| `disc(s0, s1, r, F, P, options)` | 轴上的短圆柱：圆饼、法兰、凸台、销、堵头。 |
| `ringBand(s0, s1, rIn, rOut, F, P, options)` | 有厚度的环圈：环绕他物的法兰或卡箍（collar）。 |
| `stepsFor(radius, least = 16, most = 112, scale = 0)` | 面片数量。带上 `scale`（每世界单位的屏幕 px，`scaleOf(P)`）时，它让每个面片在屏幕上保持约 3 px 长；`disc`、`ringBand`、`solidOf`、`bandOfProfile` 与 `lathe` 在未传 `steps` 时都用它（32–160）。面片只决定色调填充：轮廓、轮缘、折痕与倒角按约 1.5 px 从真实圆采样，并恰好终止在面转向背面的位置。 |
| `saddleOf(F, a, start, s1, P, { steps })` | 鞍座垫（saddle pad）：沿 `F.a`、半径 `a` 的圆柱，从弯曲的宿主表面延伸到 `s1` 处的平面。`start(angle)` 是每条母线离开宿主的位置（用二分法求宿主的 `sdf` 得到），或传数字 `R` 表示垂直于 `F.a` 的宿主圆柱（此时 `saddleStart(R, a, angle)` 给出每条母线的起点）。填充是可见面片与端面的并集，轮廓是两条侧母线、鞍线的可见部分与端面轮缘，因此它沿真实交线与宿主相接。凡平法兰放在弯曲物体上会留轮缘缝隙的地方，都把法兰落座在鞍座垫上。 |
| `prismOf(F, polygon, s0, s1, P)` 与 `stadium(r0, r1, offset)` | 标架 `u`、`v` 平面内任意凸 `[x, y]` 多边形的棱柱，沿 `F.a` 拉伸，带面片色调，以凸包作填充与轮廓，折痕只出现在尖角。`stadium` 是两圆的凸包，用于连接电机与执行器的减速箱板。 |
| `sphereOf(centre, r, P, { steps = 48 })` | 带精确平滑色调的球：每个色调区域是球冠投影的凸包，只花四个小多边形而非回转轮廓。用于杆端轴承、三通（tee）接头与旋钮。它是 `seamless` 的。 |
| `silhouettePoints(profile, F, P, { samples = 72, slope, radius })` | `silhouetteOf` 的两条侧边，以三维点列表形式给出，供审计记录轮廓形状。 |
| `silhouetteOf(profile, F, P, { samples = 72, slope, radius })` | 穿过 `profile`（`[s, r]` 点，线性插值）的回转曲面的两条解析侧边，合为一条含两段折线的路径：即曲面转向背对观察者的那些点（`cos(θ − φ) = r′·aV/|V⊥|`）。用它为分段构建的物体画一条连续轮廓。曲面正对观察者处（`|c| > 1`）跳过采样。传入解析 `slope`（与分段着色用的同一个），边就不会在每个轮廓点处打结。 |

**回转面上的线与点。** 只绘制朝向观察者的部分。`slope` 是曲面在该环处的 `dr/ds`，锥面或钟形面上的线因此会在真实曲面转向背面的地方隐藏。

| 函数 | 作用 |
|---|---|
| `arcOf(F, s, r, P, { slope = 0, steps, least = 0.02, inward = false, from = 0, to = 2π, trim = 0.3 })` | 曲面上环的可见弧：接缝、环带、焊线、唇口。侧缘（limb）角精确求解（`t = φ ± acos(c)`），每个止于侧缘的端点缩短 `trim` px，使其圆帽留在轮廓描边之内。`inward: true` 绘制你在内表面看到的弧，例如喷嘴内侧。`least: -1` 给出整环。接缝要画在轮廓*之下*：`solidSvg(paths, { inner: lineSvg(arcOf(…), { tone: "faint" }) })`。若附加在实体之后，浅色接缝端帽会盖住边缘描边，使轮廓在每个接缝处出现缺口。 |
| `circleOf(F, s, r, P, steps)` | 完整闭合的圆，如端面圆环、孔口或要填充的空腔。 |
| `ribsOf(F, s0, s1, r, count, P, { phase, twist, least = 0.12, r1 = r, fade, seams })` | 可见的轴向线：沿任意轴的滚花、肋条或散热片。`twist` 使其成螺旋，`r1` 使其沿锥面延伸。带 `fade: [lo, hi]` 时返回 `[{ d, alpha }]`，每条肋的不透明度是其朝向你的程度的 smoothstep，整组因此朝转弯处渐隐而非停在 `least`。用 `fadedSvg` 绘制。`seams: true`（或 `"first"`、`"last"`）把 `s0` 与 `s1` 处的接缝弧加入结果（全不透明度），肋条行与其所终止的接缝始终共享同一个 `s`；仅当肋条终止于台阶或已绘出的环带端部时才可省略。 |
| `rodOut(F, from, to, r, P, { stroke, ...latheOptions })` | 一根从 `s = from` 处密封套（gland）滑出的杆（`F.a` 从密封套指向外），静止时杆端在 `to`，可再向外伸出最多 `stroke`，或同一根杆缩回密封套。返回 `{ paths, clip, from, to, stroke, reach }`：`paths` 是从 `from − stroke` 到 `to` 的一根杆，`clip` 是覆盖从密封套面到 `to + stroke` 这段杆的路径。绘制 `<clipPath id><path data-hold d=clip/></clipPath><g clip-path=url(#id)>solidSvg(paths)</g>`，移动条目，调用 `holdStill`，并记录 `A.slide(F, from, to, r)`（`craft.md` 第 3 节）。 |
| `spokesOf(F, s, r0, r1, count, P, { phase, sweep, from, to })` | 面上的径向线，用于叶片与涡轮桨叶。 |
| `dotsOf(F, s, r, count, P, { phase, slope, least = 0.08, all = false, fade })` | 可见一侧螺栓圆的屏幕点。交给 `dotsSvg`。正对着看的面用 `all: true`。`least` 远大于 0 会让圆停在曲面中央；改传 `fade: [0.04, 0.42]`，圆点便带递减的不透明度。 |
| `fadeOf(value, [lo, hi])` | 两种渐隐共用的 smoothstep。 |
| `facingOf(F, angle, P, slope)` | 该角度处曲面朝向观察者的程度。正值表示可见。 |

**三维路径**，供管道与线缆使用。

| 函数 | 作用 |
|---|---|
| `fillet(points, radius, perArc = 7, { P, tube, margin = 1.3 })` | 用 `radius` 的弯折圆滑折线的每个拐角，并限制在可用空间内（短腿的 0.48）。先把管道布置成几条直腿，再做圆角。带 `P` 与 `tube`（管半径）时，每个拐角的弯半径放大到 `margin · tube / (0.7 · f²)`，其中 `f = |n · V|` 是弯折平面正对观察者的程度：侧视的弯折投影半径为 `R·f²`，低于管自身半径时轮廓会自折叠。直腿仍是其上限，因此紧邻朝向你的端口的转弯需要足够长的直腿或一个弯头接头。 |
| `resample(points, spacing)`, `pathLength(points)`, `cut(points, from, to)` | 等距重采样、总长，以及按弧长截取的子路径。 |
| `pointAlong(points, at)`, `tangentAlong(points, at)` | 某弧长处的位置与方向。用于在路径上安放法兰、环带、阀与卡箍。 |
| `spiral(F, s, r0, r1, a0, a1, steps = 28)` | 渐宽螺旋的三维点，如泵蜗壳。连同半径函数喂给 `tubePieces`。 |
| `arcPoints(F, s, r, a0, a1, steps = 24)` | 绕环一周的三维点，如歧管、喷淋环或线束通道。 |

**相机与向量。**
- `viewOf(P)` 是指向观察者的单位方向，因此 `dot3(p, viewOf(P))` 等于 `depthOf(p, P)`。
- `lightOf(P)` 是 kit 的光照。
- `scaleOf(P)` 是每世界单位对应的屏幕单位。
- `toneOf(normal, P)` 与 `bandOf(score)` 给出四色调分桶。
- `hullOf(points)` 与 `runsOf(flags)` 是模块自用的凸包与区段辅助函数。
- 向量辅助函数有 `dot3`、`add3`、`sub3`、`mul3`、`len3`、`unit3`、`cross3` 与 `lerp3`。

**以分段构成的长弯曲物体。** 用 `bandsOf` 切分，让每个分段与缠绕其上的管道做深度排序。每个分段共享物体的 `steps`、法线场与曲面，色调曲线无缝穿过每个接缝；轮廓来自 `sidesOf`，是一条曲线，而不是每个分段一条弦、在每个接缝处打结。Raptor 钟形件：
```js
const BELL = G.bandsOf([[3.2, bell(3.2)], [116, bell(116)]], [14, 28, 44, 60, 76, 92, 106], ENGINE, P, { radius: bell, slope: bellSlope, steps: 96, rows: 8 });
BELL.bands.forEach((band, index) => put("nozzle", ex(ENGINE, (band.s0 + band.s1) / 2), S(band.paths, { tone: "hi", inner: index === 2 ? Ln(band.seam, { tone: "faint" }) : "" }), { shapes: [record(A.solid(ENGINE, band.profile, { radius: bell }))] }));
const SIDES = BELL.sidesOf(3.2, 106, { ends: "first" });
put("nozzle", ex(ENGINE, 105), Ln(SIDES.d, { tone: "hi" }), { bias: 0.2, shapes: SIDES.points.map((points) => A.tube(points, 0.4 / KS)) });
```
不要用多次独立的 `bandOfProfile` 调用手工拼分段（各自在两节点弦上做 Hermite 拟合）：那样每个分段都有自己的端部斜率，色调在每个接缝处跳变，弦轮廓也会打结。

**已知的坑。**
- 物体未平滑时，**四色调量化会跟随每个面片行的斜率**。把每个弯曲节点都标记为平滑（`[s, r, 1]`），着色自然跟随；由硬节点构成、但仍描述曲线的轮廓需要 `smooth: true`。切成分段的物体用 `bandsOf`。
- **平滑着色更重。** 带 `smooth` 的小回转球约 9 KB；`sphereOf` 不到 3 KB。球用 `sphereOf`，小零件把 `rows` 保持较低。

## 11. 管道、管件与线缆（`kit/tube.mjs`）
```js
import { tubePieces, tubeSvg, runsOf, crossingsOf } from "../../kit/tube.mjs";
```
| 函数 | 作用 |
|---|---|
| `tubePieces(points3, radius, P, { maxLength = 30, spacing = 2.2, closed = false, breaks = [], gaps = [], caps = true, stripes = "auto" })` | 把三维路径切成至多 `maxLength` 世界单位的块。每块画成二维缎带：纸色的主体、一条朝光的亮带（shine）、一条背光的暗带（shade），外加两条边线。相邻块重叠 0.5 单位，边线与主体从同一段重叠跨度截取，接缝处因此既不露缝、轮廓也不出豁口。亮带与暗带在每个开放端都比主体多延伸 0.5：条纹盖住其下主体的抗锯齿切口，它自己的切口则落在邻块完全相同的条纹上；每个帧的法线都取沿整条路径的中央差分，因此两块的条纹在重叠处精确重合（曾因两者在同一条线上截断，蜗壳的暗带在每个接缝处都横贯一条发丝线）。`radius` 是数字或 `t` 的函数（沿路径 0 到 1），用于渐细管与蜗壳。`breaks` 是块必须终止的弧长位置（交叉处，或管道折回越过某零件之处）。`gaps` 是不绘制的 `[from, to]` 弧长跨度，因为卡箍、夹子、阀或接头盖住了那里；管道在 gap 两侧平齐截止。每个紧邻 gap 的端部都得到椭圆端帽（投影出的横截面，外侧半弧作为边），路径自身的两端亦然，除非传 `caps: false`（或 `[start, end]`，例如起点藏进三通卡箍的分支用 `[false, true]`），于是管道沿正确的曲线与法兰面相接，而不是一条直线切口。亮带与暗带跟随端帽：端面背向你时延伸到椭圆的外侧半边，端面朝向你时延伸到其内轮缘（作为边绘出），条纹绝不会停在端点之前。闭合且无 gap 的路径让首尾两块相互包绕。返回 `[{ mid, body, shine, shade, edges, wide, points, radii, ends, from, to }]`：`mid` 是块中心的世界坐标，用于深度排序；`wide` 表示路径在屏幕上足够宽、可以呈现亮带与暗带（按整条路径最宽处一次判定，渐细管因此不会在某个块接缝处掉条纹；`stripes: true` 或 `false` 可覆盖）；`points` 与 `radii` 是块的三维中心线与半径（记录下来供审计）；`ends` 说明哪些端是真端点而非与邻块的接缝。 |
| `tubeSvg(piece, { tone = "hi" })` | 单块画成 `<g class="tb">`。亮带与暗带只在 `wide` 的块上出现，然后是块的环纹（`.tb-ring`），再是边线。 |
| `tubePieces(…, { rings: { pitch = 3, twist = 0, cross = false, fade = [0.1, 0.55], phase = 0, steps = 16, tone = "lo" } })` | 跟随管道横截面的纹理：编织、波纹、缠带或铠装。站位沿路径起点起每 `pitch` 单位三维弧长布置一个（`phase` 可平移），因此图案在每个块接缝处连续；每个站位画成该处横截面的可见前弧，切成短段，不透明度为 `fadeOf(facing, fade)`，纹理因此朝两侧侧缘渐隐而非终止在硬线上。`twist`（以半径为单位）把每个环变成从一侧侧缘到另一侧前进该距离的螺旋，`cross: true` 加上反方向旋向，即编织。环纹跨过每块的条纹外延，接缝处后到的块在其自身主体的切口上方画出完全相同的纹段；真端点与 gap 端面处停止（环纹与端面保持三分之一 pitch 的距离，螺旋则终止在端面上）。每块携带 `rings: [{ d, alpha }]`；`ringsSvg(rings, tone)` 可单独绘制它们。 |
| `runsOf(total, gaps)` | 长度 `total` 的路径扣除 `gaps` 后实际绘出的 `[from, to]` 跨度。审计时把每个绘制段单独记录为一根管。 |
| `crossingsOf(points, centre, axis, radius)` | 路径穿过夹子或卡箍圆盘（过 `centre`、垂直于 `axis`、`radius` 内的平面）处的弧长。每个都转成夹子厚度的 gap。 |

`.tb-body`、`.tb-shine` 与 `.tb-shade` 的填充属于 `ISO_CSS`（以及 `kit/iso.css`），因此 `tubeSvg` 在任何 kit 页面上无需额外 CSS 即可工作。只有在重新设计管样式时才在页面 CSS 中覆盖，如 Raptor 对选中零件所做的（`.it[data-on] .tb-body`）。

**深度排序管件。** 一根管会穿行于多个深度之间，因此每块按自己的深度绘制：以键 `G.dot3(piece.mid, G.viewOf(P))` 放入条目列表，紧挨它穿过的实体。管道在邻近物体间穿行处用短块（`maxLength` 6–18），平躺在地面上的环用长块（400）。要在多个 SVG 层之间拆分管，测试 `piece.mid`。波纹槽的电机电缆分三路：桥上方的块（`piece.mid[2] > Z.top - 1 && piece.mid[1] < IN.y + IN.d - 10`）放到条杠之前的前层 SVG；轮缘上方的其余块放到近端帽之后的前层 SVG；沿支腿下行到控制盒的那段放中层 SVG。灯缆把悬出底座边缘的块（`piece.mid[1] < BASE.y - 0.2`）送到后层 SVG。在像 Raptor 这样稠密的图里，不要靠眼睛调这些键：把各块记录下来，交给 `orderKeys`（第 12 节）安定。

**凡是包住管子的东西都要把管切开。** 卡箍、环带、波纹盘、夹子或阀体都比它所套的管宽。若管作为一个整块穿过它，该块既在卡箍之后又在卡箍之前，任何键都无法排序：要么卡箍藏住管的近侧，要么管把卡箍的面盖掉。在卡箍处给管开 gap（`gaps: [[at - t/2, at + t/2]]`）：远侧块终止于卡箍背面，近侧块起于其正面，两者都能干净排序。Raptor 的 `pipe()` 对自己的端法兰、环带、波纹与阀这样做，`cable()` 则经 `crossingsOf` 对线束夹这样做。

Raptor 的 `pipe()` 辅助函数是一个好范式：
1. 用 2.6 × r（至少 3）的弯折对控制点做 `fillet`。阀或波纹管需要至少与其本体加两侧圆角切割（各 `r_bend · tan(turn / 2)`）等长的直腿；把它放在弯上，端部就会切进管道。
2. 先算好各卡箍（端法兰、环带、波纹盘、阀体），把它们的跨度作为 `gaps` 传入。
3. 对路径做 `tubePieces`，记录每块的 `points` 与 `radii`，并把每个绘制段作为一根管记录给审计。
4. 绘制各卡箍，归属管道（`owner`），审计就不会把管道自己的法兰算作碰撞。

`cable()` 在留在自身组件的一端加端点圆点，在落到另一组件的一端加连接器外壳（`shells: [start, end]`），因此图爆炸（展开）时每根线缆仍终止在某物上。两个辅助函数都接受 `free: [start, end]`，表示图拆开时该端会脱离安装座；圆角用 `{ P, tube: r }`（见 `fillet`）。必须朝观察者方向离开端口的线从一个 `elbow`（凸台上的球，两处接合均为轮缘贴球面，线朝其下一点离开球）出发，转弯因此发生在接头内部。`pipe()` 把每个端法兰的背面放在路径端点上，并可用 `coupling`（快接断开处的活接螺母）而非法兰终止一条线。

## 12. 审计（`kit/audit.mjs`）
```js
import * as A from "../../kit/audit.mjs";
```
**一行式挂钩。** 建一个记录器（recorder），让每个绘制辅助函数把条目放进其中并记录其形状，安定各键，最后以 `auditOrExit` 结束构建：
```js
const R = A.recorder(P, { order: STAGGER, spring: SPRING });
R.put({ piece: "base", at: centre, bias: 0.1, svg, shapes: [R.solid(A.slab(plan, 0, 6, { name: "base.plate" }))] });
R.route("base.cable", route, 0.6, { piece: "base", gaps, limp: true });
A.settle(R, P);
R.items.sort((a, b) => a.key - b.key);
A.auditOrExit(R, P, { crowd: [] });
```
`node build.mjs --audit` 会打印报告并在任何一项不通过时以非零码退出；不带 flag 时 `auditOrExit` 立即返回。不拆分的图不传 `order`，所有爆炸（explode）检查都退化为静置姿态。

| 函数 | 作用 |
|---|---|
| `recorder(P, { order = [], spring = {}, ground = grounded })` | `{ items, solids, tubes, routes, put, solid, route, nameFor }`。`put({ at, bias, key, name, piece, move, ride, shapes, ...rest })` 添加一个已绘制条目（不传 `key` 时其键为 `dot3(at, V) + bias`；其余字段如 `svg`、`route`、`chunk` 保留在条目上），并把条目的 `move`、`ride`、`piece` 盖章到其形状上。`solid(shape)` 记录形状并返回它，用于条目的 `shapes` 或画在排序层之外的静态零件。`route(name, points, radius, { owner, touch, bundle, limp, gaps, closed, ends, rims, free, piece, move, ride })` 记录管道或线缆：`gaps` 之间每个绘制段一根管，带其 `tips`、`tipAxes`、`tipRims` 与 `free` 端，另附路径供 `folds` 与拥挤（crowding）检查使用。`nameFor(base)` 给重复名字编号（`base`、`base#2`、…）。`order` 与 `spring` 是动态页面的错峰（stagger）与弹簧。 |
| `explodeStates(order, spring, { uniform = 10, stride = 4 })` | `{ depth, frames, apart }`：深度求解状态（均匀份额、每 `stride` 帧取一次的重放帧并在每个错峰边界处加反向、前缀状态）与间隙帧（所有重放帧加前缀状态）。无拆分时为 `{ depth: [0], frames: [] }`。 |
| `settle(R, P, { states, skip = sameRun, ...overlapOptions })` | 运行 `overlapsOf` 与 `orderKeys`，把键写入条目。它保持条目原有顺序，之后你可以自行排序 `R.items`。每次构建都运行。 |
| `audit(R, P, { allow, crowd, cover, ground, frames, apart })` | 对记录的形状执行下面所有检查，汇总为供 `report` 与 `failures` 使用的一个结果：静置、拆开与每帧的间隙、实体间隙、静置与拆开的拥挤（细长实体也在内）、替身（stand-in）形状、端点（terminal）、悬空端、折叠、自重叠、抖动（wobble）、落座（seats）、支撑与深度顺序。`allow` 豁免深度对，`crowd` 豁免拥挤对，`cover` 豁免替身条目。它从不看线条标记：那要跑 `scripts/lines.mjs`。 |
| `auditOrExit(R, P, { flag = "--audit", argv, ...auditOptions })` | 带 flag 时执行 `audit`，打印报告与一行摘要，任何失败置 `process.exitCode = 1`。 |
| `sameRun(a, b)`, `grounded(shape)` | 同一路径的相邻块（重叠检查会跳过），以及默认地面（meta 中带 `ground: true` 或名字以 `stand.` 开头的形状）。 |
| `keyAt(item, state, V)`, `labelOf(state)`, `cameraBasis(P)` | 条目在爆炸状态下的排序键（`key + lift · share`）、状态的可打印标签，以及相机的屏幕右、屏幕下与朝观察者单位向量 `{ R, D, V }`，供你自建探针。 |

精细的图会以肉眼错过、直到评审放大才暴露的方式失败：管道或线缆在静置或爆炸任一帧中穿过他物；法兰漂离安装座；零件悬在空中或陷入邻居；路径在屏幕上折返或收缩成一点；弯或端落在另一条线的轮廓上；图拆开时线缆端悬荡；重叠处两物绘制顺序颠倒。审计从真实几何度量所有这些。在 Raptor 上（478 个条目、1,119 个深度状态、约 4,400 个用于间隙的重放帧与前缀帧），键计算每次构建约一分钟，完整 `--audit` 全程约三分钟，大部分耗在逐帧间隙上。

**形状。** 构建时用世界单位为绘制的每样东西记录形状：

| 构造器 | 形状 |
|---|---|
| `solid(F, profile, { radius, ...meta })` | 回转件：两端闭合到轴的轮廓，与传给 `solidOf` 的同一 `[s, r]` 列表。平滑节点段沿绘制所用的同一条 Hermite 曲线描迹（或沿 `radius(s)`，如 `bandsOf` 分段那样），因此记录的曲面就是绘出的曲面，而非其弦。 |
| `lathe(F, meridian, { radius, ...meta })` | `G.lathe(meridian, F, P)` 的形状，同样的描迹：任何闭合子午线（环、阀体、罐）。 |
| `body(F, polygon, meta)` | 绕 `F` 回转的任意闭合 `[s, r]` 多边形，按原样对待。位于轴上的边不算表面，因此圆盘深处一点的深度是真深度。 |
| `disc(F, s0, s1, r, meta)`, `ring(F, s0, s1, rIn, rOut, meta)` | 法兰、凸台、销、卡箍。 |
| `box(min, max, meta)`, `orientedBox(centre, axes, half, meta)` | 方块；45° 的梁。 |
| `slab(plan, z, height, meta, steps = 8)`, `extruded(ring, z, height, meta)`, `cylinder(cx, cy, r, z, height, meta)` | `slabOf`、`extrude` 与 `cylinder` 的形状：同一圆角 plan 或任意多边形环（无论凸凹）立于地面，圆角台面或 L 形支架按所绘样子受检。 |
| `prism(F, polygon, s0, s1, meta)` | `G.prismOf` 的形状：标架 `u`、`v` 平面内任意多边形沿 `F.a` 拉伸。 |
| `traced(meridian, { radius, error = 0.02, depth = 6 })` | `solid` 与 `lathe` 记录的多边形。 |
| `ball(at, r, meta)` | 球与端点圆点（`size / scale` 世界单位）。`meta.flats: [[nx, ny, nz, d]]` 切平（保留半空间 n·(p − c) ≤ d）：`sdf` 为 `max(\|p − c\| − r, n·(p − c) − d …)`，表面采样包含切口轮缘，`boundsOf` 不变（保守）。3D 模式的关节球记录此形。 |
| `tube(points, radius, meta)` | 带半径的管道或线缆中心线（数字、0..1 的函数，或逐点的列表）。 |
| `slide(F, from, to, r, meta)` | 从 `from` 处密封套滑出的杆（从 `from` 到 `to` 的一个 `disc`）。在爆炸或行程状态下摆姿势时，它从密封套面伸展到 `to + offset · F.a` 而非平移，因此每个检查看到的都是绘出的那根杆。放在移动条目上（带行程 `move` 的那个）。 |
| `moved(shape, offset)`, `sdf(shape, p)`, `boundsOf(shape)` | 辅助：平移、有向距离（内部为负）、世界包围盒。 |

`meta` 携带 `name`（用于报告，按前缀匹配：`"nozzle.bell*"`）、`owner`（管道自己的法兰、环带与阀）、`touch`（管允许有意接触的名字，如线束周围的夹子）、`bundle`（同一线束的线缆可以并排，夹子携带其线束的 bundle）、`joined`（一根开 gap 管的各段）、`limp`（线缆：不支撑任何东西）、`anchor: false`（夹子：端点不得把它当作安装座）、`fitting`（`crowding` 用其中心对其他线检查的夹子或卡箍）、`seated`（有意沉入宿主的垫块或球面轮缘接合）、`cut`（从本形状中减去的形状或列表：鞍座垫就是被宿主切过的圆盘）以及 `curve`（无解析斜率的回转件的平滑轮廓，供 `wobble` 用）。管段还携带 `tips`（路径的真端点，或并入本体的端为 `null`，如蜗壳起点）、`tipAxes`、`tipRims`（每端的法兰或连接器半径）、`free`（拆开时应脱离安装座的端）、`offset` 与 `total`（该段在路径上的位置）。每个形状都携带绘制它的条目的 `move`、`ride` 与 `piece`，因此可在任何爆炸状态下摆姿势。

**状态。** 爆炸状态是一个数字（所有组件处于该份额）或 `{ label, shares }`（每个组件一个份额）。`uniformStates(10)` 给出 0, 0.1 … 1；它们不是任何人会看到的帧，只用于喂给键求解器。`springStates(order, { stagger, stiffness, damping, substeps, fps = 60, settle = 1e-3, stride = 1, reversals = [] })` 按动态页面自己的弹簧与错峰重放：从静置张开、从拆开收拢，`fps` 下每帧一个状态，直到每个弹簧都进入目标的 `settle` 范围（Raptor 上单程约 1.2 s，因此每次收拢的慢尾也受检）。`stride: 4` 每四帧留一帧，供深度栅格使用。`reversals` 加入快速悬停产生的路径：张开，再在那么多秒后重新收拢（及其反向），并沿用页面自身的调度：反向时只有目标变化的组件被重新调度、在彼此之间错峰，刚离开的组件直接折返，仍在等待的组件干脆留在家中。**`reversalsOf(order, stagger)`** 在每个错峰边界两侧 8 ms 处各安排一次反向：坏窗口就在那里（下一次启动前 15 ms 的一次反向曾让热气导管飞出去、撞进仍停在家中的甲烷泵），而三个手挑的时刻一个也没碰上。**`prefixStates(order, { shares = [0.25, 0.5, 0.75, 1] })`** 加上前缀规则背后的静态状态：前 `k − 1` 个组件完全伸出，第 `k` 个处于每个份额，其余留在家中。沿错峰方向份额总是非递增（越早的组件领先，无论张开、收拢还是反向），每个可达帧都落在这些状态之间。40 ms 采样漏掉了整段穿越：两个采样之间喷嘴走了 15 个单位，超过最宽管道的两倍。`shareOf(item, state)` 从条目的 `ride`（或 `piece`）读其份额，`offsetOf(item, state)` 是其位移乘以该份额。

**`overlapsOf(items, P, { step = 0.25, tolerance = 0.3, least = 0.25, states = [0], skip })`** 找出每种状态下屏幕投影相互重叠的每一对已绘制条目，并在重叠区上以 `step` 个 viewBox 单位的网格沿相机射线对两个形状做射线投射（ray-cast）。当一个表面比另一个近 `tolerance` 以上时该格计为命中。每侧命中的格先闭运算（一次膨胀一次腐蚀，锯齿细条变实心），再腐蚀一次（真实接触处的发丝线被抹去），取最大连通块的 px² 面积；面积至少 `least` 时该侧算"在前"。旧的 0.5 步长配 4 邻域核心计数时，整个陷入泵壳的法兰和每一处线缆交叉（0.5 的线缆不到三格宽）都被读成噪声。每个条目只在原位做一次射线投射；任意状态下其栅格就是这一份沿其偏移在屏幕上平移（取整到网格）、在深度上按 `offset · V` 移动的结果——对平移而言这是精确的。成对结果按对的相对位移缓存，一起移动的对只测一次。在 Raptor 上，1,119 个状态（均匀、每条重放路径的每四帧、前缀状态）比旧的 61 个还便宜。`skip(a, b)` 排除某些对，例如同一根管的相邻块。

**`orderKeys(items, overlaps, { gap = 0.02 })`** 把每个单侧对变成其状态下的规则：`keyFront − keyBack ≥ gap + shareBack · liftBack − shareFront · liftFront`。它对键是线性的，一次最长路径松弛即可同时安定所有状态，且只会把条目往后移。无法满足的环丢弃其最弱规则（`dropped`）。在不同位置互为前后的对记为 `cycles`。

**`depthOrder(items, P, { overlaps, solved, rest, allow })`** 检查每种状态下的最终顺序。静置时（`rest`，默认状态 0 与 1）与每个重放的弹簧帧中，每一对错的都算问题：干净的乱序、互锁、以及任何 `k + lift · share` 形式的键都撑不住的对，都会在屏幕上显形。均匀状态中只有干净的乱序才算；那里的互锁列为通过。`allow` 接受 `[patternA, patternB]` 对做显式豁免。

**`clearances(tubes, solids, { ratio = 0.5, least = 0.2, state, only, mounts })`** 采样每根管，列出它与实体或另一管接近到 `max(least, ratio · r)` 之内的所有位置。在段的平端附近它量的是端环，而不是会悬伸出切口的球。可豁免的只有：端自身的安装座（其轮缘所坐落的形状，见 `mountsOf`）、该安装座的管道，以及共享该安装座或落到本管上的那条线的接头——且仅限端的 `r + gap + 1` 范围内；端附近的其他任何东西，被侵入仍不得超过 0.1。传 `state` 摆出全部姿态（豁免只在管与安装座都保持在共处位置 0.05 之内时有效）；传 `only: "through"` 只列出穿透。管到管的距离是真实的截头圆柱距离，因此平齐终止于法兰下方的段不会长出幻影圆帽、让下一段看似侵入其中。

**`mountsOf(tubes, solids)`** 与 **`terminals(tubes, solids)`** 为每个管端找出其端部轮缘（法兰或连接器半径处的 12 个点）最近的锚定形状，并列出轮缘与之相差 0.15 以外的每个端：偏离端口的法兰、弯曲物体上的平法兰（轮缘缝）、停在束旁或夹内的线缆。

**`looseEnds(tubes, solids, { state = 1 })`** 列出在该爆炸状态下、未接触任何随它一起移动的他人实体的每个管端。它自己的法兰、活接或连接器外壳不算（算上的话这检查永远不可能失败：Raptor 有 25 个端只停在自己的接头上）。标记 `free` 的端单独作为有意清单报告；其余一律失败。

**`supports(solids, tubes, { ground, touch = 0.15, sunk = 0.5 })`** 构建实体与刚性管的接触图，列出每个未连到 `ground` 的实体（默认：名字以 `stand.` 开头），以及每一对相互沉入超过 `sunk` 的实体——除非它们共享 owner，或其一 `seated` 于另一个（名字记在其 `cut` 中）。落座垫只对宿主豁免：Raptor 点火器法兰陷入另一组件热气垫 0.5 的事故，曾躲在一面一刀切的 `seated` 旗后面。`hanging` 列出未连接到任何被支撑物的刚性管，同样计为失败。

**`solidClearances(solids, { state, sunk = 0.5 })`** 摆出实体姿态，对该状态下偏移不同的每一对，把各自的表面点对对方的 `sdf` 检验；陷入对方超过 `sunk` 即失败（落座垫与宿主豁免）。在状态 1 与每个重放、前缀状态下运行：一个组件的外壳、垫块或法兰扫过另一组件的本体，对管件检查是不可见的。

**`seats(solids)`** 列出每个切到随另一组件移动的宿主的 `seated` 形状。拆开时切口底面暴露无遗：热气导管的鞍座、切到甲烷泵涡轮与环的那一个，曾以带尖尾的锯齿碎片飞出。鞍座属于其宿主的组件；移动的线在其平面端面上与之相接。

**`selfOverlaps(items, P, { fold = 0.25, least = 0.5 })`** 对每个绘制的管块自检：屏幕中心线以小于管屏幕半宽的半径转弯处，缎带内缘倒行，单块轮廓自交（一个结，隐藏腿的边缘透出）。折叠至少 `least` px 且该处未被其他条目覆盖时失败。指向观察者、并在离端口一两个半径内转弯的腿是常见原因。

**`wobble(solids, { samples = 20 })`** 以每段 20 点采样每条记录的 `curve` 的 `dr/ds`，列出在段两端之间变向的 `s` 值。

**`folds(routes, P, { fold = 140, turn = 110, pinch = 2, items })`** 列出每条屏幕方向在四个半径内转折超过 `fold` 度、而真实转折小于 `turn` 的路径（背离观察者随即下坠的管道之尖点），以及每个在屏幕上绕小于 `pinch` × 管屏幕半径的半径、转折超过 90° 的弯，度量跨度取 1.5、3、6 个半径（侧视的圆角：指向观察者、随后爬升的腿形成的 V 形）。端点三倍管宽内的捏点是端视的短桩，不计。带 `items` 时只有看得见的弯计数：每个弯都对所有已绘制形状做射线投射。

**`crowding(routes, P, { gap = 1, bend = 30, links, items, allow, state, clear = 2 })`** 列出路径的弯（±2 半径内超过 `bend` 度的三维转折）或端部落在另一条路径屏幕轮廓上的每个位置：轮廓相距小于 `gap` px 或相互重叠，且两线在该处都可见（对 `items` 做射线投射；路径自己的法兰、外壳与夹子算作该路径本身）。无论真实深度如何，这些都会被读成一个关节或一条穿进另一条的线。弯绝不因深度重叠而豁免；端只在另一条线在该处为直线、且端尖本身与其保持净距时豁免。每个 `fitting`（线束夹）也受检：夹子的轮廓落在另一条线 `clear` px 之内时，读作那条线从夹子中长出来——Raptor 的线束夹在拆开姿态中就曾这样落在氧气-预燃室线上。`links`（来自 **`linksOf(tubes, mounts)`**）豁免真正相接的线：一条装在另一条上，或两条同装于一个安装座。`state` 在爆炸状态下摆出路径与条目，拆开姿态因此也受检（路径携带 `move`、`ride`、`piece`）。`allow` 接受你已经查看并判定无碍的 `[patternA, patternB]` 对。

**`slenderOf(solids, { ratio = 4, least = 0.3 })`** 把每个细长的已记录实体（长于 `ratio` × 自身半径或半宽的回转件、棱柱或盒：拉杆、活塞杆、立柱、导轨、把手）变成伪路径，`audit` 以 `rods` 之名传给 `crowding`。管的弯或端落在杆上时，与落在另一根管上同样失败（`crowding: hose.B end × cyl.tie-rod (rod)`），除非该管装在它上面，或装在与它相连的东西上。杆只作为被落的线出现：它们自己的端不受检。

**`coverage(items, P, { least = 0.6, pad = 1, allow })`** 对每个带 `svg` 字符串的条目，把其所绘内容的屏幕盒（`markupBox(svg)`：每个 path、circle、line、rect 与 polyline，穿越各 `transform` 与 `clip-path`，不含柔影、文字与 defs）与其记录形状的屏幕盒（`footprintOf(shape, cameraBasis(P))`）比较，列出形状覆盖不足绘制 `least` 比例的每个条目：`stand-in shapes: rod.rod  its shapes cover 4% of its drawing`。为长杆只记录一条细条，或为绘制三个零件的条目只记录一个形状，会让其他所有检查对其余部分失明。`audit` 接受 `cover: [names]` 豁免绘制确实伸出本体的条目；逐个先看过再用。

**`report(result)`** 把一切格式化输出；**`failures(result)`** 统计应使构建失败的项数。

Raptor 这样接线（`examples/raptor-engine/build.mjs`）：它的 `put`、`tubeItems` 与 `record` 是对 `R.put`、`R.route` 与 `R.solid` 的薄封装，构建结尾是
```js
A.settle(R, P);
items.sort((a, b) => a.key - b.key);
A.auditOrExit(R, P, { crowd: CROWD_OK });
```
动态页面从其数据读取同一份 `STAGGER` 与 `SPRING`，审计与页面因此不会漂移。每个辅助函数（`turned`、`band`、`flangeAt`、`boss`、`padOn`、`boxAt`、`valveOn`、`pipe`、`cable`、`clampAt`、`strut`、各执行器）在放置条目时记录其形状，画在后层的静态机架（台面、火焰锥、喷淋环与供料、立柱、拼接件）记录为归属 `stand.frame` 的固定形状，因此把零件推进机架的爆炸向量同样会失败。没有任何东西能逃过检查。

## 13. WebGL 层（`kit/gl.mjs`）
| 导出 | 说明 |
|---|---|
| `GLSL_ISO` | `glLayer` 前置的 GLSL 序言。包含相机 uniform（`uCanvas`、`uView`、`uOrigin`、`uK`、`uCam`、`uTime`）与函数 `viewOf`、`project`、`towardViewer`、`rayOf`、`pixelOf`、`lightDir`、`onFloor`、`onWallX`、`onWallY`、`hash12`、`hash13`、`noise2`、`noise3`、`fbm2`、`fbm3`、`window` 与 `tonemap`。不要在片段着色器里重定义其中任何一个：GLSL 拒绝第二个 `rayOf`。 |
| `rayOf(vb)` (GLSL) | viewBox 点 `vb` 在过原点、面向相机的竖直平面上的世界点。`rayOf(vb) + t·towardViewer()` 是该像素的整条射线；`t` 越大越靠近观察者。 |
| `pixelOf()` (GLSL) | 每画布像素的世界单位，`(uView.x / uCanvas.x) / uK`。除以 `uCam.z` 得地面深度方向，除以 `uCam.w` 得沿墙面方向。 |
| `glCamera(P)` | 投影的 `{ origin, k, cam }`。可在 Node 中运行，放进页面 `DATA`。 |
| `glLayer({ canvas, fragment, view, camera, uniforms = [], fixed = {}, extensions = [], maxRatio = 1.5, onError })` | 返回 `{ gl, extensions, set(values), draw(values, time), clear() }`，无 WebGL 或编译失败时返回 `null`。`uniforms` 列出你将逐帧发送的 uniform 名。`fixed` 是 `{ name: value }`，创建时一次性设置，用于一切不变的量（plan、高度、主题常量）。`extensions` 列出 GLSL 扩展如 `"GL_OES_standard_derivatives"`；上下文支持的每个都会启用并以 `#extension … : enable` 前置，返回的 `extensions` 列出实际启用的那些。 |
| `set(values)` | 不绘制只设置 uniform，用于很少变化的值。 |
| `draw(values, time)` | 适配画布，设置相机 uniform 与 `uTime`，再设 `values`，然后绘制。 |
| `clear()` | 清为透明，仅当确实绘制过。 |
| `smooth(a, b, x)` | JS 版 smoothstep。 |
| `burst(age, rise, fall)` | 事件包络：`rise` 秒的 smoothstep 起振，随后以时间常数 `fall` 指数衰减。`age ≤ 0` 时为 0。 |
| `settle(state, target, dt, seconds, calm = false)` | 临界阻尼弹簧，作用于 `state = { x, v }`，时间常数 `seconds`，四个子步。误差与速度均低于 1e-4 即吸附到目标。仍在移动时返回 true。`calm` 直接跳到目标。 |

| `GLSL_TURN` | 3D 图形可选的 GLSL，由图形前置到自己的片段着色器：`uGroupX/Y/Z/T`、`toGroup(p)`（世界 → 组的静止世界坐标）、`toWorld(g)`、`dirToGroup(d)`、`dirToWorld(d)`，以及遮挡掩膜 `uCoverEdge[60]`、`uCoverCount[6]`、`coverOf(vb, soft)`（在效果之后绘制的零件内部为 1）。它定义了 `toWorld`：不要前置到自带 `toWorld` 的片段。该块使用 70 个 uniform 向量。 |
| `glslPose(Name, prefix = "uName")` | 任意组的同样四个函数：基于 `prefixX/Y/Z/T` 的 `toName`、`fromName`、`dirToName`、`dirFromName`，一个着色器因此可跟随多个组。喂给它 `TURN.poseUniforms(controller, group, prefix)`。 |

`fixed`、`set` 或 `draw` 的值可以是数字（float）、长度 2–4 的数组（`vec2`–`vec4`），或长于 4 的数组或 `Float32Array`，后者送往声明类型的 uniform 数组：`uniform vec3 name[N]` 用 `uniform3fv`（遮挡边、火花位置），`uniform2fv`/`uniform4fv` 同理，`uniform float name[N]` 用 `uniform1fv`。这些调用不接受 int、矩阵或采样器；`sampler2D` 默认纹理单元 0，需用 `layer.gl` 绑定纹理（波纹槽的历史纹理）。不在 `uniforms` 或 `fixed` 中的名字，以及被编译器优化掉的 uniform，都会跳过。

`smooth`、`burst` 与 `settle` 是纯 JS：在模块中从 `kit/gl.mjs` 导入，或在内联了 `glScript()` 的页面免费获得。`references/webgl.md` 有完整契约、最小页面与工艺规则。

## 14. 工具
| 脚本 | 用途 |
|---|---|
| `scripts/capture.mjs` | 一次性截图：桌面、手机、悬停、按键、定时截图、4× 放大、减弱动效、控制台检查。Chrome 启动即开 WebGL（允许 SwiftShader），着色器可见。`--times` 的延迟从第一张截图起算；`--key` 发送与 `drive.mjs` 相同的键码（`7` 是 `Digit7`，`Comma` 发送 `,`）。 |
| `scripts/drive.mjs` | 无头 Chrome 中的脚本化会话，WebGL 开启：滚动（scroll）、移动（move）、点击（click）、按键（key）、慢放（slow）、等待（until）、求值（eval）、截图（shots）与截图拼版（contact sheets）。`--no-webgl` 检查 SVG 回退，`--preload file.js` 在页面自身脚本之前运行一段脚本。着色器过渡与任何带时序的东西都用它。见 `references/verify.md`。 |
| `scripts/lines.mjs` | 线端检查：卡片中每条纹理线必须终止于描边、圆点或轮廓，或压在更晚绘制的填充之下。`--out` 圈出失败项；任何失败以 1 退出。同一检查也是 `drive.mjs` 的 `["lines"]` 动作。见 `references/verify.md` 的 "Line ends" 一节。 |
| `scripts/inline-kit.mjs` | `kitScript()`、`glScript()` 与 `turnScript()` 以普通脚本形式返回 kit、WebGL 层与 3D 运行时，供内联进独立页面。 |
| `scripts/drive.mjs` `["turn", value, group?]` | 3D 图形：经 `window.__isoTurn` 设置第一个旋转组（或 `group`，或一个 values 对象），并等待两帧。 |
| `scripts/drive.mjs` `["orbit", out.png, from, to, step, sel?, scale?, [x,y,w,h]?, columns?]` | 3D 图形：跨角度的截图拼版，每帧标注 θ°。整圈用 15°；顺序变化处用 0.1°；易折叠的弯用 1°。 |
| `scripts/turn-check.mjs` | 3D 图形。`build.mjs --fidelity`（Node：棱柱、圆件、回转体与管件对照 kit 构建器，在 P′(θ)）；`page.html --order`（需要 `--verify` 构建）、`--pops`、`--lines`、`--perf`、`--states file.json`、`--part name`、`--out dir`。每项检查先运行预置的对照样本。见 `references/3d.md` 第 14 节。 |
| `scripts/turn-bench.mjs` | 3D 图形：Node 中 360 个角度上 `emit` 与 `order` 的 CPU 时间，按零件种类（7 次取中位）。 |

## 15. 旋转组（`kit/turn.mjs`、`turn-build.mjs`、`turn-audit.mjs`）
可选的 3D 模式。`references/3d.md` 是指南；这里是参考。
```js
import { turning, separate, gjk } from "../../kit/turn-build.mjs";
import { TURN, TURN_CSS } from "../../kit/turn.mjs";
import * as TA from "../../kit/turn-audit.mjs";
```

### `turning(P, { recorder, scale = 1 })` → `T`
| 成员 | 作用 |
|---|---|
| `T.group(name, { parent?, turn: { pivot } \| slide: { direction, travel } \| hinge: { point, axis, range } \| free: { origin } })` | 刚性组。取值：角度（turn、hinge）、世界单位（slide）、`{ R \| q \| axis+angle, t, world? }` 或 `null`（free）。 |
| `T.layer(name)`, `T.stack(names)` | 动态层，以及动态与静态层名完整的从后到前堆栈。 |
| `T.prism`, `T.round`, `T.lathe`, `T.ball`, `T.tube`, `T.fixed`, `T.plane`, `T.billboard`, `T.split` | 零件，静止状态的世界坐标。公共选项 `{ name, group, layer, tone, crease, lit, material, owner, details }`。签名见 `3d.md` 第 4 节。 |
| `T.build({ statics, poses, verify, strict })` | `{ svg, data, css, report, layerSvg(name, { width, height }), verify, planesByName }`。 |
| `T.parts`, `T.groups`, `T.layers`, `T.routes`, `T.data`, `T.built` | 场景，供审计与各项检查使用。 |

`details`：`seams`、`ribs: { s0, s1, count, phase, twist, fade, seams, tone }`、`bolts: [{ s, r, count, phase, size, tone, fade }]`、`dots: [{ at, normal, size, tone }]`、`rings: [{ at, normal, r, tone }]`、`rules: [{ points, tone, free }]`、`bevel`。每个构造器在自己的标架里接受自己的子集（球的 `s` 沿其第一个切平面的法线；棱柱只接受 dots、seams 与 bevel）：表见 `3d.md` 第 4 节。材质：`gold`、`chrome`、`steel`、`gunmetal`、`rubber`、`brass`、`copper`。

`separate(A, B, hints)` 与 `gjk(A, B, warm)` 对两个点集返回 `{ n, d, margin }`（构建的平面求解器）。

### `TURN`（运行时）
| 成员 | 作用 |
|---|---|
| `TURN.mount(stage, data, { onHold, onFrame, follow })` | 控制器：`set(values | θ, { detail })`、`render()`、`angleAt(x, y, z, group)`、`values`、`pose(name)`、`detach(name)`、`anchor(group, restPoint, restDir?)`、`cover(names, options)`、`order(layer)`、`orderIndex(layer)`、`events()`、`stats()`、`frames()`、`cams()`、`scene()`、`destroy()`。安装 `window.__isoTurn`。 |
| `TURN.cover(controller, names \| [names…], { polygons = 6, edges = 10, pad = 0.3, layers })` | 供 `coverOf` 使用的 `{ uCoverEdge, uCoverCount, dropped, parts }`。 |
| `TURN.poseUniforms(controller, group, prefix = "uGroup")` | 供 `GLSL_TURN` 或 `glslPose` 使用的四个 `vec3`。 |
| `TURN.anchor(scene, cams, group, point, dir)`, `TURN.floorOf(scene, group, R)` | 控制器之外的锚定；组在旋转下的最低 z。 |
| `TURN.rotation(axis, deg)`, `TURN.quat(q)`, `TURN.compose(A, B)` | 以 `[9]` 表示的旋转，用于自由姿态。 |
| `TURN.materials`, `TURN.materialCss()`, `TURN.css()`, `TURN_CSS` | 材质名；CSS。 |
| `TURN.camera`, `TURN.poses`, `TURN.cams`, `TURN.prepare`, `TURN.emit`, `TURN.order`, `TURN.frame`, `TURN.gjk`, `TURN.disjoint`, `TURN.q`, `TURN.levels` | 流水线，供工具使用：每组相机、预备好的场景、单个零件的帧、一个层的顺序。 |

### 环绕审计（`kit/turn-audit.mjs`）
| 导出 | 作用 |
|---|---|
| `orbitOrExit(T, R, P, { flag = "--audit", argv, sweep, clips, poses, crowd, cover, fittings, quick, only, log })` | flag 存在时运行 `orbit`，打印报告，任何失败置退出码 1。只有完整运行才以 `turn audit passed` 结束；带 `only` 以 `turn audit partial` 结束，带 `quick` 以 `turn audit quick` 结束（`3d.md` 第 13 节）。 |
| `orbit(T, R, P, options)`, `orbitReport(result)`, `orbitFailures(result)` | 检查、文本、计数。`result.skipped` 列出 `only` 略过的行（打印为 `skipped`）；`result.quick` 在快速运行时置位。 |
| `posed(shape, pose)`, `eventsOf(T, layer)` | 被组姿态移动的已记录形状；一个层的侧视（edge-on）角度。 |
