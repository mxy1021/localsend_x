# AGENTS.md

LocalSend disallows AI generated contributions unless:

- they are bug fixes or
- very small or
- you prove your expertise in your field

This file provides guidance to LLMs when working with code in this repository.

## Repository layout

This is a multi-language monorepo: a Flutter app on top of a Rust protocol implementation.

| Path                           | What it is                                                                                                                                                  |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `app/`                         | The Flutter app (`localsend_app`). UI, providers, persistence, platform channels.                                                                           |
| `packages/localsend_isolates/` | Dart isolate layer + `flutter_rust_bridge` (FRB) bindings. Owns `rust/` (the Flutter plugin crate `rust_lib_localsend_app`) and `rust_builder/` (cargokit). |
| `packages/core/`               | Rust crate `localsend`: protocol, HTTP server/client, crypto, WebRTC. No Flutter dependency.                                                                |
| `packages/typed_isolates/`     | Small standalone package wrapping Dart `Isolate` with typed send/receive channels.                                                                          |
| `server/`                      | Axum WebSocket signaling server for WebRTC (`/v1/ws`). Deployed separately, see `server/Dockerfile`.                                                        |
| `cli/`                         | Rust CLI crate (`localsend-cli`): interactive terminal client on top of `packages/core` (v2 HTTP + multicast).                                              |
| `support/scripts/`             | Release/packaging scripts (per-platform builds, MSIX, Inno Setup, FOSS stripping).                                                                          |

The four Rust crates (`packages/core`, `packages/localsend_isolates/rust`, `server`, `cli`) form a single Cargo workspace rooted at the repository root: one shared `Cargo.lock` and `target/`, and `[profile.*]` settings live only in the root `Cargo.toml` (member profiles would be ignored). Cargokit still builds the plugin crate into its own target dir during Flutter builds.

The three Dart packages (`app`, `packages/localsend_isolates`, `packages/typed_isolates`) form a pub workspace rooted at the repository root: one shared `pubspec.lock` and `.dart_tool/`, resolved by running `pub get` from any member. `rust_builder` and the vendored `cargokit/build_tool` are deliberately not members and still resolve standalone.

Dependency direction: `app` → `localsend_isolates` → (`typed_isolates`, `rust_lib_localsend_app` → `localsend` core).
The app depends on **only** `localsend_isolates` — not on `flutter_rust_bridge`, `typed_isolates`, or the plugin crate directly.

## Flutter version

Pinned to the version in `.fvmrc` (also mirrored in `.github/workflows/ci.yml` and `app/pubspec.yaml`, plus the `support/submodules/flutter` git submodule). Use **`fvm flutter` / `fvm dart`** instead of the system-wide toolchain.
Bumping the version means updating all four places — see the "Bump Flutter" section of `CONTRIBUTING.md`.

## Commands

Run from `app/` unless stated otherwise.

```bash
fvm flutter pub get
fvm dart run build_runner build  # dart_mappable, freezed, flutter_gen, mockito
fvm dart run slang               # i18n codegen (slang_build_runner is disabled in build.yaml)
fvm flutter run
```

Checks (what CI runs):

```bash
fvm dart format --set-exit-if-changed lib test   # CI deletes lib/gen first; generated code is not format-checked
fvm flutter analyze
fvm flutter test
fvm flutter test test/unit/util/security_helper_test.dart          # single file
fvm flutter test --plain-name 'some test name'                     # single test
```

Formatting is **150 columns** (`page_width: 150` in `analysis_options.yaml`, `trailing_commas: preserve`). Any tool that reformats generated Dart at 80 columns creates pure noise — reformat with `fvm dart format` afterwards.

Rust:

```bash
cargo test --features full       # in packages/core — see "Core crate features" below
cargo clippy --features full
cargo check                      # in packages/localsend_isolates/rust, server, cli
```

FRB codegen — run from `packages/localsend_isolates/`:

```bash
flutter_rust_bridge_codegen generate    # config in flutter_rust_bridge.yaml (dart_format_line_length: 150)
```

Codegen has a habit of rewriting `app/test/mocks.mocks.dart` at 80 columns; revert that file if it shows up in the diff.

`packages/localsend_isolates` has its own `build.yaml` and needs its own `build_runner` run when its models change (`pub get` is shared via the pub workspace). CI additionally runs `flutter pub get` in `packages/localsend_isolates/rust_builder/cargokit/build_tool`.

## Core crate features

