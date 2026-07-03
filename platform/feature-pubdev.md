# pub.dev 发布集成

`publish --registry pub-dev` 当前行为：创建 tag + GitHub Release → 打印提示。

和其他 registry（crates / pypi）行为一致，**不做本地发布**。发布由 CI 或用户手动执行。

2026-07-02 确认：框架支持已完备（`Registry::PubDev` → `PublishTarget::PubDev` → `--registry pub-dev`），无需额外改动。
