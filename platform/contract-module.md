# 契约模块（contract.rs）提取

## 背景

`contract.yaml` 解析、scope 发现、语言检测、版本状态聚合等功能在 `publish.rs`、`status.rs` 等多个命令中重复实现，需要提取为公共模块 `src/contract.rs`。

## 职责

- **scope 解析**：读取 `contract.yaml`，返回 scope→目录映射
- **语言检测**：根据目录下的配置文件（`Cargo.toml` / `pyproject.toml` / `go.mod` / `pubspec.yaml`）判断项目类型
- **版本状态聚合**：获取 scope 对应的最新 tag、配置文件版本号、CHANGELOG 状态等

## 用法

```rust
let scopes = contract::load_scopes(repo_path)?;    // → Vec<Scope>
let lang = contract::detect_language(&scope_dir);   // → Language
let status = contract::version_status(&scope)?;     // → VersionStatus
```

## 优先级

高。契约（Contract）是核心领域概念，scope 解析、语言检测、版本状态聚合都属于契约的职责。在 build 命令实现前先提取，后续 `build status`、`release status`、`release publish` 直接复用。
