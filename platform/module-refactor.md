# 模块重构工作蓝图

> 将 CLI 内的 model / git / code 拆分为独立 crate。

## 背景

`qtcloud-devops-cli` 当前所有代码在单一的 `src/` 下。随着功能增加（code、release、build、test），模型定义、git 操作、业务逻辑混在一起，导致：

- `status` 的 fetch 作用域错误（见 code-sync-command.md）
- 状态枚举混杂交织（Synchronized / OutOfSync / Anomaly 未区分）
- 无法独立测试底层逻辑

## 目标结构

```
cli/
├── Cargo.toml              # workspace 清单
├── crates/
│   ├── model/              # 纯数据模型，无 I/O 依赖
│   ├── git/                # 底层 git 操作封装
│   ├── code/               # 子模块管理业务逻辑
│   └── release/            # 发布管理业务逻辑
└── src/                    # CLI 主程序，仅参数解析与分发
```

## 分层依赖

```
model ← git ← code / release ← src (CLI)
```

- **model**：`CommitHash`、`SubmoduleStatus`（三分法）、`Submodule`、`RepoState`、`AggregateStatus` 等纯数据结构。**绝不引入 git2 等 I/O 依赖。**
- **git**：封装所有 git2 / 外部命令调用，返回 model 类型。修复 fetch 作用域问题。
- **code / release**：协调 model 和 git，实现业务逻辑（sync 安全守卫、发布状态机等）。
- **src**：CLI 入口，clap 参数解析，分发到各 crate。

## 优先级

低。满足以下任一条件时推进：

1. 第二个 Rust 二进制项目出现（TUI、agent 等）需要复用 model
2. code 命令的三分法重构实施，需要明确的模块边界
3. contract 模块实施后，需要为 build / test 命令提供公共能力

## 参考

- `data/roadmap/platform/code-sync-command.md` — 三分法状态重构
- `data/roadmap/platform/contract-module.md` — 公共能力提取
