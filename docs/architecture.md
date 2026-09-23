# 架构设计：Android 后台常驻接收与开机自启

| 项目 | 内容 |
|---|---|
| 文档状态 | **草稿 v0.1（2026-09-23）—— 待仓库所有者确认** |
| 路径 | `docs/architecture.md`（`AGENTS.md` §1 原记"路径未定"，本文件即该路径；如需移动请指出） |
| 依据 | `docs/requirements.md` **v1.6**（已确认为需求基线，见其 §16.3）：REQ-001…011、NFR-001…010、AC-01…AC-13、DEC-01…DEC-37 |
| 代码基线 | 上游 `230fb692`；本 fork 规则基线 `fe5e5929`；本文档落点见 §2 |
| 本期范围 | 仅 Android：`app/**`（含 `app/android/**`）与 `packages/localsend_isolates/**`（DEC-01、NFR-006 允许清单） |
| 明确不动 | `packages/core`、`server/`、`cli/`、非 Android 平台行为（§5.2、NFR-006） |
| 设计决策 | **DD-01…DD-12**（本文档自定，只落实需求，**不改变需求**；需要需求侧改动的见 §9） |

> 阅读方式：本文档不重复需求的理由与背景，只写"怎么做、落在哪、怎么验收、哪些还没定"。
> `REQ-xxx` / `NFR-xxx` / `AC-xx` / `DEC-xx` / `A-xx` / `Q-xx` 均指向 `docs/requirements.md`。

---

## 1. 目标与不变量

### 1.1 设计必须交付的四项（需求 §16.2 尾注）

1. **服务端生命周期状态机**（DEC-33）→ §4.3
2. **「可连接」探测规程**（需求 §14.5 的四条强制性质）→ §6.3
3. **追踪矩阵**（需求 §14.2.1）→ §6.1
4. **解锁前能力边界的核实结论与存储方案**（DEC-37）→ §4.4、§4.5、§6.4

### 1.2 不变量（任何设计分支都不得破坏）

| 编号 | 不变量 | 来源 |
|---|---|---|
| I-01 | 既有功能语义与默认值不变；唯一例外是"用户主动关闭「后台接收」**且应用不在前台**"会中断进行中的传输 | REQ-011、BR-10、DEC-18 |
| I-02 | 非 Android 平台行为不变；平台隔离判断（`_isSupported` 一类）不被改动 | NFR-006 |
| I-03 | 同一时刻：HTTP 服务端 ≤ 1 实例、常驻前台服务 ≤ 1 实例；两者与传输期服务/通知互不干扰 | DEC-02、REQ-002.4 |
| I-04 | 开关状态**不得呈现假象**：呈现必须对应"服务运行 **且** 监听可用" | BR-05、DEC-22 |
| I-05 | 不采用 1 像素 Activity / 无限 Alarm / 高频 WorkManager / 多进程互守 / 伪造通知 | NFR-001 |
| I-06 | 改动集中在允许清单内，且可整体回滚 | NFR-010 |

---

## 2. 现状与落点（含设计期核实的事实）

### 2.1 已核实的扩展点

| 事实 | 位置 | 设计含义 |
|---|---|---|
| 传输期前台服务封装：固定 `_channelId = localsend_foreground_service`、`_serviceId = 1`、`ForegroundServiceTypes.dataSync`、`allowWakeLock/WifiLock: true`、`_isSupported` = 仅 Android、串行 `FutureQueue` | `packages/localsend_isolates/lib/util/foreground_service.dart:8-9,32,51-87,146-173` | 常驻服务**另建**（DD-01）；唤醒锁/WiFi 锁已开，直接复用配置思路 |
| `flutter_foreground_task` **9.2.2**：`ForegroundServiceStatus` 与 `ForegroundServiceTypes` 都写在**插件的全局 prefs**，`ForegroundServiceTypes` 支持 `connectedDevice`（flag 1） | pub 缓存 `flutter_foreground_task-9.2.2/android/.../models/ForegroundServiceTypes.kt`、`.../service/ForegroundServiceManager.kt:31-76` | 插件是**单服务模型**（状态与类型都全局）⇒ 无法同时跑两个不同 `serviceTypes` 的插件服务（DD-01 的直接依据） |
| 插件自带清单贡献 `FOREGROUND_SERVICE`、`WAKE_LOCK`、`POST_NOTIFICATIONS`、**`RECEIVE_BOOT_COMPLETED`** 与 `RebootReceiver`（BOOT_COMPLETED / MY_PACKAGE_REPLACED / QUICKBOOT_POWERON） | 插件 `android/src/main/AndroidManifest.xml` | 见 §2.2 的事实更正；接收者仍需自建 |
| app 清单把插件服务声明为 `android:foregroundServiceType="dataSync"`，并注明服务名不可改 | `app/android/app/src/main/AndroidManifest.xml:93-96` | 传输期链路保持原样（I-01） |
| MethodChannel `org.localsend.localsend_app/localsend`，已承载 `pickDirectory` / `getFileDescriptor` / `requestLocalNetworkPermission` / `getDownloadsDirectory` 等 | `app/android/app/src/main/kotlin/org/localsend/localsend_app/MainActivity.kt:26,73-160` | 常驻服务的启停/查询走**同一渠道**新增方法（DD-02），不新建插件 |
| 服务端：`ServerService extends Notifier<ServerState?>`，方法 `startServerFromSettings()` / `startServer(...)` / `stopServer()` / `restartServerFromSettings()` / `ensureRunning()` | `app/lib/provider/network/server/server_provider.dart:65,102,115,229,238` | 状态机直接调用这些既有方法，不新增服务端能力 |
| 启动引导一次性调用 `startServerFromSettings()`（服务端随进程存活） | `app/lib/config/init.dart:178,208` | DEC-33 要求改为**按状态机决定**是否启动/停止（DD-03） |
| 生命周期：`LifeCycleWatcher` 转发**全部** `AppLifecycleState`；`resumed` 时重建 IP 并 `ensureRunning()`，`detached` 时释放 isolates | `app/lib/main.dart:57-80`、`app/lib/widget/watcher/life_cycle_watcher.dart:27-30` | 前台/后台判定的现成挂钩；`paused`/`hidden` 目前**无处理**，正是 DEC-33 要补的 |
| 设置：`SettingsService extends PureNotifier<SettingsState>`，每个字段一个 `setX(...)`，经 `persistenceProvider` 落盘 | `app/lib/provider/settings_provider.dart:41-215` | 两个开关按同一模式新增（DD-04） |
| 快捷磁贴 `QuickTileService` 跑在 `:quick_tile_service` 独立进程，注释明确"不保活" | `app/android/app/src/main/AndroidManifest.xml:78-89` | 本期不复用（需求 §3.1） |

