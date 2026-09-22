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

# localsend_x 规则（本 fork 维护）

> 本文件上半部分是上游 LocalSend 的 AGENTS.md 原文，本 fork **未改动一字**（便于与基线逐行对比、便于回滚）。
> 冲突时：**流程、授权与验证规则以本节为准；架构事实以上游章节为准**；构建 / 测试命令的可用性以 §4 与 §11 的实测结论为准。
> 本节只写规则，不复制需求与架构正文 —— 需求与架构正文属于专门文档（见下表）。

## 1. 目标与文档入口

- 本仓库是 LocalSend 的**私有魔改 fork**，目标用户为作者本人，交付形式为**安卓 APK**，仅在本 fork 内演进、不向上游提交 PR —— 因此上游开头"不接受 AI 生成贡献"的条款不约束本仓库的工作。
- **项目目标：待 `requirement.md` 生成后确定。** 已记录候选目标「原版 LocalSend 在 Android 上无法开机自启、熄屏后无法后台持续运行」；在需求文档落地前**不定性、不据此开始设计或实现**。
- 目标平台 Android 16（API 36）。`app/android/app/build.gradle` 现状为 `compileSdkVersion 36` / `targetSdkVersion 37`（后者由上游 `5264f659` 引入）；除非任务明确要求，不改 SDK 级别配置。

| 文档 | 位置 | 状态 |
|---|---|---|
| 流程与协作规则 | `AGENTS.md` 下半部分（本文件） | 有效 |
| 上游架构 / 仓库布局 / 命令原文 | `AGENTS.md` 上半部分 | 有效 |
| AI 入口，独立一行 `@AGENTS.md` 导入本文件 | `CLAUDE.md` | 有效 |
| 完整需求文档 `requirement.md` | 文件名已定，**目录未指定** | **待生成**（项目目标在其生成后确定） |
| 架构 / 设计文档 | 路径**待架构确定**（仓库无 `docs/`） | 不存在 |
| 交接文档 | 路径**待架构确定**（当前无 `docs/handoff.md`） | 不存在 |

职责边界：

| 任务类型 | 可修改范围 | 越界时 |
|---|---|---|
| 只读复核 | 不改任何文件、不改外部状态（§6） | — |
| 设计 | 仅任务指定的设计文档 | 停 |
| 实现 | 仅当前任务相关文件 | 停下并确认 |

## 2. 开始任务前必做

1. `git status --short --branch`：确认工作区状态、当前分支、与远端的领先 / 落后关系。
2. **保留已有改动**：存在未提交改动或未 push 提交时不动它们 —— 不 stash、不 checkout、不 reset、不覆盖、不代为提交。
3. 读本文件、`CLAUDE.md` 与任务相关的设计 / 需求文档；**不存在的文件如实说明，不假设其存在**。
4. 不假设能读到其他工具或会话的聊天记录；跨会话信息只能来自仓库内载体（§7）。
5. 命令可用性以 §4 / §11 为准，不假设工具链可用。

## 3. 任务粒度与边界

- **一次只做一个可验收的任务**：先写清验收标准，再动手；当前任务未验收通过，不为下一个任务做铺垫。
- 不擅自扩大需求、不顺手重构、不"顺便"升级依赖、不改架构与目录结构。
- 默认只改 `app/`（Flutter 客户端）。改 `packages/`（Rust 协议层、isolate 层）、`cli/`、`server/` 会扩大与上游的差异，**需任务显式授权**。
- 沿用既有技术栈（Flutter + Rust + Refena），**不引入新技术栈或新依赖**，除非任务明确要求并在提交信息中记录理由。
- 发现的问题按 §6 分级处理，不擅自修。

## 4. 同步要求与命令可用性

- 代码、测试、依赖、配置与文档**同步更新**：行为改了同步测试与文档，配置改了同步文档描述 —— 漏改任一即视为任务未完成。
- **只有实际跑通的命令才能标为"可用"**；未跑通、未安装的一律标 **"待工程验证"**。上游章节列出的命令不等于本机可用。
- **禁止声称未实际运行的测试、构建或人工验收通过**。无法验证时明确写"未验证 + 原因"，不得省略、不得推测。
- 命令可用性（实测见 §11）：

