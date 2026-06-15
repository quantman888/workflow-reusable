# workflow-reusable

该仓库提供可复用工作流与 shared composite actions，当前包含：

- `.github/workflows/docker-publish.reusable.yml`：Docker 镜像构建/发布原语
- `.github/workflows/docker-promote.reusable.yml`：按 digest 把一个源镜像显式提升到目标 refs
- `.github/workflows/python-package-publish.reusable.yml`：Python 包构建/发布原语
- `.github/workflows/fork-sync.reusable.yml`：fork 分支快进同步
- `.github/workflows/branch-sync-pr.reusable.yml`：分支差异检测并自动开 PR
- `.github/workflows/reusable-workflow-update-pr.reusable.yml`：批量更新调用方仓库里的 reusable workflow 引用并开 PR
- `.github/workflows/runner-fallback.reusable.yml`：先探测 self-hosted，再回退 github-hosted
- `.github/workflows/github-app-secret-sync.controlplane.yml`：把 `GH_APP_*` 同步到 private repos 的公共控制面
- `.github/actions/resolve-github-app-bot/action.yml`：解析 GitHub App bot 的 login/name/email/user_id
- `.github/actions/write-commit-status/action.yml`：通过 GitHub Statuses API 写 commit status

变更策略：各模块输入字段采用硬切命名，不提供旧字段向后兼容。

## 0. 仓库关系与升级入口

当前组织把 workflow 治理拆成两层：

- `quantman888/workflow-reusable`：运行时控制面与 reusable workflow 实现
- `quantman888/.github`：组织模板入口与接入/升级/审计文档

推荐入口：

- 模板入口仓：`https://github.com/quantman888/.github`
- 新仓接入清单：`https://github.com/quantman888/.github/blob/main/docs/new-repo-onboarding-checklist.md`
- 发布与升级规范：`https://github.com/quantman888/.github/blob/main/docs/workflow-governance-release-process.md`
- 组织基线审计：`https://github.com/quantman888/.github/blob/main/docs/org-workflow-baseline-audit-2026-03-07.md`

治理闭环固定为：

1. 先在本仓发布新的 reusable/control-plane 实现
2. 再把 `quantman888/.github` 模板 pin 更新到新的 blessed SHA
3. 最后由 caller 仓内的 `reusable-workflow-update-pr` 开 PR 升级现有业务仓引用

不要把 `.github` 当成运行时控制面，也不要让 caller 仓直接长期跟随 `@main`。

## 1. Docker Publish Reusable

文件：`.github/workflows/docker-publish.reusable.yml`

用途：只负责 checkout、registry login、buildx build/push、digest 输出。

不负责 release 语义、tag 推导、漏洞扫描、启动门禁、Nexus 验收或 promotion。

默认关闭 Buildx provenance/SBOM attestation，保持普通 registry 兼容的单平台 image manifest；需要供应链 attestation 的调用方应在独立扫描/签名流程处理。

### 调用示例

```yaml
jobs:
  docker-publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@<pinned-ref>
    with:
      OCI_REGISTRY: registry-push.example.com
      OCI_IMAGE_NAME: team/app
      OCI_DOCKERFILE_PATH: Dockerfile
      OCI_BUILD_CONTEXT: .
      OCI_TARGET_PLATFORMS: linux/amd64
      OCI_BUILD_ARGS: |
        APP_ENV=prod
      OCI_TAGS: |
        sha-${{ github.sha }}
        main
      OCI_PUSH_IMAGE: true
      OCI_EXTRA_PULL_REGISTRY: registry.example.com
      OCI_CACHE_SCOPE: team-app
      RUNS_ON_JSON: '["self-hosted","Linux","X64","platform-image-build"]'
    secrets:
      registry_username: ${{ secrets.NEXUS_REGISTRY_USERNAME }}
      registry_password: ${{ secrets.NEXUS_REGISTRY_PASSWORD }}
      pull_registry_username: ${{ secrets.NEXUS_REGISTRY_USERNAME }}
      pull_registry_password: ${{ secrets.NEXUS_REGISTRY_PASSWORD }}
```

生产环境不要使用 `@main`。统一改为受控版本 tag 或完整 commit SHA（例如 `@v1` 或 `@<40位SHA>`）。

