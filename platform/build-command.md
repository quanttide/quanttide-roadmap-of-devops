# build 命令工作蓝图

> 基于 handbook lifecycle 规划，在 plan → code → build → test → release 流程中承接"构建"环节。

## 定位

`build` 命令围绕项目的构建流程，提供构建状态查看和构建触发能力。不管理 CI 流水线本身，而是与 CI 系统交互的客户端。

## 命令设计

```
qtcloud-devops build status     查看构建状态
```

### build status

输出当前项目的构建状态，包括：

- 最后一次构建结果（成功/失败/运行中）
- 构建产物信息
- 构建耗时

不触发构建，构建由 CI 系统（GitHub Actions）自动执行。

## 输出示例

```
构建状态
────────────────────────────────────────
  最新构建:     #42 main  ✅ 通过
  触发时间:     2026-06-28 10:30
  耗时:         2m 34s
  产物:         qtcloud-devops-cli v0.6.1
```

## 待确认

- `build status` 的数据来源（gh CLI / GitHub API）
- 是否需要 `build run` 触发手动构建（当前 CI 自动触发，暂不需要）
