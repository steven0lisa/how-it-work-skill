# 验证（Verify）

像一位挑剔的审稿人那样审视图形，赶在其他任何人之前。渲染很便宜，瞎猜很昂贵。

## 截图（Capture）
`scripts/capture.mjs` 会自行启动一个无头 Chrome（找不到时设置 `CHROME_PATH`），加载文件或 URL，默认裁剪到 `.iso-plate`。它还会打印读数（readout）、所有控制台（console）错误或异常，以及页面是否出现横向滚动。

```sh
node scripts/capture.mjs page.html --out shots/desk.png                         # desktop 1440, dpr 2
node scripts/capture.mjs page.html --out shots/phone.png --width 390            # phone
node scripts/capture.mjs page.html --out shots/hover.png --hover 0.7,0.5        # pointer at 70% across
node scripts/capture.mjs page.html --out shots/tour.png --times 1200,2400,3600  # motion: extra shots over time
node scripts/capture.mjs page.html --out shots/keys.png --key ArrowRight,ArrowRight
node scripts/capture.mjs page.html --out shots/joint.png --zoom 200,80,220,170  # close-up at 4x, in card px
node scripts/capture.mjs page.html --out shots/still.png --reduced              # reduced motion
node scripts/capture.mjs http://localhost:3000/docs --select "figure:nth-of-type(3) .iso-plate"   # one figure on a dev-server page
```
`--select` 接受任意 CSS 选择器；默认裁剪到第一个 `.iso-plate`。每个 `--times` 延迟都从第一张截图起算，所以 `1200,2400,3600` 大约在其后 1.2、2.4、3.6 秒拍摄（一次截图耗时 60–200 ms，比这更近的延迟会拍得偏晚），保存为 `name-<ms>.png`。`--key` 发送的键码与 `drive.mjs` 相同（`7` 发 `Digit7`，`.` 发 `Period`；逗号要写成 `Comma`，因为列表本身按逗号分隔）。其余选项有 `--height`（1000）、`--scale`（2）、`--wait`（首拍前 2500 ms）、`--zoom-scale`（4）和 `--theme dark|light`。生成的每一张 PNG 都要看：打开它，亲眼过目。

两个脚本都在开启 WebGL 的状态下启动 Chrome（`--enable-unsafe-swiftshader --ignore-gpu-blocklist`），所以着色器（shader）也会出现在 `capture.mjs` 的静帧里。凡是需要计时或放慢的，都用 `drive.mjs`。

## 驱动（Drive）
`scripts/drive.mjs` 在开启 WebGL 的无头 Chrome 里运行一段脚本化会话。用于序列操作：悬停、点击、键入、再等待某个阶段。也用于放慢的联络表（contact sheet，多格连拍）来捕捉跳变（pop），以及无 WebGL 时的回退（fallback）。每张截图后它打印读数，随后是控制台错误和异常。

```sh
node scripts/drive.mjs page.html '[["wait",1500],["move",".rt-stage",0.7,0.4],["wait",600],["shot","shots/hover.png"]]'
node scripts/drive.mjs page.html --actions steps.json --height 1300
node scripts/drive.mjs page.html --actions steps.json --width 390
node scripts/drive.mjs page-light.html --actions steps.json --reduced
node scripts/drive.mjs page.html --actions steps.json --no-webgl
node scripts/drive.mjs page-light.html --actions steps.json --preload slow.js
```
选项有 `--width`（1440）、`--height`（1100）、`--scale`（1）、`--reduced`、`--theme dark|light`、`--no-webgl`（让每个 `getContext("webgl")` 或 `"webgl2"` 都返回 `null`，用于检查 SVG 回退）、`--preload file.js`（在页面自身脚本之前运行的脚本：计数器、桩（stub）、钩子（hook），或 `window.__driveSpeed=0.04` 让页面从第一帧起就放慢）以及 `--actions file.json`。选择器默认取第一个 `.iso-plate`，分数坐标在元素范围内按 0..1 取值。

`--theme`（两个脚本都有）只模拟 `prefers-color-scheme`。Kit 页面在构建时就定下主题（`<body>` 上的 `data-theme`），不理会该媒体查询，所以要检查浅色主题，得用 `--light` 构建并驱动 `-light.html` 文件。

| 动作 | 作用 |
|---|---|
| `["scroll", sel]` | 把元素滚动到视口中部。 |
| `["wait", ms]` | 等待。 |
| `["move", sel, fx, fy]` | 分 10 步把鼠标滑到该处，沿途触发 `pointermove`。 |
| `["click", sel, fx?, fy?]` | 点击，默认在中心。 |
| `["drag", sel, fx0, fy0, fx1, fy1, steps?]` | 在一点按下，用 `steps`（20）次逐帧相隔的移动滑到另一点，然后松开。这些是真实输入事件（pointer id 1），所以调用 `setPointerCapture` 的处理器能正常工作。用它测试拖拽转动：从 `eval` 派发的 `PointerEvent` 没有活动指针，`setPointerCapture` 会抛 `NotFoundError` 并中止处理器。 |
| `["key", key, sel?, code?]` | 聚焦 `sel`（默认 `[tabindex="0"]`）并按一个键：`"i"`、`"Escape"`、`"ArrowRight"`、`" "`、`"7"`。数字发 `DigitN`，字母发 `KeyX`，标点发其正确的键码（`Space`、`Minus`、`Comma`…）。同名键要传 `code`：`["key", "Shift", null, "ShiftRight"]`、`["key", "Enter", null, "NumpadEnter"]`。 |
| `["slow", factor]` | 让页面的 `requestAnimationFrame` 时间戳和 `performance.now` 以 `factor` × 实时速度运行：`["slow", 0.05]` 是 20 倍慢动作，联络表能在多帧里捕到一个 100 ms 的事件。`["slow", 1]` 恢复实时。 |
| `["until", js, timeoutMs?]` | 每 25 ms 轮询一个 JS 表达式，直到其为真（默认 10 s），超时则打印一行提示。 |
| `["eval", js]` | 在页面里运行 JS 并打印结果。Promise 会被等待。 |
| `["shot", out, sel?, scale?, [x,y,w,h]?]` | 截取一个元素，或其中以 CSS px 表示的区域。 |
| `["sheet", out, every, count, sel?, scale?, [x,y,w,h]?, columns?]` | 每隔 `every` ms 拍 `count` 帧，并拼成一张带序号的 PNG。 |
| `["turn", value, group?]` | 3D 图形：设置 `window.__isoTurn`（数字设置第一个转动组（turn group），`group` 指定另一组，对象一次设多组）并等待两帧。 |
| `["orbit", out, from, to, step, sel?, scale?, [x,y,w,h]?, columns?]` | 3D 图形：跨角度拍摄第一个转动组的联络表，每帧带标注。`["orbit", "swap.png", 129.7, 130.2, 0.1, null, 6, [x, y, w, h]]` 是一次顺序变化（order change）的 6× 特写。 |
| `["lines", out?, sel?, { tolerance }?]` | 下文的线端检查（line check），在前面动作把图形留下的任意姿态下执行（分离态、行程中途）。`out` 会输出一张图，把每个悬空端都做圈痕标注（ring）。检查失败会让运行以退出码 1 结束。 |

