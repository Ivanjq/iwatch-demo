# iwatch-demo：无 Mac 也能开发 iOS/watchOS

这个仓库用于演示一种实用工作流：

- 在 **Windows** 上写 Swift 代码、改资源、提交 Git
- 由 **GitHub Actions 的 macOS Runner** 负责 `xcodebuild`
- 先做 **无签名模拟器构建**（快速验证代码能编译）

> 结论：没有 Mac 也能开发大部分内容，但最终真机签名、上架仍依赖 Apple 工具链（在云端 macOS 完成）。

## 已配置内容

- 工作流文件：`.github/workflows/build.yml`
- 触发方式：
  - push 到 `main`
  - push tag（`v*`，例如 `v1.2.3`）
  - Pull Request
  - 手动触发（`workflow_dispatch`）
- 构建目标（默认）：
  - iOS Simulator（无签名）
  - watchOS Simulator（无签名）
- Scheme 选择：
  - 手动触发时可在页面输入 `scheme`
  - 不填时自动选择项目第一个 Scheme
- 发布能力（手动触发可选）：
  - `release_ipa=true`：执行签名归档并导出 `ipa`
  - `upload_testflight=true`：导出后自动上传 TestFlight
  - `deploy_env`：`dev` / `staging` / `prod`
  - `export_method`：`auto` / `app-store` / `ad-hoc` / `development`
- 自动版本处理：
  - `build number` 自动使用 GitHub `run_number`
  - 当你 push `v1.2.3` tag 时，`marketing version` 自动取 `1.2.3`

## 你需要做的最少配置

### 1) 确认项目名

当前工作流默认：

- `XCODE_PROJECT=iwatch-demo.xcodeproj`

如果你的工程名不同，改 `.github/workflows/build.yml` 里的 `env.XCODE_PROJECT` 即可。

### 2) 确认 Scheme 可共享（很重要）

GitHub Actions 只能看到 **Shared Scheme**。  
在 Xcode 中把目标 Scheme 勾选 Shared 后提交到仓库。

如果你暂时没有 Mac，可先在手动触发时直接传 `scheme` 测试。

## 如何使用

1. 在 Windows 上开发并提交代码
2. 推送到 GitHub
3. 打开仓库的 **Actions** 页面查看构建结果
4. 如需指定 Scheme，使用 **Run workflow** 手动触发并填写 `scheme`

## 导出 IPA / 上传 TestFlight（已补充）

### 1) 在 GitHub 仓库中设置 Secrets

进入：`Settings -> Secrets and variables -> Actions -> New repository secret`

必需（用于签名打包）：

- `IOS_CERTIFICATE_BASE64`：开发者证书 `.p12` 的 Base64
- `IOS_CERTIFICATE_PASSWORD`：`.p12` 密码
- `IOS_PROVISION_PROFILE_BASE64`：`.mobileprovision` 的 Base64
- `KEYCHAIN_PASSWORD`：CI 临时 keychain 密码（可自定义随机值）
- `APPLE_TEAM_ID`：Apple Team ID

可选（仅在上传 TestFlight 时必需）：

- `APPSTORE_API_KEY_ID`
- `APPSTORE_API_ISSUER_ID`
- `APPSTORE_API_PRIVATE_KEY_BASE64`：`.p8` 私钥 Base64

### 2) 如何生成 Base64（本地）

PowerShell 示例（Windows）：

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("D:\path\to\cert.p12"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("D:\path\to\profile.mobileprovision"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("D:\path\to\AuthKey_XXXXXXX.p8"))
```

把输出完整复制到对应 Secret（不要换行、不要多空格）。

### 3) 触发发布

1. 打开 **Actions** -> `Build iOS and watchOS` -> **Run workflow**
2. 参数建议：
   - `release_ipa = true`
   - `upload_testflight = false`（先验证导包）
   - `deploy_env = staging`
   - `export_method = auto`（推荐）
   - `scheme = <你的主 Scheme，可留空自动选>`
3. 运行成功后，在 Artifacts 下载 `ipa-xxx`

### 4) 上传 TestFlight

当你已验证导包没问题后，再次手动触发：

- `release_ipa = true`
- `upload_testflight = true`
- `deploy_env = prod`
- `export_method = auto`

上传成功后，去 App Store Connect 的 TestFlight 页面查看构建处理状态。

### 5) Tag 自动发布（推荐）

当你 push `v*` tag 时（例如 `v1.2.3`），工作流会自动进入发布流程：

- 自动执行签名归档并导出 `ipa`
- 自动使用 `marketing version = 1.2.3`
- 自动使用 `build number = github.run_number`
- 默认发布环境为 `prod`
- 默认尝试上传 TestFlight（若未配置 API Key Secrets 会报错）
- 自动校验 tag 格式，仅允许 `vX.Y.Z`（如 `v1.2.3`），不符合会直接失败

示例命令：

```bash
git tag v1.2.3
git push origin v1.2.3
```

## 环境与导出方式映射

当 `export_method=auto` 时，按环境自动选择：

- `dev -> development`
- `staging -> ad-hoc`
- `prod -> app-store`

你也可以手动覆盖成固定导出方式。

## 常见失败与处理

- **未找到 xcodeproj**  
  检查 `XCODE_PROJECT` 路径是否正确。

- **无法解析 Scheme**  
  手动触发并填写 `scheme`，或在仓库变量里设置 `SCHEME`。

- **签名相关错误**  
  检查证书、描述文件、Bundle ID、Team ID 是否匹配；确认 Secrets 是否完整。

- **Archive 失败（Provisioning/Capabilities）**  
  说明项目签名配置与提供的 profile 不一致，需在 Xcode 里统一 Signing（可在云 Mac 完成）。

- **TestFlight 上传失败（401/鉴权）**  
  检查 `APPSTORE_API_KEY_ID`、`APPSTORE_API_ISSUER_ID`、`APPSTORE_API_PRIVATE_KEY_BASE64` 是否正确。

- **Tag 触发后立即报上传 Secret 缺失**  
  说明走了自动 TestFlight 上传流程，请补齐 App Store API Key 三个 Secrets，或改用手动触发并关闭上传。

- **Tag 格式不合法导致发布失败**  
  当前只允许严格语义化版本：`vX.Y.Z`，例如 `v2.0.1`。

## 下一步（可选）

如果你要继续做到“团队可维护的稳定发布”，下一步建议加：

- Fastlane（统一管理签名、版本号、上传）
- Release Notes 自动生成
- Tag 规则校验（例如只允许 `vX.Y.Z`）