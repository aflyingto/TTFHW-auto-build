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