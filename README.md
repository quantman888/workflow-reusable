# workflow-reusable

该仓库提供可复用的 Docker 镜像发布工作流：

- 文件：`.github/workflows/docker-publish.reusable.yml`
- 名称：`Docker Publish Reusable`
- 触发方式：`workflow_call`（供其他仓库复用）和 `workflow_dispatch`（本仓库手动触发）

## 1. 调用方式（caller）

在调用方仓库的 workflow 中通过 `uses` 引用：

```yaml
jobs:
  publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@main
    with:
      registry: ${{ vars.NEXUS_REGISTRY }}
      image_name: ${{ vars.IMAGE_NAME }}
      dockerfile_path: Dockerfile
      build_context: .
      target_platforms: linux/amd64
      build_args: |
        APP_ENV=prod
      publish_latest: true
    secrets:
      nexus_username: ${{ secrets.NEXUS_USERNAME }}
      nexus_password: ${{ secrets.NEXUS_PASSWORD }}
```

生产环境建议把 `@main` 换成受控版本 tag 或 commit SHA（例如 `@v1` 或 `@<40位SHA>`）。

## 2. Inputs 说明

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `registry` | `string` | 是 | 无 | Docker Registry 地址，例如 `nexus.example.com` |
| `image_name` | `string` | 是 | 无 | 镜像名，例如 `team/app` |
| `dockerfile_path` | `string` | 否 | `Dockerfile` | Dockerfile 路径 |
| `build_context` | `string` | 否 | `.` | Docker build context |
| `target_platforms` | `string` | 否 | `linux/amd64` | Buildx 平台列表 |
| `build_args` | `string` | 否 | 空字符串 | 多行 `KEY=VALUE` |
| `publish_latest` | `boolean` | 否 | `true` | 是否在 `main`/`workflow_dispatch` 产出 `latest` |

## 3. Secrets 说明

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `nexus_username` | 是 | Registry 登录用户名 |
| `nexus_password` | 是 | Registry 登录密码 |

## 4. 变量与密钥配置建议（调用方仓库）

在调用方仓库配置：

- Repository Variables：
  - `NEXUS_REGISTRY`：例如 `nexus.example.com`
  - `IMAGE_NAME`：例如 `team/app`
- Repository Secrets：
  - `NEXUS_USERNAME`
  - `NEXUS_PASSWORD`

然后在 caller workflow 里映射到 reusable workflow 的 `with` / `secrets`。

## 5. 标签策略

当前实现的 tag 生成规则：

- 所有场景都会产出：`sha-<commit前7位>`
- `main` 分支（`refs/heads/main`）且 `publish_latest=true`：额外产出 `latest`
- 语义版本 tag（`refs/tags/v*`）：额外产出对应版本 tag（例如 `v1.2.3`）
- `workflow_dispatch` 且 `publish_latest=true`：额外产出 `latest`

对齐后的关键策略：

- `main` => `latest` + `sha-*`
- `tag v*` => `v*` + `sha-*`
- `workflow_dispatch`（默认 `publish_latest=true`）=> `latest` + `sha-*`

## 6. Caller Workflow 完整示例

```yaml
name: Build and Publish

on:
  push:
    branches:
      - main
    tags:
      - 'v*'
  workflow_dispatch:

jobs:
  docker-publish:
    uses: <owner>/workflow-reusable/.github/workflows/docker-publish.reusable.yml@main
    with:
      registry: ${{ vars.NEXUS_REGISTRY }}
      image_name: ${{ vars.IMAGE_NAME }}
      dockerfile_path: Dockerfile
      build_context: .
      target_platforms: linux/amd64
      build_args: |
        APP_ENV=prod
      publish_latest: true
    secrets:
      nexus_username: ${{ secrets.NEXUS_USERNAME }}
      nexus_password: ${{ secrets.NEXUS_PASSWORD }}
```

## 7. Fork Sync Reusable

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

## 8. Branch Sync PR Reusable

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
