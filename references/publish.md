# 将图形发布到社区画廊（community gallery）

社区画廊是仓库 `wheresryan22/anatomy-community`。人们在其中分享纯 HTML 图形（figure）：一个把独立页面写在自身旁边的构建脚本，正如本技能的示例那样，主题不限。React 和 Next.js 图形无法提交。

仅当用户要求分享或发布图形时才发布。未经用户明确同意，绝不发送任何内容。

## 步骤

1. **检查图形已完成。** 页面已构建且为最新，行数检查（line check）通过，若图形含管道、线缆或圆形部件则 `--audit` 通过，并且在开启减少动态效果（reduced motion）时保持一个静止姿态。画廊也会检查所有这些项。
2. **向用户询问只有他们能提供的信息：**
   - 他们的 GitHub 用户名（即作者）；`gh api user --jq .login` 可显示 GitHub CLI 当前登录的账号；
   - 一个标题（最多 80 个字符）和一个简短的 kebab-case id；
   - 许可证：MIT（默认）或 CC-BY-4.0；
   - 确认这是他们自己的作品，且愿意分享。
   prompt（提示词）就是生成该图形的那次请求；直接复用它，截断到 600 个字符以内。
3. **获取画廊的本地克隆（clone），**如果用户还没有：`gh repo clone wheresryan22/anatomy-community -- --single-branch`（只取 `main`：previews 分支包含所有图片，体积会越来越大）。克隆主仓库没有问题：提交步骤会在需要时自动为他们创建 fork。
4. **展示计划，**不做任何实际操作：
   ```sh
   node <skill>/scripts/publish.mjs <figure folder> --into <clone> --id <kebab-id> \
     --title "..." --author <github user> --prompt "..." \
     [--agent "Claude Code"|OpenCode|Codex|Antigravity|Other] --model "..." [--minutes 45] [--tokens 1.2M] \
     [--license MIT|CC-BY-4.0] \
     [--tags a,b] [--audit] [--pages a.html,b.html] --submit --dry-run
   ```
   用 `--agent` 传入你运行所在的 agent，用 `--model` 传入你所用的模型（画廊要求提供，如 `claude-opus-5-5` 或 `gpt-5-codex`）。在 Claude Code 中，制作耗时与 token 消耗会自动从本会话日志读取：从生成图形的请求到发布请求为止，跳过超过 10 分钟的空闲间隔，缓存读取不计入。在其他 agent 中，如果你能看到本会话的用量，就传 `--minutes` 和 `--tokens`；否则宁可省略也不要猜测。把计划复述给用户：它会写明 fork（如有）、分支、新增的文件夹以及 pull request 提交到哪里。
5. **仅在用户同意之后，**用 `--yes` 替代 `--dry-run` 运行同一命令。它只把图形的源码复制到 `figures/<author>/<id>/src/`，将其 kit 导入指向克隆中的 `.skill/` 链接，记录 kit 指纹，检查构建，把预览保存到 `shots/<author>-<id>/preview.png`，然后仅提交该文件夹、推送并创建 pull request。把它打印出来的链接交给用户。
   - 如果提示某个文件引用了图形文件夹和技能之外的内容，就把该文件所需的部分复制进图形文件夹，重新构建，再发布一次。
   - 如果无法判断哪些页面属于该图形，就传 `--pages`。
   - 之后要发送更新版本，带 `--force` 再运行一次；它会更新同一个 pull request。
6. **不带 `--submit` 时**脚本会在准备好文件夹后停止，改为打印 git 和 `gh` 命令；如果用户想自己发送，就用这种方式。

没有终端的人也可以通过画廊的 **Submit a figure** issue 表单提交，附上一个 GitHub Gist 链接（其中存放图形源码）。

## 画廊拒绝的内容

- 网络调用或外部地址；
- `eval` 或 `new Function`；
- 使用 frame、cookie，或触及图形之外的页面；
- 构建导入除技能、图形自身文件以及 `node:fs`、`node:path`、`node:url`、`node:crypto` 或 `node:process` 之外的模块；
- 读取环境变量；
- `src/` 下源码文本之外的文件，或把构建后的页面提交进仓库；
- 构建时写出了页面之外的任何东西，或两次运行结果不一致；
- 在开启减少动态效果（reduced motion）时仍持续移动的图形。

如果用户打算分享图形，从一开始就按这些要求来写。
