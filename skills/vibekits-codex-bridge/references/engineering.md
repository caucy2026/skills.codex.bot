# 工程检查与案例记录

## 目录

- 版本与授权边界
- 常见缺陷和验证方法
- 分层扫描、编译与部署交接
- 2026-11-14 案例证据与限制

## 版本与授权边界

以所用 Codex 官方 schema 和真实 `initialize` 响应为依据，保存版本、平台、home、有效配置。安装一个新 CLI 不证明已经运行的桌面端使用同样版本/home，也不证明有可用的受保护 owner listener。不要通过放宽版本匹配来宣布兼容。

Unix：区分官方广告 symlink、确定性别名与受保护实际 endpoint，核对 owner、权限和初始化身份，不仅凭进程名/路径。Windows：核对受保护目录/ACL、kernel peer PID/SID 和权限等级，失败关闭；不得凭用户传入 PID/URL 或 localhost 身份认定可信。

不要将文件系统 readonly sandbox 等同“无外部副作用”：MCP、apps、hooks 仍可能产生作用。真实控制权限和审批语义需独立展示、授权。官方 command approval 需要完整 schema 参数；未知审批类别或缺少权威文件 diff 时不接受。grantRoot 可能是会话级授权，不能描述成单次。

## 常见缺陷和验证方法

### Owner、生命周期和并发

- `thread/resume` 可能与线程卸载竞态，甚至用当前配置恢复冷线程。恢复需要明确同意；核对返回 cwd/有效策略后才允许控制。写入后未知结果不得自动重试。
- `turn/start` 可能在竞态中变成向活跃 turn 追加。界面、授权、操作日志必须统一 send-or-steer 语义；精确文本、thread、host、scope、epoch、确认信息进入去重指纹。steer/interrupt 绑定 expectedTurnId。
- 池容量查询不能创建子进程；容量定义共享，失败/null/closed 项淘汰。撤销正在连接的来源后，迟到成功需关闭，不能重新入池。
- 空闲 async stream 的取消不能只依赖被阻塞 await 后的 finally。先发独立取消信号/关闭 wire/唤醒本地等待，再 await 清理。覆盖“没有下一条事件”、同线程重入、账户/项目切换。
- 共享桌面审批超时仅移除手机提示，不能代替用户自动拒绝。处理外部 `serverRequest/resolved` 时同时释放 pending/deferred 容量；用超过容量上限的连续解决序列测试无泄漏。
- 事件需 thread/turn/item/epoch/offset 锚定、容量和大小上限。旧历史快照不得覆盖更新的 live 文本。关闭连接不等于停止 Codex 或取消其运行 turn。

### 展示限制与缓存

统一历史和 live 的可见文本 predicate；未知展示 metadata 不泄露 raw body。已确认限制使用共享 deny-only registry，而非某个页面临时布尔值。稳定隔离键至少涵盖服务器/账户/设备/thread，必要时包括本地 profile/source。

并行 metadata/history/text 可能使已清除内容复活：历史 await 前后加限制栅栏，后端也查 registry，metadata 通道保留限制通知。查询限制位之前先以 metadata-only 证明项目成员关系，避免泄露其他项目 thread 存在性。

joining 阶段也必须先处理限制再早返回；resume 失败不能清掉限制，observer 不得因此获得正文/审批/发送。区分普通传输 resync 和已确认展示限制，不能将每个断线都永久禁显。限制时保留经授权 Stop，但不恢复发送、attach 或批准功能。

### Windows 传输

候选 AF_UNIX/Websocket.dll + Flutter MethodChannel 的平台代码必须用 Windows SDK 编译。portable C++ 安全策略测试只覆盖策略逻辑。opaque handle 用安全随机值，连接/队列/帧大小有界，不接受远端传入本机路径；平台线程回传结果，WebSocket 单 worker，队列满也能取消。

公共头不要在 Flutter/windows.h 之后引入 Winsock2。传输私有头在 cpp 中按 Windows 头顺序组织。源码接线不等于 ABI 正确；保持 native 和 Dart gate 一致且默认关闭，等真实目标平台编译后再审阅开启。

### 手机功能映射

合并列表使用独立 source/session/cache 命名空间，分页防 cursor 循环并限预算。真实品牌图保持来源、比例和色彩，不伪造图标。冷恢复、改名、归档、删除、fork、附件和审批分别维护能力矩阵；未实现的操作显式禁用。归档可能关闭派生线程/运行树，不能当作本地隐藏或 Stop 替代。

## 分层扫描、编译与部署交接

按仓库现有配置选工具，不自动上传源码到外部扫描服务：

1. 文件/patch 完整性：SHA256、相对引用闭包、patch apply-check；解析 JS/Python 语法。给闭包检查器负向样本，例如缺失引用、路径逃逸、非法语法。
2. 源码 guards：可防特定回归，但字符串/正则存在性不验证行为或 Dart 类型。曾出现 guards 通过而容量代码仍有运行逻辑错误，应补行为夹具，不只增加匹配词。
3. 行为夹具：Node 依赖注入模型测试只证明该模型；Dart synthetic fixtures 必须由 Dart 实际执行后才算通过。记录是否连接真实服务。
4. 类型与编译：按仓库要求执行 Dart/Flutter analyzer 和平台 build，保存输出。可用编译器对 portable C++ 加严格警告及 `-fanalyzer`；该步骤不能覆盖 Win32、Flutter runner 或 macOS 签名。
5. 部署（仅当用户要求）：合格 checkout/执行者资格 → 目标平台编译 → 打包与既有签名/升级门禁 → 授权发布 → 按约定验收。若用户只要编译，到编译交付为止。不能擅自换签名身份、重置应用数据或绕过发布规则。

