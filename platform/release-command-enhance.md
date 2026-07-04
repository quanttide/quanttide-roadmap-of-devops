# release 命令增强

基于自举发布 v0.8.3 的经验，提炼可封装到 CLI 内的功能。

## v1：一键预检（`release precheck`）

### 问题

发布前需要跑三个 `status` 命令然后人工逐条检查输出，容易漏看。

### 方案

新增 `release precheck` 命令，汇总各状态并给出阻断结论：

```
release precheck [scope]
  → build:   ✅ / ❌
  → test:    ✅ / ❌（含失败数和覆盖率）
  → version: ✅ / ❌（配置文件 vs tag 一致性）
  → workspace: ✅ / ⚠ 有未提交变更
  → CHANGELOG: ✅ / ⚠ 缺少条目（发布会自动生成）
  → Result:  ✅ 可发布 / ⛔ 阻断（列出具体原因）
```

### 实现

- 复用 `build::status_to`、`test::status_to`、`release::status_to` 的内部逻辑汇总到单一输出
- 不写文件，只读只判断
- 输出 exit code：0 = 可发布，1 = 阻断

## v1：发布后验证（`release verify`）

### 问题

发布后需要人工确认 tag、CHANGELOG、GitHub Release、CI 是否全部到位。

### 方案

```
release verify <version>
  → tag:       ✅ cli/v0.8.3（本地 + 远端）
  → CHANGELOG: ✅ GitHub Release body 一致
  → CI:        ✅ build-cli 已触发 (#52)
  → Result:    ✅ 发布完整
```

### 实现

- 检查本地 tag + `git ls-remote` 远端 tag
- `gh release view` 对比 CHANGELOG
- `gh run list --workflow build-cli` 检查最新运行
- 纯查逻辑，无副作用

## v2：预发布晋升（`release promote`）

### 问题

rc 版本 CI 验证通过后，发正式版是机械操作（同样的版本号去 rc 后缀）。

### 方案

```
release promote cli/v0.8.3-rc.1
  → 检查 rc tag 存在
  → 检查 CI 是否通过（build-cli / publish-cli）
  → 自动发布 cli/v0.8.3
```

### 实现

- 从版本号剥离 `-rc.N` / `-alpha.N` 后缀得到正式版号
- 调用 `publish` 内部逻辑

## v3：失败恢复（`release recover`）

### 问题

发布中途超时时人需要手动判断 tag/release 到哪一步。

### 方案

```
release recover cli/v0.8.3
  → 检查各阶段状态
  → 自动补全缺失步骤
```

### 优先级

- `release precheck` — 最高，纯查逻辑，无副作用
- `release verify` — 次高，纯查逻辑
- `release promote` — 中等，有写操作
- `release recover` — 中等，有写操作，需谨慎设计幂等性
