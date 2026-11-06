# FlClash 个人版 Android arm64 构建报告

日期：2026-11-06（Asia/Shanghai）  
范围：仅本地起草，**未 push、未开 PR、未部署、未访问需要登录的远端**。

## 确认项

| 项 | 结果 |
| --- | --- |
| 官方仓库 | https://github.com/chen08209/FlClash |
| 用户 Fork | https://github.com/frankgrr/FlClash（公开，默认分支 `main`） |
| 工作副本 | `/workspace/FlClash`，本地分支 `personal/android-arm64-ci` |
| Tag | **已确认** `v0.9.3`（`git describe --tags --exact-match`） |
| Commit | **已确认** `be84736605527034689c95bf27828ff27d912c5e`（`chore(release): v0.9.3`） |
| 许可证 | **GPL-3.0**（根目录 `LICENSE` 为 GNU GPL Version 3, 29 June 2007；README 声明相同） |
| 包名 | `com.follow.clash`（`android/app/build.gradle.kts` `applicationId` / `namespace`，`lib/common/constant.dart`） |
| 版本号 | `pubspec.yaml`：`0.9.3+2026110401` |
| 目标设备 | 小米 15，HyperOS 4，ARM64 — **真机安装与拦截状态：待验证** |

## 工具链钉选（与上游 `.github/workflows/build.yaml` / 源码一致）

| 组件 | 钉选 |
| --- | --- |
| Flutter | 3.47.6（README：3.47，正式构建用 3.47.6） |
| Dart SDK 约束 | `>=3.10.0 <4.0.0`（随 Flutter 3.47） |
| Go | 1.27.1（workflow）；`core/go.mod` 为 `go 1.27.0` |
| Rust | rustup；`plugins/rust_api/rust/rust-toolchain.toml` 钉选 **1.99.0**，含 `aarch64-linux-android` |
| JDK | 17（Temurin；Gradle `JavaVersion.VERSION_17`） |
| Android NDK | r28c；`android/gradle/libs.versions.toml` 中 `ndkVersion = "28.2.13676358"` |
| compileSdk / targetSdk / minSdk | 37 / 37 / 24 |
| Clash.Meta | `build_config.yaml`：`core_version: v1.19.33`，tags `with_gvisor` |
| 构建入口 | `dart setup.dart android --arch arm64` → `flutter_distributor` + `--build-target-platform=android-arm64` + `split-per-abi` |
| 产物路径 | `dist/`，命名形如 `FlClash-0.9.3-android-arm64-v8a.apk` |
| 上游签名 | 正式包读取 `android/app/keystore.jks` + `android/local.properties` 的 `keyAlias` / `storePassword` / `keyPassword`。缺失时回退 debug 签名并给 `applicationId` 加 `.dev` 后缀。 |
| 上游 Secrets | `KEYSTORE`（base64）、`KEY_ALIAS`、`STORE_PASSWORD`、`KEY_PASSWORD`。个人工作流**不使用**这些名字。 |

子模块（`.gitmodules` 为 SSH URL）：

- `core/Clash.Meta` → `chen08209/Clash.Meta`（分支 `FlClash`）
- `plugins/prism_ui` → `chen08209/prism_ui`

个人工作流在 runner 上把 `git@github.com:` 改写为 `https://github.com/` 后再 `submodule update`，不改仓库里的 `.gitmodules`。

## 本轮已起草的内容

