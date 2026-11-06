# 可打印图版（A4）

完成的图形也可以打印出来：一份 A4（或 US Letter）PDF，把这台机器讲在纸上，与动态图形（live figure）出自同一个页面、同一个模型。只在用户索要可打印的东西时才制作：讲义（handout）、海报（poster）、PDF、一张 A4 或 Letter 图版（sheet）、教室墙面张贴物或活页夹材料。

`scripts/print.mjs` 负责这项工作。它在无头 Chrome（headless Chrome）中打开构建好的浅色页面，播放你列出的各个姿态（pose），并把每个姿态定格（freeze pose）为矢量 SVG（WebGL 层则为 2× PNG）。随后它把这些静帧（still）按三种版式（format）之一排版，并用 Chrome 打印出 PDF。

```bash
node scripts/print.mjs examples/raptor-engine/sheet.mjs                 # poster, A4
node scripts/print.mjs examples/raptor-engine/sheet.mjs --format all --png
node scripts/print.mjs examples/arcade-cabinet/sheet.mjs --format detailed --letter
```

它会把 `<out>-<format>-a4.pdf` 和 `.html` 写在规格说明（spec）旁边（使用 `--letter` 时为 `-letter`），加 `--png` 时再输出一份预览 `.png`。当某个版式超出其页数上限时，脚本以退出码 1 结束。

## 三种版式

| 版式 | 页数 | 版面内容 | 适用场景 |
|---|---|---|---|
| `poster`（默认） | 1 | 大字呈现的名称、问题与一段简短导语、一列大数字、带编号定位钉（pin）的图形，随后是每个零件配一句功能说明 | 墙面张贴、封面页、初次浏览 |
| `drawing` | 1 | 带编号的图纸（drawing）标题栏、带钉图形、零件表（parts list）、规格说明（specification）、另一姿态的小幅局部放大（inset），随后是一条步骤条，点亮正在工作的零件 | 技术讲义 |
| `detailed` | 2 | 第 1 页：带定位钉与字母特写标记的图形、问题、导语、数字、每个零件及其说明，以及图形下方的圆形特写。第 2 页：每个步骤为一幅大幅点亮静帧，配两三句话、缘由（why）与规格说明 | 作业、活页夹、想彻底理解它的人 |

三种版式都极简，黑白打印效果良好：发丝线（hairline）分隔线、正文用一种无衬线字体、标签只用等宽大写、黑色定位钉，图形以浅色主题印在白纸上。

## 规格说明

图版是放在示例旁的一个模块 `sheet.mjs`，其默认导出声明页面在哪里、要捕捉什么、要说什么。其中的每个数字与句子都必须来自图形的模型或说明文字（caption），与图形本身遵循同一条真相规则（truth rule）。

```js
export default {
  page: "raptor-engine-light.html",             // the built light page, relative to this file
  out: "raptor-engine",                         // output base name
  layers: ".rp-stage > svg.rp-back, .rp-stage > svg.rp-front", // the SVG layers to freeze, back to front
  rasterSelector: ".rp-stage",                  // the box that WebGL shots are cropped from
  strip: [".rp-deckglow"],                      // elements to drop from every still
  reduced: true,                                // emulate reduced motion (calm, no tour); false to catch animation
  views: {                                      // poses, captured in this order, each after its actions
    rest: {},
    apart: { actions: [["key", " "], ["wait", 2500]], reset: [["key", " "], ["wait", 2500]] },
    firing: { actions: [["key", "i"], ["wait", 3500]], raster: { region: [250, 150, 432, 530] } },
  },
  name, title, number, tagline, question, lead, quote,
  numbers: [["230", "tf", "thrust at sea level"]],
  spec: [["Chamber", "300 bar"]],
  why: { title: "Why full flow?", text: "…" },
  parts: [{ id: "otp", name: "Oxygen turbopump", say: "…", select: '.it[data-piece="otp"]', at: [x, y] }],
  main: { view: "apart", crop: [x, y, w, h], layers: [1], hide: ["stand"] },
  inset: { label: "Assembled", view: "rest", crop: [x, y, w, h] },
  steps: [{ view: "apart", title: "…", text: "…", focus: ["otp", "ftp"] }],
  details: [{ title: "…", text: "…", around: ["otp"], focus: ["otp"] }],
};
```

