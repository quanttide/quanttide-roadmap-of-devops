# code 命令观测记录

> 基于 `qtcloud-devops-cli v0.3.0` 实际使用体验整理。已修复项详见 `docs/report/platform/code-command.md`。

## 一、体验摘要

2026-05-25 对 `code status` + `code sync` 进行了全流程实测，覆盖 17 个子模块的同步操作。

## 二、观测记录

### 2.1 状态误判

`code status` 将 `domains/quanttide-devops` 标记为 **Dirty**，但实际属于 **AheadOfParent**（子模块有新提交，父仓库未记录）。

**根因分析**：状态判定逻辑混用了"子模块工作区是否干净"和"父指针是否落后"两个维度。

### 2.2 `--dry-run` 放置位置

`--dry-run` 是 `code` 级别的选项，而非子命令级别的选项。

### 2.3 同步输出冗余

同步 17 个子模块时生成 51 行日志，信息密度低。

### 2.4 失败的子模块缺乏明确提示

无法区分成功和失败。

### 2.5 remote_head 是本地缓存

三路比对中的 `remote_head` 是本地缓存的 `refs/remotes/origin/main`，不反映远程实时状态。

## 三、常见错误与状态映射

| Git 错误 | 工具状态 | 建议操作 |
|---------|---------|---------|
| Detached HEAD | `Detached` | `code sync <name>` |
| 重复子模块 | — | 去重 `.gitmodules` 后 `code sync` |
| 空仓库 | `Uninitialized` | 初始化并推送后再 `code sync` |
| 远程未配置 | `BehindRemote` 🛰 | 手动 `git remote add origin <url>` |
| 未初始化 | `Uninitialized` | `git submodule update --init --recursive` |
| 提交哈希冲突 | `Orphaned` | 手动修复父指针 |
| config 残留 | — | retire 自动清理 |
