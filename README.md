# Android Build Workflows

这个项目包含了用于构建Android应用的GitHub Actions工作流。

## 工作流结构

### 可重用工作流

- **`android-build-reusable.yml`**: 核心的可重用工作流，包含所有构建逻辑
  - 支持不同的应用类型
  - 自动选择正确的JDK版本
  - 支持构建类型选择（release/debug）
  - 支持上传到GitHub Release和百度网盘
  - 包含完整的构建、打包和上传流程

### 调用工作流

- **`android-build-dc.yml`**: DC应用构建工作流
- **`android-build-lc.yml`**: LC应用构建工作流

这两个工作流都调用可重用工作流，只需要传递不同的 `app_type` 参数。

## 优化效果

### 代码减少
- 原来每个工作流约288行代码
- 优化后每个调用工作流只有约40行代码
- 代码重复率从100%降低到约14%

### 维护优势
1. **单一真实来源**: 所有构建逻辑集中在可重用工作流中
2. **易于维护**: 修改构建逻辑只需要更新一个文件
3. **一致性**: 确保所有应用使用相同的构建流程
4. **扩展性**: 添加新的应用类型只需要创建简单的调用工作流

## 使用方法

### 手动触发构建

1. 进入GitHub仓库的Actions页面
2. 选择对应的工作流
3. 点击"Run workflow"
4. 配置构建参数：
   - **构建类型**: release 或 debug
   - **上传到GitHub Release**: 是否上传到Release（仅限release构建）
   - **上传到百度网盘**: 是否上传到百度网盘
   - **启用详细日志**: 为百度网盘上传启用详细日志

### 添加新的应用类型

如果需要添加新的应用类型（例如"XY"），只需要：

1. 创建新的工作流文件 `android-build-xy.yml`：

```yaml
name: Android Build For XY

on:
  workflow_dispatch:
    inputs:
      build_type:
        description: '选择构建类型 (release 或 debug)'
        type: choice
        options:
          - release
          - debug
        default: 'release'
      upload_to_release:
        description: '上传到 GitHub Release (仅限 Release 构建)'
        type: boolean
        default: false
      upload_to_baidu:
        description: '上传到百度网盘'
        type: boolean
        default: false
      enable_logs:
        description: '为百度网盘上传启用详细日志'
        type: boolean
        default: false

jobs:
  build:
    uses: ./.github/workflows/android-build-reusable.yml
    with:
      java_version: '21'  # 或其他需要的版本
      baidu_target_dir: '/apps/android/xy'
      build_type: ${{ inputs.build_type }}
      upload_to_release: ${{ inputs.upload_to_release }}
      upload_to_baidu: ${{ inputs.upload_to_baidu }}
      enable_logs: ${{ inputs.enable_logs }}
    secrets:
      PRIVATE_REPO_TOKEN: ${{ secrets.PRIVATE_REPO_TOKEN }}
      APP_REPOSITORY: ${{ secrets.REPO_XY }}
      REPO_LIBS: ${{ secrets.REPO_LIBS }}
      BAIDU_BDUSS: ${{ secrets.BAIDU_BDUSS }}
      BAIDU_STOKEN: ${{ secrets.BAIDU_STOKEN }}
```

2. 添加对应的仓库密钥（如 `REPO_XY`）

**优势：** 无需修改可重用工作流，每个应用可以独立配置自己的仓库、JDK版本和目标路径。

## 配置要求

### GitHub Secrets

需要配置以下secrets：

- `PRIVATE_REPO_TOKEN`: GitHub个人访问令牌
- `REPO_DC`: DC应用仓库地址
- `REPO_LC`: LC应用仓库地址  
- `REPO_LIBS`: 共享库仓库地址
- `BAIDU_BDUSS`: 百度网盘BDUSS（可选）
- `BAIDU_STOKEN`: 百度网盘STOKEN（可选）

## 构建产物

构建完成后会生成：

1. **APK文件**: 应用安装包
2. **Mapping文件**: ProGuard混淆映射文件（仅release构建）
3. **上传位置**:
   - GitHub Release: `{日期}-{构建类型}` 标签
   - 百度网盘: `/apps/android/{应用类型}/{日期}/{运行编号}/{构建类型}/`

## 缓存策略

- 使用Gradle缓存加速构建
- 缓存按天更新，自动清理旧缓存
- 支持增量构建