陷阱：
- **`scroll` 之后的截图是正确的**，所以可以滚动到页面中更低处的图形。不过让视口高到装下整张卡片（`--height 1300`）仍是最简单的做法：滚动位置为 0 时图形即可见，其 `IntersectionObserver` 会将它启动。
- **联络表的帧间距是 `every` 加上截图耗时**（60–200 ms，重的 WebGL 页面更久，且截图还会卡住动画帧）。比这更快的过渡，在联络表前放 `["slow", 0.05]`、之后放 `["slow", 1]`。`wait`、`every` 和页面的 `setTimeout` 定时器仍走实时，所以在 0.05 速率下，每 100 ms 一帧的联络表在页面时间里只隔 5 ms，而空闲导览的恢复定时器仍按墙上时钟触发：一次按键只能把它推迟约 3.6 s 的真实时间，所以每次放慢的联络表前要再按一个无害的键。加载时就启动的事件需要页面从第一帧起就放慢：`--preload` 一个含 `window.__driveSpeed=0.04` 的文件。
- **慢速模式不会放慢定时器。**`setTimeout` 走墙上时钟，所以在一次放慢的长截图期间，空闲导览的恢复定时器（Raptor 的 3.6 s `quietTour`）可能重启导览，在联络表中途把引擎合上。每次放慢的联络表之前，先停掉导览并等待静止：预载一个针对该定时器的桩（stub）（`const st = window.setTimeout; window.setTimeout = (f, d, ...a) => (d === 3600 ? 0 : st(f, d, ...a));`），按 `Escape`，`["until", "document.querySelector('[data-readout]').textContent.includes('cold')"]`，然后再按启动过渡的键。标注为"opening"（打开）却从已经打开的导览开始的联络表，拍到的是关闭过程。
- **等待阶段，不要靠猜。**`["until", "document.querySelector('[data-readout]').textContent.includes('main stage')", 8000]` 会在读数一到达该阶段就返回。
- **`null` 选择器就是默认值。**在每个动作里，缺失或 `null` 选择器都表示第一个 `.iso-plate`（对 `key` 来说是可聚焦的 `[tabindex="0"]`），所以 `["sheet", "s.png", 200, 16, null, 0.5]` 拍的就是整张卡片。
- **长的 eval 字符串会破坏内联 JSON 的引号。**凡含 `eval` 或 `until` 的内容都放进 `--actions steps.json`。
- **页面把活动代码以内联 `<script type="module">` 形式嵌入**，所以顶层变量对 `eval` 不可见。改从 DOM 读取状态：读数文本、`aria-valuetext`、`data-*` 属性。要数帧或逐帧读值，用 `--preload` 装一个钩子（它在页面之前运行，所以能包裹 `requestAnimationFrame` 或 `HTMLCanvasElement.prototype.getContext`）。

## 线端（Line ends）
审计看到的是记录下来的形状，从不看线条标记，所以它看不见停在某个面中间的接缝、刻度线、加强筋或刻度。`scripts/lines.mjs` 检查的是图纸本身，在无头 Chrome 里运行：

```sh
node scripts/lines.mjs page.html --out shots/lines.png
node scripts/lines.mjs page-light.html --out shots/lines-light.png --reduced
node scripts/drive.mjs page.html '[["key"," "],["wait",2000],["lines","shots/lines-apart.png"]]'
```
```
line ends: 3 of 1148 ends on 295 lines float
  iso-line faint in it k -0.080  end  7.25 px from anything   at card 507.4,407.2   length 75.7   d M392.1 381.7L507.4 407.2…
```
卡片里每条 `.iso-line` 的每条开放子路径（接缝、凹槽、来自 `fadedSvg` 的加强筋、刻度线、刻度、表针、指针、导线）都必须终结在另一笔画、一个点或一条轮廓线（silhouette）的 1 viewBox px 之内，或被之后绘制的填充盖住。短直标记（7 px 以内：刻度）需要一端在 2 px 内扎根；两端都悬空即失败。Kit 自身的轮廓、折痕（crease）、倒角（bevel）和管件边缘不检查，被 `fadeLineSvg` 渐变淡出的端点也不检查。`lineSvg(d, { free: true })`（或 `"start"`、`"end"`）标记本来就该停住的端点：代表印刷文字的刻度线、表针的尾部、指针的尖端。除此之外不得滥用。它以卡片 px 打印每个悬空端，可直接用于 `capture.mjs --zoom`，并附其 `d` 的开头以便在构建中定位；`--out` 会在卡片的 2× 截图上做圈痕标注。两个主题都要跑，图形的每个静止姿态都要跑，修到打印 0 为止：

