# code status v2 路线图

> 2026-07-16 · 来自架构师视角评估：7 个缺口 + 9 个架构缺陷分析

## 背景

当前 `qtcloud-devops code status` 只做一件事——检查子模块同步状态（Synced/PendingPush/PendingPull/Conflict）。架构师视角评估暴露了 7 个缺口，其中最核心的是 **CLI 知道项目有哪些模块（scope），但不知道模块之间的关系（依赖）和约束（规则）**。

`code status v2` 将其升级为"代码架构健康状态"视图，包含 4 个 section：同步状态、模块依赖图、跨 scope 一致性、质量趋势。

实现这个设计会暴露当前架构的 9 个缺陷，主要是：**当前架构是按"单一职责命令"设计的，但 v2 是一个"聚合视图"**。

## 双轨策略

### 重构轨道：修复现有架构缺陷，为 v2 铺路

这些重构是有预见性的基础设施投资——即使不做 v2，它们也提高了代码质量。

| 优先级 | 任务 | 工作量 | 缺陷关联 | 说明 |
|--------|------|-------|---------|------|
| P0 | ⬜ 新增 `source::manifest` 模块封装 `cargo metadata` | 中 | #1 #7 | 提供 `Manifest::from_path`、`Workspace::from_metadata`、依赖对比 API |
| P0 | ⬜ 统一 `code/` 错误类型为 `CodeError`（thiserror） | 小 | #5 | 替换 `Box<dyn Error>`，定义 3-5 个变体 |
| P1 | ⬜ Scope 迭代器共享化 `ScopeIter` | 中 | #2 | `for_each_scope()` + `ScopeContext` 预加载 manifest，替换 6 处重复 |
| P1 | ⬜ `code/status.rs` 改为 `status_to(writer)` 模式 | 小 | #3 | 将 `print_report` 从 `main.rs` 移入 `code/status.rs` |
| P2 | ⬜ 新增 `persist` 模块（审计快照存储） | 小 | #4 | 追加式 JSON 行文件，存储位置决策 |
| P2 | ⬜ contract 增加查询方法 | 中 | #6 | `scopes_by_language()`、`shared_dependencies()` 等 |
| P3 | ⬜ 引入 `PartialResult<T>` 模式处理部分失败 | 小 | #8 | 每个 section 独立可用状态 |
| P3 | ⬜ 树形/表格输出基础设施 | 小 | #9 | 依赖 `prettytable-rs` 或手写缩进渲染 |

### 新增轨道：v2 section 实现

按依赖顺序推进，每个 section 依赖前面的重构完成。

| 优先级 | 任务 | 前置重构 | 说明 |
|--------|------|---------|------|
| P0 | ⬜ `SyncSection` 保持不变 | — | 现有子模块同步状态逻辑不变 |
| P1 | ⬜ `DepsSection` — 模块依赖图 | #1 #5 #2 | 解析 `cargo metadata` → 构建有向图 → 循环检测 → 跨 scope 引用 |
| P2 | ⬜ `ConsistencySection` — 跨 scope 一致性 | #2 #6 | Rust 版本、公共依赖版本、CI 模板、lint 配置一致性对比 |
| P3 | ⬜ `HealthSection` — 质量趋势 | #4 | audit 结果快照持久化 + 趋势方向计算 |

## 分界线原则

| 判定条件 | 归属 |
|---------|------|
| 只改 `code/status.rs` 内部实现，不新增外部依赖 | 重构轨道 |
| 需要新增 crate 依赖（如 `cargo_metadata`） | 重构轨道（#1） |
| 需要定义新的 CLI 参数（`--section`、`-v`、`--json`） | 新增轨道 |
| 需要设计新的数据持久化格式 | 新增轨道（#4 的 `HealthSection`） |
| 依赖分析需要理解单仓的 workspace 结构 | 新增轨道的 `DepsSection` |
| scope 版本/配置对比是确定性计算，无外部依赖 | 新增轨道的 `ConsistencySection` |
| 输出格式优化不影响功能正确性 | 可延后到 P3 |

## 命令接口（终态）

```bash
# 基础用法（默认输出概要）
qtcloud-devops code status

# 完整信息（展开所有 section）
qtcloud-devops code status -v

# 仅查看依赖图
qtcloud-devops code status --section deps

# JSON 输出（供 Agent 消费）
qtcloud-devops code status --json
```

## 相关文档

| 文档 | 位置 |
|------|------|
| v2 设计方案 | `data/context/architecture/cli-code-status-v2.md` |
| 9 个架构缺陷分析 | `data/context/architecture/cli-code-status-v2-defects.md` |
| 架构师视角评估 | `data/context/architecture/cli-architect-review.md` |
