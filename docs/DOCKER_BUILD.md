# Docker 构建指南

本项目使用 GitHub Actions 自动构建 Docker 镜像（仅构建，不推送）。

## 配置步骤

### 工作流说明

项目包含两个工作流文件：

#### `docker-build.yml` - 分别构建多个架构

- 分别构建 `linux/amd64` 和 `linux/arm64` 架构
- 适用于需要单独控制每个架构构建的场景
- 仅构建，不推送到 Docker Hub

#### `docker-build-multiarch.yml` - 同时构建多架构镜像

- 一次性构建 `linux/amd64` 和 `linux/arm64` 架构
- 创建真正的多架构镜像（Manifest）
- 推荐使用此工作流
- 仅构建，不推送到 Docker Hub

## 触发条件

工作流在以下情况下自动触发：

- **推送到 main 分支**: 构建镜像，不推送
- **Pull Request**: 构建镜像，不推送（用于测试）
- **创建 Release**: 根据 release 标签构建镜像（如 `v1.0.0`, `1.0`, `1`），不推送
- **手动触发** (仅 multiarch): 通过 GitHub Actions 界面手动运行

## 工作流特性

当前工作流配置为 **仅构建，不推送**。这意味着：

- ✅ GitHub Actions 会自动构建 Docker 镜像
- ✅ 支持多架构（linux/amd64, linux/arm64）
- ✅ 使用 GitHub Actions 缓存加速构建
- ❌ 不会推送到 Docker Hub
- ❌ 不需要配置 Docker Hub 凭证

### 如需启用推送功能

如果以后需要推送到 Docker Hub，请按以下步骤操作：

1. 在 GitHub 仓库配置 Secrets：
   - `DOCKER_USERNAME`: Docker Hub 用户名
   - `DOCKER_PASSWORD`: Docker Hub 密码或访问令牌

2. 修改工作流文件：
   - 将 `if: false` 从 Login to Docker Hub 步骤中删除
   - 将 `push: false` 改为 `push: ${{ github.event_name != 'pull_request' }}`

## 镜像标签策略

| 事件类型 | 构建操作 |
|---------|---------|
| push to main | 构建镜像（不推送） |
| pull request | 构建镜像（不推送） |
| release v1.2.3 | 构建镜像（不推送） |
| 手动触发 (multiarch) | 构建镜像（不推送） |

## 使用镜像

### 本地构建镜像

工作流只构建镜像，不推送到 Docker Hub。如需使用镜像，请按以下方式本地构建：

#### 方法 1: 使用工作流构建的镜像

在 GitHub Actions 中构建完成后，可以下载构建的镜像文件，然后在本地加载：

```bash
# 假设已下载镜像文件
docker load -i h-ui-image.tar
docker tag h-ui h-ui:local
```

#### 方法 2: 本地构建 Dockerfile.ci（CI 优化的多阶段构建）

**注意**: Dockerfile.ci 会自动构建前端（Vue）和后端（Go）。

```bash
# 构建 amd64 版本
docker build -f Dockerfile.ci \
  --build-arg TARGETOS=linux \
  --build-arg TARGETARCH=amd64 \
  -t h-ui:local .

# 构建 arm64 版本
docker build -f Dockerfile.ci \
  --build-arg TARGETOS=linux \
  --build-arg TARGETARCH=arm64 \
  -t h-ui:local-arm64 .
```

构建时间较长（需要构建前端），请耐心等待。

#### 方法 3: 手动构建二进制文件

如果需要先构建二进制文件再打包 Docker 镜像：

### 手动构建二进制文件

```bash
# 使用项目提供的构建脚本
./build.sh          # Linux/Mac
# 或
build.bat           # Windows

# 然后使用原始 Dockerfile
docker build -t h-ui:local .
```

## 故障排除

### 构建失败

1. 检查 GitHub Actions 日志
2. 验证工作流文件语法
3. 检查是否有构建权限问题
4. 确保前端构建成功（Dockerfile 会自动构建前端）

### 前端构建问题

如果前端构建失败：

1. 检查 `frontend/package.json` 中的依赖是否正确
2. 确保 Node.js 版本符合要求（>=18.12.0）
3. 查看构建日志中的 npm 错误信息
4. 可以尝试在本地运行 `cd frontend && npm run build:prod` 测试

### Go 构建问题

如果 Go 编译失败：

1. 检查 `go.mod` 文件是否存在
2. 确保依赖已正确下载
3. 检查是否有编译错误
4. 确保前端 dist 目录已正确生成（frontend/embed.go 需要它）

### 多架构镜像问题

1. 确保 QEMU 模拟器正确设置
2. 验证目标平台是否支持（当前支持 linux/amd64 和 linux/arm64）
3. 检查构建日志中是否所有架构都成功构建
4. 注意：前端构建在多架构环境下可能需要更多时间

## 最佳实践

1. **使用多架构工作流**: 推荐 `docker-build-multiarch.yml`，它更高效且符合 Docker 最佳实践
2. **缓存利用**: 工作流已启用 GitHub Actions 缓存，加快后续构建速度
3. **版本标签**: 正式发布时使用 Git tag 创建 release，自动生成语义化版本标签
4. **本地使用**: 推荐本地使用 Dockerfile.ci 构建镜像（已包含前端构建）
5. **构建时间**: Dockerfile 需要构建前端和后端，首次构建时间较长（约 5-10 分钟）

## 架构支持

当前支持以下架构：

- `linux/amd64` - 标准 x86_64 架构
- `linux/arm64` - ARM 64位架构（如 Apple Silicon, ARM 服务器）

如需支持其他架构，可以在工作流文件中添加相应的平台。
