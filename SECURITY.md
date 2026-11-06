# 安全

## 适用范围

How It Works 是一组指令、一套绘图 kit 和本地 Node 脚本。启动 Chrome 的那些脚本（`capture.mjs`、`drive.mjs`、`lines.mjs`、`turn-check.mjs`）以无头模式运行它，使用一次性用户配置目录并带 `--allow-file-access-from-files`，并且 `drive.mjs` 会在页面里运行你传入的 JavaScript。只把它们指向你信任的页面。

安全问题是指本仓库中任何能让某个页面、某条提示词或某个文件以用户未曾要求的方式运行代码、读取文件或访问网络的东西：可能出现在 kit 里、脚本里、构建出的示例页面里，或出现在 `SKILL.md` 与 `references/` 中那些会引导 Claude 做出不安全行为的指令里。

## 支持的版本

仅支持 `main` 上的最新提交。

## 报告漏洞

请不要为安全问题公开开 issue。请通过 GitHub 针对本仓库的 [private vulnerability reporting](https://github.com/wheresryan22/anatomy/security/advisories/new) 私下报告。如果该链接不可用，可在 X 上向 [@wheresryan22](https://x.com/wheresryan22) 发送私信请求一个私下渠道，且不要包含细节。

请写明你发现了什么、如何复现，以及它可能让他人做到什么。