### 2.2 设计期发现的两处事实更正（**需回报需求文档，见 §9**）

| 编号 | 需求文档原文 | 设计期核实结果 |
|---|---|---|
| **F-01** | §3.1："**无** `RECEIVE_BOOT_COMPLETED` 权限、**无**任何开机广播接收者（全仓 grep 无命中）" | 就**仓库内容**而言成立；但**合并后的清单**里该权限与一个 `RebootReceiver` **已由插件的清单贡献**（位于 pub 缓存，grep 仓库看不到）。影响：① 不需要新增该权限；② **接收者仍须自建** —— 插件的 receiver 只服务它自己的 `autoRunOnBoot` / `RestartReceiver` 逻辑（现状两者都关）。 |
| **F-02** | §3.2："HTTP 服务端在 App 启动引导时启动一次，其生命周期与 App 进程一致" | 仍然成立，但 DEC-33 要求它**由状态机接管**（前台/后台 × 开关），因此将新增"停止/重启服务端"的触发路径 —— 这不是新需求，是 DEC-33 的落实。 |

---

## 3. 总体结构

```
┌─ Flutter UI（app/lib/pages/**）──────────────────────────────┐
│  设置页两个开关 · 引导区 · 状态呈现（§8.3 的状态表）           │
└───────────────┬──────────────────────────────────────────────┘
                │ Refena providers / actions
┌───────────────▼──────────────────────────────────────────────┐
│ app/lib/provider/**                                          │
│  · ResidencyController（新）：状态机（DD-03）                  │
│  · ServerService（既有，不改语义）：start/stop/ensureRunning   │
│  · SettingsService（既有模式 +2 字段）                        │
└───────┬───────────────────────────────┬──────────────────────┘
        │ IsolateHttpServer*/Sync* 动作  │ MethodChannel（DD-02）
┌───────▼───────────────┐       ┌───────▼──────────────────────┐
│ localsend_isolates    │       │ Android 宿主层               │
│  · child/server_isolate│      │  · BackgroundReceiveService  │
│  · child/multicast_... │      │    （自建，connectedDevice）  │
│  · util/resident_*.dart│      │  · BackgroundBootReceiver    │
│    （新，常驻服务封装）│       │    （directBootAware）        │
└───────┬───────────────┘       │  · MainActivity：渠道新方法   │
        │ FRB                   └──────────────────────────────┘
┌───────▼──────────────────────┐
│ packages/core（Rust，**不改**）│  HTTP v2 服务端 · 组播发现
└──────────────────────────────┘
```

---

## 4. 关键设计

### 4.1 DD-01：常驻前台服务**自建**（不复用 `flutter_foreground_task`）

- **内容**：新增 `app/android/app/src/main/kotlin/org/localsend/localsend_app/BackgroundReceiveService.kt`，独立服务类、独立通知渠道（建议 id `localsend_background_receiver`），清单声明：

  ```xml
  <service
      android:name=".BackgroundReceiveService"
      android:foregroundServiceType="connectedDevice"
      android:exported="false" />
  ```

