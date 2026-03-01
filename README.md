# workflow-reusable

该仓库提供可复用的 Docker 镜像发布工作流：

- 文件：`.github/workflows/docker-publish.reusable.yml`
- 名称：`Docker Publish Reusable Workflow`
- 触发方式：`workflow_call`（供其他仓库复用）和 `workflow_dispatch`（本仓库手动触发）
- 变更策略：`OCI_*` 输入已硬切，不提供旧字段向后兼容

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
用途：将 fork 的目标分支与上游分支做快进同步（仅允许 `--ff-only`）。

### Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `target_branch` | `string` | 否 | `main` | 当前仓库要同步的目标分支 |
| `upstream_repo` | `string` | 是 | 无 | 上游仓库，格式 `owner/repo` |
| `upstream_branch` | `string` | 否 | `main` | 上游分支 |

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
```

## 10. Branch Sync PR Reusable

文件：`.github/workflows/branch-sync-pr.reusable.yml`  
用途：比较 `target..source` 提交差异；有新增时检查是否已有 open PR，无则自动创建。

### Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `source_branch` | `string` | 否 | `main` | PR head 分支 |
| `target_branch` | `string` | 否 | `custom/docker` | PR base 分支 |
| `pr_title` | `string` | 否 | `chore(sync): merge main into custom/docker` | PR 标题 |
| `pr_body` | `string` | 否 | 自动说明文本 | PR 描述 |

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
```
