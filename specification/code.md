# Git Submodule 状态管理建模标准 v1.1

## 1. 目的

本标准为 Git Submodule 管理工具提供一套**语言无关、平台无关的核心建模规范**。任何实现者遵循本标准，即可构建出行为一致、状态判定准确、交互可预测的 Submodule 管理工具。

---

## 2. 核心概念

### 2.1 三个引用点

Submodule 管理的本质是**三个 commit 引用的一致性判定问题**。任何实现必须从父仓库和子仓库中获取以下三个值：

| 引用名 | 来源 | 含义 |
|--------|------|------|
| `parent_pointer` | 父仓库的 gitlink entry（`git ls-tree HEAD` 中子模块路径对应的 blob） | 父仓库记录的期望 commit |
| `local_head` | 子模块仓库的 `HEAD`（`git rev-parse HEAD`） | 子模块当前检出的实际 commit |
| `remote_head` | 子模块远程跟踪分支的最新 commit（`git ls-remote` 或等效查询） | 远程仓库的最新 commit |

### 2.2 基础状态集

所有 Submodule 状态必须从以下 7 种基础状态中取值。状态判定**仅依赖** 2.1 中定义的三个引用点和子模块工作区的 `git status` 结果：

| 状态 | 判定规则 |
|------|----------|
| `Clean` | `parent == local == remote`，且工作区干净 |
| `AheadOfParent` | `local == remote != parent`，且工作区干净 |
| `BehindRemote` | `parent == local != remote`，且 `remote` 是 `local` 的祖先（即远程有新增提交，本地未落后于远程的其他分支） |
| `Detached` | `local_head` 指向的 commit 不属于任何本地分支（`git branch --contains` 返回空） |
| `Dirty` | 子模块工作区存在未跟踪或已修改但未提交的变更（`git status --porcelain` 非空） |
| `Orphaned` | `parent_pointer` 指向的 commit 在远程仓库中已不存在 |
| `Uninitialized` | 子模块本地路径不存在或 `.git` 目录缺失 |

### 2.3 状态优先级

当子模块同时满足多个状态判定规则时，按以下优先级**取最高优先级的单一状态**。此优先级**直接决定 UI 中的显示顺序、颜色标注和告警级别**：

| 优先级 | 状态 | 严重级别 | 推荐颜色 |
|--------|------|----------|----------|
| 0 (最高) | `Dirty` | 错误 | 红色 |
| 1 | `Orphaned` | 错误 | 红色 |
| 2 | `Detached` | 警告 | 黄色 |
| 3 | `Uninitialized` | 警告 | 黄色 |
| 4 | `BehindRemote` | 信息 | 蓝色 |
| 5 | `AheadOfParent` | 信息 | 蓝色 |
| 6 (最低) | `Clean` | 正常 | 绿色 |

**实现约束**：同级优先级的子模块必须按名称的字母序升序排列，以保证多次扫描间 UI 的稳定性。

---

## 3. 状态判定流程

任何实现必须按以下顺序执行判定，**命中即返回**，不再继续后续判定：

```
输入: 子模块路径 P

1. 检查 P 是否存在且含 .git
   → 否 → 返回 Uninitialized

2. 获取 parent_pointer, local_head, remote_head
   获取工作区状态 dirty_flag

3. if dirty_flag == true
   → 返回 Dirty

4. if parent_pointer 在远程仓库中不存在
   → 返回 Orphaned

5. if local_head 不在任何本地分支上
   → 返回 Detached

6. if local_head != remote_head
   → 若远程不可达，跳过此步骤；若可达且远程不是本地的祖先，返回 BehindRemote

7. if parent_pointer != local_head
   → 返回 AheadOfParent

8. 返回 Clean
```

**离线策略**：步骤 4 和步骤 6 依赖远程仓库可达性。若无法连接远程仓库：
- 步骤 4（Orphaned 判定）应跳过，不将该子模块判定为 Orphaned。
- 步骤 6（BehindRemote 判定）应跳过，不进入此判定分支。
- 同时将 `remote_head` 标记为不可用，并在返回结果中标注 `remote_unreachable: true`，便于 UI 层展示状态不确定性。

---

## 4. 原子命令集

任何 Submodule 管理工具必须提供以下原子命令。每个命令对应一次不可分割的 Git 操作序列，要么全部成功，要么全部回滚。

### 4.1 查询命令（无副作用）

| 命令 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `scan_all` | 父仓库根路径 | `(Vec<Submodule>, AggregateStatus)` | 扫描全部子模块并返回状态列表和聚合统计 |
| `scan_one` | 父仓库根路径, 子模块名 | `Submodule` | 扫描单个子模块 |
| `health_check` | 父仓库根路径 | `Vec<HealthIssue>` | `scan_all` 的派生视图，等价于过滤出 `status != Clean` 的子模块并附上建议操作 |

### 4.2 变更命令（有副作用）