- **依据**：DEC-02 要求常驻服务与传输期服务**独立实例 + 独立渠道**；而插件 9.2.2 的 `ForegroundServiceStatus` / `ForegroundServiceTypes` 均为**全局**存储（§2.1），单个 Dart 侧 `startService` 会覆盖同一服务状态；且 app 已把插件服务固定声明为 `dataSync` 并注明不可改名。
- **备选（未采纳）**：用插件起第二个 `serviceId` —— 与上述单服务模型冲突；若复核发现插件另有支持多服务的 API，可退回该简化方案（标记为 DD-01-alt，需重新评估）。
- **Dart 侧封装**：`packages/localsend_isolates/lib/util/resident_service.dart`（新建，风格对齐 `foreground_service.dart`：静态方法 + `FutureQueue` 串行 + `_isSupported` 仅 Android + 失败只记日志不抛）。
- **通知**：低优先级、`onlyAlertOnce`、不可划掉不作为前提（需求 REQ-003"通知可划掉性"），提供"停止"动作。
- **回滚点**：整个服务类与清单条目可整体删除（I-06）。

### 4.2 DD-02：平台通道

- **内容**：在既有渠道 `org.localsend.localsend_app/localsend` 上新增方法：`residentServiceStart`、`residentServiceStop`、`residentServiceIsRunning`、`requestIgnoreBatteryOptimizations`（DD-06）、`openAutoStartSettings`（引导第 3 项跳转）。
- **依据**：`MainActivity.configureFlutterEngine` 已集中处理该渠道的方法表（§2.1），新增方法不引入新插件、不改插件代码。
- **约定**：所有方法返回 `bool` 或 `null`；**不抛异常**，失败经日志（NFR-007 的五问之一是"失败在哪一步"）。

### 4.3 DD-03：服务端生命周期状态机（**DEC-33 的交付物**）

**输入**：`fg ∈ {FG, BG}`（`AppLifecycleState`：`resumed` → FG；`paused` / `hidden` / `inactive` / `detached` → BG）、`sw ∈ {on, off}`（「后台接收」开关）、`userStopped ∈ {false, true}`（DEC-19/30）。

**输出（四个受控对象）**：HTTP 服务端（Rust，经 `ServerService`）、常驻前台服务 + 常驻通知（DD-01）、唤醒锁/WiFi 锁、恢复注册。

| # | fg | sw | userStopped | HTTP 服务端 | 常驻服务 + 通知 | 锁 | 恢复注册 | 依据 |
|---|---|---|---|---|---|---|---|---|
| S1 | FG | on | false | **运行** | 运行 | 持有 | 注册 | DEC-10 |
| S2 | FG | **off** | false | **运行（沿用现状）** | 不运行 | 不持有 | 不注册 | **DEC-33 / DEC-36** |
| S3 | BG | on | false | **运行** | 运行 | 持有 | 注册 | REQ-002、REQ-004 |
| S4 | BG | off | false | **停止** | 不运行 | 不持有 | 不注册 | **DEC-33**（关闭"不在前台时"停监听） |
| S5 | —（应用未运行） | on | **true** | 不运行 | 不运行 | — | — | **DEC-19 / DEC-30**（用户停止优先） |
| S6 | FG | 任意 | true | 运行 | 按 sw：on → 运行 | — | — | **DEC-30**（打开应用即恢复） |

**迁移事件与动作**

| 事件 | 动作 |
|---|---|
| 启动（进程起） | 读存储（§4.4）→ 按 S1…S6 中匹配的一行落地；`init.dart` 不再无条件 `startServerFromSettings()` |
| `resumed` | `fg = FG` → 若服务端未运行则启动（既有 `ensureRunning()` 语义保留）；若 `sw = on` 则启动常驻服务；`userStopped = false`（DEC-30） |
| `paused` / `hidden` | `fg = BG` → 若 `sw = off` 则**停止服务端**（这是本期新增的唯一"停监听"路径） |
| 开关 off（设置页，FG） | 停常驻服务与通知、释放锁与恢复注册；**不动服务端**（S2） |
| 开关 off（通知"停止"动作） | 若此刻 `fg = BG` → 同时停止服务端（S4）；**进行中的传输会被中断**（BR-10 的唯一例外）。**注意**：通知动作可在应用处于后台时被点击，这是"关闭动作**本身**发生在非前台"的情形；另一条路径是在前台关掉开关后再退到后台（由 `paused` 事件触发同样的 S4 动作） |
| 开关 on | 启动常驻服务 + 通知；确保服务端运行 |
| `LOCKED_BOOT_COMPLETED`（解锁前） | 若 `sw(on) ∧ autoStart(on) ∧ ¬userStopped` → 启动常驻服务与服务端（能力边界见 §6.4 的 K-03）；`fg = BG` |
| `BOOT_COMPLETED` / `USER_UNLOCKED` | 兜底：若上面未成功则重试一次；并做存储对齐（§4.4） |
| 系统级用户停止 | 标记 `userStopped = true`（下次启动时经 `ApplicationExitInfo.REASON_USER_REQUESTED` 判定，需求 REQ-003 途径 3） |
| 进程被杀 / 服务被回收 | 恢复机制（§4.6），有界 |

