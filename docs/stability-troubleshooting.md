# AGY 卡顿、掉线与卡死排障手册
> 核查日期：2026-10-17（北京时间）。范围：Antigravity CLI、桌面端/IDE，以及能解释同类症状的 Gemini CLI 和第三方集成。按错误关键词检索本页。

这是一份有界检索汇编，不代表 GitHub 全量穷尽。已核对上游问题正文与评论、官方版本记录及少量第三方项目；相似症状不等于相同根因，CLI 修复不能直接外推桌面端。

## 先做这五步

1. 保存编辑器未保存内容，记下产品、版本、系统、模型、错误全文和大致耗时；公开报告只保留脱敏片段。
2. 判断“界面不动”还是“后台也停止”：观察文件修改、任务输出和最后一条日志；后台仍活动时不要重复派同一任务。
3. 真正断流时停止当前轮，先核对已完成改动和外部动作，再把剩余任务拆成一个小步骤，定向重试一次。只读检查不要求重跑构建或发布。
4. 同一错误再现就暂停该路径，保存交接摘要，用备用工具接剩余工作。只有明确认证失效才重登，只有明确链路证据才排网络。
5. 更新前查对应产品发布记录、保存当前版本与工作；更新后先短只读请求，再测原触发场景。一次成功不代表长期修复。

## 证据标签

**官录**＝官方 CHANGELOG 写明该版本的修复；**官评**＝维护者评论；**社测**＝报告者提供复现或恢复结果；**建议**＝本手册的有限缓解方案，尚非该故障的实测修复。问题 Closed 本身不等于修复。

## A. 断流、重试、额度与认证

