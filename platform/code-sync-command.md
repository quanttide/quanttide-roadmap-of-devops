看了完整的 code.rs，现在我们有了最准确的上下文。让我直接针对这份代码，分析现有的状态设计并提出具体的重构方案。

—

现有 SubmoduleStatus 的问题分析

当前定义（从代码使用推断）

```rust
pub enum SubmoduleStatus {
    Clean,           // 正常
    AheadOfParent,   // 子模块领先父仓库记录
    BehindRemote,    // 远程有更新
    Detached,        // 游离 HEAD
    Dirty,           // 未提交修改
    Orphaned,        // 父仓库记录的 commit 远程不存在
    Uninitialized,   // 未初始化
}
```

问题 1：混杂了两种状态维度

AheadOfParent 和 BehindRemote 是同步状态（可以通过 push/pull 自动修复），而 Detached, Orphaned, Uninitialized 是结构异常（需要人工决策）。Dirty 则处在中间地带。

把它们平铺在一个枚举里，导致下游代码无法区分”能自动修复“和”需要人看“。

证据就在你贴的代码里——describe_issue 函数对每种状态只是返回描述文本和建议，但调用方 status() 方法只把它们统统塞进 HealthIssue 列表，没有分级处理逻辑：

```rust
// 当前代码：所有状态一视同仁
for sm in &state.submodules {
    if sm.status != SubmoduleStatus::Clean {
        let (description, action) = describe_issue(&sm.status);
        issues.push(HealthIssue { ... });
    }
}
```

问题 2：fetch 作用域导致误判

这个你已经发现了。status() 方法里：

```rust
// 第 152-161 行
if !self.offline {
    if let Ok(repo) = git2::Repository::open(&self.root) {
        if let Ok(mut remote) = repo.find_remote(”origin“) {
            // 这里 fetch 的是父仓库
            let _ = remote.fetch(
                &[”+refs/heads/*:refs/remotes/origin/*“],
                Some(&mut fetch_opts),
                None,
            );
        }
    }
}
let state = RepoState::scan(&self.root)?;  // scan 读的是子模块自己的 remote refs
```

RepoState::scan() 的实现虽然不在这个文件里，但从现象可以断定：它打开子模块仓库，读取子模块的 refs/remotes/origin/main，而这个引用没有被父仓库的 fetch 更新。

问题 3：Dirty 的检测时机有问题

status() 在检测前做了 fetch（虽然作用域错了），但它没有检查 fetch 本身是否成功。如果网络不通，remote.fetch() 静默失败，然后 scan() 基于过时的 remote refs 做出误判。这个 offline 标志的设计意图是好的，但静默失败让问题不可见。

—

重构方案（直接对应这份代码）

第一步：重新定义 SubmoduleStatus

```rust
/// 顶层状态：对应你提出的三分法
pub enum SubmoduleStatus {
    /// 理想状态
    Synchronized,
    /// 可自动修复
    OutOfSync(OutOfSyncKind),
    /// 需要人工介入
    Anomaly(AnomalyKind),
}

#[derive(Debug, Clone, PartialEq)]
pub enum OutOfSyncKind {
    /// 子模块有本地提交未推送到 origin
    AheadOfRemote,
    /// 子模块 origin 有新提交未拉取
    BehindRemote,
    /// 工作区或暂存区有未提交变更
    Dirty,
}

#[derive(Debug, Clone, PartialEq)]
pub enum AnomalyKind {
    /// HEAD 游离
    DetachedHead,
    /// 父仓库记录的 commit 在远程不存在
    Orphaned,
    /// 子模块目录存在但未注册到父仓库
    Unregistered,
    /// 子模块目录不存在
    Missing,
    /// 未知 git 错误
    Unknown(String),
}
```

第二步：重写 describe_issue，输出分级诊断

```rust
pub(crate) fn describe_issue(status: &SubmoduleStatus) -> HealthIssue {
    match status {
        SubmoduleStatus::Synchronized => {
            unreachable!()  // Clean 不产生 HealthIssue
        }
        SubmoduleStatus::OutOfSync(kind) => match kind {
            OutOfSyncKind::AheadOfRemote => HealthIssue {
                severity: Severity::Info,
                description: ”子模块有未推送的本地提交“.into(),
                suggested_action: ”code sync 将自动推送“.into(),
                auto_fixable: true,
            },
            OutOfSyncKind::BehindRemote => HealthIssue {
                severity: Severity::Info,
                description: ”远程有新的提交“.into(),
                suggested_action: ”code sync 将自动拉取“.into(),
                auto_fixable: true,
            },
            OutOfSyncKind::Dirty => HealthIssue {
                severity: Severity::Warning,
                description: ”有未提交的修改“.into(),
                suggested_action: ”请先提交或 stash 修改，然后运行 sync“.into(),
                auto_fixable: false,  // 需要用户确认是否 stash
            },
        },
        SubmoduleStatus::Anomaly(kind) => match kind {
            AnomalyKind::DetachedHead => HealthIssue {
                severity: Severity::Error,
                description: ”HEAD 处于游离状态“.into(),
                suggested_action: ”运行 git switch main 切换到主分支“.into(),
                auto_fixable: false,
            },
            AnomalyKind::Orphaned => HealthIssue {
                severity: Severity::Error,
                description: ”父仓库记录的 commit 在远程已不存在“.into(),
                suggested_action: ”需要手动检查子模块状态“.into(),
                auto_fixable: false,
            },
            AnomalyKind::Missing => HealthIssue {
                severity: Severity::Error,
                description: ”子模块目录不存在“.into(),
                suggested_action: ”运行 git submodule update —init“.into(),
                auto_fixable: true,  // init 可以自动执行
            },
            AnomalyKind::Unregistered => HealthIssue {
                severity: Severity::Warning,
                description: ”子模块目录存在但未在 .gitmodules 注册“.into(),
                suggested_action: ”手动清理或重新注册“.into(),
                auto_fixable: false,
            },
            AnomalyKind::Unknown(msg) => HealthIssue {
                severity: Severity::Error,
                description: format!(”未知异常: {}“, msg),
                suggested_action: ”请手动检查仓库状态“.into(),
                auto_fixable: false,
            },
        },
    }
}
```

