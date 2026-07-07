# plan sync — 设计方案

> 2026-07-07 · 来自巡视洞察：TODO [x] → ROADMAP [x] 缺少自动同步

## 动机

当前 TODO 标记完成 (`- [x]`) 后，ROADMAP 中对应的顶层条目仍保持 `- [ ]`，导致进度虚报（ROADMAP 0/16 完成，实际 ~4 项已完成）。需要一种机制将 TODO 的完成状态同步回 ROADMAP。

## 命令设计

```
qtcloud-devops plan sync [SCOPE]
```

### 行为

1. 读取 scope 内的 ROADMAP.md 和 TODO.md
2. 遍历 ROADMAP 条目（`- [ ]`），检查 TODO 中是否有引用该条目的已完成子项
3. 若 TODO 中对应子项全部 `[x]`，则将 ROADMAP 条目标记为 `- [x]`
4. 输出同步摘要：几项推进、几项不变、几项部分完成

### 匹配规则

TODO 条目通过标题文本匹配 ROADMAP 条目。具体方式：

| 方式 | 说明 |
|------|------|
| 精确标题匹配 | TODO 节标题与 ROADMAP 条目文本匹配（去掉 checkbox 前缀后） |
| 前缀匹配 | TODO 节标题是 ROADMAP 条目文本的前缀或语义子集 |
| 回退：LLM | 上述规则无法匹配时，调用 LLM 做语义关联 |

优先级：精确 > 前缀 > LLM。

### 推进条件

- ROADMAP 条目 → **全部**子 TODO 已完成 → 标记 `[x]`
- ROADMAP 条目 → 部分子 TODO 已完成 → 不标记，输出"部分完成"提示
- ROADMAP 条目 → 无子 TODO → 不变

### 示例

```
ROADMAP: [ ] `plan audit` 新增路径存在性、粒度达标、孤儿 ROADMAP 条目三项结构检查
TODO:    [x] src/plan.rs: 路径存在性检查
         [x] src/plan.rs: 粒度检查
         [x] src/plan.rs: 孤儿条目检查
         [x] src/plan.rs: 为不含路径的 TODO 条目自动补充文件路径
         [x] src/plan.rs: format_spec + system message 支持两种格式
→ sync 后 ROADMAP 标记 [x]
```

### 实现位置

- 新增 `plan.rs`：`pub fn sync_status(repo_path: &Path, scope: Option<&str>)` 或复用 `plan_audit` 的读取逻辑
- `main.rs`：新增 `PlanAction::Sync` 枚举变体
- LLM 回退复用 `edit_llm` 的调用机制

### 与现有命令的关系

| 命令 | 关系 |
|------|------|
| `plan audit` | sync 的输入来源：audit 暴露的不一致由 sync 修复 |
| `plan doctor/edit` | sync 只改 checkbox 状态，不改格式；doctor 负责格式 |
| `plan clean` | sync 先标记完成，clean 再删除 |

即：`plan sync` → `plan clean`，分两步走。
