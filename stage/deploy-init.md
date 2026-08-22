# deploy init — 设计方案

> 2026-08-22 · 来自 issue #22：新增 `deploy` 命令集，一键为新仓库生成/升级部署能力。
> 本文档聚焦 **`deploy init`**（`scaffold → init` 命名决策见文末）。

## 动机

qtcloud-devops 目前只覆盖 `build → test → release`，**部署环节完全靠手工**：每个仓库手动复制一份 `.github/workflows/deploy-*.yml` + `manifests/terraform/`。

量化痛点：

- 全仓 **71 个 `deploy-*.yml`**（site/studio/provider/docs/oss 多形态），彼此高度相似。
- **13 份重复的 `manifests/terraform`**（`cdn.tf` + `site-bucket.tf`/`studio-bucket.tf`），仅域名/桶名/技术栈不同。
- 平台已有可复用模板，但**无机制一键套用**，也没做模板与 per-repo 拷贝的收敛。
- CLI `src/` 里 **0 处 `deploy`**，部署尚未纳入工具链。

## 命名：`deploy init`（而非 `scaffold`）

- **就地操作**：作用于当前仓库，把部署配置写入 `.github/` 与 `manifests/terraform/`，是补齐标准配置而非产出独立项目骨架。与 `git init`/`terraform init`/`go mod init`/`npm init` 语义一致。
- **天然涵盖「生成 + 升级」**：已有仓库重跑 `init` 即收敛/更新，无需 `re-scaffold`。
- **命令族一致**：`deploy init → status → audit → apply`。
- 若未来出现「从模板实例化独立可运行项目」场景（如实验室），才适用 `scaffold`；届时可作为 `init` 的等价别名。

## 命令设计

```
qtcloud-devops deploy init [OPTIONS]
```

### 参数

| 参数 | 必填 | 默认 | 说明 |
|------|------|------|------|
| `--domain <host>` | ✅ | — | 站点域名，如 `crowd.quanttide.com`；单层子域自动推导泛域名证书适用 |
| `--kind <site|studio|provider|docs>` | ✅ | — | 部署形态 |
| `--stack <vite|flutter|python|rust|myst>` | ⚠️ | 由 kind 推导 | 技术栈；用于推导 `build_dir`、模板选择 |
| `--bucket <name>` | ❌ | `{repo}-{kind}` | OSS 桶名（`{repo}` 取当前 git 仓库名，去除 `qt` 前缀前可保留完整名） |
| `--repo <name>` | ❌ | 自动检测 | 覆盖仓库名（含 `qt` 前缀的名称推导桶名时使用） |
| `--force` | ❌ | false | 覆盖已存在文件 |
| `--dry-run` | ❌ | false | 只打印将生成/更改的文件与内容，不落盘 |
| `--from-template` | ❌ | false | 生成调用平台 `deploy-site.yml`/`deploy-studio.yml` 的**薄 wrapper**；否则生成独立完整 workflow（含缓存策略分离） |
| `--apply` | ❌ | false | 生成后执行 `terraform apply` 建桶/CDN/DNS |

### 行为

1. 读取当前仓库 `.quanttide/devops/contract.yaml`（可选）识别 scope、project、language。
2. 解析 `--domain` → 提取子域 `rr`、完整 `cdn_domain`、`cdn_domains` JSON 数组。
3. 由 `--kind` + `--stack` 查模板映射表，推导 `build_dir`、桶名、缓存策略、DSN rr。
4. 生成 workflow + `manifests/terraform/*`（见输出结构）。
5. 若目标文件已存在且内容不同：
   - 未加 `--force`：打印差异并提示 `--force` 覆盖，不写入。
   - 加了 `--force`：覆盖写入。
6. 输出生成/变更清单，末尾附「部署就绪提示」（缺 secrets、证书、SPA 回退、备案）。
7. `--apply` 时按 `terraform init -backend-config=...` → `terraform apply -target=...` 执行。

## 输出结构

生成到 `manifests/terraform/` 的 8 个文件（对照 `qtcrowd` 现有 manifest）：

```
manifests/terraform/
  cdn.tf            # CDN + DNS + 私有回源鉴权 + SPA 回退 + 强制 HTTPS + RAM 授权
  site-bucket.tf    # 或 studio-bucket.tf：OSS 静态网站桶（index/error document）
  locals.tf         # app_name_prefix / oss_bucket 推导
  outputs.tf        # oss_bucket / cdn_domain / cdn_cname
  platform.tf       # data.terraform_remote_state.platform（引用 quanttide-platform）
  providers.tf      # alicloud provider + OSS backend（凭据环境变量）
  variables.tf      # region/project/environment/oss_bucket_name/image
  versions.tf       # required_version + required_providers
```

以及 `.github/workflows/deploy-{kind}.yml`：

- **独立完整版**（默认，`--from-template` 未开启）：含 terraform apply → build → 上传 OSS（产物长缓存 + `index.html` no-cache 分离）→ CDN 刷新。对准 `qtcrowd` 已上线版本。
- **薄 wrapper 版**（`--from-template`）：`workflow_call` 引用平台 `deploy-site.yml`/`deploy-studio.yml`，仅传 `build_dir`/`oss_bucket`/`cdn_domains`。

## 模板映射表

