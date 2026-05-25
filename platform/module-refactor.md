模块重构结论

基于我们刚才的讨论，总结如下：

项目结构（多语言 monorepo）

```
qtcloud-devops/
├── cli/            # Rust 工具箱（独立 workspace）
├── provider/       # Go 服务
├── studio/         # Dart/Flutter 应用
├── docs/           # 文档
└── ...
```

cli/ 内部 Rust workspace 规划

```
cli/
├── Cargo.toml              # workspace 清单
├── crates/
│   ├── model/              # 纯数据模型（已决定立即抽取）
│   ├── git/                # 底层 git 操作（计划抽取，修复 fetch 作用域等）
│   ├── code/               # DevOps 原语：sync、status、repair（当前逻辑暂留 src，稳定后抽）
│   └── release/            # 未来：发版流程
└── src/                    # CLI 主程序，仅参数解析与分发
```

关键原则： 按 DevOps 原语（git, code, release）拆分 crate，各 crate 只依赖其下层（model ← git ← code），主程序极薄。

立即行动

1. 创建 cli/crates/model
   · 搬入 CommitHash, SubmoduleStatus（重构为三分法：Synchronized/OutOfSync/Anomaly）、Submodule, RepoState, AggregateStatus 等纯数据结构。
   · 所有相关纯函数（如 priority()）归入 model。
   · 测试随模型移动（纯数据测试进 model，集成测试留在原处）。
2. 更新 CLI 依赖
   · cli/Cargo.toml 内 workspace members 包含 crates/model。
   · 修改 use 路径，describe_issue 等匹配新枚举变体。
3. cli/crates/git 待建
   · 等 model 稳定后，将 scan()、fetch/push 等底层逻辑搬入，修复已知的 fetch 作用域问题。

为什么不现在建 quanttide-devops 独立库

· 当前只有一个 Rust 消费者（CLI），过早抽取会冻结接口、阻碍迭代。
· 触发条件：出现第二个 Rust 二进制项目（如 TUI、agent）需要复用 model 和 git 时，再独立出来。
· 当前只需保持 model/git 无产品特定依赖，即可保证未来抽出成本极低。

设计约束

· model 绝不引入 git2 等 I/O 依赖，只放定义。
· git 封装所有 git2/外部命令调用，返回 model 类型。
· code 协调 model 和 git，实现业务逻辑（如自动同步的安全守卫）。

—

这样，我们既解决了眼前状态混乱和代码质量的问题，又为未来的演进留足了空间。