### Inputs

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `OCI_REGISTRY` | `string` | 是 | 无 | Docker Registry 地址，例如 `registry-push.example.com` |
| `OCI_IMAGE_NAME` | `string` | 是 | 无 | 镜像名，例如 `team/app` |
| `OCI_DOCKERFILE_PATH` | `string` | 否 | `Dockerfile` | Dockerfile 路径 |
| `OCI_BUILD_CONTEXT` | `string` | 否 | `.` | Docker build context |
| `OCI_TARGET_PLATFORMS` | `string` | 否 | `linux/amd64` | Buildx 平台列表 |
| `OCI_BUILD_ARGS` | `string` | 否 | 空字符串 | 多行 `KEY=VALUE` |
| `OCI_TAGS` | `string` | 否 | 空字符串 | tag 或当前镜像下完整 ref，支持换行/逗号/分号；空值时只发布 `sha-<commit前12位>` |
| `OCI_PUSH_IMAGE` | `boolean` | 否 | `true` | 是否推送镜像到 registry；PR 可设为 `false` 只构建 |
| `OCI_EXTRA_PULL_REGISTRY` | `string` | 否 | 空字符串 | 构建前额外登录的 pull registry，适合私有 base image |
| `OCI_CACHE_SCOPE` | `string` | 否 | 空字符串 | Buildx GitHub Actions cache scope；非空时启用 `type=gha,scope=<value>` |
| `RUNS_ON_JSON` | `string` | 否 | `"ubuntu-latest"` | 传给 `runs-on` 的 JSON 值 |

### Secrets

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `registry_username` | 否 | push registry 登录用户名；`OCI_PUSH_IMAGE=true` 且 registry 需要认证时提供 |
| `registry_password` | 否 | push registry 登录密码 |
| `pull_registry_username` | 否 | 可选 pull registry 登录用户名 |
| `pull_registry_password` | 否 | 可选 pull registry 登录密码 |

Nexus 调用方只在 caller workflow 里把 `NEXUS_REGISTRY_USERNAME` / `NEXUS_REGISTRY_PASSWORD` 映射到上述通用 secret 名；reusable workflow 不出现 Nexus 专属字段。

### Outputs

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `image_ref` | `string` | 不带 tag 的完整镜像引用，格式 `<registry>/<image_name>` |
| `image_tags` | `string` | 逗号分隔的 tag 列表（仅 tag 名，不含镜像前缀） |
| `image_refs` | `string` | 逗号分隔的完整 ref 列表 |
| `digest` | `string` | `docker/build-push-action` 输出的 digest；未 push 时可能为空 |
| `image_with_digest` | `string` | 当 `digest` 非空时为 `<registry>/<image_name>@sha256:...`，否则为空字符串 |

### 固定步骤

1. `actions/checkout`
2. 解析 `OCI_TAGS`；空值只生成 `sha-<short-sha>`
3. `docker/setup-buildx-action`
4. 可选登录 `OCI_EXTRA_PULL_REGISTRY`
5. 可选登录 `OCI_REGISTRY`
6. `docker/build-push-action` build/push
7. 输出 digest summary

扫描、gate、Nexus 验收、release tag 生成、immutable tag 策略全部放到 caller 或独立 workflow，不放在 publish 原语里。

## 2. Docker Promote Reusable

文件：`.github/workflows/docker-promote.reusable.yml`

用途：把一个源镜像解析成 digest，再用 `docker buildx imagetools create --tag` 提升到 caller 明确给出的目标 refs。

不推导 `latest`、major/minor、semver，也不判断 release channel。

### 调用示例

```yaml
jobs:
  promote:
    uses: <owner>/workflow-reusable/.github/workflows/docker-promote.reusable.yml@<pinned-ref>
    with:
      OCI_SOURCE_REF: registry-push.example.com/team/app:sha-${{ github.sha }}
      OCI_TARGET_REFS: |
        registry-push.example.com/team/app:1.2.3
        registry-push.example.com/team/app:latest
      RUNS_ON_JSON: '"ubuntu-latest"'
    secrets:
      registry_username: ${{ secrets.NEXUS_REGISTRY_USERNAME }}
      registry_password: ${{ secrets.NEXUS_REGISTRY_PASSWORD }}
```

