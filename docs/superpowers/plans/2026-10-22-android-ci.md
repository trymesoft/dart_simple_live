# Android 自动打包 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 fork（trymesoft/dart_simple_live）加一条独立的 Android 构建流水线：手动 dispatch 或打 `android-v*` tag 时产出已签名 release APK，artifact 保留 90 天，tag 构建同时建 GitHub Release 永久留存。

**Architecture:** 新增单个 workflow 文件 `.github/workflows/android-build.yml`，运行在 ubuntu-latest，复用仓库现有 `signingConfigs.release`（只补 `key.properties`，不改 gradle / app 代码）。签名材料来自 fork 的 4 个 repository secrets。上游 4 个 workflow 文件保持原样。

**Tech Stack:** GitHub Actions、Flutter 3.38.x、JDK 17 (zulu)、Gradle 8.12（flutter 工具自动注入 wrapper）、`gh` CLI（release 创建）、keytool。

**Spec:** `docs/superpowers/specs/2026-10-22-android-ci-design.md`

## Global Constraints

- 仓库：`trymesoft/dart_simple_live`（public，默认分支 `master`，gh 权限 admin）。所有 gh 命令用 `-R trymesoft/dart_simple_live`。
- Android 包名 `com.xycz.simple_live`；版本真源 `simple_live_app/pubspec.yaml`（当前 `1.11.4+11104`）。
- 构建命令固定为 `flutter build apk --release --split-per-abi`；Flutter `3.38.x`、JDK `17`（distribution `zulu`）。
- secrets 名称固定：`KEYSTORE_BASE64`、`STORE_PASSWORD`、`KEY_PASSWORD`、`KEY_ALIAS`；keystore 别名固定 `simplelive`。
- tag 触发模式固定 `android-v*`；Release 用默认 `GITHUB_TOKEN`（`permissions: contents: write`），不新增 `TOKEN`。
- artifact 保留 `90` 天；产物文件名 `simple-live-<versionName>-<abi>.apk`。
- 不改：`simple_live_app/android/**`、`pubspec.yaml`、`.github/workflows/publish_*.y*ml`。
- 本地没有 Flutter SDK / Android build-tools；**不要尝试本地构建**，构建验证一律在 GitHub 上真实运行。

## Review Focus

- **缺 secret / keystore 还原失败**：run 必须在构建前失败并打出清晰错误，而不是让 gradle 报 `key.properties` 的空指针。
- **非 `android-v*` 的 tag 或分支 push**：不得触发本 workflow、更不得建 Release。
- **同一 tag 重跑**：Release 已存在时上传必须 `--clobber`，否则第二次构建失败。
- **产物缺失**：`upload-artifact` 与 release 文件列表若为空（比如 split 产物改名失败），run 必须失败，不能静默绿。
- **secrets 泄漏**：workflow 日志里不得出现密码明文或 keystore base64（不加 `set -x`，不回显）。
- **版本号解析**：pubspec `version:` 行格式异常时 fail fast，不产出 `simple-live-+11104-*.apk` 这类脏文件名。

---

### Task 1: 生成 release keystore 并写入仓库 secrets

**Files:**
- Create: `~/keys/simple-live-release.jks`（仓库外，永不提交）
- Create: `~/keys/simple-live-release.txt`（密码备份，`chmod 600`）

**Interfaces:**
- Produces: 4 个 repository secrets，供 Task 2 的 workflow 引用；本地 keystore 供 Task 4/5 的签名校验比对。

- [ ] **Step 1: 生成随机密码文件（不回显）**

```bash
mkdir -p ~/keys && chmod 700 ~/keys
openssl rand -hex 24 > ~/keys/store.pass
openssl rand -hex 24 > ~/keys/key.pass
```

- [ ] **Step 2: 生成 keystore**

```bash
keytool -genkeypair -v \
  -keystore ~/keys/simple-live-release.jks \
  -alias simplelive -keyalg RSA -keysize 2048 -validity 10000 \
  -dname "CN=Simple Live Fork, OU=Personal, O=Personal, L=Shanghai, ST=Shanghai, C=CN" \
  -storepass:file ~/keys/store.pass -keypass:file ~/keys/key.pass
```

Expected: `Generating 2,048 bit RSA key pair ...`，退出码 0。
若 JDK 17 的 keytool 不支持 `:file` 修饰符（报 unknown option），改用 `-storepass "$(cat ~/keys/store.pass)" -keypass "$(cat ~/keys/key.pass)"` 重试。

