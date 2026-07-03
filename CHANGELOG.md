# CHANGELOG

## [0.2.0] - 2026-07-02

### Added
- CLI v0.7.1 发布：代码质量与测试基础设施升级
  - git2 替换全部生产代码 `Command::new("git")`
  - PATH mock 编排层测试（21 个场景覆盖 gh/cargo/build/test/release）
  - gh mock 全覆盖（run list / release create / release view 共 14 场景）
- 测试覆盖率 87.81% 行 / 90.75% 函数（llvm-cov）
- toolkit v0.1.3 适配 + contract.rs 100% 覆盖

### Changed
- 纯函数提取（parse_gh_run_list / check_command / test_command 等 6 个）
- 契约四维架构 document: contract.yaml 格式适配 v0.1.3

## [0.1.0] - 2026-07-02

### Added
- 新增代码命令工作蓝图、发布生命周期管理蓝图、Git子模块状态管理建模标准等多项技术规划文档
- 新增平台文档、规范索引及CLI简化DevOps愿景记录
- 新增模块重构、代码同步命令等设计文档
- 新增构建命令和测试命令的路线图

### Changed
- 调整项目模块命名（project→contract），将合约模块优先级提升为高
- 将规范文档迁移至独立目录，并更新构建命令蓝图为多范围设计
- 合并合约QA测试至规范文档，将Git子模块错误合并至代码命令蓝图
- 移除已完成项，保留观察项和错误映射，标记代码命令文档状态为已完成

### Fixed
（无）

### Removed
- 移除过时的发布蓝图和Git子模块蓝图
- 移除过时的add models及已完成的代码命令文档
- 移除过时设计及重复内容
