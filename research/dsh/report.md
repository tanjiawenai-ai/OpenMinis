# OpenMinis 源码全量深读：最终报告

- 任务：J-20261127-0345-minis3d（第 3 轮 D，汇总）
- 账本：`work/coverage-ledger.csv`，sha256 `de878d03ca3e56cdad2dbd6e4d1fec6608f2302a23ba46250b46ba5687c6e06e`
- 口径：只读账本、三轮笔记与批次日志，不重读源码；文中所有覆盖数字由我按账本重算。

## 1. 结论先行

- 在册文件 **3,535 / 3,535**，覆盖 **100.0%**。
- 在册行数 **2,040,689 / 2,040,689**，覆盖 **100.0%**。
- 未读：**0 文件 / 0 行**。
- 范围外：`deps/` 4,378 文件 / 1,278,278 行，不计入。
- 这里"100%"的准确含义：3,535 个在册文件全部完成**静态核对**。文本逐行读或分段全读；101 个二进制/空文件做结构核验（类型、大小、sha256）；数据资产做行数、编码、哈希核验，不做语义逐行读。它不等于运行时验证，也不等于每一行都被读懂。

账本重算结果与任务 C 的申报一致：3,535 文件、2,040,689 行、unread 0。

## 2. 覆盖明细

### 2.1 按轮次

| 轮次 / 任务 | 文件 | 行数 |
|---|---:|---:|
| 第 1 轮（深读） | 75 | 72,946 |
| 第 2 轮 | 987 | 342,644 |
| 第 3 轮 A（Android 余量） | 1,332 | 526,033 |
| 第 3 轮 B（iOS 代码） | 245 | 191,161 |
| 第 3 轮 B2 | 128 | 56,276 |
| 第 3 轮 B3 | 82 | 17,730 |
| 第 3 轮 B4 | 52 | 13,812 |
| 第 3 轮 B5 | 70 | 27,565 |
| 第 3 轮 B6 | 92 | 44,897 |
| 第 3 轮 C（MinisTests/xcstrings/资源） | 472 | 747,625 |
| **合计** | **3,535** | **2,040,689** |

第 3 轮合计 2,473 文件 / 1,625,099 行，占全部行数的 79.6%。

### 2.2 按领域

| 领域 | 文件 | 行数 | 行占比 | 未读 |
|---|---:|---:|---:|---:|
| iOS 应用代码 | 778 | 441,083 | 21.6% | 0 |
| Android 资源资产（assets） | 47 | 351,644 | 17.2% | 0 |
| iOS 资源/资产 | 71 | 349,880 | 17.1% | 0 |
| Android 应用代码 | 803 | 296,747 | 14.5% | 0 |
| iOS xcstrings | 1 | 282,658 | 13.9% | 0 |
| Android 测试 | 767 | 126,997 | 6.2% | 0 |
| iOS MinisTests | 427 | 111,686 | 5.5% | 0 |
| Android res/locales | 549 | 58,312 | 2.9% | 0 |
| shared / docs / scripts | 55 | 16,382 | 0.8% | 0 |
| iOS Vendor（cppjieba 等） | 37 | 5,300 | 0.3% | 0 |
| **合计** | **3,535** | **2,040,689** | **100%** | **0** |

说明：Android assets 与 iOS Resources 的行数主要来自两份 348,982 行的 jieba 词典和 models.dev 目录，故文件少、行数大。

## 3. 架构理解

一句话：OpenMinis 是把 Linux 沙箱装进手机的端侧 Agent 应用。Android 用 Kotlin/Compose，iOS 用 SwiftUI，两端各自内置 Alpine rootfs，模型通过命令行工具操作沙箱和原生能力。

