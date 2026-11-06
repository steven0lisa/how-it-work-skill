# 绘图内部的 WebGL

如何把着色器（shader）效果（火焰、水、光、辉光（glow））放进等距（isometric）图形内部，让它读起来是同一个对象的一部分、经由同一台相机观看。两个已交付的示例做到了这一点：`examples/raptor-engine/`（火箭发动机下方的羽流、激波、溅射与蒸汽（steam））和 `examples/ripple-tank/`（焦散（caustics）、阴影与波纹水面）。请对照本页阅读它们的 `live.js`。

用户很喜欢第一个着色器图形，但注意到它在各阶段之间颜色瞬间跳变，仿佛效果被剪切过。于是标准是：**没有任何东西突现（pop）、没有任何东西被剪切、一切都落在绘图的透视之中。** 下面的每条规则都来自一个破坏该标准的 bug。

## 目录
1. 着色器何时值得一用
2. 图层栈
3. 相机契约
4. iso 相机中的光线步进
5. 将着色器内容对齐注册到 SVG
6. API 与最小页面
7. 工艺规则，每条都附上教会它的 bug
8. 浅色主题（8b. 旋转组上的着色器）
9. 验证
10. 性能

## 1. 着色器何时值得一用

着色器用于线条画不出的东西：发射光与辉光、火焰与尾焰（exhaust）、水面、焦散、来自真实灯具的柔和阴影、烟与蒸汽。这些是连续场（continuous field），而且它们正是主体本身或主体所做的事。

不要用于：
- **实体（solid）。** 零件、边缘、刻度与螺钉仍由套件（kit）的 SVG 绘制。着色器绝不画 SVG 能画的零件。
- **装饰。** 闪烁微光、背景渐变、每条边都加的泛光（bloom）、或为让图形显得"丰富"而加的噪声。如果删掉该效果而解释仍然成立，就删掉它。
- **SVG 已经表现得很好的东西。** 一颗发光的 LED 是一个 `lit` 圆点，不是着色器。

SVG 必须仍然能独立成图。没有 WebGL 时，Raptor 仍显示发动机，带轮廓光（rim light）与 CSS 甲板辉光。波纹水箱把波峰与波节线画成 SVG 线条（`model.mjs` 中的 `patternOf`），裁剪在所画的水箱投于纸上的阴影轮廓之内，并在 WebGL 运行后用 `.rt-stage[data-gl] .rt-fallback { display: none }` 把它们隐藏。

## 2. 图层栈

着色器无法与 SVG 一起做深度测试，所以改为给图层排序。基本栈是同一个 `position: relative` stage 中的三层，全部使用相同 viewBox：

```
back SVG     what the effect covers        (k.figureSvg: role="img" and the label)
canvas       the effect                     (position:absolute; inset:0; pointer-events:none)
front SVG    what covers the effect        (aria-hidden)
```

对每个零件只问一个问题：**它是否会被效果遮盖，或者它会遮盖效果？** 被遮盖的零件放进后层 SVG（back SVG），遮盖效果的零件放进前层 SVG（front SVG）。如果一个零件两者兼有，你有三种选择：

- **拆分绘图。** 把零件切成几块，把每一块送进它所属的图层、以及该图层中它应处的位置。波纹水箱的电机线缆是三组 `tubePieces`：跨过桥架的一段放在前层 SVG 中横杆之前，沿近侧边缘走的一段放在前层 SVG 中近侧端盖之后，沿左腿下行到控制盒的一段放在中间层 SVG。
- **在着色器中建模。** 如果零件是简单形状（盒、圆柱、圆环）且留在后层 SVG，就把该形状作为遮挡体（occluder）交给着色器。Raptor 的立柱是不透明盒（`columnNear`），所以每段空气积分从 `max(tGround, tColumn)` 开始。喷淋环是 `groundHit` 内的一个圆环（torus），所以溅射照亮其内侧、让其外侧保持暗。
- **拆分效果。** 当效果经过前层 SVG 零件之前时，在前层 SVG 之上再加第二个 canvas。用遮挡零件的一个解析代理（proxy）逐样本决定它去后层 canvas（权重 `1 − f`）还是前层 canvas（权重 `f`）。每个样本恰好绘制一次，因此没有接缝。代理要做得宽松：在真实轮廓线之外，两层看起来相同。

### 实战案例：Raptor
```
back SVG    deck, feet, flame cone, deluge ring, columns, ladder, cable tray
canvas      .rp-plume: plume volume, shock (back half), flash, splash, ground ring, steam behind the engine
front SVG   every engine item (depth-sorted .it groups), pipes, harness, crosshead
glow SVG    .rp-glow: the two nozzle-lip rims, screen-blended so they can only add light
canvas      .rp-veil: steam and shock samples that lie in front of the engine
div         .rp-flash: a CSS radial gradient (closest-side) for the ignition flash
```
发动机代理是 `engineR(z)`，钟形喷管（bell）、燃烧室与泵的半径函数。`inFront(p, V)` 判断样本 `p` 是否位于发动机与观察者之间。`side(front, veil)` 把样本送往其中一个 canvas。

### 实战案例：波纹水箱
```
back SVG    base plate, feet, ruler, label plate, paper sheet with tape, lamp cable over the base edge, SVG fallback pattern
canvas      .rt-paper-gl: lamp light on the paper and base, caustics, shadows of the legs, post, bridge, motor, control box and foam beach, refraction
middle SVG  post, arm, lamp and its cables, control box, motor cable down the left leg in clips, legs with levelling screws,
            far rails, three corner posts, far glass, foam beach, plain-water fill
canvas      .rt-water-gl: window reflection and room-light glints, meniscus tilt at the walls, dipper contact rings,
            the water band behind the near glass, glass tint, the lamp's beam (dark only)
front SVG   far caps, motor cable over the bridge, far pillar and spring, bar, dippers, motor, near spring, near rails,
            glass bottom, water lines, near glass, near caps, motor cable on the rim, near corner post, near pillar, beam
```
两个 canvas 位于两个深度，因为有两个效果处在两个高度：纸上的光（腿脚之下）与水面（水箱内部）。控制盒立在水箱旁的底座上，位于水 canvas 之下，所以它住在中间层 SVG；近侧角柱绘制在它所立其上的端盖之后。

这个栈的 CSS（改编自 `ripple.css`）：
```css
.fx-stage { position: relative; touch-action: pan-y; }
.fx-stage > svg, .fx-stage > canvas { display: block; width: 100%; height: auto; }
.fx-stage > .fx-layer { position: absolute; inset: 0; height: 100% !important; pointer-events: none; }
.iso:not([data-theme="light"]) .fx-gl { mix-blend-mode: screen; }
```
屏幕混合（screen blend）只能提亮。必须也能压暗其覆盖物的 canvas（波纹水箱的水会给透过它看到的纸染色）在暗色主题下也保持普通预乘（premultiplied）混合：只有 `.rt-paper-gl` 使用屏幕混合。
如果前层 SVG 必须响应 hover（Raptor 的零件如此），给它 `pointer-events: none`，而其零件用 `pointer-events: visiblePainted`，这样 canvas 与空白区域永远不会捕获指针。

## 3. 相机契约

`kit/gl.mjs` 把 `GLSL_ISO` 前置到你的片元着色器（fragment）里。它声明若干 uniform 与函数，使着色器看到与 `iso()` 相同的相机：

| 名称 | 它是什么 |
|---|---|
| `uCanvas` | canvas 的设备像素尺寸。 |
| `uView` | viewBox 尺寸 `[W, H]`。 |
| `uOrigin`, `uK` | 投影原点与每世界单位对应的屏幕单位数（`glCamera(P).k`，等于 `scaleOf(P)`）。 |
| `uCam` | 方位角（azimuth）与仰角（elevation）的 `(sinA, cosA, sinE, cosE)`。 |
| `uTime` | 你传给 `draw` 的时间。 |
| `viewOf(gl_FragCoord.xy)` | 设备像素到 viewBox 点（会翻转 y）。 |
| `project(vec3 p)` | 世界坐标到 viewBox，与套件的 `iso()` 完全一致。 |
| `towardViewer()` | 单位 3D 视线方向，`(cosA·cosE, sinA·cosE, sinE)`。每个 `p + t·towardViewer()` 都落在同一像素上。与套件 JS 的 `towardViewer(P)` 不是一回事，后者返回 2D 的地面方向。 |
| `rayOf(vb)` | `vb` 下方、位于穿过原点且面向相机的竖直平面上的世界点，因此 `rayOf(vb) + t·towardViewer()` 就是该像素的光线。 |
| `pixelOf()` | 屏幕上每个 canvas 像素对应的世界单位数，`(uView.x / uCanvas.x) / uK`。 |
| `lightDir()` | 套件的光，等于 `lightOf(P)`，因此着色器的明暗与 SVG 色调一致。 |
| `onFloor(vb, z)` | 该像素下方、高度为 `z` 的水平面上的世界点。 |
| `onWallX(vb, x)`, `onWallY(vb, y)` | 该像素下方、竖直平面 `x = const` 或 `y = const` 上的世界点。 |
| `hash12`, `hash13`, `noise2`, `noise3`, `fbm2`, `fbm3` | 哈希与值噪声（value noise），外加 4 倍频 fbm。 |
| `window(x, a, b, soft)` | 在 `[a + soft, b − soft]` 内为 1、在 `[a − soft, b + soft]` 外为 0：每个边界处中心一个宽 `2·soft` 的 smoothstep，因此在 `a` 与 `b` 处为 0.5。 |
| `tonemap(c, exposure)` | `1 − exp(−c·exposure)`。用它代替 `clamp`。 |

