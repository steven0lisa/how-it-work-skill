# 动效与交互

当图形的机械结构像真实物件那样运动时，它才是活的：滑架滑行后停稳，弹簧过冲一次，指针摆向新的读数。动效应平静得足以看清，且忠于实物模型。

## 原则
- **基于时间。** 从 `requestAnimationFrame` 时间戳取 `dt`，并将其钳制在 1/30 s 以内，避免后台标签页回来时画面跳变。绝不要每帧步进固定量。
- **向目标缓动运动（eased motion）。** 指针与键盘设置一个*目标*，部件以指数缓动（`x += (target − x)·(1 − e^(−dt/τ))`，τ = 0.1–0.3 s）或弹簧追随目标。部件绝不瞬间吸附到光标处。
- **主体有弹性就用真实弹簧。** 使用真实模型给出的刚度与阻尼，以每秒约 240 次子步进保证稳定。误差与速度都低于 5e-4 后，直接吸附到目标。
- **每个图形只有一个循环。** 用单个 `tick(dt)` 更新模型，再通过 ref 写入 DOM：`setAttribute("transform" | "d")`、`textContent`、`dataset`。不要每帧重新渲染 React。
- **到位或离屏即停止。** 使用 `IntersectionObserver`，rootMargin 设为 `120px 0px`，并读取最新一条 entry（`entries[entries.length - 1]`）：当舞台在两次回调之间离开又回来时，一次回调会按从旧到新的顺序同时送达两条 entry，若读的是第一条，图形就会冻结在屏幕上。一旦所有部件都到达目标，就停掉循环。
- **减少动态效果（reduced motion）。** 当 `prefers-reduced-motion: reduce` 生效时，直接跳到目标，不带巡游、流动或脉冲。图形仍必须响应输入。
- **自动巡游（idle tour）。** 无人交互约 1.4 s 后，播放一段脚本化循环来展示图形要点。悬停、聚焦或按下时暂停，并在最后一次交互约 3.6 s 后恢复。巡游驱动的是与用户相同的目标。
- **读数（readout）实时更新**，来自同一个模型，用真实单位且简短：`block 2 · reads 14.0 px`、`120.0 × 32.0 · lean 10.6 px`。读数取自部件实际所在处，而非目标处：Raptor 在任何弹簧仍在运动或有待处理目标时显示 `taking apart` 或 `closing up`，之后才显示 `apart · 11 assemblies` 或 `1,630 kg · cold`。阶段词和旁边的数字必须取自同一时刻、同一来源：某盲测台架曾打印 `end of stroke · relief 85 bar`（词来自模型，数字来自落后它 0.25 s 的表针），尽管其泄压阀在 92 才起跳；回缩刚结束时还打印过 `idle · 44 bar`。

## 指针
- 用某根轴的屏幕向量把指针映射到世界空间（world space）。沿 x 的运动用 `axisVector(P, [1, 0, 0])`，或把图形的宽度比例映射到部件的行程范围。行程较短时，范围映射更简单、手感也更好。
- 悬停预览、按下提交。触屏没有悬停，所以让点按直接生效，并保留 `touch-action: pan-y`，让页面仍能滚动。
- 拖动期间使用指针捕获（pointer capture），并在 `pointerup` 或 `pointercancel` 时释放。

## 键盘与无障碍
- 图形的根节点可聚焦（`tabIndex=0`），带 `role="slider"`（有离散选项时用 `"group"` 加按钮）、解释控制方式的 `aria-label`，以及用文字表述的 `aria-valuenow`/`aria-valuetext`，例如 "Pressed: 129.6 by 34.6 pixels, leaning 10.6 px"。值文本只在读者的选择变化时写入（选中某个部件、拆开或装回、点火或冷态），不要由循环写入：屏幕阅读器会播报聚焦滑块值文本的每一次变化，每帧更新的读数（"Pc 290 bar · 222 tf"）曾因此变成一串数字流。视觉读数与无障碍值要分开（Raptor 的 `say()` 只写读数，`describe()` 写值文本）；live region 只承载阶段。
- 按键：方向键以合理步长步进（Shift 为大步），Home 和 End 到两端，Space 或 Enter 按下或切换，Escape 释放。
- 显示焦点环：`outline: 1px solid var(--anatomy-muted); outline-offset: 6px`。
- `<svg>` 带 `role="img"` 和一条描述对象及其展示内容的 `aria-label`。

