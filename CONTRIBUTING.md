# 参与 How It Works 贡献

感谢你的帮助。How It Works 的评判标准是其图形在 4× 放大下的观感，因此本指南的大部分内容讲的是在别人看到之前先检查你的工作。

参与即表示你同意遵守 [Code of Conduct](CODE_OF_CONDUCT.md)。

## 环境准备

你需要：

- **Node.js 20.10 或更高版本。**无需安装任何东西：kit、构建脚本与各项检查只使用 Node 的内置模块。
- **Google Chrome 或 Chromium**，用于浏览器检查。脚本会在常见位置找到它；否则将 `CHROME_PATH` 设为该可执行文件。
- **Claude Code**、Antigravity、OpenCode、Codex 或 MiniMax Code，用于试用技能本身。

```sh
git clone https://github.com/wheresryan22/anatomy.git
cd how-it-work
```

要在修改期间把你的工作副本当作技能使用，请把它软链到你的个人技能目录（先把已安装的副本移开）：

```sh
ln -s "$PWD" ~/.claude/skills/how-it-work
```

分别对应 Antigravity、OpenCode、Codex 或 MiniMax Code：

```sh
ln -s "$PWD" ~/.gemini/antigravity/skills/how-it-work
```

```sh
ln -s "$PWD" ~/.config/opencode/skills/how-it-work
```

```sh
ln -s "$PWD" ~/.agents/skills/how-it-work
```

```sh
ln -s "$PWD" ~/.minimax/skills/how-it-work
```

## 构建示例

每个无框架示例都是一个脚本，它把独立页面写在自己旁边。构建出的页面会被提交。

```sh
node examples/dial-indicator/build.mjs
node examples/dial-indicator/build.mjs --light
node examples/desk-computer/build.mjs
node examples/desk-computer/build.mjs --light
node examples/arcade-cabinet/build.mjs
node examples/arcade-cabinet/build.mjs --light
node examples/ripple-tank/build.mjs
node examples/ripple-tank/build.mjs --light
node examples/raptor-engine/build.mjs
node examples/raptor-engine/build.mjs --light
node examples/turning-dial/build.mjs
node examples/turning-dial/build.mjs --light
node examples/turning-dial/build.mjs --light --verify
node examples/turning-dial/stress.mjs --light
```

Raptor 的构建各需几分钟；其余只需几秒。`examples/test-rig/` 与 `examples/spot-plate/` 是 React 组件，没有构建脚本。

构建之后，除非你本来就要改动某个图形，`git status --short` 必须什么都不显示。CI 会运行完全相同的这些命令，然后再运行 `git diff --exit-code`。

## 各项检查

运行与你改动内容相对应的检查，并查看它们写出的每一张 PNG：打开它，亲眼看。截图放在被忽略的 `shots/` 里。

**浏览器驱动回归。**改动 `scripts/drive.mjs` 时，检查失败的浏览器求值会以非零退出码退出、成功的求值仍然正常工作。

```sh
node --test scripts/drive.test.mjs
```

**几何审计。**用于带管道、线缆或圆形零件的图形。它必须以 `audit passed` 结束，且每一行失败项为 0。

```sh
node examples/raptor-engine/build.mjs --audit
```

**线端。**每条接缝、棱筋、刻度与指针都必须终止在描边、圆点或轮廓上。它必须打印 `line ends: 0`；`--out` 会把失败处圈出来。

```sh
node scripts/lines.mjs examples/dial-indicator/dial-indicator-light.html --out shots/lines-light.png
node scripts/lines.mjs examples/dial-indicator/dial-indicator.html --out shots/lines.png
```

**Capture。**静帧、手机宽度、悬停、按键与 4× 特写，开启 WebGL。

```sh
node scripts/capture.mjs examples/ripple-tank/ripple-tank-light.html --out shots/desk.png
node scripts/capture.mjs examples/ripple-tank/ripple-tank-light.html --out shots/phone.png --width 390
node scripts/capture.mjs examples/ripple-tank/ripple-tank-light.html --out shots/joint.png --zoom 200,80,220,170
node scripts/capture.mjs examples/ripple-tank/ripple-tank-light.html --out shots/still.png --reduced
```

**Drive。**脚本化会话：移动、点击、按键、等待某个阶段，以及放慢速度以捕捉着色器中跳变的 contact sheet 拼图。`--no-webgl` 检查 SVG 回退。

```sh
node scripts/drive.mjs examples/raptor-engine/raptor-engine-light.html '[["wait",1500],["key","i"],["wait",3000],["shot","shots/firing.png"]]'
node scripts/drive.mjs examples/raptor-engine/raptor-engine.html '[["wait",1500],["shot","shots/no-webgl.png"]]' --no-webgl
node scripts/drive.mjs examples/raptor-engine/raptor-engine-light.html --actions ignite.json
```

任何带 `until` 或 `eval` 的内容都放进动作文件。这个 `ignite.json` 先等发动机静止，然后以 20× 慢动作拍下点火开始：

