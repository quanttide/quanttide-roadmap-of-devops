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

## 三、改进方案

| 优先级 | 问题 | 方案 | 涉及文件 |
|--------|------|------|---------|
| P0 | 状态误判 | 修正状态判定：工作区脏才标 Dirty，父指针落后标 AheadOfParent | `src/cli/code/status.rs` |
| P0 | remote_head 是本地缓存 | status 默认先 fetch，失败时降级到本地缓存并标记 🛰 | `src/cli/code/status.rs` |
| P1 | `--dry-run` 位置 | 下放到 `sync` / `status` / `retire` 各子命令 | `src/cli/code/mod.rs` |
| P2 | 输出冗余 | 改为单行聚合格式 | `src/cli/code/sync.rs` 输出逻辑 |
| P3 | 失败提示 | 引入颜色和 `✓` / `✗` 状态前缀 | `src/cli/code/sync.rs` 输出逻辑 |
| P4 | 离线场景 | 增加 `--offline` 参数跳过 fetch | `src/cli/code/status.rs` |

## 四、影响范围

- `code status` 状态判定逻辑（P0 修复可能影响统计聚合）
- `code sync` 输出格式（P2/P3 仅视觉改动，不影响功能）
- `code --help` / `code sync --help` 帮助文本（P1 调整）
- 各子命令的 test 用例需同步更新

## 五、拒绝标准

- [ ] P0 修复后：Dirty 列表不再包含纯 AheadOfParent 的子模块
- [ ] P0 修复后：status 默认执行 fetch，远程有新 push 时正确显示 BehindRemote
- [ ] P1 修复后：`code sync --dry-run` 通过编译并能正常执行 dry-run
- [ ] P2/P3 修复后：17 个子模块输出在 20 行以内，失败项目显式标记