## 原生循环（不依赖框架）
```js
let frame = 0, last = 0, visible = false, touring = !still.matches, clock = 0;
function tick(now) {
  const dt = last ? Math.min((now - last) / 1000, 1 / 30) : 1 / 60;
  last = now;
  const calm = still.matches;
  if (touring) { clock += dt; target = tourAt(clock); }
  x = calm ? target : x + (target - x) * (1 - Math.exp(-dt / 0.28));
  springTo(tip, heightUnder(x), dt, 420, 26, calm);
  draw();                                   // setAttribute on refs, readout.textContent = …
  const settled = !touring && x === target && tip.rate === 0;
  frame = visible && !settled ? requestAnimationFrame(tick) : 0;
}
const run = () => { if (!frame && visible) { last = 0; frame = requestAnimationFrame(tick); } };
new IntersectionObserver((entries) => { visible = entries[entries.length - 1].isIntersecting; if (visible) run(); }, { rootMargin: "120px 0px" }).observe(stage);
```
`examples/dial-indicator/build.mjs` 包含完整版本，含巡游、指针映射和键盘停位点。

## React 循环
把静态画面保存在服务端构建的 props 里，只移动 `<g>` 包裹层和会改变形状的路径：

```tsx
"use client";
export function RigLive(props: { base: React.ReactNode; carriage: React.ReactNode; dial: React.ReactNode }) {
  const { anchor, say } = useReadout<HTMLDivElement>();
  const car = React.useRef<SVGGElement>(null);
  const needle = React.useRef<SVGPathElement>(null);
  const state = React.useRef({ x: REST, target: REST, tip: { at: 0, rate: 0 } });
  const visible = useInView(anchor);
  const kick = useLoop((dt, calm) => {
    const s = state.current;
    s.x = follow(s.x, s.target, dt, 0.28, calm);
    const settled = spring(s.tip, heightUnder(s.x), dt, 420, 26, calm);
    car.current?.setAttribute("transform", translateAlong(P, [s.x, 0, 0]));
    needle.current?.setAttribute("d", needleAt(s.tip.at));
    say(readoutOf(s));
    return !(settled && s.x === s.target);
  }, visible);
  return (
    <div ref={anchor} tabIndex={0} role="slider" onPointerMove={(e) => { state.current.target = toWorld(e); kick(); }}>
      <IsoFigure width={W} height={H} label={LABEL}>
        {props.base}
        <g ref={car}>{props.carriage}</g>
        {props.dial}
        <path ref={needle} className="iso-line" data-tone="lit" d={NEEDLE_REST} />
      </IsoFigure>
    </div>
  );
}
```
`examples/test-rig/live.tsx` 是完整的生产版本，含弹簧、巡游、按压与拉拽、键盘处理和 aria。

## 会变化的形状
当部件改变的是形状而非位置（膨胀的胶囊体、压缩的弹簧、转动的指针）时，每帧用套件只重建它的路径，使用 `slabOf`、`coil` 或 `lineOnTop`，并写入它的各条路径：`fill`、`shades[0..3]`、`top`、`crease`、`bevel`、`outline`。单个实体的重建远低于 0.1 ms。要实测：每帧的脚本耗时宜保持在约 1 ms 以内。
- **预计算帧会跳步。** 一根 ±15° 的杆用 41 帧预建帧绘制时，其旋钮每帧移动约 0.8 px，慢放下肉眼可见。旋转部件要么每帧重建，要么预计算足够多的帧，使单步移动不超过 0.25 px。
- **从密封压盖（gland）滑出的杆**不改变形状：它是一整根全长杆在移动，由一个原地不动的剪裁在压盖面处裁掉（`G.rodOut`、`holdStill`，见 `craft.md` 第 3 节）。绝不要把它画成几份重叠的拷贝。

