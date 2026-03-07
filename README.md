# workflow-reusable

该仓库提供可复用工作流，当前包含：

- `.github/workflows/docker-publish.reusable.yml`：Docker 镜像构建发布 + 中环门禁
- `.github/workflows/docker-promote.reusable.yml`：按 digest 做发布标签提升
- `.github/workflows/fork-sync.reusable.yml`：fork 分支快进同步
- `.github/workflows/branch-sync-pr.reusable.yml`：分支差异检测并自动开 PR
- `.github/workflows/reusable-workflow-update-pr.reusable.yml`：批量更新调用方仓库里的 reusable workflow 引用并开 PR
- `.github/workflows/runner-fallback.reusable.yml`：先探测 self-hosted，再回退 github-hosted
- `.github/workflows/github-app-secret-sync.controlplane.yml`：把 `GH_APP_*` 同步到 private repos 的公共控制面

变更策略：各模块输入字段采用硬切命名，不提供旧字段向后兼容。

## 1. 调用方式（caller）

在调用方仓库的 workflow 中通过 `uses` 引用：

```yaml
jobs:
  publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@main
    with:
      OCI_REGISTRY: ${{ vars.NEXUS_REGISTRY }}
      OCI_IMAGE_NAME: ${{ vars.IMAGE_NAME }}
      OCI_DOCKERFILE_PATH: Dockerfile
      OCI_BUILD_CONTEXT: .
      OCI_TARGET_PLATFORMS: linux/amd64
      OCI_BUILD_ARGS: |
        APP_ENV=prod
      OCI_PUSH_IMAGE: true
      OCI_PUBLISH_CHANNEL: release
      OCI_RELEASE_VERSION: 1.2.3
      OCI_PUBLISH_LATEST: true
      OCI_GATE_TIMEOUT_SECONDS: 60
      OCI_GATE_RUN_ARGS: ""
      OCI_GATE_HEALTH_URL: ""
      OCI_GATE_SMOKE_CMD: ""
      OCI_GATE_VULN_SEVERITY: HIGH,CRITICAL
      OCI_GATE_VULN_IGNORE_UNFIXED: false
    secrets:
      nexus_username: ${{ secrets.NEXUS_USERNAME }}
      nexus_password: ${{ secrets.NEXUS_PASSWORD }}
```

生产环境建议把 `@main` 换成受控版本 tag 或 commit SHA（例如 `@v1` 或 `@<40位SHA>`）。

## 1.1 GitHub App 治理约定

分支同步类 workflow 统一采用 GitHub App installation token，不再依赖 `github.token`、长期 PAT 或仓库特有兜底 token。

调用方统一准备以下密钥：

| 名称 | 类型 | 作用 |
| --- | --- | --- |
| `GH_APP_ID` | Secret/Variable | GitHub App ID |
| `GH_APP_PRIVATE_KEY` | Secret | GitHub App 私钥 |
| `RUNNER_PROBE_TOKEN` | Secret（可选） | `runner-fallback.reusable.yml` 探测 runner 时使用 |

当前组织配置建议：

- Public repositories：可直接使用 organization-level `GH_APP_ID` / `GH_APP_PRIVATE_KEY`
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

## 2. Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `OCI_REGISTRY` | `string` | 是 | 无 | Docker Registry 地址，例如 `nexus.example.com` |
| `OCI_IMAGE_NAME` | `string` | 是 | 无 | 镜像名，例如 `team/app` |
| `OCI_DOCKERFILE_PATH` | `string` | 否 | `Dockerfile` | Dockerfile 路径 |
| `OCI_BUILD_CONTEXT` | `string` | 否 | `.` | Docker build context |
| `OCI_TARGET_PLATFORMS` | `string` | 否 | `linux/amd64` | Buildx 平台列表 |
| `OCI_BUILD_ARGS` | `string` | 否 | 空字符串 | 多行 `KEY=VALUE` |
| `OCI_PUSH_IMAGE` | `boolean` | 否 | `true` | 是否推送镜像到 registry |
| `OCI_PUBLISH_CHANNEL` | `string` | 否 | `none` | 发布通道，仅允许 `none`/`edge`/`release` |
| `OCI_RELEASE_VERSION` | `string` | 否 | 空字符串 | `OCI_PUBLISH_CHANNEL=release` 时必填，格式 `1.2.3` 或 `1.2.3-rc.1`（可带 `v` 前缀） |
| `OCI_PUBLISH_LATEST` | `boolean` | 否 | `false` | 仅在稳定版 release（`x.y.z`）时可额外产出 `latest` |
| `OCI_GATE_TIMEOUT_SECONDS` | `number` | 否 | `60` | 启动门禁/健康检查超时秒数 |
| `OCI_GATE_RUN_ARGS` | `string` | 否 | 空字符串 | 启动检查时附加的 `docker run` 参数 |
| `OCI_GATE_HEALTH_URL` | `string` | 否 | 空字符串 | 可选健康检查 URL，非空时执行 HTTP 检查 |
| `OCI_GATE_SMOKE_CMD` | `string` | 否 | 空字符串 | 可选 smoke 命令，非空时在容器内执行 |
| `OCI_GATE_VULN_SEVERITY` | `string` | 否 | `CRITICAL,HIGH` | Trivy 漏洞门禁严重级别（逗号分隔） |
| `OCI_GATE_VULN_IGNORE_UNFIXED` | `boolean` | 否 | `false` | Trivy 扫描是否忽略未修复漏洞 |