### Inputs

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `OCI_SOURCE_REF` | `string` | 是 | 无 | 源镜像完整 ref，可是 tag ref 或 digest ref |
| `OCI_TARGET_REFS` | `string` | 是 | 无 | 目标完整 tag refs，支持换行/逗号/分号 |
| `OCI_LOGIN_REGISTRIES` | `string` | 否 | 空字符串 | 登录 registry 列表；空值时从 `OCI_SOURCE_REF` 与 `OCI_TARGET_REFS` 自动提取 |
| `OCI_PROMOTE_DRY_RUN` | `boolean` | 否 | `false` | 只打印 promotion plan，不创建 tag |
| `RUNS_ON_JSON` | `string` | 否 | `"ubuntu-latest"` | 传给 `runs-on` 的 JSON 值 |

### Outputs

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `source_ref` | `string` | 源镜像 ref |
| `digest` | `string` | 源镜像 digest |
| `source_with_digest` | `string` | 源镜像 digest ref |
| `promoted_refs` | `string` | 逗号分隔的目标 refs |
| `promoted_count` | `string` | 创建的目标 ref 数量 |

## 3. Python Package Publish Reusable

文件：`.github/workflows/python-package-publish.reusable.yml`

用途：基于当前仓库源码构建 wheel/sdist，按显式开关上传到 Python package repository，并在 run summary 输出安装命令。

变更策略：Python 包发布与 OCI 发布完全解耦；是否发布由 caller workflow 的显式 job/checkbox 决定，不做仓库智能识别。

### 调用示例

```yaml
jobs:
  python-package-publish:
    uses: <owner>/workflow-reusable/.github/workflows/python-package-publish.reusable.yml@<pinned-ref>
    with:
      PYTHON_PACKAGE_REPOSITORY_URL: ${{ vars.NEXUS_PYPI_REPOSITORY_URL }}
      PYTHON_PACKAGE_SIMPLE_URL: ${{ vars.NEXUS_PYPI_SIMPLE_URL || '' }}
      PYTHON_PACKAGE_TRUSTED_HOST: ${{ vars.NEXUS_PYPI_TRUSTED_HOST || '' }}
      PYTHON_PUSH_PACKAGE: true
      PYTHON_SKIP_EXISTING: true
    secrets:
      repository_username: ${{ secrets.NEXUS_REGISTRY_USERNAME }}
      repository_password: ${{ secrets.NEXUS_REGISTRY_PASSWORD }}
```

### 关键约定

- 包名与版本从构建产物元数据解析，不要求 caller 额外传入。
- `PYTHON_PUSH_PACKAGE=false` 时仍会构建并输出元数据，但不会上传。
- `PYTHON_SKIP_EXISTING=true` 时先查 simple index 中已存在的分发文件，只上传缺失文件。
- 仅当 `PYTHON_PACKAGE_SIMPLE_URL` 非空且可解析 wheel 下载链接时，summary 才输出精确到 wheel URL + SHA 的安装命令。
- Python package repository 凭据统一使用 `repository_username` / `repository_password`；Nexus 只在 caller 侧映射。

## 4. GitHub App 治理约定

分支同步类 workflow 统一采用 GitHub App installation token，不再依赖 `github.token`、长期 PAT 或仓库特有兜底 token。

调用方统一准备以下密钥：

| 名称 | 类型 | 作用 |
| --- | --- | --- |
| `GH_APP_ID` | Secret | GitHub App ID。现有模板与 caller workflow 统一按 `secrets.GH_APP_ID` 读取。 |
| `GH_APP_PRIVATE_KEY` | Secret | GitHub App 私钥 |
| `RUNNER_PROBE_TOKEN` | Secret（可选） | `runner-fallback.reusable.yml` 探测 runner 时使用 |

当前组织配置建议：

- Public repositories：可直接使用 organization-level secrets 下发 `GH_APP_ID` / `GH_APP_PRIVATE_KEY`
- Private repositories：在当前 GitHub 计划限制下，需为每个仓库同步同名 repo secrets
- Workflow 引用：建议统一 pin 到 commit SHA，而不是直接引用浮动分支

为了解决 private repositories 无法直接消费 organization secrets 的限制，仓库内提供了公共控制面 workflow：

- 文件：`.github/workflows/github-app-secret-sync.controlplane.yml`
- 作用：按计划任务或手动触发，把 `GH_APP_ID` 与 `GH_APP_PRIVATE_KEY` 同步到目标仓库的 repository secrets
- 前提：GitHub App installation 已授予 `Secrets: Read and write`
- 覆盖方式：基于 GitHub API 分页扫描组织仓库，不依赖固定仓库清单；新仓库会在下次计划任务中自动补齐
- 推荐策略：计划任务保持 `private` 默认值；如需一次性补齐 public/private 全量仓库，可手动以 `target_visibility=all` 触发