| 现象 | 修法 |
|---|---|
| 盒子上的接缝或焊缝线缩离了角落 | 用 `k.sideSeam(plan, z, P, steps)` 作为厚板的 `inner`：它会包过角落并终结在轮廓线上（`craft.md` 第 3 节）。 |
| 一个面上的加强筋或面板线，两端都缩进 | 从边画到边：从该面的底部画到顶部，或从端画到端。 |
| 加强筋没到自己的接缝就停了 | `ribsOf(…, { seams: true })`，或在带的两端用 `s0`、`s1`。 |
| 刻度悬空在光板上，或垂在边缘之下 | 把它们放到一条标尺条上，扎根在其边缘（面上的顶边用 `topTicks` 的 `edge`、`sideTicks` 的 `top`）。 |
| 通风口或格栅线 | 改成闭合槽：细的圆角轮廓，不是线。 |
| 一条线从零件底下穿过、刚过零件就停 | 在零件之前画，画足全长，让零件把它遮住。 |
| 标牌上的刻度线 | `free: true`。它们代表文字。 |

## 审计（Audit）
截图只能抓到你想到要看的东西。带管道、线缆和圆形零件的图形还需要检查几何：穿过导管的一条线缆、或画在前面物体之上的一个零件，1× 下看不见，4× 下一目了然。把 `kit/audit.mjs` 接入构建（`kit.md` 第 12 节）：各 helper 收进一个 `A.recorder(P, { order, spring })`，构建里调用 `A.settle(R, P)` 并以 `A.auditOrExit(R, P)` 结尾，然后运行：

```sh
node examples/raptor-engine/build.mjs --audit
```
```
clearance: 0 problems
clearance apart: 0 problems
terminals: 0 ends not on a mount
loose ends: 0
screen folds: 0
support: 0 unsupported solids · 0 hanging tubes · 0 sunk pairs
depth order: 0 problems
through in mid-flight: 0 pairs
self-overlap: 0 chunks whose outline crosses itself on screen
wobble: 0 smoothed profiles whose slope turns between its ends
saddles off their host apart: 0
stand-in shapes: 0 items whose recorded shape covers too little of what it draws
crowding: 0 pairs where a bend or end touches another line on screen
free ends apart (marked free, not failing): 35
audit · 80 tubes · 267 solids · 478 items · 1119 depth states · 1707702 overlaps · 4292 order rules · 151.9 s
audit passed
```
它打印 `audit passed` 或 `audit failed: n problems`，任何失败都以非零码退出。每一行都算数，包括飞行中途（mid-flight）发生的事和 `items with no shape`，只有两个供人阅读的清单除外：`in passing` 和 `free ends apart`。深度顺序和穿透（管件对所有东西、实体对实体）都在同一批回放帧上检查：弹簧打开再关闭直到静止的每一帧，每个错峰（stagger）边界两侧 8 ms 内反转的每一帧（`reversalsOf`：Raptor 35 ms 错峰下的 0.027、0.043、0.062、0.078 … 0.393 s），以及前缀状态（`prefixStates`）。深度检查用这些帧中每四帧取一。每一行怎么读、怎么修：

| 行 | 含义 | 修法 |
|---|---|---|
| `through A × B  clearance -2.4` | 管件 A 在该弧长区间与 B 相交。 | 让 A 绕开 B 重排路线，留出约 1.5 个半径的余量，或移动 B。只有 A 端缘所坐落的形状可获豁免，且仅限距该端约一个半径之内。 |
| `close A × B  clearance 0.2 < 0.6` | A 经过 B 时余量不足其半径的一半。 | 微调路线；加宽线束通道间距。 |
| `clearance apart: through share 1 …` | 分离态下有一根管件穿过了别的东西。 | 改变该组件的分离（explode）向量。 |
| `terminals: A end 1.40 from B (rim -1.1…1.3)` | A 的端点、或其端部法兰的边缘，没有坐落在安装座上：悬空、倾斜，或法兰比下面的凸台（boss）宽。 | 让路线恰好沿表面法线终结在表面上；凸台至少做到与法兰同宽；曲面体上把法兰坐在鞍形垫（saddle pad）上。线缆绝不终结在夹子上。 |
| `loose ends: share 1 A end touches nothing that moves with it` | 分离时端点悬在空中：没有任何属于其他所有者、随之移动的东西（自身的法兰或壳体不算）。 | 给它一个随之移动的安装座；若它本就设计为可脱开（带法兰的风管、装在另一组件上的接头），则标记 `free`，让它作为一项选择列入 "free ends"（自由端）。 |
| `self-overlap: A:3 its outline folds back on screen` | 单个分块的轮廓在屏幕上自交：线条在小于自身半宽的屏幕半径上转弯（一条指向观看者的腿在端口附近转弯）。 | 让线路穿过肘部球（elbow ball，Raptor 的 `elbow`）离开端口，给转弯留更长的直腿，或在与观看者相对的平面内转弯（Raptor 的 `sideOn`，`craft.md` 第 8 节）。 |
| `wobble: A dr/ds turns at s = …` | 由手打节点平滑出的轮廓，其斜率出现平台并反转，导致明暗色带呈阶梯状。 | 改用解析式 `r(s)` 并传入 `smooth: { slope }`。 |
| `saddles off their host apart: A (piece) is cut to B (host piece)` | 一块鞍形垫随另一组件飞离，露出了切削的底面。 | 把垫块划归宿主的组件；移动的管件离开其平贴面。 |
| `solid  state  A × B  sunk 1.81 deep` | 两个不同组件的实体在该帧互相穿插。 | 与管件同理：修顺序或向量，或在悬垂下方拉开间隙。 |
| `screen folds: A turns 150° on screen for 78° in 3D` | 路线在屏幕上折返，读起来像个尖点。 | 让转弯之前的那段腿在屏幕上侧向走线。 |
| `screen folds: A turns 110° on screen round a 0.4 px bend` | 一个可见的弯收窄成一点：其圆角被侧棱对着观看，两条腿以 V 形相交。 | 在管件内部转弯（球面三通、线以 45° 离开），在与观看者相对的平面内转弯，或挪动端口让法线横对屏幕。 |
| `through in mid-flight: apart 0.233 s A × B clearance -1.6` | 在那个回放帧里 A 穿过了 B。 | 修顺序或向量（`motion.md` "Order the stagger"）：外侧零件先离开；指向另一组件表面的向量需要那个组件更早离开；旋接在一起的零件共享一根弹簧。 |
| `crowding: A bend × B outlines overlap 0.05 px` | A 的弯或端落在 B 的轮廓上，读起来像个接头或一条穿进 B 的线。 | 重排路线，让交汇成为两条直线段的清晰交叉，或留出可见间隙。豁免某一对（`allow`）之前必须先用 8× 看过。 |
| `crowding: A end × B (rod)` | A 的端或弯落在细长实体的轮廓上：拉杆、活塞杆、立柱、导轨或把手（长超过其半径或半宽 4 倍的任何实体）。读起来像那根杆插进了 A 的接头里。 | 从横跨杆件的方向接近端口，或转动弯头让最后一段腿横躺在屏幕上。 |
| `stand-in shapes: A its shapes cover 4% of its drawing` | 为 A 记录的形状覆盖的屏幕范围远小于 A 画出的范围，审计只检查到一条细缝，对其余部分不置一词。 | 记录你所画内容的真实形状（滑出密封套（gland）的杆用 `A.slide`）。只有当某项的绘制确实伸出到其本体之外、且亲眼看过之后，才豁免它（`cover: [name]`）。 |
| `crowding: C fitting × B` | 一个夹子落在另一条线的 2 px 之内，读起来像那条线从它身上长出来。 | 改分离向量或夹子的角度。 |
| `support: loose A` / `hanging A` / `sunk A × B 1.2 deep` | 一个实体或刚性管件没有接触任何通向底座的东西，或埋进了 B 里。 | 添加固定它的支架、螺柱或底板；沉入的零件修好接触（边缘落在表面上、被宿主切削的垫块标记 `seated`；`seated` 只豁免其 `cut` 中的宿主）。 |
| `order  A is in front of B but drawn first` | 凡重叠处 A 的表面都更近，但 B 画在了它上面，静止态或飞行中途的帧里都可能出现。 | `orderKeys` 通常能修好；剩下的一例说明某条规则在循环中被丢掉，去那个循环里找互锁的一对。 |
| `cycle  A … (n the other way)` | A 和 B 在不同位置互为在前：它们在屏幕上互相穿插。 | 改几何：管道在箍环处留隙，箍带在圈处断开，杆终结在球上，蜗壳取相切，阀放到直段，垫块避开宿主的轮廓。 |
| `in passing` | 互锁、或无法用任何 `k + lift · share` 排序，且只出现在均匀态（所有组件处于同一 share——页面从不显示这种状态）。 | 不算失败。同样的事出现在回放帧里就是 `depth order` 问题。 |
| `items with no shape` | 某物被绘制但没有记录形状，没有任何东西检查它。 | 在绘制它的 helper 里补记。 |

