# OpenMinis 全量深读总报告（缪斯线）

- 对象：OpenMinis/OpenMinis，HEAD `1a71df0f381cefde80bf912b723de6f9831bc177`
- 日期：2026-11-26
- 读法：全量台账＋29 个分区互不重叠逐文件逐行读＋逐区销账；分区总结在同目录 `partition-P01.md`～`partition-P29.md`
- 本报告只依据实际读到的源码下判断；没逐行读的一律单列，不充数。

## 结论先行

1. **Minis 是一个成熟的「单机 Agent 客户端」，不是「个人 Agent 服务」。** 它缺的不是功能，而是三根骨架：统一账号与跨设备数据面、带来源的记忆、系统级审批与审计。三根骨架补齐后，用户现有的 agy（Antigravity Pro）、DeepSeek API、CommandCode 中转都能接上，不需要 Anthropic API 套餐。
2. **安卓端 Antigravity 是死代码**（`ProviderConfig.kt` 中 `isUsable=false`、仅可解码不可用；OAuth 管理器自述 scaffolding、client secret 是占位符）。iOS 端 Antigravity/Gemini 的 OAuth 同样出厂即坏（占位密钥）。**任何「在设备上直连谷歌」的方案都建立在流沙上。**
3. **服务端网关路线成立，但网关不能是傻代理。** 源码证据三条：①设备端谷歌 OAuth 结构性损坏（上条）；②Antigravity 走 Google 未公开的 `v1internal` 接口，要伪造客户端身份、信封包装、thoughtSignature 逐消息回放（P24），在 Kotlin 里复刻风险极高，而云主机上的 agy 已经握着一个能用的谷歌会话；③App 本来就会说 OpenAI 协议且支持自定义 baseURL（P13/P28），网关以 OpenAI 兼容面貌出现时 App 侧几乎零改动。**但**网关必须按会话保持上游身份、原样透传 reasoning 产物（加密 reasoning 只能回放给签发它的上游，issue #368，P13），否则工具调用配对和思考回放会静默损坏——这是我与「纯转发网关」方案的分歧点，见 §5。
4. **App 里已经内嵌 Tailscale 节点**（rclone 的 gomobile 库同一个 Go 运行时里带 tsnet 节点，deps 构建脚本实证；App 侧有按作用域选直连/Tailscale/代理的网络层，P28）。手机够到云主机网关的传输地基是现成的，不用另铺。
5. **权限治理的真问题不是「默认全放行」这么简单，而是闸门覆盖不一致**：只有剪贴板/联系人/日历/定位/照片五个 offload 过闸，通知（能读所有 App 通知）、会话全文、定时任务、BrowserUse、ModelUse 根本没有闸门检查（P16）；shell/文件/浏览器三类动作没有审批入口（DSH 线同判）。改默认值只是一行，补全闸门覆盖才是治理期的真活。
6. **调试通道是出厂最大的安全敞口**：安卓 DebugServer 监听 0.0.0.0 明文 JSON-RPC、`provider.export` 直接回吐 base64 包裹的真实凭据（P27）；iOS 的 DebugJSONRPC（3,531 行）是全能控制通道，只靠 DEBUG 编译开关和令牌层挡着（P02）。开发版第一件事就是把这条通道关进笼子。
7. **这套代码的工程纪律远超同类开源项目**：约 1,500+ 个安卓 JVM 回归测试＋大量 iOS 独立测试，字节级跨平台对等、反空转（先复现旧 bug 再断言已修）是系统性做法。代价是大量测试靠「源码文本钉死＋本地镜像模型」，行为漂移时测试仍可能全绿（P01/P19）——改造时不能把测试全绿当行为正确。
8. **两个上帝对象决定改造难度**：安卓 `ChatViewModel.kt` 20,341 行单类（循环＋工具＋提示词＋压缩＋落库，P14）、`ChatScreen.kt` 8,760 行（P12）；iOS `ProviderConfigStore.swift` 4,068 行（P04）。任何治理/记忆改造都要穿过它们，策略只能是「先旁路、后收编」，不能正面重构。

## 覆盖率（确数，台账为准）

| 类别 | 文件 | 行数 | 状态 |
|---|---|---|---|
| 自有代码（kt/swift/m/mm/h/hpp/cpp/c/py/sh/kts/md/plist 等） | 2,773 | 851,688 | **逐行读完 100%**（29 分区销账，P09 经补读后 111/111） |
| 数据文件（本地化 .utf8/.xcstrings、tiktoken 词表、json/xml 资源） | 601 | 1,179,277 | 全量清点分类，未逐行读（属语料/资源数据，非逻辑代码） |
| 二进制（png/ttf/apk/jar 等） | 97 | 104,230 | 全量清点，未逐行读 |
| 其他扩展名 | 41 | 859 | 全量清点，未逐行读 |
| deps 子模块（iSH/PRoot 等第三方上游） | 4,416 | 未计 | 单独清点；Minis 自有构建脚本 8 个共 2,443 行＋ffmpeg 补丁＋aot 说明已由统筹者逐行读 |