`packages/core` gates almost everything behind Cargo features (`crypto`, `http`, `multicast`, `webrtc`, `webrtc-signaling`, `full`), and `default = []`. **Always build and test it with `--features full`.** A bare `cargo check`/`cargo build` fails because modules are declared unconditionally while their dependencies are optional — that is pre-existing and expected, not a regression.

## Architecture

### State management

Refena (`refena_flutter`), not Riverpod. Providers live in `app/lib/provider/`; `NotifierProvider` for plain state, `ReduxProvider` + dispatched action classes for anything the isolate layer touches. `app/lib/config/init.dart` (`preInit`) is the bootstrap: it initialises logging, `RustLib.init()`, persistence, the isolate container, tray/window, and returns the `RefenaContainer` that `main.dart` mounts.

Models are `dart_mappable` (`@MappableClass`, `.mapper.dart` parts) with renamed methods — `fromJson`/`toJson` are the **Map** converters and `deserialize`/`serialize` are the string ones (configured in both `build.yaml` files). Freezed is used for FRB-adjacent unions.

### Isolates

The heavy networking never runs on the main isolate. `packages/localsend_isolates/lib/src/isolate/`:

- `parent/parent_isolate_provider.dart` — `ParentIsolateState` holds one `IsolateConnector` per child (http scan discovery, multicast discovery, http upload, http server) plus a `SyncState` mirrored into every child. `IsolateSetupAction` spawns them.
- `parent/actions.dart`, `parent/actions_sync.dart` — the only supported way for the app to talk to the children.
- `child/*_isolate.dart` — child entry points; they translate typed task messages into calls on `lib/src/task/`.
- `lib/src/task/` — pure helpers only; **isolate logic is prohibited there** (see its `README.md`).

State that children need (alias, port, protocol, whether the server runs, whether web send is on) is pushed via `IsolateSyncServerStateAction`. Children read `syncState` at start, so sync **before** starting the server.

### Networking (Rust)

The HTTP server and client are Rust, not Dart. `packages/core/src/http/server/` implements protocol v2 (v1 endpoints are not served) plus the "web send" download flow (`server/web.rs`) and an internal `show` endpoint used to foreground an already-running instance.

Integration is channel-based: `start_with_port` takes a `ServerConfigV2 { pin, event_tx, web_send }` and emits `ServerEventV2` events (`Register`, `PrepareUpload` with a `decision_tx` oneshot, `FileUpload` with a byte stream + `result_tx`, `PrepareDownload`, `SessionEnd`, `PrepareUploadAborted`, `CancelReceived`). Only **one upload session is active at a time**; cancellation safety comes from drop guards (`PendingSessionGuard`, `UploadGuard`, `PendingWebSessionGuard`). There is deliberately no `auto_accept` in core — the app auto-accepts by answering `decision_tx` immediately. New server→app interactions should extend `ServerEventV2` rather than adding side channels.

The FRB layer (`packages/localsend_isolates/rust/src/api/server.rs`) exposes `start_server` + an opaque `RsHttpServer` whose `listen` merges the v2, web-send and internal channels into one `RsServerEvent` stream; responder oneshots stay on the Rust side. On the Dart side `child/server_isolate.dart` turns those into `HttpServerEvent`s, which `app/lib/provider/network/server/server_provider.dart` routes to `ReceiveController` / `SendController` — these are **event handlers, not route handlers**.

Save targets are decided in Dart (`prepareFileSaveTarget`) and written by Rust: a path, or an Android SAF file descriptor obtained through the `org.localsend.localsend_app/localsend` method channel. Gallery saves go through a cache file first.

Server event `ip`s are `PeerIp` (IP + IPv6 scope): a link-local peer renders as `fe80::1%3`, which the HTTP client accepts back as a host, so event ips stay dialable.

TLS uses per-device on-the-fly certificates with **mandatory client certificates** (optional while the web pages are served, so browsers can connect); the peer identity is the uppercase-hex SHA-256 of the client cert DER, and `Register` is simply not emitted when a payload's claimed fingerprint disagrees with the cert. Prefer `event.certFingerprint ?? event.info.fingerprint` — the payload fallback only exists for encryption-off mode.

Both the receive pin and the web-send pin are fixed at server start, so changing either restarts the server.

Web assets for the browser download page are embedded from `packages/core/assets/web/`.

### Multicast discovery (Rust)

`packages/core/src/multicast/` (feature `multicast`, independent of `http`) implements UDP multicast discovery for protocol v2.2 — v1 messages are not parsed.
Integration mirrors the HTTP server: `multicast::start` takes a `MulticastConfig { group, group_v6, port, interface_filter, device, event_tx }` and emits `MulticastEvent::Discovered { ip, message }`; the returned `MulticastHandle` offers `announce` (the announcement burst) and `wait_stopped`.

