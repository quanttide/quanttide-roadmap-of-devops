# test 命令工作蓝图

> 在 lifecycle（plan → code → build → test → release）中承接"测试"环节。

## 定位

`test` 命令提供 `status` 只读模式，不运行测试。测试由 CI 系统（GitHub Actions）自动执行，`test status` 分析已有的测试报告。

## 命令设计

```
qtcloud-devops test status     查看测试状态和覆盖率
```

### test status

按 scope 输出测试状态：

- 测试总数 / 通过 / 失败 / 跳过
- 代码覆盖率（从覆盖率报告读取）
- 覆盖率是否达到阈值

### 数据来源

按优先级查找覆盖率报告：

1. `target/coverage/lcov.info`（Rust：cargo-llvm-cov）
2. `coverage/lcov.info`（Python：pytest-cov）
3. CI 输出的测试摘要（`gh run view`）

## 输出示例

```text
测试状态
────────────────────────────────────────
  [cli]
    测试数:       157  ✅ 全部通过
    覆盖率:       78%  （阈值 70%  ✅）
  [studio]
    测试数:       43   ✅ 42 通过 / 1 跳过
    覆盖率:       52%  （阈值 70%  ⚠ 不足）
```

## 待确认

- 覆盖率阈值的配置方式（contract.yaml？CLI 参数？）
- 是否支持多种框架的测试报告格式
- 是否需要在 CI 中集成作为门禁

## 优先级

中。依赖 scope 解析（contract）和 CI 查询（`gh run`）公共能力，等 contract 模块就绪后再实现。
