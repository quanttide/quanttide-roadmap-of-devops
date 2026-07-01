# build 命令工作蓝图

> 基于 handbook lifecycle 规划，在 lifecycle（plan → code → build → test → release）中承接"构建"环节。

## 定位

`build` 命令只提供 `status` 只读模式，不做 `build run`。本地构建由各语言自己的命令负责，CI 构建由 GitHub Actions 自动触发。

核心价值：**按 scope 检查构建状态**，包括 CI 结果、本地语法校验、版本一致性。

## 命令设计

```
qtcloud-devops build status      按 scope 查看构建状态
```

### build status

按 `contract.yaml` 定义的作用域列表逐个检查。每个 scope 做三件事：

#### 1. 语言检测与语法校验

| 检测到 | 项目类型 | 校验命令 |
|--------|---------|----------|
| `Cargo.toml` | Rust | `cargo check` |
| `pyproject.toml` | Python | `uv pip compile pyproject.toml` 或语法检查 |
| `go.mod` | Go | `go vet` |
| `pubspec.yaml` | Dart/Flutter | `dart analyze` |

#### 2. CI 构建状态

通过 `gh run list --workflow <scope-workflow> --limit 1` 获取最近一次 CI 结果，展示：
- 构建编号
- 触发分支
- 状态（成功/失败/运行中）
- 触发时间、耗时

#### 3. 版本一致性

复用 `release status` 现有逻辑：检查 scope 下配置文件的版本号是否与最新 git tag 一致。

### 不做的

- 不主动触发构建（CI 自动执行）
- 不做全量编译（`cargo check` 而非 `cargo build`）

## 输出示例

```text
构建状态
────────────────────────────────────────
  [cli]         Rust
    CI:         ✅ 通过 (main #42)
    syntax:     ✅ cargo check 通过
    version:    ✅ 0.6.1

  [studio]      Dart
    CI:         ✅ 通过 (main #18)
    syntax:     ⚠️ dart analyze 有 3 个 warning
    version:    ✅ 0.1.3

  [provider]    Go
    CI:         ✅ 通过 (main #7)
    syntax:     ✅ go vet 通过
    version:    ⚠️ tag v0.2.0 ≠ 配置 v0.2.1

  工作区:       ✅ 干净
```

## 实现要点

- 数据来源：`gh run list` / `gh run view`（GitHub CLI），`cargo check` / `go vet` 等本地命令
- scope 发现：复用 `release status` 的 `contract.yaml` 解析逻辑
- 语言检测：通过项目根目录下关键配置文件判断
- 版本一致性：复用 `util::get_latest_tag` 和 `normalize_version`

## 优先级

1. scope 发现 + 语言检测（已有 `release status` 的 scope 解析可复用）
2. 本地语法校验（各语言调对应命令）
3. CI 状态查询（`gh run list`）
4. 版本一致性检查（复用 `release status` 逻辑）