UDP is **announce-only**: responses go back over HTTP as a unicast register request to the announcing device.

One socket is bound per interface IPv4 address (`SO_REUSEPORT`/`SO_REUSEADDR` + `IP_MULTICAST_IF`), because a single socket only sends on one interface. Multicast loopback stays on so that instances on one host see each other; own messages are dropped by fingerprint. IPv6 is a LocalSend extension (group `ff12::fd3a:e420`, `DEFAULT_MULTICAST_GROUP_V6`), enabled by setting `group_v6`: one `IPV6_V6ONLY` socket per interface, joined by interface index. `Discovered` carries the source's scope ID (interface index), which link-local IPv6 sources need for the HTTP answer.

### i18n

Slang, source files in `app/assets/i18n/` (`<locale>.json` plus `_missing_translations_<locale>.json`), generated output in `app/lib/gen/`. Translations are managed on Weblate; fields prefixed with `@` are metadata for translators and are not used by the app. `app/test/unit/i18n_test.dart` guards the locale set.

### FOSS build

`in_app_purchase` and the donation UI are stripped for F-Droid by `support/scripts/remove_proprietary_dependencies.sh`, which relies on the `# [FOSS_REMOVE]` pubspec marker and `// [FOSS_REMOVE_START]` / `// [FOSS_REMOVE_END]` comment pairs. Preserve those markers when editing `lib/config/init.dart`, `lib/pages/donation/*`, or `lib/provider/purchase_provider.dart`.

## Release notes

`app/pubspec.yaml`'s version must match `#define MyAppVersion` in `support/scripts/compile_windows_exe-inno.iss` and the `version` in `cli/Cargo.toml` (the CLI prints it in its start banner) — CI fails on a mismatch. Platform build commands and release steps are documented in `README.md` ("Building") and `CONTRIBUTING.md` ("Release").

---

# 本项目开发与 AI 协作规则（localsend_x）

> 本节由本 fork 维护，**不属于上游 LocalSend**，上游原文到此为止、未做任何修改。
> 就本 fork 的工作流而言，本节优先；协议、架构与构建命令等**事实**仍以上游章节为准。

## 0. 项目定位

本仓库是 LocalSend 的**私有魔改 fork**，目标用户仅为作者本人，交付形式为**安卓 APK**。

- **仅在 fork 内演进**，不向上游提交 PR。因此本文件开头"LocalSend 不接受 AI 生成贡献"的条款不约束本仓库的工作。
- 已确认要解决的问题（当前唯一目标）：原版 LocalSend 在 Android 上**无法开机自启**、**熄屏后无法后台持续运行**。
- 目标平台为 Android 16（API 36）。注意：`app/android/app/build.gradle` 现状为 `compileSdkVersion 36` / `targetSdkVersion 37`，即已高于该目标；除非任务明确要求，不要改动 SDK 级别配置。

## 1. 改动边界

- 首要交付面是 `app/`（Flutter 客户端），默认只改 `app/`。
- 改动 `packages/`（Rust 协议层、isolate 层）、`cli/`、`server/` 需任务显式授权 —— 那些是上游共用代码，改动会扩大与上游的差异。
- 不顺手重构、不扩大需求、不"顺便"升级依赖。发现的问题先记录，不擅自修。

## 2. 技术栈与工具链

- 固定沿用仓库既有技术栈（Flutter + Rust + Refena），**不引入新技术栈或新依赖**，除非任务明确要求并记录理由。
- 必须使用 `fvm flutter` / `fvm dart`，禁止裸 `flutter` / `dart`（见本文件开头）。
- `packages/core` 一律带 `--features full` 构建与测试。

## 3. 任务分级与授权

| 任务类型 | 可修改范围 | 超出范围时 |
|---|---|---|
| 只读复核 | 不改任何文件 | — |
| 设计 | 仅指定设计文档 | 停 |
| 实现 | 仅当前任务相关文件 | 停下并确认 |

以下**不可逆或影响外部**的操作必须先获确认：`git push`、强推、`git reset --hard`、删除分支或文件、改包名/`applicationId`、改签名配置、增删依赖、持久化格式变更、打包发布。

普通可逆的本地操作（读文件、跑测试、本地提交）在任务授权内直接推进。

## 4. 代码规范

