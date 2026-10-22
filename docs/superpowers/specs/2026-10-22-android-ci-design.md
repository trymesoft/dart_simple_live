# Android 自动打包方案（个人自用 fork）

日期：2026-10-22
仓库：trymesoft/dart_simple_live（上游 xiaoyaocz/dart_simple_live 的 fork）
状态：待批准

## 意图与背景

用 GitHub Actions 自动产出 Android release APK，装到本人手机上；每次都能覆盖升级，产物要能长期取回。

现状（分析结论）：

- `.github/workflows/` 下 4 个文件都是上游原样文件：tag 触发（`dev_v*` / `v*` / `tv_*`），checkout 硬编码 `ref: dev` / `ref: master`，在 macOS runner 上一次打全平台（上游最近一次成功 run 41 分钟），签名与 Release 上传依赖自建 secrets。
- fork 里 0 个 secret、0 个 release；唯一一次运行（2026-01-10 手动 dispatch）三个 job 全失败（日志已过期，无法确认具体失败步骤）。
- Flutter monorepo：`simple_live_app`（手机/桌面）、`simple_live_tv_app`（TV）、`simple_live_core`、`simple_live_console`。本次只做 `simple_live_app` 的 Android。

## 目标

1. 手动 dispatch 或推 `android-v*` tag 时，在 ubuntu runner 上构建 release APK（已签名）。
2. tag 构建创建 GitHub Release，APK 作为永久资产；dispatch 构建保留 Actions artifact（90 天）。
3. 签名稳定：任意两次构建的 APK 可互相覆盖安装，不需要卸载。
4. 不修改上游 workflow 文件，便于继续 merge upstream。

## 非目标

- iOS / macOS / Windows / Linux 桌面打包。
- Android TV 应用（`simple_live_tv_app`）打包；同一套做法以后可复制。
- Play Store / AAB 上架、自动生成发版说明、自动改版本号。
- 替换或修复上游 4 个 workflow。

## 反面目标（即使"能跑"也算失败）

- 每次构建签名不同，导致必须卸载重装（丢设置/收藏）。
- 产物只有 Actions artifact，90 天后过期丢失。
- 改动上游 workflow 文件，之后 merge 冲突。

## 约束

- `simple_live_app/android/app/build.gradle.kts` 的 `signingConfigs.release` 无条件读 `key.properties` 并 `as String` 强转：文件不存在时 Gradle 配置阶段抛异常。CI 必须提供 keystore + key.properties。
- 包名 `com.xycz.simple_live` 与官方包一致，签名不同：首次安装要先卸载官方版（数据清一次）；之后自签包之间可正常覆盖升级。
- 版本真源是 pubspec：`version: 1.11.4+11104` → versionName `1.11.4`，versionCode `11104`。
- 工具链与上游一致：Flutter 3.38.x、JDK 17、Gradle 8.12（`android/gradlew` 不在 git 中，由 Flutter 工具自动注入 wrapper，不需要额外步骤）。
- 本机没有 Flutter SDK，也没有 Android build-tools；本地无法构建 APK，构建验证只能在 GitHub 上真实跑。`keytool` 可用（本地校验签名证书）。
- 公开仓库：Actions artifact 保留上限 90 天；永久保留只能靠 Release 资产。
- fork 上 gh CLI 权限为 admin，可直接写 secrets、推分支/tag。

## 方案对比（选中 A）

**A. 新增独立 Android workflow（选中）**
新增 `.github/workflows/android-build.yml`：ubuntu-latest，只做 Android，复用现有 gradle 签名配置。上游 4 个文件不动。

**B. 改造上游 `publish_app_release.yml`，把 Android job 拆到 ubuntu**
少一个文件，但改的是上游维护的文件，每次 merge upstream 都要解冲突，且它仍是"tag 全平台构建"的结构。

**C. 零 secret：gradle 缺 key.properties 时回退 debug 签名**
搭建最快，但 CI 每次生成的 debug key 不同 → 新包不能覆盖安装；artifact 90 天过期。与目标 2、3 直接冲突。

## 方案细节（已定）

- **触发**：`workflow_dispatch` + `push.tags: android-v*`。不监听 master push。
- **权限**：`contents: write`，用默认 `GITHUB_TOKEN` 建 Release，不新增 `TOKEN` secret。
- **runner**：`ubuntu-latest`。
- **步骤**：
  1. `actions/checkout`（tag 时 checkout 该 tag；dispatch 时默认分支）
  2. `actions/setup-java`（zulu 17，gradle cache）
  3. `subosito/flutter-action`（3.38.x，cache）
  4. 从 `secrets.KEYSTORE_BASE64` 还原 keystore 到 `$RUNNER_TEMP/simple-live-release.jks`，用 4 个 secret 生成 `simple_live_app/android/key.properties`（storeFile 用绝对路径）
  5. `flutter pub get`
  6. `flutter build apk --release --split-per-abi`
  7. 产物重命名：`simple-live-<version>-<abi>.apk`，另生成 `SHA256SUMS.txt`
  8. `actions/upload-artifact`（name `simple-live-android-<version>`，retention 90 天）
  9. 仅 tag 构建：`gh release create` 附带 APK 与 SHA256SUMS，Release 名 `android-<version>`
- **版本推导**：tag `android-v1.11.4` → versionName `1.11.4`；versionCode 取 pubspec 的 `+11104`。dispatch 时 versionName/versionCode 都取 pubspec，且不建 Release。
- **产物**：`app-arm64-v8a-release.apk`（主）、`app-armeabi-v7a-release.apk`、`app-x86_64-release.apk` 三个 split 包；不额外做 universal 包（需要时再加）。
- **签名材料**：本地 `keytool` 生成 `simple-live-release.jks`（RSA 2048，有效期 10000 天，随机强密码），放在仓库外 `~/keys/`，keystore 与密码明文备份说明一并留下；4 个 secrets 用 gh 写入 fork：
  `KEYSTORE_BASE64` / `STORE_PASSWORD` / `KEY_PASSWORD` / `KEY_ALIAS`。
- **不改**：app 代码、gradle 配置、pubspec、上游 workflow。

## 留给实现者的判断

- dispatch 是否支持指定 ref：默认只构建默认分支（保持简单）。
- 是否加 `concurrency` 取消同一分支的旧运行：默认加。
- Release body 内容：版本、commit、构建日期即可，不自动生成 changelog。
- workflow 文件放在 `master` 分支（自用仓库，不做 PR 流程）。

## 验收标准

1. 手动 dispatch 一次，run 绿，artifact 内 3 个命名规范的 APK。
2. 推一个 `android-v<版本>` tag，Release 出现且 APK 可下载。
3. 下载的 APK 用 `keytool -printcert -jarfile` 校验，证书指纹与本地 keystore 一致（证明签名稳定、可覆盖升级）。
4. `git diff` 显示上游 4 个 workflow 文件未改动。

## 已决定（原未决问题）

- dispatch 构建的产物只保留 Actions artifact（90 天），不创建 Release。用户确认：过期时通常上游代码已更新，重新打包即可接受；需要永久保留时打 tag 走 Release。