| 编号／检索词 | 适用范围和症状 | 现在如何处理 | 证据与边界 |
| --- | --- | --- | --- |
| A01 `tls: bad record MAC` | IDE 2.15.1/2.16.0，长思考约六分钟后流断开并重试 | 建议：停当前轮，核对改动，缩小单轮任务或降低思考强度后仅重试一次 | [#1096](https://github.com/google-antigravity/antigravity-cli/issues/1096) 社测；六分钟不是官方上限；关闭因转论坛，未确认根因或修复 |
| A02 TLS 中途失败后恢复 | Gemini CLI，相关 Cloud Code SSE 接口 | 社区有人重新请求或输入 continue 后恢复；AGY 可将其作为有限尝试 | [#17318](https://github.com/google-gemini/gemini-cli/issues/17318)、[#16075](https://github.com/google-gemini/gemini-cli/issues/16075)；关联产品，不能当作 AGY 修复证明 |
| A03 `500 / EOF`、头像请求超时 | CLI 1.2.2，无头运行已输出部分内容后异常结束 | 保存部分输出并核对副作用，重试剩余工作；不要整轮重复写入 | [#1014](https://github.com/google-antigravity/antigravity-cli/issues/1014) 社测，未找到该案例已解决确认 |
| A04 `502 / 503 / 504`、短期限流 | CLI 模型请求临时失败 | 有限等待服务恢复；防止外层脚本与内层重试叠加 | [官方记录 1.2.1][cl]：加入退避并保留已完成工具输出；不保证所有 500/TLS 均恢复 |
| A05 `Individual quota reached`、一直 Working | CLI 1.2.2，额度耗尽仍重试直到超时 | 查错误中恢复时间，等额度或换可用通道；不要把额度耗尽当网络坏 | [#1018](https://github.com/google-antigravity/antigravity-cli/issues/1018)；社测 1.2.14 已快速退出，但 retryable 字段仍可能为 true |
| A06 日额度、账单上限、余额不足 | CLI 不同认证方式，重试不能恢复 | 等恢复或由本人处理账单；停止自动重试 | [官录 1.2.12/1.2.13/1.2.15][cl]分别改进每日上限、长 Retry-After 和 AI credits 不足的快速报错 |
| A07 重试间隔像卡住 | CLI 反复请求失败，单次等待过长 | 看日志是否在退避，避免同时发多轮 | [官录 1.3.4][cl]：重试之间等待上限降为 30 秒；不是任务总时限 |
| A08 连接无数据但一直等 | CLI 模型流停止产出 | 建议升级相应 CLI 后复测，原工作仍须核对 | [官录 1.3.6][cl]：五分钟无数据会取消并重试；与 A01 的六分钟现象不能直接画等号 |
| A09 `Eligibility check failed` | CLI 启动时服务短暂异常后整会话拒绝消息 | 保存工作后重开会话；核对相应更新 | [官录 1.4.0][cl]：临时失败可重试，不再锁住会话；不涵盖真实账号资格封禁 |
| A10 登录过期／重复登录 | CLI 已明确 Authentication expired，或 Linux/WSL 无桌面密钥环 | 真过期由本人重登；WSL/无 D-Bus 环境先核对 keyring 超时日志 | [官录 1.1.3/1.3.3][cl]；网络超时不等于凭证失效 |

## B. 看着卡住、长会话和资源问题

| 编号／检索词 | 适用范围和症状 | 现在如何处理 | 证据与边界 |
| --- | --- | --- | --- |
| B01 Esc 后 Running 不更新 | CLI 1.2.7 Linux，后台仍能执行数小时 | 先检查后台和已有改动；确认停止后重开客户端，别连续补发指令 | [#1081](https://github.com/google-antigravity/antigravity-cli/issues/1081) 社测，维护者未复现；Closed 不是已修复 |
| B02 重启后旧任务永远转圈 | IDE/CLI 2.0 相关数据库残留运行状态或 malformed | 建议先备份、换新会话接手；数据库修复只交熟悉格式的人离线处理 | [#434](https://github.com/google-antigravity/antigravity-cli/issues/434) 社测；报告者的“非破坏”脚本仍会改数据库，不能照贴即跑 |
| B03 `status: CLEARED` 无限工具循环 | IDE Server 2.5.5，超长会话反复 git status、排队消息丢失 | 立即停止追加消息，保存文本草稿；从文件实况生成交接，开新会话 | [#968](https://github.com/google-antigravity/antigravity-cli/issues/968) 社测，转论坛；步骤数不是所有版本的固定阈值 |
| B04 `/resume` 内存暴涨 | CLI 1.1.9，历史索引缺失触发大量数据加载 | 优先更新并小样本复测；不要自行清空 gen_metadata 或 VACUUM 生产库 | [#727](https://github.com/google-antigravity/antigravity-cli/issues/727)；2026-10-14 官评称新版本已修，未指定最低版本；[1.2.2 官录][cl]另记历史缓存内存泄漏修复 |
| B05 长会话 CPU 高、输入和滚动慢 | CLI 大会话或全屏视图，非必然断网 | 留下交接后换短会话；只对照自己版本更新 | [官录 1.1.2/1.1.3、1.2.12/1.2.13/1.2.16][cl]涉及历史渲染、滚轮、重绘和缩放效率 |
| B06 连续执行命令后内存增长 | CLI 长会话，每条完成命令残留状态 | 建议保存工作、重开短会话，再更新复测 | [官录 1.1.28][cl]：完成命令的状态内存泄漏修复 |
| B07 键盘和 Ctrl+C 都失效 | macOS 集成终端前台死锁，后台可能仍活着 | 先核对后台任务；再重开受影响终端，不默认全机杀进程 | [#164](https://github.com/google-antigravity/antigravity-cli/issues/164) 社测；官评称新版本未再观察到，无最低版本证明 |
| B08 diff／artifact 一打开就卡 | CLI 巨大差异或长历史渲染 | 缩小查看范围或用外部编辑器看已保存文件 | [官录 1.0.8、1.3.5][cl]分别优化 artifact 和大 diff；不是模型请求问题 |
| B09 输出含非法 UTF-8 后停止 | CLI 工具结果或文件差异破坏状态流 | 定位具体文件及编码，避免继续喂同一损坏内容；保留备份再修 | [官录 1.1.21][cl]修复此类断流；二进制误读和超大文件另见 1.2.2 |
| B10 读设备／损坏媒体后会话卡死 | CLI 管道、socket、设备文件或截断媒体 | 停止读该文件，换完好输入或新会话；不要重试相同损坏附件 | [官录 1.2.14][cl]：前置拒绝特殊文件和部分截断媒体；不保证修复已损坏会话 |
| B11 空 continue 无限注入／Stop Hook 不停 | CLI 协调器或自定义停止钩子 | 查最后重复事件，停任务；钩子变更需单独审查 | [官录 1.1.9/1.1.10][cl]修复停止钩子无限阻塞及强制继续死锁 |
| B12 压缩后仍卡／上下文丢失 | CLI 长会话压缩预算或检查点失败 | 保存任务目标和已完成清单，必要时新会话；先看当前版本是否支持 /compact | [官录 1.2.8/1.2.9、1.3.5][cl]含压缩改进与手动 /compact；不能外推 IDE 同版本具备 |

## C. Windows、终端、无头调用与后台任务

| 编号／检索词 | 适用范围和症状 | 现在如何处理 | 证据与边界 |
| --- | --- | --- | --- |
| C01 非 C 盘工作区流中静默退出 | Windows CLI 1.1.0，路径拼接错误 | 优先更新 CLI；不必把业务工程搬到系统盘 | [#574](https://github.com/google-antigravity/antigravity-cli/issues/574) 官评称新版本已修，未指定最低修复版本 |
| C02 无控制台／stdin 等待 | Windows CLI 1.0.13 等，-p 无任何输出 | 对比交互终端与调用器，检查是否等待 stdin；保留常规权限流程 | [#508](https://github.com/google-antigravity/antigravity-cli/issues/508) 官评：1.1.1 修复给定 prompt 时仍读 stdin；更早 stdout 捕获修复另见评论 |
| C03 输出完整但调用器一直等 EOF | CLI 管道被子进程继承，或 daemon 未结束 | 保存结果，识别具体进程和终态，再给调用器设有限超时 | [官录 1.1.24/1.2.9][cl]涉及句柄和 daemon；不代表所有不退出故障都解决 |
| C04 答案完整、无子进程但不退出 | Linux ARM64 CLI 1.1.26，内部已 shutdown 仍存活 | 建议校验最终结果和产物后回收该调用；不要因为父进程超时丢掉全部有效输出 | [#947](https://github.com/google-antigravity/antigravity-cli/issues/947) 社测未确认修复，与 C03 区分 |
| C05 exit 0 却空输出／部分失败 | CLI 无头状态流断开或 print-timeout | 同时查最终 result/status、stderr 和实际产物；退出码只是其中一项 | [官录 1.1.18/1.1.28/1.2.6][cl]显示不同阶段行为变更；不能仅凭 exit 0 判成功 |
| C06 默认五分钟超时／无限等待 | CLI -p，不同版本默认时限不同 | 自动化显式设置合理 --print-timeout，并校验结果；它不保证恢复服务端断流 | [官录 1.2.6][cl]：默认从五分钟改为无限；期限按任务制定 |
| C07 run_command 永远 Running，无 shell | Windows CLI 1.2.16，组织强制 sandbox/AppContainer | 收集策略和启动错误给管理员；不要靠关闭安全策略或跳权限解决 | [#1161](https://github.com/google-antigravity/antigravity-cli/issues/1161) 社测；1.2.17 沙箱改进不等于该具体案例确认修复 |
| C08 PowerShell Access is denied／BOM 配置 | Windows 商店版 PowerShell 7 或配置带 BOM | 对照失败路径核查，更新后复测；别把读取配置失败归到网络 | [官录 1.3.3、1.1.23/1.2.16/1.3.1][cl]分别涉及 shell 回退和不同配置文件 BOM |
| C09 命令等隐藏编辑器／pager | CLI Git、jj 等需要交互的命令 | 改用非交互参数，明确消息和输出；先查是否在等编辑器 | [官录 1.3.3/1.3.6][cl]处理 pager 和 editor；操作本身的权限不因此放宽 |
| C10 编译后 server restart | macOS CLI，tsc 批量生成文件触发会话掉线 | 核对本地构建结果；建议分开构建与分析，按官方配置隔离生成目录 | [#501](https://github.com/google-antigravity/antigravity-cli/issues/501) 报告者改判疑似 file watcher 同步；原“输出缓冲溢出”不是已证根因 |
| C11 后台命令退出后 agent 不继续 | CLI 后台命令被 OOM 或信号杀死 | 查真实退出码及产物，不依据旧 Running 标签判断仍工作 | [官录 1.2.16][cl]修复异常退出未通知 agent |
| C12 无头任务空闲后后台工作被取消 | CLI -p 后台任务/定时器生命周期 | 区分 daemon 与应等待的任务；超时前保存状态 | [官录 1.2.9][cl]调整等待行为；不要假定 CLI 结束后开发服务器永久存活 |

## D. MCP、权限和桌面启动

| 编号／检索词 | 适用范围和症状 | 现在如何处理 | 证据与边界 |
| --- | --- | --- | --- |
| D01 stream-json 只有 init、约三分钟无动静 | Windows CLI 1.1.17，MCP initialize 不返回 | 查日志具体卡住的 MCP；备份配置后仅隔离该服务，先做无工具短请求 | [#841](https://github.com/google-antigravity/antigravity-cli/issues/841) 作者更正：提示长度无关；disabled 在其环境仍连接，不能假定点禁用就已隔离 |
| D02 启动等慢 MCP／工具调用永不返回 | CLI MCP 握手、列表或调用卡住 | 检查各阶段超时，避免把超时改成无限；交互与无头行为不同 | [官录 1.0.4/1.1.3/1.1.9][cl]：并行初始化、有限超时、交互后台加载；无头仍可能等待工具就绪 |
| D03 MCP 401、授权成功仍不可用 | 旧 CLI/IDE HTTP OAuth 没带 token | 核对是否为该 MCP 认证，不反复重登整个账号；临时 bearer 方案仅在本机安全配置 | [#25](https://github.com/google-antigravity/antigravity-cli/issues/25) 官评给临时 header 方案；不复制 token 到命令历史、仓库或聊天 |
| D04 授权码粘贴截断 | Windows CLI 1.0.6 等，长 OAuth code 失败 | 更新到包含修复的 CLI，重新走本人授权流程 | [#341](https://github.com/google-antigravity/antigravity-cli/issues/341) 官评与标签明确 1.1.25 已修；不使用浏览器控制台提取凭证 |
| D05 无头流程等待权限／问题答案 | CLI 自动化没有可见交互入口 | 用明确最小权限规则并检查拒绝提示；缺必要人工输入则退出转人工 | [官录 1.1.3/1.3.6][cl]涉及权限等待与子代理提问；不要把 --dangerously-skip-permissions 当通用解法 |
| D06 MCP 暂时掉线就被登出 | CLI MCP OAuth 连接短暂失败 | 先确认凭据被拒绝还是网络暂时异常；保留正常登录态 | [官录 1.1.9/1.3.4][cl]改善重连与临时错误清凭据问题 |
| D07 插件 MCP 不启动／同名冲突 | CLI 多插件服务同名、配置变量或初始化竞争 | 定位具体插件和服务，更新后测试该工具；不重装全部插件 | [官录 1.2.0/1.2.2/1.3.1/1.4.0][cl]覆盖启动、命名空间、文件占用和 header 变量 |
| D08 桌面窗口消失、进程仍在 | Windows Desktop 2.19.1，重复点图标无窗口 | 建议先确认后台无重要写入，再从托盘正常退出并重开；未确认停止前勿重复派工 | [#1158](https://github.com/google-antigravity/antigravity-cli/issues/1158) 社测，维护者转论坛；GC 根因是报告者分析 |

## E. 可查阅但不能混当官方修复的项目

- [antigravity-panel](https://github.com/n2ns/antigravity-panel)：作者提供重载窗口、重启后台等工具；属于第三方扩展，未在本手册安装或验收。
- [antigravity-server](https://github.com/AFSlayer/antigravity-server)：作者描述 gRPC 断线重连和空闲重启，适用于其 Linux 服务封装，不是原生 Windows AGY 的通用补丁。
- [Windows crash patch](https://github.com/erdometo/antigravity-windows-crash-patch)：作者针对 Ctrl+C 引发后台退出提供补丁/封装；涉及进程或程序修改，本手册只登记线索，不分发或建议直接运行。

## 复现记录与后续维护

最小记录格式：产品/版本、系统、模型与思考强度、交互或 -p、是否新会话、触发动作、最后日志时间、错误关键词、后台是否仍活动、已完成改动、恢复动作及结果。公开前移除账号、凭证、个人路径和会话标识。
每次更新先读问题最新评论再改结论；保留被作者更正的旧假说及更正关系。新案例按上述编号分组；只有原案例确认或对应版本记录才能标记修复，不能凭 Closed 盖章。
本页建议的有限重试、任务拆分和结果核对是工作恢复策略；不代表已定位所有内部根因。不会自动切网络、重登、清库、修改沙箱或安装补丁。

## 来源快照与检索边界

官方版本记录使用 2026-10-16 提交的固定快照，避免 main 后续变更混入本次结论；版本号表示该快照中的记录，不表示本机已安装或某发行渠道必然已提供。
[cl]: https://github.com/google-antigravity/antigravity-cli/blob/e8a400403c82b7eb27a333784dc3c37985aa622e/CHANGELOG.md

检索覆盖官方 CLI 问题中的 hang/stuck/freeze/timeout/EOF/TLS/quota/MCP/Windows/context 等关键词、关联 Gemini CLI 问题及上述第三方项目；17 条 AGY 问题已核对正文和当时可读评论。没有扫描所有 GitHub 仓库、私有讨论或论坛；不收录账号池、绕限或凭证导出方案。未来新问题须继续增补。