- **动作（Actions）**就是 `drive.mjs` 的动作。先 `["slow", 0.03]` 再 `["until", js]` 可以精确捕捉动画的某一瞬间（街机滑槽里的硬币）；`reset` 在下一个视图之前把页面复原。
- **零件（Parts）**是对动态 SVG 的 CSS 选择器，因此任何图形都适用：Raptor 的部件是 `.it[data-piece]` 组，街机的部件是取自其录制器名称的 `[data-name^="mech"]` 前缀。没有 `name` 的零件不会被列出或钉住，但步骤仍可点亮它（`coin`）或隐藏它（`stand`）。定位钉位于零件最大形状的中心；当该处落在空白区时，用 `at`（viewBox 单位）指定钉的位置。相互拥挤的定位钉会彼此散开，并画一条发丝线连回各自的点位。
- **聚焦（Focus）**点亮列出的零件（黑色线条），其余一切变灰，无论是否带标签。**隐藏（Hide）**把某个零件从该静帧中移除：把挡在特写前面的东西隐藏掉（街机挡在机械装置前面的投币门）。
- **裁剪（Crops）**使用图形的 viewBox 单位。不指定时，主视图裁剪到其绘制内容；步骤与特写围绕各自的 focus 或 `around` 零件裁剪，宽度绝不低于主裁剪宽度的 42%。
- **栅格化（Rasters）**仅用于 WebGL。给视图标记 `raster`，并提供要拍摄的 viewBox `region`；羽流（plume）以 2× PNG 的形式印在矢量页面之中。
- **文字**必须放得下：步骤标题不超过五个词，每个零件一行，每个步骤两三句话。当某个版式溢出到另一页时，脚本会失败。

## 检查

1. 先构建浅色页面（`node build.mjs --light`）。
2. 用 `--format all --png` 运行，并逐张查看 PNG。留意落在空白处的定位钉、展示错误零件的特写、被其他零件遮住的点亮零件，以及页面底部被截断的文字。
3. 以 4× 放大检查 PDF 或 HTML（对 `<name>.html?print` 使用 `drive.mjs` 的 `["shot", out, ".ps", 4, [x, y, w, h]]`）：线条必须是矢量且锐利，点亮零件在步骤静帧的尺寸下必须足够醒目。
4. 如果条件允许，实际打印一页。浅灰色块与发丝线在办公室激光打印机上必须仍然可辨。

## 图形上的 PDF 链接

`kit/pdf-button.mjs` 导出 `pdfButton(href)`：卡片右上角一个带图标的小 "PDF" 链接，自带 CSS，因此没有它的页面保持不变。把它放在板体（plate body）内的最前面，并且只在文件存在时才添加（`examples/push-button/build.mjs`）。顺序：先构建浅色页面，再打印，然后重新构建，让按钮出现。

## 更长的详图版

`detailed` 版式可排 2–4 页，由规格说明决定：

1. 总览（必有）：带钉图形、数字、每个零件配一行说明、圆形特写（`details`）。
2. 工作原理：每页六个步骤，`why` 占最后一格；多余的步骤流入下一页。
3. 部件组（assemblies，当规格说明含 `assemblies` 时）：每页最多四组，每组 `{ title, text, parts: [ids] }` 绘成一大幅点亮的特写，带各自的定位钉与零件编号。零件有十个或更多的图形给出三四组。
4. 规格说明收尾于最后一页。

`maxPages`（2–4，默认 4）限制总页数：部件组页最先被裁减，步骤数超出上限则报错。`examples/raptor-engine/sheet.mjs`（3 页）是参考范例；`push-button` 与 `arcade-cabinet` 保持 2 页。