## 一次性事件的包络（envelope）
闪光、咔哒、弹跳、抖动或冲击是带「年龄」的事件，不是你设置的值。用带起振（attack）与衰减（decay）的包络驱动它。`kit/gl.mjs` 导出 `smooth(a, b, x)`（smoothstep）和 `burst(age, rise, fall)`，后者在峰值之后为 `smooth(0, rise, age)` 乘以 `exp(−(age − rise)/fall)`，事件之前为 0：
```js
import { smooth, burst, settle } from "../../kit/gl.mjs";
const flash = calm ? 0 : burst(clock - mainAt, 0.08, 0.13);
```
在独立页面上，`glScript()` 已内联了这三个函数，直接使用即可，不要自行声明：同一模块脚本里出现第二个 `const smooth` 就是 SyntaxError。
- 起振为 40–120 ms，smoothstep 使其在起始点斜率为零。一帧内把值设为 1 就是突跳（pop）。
- 记录事件的起始时刻（`mainAt = clock`），每帧由其年龄计算各包络。两个重叠的事件于是直接相加。
- 给每个事件独立的标志位（`popped`），这样快速重启既不会吞掉下一个事件，也不会截断仍在衰减的事件。
- 减少动态效果下，所有事件包络均为 0。

## 临界阻尼收敛（settle）
对于应当无过冲到位的值（色相、亮度、一盏灯、一段伸出量、一个频率），使用套件的 `settle(state, target, dt, seconds, calm)`：作用于 `state = { x, v }` 的临界阻尼弹簧，时间常数为 `seconds`，`w = 1/seconds`，更新式为 `v += (w²·(target − x) − 2w·v)·dt`，分四个子步进行。它比指数跟随（exponential follow）更平滑，因为速度也是连续的。误差与速度都低于 1e-4 时吸附到目标，`calm` 下直接跳到目标，仍在运动时返回 true，因此可用它维持循环：
```js
const moving = settle(freq, state.tf, dt, 0.625, calm);
```
水波槽（ripple tank）让频率（0.625 s）和打水器间距（0.4 s）都经过 `settle`，因此按键或指针移动绝不会让波长跳变。
- **把一个阶段改变的东西分开。** 当一个物理量（比如燃烧室压力）同时驱动颜色、形状和亮度时，在该量上用快弹簧会让三者同时跳变。给颜色单设一个更慢的 `settle`（0.3–1 s），并让亮度在变化过程中只朝一个方向移动。
- **淡出只应变暗。** 停止开始时，把亮度冻结在当前值，只用递减的量去缩放它。
- **改一个弹簧会牵连邻居。** 把一个弹簧改成临界阻尼，会让所有依赖其旧过冲的阶段变慢，所以要逐一复查。

