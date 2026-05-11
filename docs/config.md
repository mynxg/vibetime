# 发布签名配置

这份文档记录 VibeTime 发布流程里需要配置的 GitHub Secrets、SignPath 字段和 GitHub Actions 自动值。

更完整的 SignPath 两种签名方式说明见：

- [signpath-code-signing.md](./signpath-code-signing.md)

## 当前项目配置

当前推荐发布流程：

- workflow: `.github/workflows/release-signpath.yml`
- 触发方式: 推送 `v*` tag，或手动 `workflow_dispatch`
- Windows 签名方式: SignPath PowerShell/API 直提
- SignPath project slug: `vibetime-dev`
- SignPath signing policy slug: `vibetime-dev`

当前 PowerShell/API 直提方式不依赖 GitHub Trusted Build System verification，适合 SignPath free trial 场景。

## GitHub Secrets

进入 GitHub 仓库：

```text
Settings -> Secrets and variables -> Actions -> New repository secret
```

当前 `release-signpath.yml` 必需配置：

```text
SIGNPATH_API_TOKEN
SIGNPATH_ORGANIZATION_ID
```

如果改回 GitHub connector 方式，通常还需要：

```text
SIGNPATH_SIGNING_POLICY_SLUG
SIGNPATH_ARTIFACT_CONFIGURATION_SLUG
```

## Secret 来源

### SIGNPATH_API_TOKEN

来源：SignPath 用户详情/Profile 页面。

获取方式：

1. 登录 SignPath。
2. 点击右上角用户入口。
3. 进入 `My profile` 或用户详情页。
4. 找到 API Token 区域。
5. 生成新的 API token。

注意：

- token 只会在生成时显示一次。
- token 所属用户必须是对应 signing policy 的 `Submitter`。
- CI 场景建议使用专门的 CI user，不要长期使用个人账号 token。

### SIGNPATH_ORGANIZATION_ID

来源：SignPath organization 页面。

它通常是一个 GUID，例如：

```text
50865fbe-bb06-420d-b3b4-be28ea95d27b
```

如果页面上看不到 Organization ID，通常是账号权限不够，需要让 SignPath organization 管理员提供。

### SIGNPATH_SIGNING_POLICY_SLUG

来源：SignPath project 里的 signing policy。

当前开发项目使用：

```text
vibetime-dev
```

当前 `release-signpath.yml` 已经把 signing policy slug 写死为 `vibetime-dev`，所以 PowerShell/API 直提流程暂时不需要这个 secret。

### SIGNPATH_ARTIFACT_CONFIGURATION_SLUG

来源：SignPath project 里的 artifact configuration。

这个字段定义上传 artifact 后，SignPath 应该匹配并签哪些文件。

当前 `release-signpath.yml` 没有传 `ArtifactConfigurationSlug`，而是依赖 signing policy 的默认 artifact configuration。只有当 signing policy 没有默认配置，或需要显式指定配置时，才需要重新启用这个 secret。

## 不需要手动配置的值

### github.token / GITHUB_TOKEN

GitHub Actions 自动提供，不需要创建 repository secret。

常见写法：

```yaml
github-token: ${{ github.token }}
```

或者：

```yaml
github-token: ${{ secrets.GITHUB_TOKEN }}
```

### pnpm / Node / Bun / ImageMagick

这些由 workflow 自动安装。

### macOS 签名证书

当前 Electron macOS 配置是未签名/未公证：

```yaml
CSC_IDENTITY_AUTO_DISCOVERY: false
```

`packages/desktop/electron-builder.yml` 中也设置了：

```yaml
mac:
  identity: "-"
```

因此当前不需要 Apple Developer ID 相关 secrets。

## Artifact configuration

Windows release 产物是 `.exe` 文件，例如：

```text
VibeTime-2026.5.1-1.1-x64.exe
VibeTime-Setup-2026.5.1-1.1-x64.exe
```

如果 workflow 先把这些 `.exe` 打成 ZIP 再提交给 SignPath，artifact configuration 应该使用 `zip-file` 作为根节点：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <zip-file>
    <pe-file-set>
      <include path="*.exe" min-matches="1" max-matches="unbounded" />
      <for-each>
        <authenticode-sign />
      </for-each>
    </pe-file-set>
  </zip-file>
</artifact-configuration>
```

不要保留模板里的：

```xml
<include path="sample.exe" />
```

除非上传的 artifact 里真的有 `sample.exe`。

## Trusted Build System

GitHub connector 方式需要 SignPath Trusted Build System verification。

配置位置：

1. SignPath organization 添加 predefined `GitHub.com` trusted build system。
2. 将该 trusted build system link 到目标 project。
3. signing policy 允许对应 trusted build system 提交签名请求。

SignPath free trial 可能不包含这个功能。如果页面提示：

```text
Trusted build system verification is not included in your Free trial subscription.
```

则不要使用 GitHub connector 方式，改用当前的 PowerShell/API 直提方式。

## 常见错误

### ArtifactConfigurationSlug 不存在

错误示例：

```text
There is no artifact configuration with slug '...' in the project with slug '...'.
```

处理方式：

- 确认 secret 里的 slug 属于当前 project。
- 或删除 `ArtifactConfigurationSlug` 参数，改用 signing policy 默认 artifact configuration。

### GitHub connector 返回 502

错误示例：

```text
SignPath REST API is temporarily unavailable (server responded with 502).
```

可能原因：

- SignPath connector 临时故障。
- GitHub Trusted Build System 没配置或没权限。
- 当前订阅不包含 Trusted Build System verification。

free trial 场景下，优先使用 PowerShell/API 直提方式。

### 两个 workflow 同时触发

如果多个 workflow 都包含：

```yaml
on:
  push:
    tags:
      - 'v*'
```

推送同一个 tag 时它们都会运行，并可能同时创建同一个 GitHub Release。

正式发版时应该确保只有一个 workflow 负责创建 release。

## 参考

- SignPath GitHub trusted build system: https://docs.signpath.io/trusted-build-systems/github
- SignPath GitHub action: https://github.com/SignPath/github-action-submit-signing-request
