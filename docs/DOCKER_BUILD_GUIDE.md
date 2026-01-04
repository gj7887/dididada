# Docker 构建指南

本项目使用 GitHub Actions 自动构建 Docker 镜像并推送到 GitHub Container Registry (GHCR)。

## 🚀 快速开始

### 1. 启用 GitHub Container Registry

在仓库的 `Settings` -> `Actions` -> `General` -> `Workflow permissions` 中，确保选择了：
- ✅ Read and write permissions

### 2. 推送代码

推送到 `main` 分支或创建 tag 时，工作流会自动构建并推送镜像。

### 3. 拉取镜像

```bash
docker pull ghcr.io/你的用户名/h-ui:latest
```

## 📦 Dockerfile 选择

项目提供两个 Dockerfile：

### Dockerfile（推荐 - Distroless）

**优点：**
- 最小化镜像体积（更小）
- 更高的安全性（无 shell，攻击面小）
- 性能更优

**缺点：**
- 调试困难（无法进入容器）

**适用场景：**
- 生产环境
- 追求安全和性能

### Dockerfile.alpine（开发友好 - Alpine）

**优点：**
- 完整的 Alpine Linux 系统
- 可以使用 bash 调试
- 更容易排查问题

**缺点：**
- 镜像体积稍大
- 安全性略低于 distroless

**适用场景：**
- 开发测试
- 需要调试的场景

## 🔧 配置选项

### 通过 GitHub Variables 配置

在仓库的 `Settings` -> `Secrets and variables` -> `Actions` -> `Variables` 中配置：

| 变量名 | 说明 | 默认值 |
|-------|------|--------|
| `CUSTOM_IMAGE_NAME` | 自定义镜像名称 | `ghcr.io/用户名/仓库名` |
| `CUSTOM_IMAGE_PLATFORMS` | 支持的架构 | `linux/amd64,linux/arm64` |
| `CUSTOM_DOCKER_TAGS` | 自定义标签（空则使用默认） | 空 |
| `CUSTOM_DOCKERFILE` | 自定义 Dockerfile 路径 | `./Dockerfile` |

### 配置示例

#### 使用 Alpine Dockerfile

设置变量：
```
CUSTOM_DOCKERFILE = ./Dockerfile.alpine
```

#### 使用自定义镜像名称

设置变量：
```
CUSTOM_IMAGE_NAME = ghcr.io/myorg/h-ui
```

#### 仅构建 amd64

设置变量：
```
CUSTOM_IMAGE_PLATFORMS = linux/amd64
```

## 📋 标签策略

| 事件类型 | 生成的标签 | 示例 |
|---------|-----------|------|
| Push to main | `main`, `latest` | `ghcr.io/user/repo:latest` |
| Tag v1.2.3 | `v1.2.3`, `1.2.3`, `1.2`, `1`, `latest` | `ghcr.io/user/repo:v1.2.3` |
| Pull Request | `pr-{number}`, `sha-{short-sha}` | `ghcr.io/user/repo:pr-123` |
| Workflow Dispatch | 根据触发事件 | 同上 |

## 🏗️ 构建流程

### 多阶段构建

1. **Frontend Builder** - 构建 Vue 前端
   - 使用 Node.js 18 Alpine
   - 使用 pnpm 安装依赖
   - 构建生产版本

2. **Go Builder** - 编译 Go 后端
   - 使用 Go 1.20 Alpine
   - 嵌入前端静态文件
   - 交叉编译多架构

3. **Runtime** - 运行时镜像
   - Distroless 或 Alpine
   - 最小化镜像体积
   - 安全配置

## 💡 使用镜像

### 基本使用

```bash
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  --network=host \
  -v /h-ui/bin:/h-ui/bin \
  -v /h-ui/data:/h-ui/data \
  -v /h-ui/export:/h-ui/export \
  -v /h-ui/logs:/h-ui/logs \
  ghcr.io/你的用户名/h-ui:latest
```

### 自定义端口

```bash
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  -p 8081:8081 \
  -v /h-ui/data:/h-ui/data \
  -v /h-ui/export:/h-ui/export \
  -v /h-ui/logs:/h-ui/logs \
  ghcr.io/你的用户名/h-ui:latest \
  /h-ui/h-ui -p 8081
```