## 爆炸视图（exploded view）
悬停时把机器拆开是展示其部件的有力手段（`examples/raptor-engine/`）：
- **每个部件都建在装配位置上**，打上组件名标签，并给每个组件一个世界坐标移动向量（`PIECES.nozzle.move = [0, 0, -120]`）。移动方式遵循真实拆装：喷管降得最远，侧挂单元沿径向滑出（`out(deg, r, z)`），基座不动。
- **每个组件一个弹簧**，装配态取 0、拆开态取 1。Raptor 用刚度 64、阻尼 14.2，略低于临界值，使部件带着一点分量感停稳，每帧子步进 4 次。
- **错开各弹簧的时机。** 打开时，每个组件的目标在前一个之后 35 ms 变化，顺序须保证没有任何东西穿过任何东西（Raptor 的顺序：`nozzle, downcomer, controller, plumbing, lox, ftp, hotgas, chamber, orpb, otp, gimbal, stand`，见下文「安排错开顺序，使任何东西都无需穿过」）。关闭用相反顺序，机器从内向外折回。待处理目标保存为 `{ value, at }`，在 `clock >= at` 时应用。
- **反转时只重排发生变化的部分。** 读者中途反转（快速悬停、Space、点按）时，取消所有尚未离开组件的待处理目标，只对目标发生变化的组件按其在该较短列表中的下标错开。这样最后离开的正是最先返回者，立即返回。重排整个列表会让先走的组件朝拆开方向飞 0.3 s，而靠后的组件仍停在家里；爆炸之所以没穿模，只是因为 35 ms 后下一个组件挪开了路：在这个窗口内反转会驱动燃气导管撞进一个还没离开的甲烷泵。按此规则，沿错开次序的份额在每一帧都保持有序（先走者始终领先），这正是下文前缀规则所依赖的性质。
- **用 transform 移动。** 每个条目的屏幕偏移是其移动向量经投影后乘以弹簧值：`translate(tx ty)`。值为 0 时移除该属性，这样装配态图形就是原封不动的静态画面。
- **让线缆始终落在东西上。** 连接两个组件的部件（机架与泵耳之间的作动器、盒子与插头之间的线缆）归属其实际停靠的组件，拆开后仍要终止在某个东西上。线缆两端画端点圆点。
- **选择。** 悬停、键盘方向键和部件列表都可用于选中一个组件。被选中的部件成为唯一的 `lit` 高亮：`.it[data-on]` 上的 CSS 把 `hi` 提为 `lit`、`mid` 提为 `hi`，并把顶面和侧面切换为 lit 填充。其余部件通过 `[data-focus]` 上的 CSS 各降一档色调（`hi` 降为 `mid`，`mid` 降为 `lo`），读数说明被选部件的用途。