`onFloor` 与墙面函数能和 `iso()` 精确往返：`project(onFloor(vb, z)) == vb`。正是这一点让着色器内容能与 SVG 线条对齐注册（register）。

**为什么用世界空间。** 图形的透视就是投影本身。如果你在屏幕空间里画效果（凭目测大小的椭圆、以像素计的辉光柱），它就无法与绘图一致：它会漂在地面之上、穿过甲板，或者在相机变化时保持同一形状。一切都从世界点与世界尺寸构建。屏幕尺寸只来自世界单位乘以 `uK`。

以世界单位做抗锯齿：`pixelOf()` 是屏幕上每个设备像素对应的世界单位数。在地面沿深度方向把它除以 `uCam.z`（sinE），在墙面上除以 `uCam.w`（cosE）。把对齐注册的边缘软化约 `0.7 * px`，让它们出来是一个像素的柔：像 SVG 一样锐利，但没有锯齿。在折射拉伸图像的地方，固定的柔度会呈台阶状；波纹水箱启用 `GL_OES_standard_derivatives`，用 `length(fwidth(q.xy)) / length(fwidth(rest))`（偏移点足迹与未偏移点足迹之比，静止时恰为 1）加宽阴影半影，并带 `#ifdef GL_OES_standard_derivatives` 回退。

## 4. iso 相机中的光线步进

相机是均匀尺度的正交（orthographic）相机：投影的两行都标准正交。由此得到三件事。

- **每个像素都是一条直线光线** `p(t) = p0 + t·V`，其中 `V = towardViewer()`，`p0` 是该像素下方任一世界点。取 `p0 = onFloor(vb, h)`，或套件的 `rayOf(vb)`——穿过原点、面向相机的竖直平面上的那个点（Raptor 的光线状态就从这里起步：`S.g0 = rayOf(vb)`）。`t` 越大离观察者越近。
- **到点的距离就是屏幕距离。** 屏幕距离除以 `uK`，就是像素光线到世界点的精确垂直距离。球形闪光、激波壳或辉光球都能由该距离算出，而且世界坐标正确。最近点参数是 `tc = dot(C − p0, V)`。
- **遮挡体之后的可见比例。** 如果光线上第一个不透明表面位于 `tLow`，则中心在 `tc`、宽度 `sigma` 的高斯，其可见比例为 `1/(1 + exp(2.41·(tLow − tc)/sigma))`（Raptor 的 `beyond`）。这给出一个柔的裁切，但属于几何式的。

对一个体积做步进：
1. **沿光线找到第一个不透明命中**：地面，加上 SVG 画出的任何高度场（heightfield）。Raptor 从火焰锥尖端上方到甲板步进 28 次，然后在 `ground(r)` 高度场上二分 7 次——该高度场由与 SVG 车削轮廓相同的数字构建。只有位于命中之前的那段光线（`t > tLow`）是空气。
2. **给步进限定范围**，用一个简单形状（Raptor 用一个半径 132 的圆柱，从地面命中到出口平面）。把每一项在该边界之前开窗到恰好为零，使边界只是一种优化。
3. **带抖动（dither）采样。** 使用 `t = ta + (i + hash12(gl_FragCoord.xy))·dt`，这样 28 个样本不会成带。
4. **按各自弦长归一化每一项**，让中心像素无论步数多少都得到预期亮度。对竖直轴体积有 `r² = u² + (t·cosE)²`，于是半径 `σ` 的高斯沿光线积分为 `σ√π / cosE`：每个样本按 `dt·cosE/(σ·1.772)` 加权。
5. **要预期均匀圆柱看起来更柔。** 它的投影轮廓呈 sqrt 形，失去平面绘图原有的清晰边缘。暗色主题下加一个轮廓缘壳（limb shell，Raptor 在 `0.9·Re` 处的橙色剪切层）把边缘找回。浅色主题下同一壳会围绕空心火焰描出轮廓，所以那里要改为把实体权重推向轴心（见 8.4 节）。

有些项不需要步进：
- 沿光线看去，一个薄的水平圆盘是精确的椭圆。Raptor 的马赫盘（Mach disk）保持解析式：`u² + ((s0 − sc)/tanE)² < rm²`。
- 沿光线积分的薄球壳呈轮廓缘增亮：亮度正比于 `rs·w / sqrt(rs² − d² + 2·rs·w)`。对归一化轮廓取平方，否则内部读作乳白气泡。

## 5. 将着色器内容对齐注册到 SVG

凡是位于某个平面上的东西，都必须与该平面的 SVG 线条精确对齐。
- **同一组数字。** 把 SVG 的平面尺寸与高度通过 `DATA` 传给着色器（波纹水箱传入 `inside`、`outside`、`paper`、`base`、`round`、`z` 以及所有投影物的平面尺寸），并以 `fixed` uniform（其 `FIXED`）一次性交给 `glLayer`。用带符号距离的 `sdRect` 对同样的圆角矩形做测试，让被照亮的纸角终止在所画的圆角半径上。
- **同一平面。** 波纹水箱对玻璃后面的水带使用 `onFloor(vb, uZ.x)`（纸）、`onFloor(vb, uZ.z)`（水），以及 `onWallX(vb, uNear.x)` 与 `onWallY(vb, uNear.y)`（两面近侧玻璃壁）。绝不使用屏幕偏移。
- **透过开口观看时，双重遮罩。** 位于更近平面之开口后面的内容（视镜（sight glass）、窗、凹槽、表圈之下的表盘面）既要被其自身平面遮罩（`onWallY(vb, back)`），也要在开口自身平面内对开口做测试遮罩（`onWallY(vb, face)`），两个软遮罩相乘。只在后平面遮罩时，盲测中的一个视镜把油与玻璃溢出顶缘约 0.4 px 到本体面上，底部偏移则在开口内露出一条裸露本体。把凹槽的内壁画在前层 SVG 里，按朝向着色（`craft.md` 第 10 节），这样两平面之间的偏移读作壁，而不是裸面。
- **只有偏差在动。** 如果效果移位所见之物（透过波纹的折射），只应用与静止状态不同的部分。波纹水箱分别计算被波纹面弯折与被平水面弯折的视线，只应用二者之差。静止时，水下的纸与 SVG 纸精确对齐。视线到纸的路径在距一条腿几各单位之内经过时（`legClear`），偏移也淡化为零，这样折射的纸绝不会从中间层 SVG 画在其之上的腿后面滑出来。
- **高度场与车削轮廓一致。** Raptor 的 `ground(r)` 是甲板圆盘加火焰锥轮廓，半径与高度和 `build.mjs` 中 `G.solidOf` 的轮廓相同。锥体后面的流动被几何性地隐藏，硬边只出现在 SVG 画出真实轮廓线的地方。

在 4× 放大下检查对齐注册（第 9 节）。

## 6. API 与最小页面

```js
import { GLSL_ISO, glCamera, glLayer, smooth, burst, settle } from "../../kit/gl.mjs";
import { kitScript, glScript } from "../../scripts/inline-kit.mjs";
```

