# publish 幂等性

## 问题

`release publish` 超时后，`create_release` 不幂等：

| 阶段 | 幂等 | 重跑行为 |
|------|------|---------|
| update_config_version | ✅ | 版本号相同，覆盖无影响 |
| ensure_changelog | ✅ | 版本已存在，跳过 |
| create_tag | ✅ | tag 已存在，视为成功 |
| push_tag | ⚠️ | `git push origin <tag>` 在已存在时报 non-fast-forward |
| create_release | ❌ | `gh release create` 在已存在时报错 |

## 方案

1. `push_tag` 加 `--force` 或先检查远端是否存在
2. `create_release` 先 `gh release view` 检查，已存在则跳过