1. **独立 workflow**（不替换上游 `build.yaml` 的 release job）  
   路径：`.github/workflows/personal-android-arm64.yml`  
   - 仅 `workflow_dispatch`  
   - 安装 Flutter 3.47.6、Go 1.27.1、Rust 1.99.0（rustup / dtolnay rust-toolchain）、JDK 17、Android SDK + NDK r28c  
   - `dart setup.dart android --arch arm64 --env <stable|pre|dev>`，默认为 `stable`  
   - **自有测试签名**：优先使用 Fork secrets `PERSONAL_KEYSTORE`（jks/p12 的 base64）、`PERSONAL_KEY_ALIAS`、`PERSONAL_STORE_PASSWORD`、`PERSONAL_KEY_PASSWORD`；未配置则在 job 内生成一次性 PKCS12。不会打印私钥或订阅地址；构建后删除 keystore，并从 `local.properties` 去掉口令行。  
   - Artifact 名：`flclash-personal-android-arm64-v8a`（APK + `SHA256SUMS` + `BUILD-INFO.txt`）  
   - `BUILD-INFO.txt` 含版本、commit、run URL、签名来源、证书指纹（`apksigner verify --print-certs`）  
   - 权限：`contents: read`，没有 release / Telegram / Homebrew / F-Droid / 部署 job  
   - 构建 job 不依赖测试 job，避免检查失败时拿不到 APK  

2. **Gemini / 分应用代理审查**  
   - 仓库内：`.github/personal-android-arm64/gemini-per-app-proxy-review.md`  
   - 副本：`/workspace/FlClash-personal/docs/gemini-per-app-proxy-review.md`  
   - **未修改**探测逻辑、应用枚举或访问控制代码。

3. **本报告**  
   - 仓库内：`.github/personal-android-arm64/REPORT.md`  
   - 副本：`/workspace/FlClash-personal/REPORT.md`

说明：上游 `.gitignore` 忽略了仓库根目录的 `docs/`，因此审查文档放在 `.github/personal-android-arm64/`，以免提交被忽略。

## 本机实际执行的检查（未跑的不标通过）

| 检查 | 结果 | 说明 |
| --- | --- | --- |
| Tag / commit 核对 | **通过** | detached HEAD 即 `v0.9.3` / `be84736…` |
| GPL-3.0 许可证阅读 | **通过** | `LICENSE` + README |
| 构建系统清点 | **通过** | `setup.dart`、`build.yaml`、`android/app/build.gradle.kts`、manifest、工具链钉选 |
| 权限清单阅读 | **通过** | 见下方权限表 |
| Gemini / 分应用代理源码审查 | **通过** | 见审查文档（设计审查，非测试跑通） |
| Workflow 密钥名审查 | **通过** | 未引用上游 `secrets.KEYSTORE` 等；无发布步骤 |
| `dart format` / `flutter analyze` | **跳过** | 本机未安装与项目匹配的 Flutter 3.47.6 |
| Dart 单元测试 `tool/test.dart` | **跳过** | 同上 |
| 覆盖率 75% 门槛 | **跳过** | 个人 workflow 有意不跑（步骤 `if: false`，总结里记为 skipped） |
| Go core 测试 | **跳过** | 本机未在此环境执行 |
| Android 单元测试 | **跳过** | 需要 NDK / Gradle / Flutter |
| Rust 测试 | **跳过** | 本机未执行 |
| `tool/check_plugins.sh` | **跳过** | 需要 Flutter |
| 实际打 arm64 APK | **跳过** | 留给 GitHub Actions；本机无完整 Android/Flutter/Go/Rust/NDK 工具链验证 |
| 小米 15 安装 | **待验证** | 未真机安装 |
| HyperOS 拦截 / 厂商复核 | **待验证** | 若安装被拒，保留系统错误日志，不关闭防护、不伪装应用身份 |

## Android 权限（源码阅读，非动态审计）

`android/app/src/main/AndroidManifest.xml`：

- `INTERNET`
- `FOREGROUND_SERVICE`
- `ACCESS_NETWORK_STATE`
- `ACCESS_WIFI_STATE`
- `CHANGE_NETWORK_STATE`
- `QUERY_ALL_PACKAGES`（应用枚举 / 分应用代理）
- `com.android.permission.GET_INSTALLED_APPS`（小米等 OEM 的应用列表权限）
- `FOREGROUND_SERVICE_SPECIAL_USE`
- `POST_NOTIFICATIONS`
- `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`
- `ACCESS_LOCAL_NETWORK`
- `REQUEST_INSTALL_PACKAGES`