## 3. Outputs 说明

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `image_ref` | `string` | 不带 tag 的完整镜像引用，格式 `<registry>/<image_name>` |
| `image_tags` | `string` | 逗号分隔的 tag 列表（仅 tag 名，不含镜像前缀） |
| `digest` | `string` | `Build and push image` 步骤输出的 digest（来自 `docker/build-push-action`） |
| `image_with_digest` | `string` | 当 `digest` 非空时为 `<registry>/<image_name>@sha256:...`，否则为空字符串 |

调用方可直接消费 outputs（包含 `image_with_digest`）：

```yaml
jobs:
  docker-publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@main
    with:
      OCI_REGISTRY: ${{ vars.NEXUS_REGISTRY }}
      OCI_IMAGE_NAME: ${{ vars.IMAGE_NAME }}
  deploy:
    needs: docker-publish
    runs-on: ubuntu-latest
    steps:
      - name: Print image digest ref
        run: echo "${{ needs.docker-publish.outputs.image_with_digest }}"
```

## 4. Secrets 说明

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `nexus_username` | 否 | Registry 登录用户名；当 `OCI_PUSH_IMAGE=true` 时应提供 |
| `nexus_password` | 否 | Registry 登录密码；当 `OCI_PUSH_IMAGE=true` 时应提供 |

## 5. 门禁步骤（硬切规则）

该工作流采用“中环 + 外环”门禁基座，规则全部由 `OCI_*` 入参显式驱动，不再按分支名或 Git tag 自动推断。

步骤与门禁如下：

1. 外环发布前置：Tag 解析门禁（`Resolve image tags`）：
   - 固定生成 `sha-<commit前12位>`。
   - `OCI_PUBLISH_CHANNEL` 仅允许 `none`/`edge`/`release`。
   - `OCI_PUBLISH_CHANNEL=release` 时必须提供 `OCI_RELEASE_VERSION` 并通过正则校验。
   - 稳定版 release（`x.y.z`）才可能追加 `latest`（同时要求 `OCI_PUBLISH_LATEST=true`）。
2. 外环发布：`OCI_PUSH_IMAGE=true` 时执行 `Login to Nexus registry` 与 `Build and push image`。
3. 输出收敛：`Resolve digest reference` 生成 `image_with_digest`（空 digest 时输出空字符串）。
4. 中环门禁触发条件：
   - 仅当 `OCI_PUSH_IMAGE=true` 且 `image_with_digest` 非空时，才运行 `OCI Middle-Ring Gate`（`image_gate` job）。
5. 镜像启动门禁（`Startup gate`）：
   - 先按 digest 拉取镜像，再执行 `docker run`，容器必须在超时内进入 `running` 状态。
   - 超时由 `OCI_GATE_TIMEOUT_SECONDS` 控制。
   - 可通过 `OCI_GATE_RUN_ARGS` 传入启动参数。
6. 可选健康检查门禁（`Optional health URL check`）：
   - `OCI_GATE_HEALTH_URL` 非空时执行 HTTP 健康检查。
7. 可选 smoke 门禁（`Optional smoke command`）：
   - `OCI_GATE_SMOKE_CMD` 非空时，在容器内执行 smoke 命令。
8. 漏洞扫描门禁（`Vulnerability scan`）：
   - 使用 Trivy 扫描 `image_with_digest`。
   - `OCI_GATE_VULN_SEVERITY` 控制失败阈值，`OCI_GATE_VULN_IGNORE_UNFIXED` 控制是否忽略未修复漏洞。