零问题的审计是必要条件而非充分条件：它看不见停住的接缝、阶梯色带或纹理，而且只检查你记录过的东西。让审计沉默的替身形状（stand-in shape）本身就是 bug，这正是审计要为它报一条失败的原因。先跑线端检查，然后亲眼看。

## 环绕（Orbit，3D 图形）
带转动组（`references/3d.md`）的图形还要加做环绕审计和浏览器检查。浅色页和深色页都要跑。
```sh
node build.mjs --light --audit > audit.txt 2>&1 &   # the full run, in the background: ends "turn audit passed"
node build.mjs --light --audit --quick    # ≈ 10× fewer states while iterating: ends "turn audit quick"
node build.mjs --light --audit --quick --only=order,continuity   # a subset: the rest print "skipped", ends "turn audit partial"
node build.mjs --light --verify           # a page with window.__isoVerify (recorded shapes) for --order
node ../../scripts/turn-check.mjs build.mjs --fidelity
node ../../scripts/turn-check.mjs page-verify-light.html --order --step 1
node ../../scripts/turn-check.mjs page-verify-light.html --order --states clip.json     # hinges and clips: --step turns the first turn group only
node ../../scripts/turn-check.mjs page-light.html --pops --no-webgl --states tilt-path.json   # one hinge as a continuous path at 0.02°
node ../../scripts/turn-check.mjs page-light.html --pops --no-webgl --step 1 --out shots/pops
node ../../scripts/turn-check.mjs page-light.html --lines
node ../../scripts/turn-check.mjs page-light.html --perf --width 1440
node ../../scripts/turn-check.mjs page-light.html --perf --width 390 --throttle 4
node ../../scripts/drive.mjs page-light.html '[["orbit", "shots/orbit.png", 0, 345, 15]]'
```
**耗时。**一个 28 零件、一个转动、一个铰链、一个卡扣的图形，quick 模式 85 s，完整模式 17 分钟，超过了 shell 的 10 分钟超时：完整审计放到后台跑。quick 运行采样不足（`continuity` 只查 100 个折半候选而非 800 个，顺序扫描只取十分之一的状态），所以只能作参考，只有完整运行打出 `turn audit passed` 才算数。带 `--only` 的运行以 `turn audit partial` 结束，永远不会是 `passed`。

**状态。**审计以 0.3°（顺序）和 0.05°（连续性）扫掠每个转动组，外加 2000 个随机状态和每个顺序变化事件 ± 0.001° 的邻域。动作片段（`clips: [{ name, at(t), from, to, frame }]`）逐帧走查：一个手势、一次抓取、一次拆解。自由组和铰链组只在你给出的状态上验证，所以每个片段都要给，并在若干转动角度下给。

**检查行。**`separation`、`unseparated`、`cycles`、`forced`、`swaps`、`layers`、`rigid`、`swept`、`posed`、`folds`、`self`、`crowding`、`endon`、`coverage`、`depth`、`continuity`。每条检查行的失败样貌与修法见 `3d.md` 第 13 节。每条都有一个植入的故障会让它失败：一块沉入的垫（`separation`）、三根倾斜细棒组成的风车（`cycles`）、在竖直平面内弯折的纤维（`folds`）、30° 偏离的管件（`endon`）、中途停住的加强筋排（`continuity`）。

