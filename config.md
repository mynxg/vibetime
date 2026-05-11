这些值分两类：SignPath 后台来的和 GitHub 自动/仓库配置来的。

SignPath Secrets 从哪里来
在 SignPath 控制台里准备：

SIGNPATH_API_TOKEN
从 SignPath 用户/访问令牌页面创建。这个 token 对应的用户必须对指定 Project / Signing Policy 有 submitter 权限。官方文档也明确说 api-token 是必填，并且需要 submitter permissions。

SIGNPATH_ORGANIZATION_ID
SignPath Organization 的 ID。一般在 SignPath 组织设置、URL、或组织详情里能看到。传给 action 的 organization-id。

SIGNPATH_SIGNING_POLICY_SLUG
SignPath 项目里的 Signing Policy slug，比如你可以建一个 release-signing。这是 action 的必填项。

SIGNPATH_ARTIFACT_CONFIGURATION_SLUG
SignPath 项目里的 Artifact Configuration slug。这个定义“上传的 artifact 里哪些文件要签”。因为 GitHub upload-artifact 默认会打成 ZIP，SignPath 官方提醒 Artifact Configuration 根节点通常要是 <zip-file>。

project-slug: vibetime
我现在写死在 workflow 里了。这个不是 GitHub secret，而是 SignPath Project slug。你需要在 SignPath 里创建/确认项目 slug 就叫 vibetime；如果不是，要改 .github/workflows/ci.yml 里的这一行。

GitHub 里怎么配
进入你的仓库：

Settings -> Secrets and variables -> Actions -> New repository secret

添加这四个 repository secrets：
```
SIGNPATH_API_TOKEN
SIGNPATH_ORGANIZATION_ID
SIGNPATH_SIGNING_POLICY_SLUG
SIGNPATH_ARTIFACT_CONFIGURATION_SLUG

```

另外 workflow 里用的 ${{ github.token }} / GITHUB_TOKEN 是 GitHub Actions 自动提供的，不需要你手动配置。我已经在 Windows job 里加了：

```
permissions:
  actions: read
  contents: read
```

SignPath 官方文档说 github-token 需要 actions: read 和 contents: read 权限来读取 job details 和下载 artifact。

你这个 ci.yml 目前需要 GitHub 配置的东西
目前仓库里真正需要你额外配置的只有 SignPath 这 4 个 Secrets。其它这些不需要你配置：

GITHUB_TOKEN：GitHub 自动给
softprops/action-gh-release：用的是 release job 的 contents: write 权限，workflow 已经配置
pnpm / Node / Bun / ImageMagick：workflow 自动安装
macOS 打包证书：当前 CSC_IDENTITY_AUTO_DISCOVERY: false 且 electron-builder.yml 里 mac.identity: "-"，所以现在是不签名/不公证 macOS，不需要 Apple secrets
还有一个 SignPath 后台前置项：要把 GitHub.com 作为 Trusted Build System 加到 SignPath Organization，并把它关联到对应 SignPath Project。官方文档把这个列为 prerequisite。

来源：SignPath GitHub 集成文档和 action 输入定义
https://docs.signpath.io/trusted-build-systems/github
https://raw.githubusercontent.com/SignPath/github-action-submit-signing-request/main/action.yml


## 获取 SIGNPATH_API_TOKEN 

去 SignPath 网页里，用你的 SignPath 账号登录后：

右上角点你的名字
进 My profile
找 API Token 区域
点 Generate token
官方文档写得很明确：token 只会在生成时显示一次，生成完就把它存到 GitHub Secret 里。
如果你是给 CI 用，SignPath 也建议建一个 CI user，再给这个 CI user 生成 token，而不是拿个人账号硬上。

SIGNPATH_ORGANIZATION_ID 怎么拿
这个是在 组织页面里拿。文档说的是：

点击右上角的组织名，进入 organization page
组织 ID 就是那个组织的 ID
如果你自己不是组织管理员，或者页面里看不到这个值，就只能让管理员给你，或者让他们直接帮你建好这几个 secret。

你这次真正要配的东西
在 GitHub 仓库里只需要放这些 Secrets：

SIGNPATH_API_TOKEN
SIGNPATH_ORGANIZATION_ID
SIGNPATH_SIGNING_POLICY_SLUG
SIGNPATH_ARTIFACT_CONFIGURATION_SLUG
其中：

project-slug: vibetime 是我写在 workflow 里的，不是 secret
signing-policy-slug / artifact-configuration-slug 也是 SignPath 后台里先建好，再把 slug 填进来
如果你现在找不到
最常见原因是你只有 GitHub 仓库权限，没有 SignPath 组织权限。
这时你要找的是 SignPath 组织管理员，让他给你：

一个 CI user 或允许你个人账号生成 API token
组织 ID
project slug
signing policy slug
artifact configuration slug


## 说明

api-token 和 organization-id：来自 GitHub Secrets，本质上是从 SignPath 控制台拿

project-slug：SignPath 项目的固定 slug，我现在写的是 vibetime

signing-policy-slug / artifact-configuration-slug：也是 SignPath 后台里对应项目的 slug，再存进 GitHub Secrets

github-artifact-id：GitHub Actions 上一步 
upload-artifact 自动产出的，不用手填

github-token：GitHub 自动提供
