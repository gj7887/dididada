# GitHub Actions 工作流

本目录包含用于自动化构建 Docker 镜像的工作流文件。

## 工作流文件

### 1. `docker-build.yml`
分别构建多个架构的 Docker 镜像。

**触发条件:**
- Push to main
- Pull request to main
- Release created

**功能:**
- 分别构建 linux/amd64 和 linux/arm64
- 仅构建，不推送到 Docker Hub
- 自动生成镜像标签

### 2. `docker-build-multiarch.yml` (推荐)
同时构建多架构 Docker 镜像（Manifest）。

**触发条件:**
- Push to main
- Pull request to main
- Release created
- 手动触发 (workflow_dispatch)

**功能:**
- 一次性构建多架构镜像
- 创建 Manifest 列表
- 仅构建，不推送到 Docker Hub
- 自动生成镜像标签

## 配置要求

当前工作流配置为 **仅构建，不推送**，因此：

- ✅ 不需要配置 Docker Hub 凭证
- ✅ 不需要配置 Secrets
- ✅ 可以直接使用

## 使用方法

1. 推送代码到 main 分支或创建 Pull Request
2. 工作流将自动运行并构建镜像
3. 在 GitHub Actions 中查看构建结果

## 手动触发

如需手动触发构建，请使用 `docker-build-multiarch.yml` 工作流：

1. 进入 GitHub 仓库的 Actions 标签
2. 选择 "Build and Push Multi-Arch Docker Image"
3. 点击 "Run workflow"
4. 选择分支并点击 "Run workflow"

## 本地使用镜像

工作流只构建镜像，不会推送到 Docker Hub。如需使用镜像，请本地构建：

```bash
# 构建 amd64 版本
docker build -f Dockerfile.build \
  --build-arg TARGETOS=linux \
  --build-arg TARGETARCH=amd64 \
  -t h-ui:local .

# 构建后运行
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  --network=host \
  -v /h-ui/bin:/h-ui/bin \
  -v /h-ui/data:/h-ui/data \
  -v /h-ui/export:/h-ui/export \
  -v /h-ui/logs:/h-ui/logs \
  h-ui:local
```

## 如需启用推送功能

如果以后需要推送到 Docker Hub，需要：

1. 配置 GitHub Secrets:
   - `DOCKER_USERNAME`: Docker Hub 用户名
   - `DOCKER_PASSWORD`: Docker Hub 密码或访问令牌

2. 修改工作流文件，取消 `if: false` 和将 `push: false` 改为 `push: true`

## 详细文档

关于 Docker 构建的更多详细信息，请参考: [DOCKER_BUILD.md](../../docs/DOCKER_BUILD.md)
