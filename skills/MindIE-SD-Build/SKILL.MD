# MindIE-SD 自动化构建 Workflow 生成 Skill

## Skill 概述

本 Skill 用于自动生成 MindIE-SD 项目的 GitHub Actions workflow，实现自动化构建 wheel 包的完整流水线。

## 功能特性

- 自动生成 GitHub Actions workflow 文件
- Docker 环境配置和依赖检查
- Docker 镜像缓存优化
- 构建时长统计（镜像拉取、构建、制品归档）
- 自动提交代码并推送到远端仓库
- 完整的错误处理和故障排查

## 环境要求

### 必需环境
- Git 工具
- GitHub 仓库访问权限
- Docker（本地测试用）

### Docker 镜像
```
swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311
```

## 配置参数

### 镜像配置
```yaml
DOCKER_IMAGE: swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311
SET_ENV_PATH: /usr/local/Ascend/ascend-toolkit/set_env.sh
```

### 代码仓库
```yaml
REPO_URL: https://gitcode.com/Ascend/MindIE-SD.git
```

### 环境变量
```yaml
TORCH_DEVICE_BACKEND_AUTOLOAD: 0
USER_ABI_VERSION: 1
```

## Workflow 结构

### 触发条件
```yaml
on:
  push:
    branches: [ main, master ]
    paths:
      - '.github/workflows/build-mindie-sd.yml'
      - 'build/**'
      - 'src/**'
      - 'setup.py'
      - 'pyproject.toml'
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:
```

### 构建步骤

1. **Checkout code** - 检出代码
2. **Set up Docker Buildx** - 设置 Docker Buildx
3. **Check Docker environment dependencies** - 检查 Docker 环境
4. **Cache Docker image** - 缓存 Docker 镜像
5. **Pull Docker image** - 拉取 Docker 镜像
6. **Build MindIE-SD wheel package** - 构建 wheel 包
7. **Upload wheel artifacts** - 上传构建产物
8. **Calculate and display build statistics** - 显示构建统计

## 核心实现

### Docker 构建命令
```bash
docker run --rm \
  -v ${{ github.workspace }}:/workspace \
  swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311 \
  bash -c "cd /workspace && git clone https://gitcode.com/Ascend/MindIE-SD.git mindie-sd && cd mindie-sd && source /usr/local/Ascend/ascend-toolkit/set_env.sh && export TORCH_DEVICE_BACKEND_AUTOLOAD=0 && export USER_ABI_VERSION=1 && python setup.py bdist_wheel && ls -lh dist/"
```

### 镜像缓存策略
```yaml
- name: Cache Docker image
  uses: actions/cache@v4
  with:
    path: /tmp/docker-image.tar
    key: docker-image-ascend-mindie-ubuntu-x86-20260119
    restore-keys: |
      docker-image-ascend-mindie-ubuntu-x86-
```

## 常见问题与解决方案

### 1. op_gen 模块找不到错误

**错误信息：**
```
ModuleNotFoundError: No module named 'op_gen'
```

**解决方案：**
使用 `set_env.sh` 脚本加载 Ascend 工具链环境：
```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
```

### 2. 文件权限错误

**错误信息：**
```
Error: You are not the owner of path /workspace/mindie-sd/build/ir_demo.json
```

**解决方案：**
在 Docker 容器内下载代码，避免宿主机和容器之间的文件权限冲突：
```bash
docker run --rm \
  -v ${{ github.workspace }}:/workspace \
  swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311 \
  bash -c "cd /workspace && git clone https://gitcode.com/Ascend/MindIE-SD.git mindie-sd && ..."
```

### 3. Docker 镜像缓存失效

**问题：**
每次 workflow 运行都重新拉取 Docker 镜像

**解决方案：**
使用固定的缓存键，基于镜像名称而不是 workflow 文件：
```yaml
key: docker-image-ascend-mindie-ubuntu-x86-20260119
```

## 自动化脚本

### 使用方法
```bash
# 创建目录
mkdir -p /path/to/project/.github/workflows

# 生成 workflow 文件
# （参考上面的核心实现部分）

# 初始化 Git 仓库
cd /path/to/project
git init
git add .
git commit -m "Add MindIE-SD automated build workflow"

# 推送到远端
git remote add origin https://github.com/username/repo.git
git push -u origin main
```

## 文件结构

```
project/
├── .github/
│   └── workflows/
│       └── build-mindie-sd.yml
└── README.md
```

## 构建统计

Workflow 会自动统计并显示各阶段的构建时长：

- **Image Pull** - 镜像拉取时长
- **Build (includes code download)** - 构建时长（包含代码下载）
- **Artifact Upload** - 制品归档时长
- **Total** - 总时长

## 最佳实践

1. **在容器内操作**：所有文件操作都在 Docker 容器内进行，避免权限问题
2. **环境变量配置**：使用 `set_env.sh` 脚本而不是手动设置环境变量
3. **镜像缓存**：使用固定的缓存键确保缓存有效性
4. **错误处理**：每个步骤都有清晰的错误信息和解决方案
5. **文档维护**：及时更新 README 和故障排查指南

## 扩展功能

### 添加额外的构建步骤
```yaml
- name: Custom build step
  run: |
    echo "Custom build operations"
```

### 支持多个分支
```yaml
on:
  push:
    branches: [ main, master, develop ]
```

### 添加通知功能
```yaml
- name: Send notification
  if: always()
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: 'Build completed!'
      })
```

## 维护和更新

### 更新 Docker 镜像
修改 `DOCKER_IMAGE` 变量并提交更改

### 更新环境变量
修改构建命令中的 `export` 语句

### 更新缓存策略
修改缓存键和 restore-keys 配置

## 参考资源

- [MindIE-SD 官方仓库](https://gitcode.com/Ascend/MindIE-SD)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Docker 文档](https://docs.docker.com/)
- [华为云 Ascend 文档](https://www.huawei.com/en/product/ascend)

## 许可证

请参考 MindIE-SD 项目的许可证。