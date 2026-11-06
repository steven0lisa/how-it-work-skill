# 变更日志

How It Works 的所有显著变更都记录在此。格式遵循 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)。发布遵循 [Semantic Versioning](https://semver.org/spec/v2.0.0.html)；首次发布之前的条目以日期标注。

## [Unreleased]

### 新增

- 可打印图版。`scripts/print.mjs` 将已构建图形的姿态冻结为矢量静帧（WebGL 输出为 2× PNG），并以三种格式打印一份 A4 或 US Letter PDF：`poster`（一页：大标题、数字、固定姿态的图形与每个零件一行说明）、`drawing`（一页：零件清单、规格说明、局部放大图与一条高亮步骤条）和 `detailed`（两页：带圆形特写的固定姿态图形，然后是带文字与缘由说明的大幅高亮步骤）。零件是对实时 SVG 的 CSS 选择器，因此任何图形都可以打印；拥挤的定位针会以发丝级引线散开，当某种格式超出其页数时脚本会报错失败。`examples/raptor-engine/sheet.mjs` 与 `examples/arcade-cabinet/sheet.mjs` 是参考实现，`references/print.md` 是指南。
- 新增 Antigravity 支持，与 Claude Code、OpenCode 和 Codex 并列。在 README 与 CONTRIBUTING 中记录了用户级与项目级安装路径，更新了 SKILL.md 的兼容性说明，并将 Antigravity 加入 bug 报告表单。
- 新增 MiniMax Code 支持。README 与 CONTRIBUTING 记录了其用户级（`~/.minimax/skills`）与项目级（`.minimax/skills`）目录以及 `npx skills add ... -a minimax-code`，SKILL.md 列出了它，bug 报告表单也提供了该选项。

### 变更

- Figure 不再向用户报出时间预估。30–90 分钟的估计常常远长于构建本身（[#41](https://github.com/wheresryan22/anatomy/issues/41)）。

### 修复

- 线端审计以有界的屏幕空间近似来跟踪三次与二次曲线描边，因此连接在曲线采样点之间的线不会再被报告为松脱。

- `drive.mjs` 在运行第一个动作之前等待页面加载完成（最多 10 秒），当 Chrome 对请求 60 秒无响应时报错放弃而不是挂起，`capture.mjs` 现在也如此。
- `drive.mjs` 在 Chrome 启动之前就拒绝步长为零或指向偏离其结束角度的 `orbit`，以及数量不是至少为 1 的整数的 `sheet`。步长为零过去会不断捕获帧且永不结束；其余情况则以 `Cannot read properties of undefined (reading 'width')` 报错。
- `publish.mjs --force` 在新发布失败时（`--pages` 文件不存在、文件越出文件夹之外、构建失败）保留它原本要替换的图形；缺失的 `--pages` 文件现在会报 `page X not found` 而不是干巴巴的 `ENOENT`。
- `publish.mjs` 在构建图形之前拒绝不受支持的许可证取值，以符合社区画廊的 MIT 与 CC-BY-4.0 模式。
- `capture.mjs` 在 Chrome 启动之前就拒绝它无法使用的选项取值（`--zoom` 缺少四个数字、`--hover` 缺少两个、非数字的 `--width`、`--scale`、`--wait` 或 `--times`、`--theme` 不是 dark 或 light），而不是假装这些选项已生效并保存截图。
- `turn-fidelity.mjs` 在调用 `k.extrude` 之前，先通过立式棱柱的创作坐标系 `F`（`F.o`、`F.u`、`F.v` 与向下的 `F.a`）对其进行变换，因此 `turn-check.mjs --fidelity` 能在立式坐标系创作与垂直切分的棱柱上与 `T.prism` 相匹配。
- `audit.mjs` 的 `supports` 改为在与每段切线垂直的圆环（`ringAround`）上采样刚性管表面，而不是水平的 `XY` 平面圆，因此水平与倾斜的管能检测到垂直及垂直于管身的支撑接触，不再越过其平端帽桥接空气间隙。
- `T.prism` 多边形中的重复顶点不再把圆角推到多边形之外：`roundedPolygon` 在圆角化之前丢弃连续的重复点。
- `T.prism` 拒绝星形多边形（如五角星）——它在每个顶点都同向转弯但绕行两圈——并报出与其他非凸零件相同的“拆分为凸块”错误。
- 3D 旋转审计对剪辑的采样覆盖到最后一个时刻且绝不超过它。长度不是整数帧的剪辑过去会被采样到其结束之后，或止步于其最终姿态之前。
- `capture.mjs` 在接触页面之前先等待其加载完成，即使 `--wait 0` 也如此；当 Chrome 对请求 60 秒无响应时报错放弃而不是挂起。
- `drive.mjs` 的 `until` 在页面内部为条件计时，因此已经为真的条件无论等待多短都能通过，即使在慢机器上也是如此；永不落定的 promise 仍会超时。
- 在 Windows 上从提权终端启动无头 Chrome：`capture.mjs`、`drive.mjs` 与 `turn-check.mjs` 在该平台传入 `--do-not-de-elevate`。否则 Chrome 会以非提权方式自行重启，脚本启动的进程以退出码 0 退出，检查以 `Chrome exited before startup: 0` 停止，而重启后的浏览器仍在运行。
- 每次写入时都重新解析当前的 React 读数目标，使被替换的读数与锚点不会把更新留在已脱离的元素上。
- 当页面没有可调用的 3D 旋转控制器时拒绝 drive 的 `turn` 与 `orbit` 动作，而不是把未变化的图形报告为一次成功的旋转。
- 以 await 等待异步 `until` 条件，而不把 promise 本身当作成功，同时执行等待期限。
- Capture 在每次截图之前测量当前图形，包括移动过、调整过尺寸与按时间触发的 zoom 裁剪。
- `publish.mjs --submit` 在 gh 2.101 及之后版本上重新执行 fork——这些版本在指定仓库名时拒绝 `--remote`；这种情况下 gh 从不添加 remote，因此该旗标已不存在（#53）。
- Ripple Tank 的水面与纸面着色器改为从与图形中其他所有调校常量相同的按主题固定的 uniform 读取墨色，而不是在 GLSL 中内联携带自己硬编码的 `pale ? light : dark` 配对。

## [1.1.0] - 2026-10-12

### 新增

- `scripts/publish.mjs` 与 `references/publish.md`：在社区画廊 `wheresryan22/anatomy-community` 中分享一个完成的无框架图形。脚本复制该图形，把其 kit 导入指向画廊的 `.skill/` 链接，记录 kit 指纹（`scripts/fingerprint.mjs`）以便画廊能用同一 kit 版本重建它，且只复制源文件，放在 `figures/<author>/<id>/` 下。画廊在锁定的沙箱中构建页面。`--submit` 在询问之后将其作为 pull request 发送，需要时创建 fork。
- `publish.mjs` 记录图形制作耗时与消耗的 token 数。在 Claude Code 中，它从会话自身的日志读取这些数据（`scripts/session.mjs`，通过 `CLAUDE_CODE_SESSION_ID`，包含子智能体）：从制作图形的请求到发布它的请求，剔除超过 10 分钟的空闲间隔且不计缓存读取。在任何内容被发送之前，计划中会先展示这两项。`--minutes` 与 `--tokens` 手动设置这两项，`--no-session` 则不含它们。
- `publish.mjs` 接受 `--agent Antigravity`，与 Claude Code、OpenCode、Codex 及 Other 并列。
- 新增 OpenCode 与 Codex 支持，与 Claude Code 并列。`SKILL.md` 通过 `metadata` 向 OpenCode 暴露 `/how-it-work`，`agents/openai.yaml` 为 Codex 提供其显示名称，README、CONTRIBUTING 与 bug 报告表单覆盖全部三个智能体。

### 修复

- 在 Linux 上启动无头 Chrome：`capture.mjs`、`drive.mjs` 与 `turn-check.mjs` 在该平台传入 `--no-sandbox`，并在 `~/.cache/ms-playwright` 下寻找 Playwright 的 Linux Chromium，因此浏览器检查与画廊的验证器能在 Linux 主机上运行。
- `references/publish.md` 以 `--single-branch` 克隆画廊，贡献者由此跳过 previews 分支及其图片。
- 当早前不带 `--submit` 的运行把图形文件夹留在了克隆中时，`publish.mjs --submit` 不再以 “already exists” 停止；只有当该文件夹已在画廊的 main 上时才要求 `--force`。
- 在宽阶段的实体与管采样中保留已配置的管间隙。
- 通过 useInView 观察被替换及延迟挂载的 DOM 目标。
- 对照圆角矩形的角检查纹理线的附着，包括分离的半径、SVG 钳制与变换后的轮廓。
- 遵循显式的 capture、drive 与线检查选择器，并把报告的读数限定在所请求的图形内。
- 在审计边界中解码合法的 SVG 属性引号、空白与 XML 字符引用。
- 从创作的 SVG 边界中排除不可见的空裁剪交集。
- `scripts/drive.mjs` 现在在浏览器求值抛出或拒绝时失败，而不是报告一次干净成功的会话。
- 街机柜现在在启用减弱动态效果时也能接受投币，在跳过投币动画时保留电表、积分与开机事件。
- 旋转表盘在页面打开期间启用减弱动态效果时停止其闲置巡游，并取消其待定的自动重启，同时保留键盘与指针输入。
- `scripts/drive.mjs` 现在在 `until` 条件超时处以状态码 1 退出并停止后续动作。
- `scripts/capture.mjs` 现在会为截图创建输出目录，包括 `shots/` 下的嵌套路径。
- 当 `--select` 指定的是一个 SVG 时，线端检查现在会包含该元素本身，而不是只检查其内部的 SVG。
- `scripts/drive.mjs` 现在在运行会话之前拒绝未知动作，因此拼写错误的验证步骤无法报告成功。
- 当 Chrome 在其调试端口就绪之前退出时，浏览器检查现在会立即失败并移除其临时配置目录。
- 线端检查现在使用椭圆边界与变换后的圆边界，并把空心内部视为空而不是实心圆点。
- 线端检查现在沿 SVG 弧曲线跟踪，而不是把每段弧当作直线弦。
- 当 Chrome 无法加载所请求的页面时，capture 与 drive 现在会失败，而不是报告一个干净的浏览器错误页。
- 轨道检查在零状态扫掠得以通过之前拒绝无效的采样步长与空的姿态列表。
- 当采样间距或单元格尺寸变化时，实体间隙审计会重建缓存的面网格。
- 管几何审计使用锥形变化的半径，而不是把每一段都替换成平均半径的圆柱。
- 线验证器仅在匹配到对应曲线族之后才反映平滑 Bezier 控制点。
- Capture 与 drive 发送原生的 Enter 激活，使按钮与表单检查能执行其默认动作。
- 固定零件与覆盖范围边界在相对移动之前恢复闭合的 SVG 子路径起点。
- 审计光线投射现在在投影方向变化时刷新主体相机系数，因此重复构建仍保持正确的深度顺序。
- 审计轮廓描迹现在检查四分之一跨度的偏差，保留中点恰好落在其弦上的带拐点解析轮廓与平滑轮廓。
- 深度键求解现在允许大型无环图形在 400 轮之后继续收敛，并避免从空的被违反规则集合中选择循环牺牲品。
- 切分现在接受默认的固定轮次并保留显式的棱柱/圆形归属，因此管接头可以被切分而不会凭空造出与其自身管道的碰撞。
- 旋转构建器现在在创作好的动作静默变成静态或多个动作别名共用一个值之前，拒绝缺失的命名零件/父组与重复的组名。
- 原生保真检查现在对切割零件比较完整的填充轮廓，并遵循 kit 参考中开放的圆形端部，保留刻意不加描边的切割边缘。
- 当顶点数量不同时，管保真检查在两个方向上检查路径覆盖，拒绝空或被截断的实时轮廓，同时保留等价的重新采样。
- 实体间隙检查会检查所绘延伸范围发生变化的滑动杆，即使其伸展几何相对静止障碍没有留下任何相对平移。
- React 的 `Solid` 遵循现有的 `paths.seamless` 标记，使平滑色调区域获得与纯 SVG 渲染器相同的接缝处理。
- React 的 `Dots` 保留 kit 可选的逐点不透明度，包括淡出的螺栓圆与脉动的圆点，同时普通点保持不透明。
- 即使为空或在被动绑定之前发出，也写入第一个读数值。
- 通过真实的仿射边界测量变换后的矩形与圆形 SVG 图元。

## [1.0.0] - 2026-10-10

### 新增

- `examples/arcade-cabinet/`：一台拆掉侧板的立式街机柜。一枚 25 美分硬币沿投币滑槽滚过投币开关的金属丝落入钱箱，开关的脉冲沿其线缆传到主板，CRT 预热、自检并在倾斜显示器的平面内运行一段演示模式；摇杆在面板下方按下微动开关。它以亮色与暗色主题绘制并通过 `--audit`。
- `examples/spot-plate/`：与 `test-rig` 同一站点的第二个 React 图形：当一排探测叶片设定 `merge` 时，六对玻璃液滴合并成一块玻璃。它以亮色与暗色主题绘制。
- 第一个工作流步骤，**Size it before you start（动手前先定规模）**：每个请求都被定为 Figure（约 30–90 分钟）、Hero（2–4 小时）或 Epic（6 小时或更多）中的一个规模，且先把预估告知用户。只有当用户在 Epic 与提议的 Hero 版本之间选择了 Epic 时，Epic 才会开始。每个规模都要在第一个小时内拿出一个可工作的页面，当某一步骤超出其预估份额的两倍时 Claude 会报告。

### 修复

- `examples/test-rig/` 在亮色主题下把玻璃按钮画成了暗色。`stretch.css` 现在为它自己的颜色提供了亮色取值。

### 变更

- 工作流步骤重新编号为 1–10。
- 4× 细节扫掠随规模缩放：Figure 扫主体与机构，Hero 或 Epic 扫整幅图。密度规则本身仍适用于每个图形。
## 2026-10-08

### 变更

- 该技能现在名为 How It Works：`SKILL.md` 中为 `name: how-it-work`，以 `/how-it-work` 调用。在此日期之前它以 `isometric-objects` 之名私下开发，没有发布过。
- CSS 令牌现在是 `--anatomy-*`（例如 `--anatomy-card`、`--anatomy-face`、`--anatomy-shade-0`）；之前是 `--iso-*`。每个示例都用新名称重新构建。
- 图形卡片（`.iso-plate`）是方形且扁平的：没有圆角，没有悬浮阴影，只有一条绘图中间色调的 1 px 细线。
- `SKILL.md` 把高细节设为每个图形的常设规则，把零件清单提高到至少 40 个零件（主打图形为 60–150 个），并在最终检查中加入 4× 细节检查。

### 新增

- 公开仓库：一份带示例截图的 README、MIT 许可证、贡献指南、行为准则、安全策略、issue 表单、pull request 模板，以及一个会重建每个示例、若已提交页面发生变化则失败的 CI 工作流。
- 此时该技能包含的内容：`SKILL.md` 中的工作流与工艺规则；kit（SVG 核心、圆形零件、管道、几何审计、WebGL 层、React 组件，以及 beta 阶段的 3D 模式）；八篇参考文档；`scripts/` 中的无头 Chrome 检查；六个示例（`dial-indicator`、`desk-computer`、`raptor-engine`、`ripple-tank`、`test-rig`、`turning-dial`）；以及五个 eval。