- **双端同构，不是共享 UI。** Android 与 iOS 各自实现导航、会话、设置；共享的是 `src/shared/` 下的协议文档和少量数据契约（`src/shared/cli/`、`src/shared/network/`）。多份共享文档写的 "iOS 参考实现" 在代码里并不存在，见第 6 节。
- **Android 启动顺序是硬契约。** `MinisApp.onCreate` 先建数据库和仓库，再把 `subsystemsInitialized` 置真（`MinisApp.kt:530-607`）；`MainActivity` 在该标志为假时只显示崩溃分享对话框，不进入 Compose（`MainActivity.kt:252-290`）。
- **Android 沙箱是 PRoot。** 每次命令拼接同一套 proot argv（`FreshProcessShell.kt:253-286`）；`/var/minis` 挂载由 `SessionMounts.build` 决定（`SessionMounts.kt:64-117`）。
- **原生能力走 native offload。** 小端分帧的抽象 socket 协议，按可执行文件名分发，限同 uid（`NativeOffload.kt:445-504`）。Android 注册 20 个 handler（`MinisApp.kt:748-814`），iOS 侧每个 `.m` 用 `native_offload_add_handler` 注册（如 `AlarmOffload.m:562-570`）。
- **iOS 沙箱是 iSH。** `ISHKernel.m` 是内核门面；fork 前有内存护栏，避免 fork 风暴触发 jetsam（`ISHKernel.m:99-190`）。
- **Agent 循环两端各自实现。** iOS 入口是 `runAgentLoop` → `runAgentLoopCore`，一个约 2,300 行的单函数，按"裁剪/卸载/压缩 → 请求 → 流式 → 工具批 → 提交"迭代（`AIChatViewModel+AgentLoop.swift:106`）。防死循环用优先级阶梯：未知工具 → 全局熔断 → 无进展轮询 → 通用重复（`ToolLoopDetector.swift:93`；Android 对应 `ToolLoopDetector.kt:180`）。
- **Provider 层是适配器 + 线上改写。** Anthropic 请求在 `AnthropicAgentProvider` 组装参数（`AnthropicAgentProvider.swift:1204`），再由 `OAuthHTTPClient` 在编码后的 JSON 上打补丁（`OAuthHTTPClient.swift:1897-1934`）。代价是若干进程级全局状态，见第 5 节。
- **存储与同步。** Provider 配置存 SQLite，经 `bulkReplace` 与 iCloud 对账（`ProviderConfigDB`，风险见第 5 节）；端上同时存在 v1 `CloudSyncEngine` 与 v2 `SyncCore` 两套同步引擎，靠 `SyncV2Bootstrap.shouldPauseV1()` 互斥（`MinisApp.swift:463-489`）。
- **测试是两套体系。** Android 767 个测试文件（含 `test/resources` 与 `androidTest`），iOS 427 个 MinisTests 文件。相当比例是独立 `swift` 脚本或源码文本 grep，不是对生产代码的调用，见第 5 节。

## 4. 数据资产单独列账

以下行是**结构核验**，不是逐行读。它们计入 100% 文件/行覆盖，但统计口径必须与"逐行读"分开。

### 4.1 文本型数据资产（全部重算行数、字节、sha256）

| 文件 | 行数 | 字节 | sha256 |
|---|---:|---:|---|
| `src/android/app/src/main/assets/jieba/jieba.dict.utf8` | 348,982 | 5,071,207 | `6f7d4350e8861ef4139b2e3a6fad05430c19ae71f4b8378190edecac8aae2e6a` |
| `src/ios/Resources/jieba.dict.utf8` | 348,982 | 5,071,207 | 同上（字节级相同） |
| `src/android/app/src/main/assets/jieba/hmm_model.utf8` | 34 | 519,739 | `f17790586ac86dd048c8adffed052c4bd2b28ed0682972c1275e59040c0589a7` |
| `src/ios/Resources/hmm_model.utf8` | 34 | 519,739 | 同上 |
| `src/ios/Shared/cl100k_base.tiktoken` | 100,256 | 1,681,126 | `223921b76ee99bde995b7ff738513eef100fb51d18c93597a113bcffe865b2a7` |
| `src/android/app/src/main/assets/models-dev-api.json` | 1 | 5,233,322 | `6c4aa6f49ec7f38379b0ac5823e1ef9d8b47a1a34dfe566ae05fb50e80544a01` |
| `src/ios/Resources/models-dev-api.json` | 1 | 5,233,322 | 同上 |
| `src/ios/Resources/KaTeX/katex.min.css` | 1 | 21,238 | `4dfb1340464392a596cfdafe6d0443a69123b0f8fbae97e105e6bdd2b2599813` |
| `src/ios/Resources/KaTeX/katex.min.js` | 1 | 276,574 | `863811e2baa0849c77bc92d26d44f4d0a4843c2a8ab52c462017dea57316e4d8` |
| `src/ios/Resources/KaTeX/mhchem.min.js` | 1 | 33,712 | `9f87e5e9c384a160472d0045035a8641f6013358eddb3ece708634a50f946a40` |
| `src/ios/Resources/zh_background_wordfreq.txt` | 204 | 2,180 | `4a013c69a686d36121e53db29098542bf237090f04585363acef7db98afd27f3` |

