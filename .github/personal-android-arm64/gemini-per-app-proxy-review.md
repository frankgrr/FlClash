# Gemini 应用枚举与分应用代理审查（v0.9.3）

本文基于官方 tag `v0.9.3`（commit `be84736605527034689c95bf27828ff27d912c5e`）源码阅读，**未改业务代码**。建议均为可兼容现有行为的增量方案，不要求本 PR 落地。

涉及两条独立路径，不要混为一谈：

1. **Gemini 服务探测**（网络连通 / 区域是否可用）
2. **已安装应用枚举 + 分应用代理**（VPN 访问控制）

源码中没有针对「Gemini Android 应用」的特殊白名单或伪装逻辑。

## 1. Gemini 服务探测

- Dart 侧将 Gemini 登记为服务卡片：`lib/common/service_probe.dart` 中 `ServiceTarget.gemini`（id=`gemini`）。
- Go 核心实现：`core/service_rules.go` 的 `checkGemini`，由 `core/service_check.go` 注册。
- 行为：扫描 `https://gemini.google.com` 响应，查找标记 `,2,1,200,"`，其后 3 个大写字母视为 ISO 3166-1 alpha-3 区域码；若落在 `geminiBlockedRegions`（CHN、RUS、BLR、CUB、IRN、PRK、SYR、HKG、MAC）则返回 `unsupported-region`。
- 注释写明 Gemini 区域码可能出现在响应体 800KiB 之后，因此使用 `scan` 而不是小 body GET。
- `ChinaPackageMatcher` 将 `com.google` 列入跳过前缀，智能选择不会把 Google/Gemini 包当成「国内应用」。这是正确的。

### 不破坏现有行为的改善建议

1. **为页面标记增加夹具测试**：把一段含 `,2,1,200,"USA"` / `"CHN"` 的响应片段放进 `core/service_rules_test.go`，避免日后标记移位时静默变成 `failed`。
2. **增加温和回退，而不是改判定规则**：若标记缺失，保持现在的 `failed`，可额外记录「marker-missing」以便区分「页面改版」和「网络失败」。不要为此放宽或收紧 `geminiBlockedRegions`。
3. **不要把服务探测和分应用代理绑死**：探测的是网站区域，不是 `com.google.android.apps.bard` 是否在 VPN 列表里。两者独立，避免「选了 Gemini 应用就自动改探测结果」这类隐式耦合。

## 2. 应用枚举（PackageResolver）

实现：`android/app/src/main/kotlin/com/follow/clash/packages/PackageResolver.kt`，经 `AppPlugin` / `lib/plugins/app.dart` 暴露给 `lib/views/access.dart`。

当前行为：

- 使用 `PackageManager.getInstalledPackages(GET_PERMISSIONS)` 列出应用。
- 去掉 FlClash 自身（`com.follow.clash`）和 `android`。
- 每项包含：包名、label、是否系统应用、是否声明 `INTERNET`、`lastUpdateTime`。
- 在声明了 `com.android.permission.GET_INSTALLED_APPS` 的系统（包括小米 / HyperOS）上，先检测该权限是否存在，再请求；Android 11+ 同时声明了 `QUERY_ALL_PACKAGES`。
- 列表有缓存，安装变化时 `invalidate()`；Flutter 侧有 `packagesChanged`。
- 「智能选择」用 `ChinaPackageMatcher`：包名 / 组件名 / dex 类名前缀匹配国内 SDK，同时显式跳过 `com.google`、`com.android.chrome`、`com.android.vending` 等。

HyperOS 4 / 小米 15 上，未授予「获取应用列表」时枚举会不完整，访问控制页已有授权与跳转设置的流程（文案见 `installedAppsPermissionDesc`）。**不要**通过改包名、关系统防护或伪造其它应用身份来绕过拦截。

### 不破坏现有行为的改善建议