范围总计 3,512 个文件、2,136,054 行（`src/android`、`src/ios`、`scripts/`、`docs/`、根目录）。台账：`coverage-ledger.csv`；分区表：`partitions.json`。

## 架构理解（双原生、零共享代码、靠纪律对齐）

- **双端同构、实现独立**：iOS（Swift＋iSH 沙箱）与 Android（Kotlin＋PRoot 沙箱）是两套完整实现，没有共享代码层；一致性靠共享 fixture、字节级对等测试和源码互查（解析对方源码的测试）维持。这是全仓最贵也最脆的维护面：跨端接缝（备份线格式、ISO 日期 vs epoch、会话嵌套 vs 扁平形状、thinking 档位大小写）是历史事故的系统性来源，曾一次丢 2,350 个会话（P01）。
- **Agent 循环**：iOS 在 `AgentLoop.swift` 的单个约 2,000 行函数里（P10），安卓在 ChatViewModel 里（P14）。共同骨架：组提示词→流式调 provider→解析 tool_use→并发执行（安卓上限 10）→回灌→再调；上下文管线是「offload 大结果落盘→trim→compact（≤3 次）→exhausted」；token 计数靠按字符类校准的估算器，只有一次未校准的超窗发送当安全阀。
- **Provider 层**：统一消息模型翻译成 Anthropic/OpenAI（Chat Completions 与 Responses 双路）/Gemini/Antigravity 四种协议；历史修复（tool-call id 去重、配对消毒、reasoning 只回放给签发上游）是正确性核心。厂商识别靠 baseURL 子串匹配（P13），是已知脆点。
- **沙箱与原生桥**：guest（Alpine）里的桩命令回挂宿主原生能力（约 20 个能力域：日历、健康、NFC、蓝牙、HomeKit、浏览器、无障碍、Shizuku 等）。安卓在 handler 入口拦截权限（设计正确），iOS 靠命令首词匹配（有 `sh -c` 旁路）。PRoot 是用户态 chroot、与 App 同 uid，**不构成安全边界**（P28）。
- **数据与同步**：会话/消息存本地 DB（安卓 Room v14；iOS SQLite＋CloudKit）。iOS 有 iCloud 同步但实现有静默失败面（记录类型白名单外的脏行永不上传、无字段级时间戳导致解钉/移出文件夹不传播，P10/P17）；安卓**没有任何同步**，只有手动 `.minisbak` 全量备份包（格式已冻结为跨端线格式，加密版 PBKDF2-600k＋AES-GCM 分段，P26）。
- **记忆**：Markdown（GLOBAL.md＋每日日志），注入时整段塞提示词；有时间戳，无来源字段、无置信度、无取代链（P05/P14 与 DSH 线同判）。
- **配置**：双端都是上帝对象管配置（iOS ProviderConfigStore、安卓 ProviderRepository 3,989 行），都有「加载降级时禁止保存」的硬闸——历史上一次误判加载曾把配置清空并同步上云（P04/P14）。

## 跨模块关键流

1. **发消息全链**：UI→ViewModel 组上下文（记忆注入＋附件摄入＋图片预算按 base64 线字节计）→provider 流式→tool_use 并发执行（shell 进沙箱、原生能力走 offload 闸）→tool_result 回灌（必须连续排布在图片载体前）→落库（sort_order 定序，删除/墓碑按序敏感，P15）。
2. **上下文溢出自愈**：provider 真报 400 超长→抬高校准系数→把最大历史块 offload 落盘→重试一次；但 iOS 有一处已证实的活 bug：循环序言会重置溢出阀，Retry 会把刚被拒的请求原样重发（P07，测试标 EXPECTED TO FAIL）。
3. **凭据流**：API Key/OAuth token 存 Keychain（iOS）/加密偏好（安卓，但主密钥失败时静默降级明文，P28）；备份默认带凭据且加密默认关（P06/P25）；调试 RPC 能把凭据读出来（P02/P27）。**凭据在这套代码里四处漏风，这是个人版必须先堵的。**
4. **子代理**：安卓注册表易失、每父最多 3 并发子、子刻意不走会话并发管理器（最坏 24 个循环，P21）；恢复的子代理任务有永不 finish 的泄漏 bug（P05 测试钉死、未修）。

## 桩、死代码与占位（定稿清单，均带分区出处）