**不变式（可断言，供 M-07 使用）**
- 任何时刻服务端实例 ≤ 1、常驻服务实例 ≤ 1（`dumpsys activity services`）。
- `fg = BG ∧ sw = off` ⇒ 服务端不存在（`dumpsys` + 端口探测）。
- `fg = FG` ⇒ 服务端存在（与 `sw` 无关）。
- `sw = on ∧ fg = BG ∧ ¬userStopped` ⇒ 常驻服务存在且 `foregroundServiceType` 含 `connectedDevice`；传输期服务（若在传输）仍为 `dataSync`，两者并存。
- 停止常驻服务后无残留唤醒锁（`dumpsys power`）。

**落点**
- 新建 `app/lib/provider/network/residency/residency_controller.dart`（`NotifierProvider`，持有 `fg` / `sw` / `userStopped` 三个输入并驱动上表）；
- `app/lib/main.dart` 的 `LifeCycleWatcher.onChangedState` 改为把状态转交给该 controller（`resumed` → FG 迁移；`paused`/`hidden` → BG 迁移）；
- `app/lib/config/init.dart` 的 `startServerFromSettings()` 调用改为"经 controller 决定"；
- **不改** `ServerService` 的既有语义与签名（I-01）。

### 4.4 DD-04：开关与持久化

- **新增字段（2 + 1）**：
  | 字段 | 默认 | 用途 |
  |---|---|---|
  | `backgroundReceiveEnabled` | **true** | REQ-001（DEC-10；覆盖升级/清除数据同默认，DEC-23） |
  | `autoStartEnabled` | **true** | REQ-006（DEC-14、DEC-23） |
  | `userStoppedResident`（**新增，见下**） | false | DEC-19 / DEC-30 的"用户已停止"标记 |
  | 引导完成状态（逐项） | 未完成 | REQ-009（三态：已满足/未满足/无法检测） |

- **存储方案（DEC-37 的交付物）**：**主存 + 设备加密镜像**（DD-04）
  - 主存：沿用现有 `PersistenceService`（凭据加密存储，现状即如此）。
  - **镜像**：解锁前必须可读的三个值 —— `backgroundReceiveEnabled`、`autoStartEnabled`、`userStoppedResident` —— 以最小结构写入**设备加密存储**（`createDeviceProtectedStorageContext()` 下的独立 prefs 文件，建议文件名 `resident_flags`，键名 `re_` / `as_` / `us_` + 一个 `schema` 版本号）。
  - **对齐时机**：① 每次改开关时同步写镜像（App 在前台，两处都可写）；② `USER_UNLOCKED` / `BOOT_COMPLETED` 时校验并修复（镜像缺失 → 从主存补；不一致 → 以主存为准）。
  - **为什么不直接改主存位置**：把**全部**设置搬进设备加密存储会让用户数据（别名、历史、PIN 等）在解锁前暴露在弱保护存储中；镜像只承载 3 个布尔，风险面最小（NFR-008）。
  - **`userStoppedResident` 必须进镜像的理由（设计要点）**：否则"解锁前"无法判定用户是否已停止，DEC-19 的优先级（需求 DEC-37 的优先级说明）在解锁前的路径上**无法实现**。
- **迁移**：镜像缺失 → 按 DEFAULTS 写入并标记；主存缺失（新装/升级）→ 按 DEC-23 取 true。
- **授权**：键的授权见 DEC-27，已覆盖。

### 4.5 DD-05：Direct Boot 路径（DEC-37）

| 组件 | 声明 | 职责 |
|---|---|---|
| `BackgroundBootReceiver`（新） | `android:directBootAware="true"`，`exported="false"`，intent-filter：`LOCKED_BOOT_COMPLETED` + `BOOT_COMPLETED` | 读**镜像** → 决定是否启动常驻服务与服务端（S6/上面迁移表）；并把结果写日志（NFR-007） |
| `BackgroundUserUnlockedReceiver`（新，或合并到上者） | `exported="false"`，`ACTION_USER_UNLOCKED` | 存储对齐（§4.4）+ 引导状态刷新 + 触发一次 `resumed` 等价的检查 |
| Rust 侧服务端 | 不改 | 解锁前能否工作见 §6.4 的 K-03 |

- **启动序列**（解锁前）：系统广播 → receiver 读镜像 → 均开且未停止 → `startForegroundService(BackgroundReceiveService)` + 经 MethodChannel 通知 Dart 侧启动服务端（**或**改由常驻服务在原生侧直接拉起进程内的 Dart/isolate：二选一取决于 §6.4 的 K-03 —— 解锁前能否运行 isolates 与 TLS/目录相关能力）。**两条路线的选择时机：K-03 实测完成后**。
- **解锁后**：对齐 + 刷新 IP + 若引导未完成则提示（REQ-009）。
- **能力边界候选结论（**必须实测，见 §6.4**）**：解锁前可写设备加密存储、可发广播、可起前台服务（`directBootAware`）；**不确定**的是：TLS 证书生成是否需要凭据加密存储、下载目录是否可写、`ACCESS_LOCAL_NETWORK`（API 37+）在解锁前是否已授予生效。**若其中任一不成立，按需求 DEC-37 的要求回到需求文档修订，不得静默降级。**