- [ ] **Step 3: 校验 keystore**

```bash
keytool -list -keystore ~/keys/simple-live-release.jks -storepass:file ~/keys/store.pass
```

Expected: 输出含 `simplelive, ... PrivateKeyEntry`，并记录 SHA-256 指纹（Task 4/5 比对用）。

- [ ] **Step 4: 写备份说明并收紧权限**

```bash
{
  echo "# Simple Live Android release keystore"
  echo "keystore: ~/keys/simple-live-release.jks"
  echo "alias: simplelive"
  echo "storePassword: $(cat ~/keys/store.pass)"
  echo "keyPassword: $(cat ~/keys/key.pass)"
  echo "备份提示：keystore 丢失后无法覆盖升级，只能卸载重装。"
} > ~/keys/simple-live-release.txt
chmod 600 ~/keys/simple-live-release.txt ~/keys/simple-live-release.jks
```

- [ ] **Step 5: 写入 4 个 repository secrets**

```bash
base64 -i ~/keys/simple-live-release.jks > ~/keys/keystore.b64
gh secret set KEYSTORE_BASE64 -R trymesoft/dart_simple_live < ~/keys/keystore.b64
gh secret set STORE_PASSWORD -R trymesoft/dart_simple_live < ~/keys/store.pass
gh secret set KEY_PASSWORD -R trymesoft/dart_simple_live < ~/keys/key.pass
printf 'simplelive' | gh secret set KEY_ALIAS -R trymesoft/dart_simple_live
```

- [ ] **Step 6: 验证 secrets 存在**

```bash
gh secret list -R trymesoft/dart_simple_live
```

Expected: 恰好这 4 行：`KEYSTORE_BASE64`、`STORE_PASSWORD`、`KEY_PASSWORD`、`KEY_ALIAS`。

- [ ] **Step 7: Commit（本任务无仓库内文件改动，跳过）**

---

### Task 2: 编写 `.github/workflows/android-build.yml`

**Files:**
- Create: `.github/workflows/android-build.yml`

**Interfaces:**
- Consumes: Task 1 的 secrets 名称。
- Produces: workflow 名为 `android-build`；dispatch 与 tag 两条触发；artifact 名 `simple-live-android-<version>`；Release tag 等于推送的 tag。

**已定的实现决策（实现者不再另选）：**

- 触发：`workflow_dispatch`（无输入）+ `push.tags: ["android-v*"]`；不加 master push。
- `upload-artifact` 必须带 `if-no-files-found: error`：产物为空时 run 必须红，不能静默绿。
- 顶层 `permissions: contents: write`；`concurrency: group: android-build-${{ github.ref }}, cancel-in-progress: true`。
- 步骤顺序：checkout → setup-java（zulu 17，`cache: gradle`）→ `subosito/flutter-action@v2`（`flutter-version: 3.38.x`，`cache: true`）→ 还原 keystore → 写 `key.properties` → `flutter pub get`（workdir `simple_live_app`）→ 解析版本（`Resolve version`，同时校验 pubspec 行格式）→ `flutter build apk --release --split-per-abi --build-name <ver> --build-number <code>` → 重命名到 `dist/` 并生成 `SHA256SUMS.txt` → `actions/upload-artifact@v4`（`retention-days: 90`）→ 仅 tag 时 `gh release create`/`gh release upload --clobber`。
- keystore 还原：`echo "$KEYSTORE_BASE64" | base64 -d > "$RUNNER_TEMP/simple-live-release.jks"`，空值时 `exit 1` 并提示缺 secret；`key.properties` 写 `simple_live_app/android/key.properties`，`storeFile` 用绝对路径。
- 版本推导：`PUBSPEC_VERSION` 取 `simple_live_app/pubspec.yaml` 中 `^version: (\S+)$`；`ver=${PUBSPEC_VERSION%%+*}`、`code=${PUBSPEC_VERSION##*+}`；tag 触发时 `ver=${GITHUB_REF_NAME#android-v}`。任一为空或 `code` 非数字 → `exit 1`。
- 重命名映射：`app-armeabi-v7a-release.apk` → `simple-live-<ver>-armeabi-v7a.apk`；`app-arm64-v8a-release.apk` → `simple-live-<ver>-arm64-v8a.apk`；`app-x86_64-release.apk` → `simple-live-<ver>-x86_64.apk`。
- Release：`gh release view "$GITHUB_REF_NAME"` 存在则 `gh release upload ... --clobber`，否则 `gh release create "$GITHUB_REF_NAME" --title "Android <ver>" --notes <多行说明> dist/*.apk dist/SHA256SUMS.txt`；`GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}`。