- 安卓 Antigravity provider：decode-only、`isUsable=false`（P27/DSH 线）；`HealthManager.kt` 纯桩（步数/心率回「集成待定」，P16）
- Gemini OAuth 双端占位 client secret，出厂不可用（P22/P27）；iOS Antigravity OAuth 占位密钥（P24）；Claude OAuth 的 state 校验在安卓形同虚设（P26）
- iOS `MarkdownRenderView.swift` 779 行无调用点（P03）；`/thinking` 斜杠命令列了但执行 switch 无此 case（P15）；`scheduleSilentAudioActivationRetry` 无调用点（P08）
- 同步残件：MigrationEngine 一期 fetch 是返回 0 的空操作、`ForceSyncHelper` 恒返回 0、`TombstoneManager.isTombstoned()` 恒 false、LANTransport 是 preconditionFailure 骨架且 SyncCore 拒绝注册（P17）
- 功能杀开关：WebApp「添加到主屏幕」双端被 `if (false && …)` 杀死（P11/P12）；安卓挂载文件夹 bind-mount 自述 MVP 脚手架未实现（P27）；无障碍 `dialog watch`/`extract table` 回 NOT_IMPLEMENTED（P16）；`HOOK(notification, P2)` 通知来源后缀未实现（P08）
- 整块 DEBUG-only：ICloudBackupManager、ChatSearchDebugRPC（P17）；DebugAuth 明文存配对令牌＋600 秒自动批准窗（P26）
- 零星死函数：`parseStringParam`（P10）、`formatContextWindow`（P15）、`AlarmOffloadHandler.wrap()`、`SoulSettingsView.applyEmojiDraft()`（P16）、安卓 `SyntaxHighlighter.kt` 疑似无调用（P21）、`AssistantMessageView` 自述 legacy 无引用（P12）
- 伪成功类：HomeKit offload 写超时仍回 success（P25）；Shizuku `networkRestrict` 无选择器时静默回成功（P16）
- 文档与代码互相矛盾处多发：备份头注释还写「未加密 stage 2/3a」而正文已是 AES-GCM（P18）；根 README 的构建口径与 BUILDING.md 亦有出入——读这套代码只能信正文，不能信注释头。

## 我的改造方案（缪斯判断，安卓先行）

用户家底：Antigravity Pro（云主机 agy，27 个模型含 Opus 5.5、Gemini 4 全家）、DeepSeek 官方 API、CommandCode 中转；无 Anthropic API 套餐。路线分五期，每期交一个能装机用的版本：

**第 0 期 基线与止血（先于一切功能）**
- 改 applicationId、关上游更新检查（更新摘要缺失时它居然跳过校验而非失败，P27）、打私有 tag。
- 关进笼子的调试通道：DebugServer 改回环＋令牌、release 构建物理移除 `provider.export` 类方法。
- 备份默认开加密、默认不带凭据；API Key 不再走 URL 查询串（Gemini 三处，P25）。
- 验收：原版构建出 APK 装机能跑；grep 证明 release 包无凭据导出方法。

**第 1 期 网关接算力（同意服务端网关，但按 §5 的条件）**
- 云主机建 OpenAI 兼容网关：`agy/*`（子进程包 agy）、`deepseek/*`（官方 API）、`cc/*`（CommandCode）；只听 Tailscale；手机只持一个网关令牌，原始凭据不出服务器。
- 前置门（半天实验，不过不盖楼）：agy 结构化 tool_calls 合法率实测——20 次带工具 schema 的请求，合法率 ≥90% 才让 agy 干工具回合；否则 agy 只做无工具回合（规划/写作/审阅），工具回合默认 DeepSeek。
- App 侧零代码先行：自定义 baseURL 指向网关跑通；再补模型目录拉取与网关健康徽章。
- 验收：手机上用 agy 的 Opus 档完成一轮带 shell 工具的对话；断网有明确报错不崩溃；网关日志能对账每次请求。

**第 2 期 账号与同步（治「手机一个户、Mac 一个户」）**
- 网关同进程长出账号面：设备配对码换令牌、服务端自增序号＋墓碑的增量同步（会话/消息只追加、设置后写胜、记忆按条目合并）。
- 会话租约防双写（借 iOS 同步的教训：不做字段级合并，只追加＋租约最省事且不丢）。
- Mac 先用 CLI 当第二设备验收，不急着做 Mac 客户端。
- 验收：两设备消息数一致且 id 无重复；离线改、重连后收敛；重装配对后会话数恢复。

**第 3 期 记忆溯源**
- 记忆入 SQLite：来源字段（session/message/tool id）由系统在执行点盖章，不靠模型自报；分类用数据/推论/观点三层＋置信度；Markdown 降为只读导出。
- 注入改按「置信度×时效×相关度」取 Top-N 并带来源标签；TTL 到期降权标「可能过时」。
- 验收：任一条记忆能点回原消息；无来源的新条目数 = 0（SQL 查）。

