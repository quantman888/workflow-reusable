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