**浏览器检查**（每项先跑对照（control）；对照没有失败说明检查本身坏了）：
- `--order`：用 WebGL z-buffer 里的记录形状对照 DOM 的填充：在任何角度和片段状态下，重叠区域内部像素 ≥4 的活对为 0。
- `--pops`：整圈逐度做一阶差分（θ → θ + 0.02°）、以 0.1° 做二阶差分：存活区域为 0、6 px 及以上的核为 0。把它列出的 1–5 px 核逐一看过。
- `--lines`：24 个角度上悬空端为 0。
- `--perf`：`3d.md` 第 15 节的预算，1440 宽度下，以及 390 宽度加 4× CPU 节流下。

**环绕联络表。**先 0–345° 每 15° 一张，再以 0.1° 步进扫过三个最大的顺序变化（审计把它们列在 "order changes" 下），然后是每个放慢的片段（`["slow", 0.05]`）。看的时候留意蜘蛛角（某个侧面看过去像几条腿的手）、消失在别的零件背后的零件，以及终结在另一零件轮廓上的线条。

### 3D 图形核对清单
- [ ] 用完整状态集（不用 `--quick`，不用 `--only`）、包含片段，跑出 `turn audit passed`。
- [ ] 每个铰链和卡扣都在 `--states` 上跑 `--order`；每个铰链都在 0.02° 路径上跑 `--pops`（`3d.md` 第 14 节）。
- [ ] `turn-check` 的 order、pops、lines 在 0 度、fidelity 通过，每个对照都按设计失败。
- [ ] 15° 间隔的环绕联络表已读过；每个关节、交叉与纹理末端都在 8 个角度出 8× 裁切，浅色深色各一遍。
- [ ] 浅色页面上材质读起来是金属：高光带随转动移动，明暗交叉淡变处无跳变。
- [ ] 每个锚定于自身组的着色器都以 0.1° 步进跟随转动；世界系特效不转；墨止于遮挡笔画（在遮挡掩模与零件相接处出 8× 裁切）。
- [ ] 预算在 1440 与 390 × 4 下实测；图形过密需要时可用半帧率杠杆。

## 特写（Close-ups）
图形的主人是在浅色主题的一张特写上发现 Raptor 缺陷的：盖在主导管上的线缆、呈阶梯状的钟形件、戛然而止的螺栓点排。赶在任何人之前，自己把每个地方的特写都拍一遍。

1. **构建两个主题**（`node build.mjs` 和 `node build.mjs --light`），列出所有接合处：每个法兰、三通、弯头、球关节、夹子、阀、波纹管、箍带、鞍形垫和交叉处，每个曲面体（钟形件、喉部、穹顶、锥、阀体凸鼓、球），用 `bandsOf` 切出的每个带状关节，以及纹理终结的每一处。平件也不例外：盒面上每条接缝、焊缝、加强筋和修饰线（够到两个角落了吗？）、板上每排刻度或标记（在标尺条上吗？）、沿轴线的每排加强筋的两端、每根带纹理的软管在分块关节处和其套节（ferrule）处、每根滑动的杆（静止、行程中途、满行程），以及每个含文字或着色器内容的窗口或显示屏。
2. **两个主题、装配态下以 4× 拍下每个区域。**`--zoom x,y,w,h` 取卡片 px 的一个区域并按 `--zoom-scale`（4）放大：
   ```sh
   node scripts/capture.mjs page.html --out shots/joint-dark.png --zoom 200,80,220,170
   node scripts/capture.mjs page-light.html --out shots/joint-light.png --zoom 200,80,220,170
   ```
3. **分离态也要拍**，如果图形可拆解。驱动到分离位，等它静止，再以 4 倍拍一个区域。Raptor 用空格键切换分离，且读数只有在每根弹簧都到位后才开始 `apart ·`，所以 `apart.json` 是
   ```json
   [["key", " "], ["until", "document.querySelector('[data-readout]').textContent.startsWith('apart')"], ["wait", 400],
    ["shot", "shots/apart-joint.png", null, 4, [200, 80, 220, 170]]]
   ```
   用 `node scripts/drive.mjs page.html --actions apart.json` 运行，再对 `page-light.html` 跑一遍。
4. **按这个顺序在每张 PNG 里排查这些缺陷：**穿过或扎进别物的线；交叉处较近的线被画在了后面；坐在另一条线轮廓上的弯、端或夹；法兰悬离端口或沉入其中；曲面上的阶梯色块、扇贝边或淡色楔形；终结在直线切口上的箍带；停在面中间的一排加强筋、圆点或刻度线；分块关节处横过管件高光或阴影的发丝线；在轮廓上切出缺口的接缝；没到该到达边缘就停住的接缝、加强筋或筋条；顶面上碰不到任何东西的短标记；侧边硬直的软管纹理，或在分块关节处重新开始或被切断的纹理；移动中轮廓断裂的杆；越过窗口的文字；溢出开口边缘的着色器内容。与上一轮同一区域的截图对比。
5. **着色器：**同样的区域在特效进行中拍摄，外加用 `["slow", 0.05]` 放慢的联络表（`webgl.md` 第 9 节）。

**细节遍历。**最后，以 4× 按裁切网格扫一遍（Figure 级看主体和机构，Hero 或 Epic 级看整张图；`SKILL.md` 工作流第 1 步），找出任何没有真实细节的面、角落或底座区段。每个图形都要高度精细（`SKILL.md` "Every creation is highly detailed"）：补上属于那里的真实零件（螺丝、接缝、倒角、标牌、刻度、固定用的支架），绝不加不属于任何零件的装饰。

## 浅色主题着色器
浅色是发布主题，而在纸上，辉光就是墨。迄今每一次浅色转换都是先败在单帧截图显示不出的东西上：光落下墨却升起来、辉光出现前的淡色一闪、橙红色阶段、笔画旁边的彩色镶边。所以要一边看一边量墨（`webgl.md` 第 8 节）。