第三步：让 status() 返回分级结果，并修复 fetch 作用域

```rust
fn status(&self) -> Result<Vec<HealthIssue>, Box<dyn std::error::Error>> {
    if !self.offline {
        // 对每个子模块单独 fetch，而不是只 fetch 父仓库
        if let Ok(repo) = git2::Repository::open(&self.root) {
            if let Ok(submodules) = repo.submodules() {
                for mut sm in submodules {
                    if let Ok(sm_repo) = sm.open() {
                        if let Ok(mut remote) = sm_repo.find_remote(”origin“) {
                            let mut fetch_opts = git2::FetchOptions::new();
                            fetch_opts.download_tags(git2::AutotagOption::None);
                            // 对子模块自己的仓库执行 fetch
                            let _ = remote.fetch(
                                &[”+refs/heads/*:refs/remotes/origin/*“],
                                Some(&mut fetch_opts),
                                None,
                            );
                        }
                    }
                }
            }
        }
    }

    let state = RepoState::scan(&self.root)?;
    let mut issues = Vec::new();
    for sm in &state.submodules {
        if sm.status != SubmoduleStatus::Synchronized {
            issues.push(describe_issue(&sm.status));
        }
    }
    // 按严重程度排序：Error > Warning > Info
    issues.sort_by_key(|i| i.severity as u8);
    Ok(issues)
}
```

第四步：改造 sync 逻辑，让状态守卫安全操作

你现有的 sync_to_parent 推了三个步骤，但它没有先检查状态。应该变成：

```rust
fn sync_to_parent(&self, name: &str) -> Result<(), Box<dyn std::error::Error>> {
    let state = RepoState::scan(&self.root)?;
    let sm_state = state.submodules.iter()
        .find(|s| s.name == name)
        .ok_or(format!(”子模块 ’{}‘ 不存在“, name))?;

    match &sm_state.status {
        SubmoduleStatus::Synchronized => {
            println!(”  {:<35} ✓ 已同步“, name);
            return Ok(());
        }
        SubmoduleStatus::OutOfSync(kind) => {
            // 根据种类决定操作
            match kind {
                OutOfSyncKind::AheadOfRemote | OutOfSyncKind::BehindRemote => {
                    // 执行现有的三步推送逻辑
                    self.do_sync(name)?;
                }
                OutOfSyncKind::Dirty => {
                    // 提示用户，不自动操作
                    println!(”  {:<35} ⚠ 有未提交修改，跳过“, name);
                    return Ok(());
                }
            }
        }
        SubmoduleStatus::Anomaly(kind) => {
            // 打印诊断，中止
            let issue = describe_issue(&sm_state.status);
            println!(”  {:<35} ✗ {}: {}“, name, issue.severity, issue.description);
            println!(”     建议: {}“, issue.suggested_action);
            return Ok(());
        }
    }
    Ok(())
}
```

—

对你现有测试的影响

好消息是，你现有的测试基本不需要大改。describe_issue 的测试只需要把 SubmoduleStatus::AheadOfParent 改成 SubmoduleStatus::OutOfSync(OutOfSyncKind::AheadOfRemote)，断言内容可以保持不变。

test_editor_status_with_dirty_submodule 里：

```rust
assert_eq!(issues[0].status, SubmoduleStatus::Dirty);
// 改成
assert_eq!(issues[0].status, SubmoduleStatus::OutOfSync(OutOfSyncKind::Dirty));
```

—

迁移路径

1. 在 model/code.rs 里定义新的 SubmoduleStatus 和辅助结构体（Severity 枚举、HealthIssue 加字段）。
2. 更新 RepoState::scan()（这个文件不在这里，但你知道位置）——让它在检测每个子模块时，先 fetch 子模块自己的 remote。
3. 改 describe_issue——用上面的新实现。
4. 改 status() 和 sync_to_parent()——加入状态守卫逻辑。
5. 更新测试——枚举变体名称调整。
6. 兼容期：保留旧的 SubmoduleStatus 变体作为 deprecated 别名一两个小版本，给其他调用方迁移时间。