分支同步类 workflow 会把 token 权限显式收敛为当前仓库所需的最小集合：

- `contents: write`
- `pull-requests: write`（仅 `branch-sync-pr.reusable.yml`）
- `workflows: write`

批量更新 workflow 引用并开 PR 的场景，也统一采用 GitHub App installation token：

- 文件：`.github/workflows/reusable-workflow-update-pr.reusable.yml`
- 作用：扫描调用方仓库 `.github/workflows` 中对中央 reusable 的引用，批量替换目标 ref，并用 App bot 推分支/开 PR
- 适用：`workflow-reusable` 发布后，批量驱动各业务仓库 bump 到新的 SHA 或版本 tag
- 运行方式：支持调用方先经 `runner-fallback.reusable.yml` 决定 `RUNS_ON_JSON`，再把选中的 runner 透传给该 reusable
- `reusable_workflow_path` 留空时，会一次升级该仓 `.github/workflows/` 下所有指向 `reusable_repo` 的 central workflow refs；指定具体 path 时，则只升级该 path
- 分支/PR 策略：始终复用固定 bot 分支 `automation/reusable-workflow-update`；PR title 会继续写入当前 `target_ref`，但不会再按 `target_ref` 派生新分支，因此调用方仓库只维护同一张 open 更新 PR

shared managed-PR 基础能力统一沉淀在两个 composite action 中：

- `.github/actions/resolve-github-app-bot/action.yml`
  - 输入：`github-token`、`app-slug`
  - 输出：`login`、`name`、`email`、`user_id`（并提供同值 `user-id` 输出）
  - 用途：统一为 `fork-sync.reusable.yml`、`branch-sync-pr.reusable.yml`、`reusable-workflow-update-pr.reusable.yml` 解析 App bot 身份，避免各 workflow 内联 `gh api /users/<slug>[bot]`
  - 发布约束：跨仓调用 reusable workflow 时，内部 action 也必须使用 immutable ref；当前仓内 workflow 会把它 pin 到同仓已发布 SHA，再随后续 release/升级流程统一 bump
- `.github/actions/write-commit-status/action.yml`
  - 输入：`github-token`、`repository`、`sha`、`context`、`state`、`description`、`target-url`
  - 实现：直接调用 GitHub Statuses API `POST /repos/{owner}/{repo}/statuses/{sha}`
  - 控制面约束：调用该 action 的 token 需要具备 `statuses: write`；`context` 应保持稳定且可审计，`description` 保持简短，`target-url` 指向 run、PR 或控制面页面

## 5. Fork Sync Reusable

文件：`.github/workflows/fork-sync.reusable.yml`  
用途：将 fork 的目标分支与上游分支做同步；默认使用 `--ff-only`，也支持显式强制重置。

### Inputs

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `target_branch` | `string` | 否 | `main` | 当前仓库要同步的目标分支 |
| `upstream_repo` | `string` | 是 | 无 | 上游仓库，格式 `owner/repo` |
| `upstream_branch` | `string` | 否 | `main` | 上游分支 |
| `force_reset` | `boolean` | 否 | `false` | 是否强制把目标分支重置到上游分支 |

### Outputs

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `needs_sync` | `string` | 本地与上游 SHA 是否不同（`true/false`） |
| `local_sha` | `string` | 本地目标分支 SHA |
| `upstream_sha` | `string` | 上游分支 SHA |
| `synced` | `string` | 是否执行并完成同步推送（`true/false`） |

## 6. Branch Sync PR Reusable

文件：`.github/workflows/branch-sync-pr.reusable.yml`  
用途：比较 `target..source` 提交差异；有新增时检查是否已有 open PR，无则通过 REST API 创建 PR。

### Inputs

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `source_branch` | `string` | 否 | `main` | PR head 分支 |
| `target_branch` | `string` | 否 | `custom/docker` | PR base 分支 |
| `pr_title` | `string` | 否 | `chore(sync): merge main into custom/docker` | PR 标题 |
| `pr_body` | `string` | 否 | 自动说明文本 | PR 描述 |
| `auto_resolve_theirs_paths` | `string` | 否 | 空字符串 | 多行路径列表；发生冲突时这些路径自动采用 source(theirs) |