核验内容：jieba 词典 348,982 行全部为 `词 频次 词性` 三列，0 重复行，UTF-8 合法，LF 换行；hmm 为 34 行 B/M/E/S 发射概率表；tiktoken 每行 `base64(token) rank`，rank 恰为 0…100,255；models.dev 目录 225 个 provider、8,279 个模型；KaTeX 三个文件是压缩单行，UTF-8。全量 112 行资产索引（含 PNG/TTF/so/jar/apk 与 4 个空 `__init__.py`）在 `work/asset-index.csv`，由我从源码树重算。

`src/ios/Localizable.xcstrings`（282,658 行 / 7.7 MB）不属于结构核验：它按 24 段全读，并用 `json.load` 做整文件校验（3,137 个 key、18 个 locale）。

### 4.2 二进制与空文件（按类分组，全部 file(1) + 字节 + sha256）

| 类别 | 文件数 | 总字节 | 说明 |
|---|---:|---:|---|
| PNG | 47 | 28,228,295 | Android mipmap 30 + iOS Assets 13 + iOS AlternateIcons 4 |
| TTF 字体 | 42 | 1,579,056 | Android katex 20 + iOS KaTeX 20 + JetBrains Mono 2 |
| `.apk` | 5 | 987,658 | Alpine 3.21 busybox/musl/musl-utils/ssl_client/busybox-binsh |
| `.so` | 2 | 24,380 | `libproot-loader.so`、`libproot-loader32.so` |
| `.jar` | 1 | 43,583 | `gradle-wrapper.jar` |
| 空 `__init__.py` | 4 | 0 | 0 字节包标记 |
| **合计** | **101** | — | 账本 `lines=0` 的全部行 |

### 4.3 重复内容：读一次，按结构记为两份

- **跨平台重复。** Android 与 iOS 各存一份 jieba 词典、hmm 模型、models.dev 目录，sha256 两两相同；20 个 KaTeX TTF 也两两相同。内容只读一次，另一端按"同一 sha256 的副本"记账，不重复消耗阅读。
- **cppjieba 自带重复头文件。** `src/ios/Vendor/cppjieba/include/*.hpp` 与 `include/cppjieba/*.hpp` 共 18 对，sha256 **18/18 完全一致**，每侧 2,640 行。正文只读一份，另一份记为结构副本。Vendor 全部 37 文件 / 5,300 行 = 18 对（5,280 行）+ LICENSE（20 行）。

## 5. 关键发现与纠正

### 5.1 原生层缺陷

- **cppjieba `DeleteNode` 是 use-after-erase。** 先 `erase` 再解引用失效迭代器的值（`Trie.hpp:163-166`）。当前无人调用 `DeleteUserWord`，所以不可达，但一旦启用即 UB。
- **词典损坏无法按契约返回失败。** `XCHECK`/`Logger` 直接 abort（`Utils.hpp:32,107`），`jieba_jni.cpp:113-121` 的 "返回 false" 契约守不住，坏词典会杀进程。
- **崩溃处理器有栈缓冲越界写风险。** `crash_handler.cpp:527-534` 用 `snprintf` 的"将写长度"推进指针，未按栈缓冲上限截断。
- **pty 桥信任 Kotlin 传入的长度。** `pty_bridge.c:187` 不校验 `GetArrayLength`，`off/len` 越界即越界读。
- **HealthKit offload 枚举错位。** `resolveCategoryValue` 先返回 0 基名字下标，后面的显式枚举映射永远走不到（`HealthKitOffload.m:2635`），睡眠阶段/严重度被写错位。