**每帧墨量。**把下面这段存为 `ink.js`，用 `--preload` 传入。页面画的每一帧，它都记录一个画布区域铺在白色上的平均墨量，`255 − lum(rgb + 255 − a)`，与绘制在同一任务中回读：
```js
const get = HTMLCanvasElement.prototype.getContext;
HTMLCanvasElement.prototype.getContext = function (kind, ...rest) {
  const gl = get.call(this, kind, ...rest);
  if (gl && String(kind).startsWith("webgl") && !this.__gl) {
    this.__gl = gl;
    for (const name of ["drawArrays", "clear"]) {
      const call = gl[name].bind(gl);
      gl[name] = (...args) => { gl.__drew = true; return call(...args); };
    }
  }
  return gl;
};
const raf = window.requestAnimationFrame.bind(window);
window.__ink = [];
window.requestAnimationFrame = (callback) => raf((now) => {
  callback(now);
  const canvas = document.querySelector(window.__inkCanvas || "canvas");
  const gl = canvas && canvas.__gl;
  if (!gl || !gl.__drew) return;
  gl.__drew = false;
  const [x, y, w, h] = window.__inkBox || [0, 0, canvas.width, canvas.height];
  const px = new Uint8Array(w * h * 4);
  gl.readPixels(x, y, w, h, gl.RGBA, gl.UNSIGNED_BYTE, px);
  let ink = 0;
  for (let i = 0; i < px.length; i += 4) {
    const bare = 255 - px[i + 3];
    ink += 255 - (0.2126 * (px[i] + bare) + 0.7152 * (px[i + 1] + bare) + 0.0722 * (px[i + 2] + bare));
  }
  window.__ink.push(Math.round((ink / (w * h)) * 100) / 100);
});
```
- **选好测量对象。**过渡开始前，用 `eval` 设置 `window.__inkCanvas`（一个选择器）和 `window.__inkBox`（画布设备 px 的 `[x, y, w, h]`，从其左下角起量；CSS px 按 `canvas.width / canvas.clientWidth` 换算）。之后用 `["eval", "window.__ink.join(' ')"]` 读取各行数值。
- **每个重要区域各做一次：**一个凹腔、核心、只有回退才覆盖的区域。放慢运行，覆盖初次亮起（`--preload` 再传一个含 `window.__driveSpeed=0.04` 的文件）、每次相位变化和淡出。
- **要看什么。**墨量随物理量单调变化：光进来就升，光退去就降。交接处没有隆起，初次亮起时没有下凹。
- **它抓到过什么。**修复之前，加力燃烧室的凹腔在剖切面上墨量峰值达 3600；修复后从 1894 单调降到 155。那些行数值用的是该图形自己的单位——对其窗口区域的求和，而非本钩子的均值。波纹槽的浅色页上，钩子读数静止时为 14.2，随灯亮起而下降，因为那一页的光是照亮纸面的。

**沿温度坡的配色。**对任何会升温或降温的东西，用 `eval` 在几个温度点（或放慢运行中相当于数个温度段的时长）读取中心和边缘的像素值并记录下来。顺序必须是暗红 → 橙 → 黄 → 近白，边缘比中心冷，途中不得出现橙红（salmon）或灰色阶段。

**横截面。**读取穿过火焰、射流或流束的一行墨量。它应在轴线上达到峰值，向半透明边缘递减。边缘偏重的剖面是一团空心的、只剩轮廓的火焰。

**8–16× 下的笔画。**浅色主题下，把墨色与 SVG 笔画相接的每一处（切面、边缘、管口、折痕、有物体进入的画布边缘）都以 8–16× 裁切。找墨与线之间的淡色缝隙、盖在线上的墨、或被墨清掉的笔画。

**两条路径，以及嵌套。**深色构建应与其基线一致，因为每处浅色改动都以 `uLight` 为闸。渲染一张浅色页面上的深色图版和一张独立的浅色图版，各自分有 WebGL 和无 WebGL 两版：一条 `:not([data-theme="dark"])` 规则可以穿过页面命中深色图版。

## 验收清单（review checklist）
**它是一件实物**
- [ ] 一眼看上去是一件真实、做工考究的实物，而且你能叫出它的名字。
- [ ] 它的机构与概念对应：什么在动、那意味着什么，一句话能说清。
- [ ] 它不是一摞抽象的板子，也不是一块光秃的厚板。

**零件与体量**
- [ ] 行内图形 30–60 个实体；零件确实更多的主角题材可以远超此数（Raptor 约 270 个）。每个零件都倚靠别的零件，且各有职责。
- [ ] 侧面着深色、顶面更亮；主厚板带倒角；底座下和抬起的零件下有光晕投影（halo）。
- [ ] 小细节归属零件：板角的螺丝、凹槽、标尺、滚花、标牌。