### 4.6 DD-06：恢复机制（REQ-007 / REQ-010）

| 触发 | 机制 | 有界性 |
|---|---|---|
| 进程被杀 / 服务被回收 | ① 常驻服务 `onStartCommand` 返回 `START_STICKY`（是否被允许见 §6.4 的 K-06）；② 有界重试：`AlarmManager` 的单次重试或 `JobScheduler` 一次作业（**不得**高频轮询，NFR-001） | 重试次数上限与退避上限在实现时写常量并记日志；Android 16 的作业配额见需求 §3.4 P-06 |
| 网络变化（REQ-010） | `ConnectivityManager.registerDefaultNetworkCallback` 回调 → 重新确保服务端与组播在监听 | 回调驱动，无轮询 |
| 开机 | §4.5 的接收者 | 每启动一次 |

- **纯 Dart 侧不做任何定时器轮询**（I-05）。
- 每次尝试都写日志，允许回答"恢复尝试了几次、结果如何"（NFR-007）。

### 4.7 DD-07：探测规程（需求 §14.5 的四条强制性质）

| 性质 | 具体方案 |
|---|---|
| **可重复** | 提供脚本化入口：`adb forward tcp:53317 tcp:53317` + 一次自己的 HTTP 请求（走 v2 协议）、或"本机多实例"（模拟器 + 桌面构建，需求 §14.3 第 2 条）。探测**不依赖 UI** |
| **可清理** | 请求到达后由 Dart 侧走**拒绝**路径（既有 `declineFileRequest()`）或建立后立即 `cancelSession()`；探测结束用 `dumpsys` / 日志**断言无残留会话**（需求 §3.2：服务端同一时刻只允许一个上传会话） |
| **不干扰真实传输** | 探测前查询"是否存在活动会话"；存在则**跳过**本次探测（记为"跳过（不可判定）"，**不判失败**），也不得中断真实传输 |
| **成功判定** | 本机给出**协议合法的 prepare-upload 应答**（超时/连接失败/5xx 均算失败）；是否**受理**另行记录（DEC-05 另一半） |
| **发现判据（AC-01/02/10）** | **不得**用上面的直连探测替代：另需一次经组播发现链路的扫描（本机多实例或 `adb` 下的组播可达性，需求 §14.3 标注"需实测确认"）。**若 L1 无法可靠验证，则该判据记 L2 或"未验证 + 原因"**（DEC-35） |

- 该规程是**验收工件**，不进产品代码；其成功判定与清理要求是实现的验收前提。
- 与 AC-05"无任何交互"的关系：探测属允许，但受"不干扰"约束（需求 §14.5 第 5 条）。

### 4.8 DD-08：引导与状态呈现（DEC-21 / 22 / 24 / 30）

| 需求条目 | 设计 |
|---|---|
| 三项引导、三态取值 | 引导区（设置页内联，A-18）+ 首次启动提示；取值 `satisfied / unsatisfied / undetectable`；`undetectable` 提供"我已设置（用户自述）"入口（DEC-24） |
| 引导跳转 | ① 电池优化：`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`（DD-09）或 `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS`；② 后台不受限：`ACTION_APPLICATION_DETAILS_SETTINGS`；③ 厂商自启动：按 ROM 尝试已知组件名，失败则给文字说明（文案待定，Q-14） |
| "生效中"的口径 | 必须是"服务运行 **且** 监听可用"（I-04）；监听不可用 → 保持"开" + 显示"异常 / 暂不可被发现"（DEC-22） |
| 前台关闭 | 新增一行状态（需求 §8.3）：开关为关、无常驻通知、**接收仍可用**（DEC-36） |
| 系统停止后的恢复 | 打开应用即恢复（DEC-30）；呈现按 Q-30 的选项 ①（本次未决，见 §8 开放项）→ **设计暂按"**开关显示为开且确为运行中**"实现**，因为 DEC-30 已定"打开应用即恢复" |

### 4.9 DD-09：权限与清单（最小集 + 缺失降级）