1. **共享 UID 提示（仅展示）**：`VpnService.resolvePackage` 对同一 UID 只取 `getPackagesForUid(uid)?.firstOrNull()`，连接列表可能张冠李戴。建议在应用列表上标注「与 xxx 共享 UID」，不要自动勾选或改写已保存的 accept/reject 列表。
2. **授权后重试一次**：部分 OEM 在用户刚授权 `GET_INSTALLED_APPS` 后，第一次 `getInstalledPackages` 仍可能偏少。可在授权成功后再刷新一次缓存；失败时保持现有列表，不清空用户选择。
3. **工作资料 / 小米双开**：当前只枚举当前用户。双开或 work profile 中的 Gemini 可能不出现在列表里，VPN 规则也不覆盖其它 user。可作为后续增强，默认行为保持不变。
4. **空列表诊断**：授权已授予但列表仍为空时，给出「OEM 限制 / 需重启应用」的说明，避免被理解成功能损坏。

## 3. 分应用代理（Access Control）

数据：`AccessControlProps`（`lib/models/config.dart` 与 `android/.../VpnOptions.kt`）。

- 默认关闭；模式默认 `rejectSelected`（排除名单）。
- 两种模式：
  - `acceptSelected`：`VpnService.Builder.addAllowedApplication`，并且**始终并入** FlClash 自身包名，保证客户端能走隧道。
  - `rejectSelected`：`addDisallowedApplication`，并从排除名单中去掉自身包名，避免把自己踢出 VPN。
- 已卸载的包名会捕获 `NameNotFoundException` 并跳过，避免一条失效规则让整个隧道建立失败。
- 智能选择：可见的非国内应用写入 accept 列表，国内应用写入 reject 列表；被过滤隐藏的已选包会保留，以免用户筛选系统应用后丢掉选择。
- 默认过滤系统应用和无 `INTERNET` 的应用。独立 Gemini 客户端通常是用户应用且声明了网络权限，默认可见。

Android 文档指出：对某个包调用 `addAllowedApplication` / `addDisallowedApplication` 时，**同一 UID 下的其它包也会一并生效**。这是系统行为，不是 FlClash 额外实现的。

### Gemini 应用与分应用代理

源码没有写死 Gemini 包名。公开的独立 Gemini 应用 historically 使用 `com.google.android.apps.bard`；部分能力也会走 Google 应用（`com.google.android.googlequicksearchbox`）或 Play 服务（`com.google.android.gms`）的网络。

因此在 **仅允许名单** 模式下只勾选 Gemini 时，若其依赖 GMS 或 Google 应用发起网络请求，这些请求的 UID 并不等于 Gemini 本身，看起来会像「代理没有生效」。这不是安全检测问题，也不应通过伪装应用身份解决。

### 不破坏现有行为的改善建议

1. **可选的关联应用提示**：当用户勾选疑似 Gemini / Bard 包时，用一次信息提示（非强制）说明：若功能异常，可再勾选 Google 应用或 Play 服务。**不要自动写入** accept/reject 列表，尤其不要默认放行整个 `com.google.android.gms`。
2. **连接归因改进**：`resolvePackage` 在同一 UID 有多个包时，优先显示出现在当前访问控制列表中的包名；若无匹配再退回 `firstOrNull()`。显示层改动，不改变 VPN 系统 API 调用。
3. **保持现有失败策略**：继续忽略已卸载包，而不是让 `establish()` 失败。
4. **测试补强（加法）**：为 `configureAccessControl` 增加单元测试：accept 模式必含自身包名、reject 模式必不含自身包名、未知包名被跳过。现有 `VpnOptionsTest` / `EnumsTest` 已覆盖序列化，不必改生产逻辑即可补测试。
5. **明确不做的事**：不关闭 HyperOS 安全检测、不改 `applicationId`、不伪造其它应用签名或包名、不移除 `QUERY_ALL_PACKAGES` / `GET_INSTALLED_APPS`（二者是枚举的正当权限，不是规避手段）。

## 4. 结论

| 区域 | 现状 | 本轮是否改代码 |
| --- | --- | --- |
| Gemini 网站区域探测 | 可用，标记解析较脆 | 否，仅建议测试与诊断 |
| 应用枚举 | 通用 `PackageManager` 列表 + OEM 权限 | 否 |
| 分应用代理 | allow/deny + 自身包名保护 + 卸载容错 | 否 |
| 小米 15 / HyperOS 4 真机 | 未验证 | 待验证 |

优先可做且风险最低的后续改动：共享 UID 展示、授权后重试刷新、Gemini 探测夹具测试、允许名单下的关联应用提示（只提示不自动勾选）。
