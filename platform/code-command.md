# code 命令工作蓝图

> 基于 `qtcloud-devops-cli v0.3.0` 实际使用体验整理

## 一、体验摘要

2026-05-25 对 `code status` + `code sync` 进行了全流程实测，覆盖 17 个子模块的同步操作。整体评价：功能完整、自动化程度高，但存在状态误判和 CLI 设计一致性两个可改进点。

## 二、观测记录

### 2.1 状态误判

`code status` 将 `domains/quanttide-devops` 标记为 **Dirty**，但实际属于 **AheadOfParent**（子模块有新提交，父仓库未记录）。

**根因分析**：状态判定逻辑混用了"子模块工作区是否干净"和"父指针是否落后"两个维度。Dirty 应仅指子模块工作区有未提交修改，而父指针落后应归入 AheadOfParent。

**影响**：用户拿到 Dirty 提示后不知道该做什么（提交还是同步），降低了工具的决策辅助价值。

### 2.2 `--dry-run` 放置位置不直觉

`--dry-run` 是 `code` 级别的选项，而非 `sync` / `status` 子命令级别的选项：

```
# 实际用法（违反直觉）
qtcloud-devops code --dry-run sync

# 直觉用法（当前不支持）
qtcloud-devops code sync --dry-run
```

**建议**：将 `--dry-run` 下放到每个子命令，或至少在 help 文本中醒目提示当前用法。

### 2.3 同步输出冗余

同步 17 个子模块时，每行输出三段式信息（已推送 + 已同步 + 已推送父仓库），共生成 51 行日志。信息密度低，滚动查看困难。

**建议**：改为单行聚合，例如：

```
assets/quanttide-platform    ✓ push · sync · push-parent
domains/quanttide-devops     ✓ push · sync · push-parent
```

### 2.4 失败的子模块缺乏明确提示

如果某个子模块推送失败，当前输出中无法区分成功和失败。建议引入颜色或状态标记：

```
assets/quanttide-platform    ✓ push · sync · push-parent
domains/quanttide-devops     ✗ push: 权限不足  · 已跳过
```

### 2.5 remote_head 是本地缓存，不反映远程实时状态

三路比对中的 `remote_head` 来自于模块本地缓存的 `refs/remotes/origin/main`，是上次 `git fetch` 时的快照，不是远程的实时状态。

这导致两个问题：
- `BehindRemote` 无法检测真正的远程更新——只有 fetch 过才知道
- 如果远程有新的 push，`code status` 完全看不到，和 `git submodule status` 没有区别

`code status` 的核心价值就在于告知远程变化，这必须实时 fetch。

**建议**：
- 默认每次执行 `status` 时先 fetch，失败时降级到本地缓存并标记 🛰
- 增加 `--offline` 参数跳过 fetch，给离线场景使用

## 三、改进方案（v0.3.x 已全部完成 ✅）

| 优先级 | 问题 | 方案 | 状态 |
|--------|------|------|------|
| P0 | 状态误判 | 修正状态判定：工作区脏才标 Dirty，父指针落后标 AheadOfParent | ✅ `src/model/code.rs` |
| P0 | remote_head 是本地缓存 | status 默认先 fetch，失败时降级到本地缓存并标记 🛰 | ✅ `src/commands/code.rs` |
| P1 | `--dry-run` 位置 | 下放到 `sync` / `retire` 各子命令 | ✅ `src/main.rs` |
| P2 | 输出冗余 | 改为单行聚合格式 | ✅ `src/commands/code.rs` |
| P3 | 失败提示 | 引入颜色和 `✓` / `✗` 状态标记 | ✅ `src/commands/code.rs` |
| P4 | 离线场景 | 增加 `--offline` 参数跳过 fetch | ✅ `src/main.rs` |

## 五、拒绝标准（已全部达标 ✅）

- [x] P0 修复后：Dirty 列表不再包含纯 AheadOfParent 的子模块
- [x] P0 修复后：status 默认执行 fetch，远程有新 push 时正确显示 BehindRemote
- [x] P1 修复后：`code sync --dry-run` 通过编译并能正常执行 dry-run
- [x] P2/P3 修复后：17 个子模块输出在 20 行以内，失败项目显式标记

## 六、常见错误与状态映射

以下 Git 原生错误对应到工具状态机中的状态和建议操作：

| Git 错误 | 工具状态 | 建议操作 |
|---------|---------|---------|
| Detached HEAD | `Detached` | `code sync <name>`（自动 checkout main 后 update）|
| 重复子模块 | — | 去重 `.gitmodules` 后 `code sync` |
| 空仓库 | `Uninitialized` | 先在远程仓库初始化并推送，再 `code sync` |
| 远程未配置 | `BehindRemote` 🛰 | 子模块缺少 remote，需手动 `git remote add origin <url>` |
| 未初始化 | `Uninitialized` | `git submodule update --init --recursive` |
| 提交哈希冲突 | `Orphaned` | 父仓库记录的 commit 在远程不存在，需手动修复父指针 |
| config 残留 | — | retire 已自动清理 config，若手动删除子模块需手动 `git config --remove-section` |
| 未指定分支 | — | 子模块默认跟踪 HEAD，不构成独立错误 |

工具的目标是消除这些错误的手动处理成本：用户只需执行 `code status` 了解状态，再执行对应的 `code` 命令即可，不需要手动查文档排查。