| 权限 | 本期 | 缺失/被拒时的行为 |
|---|---|---|
| `FOREGROUND_SERVICE` | 已有（插件贡献） | — |
| `FOREGROUND_SERVICE_CONNECTED_DEVICE` | **新增** | 常驻服务启动即失败 → 回滚开关为关 + 提示（DEC-03） |
| `CHANGE_WIFI_MULTICAST_STATE`（或 `CHANGE_NETWORK_STATE` / `CHANGE_WIFI_STATE` 其一） | **新增**（普通权限，无弹窗，需求 §3.4 P-03/P-04） | 安装时授予，无运行时缺失路径 |
| `RECEIVE_BOOT_COMPLETED` | **已由插件贡献**（F-01） | — |
| `POST_NOTIFICATIONS` | 已有（插件请求） | 服务仍运行、通知不可见；状态不得误报（REQ-003 失败分支、NFR-005） |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | **DD-09：声明**（需求 §10.2 的"二选一"在此定为"声明"） | 未声明则只能跳转设置页列表。**选择理由**：直接弹窗对"关闭电池优化"这一关键前置的成功率更高；私有侧载不受 Play 政策约束（DEC-06）。**代价**：多一个安装时权限；如你认为应改走"只跳转"，见 §8 |
| `ACCESS_LOCAL_NETWORK` | 已有运行时请求逻辑 | 复用现状；失败时明确提示（§10.3） |
| 厂商自启动 | 不可声明 | 只能跳转 + 文字说明 |

### 4.10 DD-10：日志与可观测性（NFR-007 的五问）

- 事件集合（写入现有 `app/lib/provider/logging/**` 体系）：`resident.start.requested` / `resident.start.ok|failed(reason)` / `resident.stop` / `resident.listener.unavailable(reason)` / `resident.recovery.attempt(n)` / `resident.recovery.result` / `guide.state.changed(from,to)` / `resident.boot.received(locked|unlocked)` / `resident.storage.mirror.written|repaired`。
- **能回答**：① 是否启动成功、何时；② 失败在启动/权限/监听/网络哪一步；③ 恢复尝试几次、结果；④ 引导状态何时由何值变为何值；⑤ 用户停止/关闭动作是否记录（含来源：设置页 or 通知动作 or 系统 Stop）。
- 日志**不含**文件内容、密钥、证书指纹（NFR-007、§10.1）。

### 4.11 DD-11：隔离层与 Rust 侧改动边界

| 层 | 改动 |
|---|---|
| `packages/localsend_isolates/lib/util/` | 新增 `resident_service.dart`（DD-01 的 Dart 封装） |
| `packages/localsend_isolates/lib/src/isolate/` | **不改**（状态机在 app 侧；服务端启停复用既有 `IsolateHttpServerStartAction` / `StopAction`） |
| `packages/core`（Rust） | **不改** —— HTTP v2 服务端、组播、TLS 均已具备；本期不需要新端点（若探测需要新端点，改为在验收侧用既有协议完成，见 §4.7） |
| FRB 绑定 | **不改**（无新 Rust API）⇒ 不需要 `flutter_rust_bridge_codegen` |

### 4.12 DD-12：非 Android 平台

- 所有新增 Dart 代码经 `checkPlatform(...)` / `_isSupported` 短路（I-02）；`resident_service.dart` 在非 Android 上为 no-op。
- Kotlin/清单改动天然只在 Android。
- 验收：`NFR-006` 的静态隔离检查（允许清单内、非 Android 目录无行为代码改动）——见 §6.2。

---

## 5. 数据设计（键与结构）

| 数据 | 主存（凭据加密） | 镜像（设备加密，解锁前可读） | 默认 | 依据 |
|---|---|---|---|---|
| 后台接收开关 | `background_receive_enabled`（bool） | `re_bg` | true | DEC-10、DEC-23 |
| 开机自启开关 | `auto_start_enabled`（bool） | `re_as` | true | DEC-14、DEC-23 |
| 用户已停止标记 | `resident_user_stopped`（bool） | `re_us` | false | DEC-19、DEC-30 |
| 引导完成状态（逐项） | 逐项 bool + 时间戳（形态由实现定） | 不镜像 | 未完成 | REQ-009 |
| 镜像版本 | — | `re_schema` = 1 | 1 | §4.4 |

- 键名以实现为准，本文档确定"**三个布尔必须进镜像**"这一约束。
- 引导完成状态**不要求**解锁前可读（需求 §10.1）。

---

## 6. 验收设计

### 6.1 追踪矩阵（需求 §14.2.1 的交付物）

