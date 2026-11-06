# 更新日志

本项目的重要变更记录在此。格式遵循 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)，版本遵循 [Semantic Versioning](https://semver.org/spec/v2.0.0.html)。

## [0.1.0] - 2026-11-06

### 新增

- **初始化。** 技能以 `how-it-work` 之名建立：`SKILL.md`（工作流、环境与工艺规则）、`kit/` 绘图库、`references/` 十篇指南、`scripts/` 无头 Chrome 验证工具、八个完整示例与回归测试。
- 全部文档中文化：README、SKILL.md、references、本文件、CONTRIBUTING、CODE_OF_CONDUCT、SECURITY、issue/PR 模板与 evals（JSON/YAML 均通过合法性自检）。
- 许可证与署名：MIT，版权 `steven0lisa <steven0lisa@gmail.com>`。

### 变更

- 技能名与调用名为 `how-it-work`：`name: how-it-work`、安装路径 `skills/how-it-work`、调用 `/how-it-work`（Codex 为 `$how-it-work`）。
- 示例街机柜的跑马灯与演示画面文字为 `HOW IT WORKS`，全部 17 个示例构建页已随源码重建并提交。

### 验证

- 回归测试 290/290 通过；八个示例双主题（含 3D 校验与压力档）全部构建成功；敏感信息扫描 CLEAN。
