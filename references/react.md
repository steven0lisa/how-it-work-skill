# React 与 Next.js

## 单个图形的文件布局
```
components/iso/                 the kit: iso-kit.ts, iso.css, react/draw.tsx, react/live.tsx
components/figures/rig/
  model.ts       pure model: true numbers, springs, readout text (no DOM)
  view.ts        world layout constants, the projection P (fitProjection), helpers like at(point)
  geometry.ts    every static path, computed once at module load
  frame.ts       paths for the parts that change shape: frameOf(state) -> { glass, coils, needle, ... }
  art.tsx        server component: static parts as JSX constants, passed to the live component
  live.tsx       "use client": refs, loop, pointer and keyboard, aria, writes the readout
  rig.css        scoped tokens and styles (e.g. .rig { … })
  figure.tsx     <Plate fig title hint readout keys caption><RigArt /></Plate>
```
`examples/test-rig/` 遵循此布局。它使用 `rig.ts`、`view.ts`、`geometry.ts`、`pose.ts`、`art.tsx`、`live.tsx` 和 `figure.tsx`。

## 服务端/客户端拆分
- 几何计算是纯 TypeScript，因此在服务端运行，客户端收到的是算好的路径字符串。除非某个运动部件需要每帧重建，否则不会有任何几何代码交付到客户端。需要时 `frame.ts` 由 `live.tsx` 引入并交付给客户端。
- `art.tsx` 把静态部件构建为 JSX（`<Solid paths={BASE_ART.slab} tone="mid" />` 等），并作为元素 props（如 `base`、`rail`、`carriage`、`dial`）传给 `"use client"` 的 live 组件。live 组件按画者序（painter's order）摆放它们，并把运动的那些包进 `<g ref>`。
- 只有 live 组件和 `kit/react/live.tsx` 是客户端代码。`draw.tsx` 可安全地在服务端使用。

## 大页面的 key 警告陷阱
在大页面上服务端载荷（payload）很大，React 会把一些元素 props 作为延迟（lazy）分块流式传输。客户端在兄弟节点之间渲染 `{props.dial}`。React 无法把一个 lazy 标记为已校验过 key，因此一旦该 lazy 解析为不带 key 的元素，React 就会记录 "Each child in a list should have a unique key"，并指向你的 live 组件或 `IsoFigure`。同一个图形放在小测试页上则什么都不会记录，所以一次只查一个图形会漏掉它。

有两种修复办法，应当从一开始就同时采用：

1. **给每个元素 prop 的根节点一个稳定 key：** `const DIAL = (<g key="dial">…</g>)`，fragment 则用 `<React.Fragment key="sled">`。给单个元素加 key 没有任何开销。
2. **不要把 `"use client"` 组件放进服务端构建的画面。** 服务端树里的每个客户端组件（client component）元素都会变成 lazy 引用，连带使其所有祖先也变成 lazy。如果画面中有部分需要交互（例如悬停高亮），把它们做成普通的 `<g data-part="…">`，在 live 组件里用事件委托处理。

用 `.map` 构建的数组照常需要 key。

## 样式
- `iso.css` 只导入一次：全局导入或在图形内导入均可。在外层包裹元素（或 `<body>`）上放 `className="iso"` 以获得设计令牌（token），浅色主题再加 `data-theme="light"`。
- 图形专属令牌限定在该图形的类里（`.rig { --rig-glass-top: … }`），仅当卡片需要时才在 `.iso-plate` 内覆盖套件令牌。
- 不要把图形令牌放到 `:root` 上。

## 读数
`useReadout()` 从其 `anchor` 出发查找最近的 `.iso-plate [data-readout]`，`say(text)` 写入文本而不触发重渲染。在服务端渲染静息文本，使其在水合（hydration）之前就是正确的。

## 水合
在服务端和客户端首次渲染时都渲染静息姿态；只在 effect 中启动运动。渲染期间不要读取 `window`。`asksForStillness()` 是安全的，因为它只在 effect 和循环内被调用。