| 需求 | 用例 | 层级 | 判据 |
|---|---|---|---|
| REQ-001 开关 | S1/S2/S4 的行为 + 幂等 + 回滚 | L1 | 状态机不变式（§4.3）+ M-07 断言 |
| REQ-002 常驻服务 | 空闲 30 分钟仍在线；`foregroundServiceType=connectedDevice`；单实例 | L1 | `dumpsys activity services` 断言 |
| REQ-003 通知 | 两条通知共存；通知被清除不影响服务；三途径停止 | L1 | `dumpsys notification` + 状态断言 |
| REQ-004 后台/息屏 | AC-01 / AC-02 | L1（短时）+ L2 | §14.5 三类判据 |
| REQ-005 划掉任务 | AC-06 | **L2**（无可靠自动化，需求 §14.3） | 1 分钟内可连接 |
| REQ-006 开机自启 | AC-10、AC-13、以及"用户停止 → 重启 → 解锁前"组合 | **L2**（解锁前部分）+ L1（解锁后部分） | C 类窗口 30 分钟；组合情形要求"不恢复" |
| REQ-007 异常恢复 | AC-11（`am crash` / `kill -9`） | L1 | 30 分钟窗口 |
| REQ-008 关闭停止 | AC-12 | L1 | 无残留服务/锁；BG 时不可连接 |
| REQ-009 引导 | 三态显示、跳转、无法检测项的用户自述、跳过 | L1（UI + 断言） | 状态与交互断言 |
| REQ-010 网络恢复 | AC-08 | L1（`svc wifi`，需实测确认）+ L2 | 30 分钟窗口 |
| REQ-011 不回归 | 既有自动化检查 + 手工回归清单 | L1 | 全绿；**非 Android 构建检查为发布门槛（DEC-25）** |
| NFR-001 手段合法性 | 静态检索（无 1 像素 Activity / 无限 Alarm / 高频 WorkManager / 多进程互守） | L1 | 检索无命中 + 设计声明 |
| NFR-004 兼容 | 清单/gradle diff 审查；minSdk 未显式声明 | L1 | diff 断言 |
| NFR-005 通知合规 | `pm revoke POST_NOTIFICATIONS` 后状态不误报 | L1 | 状态断言 |
| NFR-006 隔离 | 允许清单静态检查（含 `app/lib/**`、`app/test/**`）；非 Android 构建 | L1 + **发布门槛** | 清单内 + 静态隔离 |
| NFR-007 可观测 | §4.10 的五问逐条用日志回答 | L1 | 五问齐全 |
| NFR-008 隐私 | 依赖清单 diff + 权限清单对照 | L1 | 无新增上报/SDK |
| NFR-009 文档与 i18n | 文案走 slang + `app/lib/gen` 同步；FOSS 标记保留 | L1 | 生成码一致 + 标记存在 |
| NFR-010 回滚 | 任务起点..HEAD 的 diff 在允许清单内 | L1 | diff 断言 |
| NFR-002/003 耗电与资源 | 只采集证据 | **证据采集（不判定）** | 记录数值，不与阈值比较 |

### 6.2 L1 用例清单（每个任务必须执行 M-01…M-07，需求 §14.2）

- M-01…M-06 照需求执行；**M-07 的专项断言**按 §4.3 的不变式逐条声明（例：任务"常驻服务" → 断言 `dumpsys activity services` 中存在 `foregroundServiceType=connectedDevice` 的服务且仅一个）。
- 既有自动化检查（`fvm dart format` / `analyze` / `test`）在允许清单内运行（DEC-28 已授权）。

### 6.3 探测规程（**验收工件**）

见 §4.7；实现时以脚本形式落在 `support/scripts/` 之外（**不得**写入产品代码路径；建议 `support/scripts/verify/`，需在实现任务中确认目录归属）。

### 6.4 待实测清单（决定验收层级，**必须在实现前完成**）

> 编号用 **K-xx**（核实项），以免与需求文档 §20 的复核编号 `R-xx` / `V-xx` 混淆。

| 编号 | 待核实 | 方法 | 影响 |
|---|---|---|---|
| K-01 | 模拟器镜像是否采用文件级加密 | `adb shell getprop ro.crypto.type` | 决定 AC-10 的解锁前部分能否进 L1（需求 §14.3、复核 05） |
| K-02 | 锁屏重启后能否被探测（解锁前） | `adb reboot` + 保持未解锁 + 探测 | 同上 |
| K-03 | 解锁前能否启动前台服务 / 运行 isolates / 生成 TLS 证书 / 写下载目录 / 使用局域网权限 | 逐项实测 | 决定 §4.5 的启动序列是否可行；**不成立则回需求修订** |
| K-04 | `svc wifi disable/enable` 在 API 36 模拟器上的行为 | 实测 | 决定 AC-08 的 L1 可行性 |
| K-05 | `cmd activity stop-app` 的实际语义 | 实测 | 决定"用户停止"用例的 L1 手段（需求 §14.3 已标注未实测） |
| K-06 | `START_STICKY` 对 `connectedDevice` 服务是否被允许 | 实测 | 决定 §4.6 机制①的可用性 |

### 6.5 L2 用例清单（发布前一次性）

- AC-03 / AC-04 / AC-05（长跑 8 h / 24 h / 72 h）
- AC-06（划掉任务）
- AC-09（真实路由器重启）
- **AC-10 的解锁前部分**（若 K-01/K-02 判定 L1 不可行）
- "用户停止 → 重启 → 解锁前"组合的解锁前部分
- 真实组播发现（跨主机）
- 耗电 / 资源证据采集（NFR-002/003）

---

## 7. 实现任务分解（建议顺序，每个任务一个提交，L1 可验收）