**线条**
- [ ] 4× 下，每条线都终结在一个面、另一条线或一个端点上。没有任何东西停在空中。
- [ ] 没有引出线、气泡标注、尺寸箭头或漂浮标签；至多一块刻印的标牌。
- [ ] 画序处处正确：后面的东西不透出来，弹簧与机架都已切分。
- [ ] **审计清零。**图形里有管道、线缆或圆件时，`node build.mjs --audit` 以 `audit passed` 结束：每个失败项都为零，装配态、分离态以及回放拆解的每一帧（打开、关闭、反转）都如此。没有用顺序键偏置掩盖互锁，每一对被豁免的（`allow`、`crowd`）都用 8× 看过。
- [ ] 没有管道或线缆穿过另一根管道、线缆或实体，每处交叉都按深度顺序绘制：较近的线在上、交叉落在直段上、深度上分得足够开。
- [ ] 凡能沿结构旁边走的线，都不横穿主体表面；没有弯在屏幕上收成 V 形；无论装配还是分离，没有弯、端或夹落在另一条线的轮廓上。
- [ ] 每个法兰都坐在自己的端口上（端口弯曲时坐在鞍形垫上）；每个箍环、夹子、箍带、波纹管和阀都与所包裹的管道留隙；每个阀和波纹管都坐在直段上；没有路线在屏幕上折返；每个端点——分离态也一样——都终结在法兰、联轴器、接头或端点上。
- [ ] **接合处特写。**每个接合区域都在两个主题、装配与分离态下以 4× 拍摄（见"特写"），每张 PNG 都读过。
- [ ] **无阶梯。**每个曲面在 4× 下都是平滑着色：没有阶梯色块、没有扇贝状台阶、没有淡色楔形、没有终结在直线切口上的箍带、带状关节处没有扭折；收腰的形体沿其真实轮廓；接缝与轮廓相接处无缺口。
- [ ] **线端清零。**`node scripts/lines.mjs` 在两个主题和每个静止姿态下都打印 `line ends: 0`；每个 `free` 端都是代表文字的刻度线、表针尾或指针尖。
- [ ] **形状真实。**没有替身形状：记录的就是所画的，包括滑动的杆。
- [ ] **无突兀的纹理断头。**每排加强筋、螺栓点、刻度线或凹槽都终结在真实边缘上或渐变淡出；没有一个在满强度下停在面中间，平面也不例外。盒子的接缝包过角落，加强筋排终结在自己的接缝上，刻度坐在标尺条上，软管上的纹理沿其横截面走并贯穿每个分块关节。管道条纹延伸到端盖上，任何分块关节处都没有发丝线，强弯曲的锥形管（蜗壳）也不例外。

**明暗与色彩**
- [ ] 底座用 `lo`、结构用 `mid`、主体用 `hi`，恰好一处 `lit` 提亮。
- [ ] 单色，除非颜色本身就是主题。

**卡片与文字**
- [ ] `Fig n` / 标题 / 提示 / 实时读数位于四角，图例键（legend keys）与说明文字（caption）在卡片下方。
- [ ] 每个数字都真实且来自模型。读数随物体移动而变化，其文字与数字出自同一瞬间：来自模型的阶段词配上滞后仪表的数字，会在阀开启之前就显示 `relief 85 bar`。
- [ ] 机器显示的文字在最宽字符串下也放得进窗口，且被裁剪在窗口之内。

**动效与可访问性**
- [ ] 缓动、从容、无突跳；有空闲导览（idle tour）；交互时暂停。
- [ ] 移出屏幕即停止；`--reduced` 下保持静止；无控制台错误（包括 React key 警告）。
- [ ] 键盘可用、焦点环可见、`aria-valuetext` 用文字读出状态。

**着色器（若图形带有；`references/webgl.md`）**
- [ ] 每个过渡（启动、每次相位变化、稳定运行、停止）都拍联络表，用 `["slow", 0.05]` 放慢（加载即启动的特效改用 `--preload` 传 `window.__driveSpeed=0.04`），让每次交叉淡变的每一帧都可见：无突然变色、无单帧闪烁、无内容被裁掉、淡出期间亮度绝不回升。
- [ ] 特效与 SVG 相接处以 2× 和 4× 裁切：无接缝、硬边、色带（banding）、暗色箍圈或双线；同一平面上的东西与该平面的线条对位。
- [ ] 画布最外侧行列在每个相位下 alpha 均为 0。
- [ ] 无 WebGL（`--no-webgl`）时仅凭 SVG 也能读懂；减弱动效下呈现一帧有代表性的静帧；空闲和移出屏幕时循环停止。
- [ ] **浅色主题**（`webgl.md` 第 8 节；上文"浅色主题着色器"）：
  - 辉光靠饱和度来表现：饱和的主体、淡色的核心、彩色的边缘，绝不用深色或棕色墨。
  - 光照到的东西带染色，容纳光的凹腔是更深的实体底色。
  - 炽热物体按黑体顺序变化，无橙红阶段。
  - 烟与蒸汽是浅灰色。
  - 空中没有光晕。
  - 每帧墨量随光单调变化，贯穿初次亮起、每次交接和淡出。
  - 8–16× 下墨止于每条笔画的内缘。
  - 回退呈现同样的墨色与底色，交接到着色器时无下陷。
  - 深色保持不变，浅色页面上的深色图版依然正常。

**构图**
- [ ] 居中，约占卡片 55–65%；不触碰角落标签。
- [ ] 390 px 宽度下无横向滚动仍可读，选中零件时、燃烧进行中也如此，不只静止态：最长的读数不得钻到控件底下。