| 命令组 | 状态 |
|---|---|
| `git` 只读命令（`status` / `log` / `diff`） | 可用（已实测） |
| Rust：`cargo` / `rustc` / `rustup` | **待工程验证**（工具链未安装） |
| Flutter / Dart：`fvm flutter` / `fvm dart` | **待工程验证**（工具链未安装） |
| 构建 / 测试 / 真机验证 / FRB codegen | **待工程验证** |

- 既有代码规范（继续有效，细节见上游章节）：150 列 + `fvm dart format`；不手改生成代码（`app/lib/gen/`、`*.mapper.dart`、`*.freezed.dart`、`app/test/mocks.mocks.dart`），改源文件后重新生成；i18n 走 slang；保留 `FOSS_REMOVE` 标记（注意 `lib/provider/purchase_provider.dart` 由 `remove_proprietary_dependencies.sh:24` 整文件删除、并无标记，上游 FOSS 节措辞不精确）；新增代码的命名、注释密度与惯用法与相邻代码一致。

## 5. 需求 ID → 任务 ID → 测试证据

- **需求 ID 格式**：`REQ-<域>-<三位序号>`（例 `REQ-AND-001`）。**需求条目正文由 `requirement.md` 维护（待生成，目录未指定）；在其落地前不得凭记忆编造需求条目或 ID。**
- **任务 ID** = 分支名，格式 `task/<REQ-ID>-<短描述>` 或 `fix/<REQ-ID>-<短描述>`；无对应需求 ID 的修复用 `fix/<短描述>`，并在提交信息中写明"无需求 ID：<原因>"。
- **测试证据**：完成报告中每个需求 ID 必须对应「执行的命令 + 真实输出」。**无证据的需求一律视为未完成**（报告格式见 §9）。

## 6. 只读复核

- **边界**：不改受版本控制的文件；不改外部状态（不 push、不改远端、不改设备、不装依赖、不改配置、不提交）。构建 / 测试会写入 `target/`、`.dart_tool/` 等被忽略目录，**默认不执行**，需单独授权。
- **问题分级**：

| 级别 | 定义 | 处理 |
|---|---|---|
| P0 | 阻塞、数据 / 安全风险、不可逆副作用 | 立即停手并报告 |
| P1 | 正确性缺陷（结果错误、崩溃、协议不兼容） | 必须修，或列入未解决项 |
| P2 | 规范 / 可维护性（命名、重复、注释） | 仅在当前任务内可顺手修时修，否则记录 |
| P3 | 风格与偏好 | 仅记录 |

- **修复与复审**：复核只报告、不改；修与不修由人决定。修复必须发生在任务分支上，并在完成报告中标注"复审修复"。复审只对修复后的 diff 重新只读复核，**不扩大范围**；复审结论同样需要证据。

## 7. 分支、提交与交接

- 分支：`main` + 独立任务分支。实现类任务建 `task/*` 或 `fix/*`（格式见 §5），验证通过后合并 `main`；不在 `main` 上长期直接开发。
- 提交：一个任务一个提交；信息沿用仓库既有风格（`feat:` / `fix:` / `chore:` / `docs:` 等）；末尾附 `Co-Authored-By: Claude Code <noreply@anthropic.com>`；提交前用 `git status` / `git diff` 核对只含本任务文件。
- **未经明确授权不 push**（授权范围见 §8）。**远端只用 `origin`**（= 本 fork 仓库）；**不使用 `upstream`、不与上游同步**（见 §11）。
- **同一工作区同一时刻只允许一个写入者**（一个人 / 一个模型 / 一个会话）。开始写入前先确认没有其他写入者；交接必须先移交写入权 —— 前一个写入者停止写入并交出状态报告，后一个再开始。
- **模型交接载体**是仓库内可见的提交信息或交接文档，**不依赖聊天记录**。交接内容至少含：当前分支、需求 / 任务 ID 及状态、已执行命令与真实输出、未解决项、下一步、未验证声明。交接文档路径**待架构确定**（当前无 `docs/handoff.md`）。
- 改动尽量小而集中：优先新增文件、避免大范围改写上游既有文件（目的不再是"同步上游"，而是便于与基线 `230fb692` 逐行对比与回滚；本规则段追加在文件末尾即遵循此原则）。