### 5.2 测试质量：覆盖数不等于验证力

- **大量测试不碰生产代码。** test-ui-01 有 20/34、test-ui-02 有 8/40 属于源码文本 grep、恒真断言或测试内重新实现。例：`AgentBlockRenderingTest.kt:49` 比较两个字面量；`EnqueuePromptIdleWindowTest.kt:64` 用本地模型，从不触碰 `ChatViewModel`。
- **iOS MinisTests 以独立脚本为主。** 427 个文件中 322 个是 standalone `swift` 脚本、48 个是 golden JSON。部分脚本直接编译生产代码块（强），更多是手工移植逻辑再 grep 源码（易漂移），且源文件读不到时会静默跳过并仍以 0 退出。
- 影响：这些测试证明"文本里有这行代码"，不证明"行为正确"。任何以测试数推导质量的说法都要打折。

### 5.3 Provider / 路由

- **自定义模型会丢推理档位。** `ModelEntry.model` 在存在用户 override 时丢掉 `reasoningEffortValues`、`declaresNoEffortTiers`、`effortDeclarationIsAuthoritative`，选择器和线上 clamp 同时失去目录声明的档位（`r3-ios-Providers-03`）。
- **Antigravity 凭证判断错误。** `computeHasAnyCredential` 把 `.antigravity` 归入 `return false`，但 `AntigravityOAuthManager` 确实持久化了 token（`r3-ios-Providers-06`）。结果是已登录实例被判"未配置"，被 `ModelGroupRouter.availableEntryIds` 从所有分组过滤掉。
- **`bulkReplace` 有级联删除风险。** 对 `provider_instances` 用 `INSERT OR REPLACE`，在外键 `ON DELETE CASCADE` 下会删掉该实例的 entries；entries 循环随后把"内容相同"的行 `continue` 掉，不重新插入。需要一次复现才能定级，安全写法是先子后父或 `ON CONFLICT DO UPDATE`（`r3-ios-Providers-05`）。
- **WebApp 路径校验弱于解析器。** `WebAppWebViewController` 用 `hasPrefix(root)` 且只做词法 `standardizedFileURL`，同前缀兄弟目录和 scope 内符号链接都能逃逸；解析器本身已用 `resolvingSymlinksInPath()` 堵住同一个洞（`r3-ios-WebApp.md` 附录）。

### 5.4 本地化

- **iOS 目录覆盖极不均。** 3,137 个 key / 18 locale，只有 2,183 个 key 有全部 18 种语言；609 个 key 只有 3 种（多为 en + 简繁）；152 个 key 有译文但无 en。`Launch Screen.storyboard` 的标语硬编码英文，18 个 locale 全不翻译（`Launch Screen.storyboard:20-43`）。
- **Android 大语种缺 key。** de/es 各缺 506 个基础 key（key 覆盖率 79.6%），文件停在 MCP OAuth 段中间（`values-de/strings.xml:1992`）；fil 缺 521、fr 缺 488。`strings_network.xml` 在 cs/fa/fil/fr 只有 21/108 个 key，这些 locale 的设置页网络部分基本是英文。

### 5.5 安全与腐化

- **Android 默认信任用户 CA 且全局允许明文。** `network_security_config.xml:7-12` 对自签名/局域网 endpoint 开放，只有 11 个官方 host 被钉到系统 CA + 禁止明文（`:31-46`）。
- **iOS 全局关闭 ATS。** `NSAllowsArbitraryLoads=true` 覆盖所有请求，包括 provider API 和 MCP（`Info.plist:197`）。
- **调试服务器监听 `INADDR_ANY`。** `DebugServer.swift:25` 绑定所有网卡，`/schema`、`/skill` 和客户端源码无鉴权（`:164-202`），仅 DEBUG 构建。
- **工具链腐化。** `gradlew:121,130` 的注释被自动替换脚本打坏（`stracks://am the program://ent://s by://`），功能代码未受影响。`Minis.entitlements:5` 把 `aps-environment` 写死为 `development`，归档时依赖 Xcode 覆盖，属发布期隐患。

### 5.6 账本纠正：第 2 轮少记 6 个文件