| 函数 | 作用 |
|---|---|
| `glCamera(P)` | 套件投影对应的 `{ origin, k, cam }`。在 Node 与浏览器里都能用。放进页面的 `DATA`。 |
| `glLayer({ canvas, fragment, view, camera, uniforms = [], fixed = {}, extensions = [], maxRatio = 1.5, onError })` | 以预乘 alpha 的 WebGL1 上下文在 `canvas` 上编译 `GLSL_ISO + fragment`。`view` 是 viewBox 的 `[W, H]`，`camera` 是 `glCamera(P)`，`uniforms` 列出你每帧发送的 uniform 名称。`fixed` 是 `{ name: value }`，为创建时一次性设置的 uniform（几何、主题常量）。`extensions` 列出 GLSL 扩展（`["GL_OES_standard_derivatives"]`）；上下文支持的每个扩展都会被启用、并作为 `#extension` 指令前置。返回 `{ gl, extensions, set(values), draw(values, time), clear() }`，其中 `extensions` 列出实际启用的那些；WebGL 缺失或着色器编译失败时返回 `null`（日志交给 `onError`，默认为 `console.error`）。 |
| `set(values)` | 设置 uniform 但不绘制。 |
| `draw(values, time)` | 把 canvas 调整到其 CSS 盒尺寸 × `min(maxRatio, devicePixelRatio)`，设置相机 uniform、`uTime = time` 与 `values` 中的每个值，并绘制一个全屏三角形。 |
| `clear()` | 清为透明，仅当确实画过东西时。 |
| `smooth(a, b, x)`, `burst(age, rise, fall)`, `settle(state, target, dt, seconds, calm)` | 每个着色器 uniform 都应搭载其上的 JS 包络（envelope）（第 7 节与 `motion.md`）。`settle` 在仍在移动时返回 true。 |
| `glScript()` | 去掉 `export` 的 `kit/gl.mjs` 源码，用于在 `kitScript()` 之后内联进独立页面。该页上的 live 代码直接使用 `glLayer`、`smooth`、`burst` 与 `settle`，不得自行声明。 |

uniform 值（在 `fixed`、`set` 或 `draw` 中）可以是数字（float）、长度 2–4 的数组（`vec2`–`vec4`），或长度大于 4 的数组或 `Float32Array`——用 `uniform1fv` 发送给 `uniform float name[N]`：24 样本的历史就是一个 `uH[24]`。这些调用不支持 int、矩阵或采样器。超出 uniform 数组容量的数据用纹理：`sampler2D` 默认读 0 号纹理单元，所以在 `layer.gl` 上创建并更新它。波纹水箱把 32 × 2 的历史打包进一张 RGBA8 纹理，每值 16 位（`PACK`、`put`、`unpack`），每层一张纹理，因为每层有自己的上下文。未知名称与被编译器优化掉的 uniform 会被跳过。

你的片元着色器声明自己的额外 uniform，以及一个写入**预乘** `gl_FragColor` 的 `main()`。canvas 必须覆盖与各 SVG 完全相同的盒子、相同的宽高比，因为 `viewOf` 假设单一均匀尺度。

一个最小页面：方形甲板之上发光的热板。热板与一个半径 40 的 SVG 圆环对齐注册（`k.ring(0, 0, 40, 0, P, 72)`），辉光止于甲板的圆角边缘，因为边缘之外没有可照亮的东西。

`build.mjs`：
```js
const P = k.fitProjection(points, W, H, { pad: 28, azimuth: 40 });
const DECK = { x: -90, y: -90, w: 180, d: 180, r: 8 };
const DATA = { W, H, camera: glCamera(P), plateR: 40, deck: [DECK.w / 2, DECK.r] };
const backSvg = k.figureSvg({ width: W, height: H, label: LABEL, body: back });
const frontSvg = `<svg xmlns="http://www.w3.org/2000/svg" class="iso-svg fx-layer" viewBox="0 0 ${W} ${H}" aria-hidden="true">${front.join("")}</svg>`;
const stage = `<div class="fx-stage" tabindex="0">${backSvg}<canvas class="fx-layer fx-gl" aria-hidden="true"></canvas>${frontSvg}</div>`;
const page = k.pageHtml({ title, theme, body: `<style>${CSS}</style>${k.plateHtml({ ...card, body: stage })}`, script: kitScript() + glScript() + LIVE.replace("__DATA__", JSON.stringify(DATA)) });
```

`live.js`：
```js
const D = __DATA__;
const stage = document.querySelector(".fx-stage");
const still = matchMedia("(prefers-reduced-motion: reduce)");
const LIGHT = document.body.dataset.theme === "light" ? 1 : 0;
const FRAGMENT = `
uniform float uHeat;
uniform float uLight;
const float R=${D.plateR.toFixed(1)};
const vec2 DECK=vec2(${D.deck[0].toFixed(1)},${D.deck[1].toFixed(1)});
float deckDist(vec2 p){vec2 q=abs(p)-DECK.x+DECK.y;return length(max(q,0.))+min(max(q.x,q.y),0.)-DECK.y;}
void main(){
  vec2 vb=viewOf(gl_FragCoord.xy);
  float px=pixelOf()/uCam.z;
  vec3 p=onFloor(vb,0.);
  float r=length(p.xy);
  float disc=1.-smoothstep(R-.7*px,R+.7*px,r);
  float onDeck=1.-smoothstep(-.7*px,.7*px,deckDist(p.xy));
  float halo=exp(-pow(r/(R*1.6),2.))*onDeck;
  vec3 col=tonemap(vec3(1.,.42,.12)*(disc*.9+halo*.35)*uHeat,1.2);
  float a=max(col.r,max(col.g,col.b));
  vec4 dark=vec4(col,a);
  vec4 light=vec4(vec3(.62,.24,.08)*a*.8,a*.8);
  gl_FragColor=mix(dark,light,uLight);
}`;
const layer = glLayer({ canvas: stage.querySelector(".fx-gl"), fragment: FRAGMENT, view: [D.W, D.H], camera: D.camera, uniforms: ["uHeat"], fixed: { uLight: LIGHT } });
if (layer) stage.setAttribute("data-gl", "");
const heat = { x: 0, v: 0 };
let target = 1, clock = 0, last = 0, frame = 0, visible = false;
function tick(now) {
  const dt = last ? Math.min((now - last) / 1000, 1 / 30) : 1 / 60;
  last = now;
  clock += dt;
  const moving = settle(heat, target, dt, 0.35, still.matches);
  if (layer) heat.x > 1e-3 ? layer.draw({ uHeat: heat.x }, still.matches ? 0.8 : clock) : layer.clear();
  frame = visible && moving ? requestAnimationFrame(tick) : 0;
}
const run = () => { if (!frame && visible) { last = 0; frame = requestAnimationFrame(tick); } };
new IntersectionObserver((entries) => { visible = entries[entries.length - 1].isIntersecting; if (visible) run(); }, { rootMargin: "120px 0px" }).observe(stage);
```
没有 `uTime` 动画时，这个循环在热量稳定后就停止。有动画场（闪烁、流动）时，只要效果高于其活动阈值且可见就继续运行，否则停止。

## 7. 工艺规则，每条都附上教会它的 bug

### 没有突现
每个 uniform 都沿时间上的平滑包络移动，带起振（attack）与衰减（decay）。绝不在单帧内把值直接设为 1。

- **一次性事件用 `burst`。** 最初的点火闪光在一帧内从 0 直冲满值，把卡片闪成全白。现在每个事件（闪光、激波、地面环、停机爆音、卡片震动）都使用 `kit/gl.mjs` 中套件的 `burst(age, rise, fall)`——smoothstep 起振接指数衰减：
  ```js
  const flash = calm ? 0 : burst(sinceMain, 0.08, 0.13);
  ```
  起振取 50–90 ms（起始处斜率为零）。alpha 函数 `t·e^(−t)` 把衰减与上升绑在一起，白色停留得过久。整卡闪光也要压低：一次闪光加上不透明度 0.9 的 CSS 遮罩（veil）曾把整张卡片闪白六帧，所以遮罩现在是 `flash × 0.32`。
- **阶段切换是由物理量驱动的交叉淡化（crossfade）。** 一个在主羽流增长时消亡的独立预燃项，看起来就像一次瞬间变色。现在颜色、亮度、参差度、火舌长度与溅射都是燃烧室压力的函数，而压力沿弹簧移向每个阶段的目标。
- **颜色有自己的慢包络。** 即便由压力驱动，橙到紫的变化在快弹簧下约 0.23 s 就穿过了它的 smoothstep 窗口，而且亮度在 130 ms 的闪光内下探 33% 又回升：一个被闪光遮住的剪切。修复：
  - 色相有自己的临界阻尼包络，即套件的 `settle`：`settle(burn.hue, lit && burn.pc > 0.2 ? 1 : 0, dt, lit ? 0.32 : 0.5, calm)`，作为 `uHue` 发送。
  - 亮度是单独的 `uGlow`，在 JS 中计算，让它在色相上升时从 1.15 单调降到 1.0。
  - 压力弹簧为临界阻尼（刚度 20、阻尼 2√20），因此不会过冲到 318 bar。