| 命令 | 前置条件 | Git 操作序列 | 后置状态 |
|------|----------|-------------|----------|
| `add` | URL 可访问，路径不冲突 | `git submodule add -b <branch> <url> <path>` | `Uninitialized` → `Clean` 或 `BehindRemote` |
| `init` | 状态为 `Uninitialized` | `git submodule init && git submodule update` | `Uninitialized` → `Clean` 或 `BehindRemote` |
| `update` | 状态可为任意非 `Dirty` 状态 | `git submodule update --remote --merge` | `BehindRemote` → `Clean` 或 `AheadOfParent`；`Detached` → `Detached`（指向更新的 commit，状态不变） |
| `sync_to_parent` | 状态为 `AheadOfParent` | 在父仓库中 `git add <submodule_path>` | `AheadOfParent` → `Clean` |
| `checkout_branch` | 分支名在远程存在 | 在子模块中 `git checkout <branch>` | `Detached` → `Clean` |
| `retire` | 状态为 `Clean` 或 `Orphaned` | `git submodule deinit <path> && git rm <path>`，保留归档记录 | 从活跃列表中移除 |
| `cancel_retire` | 子模块曾被 `retire`，归档记录存在 | 从归档中恢复 `.gitmodules` 条目并重新 `init` | 恢复到退役前状态 |

---

## 5. 只读模型层与可写命令层的分离

### 5.1 原则

- **模型层**负责扫描仓库、计算状态。模型层的所有公开 API 必须是**纯读取**的，不产生任何磁盘写入或 Git 操作。
- **命令层**负责执行实际的 Git 操作。命令层在执行前后必须通过模型层扫描来验证前置条件和后置结果。

### 5.2 验证闭环

```
扫描（模型层） → 判定状态 → 用户选择操作 → 执行命令（命令层） → 再次扫描（模型层） → 验证结果
```

如果命令执行后的实际状态与预期不符，命令层必须返回错误并包含详细的状态差异。

---

## 6. 异常收敛路径

所有异常状态必须存在收敛到 `Clean` 的路径。唯一的例外是 `Orphaned`：

| 当前状态 | 收敛操作 | 目标状态 | 安全级别 |
|----------|----------|----------|----------|
| `Dirty` | 用户手动提交或丢弃变更 | `AheadOfParent` 或 `Clean` | ⚠️ 谨慎 |
| `Orphaned` | **无自动收敛路径**，必须人工干预 | 人工修复后进入 `Clean` | — |
| `Detached` | `checkout_branch` | `Clean` | ⚠️ 谨慎（会丢弃 detached 指向的 commit 引用） |
| `BehindRemote` | `update` | `Clean` 或 `AheadOfParent` | ✅ 安全 |
| `AheadOfParent` | `sync_to_parent` | `Clean` | ✅ 安全 |
| `Uninitialized` | `init` | `Clean` 或 `BehindRemote` | ✅ 安全 |

**安全级别说明**：
- ✅ **安全**：不会丢失本地数据（如 `sync_to_parent`）
- ⚠️ **谨慎**：可能覆盖本地未推送的变更（如 `checkout_branch` 强制切换）
- 🚫 **危险**：会丢弃本地修改（如 `reset --hard` 类操作），本标准不提供危险级收敛操作

---

## 7. 类型定义（实现参考）

以下类型定义以 Rust 语法给出，作为实现的类型参考。其他语言实现时需保持语义等价。

```rust
/// Commit hash 必须定义为新类型，不得退化为 String。
/// 要求实现: Display (截断为 7 位短 hash), FromStr (接受完整或短 hash)。
struct CommitHash(String);

enum SubmoduleStatus {
    Clean,
    AheadOfParent,
    BehindRemote,
    Detached,
    Dirty,
    Orphaned,
    Uninitialized,
}

struct Submodule {
    name: String,
    path: PathBuf,
    url: String,
    tracked_branch: String,
    parent_pointer: CommitHash,
    local_head: CommitHash,
    remote_head: CommitHash,
    status: SubmoduleStatus,
}

struct AggregateStatus {
    total: usize,
    clean: usize,
    dirty: usize,
    orphaned: usize,
    detached: usize,
    uninitialized: usize,
    behind: usize,
    ahead: usize,
}
```

---

## 8. 合规性检查清单

任何声称符合本标准的实现，必须满足以下所有条件：

- [ ] 状态判定完全基于"三个引用点 + 工作区状态"，不引入额外判定源
- [ ] 7 种基础状态全部实现，无遗漏
- [ ] 状态优先级按本标准定义，驱动 UI 排序和颜色标注
- [ ] 所有查询操作无副作用
- [ ] 所有变更操作是原子事务，失败时回滚
- [ ] `Orphaned` 状态不提供自动收敛路径
- [ ] `CommitHash` 定义为独立类型，不与普通 String 混用
- [ ] 异常收敛路径标注了安全级别
- [ ] 离线场景下 Orphaned 和 BehindRemote 判定安全降级
- [ ] 状态判定不将 `local != remote` 简单不等判定为 BehindRemote，必须确认远程是祖先