## 运动部件的深度重排
朝向或远离观察者移动的部件，会随移动改变画者序（painter's order）。当它们确实如此时，每帧都重新排序：
- 给每个条目一个静态键 `k = dot3(ref, viewOf(P)) + bias`，其中 `ref` 是其中心，`bias` 用于消解相贴部件之间的平局。再给它一个抬升量：`lift = dot3(move, viewOf(P))`。
- 每帧其键为 `k + lift × spring`。按键排序各下标，并且**只在顺序真正变化时才重新追加（re-append）DOM 节点**（用一个 `DocumentFragment`）。大多数帧没有任何变化。
- 把键与移动向量存在元素上（`data-k`、`data-m`），这样 live 脚本就不需要任何几何计算。
- 管道按段排序（`tubePieces` → `piece.mid`），这样一根管可以从一个部件前方经过、又从另一个部件后方经过。
- **静态键要按运动过程校定，不能只看静息姿态（rest pose）。** 每个条目的键是 `k + lift × spring`，但弹簧错开后，任一帧里每个组件都有自己的份额，因此装配态与拆开态顺序都正确的两个条目，中间过程仍可能画反：初版 Raptor 里一根增压管在每个爆炸过程的 40% 时间里藏在它本应在其前方的作动器柱后面。Raptor 对每一对重叠部件在 0、0.1 … 1 各处、在其弹簧按 live 错开重放（直至收敛）的每第四帧、在打开、关闭以及每个错开边界两侧的反转情形、以及在前缀状态（`A.explodeStates(STAGGER, SPRING).depth`，即 `[...A.uniformStates(10), ...A.springStates(STAGGER, { ...SPRING, stride: 4, reversals: A.reversalsOf(STAGGER, SPRING.stagger) }), ...A.prefixStates(STAGGER)]`，共 1,119 个状态；对以 `{ order: STAGGER, spring: SPRING }` 创建的记录器，这也是 `A.settle(R, P)` 的默认值）逐一测量真实深度，`orderKeys` 一次性解出全部状态（每条规则都用当帧的份额）。只测打开与关闭，仍会在快速悬停产生的帧里留下错误对：一个本应在燃烧室带前方的吹除法兰，在 0.20 s 的一次反转中消失到了它后面。审计对静息态和任何重放帧中的任何错误对都会判失败；只有无人会看到的均匀状态才允许保留互锁。
- **弹簧只有一份来源。** 把错开顺序与弹簧常数（`SPRING = { stagger, stiffness, damping, substeps }`）保存在构建中，并随数据传给页面，使审计重放的与观众看到的完全一致。
- **部件跟随其挂载物移动（`ride`）。** 螺栓固定在燃烧室上的传感器高亮时随管路，但移动跟随燃烧室的弹簧。每个条目在 `piece`（高亮跟随的组件）旁记录 `ride`（移动它的弹簧所属组件，由其移动向量求出），页面读取 `data-ride`，`place()` 在平移与排序时都用 `piece[node.ride].x`。借用其他组件的向量却保留自己弹簧的条目，会沿挂载物滑移，并在半途沉入其中。
- **前缀规则。** 在先走者已出的前提下，每个组件的路径都必须避开所有还停在家里的更晚离开者：这正是错开边界处反转所冻结的那一帧。`A.prefixStates(STAGGER)` 摆出的正是这些姿态（前 `k − 1` 个已出，第 `k` 个在 0.25 … 1，其余在家），审计将其与重放帧一并检查。在 Raptor 上，它把燃气歧管排到其所螺栓连接的甲烷泵之后（其向量指向泵内），把控制器排到管路之前（氧气到预燃室的管线被控制器的感测线包裹，只能在它们之后离开）。
- **安排错开顺序，使任何东西都无需穿过。** 弹簧在时间上重叠：启动间隔 35 ms、收敛约一秒，所有组件同时都在动，相对运动是所有向量的混合。初版 Raptor 爆炸时有 66 对部件在半途互相穿过。以下是经审计逐帧并在反转时检验、把穿模降到零的规则：
  - **在外面者先走。** 每一侧，其他一切都藏在其后或缩于其下的那个部件先动：Raptor 的喷管先于一切下降（所有东西都坐在它上面）；下导管（downcomer）沿所有通向甲烷泵的感测线与线束的外侧走，排第二。关闭时顺序反转，所以外侧部件也是最后回家的。
  - **当一个部件的爆炸向量指向另一组件的某个面时，该组件在顺序中更早。** 下导管下降 84，落到喷管的端口三通（port tee）上；喷管下降 120 且早 35 ms 出发，所以无论打开还是关闭，端口始终位于法兰下方。
  - **每个向量都要避开该部件相触的每一个面——在另一侧仍在原位时**，而不只是它螺栓连接的那两个：穿行于燃烧室与泵之间的管线，必须在两个安装法线共同允许的扇区内移动（Raptor 的夹套管线转到 −120° 而不下降，从两块垫板上滑脱）；缠绕核心部件的线缆，只能在对其整段缠绕都朝外的扇区内移动（90° 的缠绕允许 90°；超过 180° 的缠绕不允许任何活动，所以要缩短它或在近侧终止）；会向下扫过其下方所连部件的管线不得下降。
  - **互相穿绕的东西共用一个弹簧。** 从线束环中升起、进入基块的感测线，与该基块下方的线束，不能各用各的弹簧：相隔 35 ms 它们会互相剪切。Raptor 的感测线与基块跟随控制器（`BLOCK_MOVE = PIECES.controller.move`，因此 `ride` 解析为 `controller`），高亮时仍归管路。
  - **只挂载于单一部件的管线，只取该部件的移动。**
  - **检查观众能造出的路径**，而不只是从静息态出发的那两条：快速悬停会让弹簧中途反转（`reversals`），回程偏晚的组件可能扫过一条已到家的线缆。在每个错开边界处反转（`A.reversalsOf`），不要只挑几个整时间点：坏窗口约 15 ms 宽，0.15、0.3 和 0.5 s 一个都没撞上。把这些帧既喂给深度状态，也喂给间隙检查，并做实体对实体的检验（`A.solidClearances`）：一个藏在宿主悬垂结构下的法兰，在回家路上撞上了宿主弹簧的尾段，重叠深度 0.67，而所有管道检查都是零。