- **淡出期间亮度只降不升。** 停机时羽流曾以橙色火焰重新窜起，因为 `b < a` 的 `mix(a, b, hue)` 在色相衰减时又升了回去。现在辉光水平在停机开始时冻结，之后只被下降中的量缩放。
- **改动一个弹簧时，复查它的邻居。** 压力弹簧改为临界阻尼后，预燃气舌几乎消失。那一支需要自己更硬的临界弹簧（刚度 64）。
- **一次性标志属于事件。** 停机后 2 s 内重新点火会丢掉下一次停机爆音，因为爆音的时钟只在一段安静期之后才重置。现在由每次燃烧的 `popped` 标志决定。若在 `ignite()` 里重置时钟，会截断一个仍在淡出的爆音。
- **变化要经过介质传播。** 波纹水箱的频率变化时，新波长不会一次性重铺整个场。页面每帧记录波源的真实相位、ω、k、每 mm 阻尼与 1/群速，保留相隔 50 ms 的 32 个样本，并把它们作为 32 × 2 纹理上传给两个着色器。每个点用它自己历史的群速做一次不动点迭代，求出年龄 `r / groupSpeed`，再从该年龄读取 k 与阻尼，于是新波长像真实情况那样从点波源（dipper）向外扩散。
- **变化期间保持相位连续。** 最初版本用 `θ = k(age)·r − φ(t)`。扫频期间，`r·dk/dt` 项让远处的波峰竞速并混叠；而频率移动时对振幅做阻尼（`calm`）又把图案抹白：纸面平均亮度下跌了 17%。现在用 `θ = kE·r − uPhase + Ψ(age)`，其中 `Ψ` 是自该样本以来，波源在其 ω 之外额外获得的相位（`packHistory` 里的 `lag`）。在固定点上，波峰于是只以 ω 变化的速度移动，因此不需要振幅阻尼；再加一个小的运动模糊项（`gMove`，波峰每帧移动多远）在快速变化期间柔化焦散。
- **首次亮起要淡入，且浅色下只应变亮。** 波纹灯在首次出现时用 0.9 s 淡入（`smooth(0, 1, (lampClock − 0.15) / 0.9)`）。暗色主题下，SVG 静水填充通过 CSS 变量（`--rt-lamp`）在同一数值上淡出。浅色主题下，最初版本从裸纸开始，于是灯亮起时波谷的墨色（ink）反而让水箱变暗，静水填充又进一步压暗它。现在只要 WebGL 运行就隐藏静水填充，纸从第一帧（灯为 0）起就按其未点亮的灰色（`off`）绘制，并由 `uLight` 向点亮后的墨色混合，因此首次亮起只会变亮。对纸上的发光体，同一条规则反着读：光进来时它的墨色只增不减，画面绝不先变灰白、也绝不先变暗（见 8.2 节）。

### 没有生硬的空间截断
每一项都在任何边界之前到达零：一个 `if`、一个包围形状、一个平面或 canvas 边缘。`if` 只用作优化，且仅在该项可证明已近似为零处。
- 一个珠状项曾被 `if` 在其区间两端生硬截断。现在它用 smoothstep 淡入淡出。
- 羽流曾有一个 `if` 盒、一个 `s < .01 continue` 与一个 `step(ax, R0)` 光晕遮罩。三者都显现为接缝。
- 一个屏幕空间辉光柱从喷管直通 canvas 底部。canvas 最后几行仍有 11–19/255 的 alpha，于是它在甲板下方以一条笔直的水平线收尾。现在它是以地面为界的世界空间光晕，最后 10 行与 10 列读数恰为 0。
- 一个宽 `exp()` 闪光与地面坐标的蒸汽在 canvas 边缘仍可见。现在二者都在 canvas 内部衰减到零。
- 单元噪声（cell noise）只看自己单元的内部，因此偏移加半径加柔和尾部越过单元半宽的特征，会在单元边缘被齐平裁掉。让 `offset + wobble + radius + tail` 小于半个单元，或测试 3 × 3 相邻单元。盲测中的一个气泡场让中心偏在离单元中心 0.31 个单元处，触达 0.86 而半宽只有 0.85。
- 使用 `atan` 的径向噪声沿某个角度有接缝。改用方向向量的 `noise3(vec3(dir, r·k − t))`。
- 带百分比停靠点的 CSS `radial-gradient` 结束在元素之外。使用 `circle closest-side`，让渐变结束在元素内部。

### 世界空间，处于相机之中
- 点火激波曾是围绕投影出口的 2D 圆。它的下弧越过甲板上方与下方，并涂盖两根立柱。现在它是一个真实的球：每个像素光线在两个根处击中它，地面以下的根被柔和隐藏，每个根按轮廓缘（limb）加权，并由发动机代理决定送往后层还是前层 canvas。
- 地面溅射曾平铺在喷淋环之上，让环管失去了体积。该环现在是一个圆环遮挡体，用 `max(dot(n, toPlume), 0.)` 照明。
- 近侧立柱身后的雾霾涂盖了它的正面。现在立柱在着色器中是不透明盒。

### 图层与混合
- 暗色主题下，只发光的 canvas 用 `mix-blend-mode: screen`，写入预乘颜色且 alpha = 最大通道。同时会压暗覆盖物的 canvas 保持普通混合：波纹水箱的水给透过它看到的纸染色，屏幕混合会静默丢弃这一点。浅色主题下每个 canvas 都是普通混合，其墨色从纸张应显示的颜色反解出来（第 8 节），而不是从暗色主题颜色压暗。
- 画在白热核心之上、普通混合的模糊橙色缘，在喷管唇下读作一圈暗棕领圈；稍微偏离唇部绘制时又形成双弧。着色器之上的辉光轮廓现在放进单独的覆盖 SVG（`.rp-glow`），暗色下屏幕混合，位置精确在唇部。浅色下没有任何东西能增加光，所以缘是墨色本身的一部分：围绕浅色核心、位于物体边缘的一条饱和色带（见 8.1 节）。
- 本应经过钟形喷管前方的蒸汽总在它的后面。现在用发动机代理把它拆分到两个 canvas（第 2 节）。

### 色调映射（tonemap）与抖动
- 用 `tonemap()`，绝不用 `clamp`，让明亮的核心滚降而不是被削平。
- 没有输出抖动时，大的柔和项变成 8 位阶梯（每级连续 18–34 px），屏幕混合又使之更糟。在色调映射之后加约 ±1 LSB 的动态三角噪声抖动，并加门控让空像素保持恰好为 0（Raptor 的 `inkOf`）：
  ```glsl
  col=tonemap(col,1.15);
  float m=max(col.r,max(col.g,col.b));
  float n=(hash12(gl_FragCoord.xy+fract(uTime*.61)*91.)+hash12(gl_FragCoord.yx*1.37+fract(uTime*.37)*57.)-1.)/255.;
  col=max(col+n*smoothstep(0.,1.5/255.,m),0.);
  ```
- 给精细细节做带限。波纹水箱把投影波长低于 2.5–5 设备像素的每道波淡出（其 `fine` 项）。否则 28 Hz 的图案会变成摩尔纹。

### 只在有意义时才渲染
- 设备比封顶 1.5（`maxRatio` 默认值）。只在效果活跃时绘制：Raptor 清空两个 canvas，并让循环在每个包络低于 1e-3 后停止。离屏时用与图形其余部分相同的 `IntersectionObserver` 停止，`resize` 时重绘。
- 永不静止的效果（回流中的气泡、常明火（pilot flame））让着色器在空闲时保持绘制，但图形其余部分不必：拆分循环，让 SVG 工作（深度排序、仪表、读数）只在其中有变化时运行，效果单独运行，慢的话可降低频率。盲测中的一个台架，因为一个包络停在 0.54 而不是 0，它的整个 tick 永远保持 60 fps。
- 用 uniform 检查门控高成本项（`if(uSmoke<.0004&&uVent<.0004)return vec3(0.);`）。这些检查对整次绘制都相同，所以关闭时不产生开销。

### 减少动态
直接跳到有代表性的稳态（Raptor 跳到主级工况，波纹水箱跳到其当前 f 与 d），冻结 `uTime`（Raptor 用 0.8），把每个事件包络清零（没有闪光、激波或震动），只在 uniform 的 JSON 变化时重绘。图形仍必须响应输入。