| # | 任务 | 交付 | 验收（L1） |
|---|---|---|---|
| T1 | **待实测清单 K-01…K-06** | 实测结论回填本文档 §6.4 | 结果与证据；**不修改任何产品代码** |
| T2 | 存储与开关（DD-04） | 两个开关 + `userStoppedResident` + 设备加密镜像 + 迁移 | 单元测试：默认值三情形、镜像读写与对齐 |
| T3 | 常驻服务与渠道（DD-01、DD-02） | Kotlin 服务 + Dart 封装 + 渠道方法 | 服务可启停；`dumpsys` 断言；单实例 |
| T4 | 状态机（DD-03） | `ResidencyController` + 生命周期挂钩 | 四行状态表 + 不变式断言（M-07） |
| T5 | 通知与状态呈现（DD-08） | 独立渠道通知 + 状态表映射 | 两通知共存；状态不误报 |
| T6 | 引导（DD-08） | 三项引导 + 三态 + 用户自述 | UI 用例 + 跳转断言 |
| T7 | 恢复机制（DD-06） | 有界重试 + 网络回调 | AC-11 的 L1 近似 |
| T8 | Direct Boot（DD-05） | 接收者 + 解锁前启动路径 | 依 K-01…K-03 的结论决定层级 |
| T9 | 探测规程与追踪矩阵（§4.7、§6.1） | 规程脚本 + 矩阵文档 | 规程可重复执行且可清理 |
| T10 | 文案与 i18n（NFR-009） | slang 文案 + 生成码 | 格式/分析/测试全绿（Q-06 前需授权写翻译文件） |

> 顺序依据：T1 决定 T8 的可行性；T2/T3 是 T4 的前置；T7 依赖 T3。

---

## 8. 风险、假设与开放项

### 8.1 设计假设（沿用需求侧的未确认假设，**不得写成已确认**）

| 编号 | 假设 | 影响面 | 若被否决 |
|---|---|---|---|
| A-16 | 覆盖安装后不自动恢复 | T8 | 需新增触发路径与用例 |
| A-18 | Q-13/Q-14/Q-17 的临时取向 | T5、T6 | 对应 UI 返工 |
| A-19 | 并发关闭取"最后一次用户意图优先" | T4 | 状态机需另设计 |
| A-20 | 负向判据用 30 分钟窗口 | §6.1 | 判定表调整 |

### 8.2 需要所有者确认的设计选择

| 编号 | 选择 | 备选 | 若不认可 |
|---|---|---|---|
| DD-09 | `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` **声明并直接弹窗** | 只跳转设置页列表 | 删一行权限 + 改跳转 |
| DD-04 | 主存 + **最小镜像**（3 个布尔） | 直接把全部设置搬到设备加密存储 | 需重评隐私面（NFR-008） |
| DD-01 | 常驻服务**自建 Kotlin 服务** | 复用插件（判定不可行，见 §4.1） | 若复核发现插件支持多服务，可简化 |
| Q-30 | 系统停止后：打开应用即恢复、开关显示为"开" | 显示为"关" + 提示 | 需求侧 Q-30 未落定，见 §9 |

### 8.3 设计期风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| K-03 判定解锁前能力不可用 | DEC-37 的承诺无法完全兑现 | **按需求要求回文档修订**，不静默降级 |
| Rust 服务端在解锁前拿不到凭据加密存储（TLS 证书） | 解锁前只能"可被发现"（组播）而不能"可连接" | 需需求侧决定是否把判据降为"可被发现"（§9） |
| 常驻服务与传输期服务同时运行占用资源 | NFR-002/003 证据变差 | 只观测，不设阈值（DEC-07） |
| 前台关闭后（S2）服务端仍在运行，与"关闭"的直觉不符 | 用户困惑 | 界面文案明确"仅停止后台常驻"（T5） |

---

## 9. 需要回报需求文档的事项（**待确认，未修改需求**）

| 编号 | 事项 | 类型 |
|---|---|---|
| F-01 | 需求 §3.1 的"无 `RECEIVE_BOOT_COMPLETED` / 无开机接收者"需更正为"插件清单已贡献该权限与一个内部接收者；本项目仍需自建接收者" | 事实更正 |
| F-02 | 需求 §3.2 关于服务端生命周期的描述需补一句"自 DEC-33 起由状态机接管" | 事实补充 |
| R-01 | **Q-30 仍未落定**（系统停止后的界面呈现）：本设计暂按 DEC-30 的"打开应用即恢复、开关显示为开"实现 | 开放项 |
| R-02 | 若 K-03 判定解锁前无法"可连接"，则 AC-10 的判据需回到需求文档重新决定（可被发现 vs 可连接） | 潜在需求变更 |
| R-03 | 探测脚本目录归属（建议 `support/scripts/verify/`）需确认 | 工程约定 |

---

## 10. 变更记录

| 版本 | 日期 | 内容 |
|---|---|---|
| v0.1 | 2026-09-23 | 首版：依据 requirements v1.6 起草；含状态机、存储、Direct Boot、探测规程、追踪矩阵、任务分解与待实测清单 |