9. 中环收尾：`Cleanup gate container` 在结束时清理容器（`always()` 条件）。

## 6. 标签示例（仅示意）

设 `image_ref=nexus.example.com/team/app`，`sha=abcdef123456`：

- `OCI_PUBLISH_CHANNEL=none`  
  产出：`sha-abcdef123456`
- `OCI_PUBLISH_CHANNEL=edge`  
  产出：`sha-abcdef123456,edge`
- `OCI_PUBLISH_CHANNEL=release` + `OCI_RELEASE_VERSION=1.2.3-rc.1` + `OCI_PUBLISH_LATEST=true`  
  产出：`sha-abcdef123456,1.2.3-rc.1`
- `OCI_PUBLISH_CHANNEL=release` + `OCI_RELEASE_VERSION=v1.2.3` + `OCI_PUBLISH_LATEST=true`  
  产出：`sha-abcdef123456,1.2.3,1.2,1,latest`

## 7. Caller Workflow 完整示例

```yaml
name: Build and Publish

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  docker-publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@main
    with:
      OCI_REGISTRY: ${{ vars.NEXUS_REGISTRY }}
      OCI_IMAGE_NAME: ${{ vars.IMAGE_NAME }}
      OCI_DOCKERFILE_PATH: Dockerfile
      OCI_BUILD_CONTEXT: .
      OCI_TARGET_PLATFORMS: linux/amd64
      OCI_BUILD_ARGS: |
        APP_ENV=prod
      OCI_PUSH_IMAGE: true
      OCI_PUBLISH_CHANNEL: release
      OCI_RELEASE_VERSION: ${{ github.ref_name }}
      OCI_PUBLISH_LATEST: true
      OCI_GATE_TIMEOUT_SECONDS: 60
      OCI_GATE_RUN_ARGS: ""
      OCI_GATE_HEALTH_URL: ""
      OCI_GATE_SMOKE_CMD: ""
      OCI_GATE_VULN_SEVERITY: HIGH,CRITICAL
      OCI_GATE_VULN_IGNORE_UNFIXED: false
    secrets:
      nexus_username: ${{ secrets.NEXUS_USERNAME }}
      nexus_password: ${{ secrets.NEXUS_PASSWORD }}
```

## 8. 变量与密钥配置建议（调用方仓库）

在调用方仓库配置：

- Repository Variables：
  - `NEXUS_REGISTRY`：例如 `nexus.example.com`
  - `IMAGE_NAME`：例如 `team/app`
- Repository Secrets：
  - `NEXUS_USERNAME`
  - `NEXUS_PASSWORD`

然后在 caller workflow 里映射到 reusable workflow 的 `with` / `secrets`。

## 9. Fork Sync Reusable

文件：`.github/workflows/fork-sync.reusable.yml`  
用途：将 fork 的目标分支与上游分支做同步；默认使用 `--ff-only`，也支持显式强制重置。

### Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `target_branch` | `string` | 否 | `main` | 当前仓库要同步的目标分支 |
| `upstream_repo` | `string` | 是 | 无 | 上游仓库，格式 `owner/repo` |
| `upstream_branch` | `string` | 否 | `main` | 上游分支 |
| `force_reset` | `boolean` | 否 | `false` | 是否强制把目标分支重置到上游分支 |

### Outputs 说明

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `needs_sync` | `string` | 本地与上游 SHA 是否不同（`true/false`） |
| `local_sha` | `string` | 本地目标分支 SHA |
| `upstream_sha` | `string` | 上游分支 SHA |
| `synced` | `string` | 是否执行并完成同步推送（`true/false`） |

### 调用示例（caller）

```yaml
jobs:
  sync-main:
    uses: <owner>/workflow-reusable/.github/workflows/fork-sync.reusable.yml@main
    with:
      target_branch: main
      upstream_repo: upstream-owner/upstream-repo
      upstream_branch: main
      force_reset: false
    secrets:
      GH_APP_ID: ${{ secrets.GH_APP_ID }}
      GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}
      RUNNER_PROBE_TOKEN: ${{ secrets.RUNNER_PROBE_TOKEN }}
```

## 10. Branch Sync PR Reusable

文件：`.github/workflows/branch-sync-pr.reusable.yml`  
用途：比较 `target..source` 提交差异；有新增时检查是否已有 open PR，无则通过 REST API 创建 PR。

### Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `source_branch` | `string` | 否 | `main` | PR head 分支 |
| `target_branch` | `string` | 否 | `custom/docker` | PR base 分支 |
| `pr_title` | `string` | 否 | `chore(sync): merge main into custom/docker` | PR 标题 |
| `pr_body` | `string` | 否 | 自动说明文本 | PR 描述 |
| `auto_resolve_theirs_paths` | `string` | 否 | 空字符串 | 多行路径列表；发生冲突时这些路径自动采用 source(theirs) |

### Outputs 说明

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `has_new_commits` | `string` | source 是否有 target 没有的新提交 |
| `pr_exists` | `string` | 是否已存在 open PR（base=target, head=`owner:source`） |
| `pr_created` | `string` | 本次是否新建 PR |
| `pr_url` | `string` | 已有或新建 PR 的 URL |

### 调用示例（caller）

```yaml
jobs:
  open-sync-pr:
    uses: <owner>/workflow-reusable/.github/workflows/branch-sync-pr.reusable.yml@main
    with:
      source_branch: main
      target_branch: custom/docker
      pr_title: "chore(sync): merge main into custom/docker"
      pr_body: "自动同步分支变更并创建 PR。"
      auto_resolve_theirs_paths: ""
    secrets:
      GH_APP_ID: ${{ secrets.GH_APP_ID }}
      GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}
      RUNNER_PROBE_TOKEN: ${{ secrets.RUNNER_PROBE_TOKEN }}
```

## 11. Runner Fallback Reusable

文件：`.github/workflows/runner-fallback.reusable.yml`  
用途：先探测 `self-hosted` runner 可用性；可用则走 `self-hosted`，不可用则按策略回退到 `github-hosted`。  
变更策略：`RUNNER_*` 输入已硬切，不提供旧字段向后兼容。

### Inputs 说明

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

### Outputs 说明

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

### Secrets 说明

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `runner_probe_token` | 否 | 用于调用 runner API 的 token；未提供时回退使用 `${{ github.token }}`。组织级探测建议显式提供具有 `admin:org` 的 token |

### 调用示例（caller：先探测，再分 self-hosted / github-hosted 两个 job）

```yaml
name: CI With Runner Fallback

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  actions: read

jobs:
  runner-probe:
    uses: <owner>/workflow-reusable/.github/workflows/runner-fallback.reusable.yml@main
    with:
      RUNNER_SELF_HOSTED_LABELS: self-hosted,linux,x64,ci-main
      RUNNER_GITHUB_HOSTED_LABEL: ubuntu-22.04
      RUNNER_PROBE_SCOPE: auto
      RUNNER_PROBE_REQUIRE_ONLINE: true
      RUNNER_PROBE_REQUIRE_IDLE: true
      RUNNER_MIN_MATCH_COUNT: 1
      RUNNER_FALLBACK_POLICY: github-hosted
    secrets:
      runner_probe_token: ${{ secrets.RUNNER_PROBE_TOKEN }}

  build-self-hosted:
    name: Build on self-hosted
    needs: runner-probe
    if: ${{ needs.runner-probe.outputs.use_self_hosted == 'true' }}
    runs-on: [self-hosted, linux, x64, ci-main]
    steps:
      - name: Checkout
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
      - name: Build
        run: |
          set -euo pipefail
          echo "Build on self-hosted runner"

  build-github-hosted:
    name: Build on github-hosted
    needs: runner-probe
    if: ${{ needs.runner-probe.outputs.use_github_hosted == 'true' }}
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
      - name: Build
        run: |
          set -euo pipefail
          echo "Build on github-hosted runner"
```

生产环境建议把 `@main` 换成受控版本 tag 或 commit SHA（例如 `@v1` 或 `@<40位SHA>`）。

### 组织级最佳实践

- 统一标签体系：组织内约定 `self-hosted` 基础标签与业务标签（如 `ci-main`/`build`/`deploy`）。
- 回退策略分级：普通 CI 用 `github-hosted` 回退；关键发布链路可设 `RUNNER_FALLBACK_POLICY=fail` 防止静默降级。
- 观测与审计：在调用方记录 `selected_runner`、`probe_reason`、`probe_summary` 便于容量规划和故障复盘。
- 权限最小化：`runner_probe_token` 仅授予探测所需权限；组织探测场景推荐组织级 secret 统一分发。