### 回退
`glLayer` 会返回 `null`，所以每次调用都要做防护。WebGL 运行时给 stage 打标记（`data-gl`），让 CSS 能在 SVG 回退与着色器之间切换。通过打桩（stub）WebGL 来测试（第 9 节）。

## 8. 浅色主题

浅色是交付主题。浅色页面应当得到为纸张设计的着色器，而不是最后才转换的暗色底板辉光。等距图形站上有七个图形是先在暗色底板上建好再转换的：池式反应堆、铸造浇注、加力燃烧室、玻璃炉口（glory hole）、托卡马克、等离子切割机与锻炉。每次转换都花了三轮评审，本节每条规则都是其中的一条发现。

Raptor 的浅色羽流（`examples/raptor-engine/live.js` 中的 `inkOf`）展示了不该怎么做。它把每个色相乘以固定的暗红 `vec3(.62,.34,.3)`，并把 alpha 设为亮度，结果最热的气体反而最暗、最不透明。评审者称之为"红墨烟雾"。不要照抄。波纹水箱的灰墨（`uPale`）仍是非辉光类光的好模型：一条色调曲线 `1 − uExpo.x/(1 + (e/uExpo.y)^uExpo.z)` 把波峰带到接近纸白、把波谷带到套件的灰色系，而且它的水只写入相对平水的差异。

### 8.1 纸上的墨色
在白色之上，canvas 是普通混合、预乘的，绝不用 `screen`，因为屏幕混合叠在白上还是白。没有像素能比纸更亮，所以光必须靠**饱和度与局部对比**来读出：
- **辉光是饱和的实体、一个近白的浅色核心、外加一圈彩色缘。** 浅色核心之所以读作炽热，是因为它周围的实体饱和、再往外的底色更暗。
- **光落到的东西接受染色**（类似正片叠底（multiply），alpha 封顶）。它不变亮。
- **容纳光的体积更暗，且有物理上的原因：** 烟灰（soot）、耐火材料、深水或阴影（见 8.3 节）。它绝不是黑箱。

**先设计纸张应显示的颜色，再反解墨色。** 写一条从光能量到读者在白底上应见颜色的渐变表（ramp）。然后对白底上的普通合成——显示为 `rgb + (1 − a)`——求逆：
```glsl
vec4 inkFor(vec3 seen,float cover){
  vec3 absorbed=1.-seen;
  float a=max(max(absorbed.r,max(absorbed.g,absorbed.b)),cover);
  return vec4(vec3(a)-absorbed,a);
}
```
在白底上，它恰好合成出 `seen`。
- **反应堆。** `inkOf` 用它的 `paperOf(e)` 渐变表做到这一点，其中 `e = 1 − exp(−2.3·max(lin))`。渐变表从白 → `TINT (.85,.93,1)` → `BODY (.2,.46,.94)`（e 在 .12–.5）→ `RIM (.2,.68,1)`（.4–.65）→ `ICE (.9,.98,1)`（.6–.9）。
- **加力燃烧室。** 它的 `paperOf` 混合 白 → 实体色 → 浅色核心，然后以同样方式求逆。
- **`cover` 是热端的 alpha 下限：** 反应堆用 `WASH·smoothstep(.6,.9,e)`（WASH .52），加力燃烧室用 `COVER·smoothstep(.45,.92,m)`（COVER .94）。没有它，近白核心的 alpha 接近 0，其后的 SVG 线条直接透出，看起来是洞而不是光。

其余图形用 `vec4(colour·a, a)` 的形式：`colour` 来自渐变表，`a` 来自能量的饱和曲线。托卡马克的 `glowInk` 是一例。两种形式都可行；`inkFor` 还额外保证读者看到的颜色。

**每种光一种墨色，在着色器内合成。** 保持各自独立的累加器，给每种光自己的墨色，并从后往前用 `vec4 over(vec4 top,vec4 under){return top+under*(1.-top.a);}` 叠放：

| 项 | 墨色 | 出现在 |
|---|---|---|
| 发射，热物体本身 | 渐变表实体色加浅色核心，alpha 最高 .96–.97 | 等离子切割机 `bodyInk`、锻炉 `barInk`、托卡马克 `glowInk` |
| 落在表面上的光 | 色相被拉暖或拉冷，alpha 封顶在 .5–.8 | 铸造 `castInk` 与 `tintInk`、托卡马克 `tintInk`（.5）、等离子切割机 `tintInk`（.75–.8） |
| 雾霾：蒸汽、烟、烟气、击穿气体 | 灰色或轻染色，alpha `1 − exp(−density·k)` | 等离子切割机 `hazeInk`、托卡马克 `gasInk`、锻炉蒸汽 |
| 火花（spark）与条痕 | 叠在所有东西之上 | 铸造与锻炉 `sparks` |

等离子切割机的 `lightInkOf` 是最清晰的例子：`over(arcCore, over(bodyInk(body), over(arc, over(fumeInk, over(steamInk, tintInk(tint))))))`。如果先把能量求和、再对和上墨，所有东西都得到同一个色相，浅色核心就丢了。

**收尾。**
- 每次求和后钳制 `rgb ≤ a`：加力燃烧室曾出现 1 LSB 的预乘溢出。
- 对最终预乘值做抖动，rgb 与 alpha 都要，用 `smoothstep(0.,1.5/255.,a)` 门控（玻璃炉的 `paperOut`、铸造的 `dithered`）。
- canvas 边缘保持恰好为 0。
- 在能量到达渐变表之前先做色调映射（`tonemap`，或 `1 − exp(−k·e)`），绝不用 `clamp`。

### 8.2 墨色随光增长
暗色下，更多的光只做加法。在纸上，更多的光意味着更多的墨，粗心的公式可能在光减弱时加墨、或在光增强时减墨。以下是最常被评审者发现的错误：
- **对完成的墨色只淡化一次。** 把预乘墨色乘以 `u`，是从裸纸到完整画面的精确线性交叉淡化，在每个像素上都单调：反应堆的 `ink*=uFirst`、玻璃炉的 `paperOut(overInk(gas, ink)*uLit)`。玻璃炉口曾在每一项上墨之前先将其淡化（`holeOf` 中的 `*uLit`，取料孔与球体）。浅色核心先于其饱和实体到达，然后底色变暗，于是首次亮起先变浅再变暗。每种光墨色都从未淡化的色调构建（其 `litOf()` 在浅色下返回 1），只对结果淡化。
- **辉光的淡化要能量与 alpha 一起进行。** 只淡 alpha，浅色核心保持浅色、变稀的辉光变成乳灰：托卡马克的接触带曾如此。只淡能量，辉光沿渐变表倒退回去。托卡马克现在把能量缩到一个下限、其余交给 alpha：`hot+=glowInk(c*level*(.3+.7*hold))*hold*(1.-hot.a)`。该带淡出全程保持粉紫色（pink-mauve）。
- **在稳定亮度下取色相。** 能量下降时，色调映射让色相变红变暗，而 alpha 接近上限时墨色上升：加力燃烧室的腔体在火焰熄灭时墨量重了 55%。修复：
  - 取色相前把燃烧除以局部点亮强度（`burn/lit`，`lit = la.x·glowOf(la.y)/LIT_GLOW`）。
  - 按各部分的能量份额混合色相。
  - alpha 与能量绑定：`cap·(1 − exp(−1.6·e/cap))`。
  - 低覆盖时把色相向 `EMBER` 暖色偏移，并按 `inkOf(hue)/inkOf(ember)` 缩放 alpha，让墨色保持不变。
  
  由此，切割时窗口的墨色从 1894 单调降到 155。
- **交接就是普通的预乘混合。** 加力燃烧室的尾焰有两份拷贝——未点火的蓝与点亮的暖——用 `mix(blue, warm, lift)` 交叉淡化。在那里用 `max(alpha)` 做覆盖使未点火尾焰的墨色翻倍。还要对每份拷贝做门控，让它拾取不到另一份的项：未点火激波盘曾泄漏进暖色拷贝，成为一块奶油色斑块。
- **相乘的包络可能在淡出期间上升。** 如果首次亮起的包络仍在上升而效果自身的边缘在下降，淡出会变亮：托卡马克在按下 Home 后的前半秒出现过这种情况。效果下降期间冻结首次亮起的增长。
- **从 SVG 回退的交接要精确，** 否则着色器进场时画面会下陷（见 8.5 节）。