```json
[
  ["key", "Escape"],
  ["until", "document.querySelector('[data-readout]').textContent.includes('cold')", 15000],
  ["slow", 0.05],
  ["key", "i"],
  ["sheet", "shots/ignite.png", 100, 16],
  ["slow", 1]
]
```

**3D 图形。**轨道审计很慢（完整运行每个零件预算 30 到 40 秒），因此用 `--quick` 迭代，并把完整审计放到后台运行。只有完整运行的 `turn audit passed` 才算数。

```sh
node examples/turning-dial/build.mjs --light --audit --quick
node examples/turning-dial/build.mjs --light --audit > audit.txt 2>&1 &
node scripts/turn-check.mjs examples/turning-dial/build.mjs --fidelity
node scripts/turn-check.mjs examples/turning-dial/turning-dial-verify-light.html --order --step 1
node scripts/turn-check.mjs examples/turning-dial/turning-dial-light.html --lines
node scripts/turn-check.mjs examples/turning-dial/turning-dial-light.html --perf --width 390 --throttle 4
node scripts/turn-bench.mjs examples/turning-dial/build.mjs
```

[`references/verify.md`](references/verify.md) 列有全部选项、评审清单与常见错误。[`references/3d.md`](references/3d.md) 涵盖 3D 检查。

## 内部规则

这些规则来自 [`SKILL.md`](SKILL.md) 与 [`references/`](references)。违反其中任何一条的 pull request 都会被要求修改。

- **代码里不写注释。**不要文件头注释，不要难懂行旁的注解，也不要 JSDoc。把解释放进命名与结构里。`"use client"` 这类工具指令不是注释，予以保留。
- **kit 改动须选择加入。**对 `kit/` 或 `scripts/inline-kit.mjs` 的改动要新增一个选项或函数，其默认值是旧行为。每个现有示例都必须重建后逐字节相同；CI 会检查这一点。如果某个改动意在改变图形，请在 pull request 中说明，并把重建后的页面一并提交。
- **两套主题都能工作，且亮色随包发布。**每个颜色都是一个令牌，有亮色取值与暗色覆盖，绝不使用图形 CSS 或着色器里的字面量。亮色着色器是为纸张设计的（正常的预乘 alpha，绝不用 `screen`），而不是把暗色光晕调暗。用 `--light` 构建，并检查两个页面。
- **不得重叠。**静止时、分开时或运动的任何一帧里，任何管道或线缆都不得穿过任何东西。每处交叉都以真实的深度顺序绘制，由审计证明，绝不靠肉眼调校。联锁问题在几何中修复，绝不用键偏移糊弄；并记录你所绘内容的真实形状：一个能让审计闭嘴的替身形状就是 bug。
- **线条有着落，纹理会渐隐。**每条线都终止在某样东西上。棱筋行、螺栓圆、接缝或直规线要么终止在真实边缘上，要么渐隐消失；绝不停在表面中间。曲面平滑着色，没有色调阶梯。
- **着色器绝不跳变。**每个 uniform 都沿带起振与衰减的包络变化，相位切换是一次交叉淡化，每一项都在任何边界或画布边缘之前归零。特效位于图形自身相机的世界空间中，没有 WebGL 时 SVG 依然可读。
- **每个数字都真实。**读数、运动与说明文字来自同一个带真实数字的模型。文档遵循同一规则：绝不写下你没有核实过的数字或功能。
- **逐张查看 PNG。**在请求评审之前，把两套主题下每个接合处的 4× 特写都看一遍。

## 修改技能文本

`SKILL.md` 与 `references/` 是 Claude 遵循的指令，因此要保持简短、精确、同样平实的语气。改动 kit API 时，在同一个 pull request 中更新 [`references/kit.md`](references/kit.md)。新增示例时，在 `examples/` 下给它一个文件夹，内含一个 `build.mjs`，写出 `<name>.html`，并在带 `--light` 时写出 `<name>-light.html`；把它加入 `SKILL.md` 的清单、README 的表格以及 [`.github/workflows/ci.yml`](.github/workflows/ci.yml) 的构建清单。

## Pull request

1. 对于任何超出一次修复规模的事，先开一个 issue，以便我们就方案达成一致。
2. fork 仓库并从 `main` 建分支。
3. 每个 pull request 只做一个改动。
4. 重建每个示例，并确认 `git status --short` 只显示你打算改动的部分。
5. 运行适用的检查，并为任何视觉改动附上两套主题的截图。
6. 在 [`CHANGELOG.md`](CHANGELOG.md) 的 `Unreleased` 之下加一行。
7. 填写 pull request 模板。

合并之前 CI 必须通过。

## 报告 bug 或请求图形

使用 [issue forms](https://github.com/wheresryan22/anatomy/issues/new/choose)。对于产出有误的图形，请附上提示词、主题、图形的载体（HTML、SVG 还是 React），以及故障的截图或特写。