### Outputs

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `has_new_commits` | `string` | source 是否有 target 没有的新提交 |
| `pr_exists` | `string` | 是否已存在 open PR（base=target, head=`owner:source`） |
| `pr_created` | `string` | 本次是否新建 PR |
| `pr_url` | `string` | 已有或新建 PR 的 URL |

## 7. Runner Fallback Reusable

文件：`.github/workflows/runner-fallback.reusable.yml`  
用途：先探测 `self-hosted` runner 可用性；可用则走 `self-hosted`，不可用则按策略回退到 `github-hosted`。  
变更策略：`RUNNER_*` 输入已硬切，不提供旧字段向后兼容。

### Inputs

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `RUNNER_SELF_HOSTED_LABELS` | `string` | 是 | 无 | self-hosted 目标标签（逗号/分号/换行分隔），例如 `self-hosted,linux,x64,ci-main` |
| `RUNNER_GITHUB_HOSTED_LABEL` | `string` | 否 | `ubuntu-latest` | 回退使用的 GitHub-hosted 标签 |
| `RUNNER_PROBE_SCOPE` | `string` | 否 | `auto` | 探测范围：`repo` / `org` / `auto`（按 `repo -> org` 顺序探测） |
| `RUNNER_TARGET_REPOSITORY` | `string` | 否 | 空字符串 | 目标仓库，格式 `owner/repo`；为空则用当前仓库 |
| `RUNNER_TARGET_ORG` | `string` | 否 | 空字符串 | 目标组织名；为空则使用目标仓库 owner |
| `RUNNER_PROBE_REQUIRE_ONLINE` | `boolean` | 否 | `true` | 是否要求命中的 self-hosted runner 处于 `online` |
| `RUNNER_PROBE_REQUIRE_IDLE` | `boolean` | 否 | `true` | 是否要求命中的 self-hosted runner 处于 `busy=false` |
| `RUNNER_MIN_MATCH_COUNT` | `number` | 否 | `1` | 满足条件的最小命中数量 |
| `RUNNER_PROBE_TIMEOUT_SECONDS` | `number` | 否 | `20` | 每个探测请求的超时秒数 |
| `RUNNER_FALLBACK_POLICY` | `string` | 否 | `github-hosted` | 回退策略：`github-hosted` / `fail` |

### Outputs

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `selected_runner` | `string` | 最终选择：`self-hosted` 或 `github-hosted` |
| `use_self_hosted` | `string` | 是否使用 self-hosted（`true/false`） |
| `use_github_hosted` | `string` | 是否使用 github-hosted（`true/false`） |
| `resolved_self_hosted_labels` | `string` | 归一化后的 self-hosted 标签串（逗号分隔） |
| `resolved_github_hosted_label` | `string` | 归一化后的 fallback 标签 |
| `self_hosted_runs_on_json` | `string` | self-hosted 路由的 `runs-on` JSON 值（数组） |
| `github_hosted_runs_on_json` | `string` | github-hosted 路由的 `runs-on` JSON 值（字符串） |
| `selected_runs_on_json` | `string` | 当前选中路由对应的 `runs-on` JSON 值 |
| `probe_scope_used` | `string` | 命中的探测范围：`repo` / `org` / `none` |
| `matched_runner_count` | `string` | 命中的 runner 数量 |
| `probe_reason` | `string` | 探测结果原因 |
| `probe_summary` | `string` | 探测摘要 JSON（可用于审计与观测） |

### Secrets

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `runner_probe_token` | 否 | 用于调用 runner API 的 token；未提供时回退使用 `${{ github.token }}`。组织级探测建议显式提供具有 `admin:org` 的 token |

### 组织级最佳实践

- 统一标签体系：组织内约定 `self-hosted` 基础标签与业务标签（如 `ci-main`/`build`/`deploy`）。
- 回退策略分级：普通 CI 用 `github-hosted` 回退；关键发布链路可设 `RUNNER_FALLBACK_POLICY=fail` 防止静默降级。
- 观测与审计：在调用方记录 `selected_runner`、`probe_reason`、`probe_summary` 便于容量规划和故障复盘。
- 权限最小化：`runner_probe_token` 仅授予探测所需权限；组织探测场景推荐组织级 secret 统一分发。