- [ ] **Step 1: 写 workflow 文件**

按上面决策写完整 YAML（约 90 行）。不要 `echo` 密码，不要 `set -x`。

- [ ] **Step 2: YAML 语法校验**

```bash
cd /Users/wang/develop/workspace/self/dart_simple_live
python3 -c "import yaml;d=yaml.safe_load(open('.github/workflows/android-build.yml'));print(sorted(d.keys()));print(d[True]['push'] if True in d else d['on']['push'])"
```

Expected: 解析成功；输出含 `jobs`、`on`，push tags 为 `['android-v*']`。

- [ ] **Step 3: 静态检查关键约束**

```bash
grep -nE "secrets\.(KEYSTORE_BASE64|STORE_PASSWORD|KEY_PASSWORD|KEY_ALIAS)|retention-days: 90|if-no-files-found: error|android-v\*|contents: write|split-per-abi|--clobber" .github/workflows/android-build.yml
grep -nE "store\.pass|key\.pass|echo \\\$STORE_PASSWORD|set -x" .github/workflows/android-build.yml && echo "LEAK?" || echo "no plaintext secrets"
```

Expected: 4 个 secret 引用、`retention-days: 90`、`if-no-files-found: error`、`android-v*`、`contents: write` 全部命中；第二条无命中（输出 `no plaintext secrets`）。

- [ ] **Step 4: 确认未改上游文件**

```bash
git status --short
```