### 8.3 给光一个底色
白底上四周无物的辉光读作一块污渍。给它真正应有的底色：
- **SVG 中的腔体采用更暗的一组令牌。** 一个组类为其内部所有元素重定义套件令牌：
  - 加力燃烧室的 `.ab-inside`：纸 `#4a453f`，暗部 `#3c3833` 到 `#544e47`，线条 `#5e5952` 到 `#9b958b`；
  - 托卡马克的 `.tk-cavity`：`#66646e` 到 `#8b8992`；
  - 铸造的烟熏孔色调（`--fd-bore-*-sooted`，`--anatomy-shade-0` 与 `--fd-soot-ink: #2f2a25` 的 `color-mix`）；
  - 玻璃炉口的 `--gf-void` 与 `--gf-bore`；
  - 反应堆的水柱渐变（`--rx-water-top`、`-mid`、`-deep`，从 `#edf2f4` 到 `#c9d7e0`）。
  
  暗色下同样的令牌设为 `inherit`。四个暗部色调都保留，于是内部仍有形体，读作耐火材料、烟灰或深水，而不是一个洞。
- **底色实时接受光的染色。** 同样的令牌由循环写入的变量做朝光色相的 `color-mix`：`--ab-flame` 在面上 30%、在线条与背板上 55%，`--tk-cold-mix` 用于击穿的紫色。于是即使没有 WebGL，SVG 底色也随火焰变暖。
- **着色器的底色与 SVG 的相等。** 着色器画不透明腔体处（玻璃炉口的 `holeOf = overInk(emberInk(t,1.), vec4(SOOT,1.))`），读取令牌而不是重抄数值。铸造添加一个探针 path，把填充设为每个 `var(--fd-…)`，读取 `getComputedStyle(probe).fill`，并把结果作为 `fixed` uniform 传入（其 `mount.ts` 中的 `paintOf`）。
- **光源周围的光是带长尾的染色。** 托卡马克的容器在平顶处读作灰色，直到一个长程晕染（`skin*(.14*exp(-gap/(.22*U))+.17*exp(-gap/(.9*U)))`）把它的壁染成品红。在立柱上实测：等离子关闭时 `#6e6c76`，晕染 .17 时 `#786778`，这看起来是被照亮的。.1 时仍读作灰色。
- **空旷空气中不放辉光。** 浅色下，悬空流柱或薄板周围的光晕是纸上的一块污渍（铸造的流柱与缺口辉光曾如此）。浅色下去掉空气中的光晕。把轮廓缘保留在物体内部，把染色留在光能到达的表面上。
- **光只落在光源看得见的地方。** 反应堆的表面光曾泄漏到屏蔽体的切面与压顶上，直到其密封测试改用水箱唇的真实顶部（`RIMZ`）而不是甲板。等离子切割机曾照亮尾焰下端所在位置下方的板条仓，而尾焰明明在隔壁仓；其 `bayLight` 现在按各仓占尾焰柱的份额照明，于是光随尾焰移动在仓与仓之间交叉淡化。

### 8.4 按现象分类的配方
热物体按暗红 → 橙 → 黄 → 近白的顺序行进，只有这一种顺序，别无其他。在模型里保留一张停靠点表（stop table），把它编译成 GLSL，回退也用同一张表：铸造的 `INK_STOPS`（`inkRampGlsl`，SVG 用 `glowInk`）、锻炉的 `STEEL_INK`（`steelInk`、`steelInkOf`）。
- **熔融金属、锻钢、坯料、切缝。**
  - 按亮度温度索引该表，让暗淡的远墙与明亮的浇注从同一张表读取。铸造的 `brightOf` 从绿色通道反解维恩（Wien）位移。
  - 锻炉的停靠点：560 °C `(.45,.09,.05)`、680 `(.56,.07,.04)`、780 `(.78,.10,.03)`、950 `(.95,.38,.06)`、1150 `(1,.78,.12)`、1260 `(1,.93,.78)`。
  - alpha 在可见辉光处打开：锻炉 `steelGlows` 覆盖 560–820 °C，或铸造 `INK_FROM`/`INK_FULL` 覆盖 470–590 °C，从金属自身底色调交叉淡化进入。
  - 缘部更冷更暗（`steelInk(t − 140)`，铸造 150 K 的轮廓缘降落）。奶油色核心只出现在最热的中心（`steelPale` 覆盖 1050–1280 °C，取 .8）。
  - 暗色表皮位于辉光之下并随金属冷却而增长：锻炉的 `barInk` 从氧化物 `(.28,.29,.32)` 到表皮 `(.24,.22,.23)`，覆盖最高 .86。
  - 评审者对铸造流柱的形容："奶油色核心带橙色表皮"。
- **玻璃聚光（glass gathers）。**
  - alpha 是一个亮度门限：`GLOW_SEEN` .008–.028，约 600–710 °C。
  - 颜色是 `pow(hue, mix(1.6,.8,warm))*mix(.62,1.,warm)`，沿深樱桃红 → 红 → 橙行进。
  - 浅色核心（`CORE_PALE (1,.88,.62)`）位于该像素自身发射最亮处（`cored`，亮度 .14–.24），而不是固定半径。
  - 冷表皮（`SKIN_COOL (.3,.11,.07)`）只退缩。
  - 最初版本有一个橙红色阶段。红色向白混合就是橙红色，没有任何温度看起来像那样：任何东西变浅之前必须先经过橙与黄。
- **火花。**
  - 饱和的头部加更暗更冷的尾部。等离子切割机的 `tailCooling` 把尾部温度设为头部的 0.62；锻炉的尾部墨色是 `(.42,.07,.02)`，alpha .75、上限 .8。只有火花最年轻的三分之一才带浅色核心（`u < .35`）。
  - 条痕宽度至少约 1.25 设备像素（`max(.35+.6px, 1.25px)`），否则斜向条痕在 10–24× 下成串珠。
  - 白底上火花就是墨，所以在更热的发光体之上必须消失。铸造把火花墨色乘以 `1 − cover·smoothstep(sparkT−60, sparkT+60, tb)`，对金属与炽热孔都生效。
  - 从尾焰后面经过的火花按深度淡出：`cover*smoothstep(-.4,.4,tJet-tS)`。
- **火焰与尾焰。**
  - 饱和的橙黄实体、半透明的边缘，浅色下权重推向轴心：`mix(1.,1.3-.6*smoothstep(0.,rf,r),uLight)`。
  - 裁掉轮廓缘幔（`mix(.8,.3,uLight)`）。给暗色羽流提供边缘的轮廓缘壳（第 4 节）在纸上会描出轮廓，让火焰显得空心。
  - 参差感只留在外侧部分，加宽热轴，浅色核心放在喷管处，在长度的 .4–.9 之间淡出。
  - 检查横截面。加力燃烧室最终火焰从边到边的墨色读数为 `0 2 13 33 31 49 78 72 78 52 40 32 8`。
  - 未点火的尾焰是一条淡蓝柱。激波菱形是浅奶油色圆盘（`CREAM (1,.95,.86)`），带暖缘与柔边（约圆盘半径的 .65）。推得更强时，第二个菱形读作旋钮或垫圈，所以它保持为一个淡淡的透镜。
- **烟与蒸汽。**
  - 浅灰，绝不把暗色辉光墨铺在它上面。边缘 `(.84,.87,.90)`、云顶 `(.97,.975,.98)`、底面 `(.72,.76,.81)`——由云团自身的自阴影产生；alpha `1 − exp(−density·k)`。
  - 靠近热源时，把云团向浅琥珀色暖化（`mix(steelInk(T), CREAM, .5)`）。锻炉最初把辉光墨铺在云上，评审者看到一块橙红色团块。
  - 淬火气团是一个离开水槽的 `burst`：从水面上方 4 个单位爬到约 40 个单位，途中减速（锻炉 `shader.ts` 中的 `PUFF`），约 0.15 s 内越过缘口，宽度从 7 增到 13，随增大而变稀，并共享该柱的高度窗口，因此绝不生硬截断。
- **等离子体（托卡马克）。**
  - 饱和的品红-紫：`inkBody(chroma)=pow(chroma,vec3(1.8))*vec3(.93,.86,.97)`。
  - m 超过 .55 后核心变浅，向白过渡并保留 8% 的彩度。
  - alpha：`.97*smoothstep(.02,.42,m)*(1.-exp(-2.6*(m+2.*m*m)))`。
  - 击穿气体是紫色雾霾，`(.52,.44,1)` 到 `(.74,.68,1)`，alpha 至多 .64。透过剖视看时，容器内部使用腔体令牌组。
- **切伦科夫（反应堆）。**
  - 蓝染水柱中鲜艳的青蓝：8.1 节的纸色渐变表，叠在 SVG 水渐变之上，热端由 `WASH` 设下限，让核心盖住其后的燃料管线。
  - 超临界后压缩脉冲（`1.+.75*log(level)`），让 1.5 GW 的脉冲点亮水池而不冲爆整张卡片。
  - 大厅墙上的辉光使用单独的、更弱的墨色（`HALL_INK` .35）。