## 手动旋转
对 3D 图形（`references/3d.md`）：
- **拖动。** `controller.angleAt(clientX, clientY, zFace, "dial")` 是指针在该面平面上的角度，跨圈连续展开。按下指针时，记录该角度与当前目标。移动时设 target = start + Δ，移动超过约 1.5° 后捕获指针。该组以指数跟随追随（τ = 0.12 s）。舞台保持 `touch-action: pan-y`。
- **按键。** 方向键步进一格，Shift 一步十格，Home 回到基准刻度。
- **联动组。** 由一个值经 `follow(values)` 驱动（刻度盘四分之一处的轮子、由单个 `lift` 带动的指针链），使工具、拖动与巡游三者保持一致。
- **手势与剪裁（clip）是时间的纯函数**（`tapAt(t)`、`breakAt(t)`）：构建对它们采样，审计逐帧走查，页面可以倒放它们来无跳变地复位。
- **钩子接管。** 任何 `window.__isoTurn` 调用都会触发 `onHold`：停止巡游，并保留工具设置的值，直到读者再次操作。
- **半帧率。** 稠密图形把 `controller.set` 的平均耗时控制住：帧耗时高于 14 ms 时隔帧渲染，低于 9 ms 时逐帧渲染，基于时间的运动保持不变。
- 减少动态效果时只给一帧静帧。离屏时循环停止（IntersectionObserver）。

## 预备（arming）
如果一个动作需要机器处于特定状态（点火前先装配、旋转前先关门），先预备：
1. 设定使其进入该状态的目标（`setApart(false)`、`select(null)`）。
2. 标记为已预备：按钮显示 `data-state="armed"`，读数显示 `closing up · arming`。
3. 只有当每个弹簧都已到位且没有任何待处理目标时，才启动动作的时钟。

绝不要在半拆状态的机器上运行动作，也绝不要为省时间把部件瞬间弹回家。

## 从巡游中接管
当用户在自动巡游期间操作时，停止巡游并把状态留在原处。如果你的接管辅助函数会重置某个东西（例如强制打开电源），而用户的操作本身正是对该东西的切换，两者会在同一帧内互相抵消并重播一段动画。给辅助函数传一个标志（`takeOver(switching)`），并在巡游的每个阶段测试每个切换项，而不只是默认阶段。

巡游期间按下的步进键必须从读者看到的值出发，而不是从可能远在前方的巡游目标出发。水波槽的按键在巡游时从 `freq.x` 和 `gap.x` 出发（其余时候用 `state.tf` 和 `state.td`），`steer()` 在接管时把两个弹簧的速度清零，使值只朝按键的方向移动。

## 播报阶段
对有阶段的机器，添加一个视觉上隐藏的 `aria-live="polite"` 元素，只把阶段变化写入其中：`arming`、`chill-down, venting`、`preburners lit`、`main stage`、`shutdown`、`cold`。绝不要把每帧数字写进去。对连续控制，等输入安静约 700 ms 后再播报（水波槽的 `announce`）。加载时让 live region 保持为空，在 `focus` 时填入当前状态，让屏幕阅读器在焦点到达时听到状态，而不是听到一条过期内容。标签会变化的按钮（"Ignite" 变为 "Shut down"）不需要再加 `aria-pressed`。

## CSS 动效
- `.iso-line[data-flow]` 让虚线流动，用于沿路径移动的东西。
- `.iso-dots circle[data-pulse]` 缓慢闪烁点阵，用于表示「激活」状态的表面。
- 悬停抬升：给一组包上 `transition: transform 700ms var(--anatomy-ease)`，并在悬停时向上平移。

以上所有效果在 `ISO_CSS` 中都会随减少动态效果而关闭。
