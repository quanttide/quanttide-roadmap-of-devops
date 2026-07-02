# plan 命令工作蓝图

> 在 lifecycle（plan → code → build → test → release）中承接"规划"环节。

## 定位

`plan` 命令管理 scope 级规划文件（`<scope-dir>/ROADMAP.md`），由 AI 在工作流中持续读写维护。

核心价值：**每个 scope 一份自维护的规划清单**。

## 命令设计

```
qtcloud-devops plan status [scope]    查看 scope 规划进度
qtcloud-devops plan clean [scope]     删除 scope 已完成条目
qtcloud-devops plan doctor [scope]    修复 scope 格式问题
```

`scope` 省略时默认当前目录所在 scope。

### plan status

解析指定 scope 根目录下的 ROADMAP.md，按版本输出规划完成进度：

- 各版本条目总数 / 完成数 / 完成率
- 底部汇总

### plan clean

从 scope 的 ROADMAP.md 中删除所有 `[x]` 条目。

- 只删条目，保留版本标题和分类标题
- 空版本标题和空分类标题一并清理

### plan doctor

检测并修复 scope 的 ROADMAP.md 格式问题。**只修格式，不增删或修改条目内容本身**。

规则化修复（确定性场景）：
- 分类标题拼写和大小写（`### Added / Changed / Fixed / Removed / Deprecated / Security`）
- `- [x]` / `- [ ]` 格式（`[` 后保证一个空格）
- 条目缩进层级（`###` 下直接跟列表项，不多余缩进）
- 版本号格式（`## [X.Y.Z]`，不带 `v` 前缀）

LLM 修复（不确定性场景）：
- 当格式偏差超出规则化修复能力时（如分类被错误嵌入其他结构内部、分隔符和标题混淆），以当前格式约定为规范调用 LLM 修正
- **LLM 只负责格式恢复，不修改条目是否勾选、不增减条目**
- 以 dry-run diff 展示变更，确认后写入

### 数据源

- `<scope-dir>/ROADMAP.md` — per-scope 规划文件，AI 负责维护

### 完成度判定

直接解析 ROADMAP.md 中的 checkbox：

- `- [x]` → 已完成
- `- [ ]` → 未完成

## 输出示例

```text
[cli] 规划进度
────────────────────────────────────────
  [0.2.0]     — 1/4 已完成
  [0.1.0]     — 8/8 已完成
────────────────────────────────────────
  总计：9/12 已完成
```

## 数据源格式约定

```markdown
## [0.2.0]

### Added
- [x] build status：按 scope 检查构建状态
- [ ] test status：测试覆盖率检查
```

语法规则：
- `## [X.Y.Z]` — 版本号标题
- `### Added / Changed / Fixed / Removed / Deprecated / Security` — 分类标题
- `- [x] xxx` — 已完成条目
- `- [ ] xxx` — 未完成条目

## 实现要点

- scope 解析复用 contract 模块的 `find_scope_by_path()`，定位对应根目录
- ROADMAP.md 路径为 `<scope-dir>/ROADMAP.md`
- 文件不存在时 `plan status` 输出"未创建规划文件"，`plan clean` / `plan doctor` 跳过