- **等离子弧（等离子切割机）。**
  - 紫色电弧 `vec3(.6,.34,1.)` 加浅色核心 `vec3(.96,.95,1.)`。
  - 板材与水上的暖染色墨：`mix(vec3(1.,.5,.08),vec3(1.,.68,.3),…)`，alpha `.75*(1.-exp(-3.2*sqrt(m)))`。

### 8.5 浅色下的 SVG 与回退
- **主题只读一次。** 构建时确定主题（`pageHtml({ theme })`，或 React 包装器上的 `data-theme`；在等距图形站上，图形的 build 返回 `theme`，`LivePlate` 把暗色底板包进它自己的 `.iso[data-theme="dark"]`）。live 代码从最近的 `[data-theme]` 读取主题（`root.closest("[data-theme]")?.getAttribute("data-theme") !== "dark"`），并把 `uLight` 作为 `fixed` 传入。每个仅限浅色的改动都用 `uLight` 门控（`mix(dark, light, uLight)`，或按该 uniform 走分支），让暗色输出保持原样。如果代价不大，就让暗色继续可用。
- **不写颜色字面量。** 如果图形在 CSS 里硬编码暗色，把它们转成带暗色覆盖的令牌：`.x { --x: light }` 然后 `.iso[data-theme="dark"] .x { --x: dark }`。必须与某个令牌一致的着色器常量来自探针（见 8.3 节）。
- **嵌套主题。** `.iso:not([data-theme="dark"]) .x` 也会透过暗色底板周围的浅色页面命中。反应堆的暗色回退就是这样被乘成黑色的。把暗色规则放在浅色规则之后、同等特异性，或以底板自身主题为键：锻炉用 `.iso-plate[data-plate="light"]`。
- **回退如何混合。** 其饱和染色用正片叠底——既压暗又让其下的线条可见。其浅色核心用普通混合，因为正片叠底无法提亮：反应堆的无 WebGL 核心是一个从 .85 处的 `#fff` 到 .3 处的 `#dcefff` 的径向渐变，普通混合。
  - 回退颜色与着色器来自同一张渐变表：铸造的 `glowLightCss`，以及玻璃炉的 `gatherInk`——它与着色器的聚光墨色完全一致。
  - 浅色令牌上加不透明度会褪成灰色，所以给回退饱和的色相：托卡马克的回退环设置 `--anatomy-lit: #e0409c` 与 `--anatomy-hi: #f27ec0`。
- **回退形状变化用交叉淡化。** 托卡马克的无 WebGL 等离子环是 11 个预计算环（`LIMITED_RINGS`）。相邻环用和为 1 的线性权重交叉淡化，画在与着色器相同的缓动半径上。用 6 个环时，淡化中途出现过一条双描边。
- **带缝隙的东西不要放到半透明墨下面。** SVG 画的东西都会透过不完全不透明的墨显现。锻炉的受热带曾是一组笔画、笔画之间露出纸，透过钢条 14% 的透明度印出灰色条纹。现在它是一个以热渐变填充的面。
- **从回退到着色器的交接要精确。** 首次亮起时，canvas 墨色按 `u` 上升，而其下的回退退场。回退上用线性 `1 − u` 会让淡化的中段比两端都浅。要精确的线性交叉淡化，把回退的不透明度设为 `(1 − u)/(1 − u·α)`，其中 α 是 canvas 墨色在关键处的 alpha：玻璃炉的 `--gf-gather-fade`，α 取聚光的中心停靠点。其余回退在着色器几乎进场之前保持全强度：`clamp(0, calc(var(--gf-fallback, 1) * 12), 1)`。
- **着色器画的每个物体都有回退。** 铸造的浇注流柱原本没有，所以无 WebGL 时缺失。现在它是一个 SVG 轮廓，每帧用着色器读取的同一份历史重建。

### 8.6 墨色与线条相遇之处
canvas 叠在后层 SVG 的 0.6 px 笔画之上。浅色下，对位不准的边缘显示为笔画之上的浅色缝隙或彩色线，而不是暗色接缝。
- **止于笔画内缘。** 把面的墨色在笔画之下推到全值，并终止于笔画内缘——深入线宽一半处：套件的 0.6 px 线条即 0.3 vb。要测量计算出的笔画宽度；锻炉的评审者猜的是 1.1。例子：反应堆的 `cutFade=smoothstep(-SEAM-.5*px,-SEAM+.5*px,gap)`（`SEAM=.3`），锻炉的 `strokeMeet`。染色同样内缩半个笔画（铸造的 `ringLight`），让它绝不涂到线之上、也不在线内留下浅色条带。
- **没有描线的地方不要清墨。** 锻炉的遮罩层曾在两个面共享、SVG 又不描线的折痕处清墨，钢条沿线出现白色接缝。这类折痕改画进墨色：×0.72，宽度至少 0.8 canvas 像素（更锐的会呈点状）。
- **相邻光线不一致处做超采样。** 反应堆追踪四条亚像素光线，当它们的命中、深度、种类、法线或切面份额不同时（`apart`）取 16 个样本，使切面边缘不成阶梯。dpr 为 1 时，canvas 像素与笔画差不多宽，所以仍保留一个像素的柔。
- **从画幅外进入的物体**可以在屏幕空间里从 canvas 边缘淡入。铸造的浇注流柱在 11 vb 内淡入，起点在 canvas 顶下方 10.5 设备像素处，于是边缘读数仍为 0，流柱看起来来自上方。

### 8.7 评审者所见与修复

| 浅色下所见 | 原因 | 修复 |
|---|---|---|
| 红棕色"墨色烟雾"羽流 | 色相 × 暗色乘数，alpha 设为亮度 | 纸色渐变表：饱和实体、浅色核心、alpha 下限（8.1） |
| 火焰被画成轮廓、内部空心 | 暗色轮廓缘幔 | 把实体权重推向轴心，裁掉缘幔（8.4） |
| 首次亮起先变浅再变暗 | 各项在上墨前逐个淡化 | 对完成的预乘墨色只淡化一次（8.2） |
| 辉光淡出途中变成乳灰 | 浅墨色只淡 alpha | 能量与 alpha 一起调暗（8.2） |
| 火熄灭时腔体反而变暗 | 色调映射使色相变红而 alpha 不变 | 稳定亮度下取色相，alpha 由能量决定，向余烬色对齐（8.2） |
| 玻璃或钢冷却时出现橙红色阶段 | 红色向白混合 | 黑体顺序的渐变表，过黄之后才变浅，核心放在发射峰值处（8.4） |
| 火花看起来像金属上的脏东西 | 更热发光体上的暗墨 | 在更热物体上淹没火花；饱和头部、更暗尾部（8.4） |
| 放大时火花条痕成串珠 | 尾部太细 | 宽度至少约 1.25 px；尾部少变细（8.4） |
| 火花透过尾焰可见 | 对尾焰没有深度测试 | 在尾焰后方按深度淡出（8.4） |
| 光晕在流柱周围污了纸 | 暗色主题的辉光出现在空中 | 浅色下空中不放光晕（8.3） |
| 蒸汽中一块橙红色团块 | 辉光墨铺在云上 | 把云团向浅琥珀色暖化（8.4） |
| 等离子开启后容器仍读作灰色 | 染色衰减太快 | 加长程晕染项（8.3） |
| 边缘沿线有浅色镶边或白缝 | 墨色停在笔画之前，或清掉了未描线的折痕 | 墨色止于笔画内缘；把折痕画进墨色（8.6） |
| 尾焰中一块奶油色斑块 | 一份拷贝拾取了另一份的项 | 对每份拷贝做门控（8.2） |
| 暗色底板的回退不可见 | `:not([data-theme="dark"])` 规则透过浅色页面命中 | 暗色规则放在最后，或以底板为键（8.5） |
| 回退辉光发灰发淡 | 浅色令牌加不透明度 | 用饱和令牌；染色用正片叠底、浅色核心用普通混合（8.5） |
| 热面上透出灰色条纹 | 半透明墨下带缝隙的 SVG 笔画 | 用面填充（8.5） |
| 光落在光源看不见的面上 | 表面光没有用真实遮挡体 | 使用与 SVG 所画相同的几何（8.3） |

