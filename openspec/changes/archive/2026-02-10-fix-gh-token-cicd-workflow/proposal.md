# Proposal: Fix GH_TOKEN Configuration in CI/CD Workflow

<!-- OPENSPEC:STATUS
Status: ExecutionCompleted
ExecutionDate: 2026-02-10
-->

## Summary

修复 CI/CD 工作流中 GitHub CLI (`gh`) 调用失败的问题。该问题导致 `index.json` 索引文件生成步骤无法正确获取 Release 的发布时间戳，影响应用更新检测功能。

## Problem

在 GitHub Actions 工作流中，`gh release view` 命令失败并显示以下错误：

```
gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable. Example:
  env:
    GH_TOKEN: ${{ github.token }}
Error: Process completed with exit code 4.
```

### Root Cause

经过代码分析，问题定位在 `.github/workflows/sync-azure-storage.yml` 文件的第 154 行：

```yaml
# Get GitHub Release published_at for fallback
echo "Fetching GitHub Release publish time..."
PUBLISHED_AT=$(gh release view "${RELEASE_TAG}" --json publishedAt -q .publishedAt)
```

该步骤调用 `gh release view` 命令获取 Release 的 `publishedAt` 时间戳，但**未设置 `GH_TOKEN` 环境变量**。

对比同文件中的其他 `gh` 命令调用：

| 步骤名称 | 行号 | GH_TOKEN 配置 | 状态 |
|---------|------|--------------|------|
| Determine release tag | 48 | ✅ 有 (第 54 行) | 正常 |
| Download release assets | 63 | ✅ 有 (第 67 行) | 正常 |
| Generate and upload index file | 154 | ❌ **缺失** | **失败** |

### Impact

- CI/CD 工作流在生成 `index.json` 时失败
- 应用更新检测功能无法正常工作
- Azure Storage 同步流程中断

## Solution

### Approach

在调用 `gh release view` 的步骤中添加 `GH_TOKEN` 环境变量配置。

### Implementation

修改 `.github/workflows/sync-azure-storage.yml` 文件的第 143-154 行：

**修改前：**
```yaml
      - name: Generate and upload index file
        run: |
          ACCOUNT="${{ steps.sas_info.outputs.account }}"
          CONTAINER="${{ steps.sas_info.outputs.container }}"
          SAS_TOKEN=$(cat /tmp/sas_token.txt)
          RELEASE_TAG="${{ steps.release_info.outputs.tag }}"

          echo "Generating index.json..."

          # Get GitHub Release published_at for fallback
          echo "Fetching GitHub Release publish time..."
          PUBLISHED_AT=$(gh release view "${RELEASE_TAG}" --json publishedAt -q .publishedAt)
```

**修改后：**
```yaml
      - name: Generate and upload index file
        run: |
          ACCOUNT="${{ steps.sas_info.outputs.account }}"
          CONTAINER="${{ steps.sas_info.outputs.container }}"
          SAS_TOKEN=$(cat /tmp/sas_token.txt)
          RELEASE_TAG="${{ steps.release_info.outputs.tag }}"

          echo "Generating index.json..."

          # Get GitHub Release published_at for fallback
          echo "Fetching GitHub Release publish time..."
          PUBLISHED_AT=$(gh release view "${RELEASE_TAG}" --json publishedAt -q .publishedAt)
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Scope

### In Scope
- 在 `Generate and upload index file` 步骤中添加 `GH_TOKEN` 环境变量
- 验证修复后工作流正常运行

### Out of Scope
- 修改其他已正确配置 `GH_TOKEN` 的步骤
- 修改工作流的其他逻辑
- 添加新的功能

## Impact

### User Impact
- 应用更新检测功能能够正常工作
- 用户可以及时获取到新版本更新

### Technical Impact
- CI/CD 工作流能够完整执行
- `index.json` 生成步骤可以正确获取 Release 的 `publishedAt` 时间戳
- Azure Storage 同步流程不会中断

## Risk Assessment

**Risk Level: Low**

- 修改范围极小，仅添加环境变量配置
- 不涉及业务逻辑变更
- 使用 GitHub Actions 内置的 `GITHUB_TOKEN`，无需额外配置
- 向后兼容，不影响现有功能

## Dependencies

None. This is an independent fix.

## Success Criteria

1. `Generate and upload index file` 步骤成功执行
2. `gh release view` 命令能够正确返回 `publishedAt` 时间戳
3. `index.json` 文件成功生成并上传到 Azure Storage
4. CI/CD 工作流完整执行无错误

## Related Issues

None.

## Code Changes Summary

| 文件路径 | 变更类型 | 变更原因 | 影响范围 |
|---------|---------|---------|---------|
| `.github/workflows/sync-azure-storage.yml` | 添加环境变量 | 在 `Generate and upload index file` 步骤添加 `GH_TOKEN` 环境变量配置 | CI/CD 工作流 |