- 沿用上游：150 列（`page_width: 150`），用 `fvm dart format` 格式化。
- **不手改生成代码**：`app/lib/gen/`、`*.mapper.dart`、`*.freezed.dart`、`app/test/mocks.mocks.dart` 均由 codegen 产出；要改就改源文件并重新生成。
- i18n 走 slang（源文件 `app/assets/i18n/`），不手写 `lib/gen/strings_*.dart`。
- 保留 `# [FOSS_REMOVE]` 与 `// [FOSS_REMOVE_START]` / `// [FOSS_REMOVE_END]` 标记（见上游 FOSS build 节）。
- 新增代码的命名、注释密度与惯用法要和相邻代码一致。

## 5. 验证与证据（硬性）

- **禁止声称未实际运行的测试、构建或人工验收通过**。结论必须附命令与真实输出。
- 无法验证时明确写"未验证"并说明原因（例如工具链缺失），不得省略、不得推测。
- 实现任务的完成报告至少包含：改动文件清单、执行过的命令、实际结果、未覆盖的风险。

## 6. 变更与提交

- 实现类任务先建 `task/*` 或 `fix/*` 分支（见第 9 节），验证通过后再合并 `main`；不在 `main` 上长期直接开发。
- 一个任务一个提交，信息格式沿用仓库既有风格（`feat:` / `fix:` / `chore:` 等）。
- 提交信息末尾附 `Co-Authored-By: Claude Code <noreply@anthropic.com>`。
- 提交前用 `git status` / `git diff` 核对只包含本任务的文件。
- **未经明确授权不 push**。远端目前只有 `origin`，没有 `upstream`。
- 尽量让本 fork 的改动对上游可追踪：优先新增文件、避免大范围改写上游既有文件（本节追加在文件末尾即遵循此原则）。

## 7. 敏感信息

- 绝不提交：`key.properties`、`*.jks`、`*.keystore`、`local.properties`、`.env`、`/secrets` 下任何内容。
- 发布签名 keystore 仅本地保管，不入库。
- 提交前扫描暂存区，确认不含上述内容。

## 8. 冲突与需求处理

- 文档、代码与用户要求冲突时，**指出冲突与影响，不静默修改需求去迁就代码**。
- 未经确认的新需求先记录，不实现。
- 缺失的关键信息集中提问；不依赖答案的工作继续推进。

## 9. 已确认的工作流决策

**分支策略**：`main` + 独立任务分支。实现类任务建 `task/*` 或 `fix/*` 分支，验证完成后合并 `main`；不在 `main` 上长期直接开发。

**上游同步**：仓库源自官方 LocalSend，已配置 `upstream` remote 指向 `https://github.com/localsend/localsend.git`。同步上游时若发生冲突，**不得自动覆盖本地修改**，必须先分析冲突再由人决定。是否长期定期同步上游，待项目定位进一步明确后再定。

**私有化范围**：允许修改应用名、包名、图标。**暂不擅自删除任何现有功能**；更新检查、遥测、捐赠入口等逐项在需求阶段确认。**许可证与第三方版权信息不得删除**。

**签名**：开发阶段用 debug 签名；正式交付版本用项目自有 release keystore。keystore、密码与签名敏感配置一律不入 Git（见第 7 节）。

**常驻后台所需的新权限**（`RECEIVE_BOOT_COMPLETED`、电池优化豁免等）由对应任务单独设计后确定。

## 10. 工具链实测状态

核查日期 **2026-09-23**，方式为检查 PATH（含 `cmd //c where`）、环境变量与常见安装目录，并在 `Program Files`、`D:\APPs`、`D:\Program Files`、`AppData\Local`、`D:\Files\tools` 等根目录做限定深度搜索。

| 工具 | 实测结果 |
|---|---|
| `fvm` | **未找到**（PATH 无，常见安装目录无） |
| `flutter` / `dart` | **未找到** |
| JDK / `java` | **未找到**（`JAVA_HOME` 未设置） |
| Android SDK | **未找到**（`ANDROID_HOME`、`ANDROID_SDK_ROOT` 均未设置，无 `sdkmanager`） |
| `adb` | 有若干**散落副本**（随微信开发者工具、腾讯 Androws、搞机工具箱、刷机工具等第三方软件附带），均不在 PATH，也不是完整 SDK 的组成部分 |

**结论：当前无法执行任何 Flutter 构建、测试或设备验证。** 在任何构建或测试结论成为有效证据之前，需先安装：`fvm` + Flutter（版本见 `.fvmrc`：**3.41.9**）、JDK、Android SDK。

**未验证声明**：以上为限定范围的搜索所得，不排除存在搜索未覆盖的路径下已安装工具的可能。