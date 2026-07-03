## 问题描述

`qtcloud-devops plan` 命令的 `status` 和 `doctor` 子命令行为与预期差距较大。

## 具体问题

### 1. `plan doctor` 格式检查太弱（v0.8.2 已修复）

当前只检查三类格式细节：
- 版本头 v 前缀（`## [vX.Y.Z]` → `## [X.Y.Z]`）
- 分类标题大小写（`### added` → `### Added`）
- checkbox 空格（`-  [x]` → `- [x]`）

对于完全非标准的格式（如 `## P0 — 阻塞/错误` 作为版本头）既不报错也不修复，直接返回 `✅ 格式无误`。

**v0.8.2 已修复**：`doctor` 接入 LLM，规则修不了的交 LLM 处理。

### 2. `plan status` 对格式问题静默失败（待修复）

当 ROADMAP.md 使用非标准版本头（如 `## P0 — 阻塞`）时，`status` 直接返回 `未找到规划条目`，不说明原因。

### 3. `plan doctor` 应能整理格式（v0.8.2 已修复）

当前 `doctor` 只修复细节，不识别和整理整体结构。

## 实际用例

用户有以下 ROADMAP 内容：

```markdown
## P0 — 阻塞 / 错误

### 0.1 scripts/run_match.cmd 指向已删除的文件
- 引用 main.py（v0.5.0 已移至 src/__main__.py）

## P1 — 重要

### 1.1 提取器候选填充逻辑三重复制
```

执行 `plan doctor` → `✅ 格式无误`
执行 `plan status` → `未找到规划条目`

两个命令各自静默忽略，用户无法得知哪里出了问题。

## 建议

### `plan status` 增强（待实现 → v0.8.3）

- `parse_roadmap` 返回空时检测文件是否有 `##` 行
  - 有 `##` 行但无标准版本头 → 输出 warning，标明行号和内容
  - LLM 已配置时：调用 LLM 尝试将非标准格式转换为标准版本头
  - LLM 未配置时：提示用户运行 `plan doctor`

### `plan doctor` 增强（v0.8.2 已完成）

- ✅ 检测所有未匹配 `## [X.Y.Z]` 格式的 `##` 行，列为诊断问题
- ✅ 检测非标准分类行（非 Added/Changed/Fixed/Removed/Deprecated/Security）
- ✅ 规则修复 + LLM 修复双通道
- ⚠️ 非标准版本头 LLM 转换（依赖 LLM 配置）

### 格式规范文档化

- 在 `plan doctor --help` 和错误信息中输出标准格式说明