**第 4 期 治理收紧**
- 一行先行：未注册工具默认 BYPASS→ASK_ONCE（`OffloadPermissionManager.kt:402`）。
- 补闸门覆盖：通知、会话、定时、BrowserUse、ModelUse 五个无闸 offload 全部过闸；shell/文件/浏览器在 `executeTool` 统一前置策略引擎（风险分级：只读/本地写/外部副作用/花钱/不可逆）。
- 审批绑参数指纹防批准后改参；审计独立哈希链账本；日预算硬停。
- 验收：审批卡实测拦截 `rm -rf` 类命令；审计链校验无断点；预算触发即停。

**第 5 期 主动跟踪与跨设备接力**
- watch 迁到服务端跑（不绑手机，借现有 scheduled 的语义、换执行位置）；完成与否到目的地读真实状态判定，不采信自报。
- 会话接力：快照上服务器、另一端恢复、事件按服务端序号回流。

**明确不做**：不修上游的谷歌/Claude OAuth（绕开）；不移植 iOS 的 Antigravity 协议到 Kotlin（等网关路线跑顺再议）；不做 CRDT、不做向量库（FTS5 先行）、不做多用户与端到端加密（单用户＋Tailscale＋静态加密，信任边界写成文档）；不重构 ChatViewModel/ChatScreen 两个上帝对象（旁路收编）；不默认开无障碍/Shizuku。

## §5 网关之争：我的立场与证据

**同意建网关**，理由全部来自源码：设备端谷歌登录三条路（Antigravity、Gemini OAuth、Claude OAuth）不是「脆弱」而是「出厂损坏或半成品」（占位密钥×2、state 校验虚设、回调服务器绑 INADDR_ANY 且错误页未转义，P20/P22/P24/P26/P27）；Antigravity 协议是未公开接口＋身份伪造＋thoughtSignature 回放，复刻成本与封号风险都高；agy 在服务器上已经是活的、付费的、能用的同一资源。把「登录」从设备上拿走、在服务器上集中，是唯一与证据相容的接法。

**不同意「网关即一切」**，三条边界：
1. 网关只管算力与账号，不管设备能力：沙箱、offload、健康、日历这些能力天然在设备上，网关碰不到也不该碰；把 agent 循环整体搬上服务器会杀死这个产品最值钱的部分（P10/P16/P28 的原生能力面）。
2. 网关不得抹平上游差异：reasoning 产物、thought signature、工具配对都有上游身份属性（P13/P22），网关必须按会话钉住上游并透传这些产物；做成「随便哪个模型都能顶上」的无状态转发，损坏是静默的、测试还测不出来（全仓最大的教训就是静默语义降级，P09）。
3. 网关是单点，必须留直连兜底：DeepSeek API Key 直连作为网关故障时的 fallback 组（App 现有 fallback 机制可用），但 fallback 不许静默改绑会话主模型——iOS 现行 fallback 会永久改写会话绑定且失败标记永不过期（GH#229，P02/P09），这个行为在个人版里要改成「临时顶替、恢复后回主」。

## 风险前五（按杀伤排序）

1. agy 工具调用不稳定/条款不允许自动化转供——第 1 期前置实验门就是为它设的，失败即降级为无工具回合，不阻塞全局。
2. PRoot 非安全边界＋凭据四处漏风（备份、调试 RPC、URL 查询串、guest 代理环境变量含密码 URL，P28）——第 0 期止血清单即针对此。
3. 上帝对象改造回归——旁路优先纪律＋每次改动跑单测与冒烟；注意镜像模型测试的假绿风险，关键行为另加真机冒烟。
4. 安卓后台不可靠（Doze/厂商杀后台）让端上定时任务漏跑——主动任务一律服务端化（第 5 期），端上只留兜底。
5. 上游跟进成本：公开源码滞后发版约一个月且不收 PR；接缝清单（SEAMS）纪律＋按季度合并，避免改动扩散进上帝对象。

## 未读清单（如实交代）

- 数据文件 601 个（1,179,277 行）：本地化语料（.utf8/.xcstrings）、tiktoken 词表、json/xml 资源——已全量清点分类，未逐行读；它们是语料与资源数据、不含逻辑，但其中 xml/json 配置若后续改造涉及具体某项，需按文件补读。
- 二进制 97 个、其他扩展名 41 个：清点未读。
- deps 子模块 4,416 个文件：第三方上游（iSH 内核、PRoot、lame、rclone-mobile 等），单独清点；只读了 Minis 自有的 8 个构建脚本（2,443 行）、ffmpeg 补丁与 aot 说明。上游本体未逐行读，不计入自有代码覆盖率。
- 本报告对分区总结做了汇总综合；个别分区（如 P12 的 cppjieba 重复头文件）以 MD5 字节级一致验证替代重复逐行读，已在该分区总结中注明。