`android/service`：`ACCESS_NETWORK_STATE`，以及 `VpnService`（`BIND_VPN_SERVICE`）与 `ProxyService`。  
`android/core`：`INTERNET`。

个人 workflow 的 permission 检查会再次抓取这些声明，并确认 `applicationId` 仍为 `com.follow.clash`。

## 可选 Fork Secrets（稳定测试签名）

不配置也能构建：每次 run 生成一次性 keystore（**不能覆盖安装**上一次的临时包，需先卸载）。

若希望多次构建可覆盖安装，在 **frankgrr/FlClash** 的 Secrets 中设置（不要使用上游那四个名字）：

| Secret | 含义 |
| --- | --- |
| `PERSONAL_KEYSTORE` | PKCS12/JKS 文件的 base64 |
| `PERSONAL_KEY_ALIAS` | 别名 |
| `PERSONAL_STORE_PASSWORD` | store 口令 |
| `PERSONAL_KEY_PASSWORD` | key 口令 |

生成示例（只在你自己的电脑上执行，不要提交 keystore）：

```bash
keytool -genkeypair -keystore flclash-personal.p12 -storetype PKCS12 \
  -alias flclash-personal -keyalg RSA -keysize 2048 -validity 3650
base64 -w0 flclash-personal.p12
```

## 上游 `build.yaml` 在公开 Fork 上的副作用

本 PR **没有改** `.github/workflows/build.yaml`。该文件里检查类 job 在「非私有仓库」的每次 push 都会跑；`build` / `upload`（发布、Telegram、Homebrew、F-Droid）被限制为 `github.repository == 'chen08209/FlClash'`，因此 Fork 不会走官方发布。公开 Fork 上的 push 仍会消耗 Actions 分钟数。若以后要关掉，应另开 PR 讨论，不在本轮修改上游 workflow。

## 下一步（需要你这边的 `gh` 登录后执行）

当前 box 上没有 GitHub 凭证，**不要**往 `chen08209/FlClash` push。目标是向 **Fork 自己的 `main`** 开 PR。

```bash
cd /workspace/FlClash
git remote add frankgrr https://github.com/frankgrr/FlClash.git   # 若尚未添加
git fetch frankgrr
git push -u frankgrr personal/android-arm64-ci

gh pr create \
  --repo frankgrr/FlClash \
  --base main \
  --head personal/android-arm64-ci \
  --title "ci: personal Android arm64 workflow_dispatch build" \
  --body "$(cat <<'EOF'
在 Fork 中增加仅 workflow_dispatch 的 Android arm64-v8a 个人构建。

- 不修改上游发布 job，不使用上游 KEYSTORE 等 Secrets
- 使用 PERSONAL_* 或每 run 生成的测试签名
- Artifact 提供 APK、SHA256、commit 与证书指纹
- 静态分析 / 单元测试 / 权限与依赖检查分 job 执行；未跑项记为 skipped
- 真机（小米 15 / HyperOS 4）安装与拦截状态待验证

请审查后由仓库拥有者合并。不会自动发布。
EOF
)"
```

合并后：打开 Actions → `personal-android-arm64` → Run workflow。下载 Artifact `flclash-personal-android-arm64-v8a`，用 `SHA256SUMS` 校验，再在小米 15 上安装。若 HyperOS 拒绝安装，保存完整系统错误日志供厂商复核，不要关闭系统防护或伪装应用身份。

## 验收对照

| 标准 | 状态 |
| --- | --- |
| 工作流可重复运行 | 已起草；需 push 后在 Actions 中验证 |
| APK 能下载 | Artifact 已配置；需首次成功 run 后验证 |
| SHA256 可核验 | `dist/SHA256SUMS` 随 Artifact 上传 |
| 签名及构建来源可追溯 | `BUILD-INFO.txt`（commit、run URL、证书指纹、signing_source） |
| 真机运行 | **待验证** |
| 小米拦截状态 | **待验证** |
