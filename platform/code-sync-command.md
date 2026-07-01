# code 命令工作蓝图

> 在 lifecycle（plan → code → build → test → release）中承接"代码管理"环节。

## 定位

`code` 命令管理 Git 子模块的状态检测和同步。当前 `status` / `sync` 已实现，以下记录已知问题与重构计划。

## 命令设计

```
qtcloud-devops code status     查看子模块状态
qtcloud-devops code sync       同步子模块指针
```

### 现状

- `status`：检测 7 种子模块状态，已实现按 scope 输出
- `sync`：fetch → rebase → push 子模块 → 更新父指针 → push 父仓库

### 已知问题

#### 1. 状态混杂

`SubmoduleStatus` 当前是平面枚举，混合了"可自动修复"和"需人工介入"两种维度。

目标：重构为三分法：

```
Synchronized       ✅ 理想状态
├── OutOfSync      🔄 可自动修复（ahead / behind / dirty）
└── Anomaly        ⚠️ 需人看（detached / orphaned / missing）
```

#### 2. fetch 作用域问题

`status` 的 fetch 只拉取了父仓库，子模块的 remote refs 未被更新，导致 BehindRemote 状态误判。

修复：对每个子模块单独 fetch。

#### 3. sync 缺少安全守卫

`sync` 不检查当前状态直接执行，应在同步前检查状态（Dirty 跳过，Anomaly 中止）。

## 优先级

| 项 | 优先级 | 说明 |
|----|--------|------|
| 状态重构（三分法） | 高 | 影响 status 和 sync 两个命令的正确性 |
| fetch 作用域修复 | 高 | 修复 BehindRemote 误判 |
| sync 安全守卫 | 中 | 依赖状态重构完成后推进 |

## 参考

- `data/roadmap/specification/code-sync-status.md` — 状态机详细设计
- `data/roadmap/platform/module-refactor.md` — 模块拆分方案
