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

## 许可证

请参考 MindIE-SD 项目的许可证。