## 常见缺陷与修法（common faults）
| 现象 | 修法 |
|---|---|
| 一个零件透过它前面的东西显形 | 它画得太晚。把它挪到更早画，或把前面的零件切分。 |
| 立柱与面相接处横着一道椭圆 | 在延续的圆柱上画整只 `ring` 会露出其后半圈。改用 `sideArc`（只画前半圈），或让圆柱自身的轮廓来构成这条边。 |
| 立柱像是贴在它穿过的方块上 | 分两段画，一段在下、一段在上，方块画在两段之间。 |
| 折痕或倒角没到角落就停了 | 在 `slabOf`/`cylinder` 上用 `bevel`。它本来就终结在轮廓线上。不要手画边线。 |
| 侧面扁平呆死 | 你传了 `flat`，或实体高度为 0。给它厚度。 |
| 太稀疏 | 加结构，不是加纹理：支脚、带标尺与限位块的导轨、一块标牌、螺丝、旋钮、第二级机构。 |
| 太繁杂 | 把类纹理的重复和色调凹槽降为 `faint`；保留一处 `lit` 提亮。 |
| 图形在卡片里显得极小 | 用更小的 `pad` 适配，或收紧传给 `fitProjection` 的点。 |
| 标签页重新聚焦时画面跳变 | 把 `dt` 钳制到 1/30 s。 |
| key 警告只在大页面上出现 | 见 `react.md`：给元素 props 的根加 key。 |
| 着色器颜色在相位变化处跳变 | 给颜色独立的慢包络，由物理量驱动，而不是由相位标志驱动。见 `webgl.md` 第 7 节。 |
| 辉光里有直线边缘，常在画布边界 | 某一项在边界处不为零。在每个 `if`、边界与画布边缘之前都把它窗化到零。 |
| 辉光穿过地板或越过立柱 | 它在屏幕空间里。用射线与地面交点在世界空间重建，并把简单零件作为遮挡体交给着色器。 |
| 发光边缘之下有一圈暗色箍圈 | 该边缘以普通混合叠在着色器上。把它挪到带 `mix-blend-mode: screen` 的覆盖 SVG 里。 |
| 柔和辉光里有平直色带 | 在 `tonemap` 之后加门控三角抖动（dither）。 |
| 浅色下辉光读起来像深红或棕色烟雾 | 那是把深色再压暗、拿 alpha 充当亮度。要从纸面坡道解出墨色：饱和的主体、淡色的核心和一个 alpha 下限（`webgl.md` 第 8.1 节）。 |
| 浅色下初亮先变淡再变暗，或淡出时发灰、变重 | 逐项淡变、只淡 alpha、或色相在固定 alpha 下变红。对成墨只淡一次，能量与 alpha 一起调暗，色相在亮度稳定处取（`webgl.md` 第 8.2 节）。 |
| 浅色下边缘有淡色镶边或白色接缝 | 墨没到笔画就停住，或清掉了一道没人描的折痕。让墨止于笔画内缘（0.3 vb），折痕用墨来画（`webgl.md` 第 8.6 节）。 |
| 深色图版的回退在浅色页面上消失 | 一条 `.iso:not([data-theme="dark"])` 规则穿页命中了它。把深色规则放到最后，或以图版自身主题作键（`webgl.md` 第 8.5 节）。 |
| 管道或线缆看着像贴上去的、或从零件中间穿过 | 它的各段在同一层里。把每个 `tubePieces` 分块按 `mid` 深度排序，穿绕处用更短的段。若它真的穿过零件，就重排路线；`--audit` 会列出每一例。 |
| 线缆或信号线扇形铺过主体表面、越过主导管 | 它们是点对点直接布的线。沿结构重布：贴着管道上行、以夹持通道绕过机体、绕到侧面接远处端口（`craft.md` "Choosing waypoints"）。 |
| 修好一处交叉的顺序，另一处或分离姿态就坏了 | 键是手工调的。记录形状，让 `A.settle` 来定键，再修它报告的几何互锁。 |
| 箍环、箍带或夹子挡住管道近侧，或管道盖住了箍环的面 | 管道作为一个整块从中穿过。在箍环处给管道留隙（`gaps`），让一段管各终结于一个面。 |
| 体上的圈露出了盖在体上的后半圈 | 体在圈的位置是一条整带。在圈处把它切开。 |
| 阀或波纹管在两端切进了管道 | 它坐在弯头上。挪到一段直管上，长度至少为它自身长度加上两端圆角切量。 |
| 钟形件、穹顶或锥面上有阶梯色块 | 多面体着色。把曲线节点标为平滑（`[s, r, 1]`，默认即平滑着色）或传 `smooth: true`；用 `G.bandsOf` 把体切成带。 |
| 喉部、锥面或短圆柱上的色带在扇贝处停滞、顿挫 | 手打节点平滑出的轮廓：斜率在每个节点出现平台。把轮廓写成解析式 `r(s)`，并把导数作为 `smooth: { slope }` 传入（`wobble` 会列出它们）。 |
| 平滑着色的零件内部有发丝接缝 | 各色调片直接相邻。用 kit 的 `solidSvg` 配 `seamless` 路径（平滑旋切体与 `sphereOf` 会自动设置）和 `ISO_CSS`。 |
| 一排加强筋或螺栓点停在面中间 | 硬性的 `least` 截断。在 `ribsOf` 或 `dotsOf` 上用 `fade: [lo, hi]`。 |
| 平滑着色的锥面或钟形件上有直边的淡色楔形 | 色调片之间的裸填充：同一色调的片按相反方向缠绕会在非零规则下相互抵消，或一条带收尾时没有收细。kit 会定向每个色调多边形并给带收细；自己构建色调路径时，要自己定向。 |
| 收窄的体看起来像带一条淡色带的直锥 | 它被做成了外包壳（hull）。让 `lathe` 自选 `fill: "auto"`（或传 `"surface"`）。 |
| 一条发丝线横过管道的高光或阴影带 | 体与条纹在分块关节的同一条线上切开。`tubePieces` 会让条纹越过体；改动时保持这一点。 |
| 接缝与轮廓相接处轮廓呈虚线 | 接缝画在实体之后。把它们作为 `inner` 传入。 |
| 管道的条纹没到管端就停了 | 有一个端没有端盖。`tubePieces` 默认给每个端加盖；看得见的端不要传 `caps: false`。 |
| 图形分离时零件沿安装座滑移 | 它们跟着某一组件的向量、却在另一组件的弹簧上移动。给该零件补上那个组件的 `ride`。 |
| 长旋切体在带状关节处轮廓出现扭折 | 用 `G.bandsOf` 构建，并用其 `sidesOf` 一次性画出侧面（`kit.md` 第 10 节）。 |
| 一条纹理线停在面中间 | 让它终结在真实边缘上，或用 `fadeLineSvg(points, { fade })` 绘制使其渐变淡出。`scripts/lines.mjs` 会逐条列出。 |
| 盒子上的接缝没到角落 | 用 `k.sideSeam(plan, z, P, steps)` 作为 `inner`。 |
| 软管编织纹或波纹侧边硬直，或在分块关节处断裂 | `tubePieces(…, { rings: { pitch, twist, cross: true } })`。 |
| 杆滑动时轮廓有缺口 | 用 `G.rodOut` 的一根整杆、在密封套处裁剪、以 `A.slide` 记录；绝不用相互重叠的副本。 |
| LCD 或计数器数字越过窗口 | 让字形适配窗口的高度与最宽字符串，并做裁剪。 |
| 阀或凸鼓上散落着轮廓三角碎片 | 旧版 kit 会把断裂的轮廓拼接起来；现在 kit 在某侧轮廓有缺口时会回退到外包壳。用当前的 `lathe.mjs` 重新构建。 |
