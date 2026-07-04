# release recover

## 问题

`release publish` 在网络阻塞时可能超时，导致中间状态不一致：

| 阶段 | 可能完成 | 可能未完成 |
|------|---------|-----------|
| update_config_version | 版本号已修改 | — |
| ensure_changelog | CHANGELOG 已写入并 commit | — |
| create_tag | tag 已创建 | tag 未创建 |
| push_tag | tag 已推送到远端 | tag 仅在本地 |
| create_release | GitHub Release 已创建 | Release 未创建 |

超时后人需要手动检查各阶段状态并补齐缺失步骤。

## 方案

`release recover <version>` 自动检测并补全：

```
release recover cli/v0.8.3
  → tag 本地:   ✅ 存在
  → tag 远端:   ❌ 缺失 → 推送
  → GitHub Release: ❌ 缺失 → 创建
  → 恢复完成
```

### 实现要点

- 只补缺失步骤，不重复已完成的步骤（幂等）
- 不清除/回滚已有操作
- 如果 tag 都未创建则报错（这不是 recover 的职责，应重新 publish）
