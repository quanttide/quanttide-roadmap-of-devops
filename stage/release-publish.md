# release publish 改进路线图

> 2026-07-07 · 来自 v0.10.0 正式版发布复盘

## 背景

v0.10.0 发布时，CHANGELOG 自动生成失败——`collect_git_log` 以 rc.2 为基准，无新提交，报错退出。所有 alpha→beta→rc 周期的变更未能自动汇总，需手动补写。

## 路线图

### [P0] 阶段晋级 CHANGELOG 占位 ✅ 已完成

**问题**：`collect_git_log` 返回 `NoNewCommits` 时 `release publish` 报错退出，无法创建 tag 和 release。

**解决**：`src/release/changelog.rs` 中 `ensure_changelog` 捕获 `NoNewCommits` 错误，自动生成占位 CHANGELOG 条目并继续发布。

### [P1] 正式版 CHANGELOG 从预发布条目汇总

**问题**：正式版发布时（rc.2 → 0.10.0），CHANGELOG 应总结整个预发布周期的所有变更，而非与上一 tag 比较。当前 `get_latest_tag` 返回最新任意 tag（rc.2），不是最新正式版 tag（v0.9.3）。

**方案**：

1. **`get_comparison_tag`**：新增函数，正式版发布时返回上一个正式版 tag 而非最新预发布 tag
   - 入参：目标版本、全部 tag 列表
   - 逻辑：若目标版本无 prerelease 后缀，找到最后一个 base 版本不同的正式版 tag
   - 返回值：作为 git log 比较基准的 tag

2. **`collect_prerelease_entries`**：新增函数，读取 CHANGELOG 中目标版本的所有预发布条目
   - 读取 `[0.10.0-alpha.1]` 到 `[0.10.0-rc.2]` 的所有条目内容
   - 返回合并文本供 LLM 处理

3. **`llm_summarize_prerelease`**：新增 LLM 调用，将预发布条目合并去重
   - prompt：给定多个预发布版本的 CHANGELOG 条目，按 Added/Changed/Fixed/Removed 分类合并
   - 去除重复条目、合并同类项
   - 输出结构化 CHANGELOG 条目内容

### [P2] 发布后自动更新父仓库子模组指针

**问题**：release publish 完成后，父仓库的子模组指针未更新，需手动 `git add` + `git commit`。

**方案**：`execute_release` 中增加步骤——检测是否在子模块内，若是则自动更新父仓库的 submodule pointer 并提交。

### [P3] release audit 跳过 CHANGELOG 检查（有占位逻辑时）

**问题**：audit 要求 CHANGELOG 包含目标版本记录，但 publish 的占位逻辑在 audit 之后执行，导致 audit 始终报错。

**方案**：audit 在检测到目标版本为正式版且最新 tag 是预发布 tag 时，允许 CHANGELOG 缺失（由 publish 后续补充）。

## 优先级

| 优先级 | 任务 | 工作量估计 | 依赖 |
|--------|------|-----------|------|
| P0 | ✅ 阶段晋级 CHANGELOG 占位 | 已完 | 无 |
| P1 | 🔲 正式版 CHANGELOG 从预发布条目汇总 | 中（3-5 函数） | 无 |
| P2 | 🔲 发布后自动更新父仓库子模组指针 | 小（1-2 函数） | P1 |
| P3 | 🔲 release audit 跳过 CHANGELOG 检查 | 小（条件判断） | P1 |