## 8. 密钥与授权

- 密钥、签名材料与本地配置**绝不入库**：`key.properties`、`*.jks`、`*.keystore`、`local.properties`、`.env`、`/secrets` 下任何内容；发布签名 keystore 仅本地保管。
- 提交前扫描暂存区，`git ls-files` 不得出现上述文件。
- 开发阶段用 debug 签名；正式交付版本用项目自有 release keystore。
- 以下不可逆或影响外部的操作**必须先获明确授权**：`git push`、强推、`git reset --hard`、删除分支或文件、改包名 / `applicationId`、改签名配置、增删依赖、持久化格式变更、打包发布。
- 普通可逆的本地操作（读文件、跑测试、本地提交）在任务授权内直接推进。

## 9. 完成报告（每次任务结束，固定四段，简短）

1. **变更**：改动文件清单（含路径）+ 一句话说明。
2. **验证**：需求 ID / 任务 ID → 执行的命令 → 真实输出；未执行则写"未验证 + 原因"。
3. **未解决项**：遗留问题及其分级（§6）；未确认的新需求在此记录，**不实现**。
4. **下一步**：需要人决策的点（集中提问）+ 不依赖该答案可继续的工作。

## 10. 冲突处理

- 文档、代码与人的要求冲突时，**指出冲突与影响，不静默修改需求去迁就代码**；不擅自删除现有功能。
- 未确认的新需求先记录（§9 第 3 段）。
- 缺失的关键信息集中提问，不依赖答案的工作继续推进。

## 11. 已确认决策与工具链实测

**已确认的工作流决策**：

- **远端与上游（已定）**：**只用 `origin`** = `https://github.com/mxy1021/localsend_x.git`；**不使用 `upstream`、不与上游同步** —— 本 fork 在基线 `230fb692`（v1.18.2，2026-09-14）上自行继续演进。本地 git config 中仍留有 `upstream` remote（未删除）；如需移除请单独授权。
- **私有化范围**：允许改应用名、包名、图标；**暂不擅自删除任何现有功能**（更新检查、遥测、捐赠入口等逐项在需求阶段确认）；**许可证与第三方版权信息不得删除**。
- **常驻后台所需的新权限**（`RECEIVE_BOOT_COMPLETED`、电池优化豁免等）由对应任务单独设计后确定。

**工具链实测**（核查日期 2026-09-23，方式：`cmd //c where`、环境变量、常见安装目录限定深度搜索）：

| 工具 | 实测结果 |
|---|---|
| `git` | 可用 |
| `fvm` / `flutter` / `dart` | **未找到** |
| JDK / `java` | **未找到**（`JAVA_HOME` 未设置） |
| Android SDK | **未找到**（`ANDROID_HOME`、`ANDROID_SDK_ROOT` 未设置） |
| `adb` | 仅第三方软件附带的散落副本（微信开发者工具、腾讯 Androws、搞机工具箱、刷机工具等），均不在 PATH，非完整 SDK 组成部分 |
| `cargo` / `rustc` / `rustup` | **未找到** |
| `flutter_rust_bridge_codegen` | **未找到** |
| `support/submodules/flutter` 子模块 | 未初始化（`git submodule status` 前缀 `-`） |

**结论：当前无法执行任何 Flutter / Rust 构建、测试或设备验证**，§4 中所有构建测试类命令均为"待工程验证"。以上为限定范围搜索所得，不排除未覆盖路径下已安装工具的可能；工具装好后须重新实测并更新本节。

**待确定项汇总**：`requirement.md` 的目录位置、架构 / 设计文档目录、交接文档路径 → **待架构确定**；Rust 与 Flutter 工具链能否安装 / 跑通 → **待工程验证**；项目目标 → **待 `requirement.md` 生成后确定**。
