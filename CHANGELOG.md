# CHANGELOG

## [0.2.0] - 2026-07-07

### Added
- `stage/release-publish.md`：release publish 改进路线图（P0-P3）
- `stage/plan-sync.md`：plan sync 命令设计方案

### Changed
- `stage/plan.md` 重命名为 `stage/plan_sync.md` → `stage/plan-sync.md`
- 平台愿景文档移至 `data/intention/stage/`
- bug 报告移至 GitHub Issues（qtcloud-devops#17）
- 代码同步状态规范移至 `data/context/specification/`
- 历史记录移至 `data/history/`

### Removed
- `platform/bug.md`（移至 GitHub Issues）
- `platform/plan-command-bug.md`（内容过时）
- `platform/release-command-enhance.md`（内容过时）
- `platform/index.md`（移至 intention）
- `specification/code-sync-status.md`（移至 context）
- `history/index.md`（移至 history 子模块）

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
