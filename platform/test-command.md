# test 命令工作蓝图

> 基于 handbook lifecycle 规划，在 plan → code → build → test → release 流程中承接"测试"环节。

## 定位

`test` 命令围绕项目的测试流程，提供测试状态查看和覆盖率检测能力。不运行测试用例，而是分析已有的测试报告。

## 命令设计

```
qtcloud-devops test status     查看测试状态和覆盖率
```

### test status

输出当前项目的测试状态，包括：

- 测试总数 / 通过 / 失败 / 跳过
- 代码覆盖率（从 lcov.info 等覆盖率报告读取）
- 覆盖率是否达到阈值

不运行测试，测试由 CI 系统自动执行。

## 数据来源

按优先级查找覆盖率报告：

1. `target/coverage/lcov.info`（Rust：cargo-llvm-cov）
2. `coverage/lcov.info`（Python：pytest-cov）
3. 其他格式待扩展

## 输出示例

```
测试状态
────────────────────────────────────────
  测试数:       157 ✅ 全部通过
  覆盖率:       78%
  最低要求:     70%
  状态:         ✅ 通过
```

## 待确认

- 覆盖率阈值的配置方式（硬编码？配置文件？CLI 参数？）
- 是否支持多种测试框架的输出格式
- 是否需要在 CI 中集成（作为门禁）
