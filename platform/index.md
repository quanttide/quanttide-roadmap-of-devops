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