用于本案例已有测试集的命令示例，运行前核对当前文件和仓库规则：

```sh
python3 -B -m unittest discover -s tool/codex_mobile -p 'test_*_source.py'
node --test tool/codex_mobile/*.test.mjs
```

这些命令不是产品编译命令，也不保证覆盖检查器自身或全部 C++/Dart 夹具。交接时列出实际逐项命令和结果，不根据文件数量估算测试数。

## 2026-11-14 案例证据与限制

本节是历史复盘，不是当前仓库状态或新版兼容性保证。

- VibeKits 基线 main `b2f97ae961fb96adeb69ca337a31307a07275dea`，pubspec `1.9.0-dev.880+2880`。输入为下载源码 ZIP，未冒充合格 Git checkout。
- 阅读 OpenManus main `3309bf4e416fb1c74b008f3e86494439a31bad53` 的 `app/tool/mcp.py`、`app/agent/mcp.py`。参考连接生命周期和 schema discovery；它不实现 Codex 原 owner 手机镜像，没有复制其代码。
- 已整理 v24 源码候选：Mac/Windows 发现、项目来源授权、中继与手机合并列表、历史、same-owner 控制、显式 send-or-steer、审批子集、取消、冷恢复、改名及跨通道展示栅栏。Codex 协议候选固定 0.169.0，不宣称任意桌面版本兼容。
- v24 源码包共 128 个累计文件；包 SHA256 `982dbf93b1307501515da8e5fda7196500bc9c14aefbc4620d7786e34b4bb673`。此哈希用于对应当时冻结交付，不代表 GitHub 上已有该源码提交。
- 已执行记录：Python 源码 guards 101 通过；Node 31 通过（17 synthetic +14 source guards）；70 个 Dart 文件/316 条相对引用闭包及 JS/Python 语法检查；portable C++ 25 项 synthetic 检查由 g++ 严格警告和 `-fanalyzer` 编译执行。完整/增量 patch apply-check、文件清单及摘要复核完成。
- **未执行**：Dart 类型分析/夹具、Flutter 产品编译、Win32 API/runner 编译、macOS 编译、签名、安装、部署、真实 Codex 会话或手机端到端验证。不能据此说“已编译通过”或“全部功能原理已证明”。
- **阻塞**：云环境缺 Dart/Flutter 和目标平台工具链；预构建资格报告 `QUALIFICATION_LATEST_MAIN_UNAVAILABLE`；ZIP 不具备合格 origin/main 与执行者注册。Windows candidate 的 native/Dart gate 保持默认 OFF。
- **仍有功能缺口**：归档/取消归档/删除/fork、媒体和文件发送、reasoning/tool 展示、更广审批类型、历史消息编辑；发现仍需本地项目授权。v23 已增加冷恢复及改名源码，不能再引用早期 v22 的“这两项尚未实现”作为最新事实。

继续工作时读取最新源码内 `tool/codex_mobile/REQUIREMENT_GAP_INVENTORY.md`、`THREAD_PARITY_STATUS.md`、`WINDOWS_OWNER_VERIFIER_STATUS.md`、`WINDOWS_DART_TRANSPORT_STATUS.md` 及新增版本附录。长文档前半段可能保留旧阶段说明，应按明确 supersedes 更新而非挑选最乐观段落。下一次真实构建后，用新证据更新本节，保留未执行项，不把历史数值机械累加。

### v25 增量记录

后续源码检查点 v25 修复了 ownerFollow 六字段 decoder 和 Windows 取消 deadline。工作记录为 Python 源码 guards 105 通过、Node 31 通过、portable GCC analyzer/25 项检查通过。这是 v24 之后的增量，不应继续把 v24 称为最新；以上 v24 哈希仅指 v24。Dart/Flutter/Win32 仍未编译，Windows gate 仍 OFF，任务和产品验证均未宣称全部完成。

复用教训：逐字段核对 wire decoder 与 producer，不能因导入闭包成立就认为协议解码正确；取消要有可到达的 deadline，覆盖无新事件和等待中的原生调用。更新证据表时同时保留基线、增量和未执行项。

### v26 增量记录

本次更新时最新已复核源码检查点为 v26，源码包 SHA256 `2a362bdf2247954436bffa485d3a0188a2a8a8aee6beacc561a22abc0573d0cd`。新增有界、需用户明确触发的逐条消息复制；UI 移除已禁用的 sandbox 选项，并纠正策略说明文案，避免让用户误认为未生效的限制已提供保护。

验证记录：Python 源码检查 108 通过、Node 31 项检查通过，patch apply-check 通过；独立源码复核未发现本次增量的新阻塞。这不是产品可运行或安全性的证明。Dart/Flutter/Win32 仍未编译，Windows gate 仍 OFF；签名、部署及真实端到端验证仍未执行。v24/v25 保留为历史记录，不能把旧哈希或旧检查数量当作 v26。

复用教训：复制也是内容出口，应限定到当前可见且获准展示的单条消息，限制大小并要求明确操作，不自动复制隐藏正文、完整会话或内部字段。权限 UI 只能描述真实生效策略；禁用控件不能让用户误认为存在可用保护。每次文案或复制路径变更也要纳入权限和展示限制的复核。