| kind | workflow 模板 | stack | build_dir 推导 | bucket 默认 | terraform 桶文件 | 缓存策略 |
|------|--------------|-------|----------------|-------------|------------------|----------|
| site | deploy-site.yml / react-web-deploy.yml | vite | `src/site/dist/`（Vite）| `{repo}-site` | site-bucket.tf | 产物 1y / index.html no-cache |
| site | deploy-site.yml | rust | `{repo}/dist/` 或由 Cargo 推导 | `{repo}-site` | site-bucket.tf | 同上 |
| studio | deploy-studio.yml | flutter | `src/studio/build/web/` | `{repo}-studio` | studio-bucket.tf | 产物 1y / index.html no-cache |
| provider | pypi-publish.yml / pubdev-publish.yml | python/dart | — | — | — | — |
| docs | myst-deploy.yml | myst | `_build/html/` | `{repo}-docs` | site-bucket.tf | 产物 1y / index.html no-cache |

**CDN/DNS 推导（`--domain crowd.quanttide.com`）：**

- `rr` = `crowd`（`quanttide.com` 前的首段）
- `cdn_domain` = `crowd.quanttide.com`
- `cdn_domains` = `["https://crowd.quanttide.com/"]`
- OSS 源站 = `{bucket}.oss-cn-hangzhou.aliyuncs.com`

## 已知坑（模板自动处理）

1. **SSL 证书**：terraform 不管理证书内容（避免私钥入库）。单层子域可用 `*.quanttide.com` 泛域名证书（`certificate_config` 注释占位）；生成后提示：证书未配置前域名仅 HTTP 可用，需在证书服务查询后填入 `server_certificate_status`。
2. **SPA 回退**：React/Router 子路由直接访问/刷新 404。生成 `back_to_origin_url_rewrite` 规则，`source_url = ^/(?!index\.html$|assets/|vite\.svg$).*` → `/index.html`，`flag = break`。
3. **私有 OSS 回源鉴权**：RAM 用户无公共读权限，CDN 需私有回源。生成 `l2_oss_key private_oss_auth=on` + 账号级 RAM 角色/策略（`AliyunCDNAccessingPrivateOSSRole`）。
4. **缓存策略分离**：`assets/*` 长缓存 `max-age=31536000`；`index.html` `no-cache`。独立 workflow 中拆两步上传。
5. **域名备案**：`.quanttide.com` 需 ICP 备案前置，生成时输出提示。

## 后续命令 preview（不在本文档展开）

- `deploy status`：检查 workflow / terraform / org secrets / CDN 状态 / DNS CNAME / 证书 / SPA 回退。
- `deploy audit`：对照平台标准模板做漂移检测，输出差异清单（针对现有 71 份 workflow）。
- `deploy apply`：（可选）把 build+upload+refresh 抽成 CLI 可调用，支持 `--dry-run`。

## 验收标准映射

| # | 验收标准 | 覆盖点 |
|---|----------|--------|
| 1 | `deploy init --domain crowd.quanttide.com --kind site --stack vite` 在空仓库生成可用配置（含 SPA 回退、缓存分离、DNS、证书提示） | 模板映射表 + 已知坑 2/4/1 |
| 2 | `deploy status` 正确报告就绪度 | 后续命令 status |
| 3 | `deploy audit` 对 71 份 workflow 漂移检测 | 后续命令 audit |
| 4 | 阶段二：terraform module 收敛后 13 份 manifest 瘦身为薄 wrapper | 依赖 module 收敛 |
| 5 | README 新增 `deploy` 部分，说明标准流程 | 文档 |

## 开发任务拆解

1. **契约/模型**：`contract/` 增加 `deploy.yaml` 解析（domain/kind/stack/bucket 声明），或在 `deploy/` 内定义 `DeployConfig` 结构。
2. **模板资产**：把 per-repo 已上线的独立 workflow 与 `cdn.tf` 收敛为 CLI 内嵌的「canonical 模板片段」，支持 ${VAR}/{{DOMAIN}} 占位符。
3. **`deploy/mod.rs` + `deploy/init.rs`**：核心生成逻辑：
   - `resolve_stack(kind, stack) -> Stack`（缺省推导）
   - `derive_domain(host) -> (rr, cdn_domain, cdn_domains)`
   - `derive_bucket(repo, kind, bucket)`
   - `render_workflow(...)` / `render_terraform(...)`
   - `emit(paths) -> GenReport`（含 diff / force / dry-run）
4. **main.rs 路由**：`Commands::Deploy { action: DeployAction }`，注册 `Init/Status/Audit/Apply`。
5. **lib.rs 导出**：`pub mod deploy;`。
6. **测试**：`deploy/init.rs` 纯逻辑单测（domain/bucket/stack 推导）；`tests/cli.rs` 端到端（tempdir 空仓库 → 生成文件 → 断言内容）。
7. **文档**：README 新增 `deploy` 章节；CLI `docs/deploy/init.md`。
8. **幂等 + 状态**：`init` 生成的清单写入审计快照，供 `deploy status` 消费。

## 风险 / Open Questions

- **模板以谁为准**：以各 per-repo **已上线**版本（含缓存分离 + SPA 回退 + terraform apply）为源反推，平台 `assets/.../workflows/deploy-*.yml` 作为简化 `workflow_call` 变体，逐步收敛两套并存。
- **Terraform module 归属**：倾向 `modules/site`、`modules/studio` 放 `quanttide-platform`（共享 IaC），CLI 只负责生成/审计（`deploy init` 据此生成薄 wrapper）。
- **`deploy` 偏向哪种**：本方案取「生成配置」为主（`init`），`apply` 作为可选执行入口，兼顾两态。
