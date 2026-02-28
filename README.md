# TTFHW-auto-build

自动化构建工作流，用于构建 MindIE-SD wheel 包。

## 功能特性

- 自动化构建 MindIE-SD wheel 包
- Docker 环境依赖检查
- Docker 镜像缓存优化
- 构建时长统计（镜像拉取、代码下载、构建、制品归档）
- 支持多种触发方式（push、pull_request、手动触发）

## 触发条件

工作流会在以下情况下自动触发：

- Push 到 `main` 或 `master` 分支，且修改了以下文件：
  - `.github/workflows/build-mindie-sd.yml`
  - `build/**`
  - `src/**`
  - `setup.py`
  - `pyproject.toml`
- 针对上述文件的 Pull Request
- 手动触发（workflow_dispatch）

## 构建环境

使用华为云镜像：
```
swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311
```

## 使用方法

1. Fork 或克隆本仓库
2. 推送代码到 `main` 或 `master` 分支
3. 工作流将自动触发构建
4. 构建完成后，wheel 包将作为 artifacts 下载

## 构建统计

工作流会自动统计并显示各阶段的构建时长：

- Image Pull - 镜像拉取时长
- Code Download - 代码下载时长
- Build - 构建时长
- Artifact Upload - 制品归档时长
- Total - 总时长

## 常见问题与解决方案

### Docker 镜像缓存失效问题

**问题描述：**
每次 workflow 运行都重新拉取 Docker 镜像，缓存没有生效。

**原因分析：**
1. **缓存键不稳定**：原缓存键基于 workflow 文件的 hash，每次修改 workflow 文件都会改变缓存键
2. **缓存命中检测错误**：没有正确使用 `cache-hit` 输出来判断缓存状态
3. **频繁的 workflow 修改**：由于持续优化 workflow，缓存键不断变化

**解决方案：**
使用固定的缓存键，基于镜像名称而不是 workflow 文件：
```yaml
- name: Cache Docker image
  id: cache_image
  uses: actions/cache@v4
  with:
    path: /tmp/docker-image.tar
    key: docker-image-ascend-mindie-ubuntu-x86-20260119
    restore-keys: |
      docker-image-ascend-mindie-ubuntu-x86-

- name: Pull Docker image
  id: pull_image
  run: |
    echo "Cache hit status: ${{ steps.cache_image.outputs.cache-hit }}"
    
    if [ "${{ steps.cache_image.outputs.cache-hit }}" == "true" ] && [ -f /tmp/docker-image.tar ]; then
      echo "Loading Docker image from cache..."
      docker load -i /tmp/docker-image.tar
    else
      echo "Pulling Docker image from registry..."
      docker pull $IMAGE_NAME
      docker save $IMAGE_NAME -o /tmp/docker-image.tar
    fi
```

**说明：**
- 使用固定的缓存键，确保镜像不变时缓存有效
- 正确使用 `cache-hit` 输出来判断缓存状态
- 添加调试信息便于排查缓存问题

### 1. op_gen 模块找不到错误

**错误信息：**
```
ModuleNotFoundError: No module named 'op_gen'
/workspace/mindie-sd/build/build_ascendc_ops.sh: line 83: pop_var_context: head of shell_variables not a function context
```

**原因分析：**
- 构建脚本需要访问 Ascend 工具链中的 op_gen 模块
- Docker 容器内缺少正确的 PYTHONPATH 环境变量
- Python 无法找到 `/usr/local/Ascend/ascend-toolkit/latest/python/site/packages` 中的模块
- 未正确设置 Ascend 工具链环境

**解决方案：**
在构建前 source Ascend 环境设置脚本：
```yaml
docker run --rm \
  swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311 \
  bash -c "source /usr/local/Ascend/ascend-toolkit/latest/set_env.sh && bash build/build.sh"
```

**说明：**
- `set_env.sh` 脚本会自动设置所有必要的环境变量
- 包括 PYTHONPATH、ASCEND_HOME、LD_LIBRARY_PATH 等
- 比手动设置环境变量更可靠和完整

### 2. 文件权限错误

**错误信息：**
```
Error: -28 16:42:17 (58) - [ERROR] You are not the owner of path /workspace/mindie-sd/build/ir_demo.json
/workspace/mindie-sd/build/build_ascendc_ops.sh: line 83: pop_var_context: head of shell_variables not a function context
Error: Process completed with exit code 101
```

**原因分析：**
- Docker 容器内外文件所有者不匹配
- 容器内默认以 root 用户运行，而宿主机上的文件由其他用户创建
- 导致容器内进程无法访问或修改宿主机挂载的文件

**解决方案：**
使用 `-u` 参数指定容器内的用户ID和组ID：
```yaml
docker run --rm \
  -v ${{ github.workspace }}:/workspace \
  -u $(id -u):$(id -g) \
  swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311 \
  bash -c "source /usr/local/Ascend/ascend-toolkit/latest/set_env.sh && bash build/build.sh"
```

**说明：**
- `-u $(id -u):$(id -g)` 让容器内进程以宿主机用户身份运行
- 避免文件所有者不匹配的问题
- 确保容器内进程可以正常访问和修改挂载的文件

### 3. 必要的环境变量设置

**环境变量：**
- `TORCH_DEVICE_BACKEND_AUTOLOAD=0` - 禁用 Torch 设备后端自动加载
- `USER_ABI_VERSION=1` - 设置用户 ABI 版本

**原因分析：**
- MindIE-SD 构建过程需要特定的环境配置
- 禁用自动加载可以避免与 Ascend 工具链的冲突
- ABI 版本设置确保与 CANN 工具链的兼容性

**解决方案：**
在构建前设置必要的环境变量：
```yaml
docker run --rm \
  swr.cn-north-4.myhuaweicloud.com/inference/ascend_mindie_ubuntu_x86:20260119_ubuntu24_3.0.0_cann8.5.0_torch2.6.0_py311 \
  bash -c "source /usr/local/Ascend/ascend-toolkit/latest/set_env.sh && export TORCH_DEVICE_BACKEND_AUTOLOAD=0 && export USER_ABI_VERSION=1 && bash build/build.sh"
```

**说明：**
- 这些环境变量必须在 source set_env.sh 之后设置
- 确保与 MindIE-SD 和 CANN 工具链的正确集成
- 避免运行时 ABI 不兼容问题

### 2. 时间计算不一致问题

**问题描述：**
- upload 时间计算使用了临时文件方式，与其他步骤的计算逻辑不一致
- 代码中存在逻辑错误：`echo ${{ steps.upload_start.outputs.START_TIME }}` 无法获取到值

**原因分析：**
- 在 `upload_start` 步骤中引用自身的输出是无效的
- 使用临时文件增加了不必要的复杂性和潜在错误

**解决方案：**
统一所有步骤的时间计算逻辑：
```yaml
- name: Start upload timer
  id: upload_start
  run: |
    echo "START_TIME=$(date +%s)" >> $GITHUB_OUTPUT

- name: End upload timer
  id: upload_end
  run: |
    echo "END_TIME=$(date +%s)" >> $GITHUB_OUTPUT

- name: Calculate time
  run: |
    UPLOAD_TIME=$(( ${{ steps.upload_end.outputs.END_TIME }} - ${{ steps.upload_start.outputs.START_TIME }} ))
```

## 许可证

请参考 MindIE-SD 项目的许可证。