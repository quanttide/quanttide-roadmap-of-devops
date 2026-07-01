# 契约模块（contract.rs）提取

## 背景

`contract.yaml` 解析、scope 发现、语言检测、版本状态聚合等功能在 `publish.rs`、`status.rs` 等多个命令中重复实现，需要提取为公共模块 `src/contract.rs`。

## 职责

契约承载从四维架构（Stages / Platforms / Sources / Scopes）到代码的映射。每个 scope 声明：

| 维度 | 表示 | 说明 |
|------|------|------|
| **Scopes** | `scopes.<name>.dir` | scope 对应的子目录 |
| **Sources** | `language` / `framework` | 编程语言与框架，版本事实源的读取方式 |
| **Platforms** | `registry` | 制品发布目标（crates.io / PyPI / pub.dev / npm / GitHub Releases / Docker） |
| **Stages** | `release.changelog` / `release.pre_publish` | 发布阶段的配置项 |

## 模型

```yaml
# .quanttide/devops/contract.yaml
scopes:
  cli:
    dir: src/cli
    language: rust
    framework: clap
    build_tool: cargo
    registry: crates
    release:
      changelog: CHANGELOG.md
      pre_publish:
        - scripts/preflight.sh
```

```rust
pub struct Scope {
    pub name: String,           // scope 名称（= tag 前缀）
    pub dir: String,            // scope 子目录
    pub language: Language,     // Rust / Python / Go / Dart / TypeScript
    pub framework: String,      // 框架名（clap / axum / flutter / gin）
    pub build_tool: BuildTool,  // Cargo / Uv / Go / Flutter / Npm
    pub registry: Registry,     // Crates / PyPI / PubDev / Npm / GitHubReleases / Docker
    pub release: ReleaseConfig, // CHANGELOG 路径、发布前 hook
}
```

## 用法

```rust
let scopes = contract::load_scopes(repo_path);       // → Vec<Scope>
let lang = contract::resolve_language(&scope, dir);   // → Language（声明优先，fallback 检测）
let lang = contract::detect_language_by_files(dir);   // → Language（仅文件检测）
let status = contract::version_status(&scope);        // → VersionStatus
```

## 向后兼容

旧格式（`scopes: { cli: src/cli }`）通过 `fallback_parse` 支持，自动检测语言。

## 优先级

高。契约是核心领域概念，`build status`、`test status`、`release status`、`release publish` 都依赖它。

## 实验室验证

已在 `examples/default/src/contract.rs` 实现完整模型 + 18 个测试，两种格式（新旧）解析、语言/框架/制品库/发布配置全覆盖。