Expected: 只有 `.github/workflows/android-build.yml`（以及 docs 目录）为新增/修改。

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/android-build.yml docs/superpowers/specs/2026-10-22-android-ci-design.md docs/superpowers/plans/2026-10-22-android-ci.md
git commit -m "ci: add Android-only GitHub Actions build workflow"
```

---

### Task 3: 推送并跑通手动 dispatch 构建

**Files:** 无新增。

**Interfaces:**
- Consumes: Task 2 的 workflow；Task 1 的 secrets。
- Produces: 一次成功的 run id，供 Task 4 取 artifact。

- [ ] **Step 1: 推送 master**

```bash
git push origin master
```

Expected: 推送成功；`gh workflow list -R trymesoft/dart_simple_live` 能看到 `android-build`。

- [ ] **Step 2: 触发 dispatch 并记录 run id**

```bash
sleep 5
gh workflow run android-build.yml -R trymesoft/dart_simple_live --ref master
sleep 10
gh run list -R trymesoft/dart_simple_live --workflow android-build.yml --limit 1
```

Expected: 出现新的 `workflow_dispatch` run。

- [ ] **Step 3: 等待并查看结果**

```bash
gh run watch <RUN_ID> -R trymesoft/dart_simple_live --exit-status
```

Expected: 退出码 0，run 成功（预计 8~15 分钟）。

- [ ] **Step 4: 失败时读日志并修复（迭代步骤）**

```bash
gh run view <RUN_ID> -R trymesoft/dart_simple_live --log-failed | tail -60
```

修复后回到 Step 1（push）→ Step 2。本步骤重复直到 Step 3 通过。已知需要观察的点：ubuntu 上 Android SDK/NDK 是否就绪、`key.properties` 绝对路径是否被 gradle 接受、split 产物路径是否正确。

---

### Task 4: 验证 dispatch 产物与签名

**Files:**
- Create: `~/tmp/android-artifact/`（下载的 artifact）

**Interfaces:**
- Consumes: Task 3 的成功 run id；Task 1 keystore 的 SHA-256 指纹。
- Produces: 签名指纹比对结论，供 Task 6 汇报。

- [ ] **Step 1: 下载 artifact**

```bash
mkdir -p ~/tmp/android-artifact && cd ~/tmp/android-artifact
gh run download <RUN_ID> -R trymesoft/dart_simple_live --name "simple-live-android-1.11.4"
ls -lh
```

Expected: 3 个 `simple-live-1.11.4-*.apk` + `SHA256SUMS.txt`，arm64 包约 40~80 MB。

- [ ] **Step 2: 校验 SHA256**

```bash
cd ~/tmp/android-artifact && shasum -a 256 -c SHA256SUMS.txt
```

Expected: 3 行 `OK`。

- [ ] **Step 3: 校验签名证书指纹与本地 keystore 一致**

```bash
keytool -printcert -jarfile ~/tmp/android-artifact/simple-live-1.11.4-arm64-v8a.apk | grep -A1 "SHA256:"
keytool -list -v -keystore ~/keys/simple-live-release.jks -storepass:file ~/keys/store.pass | grep -A1 "SHA256:"
```

Expected: 两个 SHA-256 指纹完全一致（证明后续构建可覆盖安装）。

- [ ] **Step 4: 记录结论**

把指纹、文件大小、run id 记下来，供最终汇报；不通过则回到 Task 3 Step 4 排查（签名不一致 = key.properties 未生效或用了别的 key）。

---

### Task 5: 验证 tag → GitHub Release 流程

**Files:** 无新增。

**Interfaces:**
- Consumes: 已通过的 dispatch 路径；当前 `master` HEAD。
- Produces: 一个 `android-v<version>` tag、对应 Release（永久资产）。

- [ ] **Step 1: 确定版本并打 tag**

```bash
cd /Users/wang/develop/workspace/self/dart_simple_live
VER=$(sed -n 's/^version: \([^+]*\)+.*/\1/p' simple_live_app/pubspec.yaml | head -1)
git tag -a "android-v$VER" -m "Android $VER"
git push origin "android-v$VER"
```

Expected: tag 推送成功（当前应为 `android-v1.11.4`）。

- [ ] **Step 2: 等待 tag 触发的 run**

```bash
sleep 10
gh run list -R trymesoft/dart_simple_live --workflow android-build.yml --limit 1
gh run watch <RUN_ID> -R trymesoft/dart_simple_live --exit-status
```

Expected: run 由 `push` tag 触发并成功（预计 8~15 分钟）。

- [ ] **Step 3: 验证 Release 与资产**

```bash
gh release view "android-v$VER" -R trymesoft/dart_simple_live
```

Expected: Release 存在，资产含 3 个 `simple-live-<ver>-*.apk` 与 `SHA256SUMS.txt`。

- [ ] **Step 4: 下载 Release 资产并复验签名**

```bash
mkdir -p ~/tmp/android-release && cd ~/tmp/android-release
gh release download "android-v$VER" -R trymesoft/dart_simple_live
shasum -a 256 -c SHA256SUMS.txt
for f in simple-live-*.apk; do echo "== $f"; keytool -printcert -jarfile "$f" | grep -A1 "SHA256:"; done
```

Expected: `OK` ×3；指纹与 Task 4 一致。

- [ ] **Step 5: 记录结论**

---

### Task 6: 收尾核对与文档落定

**Files:**
- Modify: 无（设计文档与 plan 已在 Task 2 提交）

**Interfaces:**
- Consumes: Task 4/5 的结论。
- Produces: 最终汇报用的核对清单。

- [ ] **Step 1: 确认上游 workflow 未改动**

```bash
git diff 456d63a -- .github/workflows/publish_app_dev.yaml .github/workflows/publish_app_release.yml .github/workflows/publish_tv_app_dev.yaml .github/workflows/publish_tv_app_release.yaml --stat
```

Expected: 无输出（4 个文件零改动）。

- [ ] **Step 2: 确认工作区干净**

```bash
git status --short && git log --oneline -3
```

Expected: 无未提交改动；最近提交为 workflow + docs。

- [ ] **Step 3: 汇报**

汇报内容：run 链接、Release 链接、签名指纹比对结果、APK 文件清单与大小、后续用法（推 tag 出永久包 / dispatch 出临时包 / 手机上首次安装需卸载官方版）、keystore 备份位置警告。

---

## 实施期间的变更（记录）

- Global Constraints 中"不改 `pubspec.yaml`"被取代：为让 Gradle 配置通过，固定了 4 个 Android 插件的版本（见 spec 的实施期间决定第 1 条）；这是必要修复，不是设计变更。
- `simple_live_app/pubspec.lock` 从 `.gitignore` 中移除并纳入版本管理（spec 决定第 2 条）；workflow 的 artifact 会附带该文件，便于后续重新锁定。
- Workflow 按静态 review 调整：`overwrite: true`、Release 仅 push 事件触发、版本解析校验、版本值改由 `env:` 传入、Actions 大版本升级（spec 决定第 3 条）。
- 步骤顺序与 plan Task 2 的偏差：实现把 `Resolve version` 放在 `Flutter pub get` 之前（无依赖关系，无影响）。
- 第一次 dispatch（run 39813182152）失败于 `auto_orientation_v2 2.4.7`；第二次（run 39814016311）失败于 `dynamic_color 1.9.0`；均已在上述固定版本中修复，第三次运行待验证。