## 8b. 旋转组上的着色器
在 3D 图形（`references/3d.md`）里，零件作为刚性组移动，所以着色器必须跟随它们。
- **锚定到零件**：`controller.anchor(group, restPoint, restDir)` 每帧给出 `{ world, screen, direction }`。把它们作为 uniform 传入。LED 就是这样贴着弯曲的指尖移动、火焰就是这样留在破裂线的端头、火花源就是这样落在指节上的。
- **锚定到表面**：前置 `glslPose("Dial")`（单个组用 `GLSL_TURN`）并传入 `TURN.poseUniforms(controller, "dial", "uDial")`。于是 `vec3 q = toDial(onFloor(vb, z))` 就是该组静止世界坐标中的表面点。转台上积起的油洼、被光显现的纹理与焦痕都随之旋转。
- **世界空间始终是世界空间。** 烟、离开源之后的火花与台面上的油从不接触组变换。在锚点的世界位置发射，然后在世界中积分。
- **活动零件造成的遮挡。** 前层 canvas 位于某个活动层之上。用绘制在效果所属零件之后的零件来遮罩它的墨色：`TURN.cover(controller, [partName])` → `uCoverEdge`、`uCoverCount`，然后 `ink *= 1. - coverOf(vb, .7 * px)`。这些遮罩是外扩 0.3 vb 的精确凸轮廓，所以墨色止于遮挡体的笔画。六个多边形、每个十条边。读取 `MAX_FRAGMENT_UNIFORM_VECTORS`，让 SVG 回退保持在 128 以下。
- **位于活动层之间的后层 canvas**不需要遮罩：把受光表面放进它自己的活动层、置于遮盖它的零件之下（旋转表盘的面在手之下）。
- 向量 uniform 数组（`uniform vec3 uSpark[24]`）通过 `draw` 接受 `Float32Array`。
- **运动零件上的 SVG 回退**是一个 `T.billboard({ part, at, normal, svg })`。其标记以 viewBox 像素为单位、以投影点为中心，而不是世界单位，所以世界尺寸要乘 `G.scaleOf(P)` 缩放：为半径 1.1 的透镜准备的辉光应写成 `<circle r="${(1.1 * G.scaleOf(P)).toFixed(2)}"/>`。若写成 `r="1.1"`，会画出约为透镜三分之一大小的针尖点。用 `--no-webgl` 与 4× 缩放检查它。

## 9. 验证

凡会动的东西用 `scripts/drive.mjs`；`capture.mjs` 现在也支持 WebGL，用来对着色器页面拍静帧没有问题。用法见 `references/verify.md`。在常规检查清单之上：

- **覆盖每个过渡的连拍表（contact sheet）：** 起点、每次阶段切换、稳态与停止。重型 WebGL 页面上的实时连拍太稀疏，因为无头捕获一次耗时 60–200 ms 且阻塞动画帧。先用 `["slow", 0.05]` 减慢页面时钟（对 `requestAnimationFrame` 与 `performance.now` 做 20 倍慢放），拍表后再 `["slow", 1]`。要捕捉第一帧就开始的东西（如首次亮起的淡入），用 `--preload` 传入一个含 `window.__driveSpeed=0.04` 的文件，让页面从第一帧起就慢速运行。用 `["until", "document.querySelector('[data-readout]').textContent.includes('main stage')", 8000]` 等待某个阶段，而不是猜测的 `wait`。每次拍表前按住任何空闲巡演：它的恢复计时器走墙钟而不是减速钟，可能在拍表中途把目标拉回去。一次按键能把它按住几个真实秒；对较长的减速拍表，用 `--preload` 打桩该计时器并先等待静止（`verify.md`，"Slow mode doesn't slow timers"）。
- **要数字，不只是图。** `canvas.getContext("webgl")` 返回页面自己的上下文。在 `requestAnimationFrame` 回调里，用 `gl.getUniform(gl.getParameter(gl.CURRENT_PROGRAM), location)` 读 uniform，用 `gl.readPixels` 读 canvas 平均 RGB。回读只在与绘制同一任务内有效，因为 `preserveDrawingBuffer` 是关闭的：来自另一任务的全零帧是测试伪影。整个运行期间每帧记录一行。除刻意的爆发外，色相与平均亮度在阶段切换中各自都应单调移动。
- **边缘读数为 0。** 每个阶段都回读 canvas 最后 10 行与 10 列。任何大于 0 的值都是一个剪切，会在屏幕混合下显现。
- **浅色下记录墨色，而非亮度。** 每帧记录一个区域铺在白底上的墨量 `1 − lum(rgb + 1 − a)`，使用 `verify.md` 里的钩子（"Light-theme shaders"）。对每个关键区域（一个腔体、核心、仅回退区域）在首次亮起、每次阶段切换与淡出期间都做。墨色必须跟随物理量：光进来时上升、光离开时下降，没有鼓包也没有凹陷。用数字说，加力燃烧室的切口窗口从 1894 降到 155，其熄灯过程从 154 升到 1959 并稳定在 1914。
- **浅色下沿物理渐变表探测颜色。** 对会升温或降温的东西，在几个温度读中心与缘部像素。它们必须沿暗红 → 橙 → 黄 → 近白行进，缘部比中心冷，没有橙红或灰色阶段。玻璃炉口的表从 930 °C 时的中心 255/179/99、缘 252/114/25，经约 620 °C 时的 211/63/6 与 165/28/1，到 465 °C 变为透明玻璃。对体积，从边到边读一条墨色横截面：峰值应落在轴上。
- **浅色下也要检查暗色路径。** 每个浅色改动都在 `uLight` 之后，所以暗色构建的帧应与其基线一致。还要在浅色页面内渲染一块暗色底板，分别测有与无 WebGL（嵌套主题陷阱，见 8.5 节）。
- **效果与 SVG 相遇的每一处都拍 2× 与 4× 裁切图**：缘口、唇部、甲板边缘、壁带、纸角、遮挡体，以及透过其观看效果的每个开口（它是否在四边都止于缘口、下方露出凹槽壁？）。寻找接缝、硬边、色带、暗色领圈、双线或突兀的变色。浅色下，凡墨色与笔画相遇处（切面、缘口、口部、折痕）也要拍 8–16× 裁切图，寻找墨与线之间的浅色缝隙、涂到线上的墨、或被墨清掉的笔画。同一裁切要分别在隐藏前层 SVG 与隐藏 canvas 下对比（用 `eval` 设置 `style.display="none"`）。
- **回退。** 用 `--no-webgl` 运行 `drive.mjs`（每个 `getContext("webgl")` 都返回 `null`），检查 `data-gl` 不存在、且仅 SVG 在两种主题下仍能读懂。浅色下，回退应显示与着色器相同的墨色与相同的暗底色，且着色器画的每个物体都在。用 `--preload` 让页面从第一帧起减速，把首次亮起拍成连拍表：从回退到着色器的交接不得下陷（见 8.5 节）。
- **减少动态**（`--reduced`）：一帧静止且有代表性的画面。
- **控制台干净。** 着色器编译错误会以 `console.error` 的形式来自 `onError`。

## 10. 性能

- 成本约等于像素数 × 采样数。Raptor 每层 canvas 运行 28 个体积样本（每个含四个激波胞项），外加 28 步地面搜索（含 7 次二分）、8 样本光晕、10 样本溅射幕与每层 canvas 6 个蒸汽样本，设备比最高 1.5×。它能在 SwiftShader 下运行，但其帧时间尚未在真实 GPU 上测量。交付更重的东西之前，先在真实浏览器的 Performance 面板测量；如果太慢，先减样本、再降 `maxRatio`。
- 波纹水箱的纸着色器对每个像素求值双波源场四次（一次为折射偏移，三次为焦散的不动点反解），每个波源要花六次历史纹理读取，代替 24 步的 uniform 循环。在笔记本 GPU 上实测每次绘制约 1 ms；手机上的成本未测。
- 要知道自己测的是哪块 GPU。`drive.mjs` 与 `capture.mjs` 中的无头 Chrome 能用时用真实 GPU，否则用 SwiftShader；在采信一个帧时间之前，先记录 `gl.getParameter(gl.RENDERER)`。
- 采样只花在关键之处。给步进限定范围（圆柱或盒）、从地面命中处开始，并按 uniform 门控整个项。
- 两个 canvas 要花两次全屏绘制。把没有东西可显示的那个清空（Raptor 的 veil 层只在存在蒸汽、水汽或激波时才绘制）。
- 场里没有什么需要 `uTime`，就不要让它动。波纹水箱传 `draw(values, 0)`、只移动波的相位，因此减少动态的帧是真静止。
- 空闲时就把循环停掉。一个什么都没发生却保持在 60 fps 的着色器页面，就是一个 bug。
