# 平台愿景

## CLI 简化 DevOps 生命周期

`qtcloud-devops` 的目标是用命令行工具简化 DevOps 生命周期的每个环节。

```
plan     → qtcloud-devops plan       （待实现）
code     → qtcloud-devops code        ✅ status / sync
build    → qtcloud-devops build       （待实现）
test     → qtcloud-devops test        （待实现）
release  → qtcloud-devops release     ✅ status / publish
deploy   → qtcloud-devops deploy      （待实现）
operate  → qtcloud-devops operate     （待实现）
monitor  → qtcloud-devops monitor     （待实现）
```

每个阶段一页命令参考（`docs/handbook/lifecycle/`），格式统一：

```
操作前检查 → 执行操作 → 操作后验证
```

手册不讲通用知识，只展示 CLI 如何简化该环节的管理。

## 跨命令公共能力提取

以下功能在多个命令中重复实现，应提取为公共模块（`src/contract.rs`）：

| 功能 | 当前分布 | 用法 |
|------|---------|------|
| scope 解析（contract.yaml → scope→dir） | `publish.rs`、`status.rs` 各有一套 | `load_scopes(repo_path)` → `Vec<Scope>` |
| 语言检测 | 无，每个命令自处理 | `detect_language(path)` → `Language` |
| 版本状态聚合 | `release::util` 的 get_latest_tag、normalize_version | `version_status(scope)` → `VersionStatus` |

优先级：低。不影响现有功能，等第三个命令（build）实现时再做。