`log-r3.md:39-45` 记录：6 个标为 unread 的行其实在第 2 轮笔记里已有完整小节——`crash/NativeCrashHandler.kt`、`crash/CrashFileReporter.kt`、`backup/BackupZip.kt`、`service/OAuthWaitKeepAlive.kt`、`service/DynamicIslandSupport.kt`、`scripts/ios-unit-tests/data_guard.py`。第 3 轮把这 6 个文件重读了一遍，账本仍按 `read_round3` 记，未回改第 2 轮。因此第 2 轮账本数 987 是**略低**，实际至少 993；其自报的 1,062 文件 / 415,590 行包含第 1 轮的 75 文件 / 72,946 行（75+987=1,062，72,946+342,644=415,590），与本账本不矛盾。

## 6. 局限与未决

- **只有静态读，没有构建和运行。** 所有"会崩溃/会丢数据/会误判"的判断都来自代码路径，未在真机或模拟器复现。`bulkReplace` 级联删除尤其需要一次可重复的复现才能定级。
- **结构核验不是语义读。** jieba 词典只校验格式、行数、哈希和抽样；models.dev 只校验 JSON 结构与计数；KaTeX 压缩单行无法逐行读。它们能证明"文件完整且是它声称的东西"，不能证明"每个词条/每条价格都对"。
- **文档与代码互相矛盾。** `src/shared/cli/bonjour.md`、`network/README.md`、`markdown/html-tags.md`、`sessions-update.md` 都把 iOS 说成参考实现，但对应源码或测试不存在。这类文档只能当"设计意图"，不能当实现证据。
- **`deps/` 是明确的范围外。** 4,378 文件 / 1,278,278 行（rclone、FFmpeg、Tailscale、iSH 上游等）按任务约定排除。因此本报告的 100% 只覆盖应用自有代码与随包资产，不含第三方源码。
- **101 个零行文件计入"已核验"但不计行。** 它们对文件覆盖率有贡献，对行覆盖率没有。报告已把两类口径分开列。

## 7. 附录

### 7.1 任务链

| 任务 | 范围 | 文件 | 行数 | 余额（CNY） |
|---|---|---:|---:|---|
| A | Android 余量 | 1,332 | 526,033 | 50.34 → 41.78 |
| B | iOS 代码 | 245 | 191,161 | 41.68 → 37.10 |
| B2 | iOS 续读 | 128 | 56,276 | 37.10 → 35.10 |
| B3 | iOS 续读 | 82 | 17,730 | 34.55 → 34.03 |
| B4 | iOS 续读 | 52 | 13,812 | 33.66 → 33.47 |
| B5 | iOS 续读 | 70 | 27,565 | 33.23 → 32.70 |
| B6 | iOS 续读 | 92 | 44,897 | 32.25 → 31.22 |
| C | MinisTests / xcstrings / 资源 | 472 | 747,625 | 30.70 → 19.85 |
| D | 汇总（本任务） | — | — | 19.65 → 见 `batch-result.json` |

第 3 轮起点余额 50.37 CNY，任务 C 结束 19.85 CNY。B 的 `batch-result.json` 缺失，其 245 / 191,161 由 B2 的剩余量反算（669−424 文件，351,441−160,280 行），与 B 日志"landing 245 files"一致。

### 7.2 逐任务 batch-result 计数

- A：`scope_files_done` 1,332，`scope_lines_done` 526,033。
- B：无 batch-result（任务被传输中断杀死）；按 B2 起始剩余量反算 245 / 191,161。
- B2：128 / 56,276；B3：82 / 17,730；B4：52 / 13,812；B5：70 / 27,565；B6：92 / 44,897。
- C：472 / 747,625；`unread_remaining` 0；树校验 472 行、448 个文本行数精确匹配、24 个二进制、0 不匹配。
- D：报告 1 份，账本重算 3,535 / 2,040,689 / unread 0。

### 7.3 产物

- `work/report.md`（本文件）
- `work/batch-result.json`（机器可读汇总）
- `work/asset-index.csv`（112 行数据资产索引，含行数/字节/sha256）
- `work/log-r3d.md`（本任务日志）