### 使用特定版本

```bash
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  --network=host \
  -v /h-ui/bin:/h-ui/bin \
  -v /h-ui/data:/h-ui/data \
  -v /h-ui/export:/h-ui/export \
  -v /h-ui/logs:/h-ui/logs \
  ghcr.io/你的用户名/h-ui:v1.2.3
```

### 使用 Alpine 版本（可调试）

```bash
# 拉取 Alpine 版本
docker pull ghcr.io/你的用户名/h-ui:latest
# 然后运行（使用 Alpine Dockerfile 构建）
```

## 🔍 本地构建

### 使用默认 Dockerfile（Distroless）

```bash
# 构建本地镜像
docker build -f Dockerfile -t h-ui:local .

# 运行
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  --network=host \
  -v /h-ui/data:/h-ui/data \
  h-ui:local
```

### 使用 Alpine Dockerfile

```bash
# 构建本地镜像
docker build -f Dockerfile.alpine -t h-ui:local .

# 运行并可以进入容器调试
docker run -d --cap-add=NET_ADMIN \
  --name h-ui --restart always \
  --network=host \
  -v /h-ui/data:/h-ui/data \
  h-ui:local

# 进入容器调试
docker exec -it h-ui sh
```

## 📊 镜像大小对比

| Dockerfile | 预估大小 | 说明 |
|-----------|---------|------|
| Dockerfile (Distroless) | ~15-20 MB | 最小化，生产推荐 |
| Dockerfile.alpine | ~25-30 MB | 包含 bash，便于调试 |
| Dockerfile.ci | ~30-40 MB | 包含更多工具 |

## 🐛 故障排除

### 构建失败

1. **前端构建失败**
   ```bash
   # 本地测试前端构建
   cd frontend
   pnpm install
   pnpm run build:prod
   ```

2. **Go 构建失败**
   ```bash
   # 本地测试 Go 构建
   cd frontend && pnpm run build:prod && cd ..
   go build -o h-ui
   ```

3. **检查工作流日志**
   - 进入 GitHub Actions 页面
   - 查看失败的工作流运行
   - 检查详细错误信息

### 推送失败

1. **权限问题**
   - 检查仓库的 Workflow permissions 设置
   - 确保 `Read and write permissions` 已启用

2. **镜像名称冲突**
   - 检查 `CUSTOM_IMAGE_NAME` 变量设置
   - 确保镜像名称格式正确

### 运行问题

1. **Distroless 镜像无法调试**
   ```bash
   # 使用 Alpine 版本进行调试
   docker build -f Dockerfile.alpine -t h-ui:debug .
   ```

2. **权限问题**
   ```bash
   # 检查文件权限
   docker run --rm -it \
     -v /h-ui/data:/h-ui/data \
     ghcr.io/你的用户名/h-ui:latest \
     sh -c "ls -la /h-ui"
   ```

## 🎯 最佳实践

1. **生产环境使用 Distroless**
   - 更小的攻击面
   - 更好的安全性
   - 更快的部署

2. **开发测试使用 Alpine**
   - 便于调试
   - 包含常用工具

3. **使用语义化版本标签**
   ```bash
   git tag v1.2.3
   git push origin v1.2.3
   ```

4. **定期更新基础镜像**
   - Alpine: 每 3-6 个月
   - Distroless: 跟随更新

5. **启用 GitHub Actions 缓存**
   - 已在工作流中配置
   - 加速后续构建

## 📚 相关链接

- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

## 🔐 安全建议

1. **使用 Distroless 镜像**
   - 减少攻击面
   - 最小化依赖

2. **定期扫描镜像**
   ```bash
   docker scan ghcr.io/你的用户名/h-ui:latest
   ```

3. **使用非 root 用户运行**
   - Dockerfile 中已配置
   - UID:GID = 65532:65532

4. **限制容器权限**
   - 使用 `--cap-drop` 删除不必要的权限
   - 使用 `--read-only` 标志（如适用）

## 📞 支持

如有问题，请：
1. 查看 [Issues](https://github.com/jonssonyan/h-ui/issues)
2. 查看 [Discussions](https://github.com/jonssonyan/h-ui/discussions)
3. 提交新的 Issue
