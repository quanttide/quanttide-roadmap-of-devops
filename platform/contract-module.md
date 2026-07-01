# 契约模块（contract.rs）提取

## 背景

`contract.yaml` 解析、scope 发现、语言检测、版本状态聚合等功能在 `publish.rs`、`status.rs` 等多个命令中重复实现，需要提取为公共模块 `src/contract.rs`。

## 四维架构

契约按四维架构设计（参见 `docs/essay/contract/index.md`）：

| 维度 | YAML 顶层 | 职责 |
|------|-----------|------|
| **Stages** | `stages` | 生命周期各阶段的配置（构建命令、测试阈值、发布前检查） |
| **Platforms** | `platforms` | 外部治理载体（代码托管、CI、制品库） |
| **Sources** | `sources` | 事实源定义（版本号读取方式、CHANGELOG 格式） |
| **Scopes** | `scopes` | 上下文边界（每个 scope 可覆盖全局设置） |

## 模型

```yaml
# .quanttide/devops/contract.yaml
stages:
  build:
    command: cargo build --release
  test:
    command: cargo test
    threshold: 80
  release:
    changelog: CHANGELOG.md
    pre_publish:
      - scripts/preflight.sh

platforms:
  source_control: github
  ci: github_actions
  artifact_registry: crates

sources:
  version:
    type: cargo
    path: Cargo.toml

scopes:
  cli:
    dir: src/cli
    language: rust
    framework: clap
    build_tool: cargo
    registry: crates
  studio:
    dir: src/studio
    language: dart
    framework: flutter
    build_tool: flutter
    registry: pubdev
    release:
      changelog: src/studio/CHANGELOG.md
```

```rust
pub struct Contract {
    pub stages: Stages,         // build / test / release
    pub platforms: Platforms,   // source_control / ci / artifact_registry
    pub sources: Sources,       // version (type + path)
    pub scopes: Vec<Scope>,     // 每个 scope 继承 + 覆盖全局
}
```

## 用法

```rust
let c = contract::load(repo_path);                    // → Contract（完整四维）
let scopes = contract::load_scopes(repo_path);         // → Vec<Scope>（简化）
let lang = contract::resolve_language(&scope, dir);    // → Language（声明优先）
let lang = contract::detect_by_files(dir);             // → Language（仅文件）
let threshold = contract::scope_test_threshold(&c, s); // → f64（scope > 全局）
```

## 向后兼容

旧格式（`scopes: { cli: src/cli }`）自动识别，使用全局默认值。

## 优先级

高。契约是核心领域概念，`build status`、`test status`、`release status`、`release publish` 都依赖它。

## 实验室验证

已在 `examples/default/src/contract.rs` 实现完整四维模型 + 38 个测试，覆盖新旧格式、四维配置、声明/自动语言检测、版本状态、阈值继承。
