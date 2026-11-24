# LoopForge 最近 90 天 PR 与性能审查

**审查窗口**为北京时间 2026-08-22 00:00 至 2026-11-19 23:54。分析固定在开始审查时的快照。本地 fork 为 `d630ae0`，上游 `Tencent/LoopForge` 为 `6f606cf`。上游比两者的公共祖先多 14 个一阶提交，fork 有 1 个独有提交。文中标注了尚未进入当前 checkout 的上游改动。

## 优先处理

1. **让 worker 只读待恢复的运行。** #48 的恢复循环每两秒读一遍历史运行，先逐条读取节点和制品，再过滤状态。本地数据库探针在 1 万条终态历史下实测，单 worker 一轮执行 **20,002 次查询**。历史增长会抬高整个实例的数据库负载。
2. **把 Docker 清理移出引擎全局锁。** #48 的整 SOP 重试在持有 `stateMu` 时同步拆容器。Docker 卡住能够拖住同实例其他运行的写入和状态推进，单次拆除配置了 60 秒上限。
3. **减少聊天与工作区在没有新数据时的工作量。** #65 和 #810 涉及空闲轮询、生成记录重复读取、历史消息重复处理和 Git diff 请求重叠。虚拟时钟和挂起请求探针已复现这些行为。
4. **先缩短 CI 的共享关键路径。** 目前最重的是 `platform/core` 中的浏览器无关 Web 测试与 Go 测试。历史极值 #47 已关闭，今天可直接优化的是共享测试、依赖安装和启动失败等待。

## 覆盖范围与比较口径

| 数据 | 覆盖与口径 |
| --- | --- |
| fork 与上游 | fork 的 GitHub PR 列表为空，Actions 返回 0。使用上游 PR、Actions 和 Git 历史，并以本地 checkout 复核当前代码。 |
| PR 生命周期 | GitHub GraphQL 共返回 381 个可见 PR。窗口内新开、窗口内关闭或合并、或跨入窗口仍开放的有 **374 个**，截至截点 167 个合并、37 个关停未合并、170 个开放（其中 7 个 draft）。其中 #7、#8、#9 在窗口前创建、窗口内结束。 |
| Git 补充覆盖 | 上游公开的 664 个 `refs/pull/*/head` 中有 **284 个**窗口相关编号未出现在 GraphQL 列表，188 个可从上游 main 一阶提交标题对应到落库提交。逐号探查 #115、#666 也返回 404。合计 **658 个**编号做改动路径筛查。Git 补充项按 head 提交日期或 main 落库识别窗口，缺少可靠的开合时间与 PR 状态。 |
| 主线 | 截点前的窗口基线是 `09c7652`。本地一阶历史 347 个提交，上游 360 个。大变更按合并提交和父提交比较，未合并 PR 按 merge base 和 head 比较。筛查包括平台后端、Web/chatbot、Local、CI/交付脚本和文档，沿实际调用和差异深读高风险路径。 |
| Actions | 上游 `pull_request` 事件，按创建时间分片获取 **2,484 / 2,490** 个公开 API 计数对应的唯一 run。11/11–11/14 的三日分页原少 12 条，切分后追回 6 条，剩余 6 条的 job 无法核对。2,459 个 run 可关联，覆盖 377 个本次 PR 编号。关联优先使用 Actions 明示 PR、其次 head SHA、最后唯一的 head 仓库 + 分支 + PR 存续时间。 |
| 最慢 CI | 每个 PR 取最长的、有实际步骤执行的已完成 job，使用 `job.completed_at - job.started_at`。以 run 时间字段筛出可能超过 15 分钟的候选，并查询重跑的较早 attempt，共核实 188 份 job 列表。多个 job 可并行，局部重跑会带入原 attempt 的已成功 job，所以不把最早与最晚的时间差计为一次执行。 |
| 用户侧性能 | 以当前代码、引入该行为的 diff、可重复的小型探针或 PR 自带的实测为依据。未取得线上 APM、真实数据分布和部署拓扑。文中的 SQL 数量、合成基准与用户侧 p95 各自独立。 |

数据入口：[上游 PR](https://github.com/Tencent/LoopForge/pulls)、[platform Actions](https://github.com/Tencent/LoopForge/actions/workflows/platform.yml)、[上游代码快照](https://github.com/Tencent/LoopForge/tree/6f606cf2a7354f05d1dbf2085f4eb832b7e89ccb)、[fork 快照](https://github.com/wnz27/LoopForge-learn-k7/tree/d630ae056ec83c3bd3d510a2732730b8e5bcb6c1)。

## PR 从创建到合并的耗时

167 个可见已合并 PR 的中位数 **3 时 41 分**，P90 **3 天 5 时 46 分**，P95 **18 天 19 时 2 分**，按最近秩计算。时长包含评审、等候、改动和 CI。

| 排名 | PR | 开到合并 | 差异审查 |
| ---: | --- | ---: | --- |
| 1 | [#36 TKE 节点池错误回退](https://github.com/Tencent/LoopForge/pull/36) | 45 天 6 时 19 分 | 修正 fallback 抛出的异常并补测试。PR 未增加请求次数。两个串行列表请求和回退请求已在 #14 存在。 |
| 2 | [#8 setup-node](https://github.com/Tencent/LoopForge/pull/8) | 43 天 22 时 23 分 | 窗口前创建，改 Actions 版本。与 #7 在同一批合并。 |
| 3 | [#7 checkout](https://github.com/Tencent/LoopForge/pull/7) | 43 天 22 时 23 分 | 窗口前创建，改 Actions 版本。若按窗口内新开排名，#7、#8 从此表剔除。 |
| 4–6 | [#38 build-push](https://github.com/Tencent/LoopForge/pull/38)、[#39 Buildx](https://github.com/Tencent/LoopForge/pull/39)、[#40 Docker login](https://github.com/Tencent/LoopForge/pull/40) | 约 36 天 17 时 | 同批合并的 release workflow 依赖更新。影响面是镜像发布，未发现这些改动增加产品热路径工作量。 |
| 7 | [#32 Local 合约 E2E](https://github.com/Tencent/LoopForge/pull/32) | 22 天 16 时 21 分 | 新增合约测试，依赖现有发现流程。本机聚焦 6 例约 5.2 秒。 |

未合并关停中最长为 [#22](https://github.com/Tencent/LoopForge/pull/22) 53 天 2 时 35 分、[#30](https://github.com/Tencent/LoopForge/pull/30) 50 天 17 时 10 分。截点仍开放中最长为 [#62](https://github.com/Tencent/LoopForge/pull/62) 23 天 5 时 45 分、[#121](https://github.com/Tencent/LoopForge/pull/121) 10 天 11 时 44 分。批量 Dependency PR 的创建到合并耗时适合衡量积压。若要进一步区分评审与作者等待，需要另汇总 review-ready、首次有效 review 与修改后的复审时间。

## 最慢 CI 与当前优化

以下按可关联且有实际步骤的最长单 job 排名。时间未加上排队、审批或同一 PR 的多次重跑。部分旧 run 的更新时间晚于启动约 30 天，job 明细没有对应量级的执行时间。

| 排名 | PR | 最长 job | 耗时 | 耗时所在 |
| ---: | --- | --- | ---: | --- |
| 1 | [#47](https://github.com/Tencent/LoopForge/pull/47)，关闭 | [platform e2e](https://github.com/Tencent/LoopForge/actions/runs/38506444939/job/115664949291)，失败 | **49:58** | Browser tests **47:51**。PR 自述旧 E2E 仍等待已改名的导航、控件和 onboarding，候选界面本地有 25/40 例失败、4 例跳过。后续由 #48 的平台集成接替，现有 workflow 的 browser job 上限 30 分钟。 |
| 2 | [#1033](https://github.com/Tencent/LoopForge/pull/1033)，上游已合并 | [core](https://github.com/Tencent/LoopForge/actions/runs/44273652059/job/134696204612)，成功 | **19:43** | Web 测试 **9:02**，非浏览器 Go **4:41**，许可证清单 **2:51**。PR 只改 9 个 Go 文件，前者与其他 PR 的共享测试代价一致。 |
| 3 | [#1019](https://github.com/Tencent/LoopForge/pull/1019)，开放 | [core](https://github.com/Tencent/LoopForge/actions/runs/44343195465/job/134938451064)，失败 | **19:00** | Web 测试 **8:45**，非浏览器 Go **6:30**。压缩库依赖 PR 的 job 计时本身不足以判定为库回退。 |
| 4 | [#67](https://github.com/Tencent/LoopForge/pull/67)，已进入 fork | [first-run](https://github.com/Tencent/LoopForge/actions/runs/42043758668/job/127301089896)，job 成功 | **18:45** | Playwright 及系统依赖 **13:39**，安装 LoopForge **3:34**。同次 workflow 另有失败。 |
| 5 | [#1077](https://github.com/Tencent/LoopForge/pull/1077)，上游已合并 | [core](https://github.com/Tencent/LoopForge/actions/runs/44496444215/job/135464705656)，成功 | **18:21** | Web 测试 **8:17**，非浏览器 Go **5:53**。另有 warm SOP 多一次 Docker 探测的产品侧风险，见下。 |
| 6 | [#1015](https://github.com/Tencent/LoopForge/pull/1015)，上游已合并 | [core](https://github.com/Tencent/LoopForge/actions/runs/44224015093/job/134532613147)，成功 | **18:16** | Web 测试 **8:20**，非浏览器 Go **5:44**。工作区运行时有独立的 Git 进程开销，见下。 |

[#1018](https://github.com/Tencent/LoopForge/actions/runs/44343192013/job/134938441104) 的两个共享步骤耗时 8:53 和 6:21。仅改 README 的 [#1094](https://github.com/Tencent/LoopForge/actions/runs/44494021215/job/135456507495) 对应 7:53 和 5:44。#1033 的许可证步骤是这个长尾样本中的异常点，已查到同段 47 个成功步骤的中位数 11 秒。`collect-go-notices.py` 会对本地缺失的模块串行调用 `go mod download`，该次运行的招致因素还需要步骤日志或子阶段计时确认。

| 优先级 | 当前发现与优化 | 验收 |
| --- | --- | --- |
| P1 | `platform/core` 顺序执行 Go 与 Web 全量覆盖。`webtest/package.json` 的 Node 测试并发为 4，之前已经在 #115 隔离真实 Chromium 用例。先把 `npm ci`、测试、覆盖报告各自计时，再评估拆成并行 job。保留 Go 覆盖合并、Web 阈值和 `platform-required` 聚合。 | 比较同一 SHA 在同类 runner 的冷/暖 P50 与 P95。记录总 runner 分钟数，确认测量到关键路径收益和覆盖结果一致。 |
| P1 | 旧 CI 在两个 job 重复跑依赖安装，[#104](https://github.com/Tencent/LoopForge/actions/runs/42273048049/job/128082266067) 和 [#95](https://github.com/Tencent/LoopForge/actions/runs/42237750215/job/127959730960) 此步也各用 10:35 和 9:56。#186 已加入 DNS 和共享库 fallback。目前脚本仍先下载浏览器，部分 runner 仍执行 `install-deps`。按锁文件、操作系统和浏览器版本缓存，分别计时下载与 apt，对 apt 加总 deadline，保留共享库校验。 | 缓存冷/暖各至少 10 次，加一次模拟 Ubuntu 镜像不可达，确认出现依赖缺失时仍明确失败。 |
| P1 | `platform.yml:90–98` 启动后台 `scripts/dev.sh` 后没有检查进程退出。故障注入令子进程立即返回 42，原步骤仍触发 300 次健康探测和累计 600 秒配置等待。保留健康检查，把 PID 检测和日志放进轮询，并为 curl 设置单次连接和请求上限。冷启动当前还编译 6 批、共 11 个二进制产物。CI 可只构建实际执行的目标，release 保留完整矩阵。 | 当启动子进程立即退出时，几秒内输出根因。健康启动成功。独立比较 browser job 的冷启动计时及受影响用例。 |
| P2 | `platform.yml` 和 `validate.yml` 没有按 PR 取消已被新 push 取代的执行，`e2e.yml` 已有该机制。仅对 PR event 聚合同 PR 的 runs，推送主线的检查和发布纪律保持原有行为。 | 对同一 PR 连推两次，最新提交能正常过门禁，过时尝试被取消，并持续覆盖主线 push。 |
| P2 | #960 删除一张首页卡片后，`first-run` 因测试已发布镜像而漏掉候选构建的数量变化。#1002 更新镜像 pin 后，未发布版本走源码才暴露错误，#1013 修复断言。保留已发布镜像的从零安装检查，同时给候选构建加一个只覆盖首次设置的短检查。区分镜像不存在与仓库超时。 | 用有镜像和缺镜像两种路径验证，首次登录或首页回归应在引入的 PR 上失败。 |

代码：[platform workflow](.github/workflows/platform.yml#L1-L225)、[validate workflow](.github/workflows/validate.yml#L1-L12)、[Local E2E workflow](.github/workflows/e2e.yml#L6-L16)、[Playwright 安装](scripts/install-playwright-deps.sh#L1-L30)、[构建矩阵](scripts/build-binaries.sh#L56-L69)、[许可证收集](scripts/collect-go-notices.py#L13-L25)、[Web 测试命令](webtest/package.json)。

## 现有运行时瓶颈

### P0 · 恢复轮询随历史运行线性增负

#48 启动 worker 后立即跑一次恢复，此后每 2 秒触发。除了提升排队的运行，`ResumeInterruptedRuns` 遍历 `RunsRepo.Runs()`。后者一条查询读全部运行头，每个运行再查询节点与制品元数据并装载节点，最后按创建时间排序。恢复逻辑这时才排除工作区、历史 group 和无需恢复的状态。部分 running 候选还会重复读完整 `RunState`。这一实现同时存在于 fork 和上游快照。循环串行执行，慢于两秒时不会每两秒开一轮并发扫描。多个 worker 会各自扫描同样的历史数据。

**实测** 用仓库的 SQLite `OpenTest` 创建 0、100、1,000、10,000 条终态运行，逐组调用真实 `ResumeInterruptedRuns`，用 GORM query callback 计数。三次重跑均得到 **2、202、2,002、20,002** 次查询，包括 1 次排队桶探测和 1 次全表运行头查询。1 万条时本机单轮中位数约 379 ms。样本无排队或活跃运行、无实际节点和制品载荷，MySQL 的网络与载荷成本仍需测量。

**改法** 在 `RunsRepo` 加恢复专用候选查询，SQL 投影状态、类型、owner、lease 和控制信号。仅对需要判断完成、取消、对齐审批或接管的候选读节点与具体状态。带有效 lease 的 running 仍可能需要完成图判断，查询不能统一过滤成 lease 已过期。保留二次校验、历史兼容、工作区归属和 scheduled integrate 例外。

**验收** 固定 5 个活跃候选，构造 0、1,000、10,000 个历史终态，记录每轮 SQL 数、分配字节、MySQL P95 和双 worker lease 竞争结果。历史规模增长时，实际加载的节点和制品条数应随候选数变化。

证据：[PR #48](https://github.com/Tencent/LoopForge/pull/48)、[worker 周期](internal/engine/run_state.go#L130-L150)、[恢复逻辑](internal/engine/run_recovery.go#L107-L185)、[全量装载](internal/repo/runs.go#L95-L112)、[节点 JSON](internal/repo/runs.go#L576-L591)、[制品查询](internal/repo/runs.go#L691-L700)。

### P0 · Docker 清理可阻塞其他运行

重试复用引擎的 `stateMu` 多步读改写，锁覆盖整 SOP 重试所触发的同步容器清理。已清理工作区的部分重放路径还可能经过两处清理。Docker 拆除配置 60 秒超时，其他 run 的 `updateRunAtErr` 同样需获取该锁。#953 减少了新建脚本伴生容器的场景，仍尝试清理旧伴生容器。

**改法** 在锁内登记带 run ID、generation/revision 的清理意图，释放全局锁后按运行执行有界清理，重新进入时校验同一代并提交。合并重复清理，同时保留下一次执行前必须达到的清理顺序、workspace 锁和旧容器兼容。

**验收** 用可挂起的 fake Docker 阻塞单个 run 的 `rm -f`，并发执行另一个 run 的状态写入，应在前者解除阻塞前完成。覆盖旧世代清理与新 attempt 重叠、清理失败重入。两个 E2E 用例目前用 TODO 表示嵌套 Docker 拆除问题，修复后恢复明确执行和失败断言。

证据：[PR #48](https://github.com/Tencent/LoopForge/pull/48)、[PR #953](https://github.com/Tencent/LoopForge/pull/953)、[重试锁](internal/engine/run_ops.go#L434-L439)、[同步拆除](internal/engine/run_ops.go#L498-L555)、[共享写锁](internal/engine/run_state.go#L34-L57)、[Docker timeout](internal/runtime/sandbox_docker.go#L62-L95)、[重试 E2E](e2e/api/30-runs.test.mjs#L86-L100)。

### P1 · 聊天空闲请求、SQL 和历史渲染一起放大

- **空闲和跨副本。** #65 的生成与列表轮询在空闲时分别以 2 秒与 10 秒调度。虚拟时钟运行上游代码，预热后的 60 秒产生 30 次 generation GET、6 次会话列表 GET，未变化的 generation 触发 60 次状态 setter 调用。真实浏览器可能节流后台定时器或批处理 setter。#810 保住了慢心跳的 live generation，但没有本地 active generation 时先 Peek 再 Reap，每次读取并解码同一份完整 JSON 两次，远端 running 的新鲜度还可再查 DB 时钟。会话列表也会先装载所有其他用户的项目会话再做可见性过滤，按 issue 请求在某些共享运行上先装载完整 RunState 再过滤。
- **新会话标题。** #67 为等生成标题增加 2 秒一次、最长 30 秒的探测。探测读的是会话详情，包含消息历史。现有列表刷新已经返回标题，可复用列表投影或完成事件，缩短每次读取并避免两条定时路径同时维持。
- **正在输出。** 流事件更新 App 的状态，所有历史消息随 App 重渲染。历史 assistant 消息的工具事件在两个渲染位置重新还原，Markdown 也重新解析。用当前还原函数，100 条合成历史回复、每条 100 个事件时，两处历史还原合计的本机 Node P95 约 3.3 ms，未计 React、Markdown、DOM 和布局。

**改法** 有活动流时保持实时更新，空闲时根据可见性和 generation 版本退避，聚焦后立即补拉，终态不重复应用相同状态。服务端合并读取与时钟判断，保留跨实例 CAS 与重连语义。列表先按授权项目与 issue 投影，批读必要元数据。稳定历史消息的 props，按消息与活动版本复用归约，仅让活跃一条随流变化。

**验收** 对 100/500 条历史记录、长输出、两副本重连及切到后台的场景，收集请求数、每请求 SQL 和解析字节、React 提交次数与长任务。还原断线后终态、滞后心跳与 SSE 光标。

证据：[PR #65](https://github.com/Tencent/LoopForge/pull/65)、[PR #67](https://github.com/Tencent/LoopForge/pull/67)、[PR #810](https://github.com/Tencent/LoopForge/pull/810)、[聊天轮询](chatbot/src/useChat.js#L316-L472)、[历史渲染](chatbot/src/App.jsx#L1570-L1593)、[历史活动还原](chatbot/src/App.jsx#L1693-L1720)、[Peek 与 Reap](internal/httpapi/assistant_unfinished.go#L104-L124)、[会话列表](internal/httpapi/assistant_session.go#L41-L120)、[跨用户数据读取](internal/repo/assistant_shared.go#L93-L111)。

### P1 · 工作区关闭时仍拉差异，慢请求可跨轮询堆叠

#65 的 `useWorkspaceChanges` 对工作区存在的会话并发请求状态和完整 diff，首次触发不依赖面板打开。面板打开时每 6 秒刷新，turn 完成也能发起刷新。effect 清理只将旧结果标作过期，请求继续执行。后端按仓库运行 Git 状态和差异命令。用真实 hook 与挂起网络 stub 复现，关闭状态发起 2 次请求，打开后的两次 tick 合计 6 个未完成请求，均无取消信号。上游 #1046 扩大了工作区覆盖面，#1096 更改的是模型使用的仓库清单，没有改变面板请求。

**改法** 面板打开后才拉完整 diff，同一工作区保持最多一批在途。以 `AbortSignal` 贯穿切换和关闭，或在上一批完成后调度下一次刷新；若 UI 仍需关闭状态的徽标，可给摘要用更窄的查询并按变更版本复用。记录工作区时效时保留真实用户活动语义。

**验收** 模拟每次 Git 命令耗时超过 6 秒、多个仓库、切换会话和关闭面板。面板关闭时停止拉完整差异，任一时刻最多一批请求，重新打开仍能拿到最新变更。

证据：[PR #65](https://github.com/Tencent/LoopForge/pull/65)、[PR #1046](https://github.com/Tencent/LoopForge/pull/1046)、[工作区 hook](chatbot/src/useWorkspaceChanges.js#L27-L93)、[Git 状态和差异](internal/httpapi/run_workspace.go#L649-L715)。

### P1 · 模型前的文件准备增加空转，每轮重复付费

#902 将共享技能安装扩展至每一轮，含无项目的对话。未绑定的外部共享技能逐个查询 `SKILL.md`，原目录先移除再重新写入。空项目场景按全站 S 个符合条件的外部技能，会有一次目录查询加最多 S 次文件查询。绑定 CLI Runner 的 #494 路径另外准备并重置资产、按 32 个小文件或 10 MiB 分批串行同步、最后 start。对 F 个可按数量满批的小文件，约 `ceil(F/32)+3` 个往返，Runner 每批还遍历已安装的受管路径。#604 的自定义镜像工具链复制也有实测，一次 PR 作者的镜像场景复制 **477 MB / 2.5 秒**，其另报的 54 秒是整轮耗时。

**改法** 批读所需技能文件，按授权集合和内容版本计算清单。常驻工作区或 Runner 只传变化和删除项，校验整批后原子发布。保留撤销、切换运行身份和 legacy 资产行为。工具链复制单独以源版本和目标生命周期验证复用收益。

**验收** 用 0/50/250 技能、冷/热工作区与注入 Runner RTT 运行同一请求，测准备阶段、文件字节、RPC 数和首输出 p95。增删授权共享技能后重复验证。

证据：[PR #902](https://github.com/Tencent/LoopForge/pull/902)、[PR #494](https://github.com/Tencent/LoopForge/pull/494)、[PR #604](https://github.com/Tencent/LoopForge/pull/604)、[技能查找](internal/httpapi/assistant_subagents.go#L208-L247)、[重建技能树](internal/runtime/skill_tree.go#L36-L64)、[Runner 上传批次](internal/httpapi/assistant_process.go#L199-L220)、[同步校验](internal/runtime/conversation_process.go#L164-L179)。

### P2 · 平台首屏和低频高成本路径

| 路径 | 证据和优化 |
| --- | --- |
| #48 / #65 平台首屏 | 本地壳页声明 53 个同步 JS 和 34 个 CSS，约 3.08 MB 原始数据、约 844 KB 逐文件 gzip 估算。#65 之后的壳素材比 #48 增约 46%，上游快照为 54 JS、34 CSS，原始约 3.11 MB。壳鉴权后等待全量 bootstrap 并渲染包含隐藏视图的全套页面，随后聊天 iframe 还会再拉五个 API，然后拉会话列表。所有 `/chatbot/` 文件统一 `no-store`，构建后带指纹的 chunk 也无法跨访问复用。先拆出当前视图的数据和渲染，再让经验证的指纹资源长缓存，HTML 保持刷新。用首屏请求瀑布、传输字节和交互可用时间验收。 |
| #106 单条审批 | 用该 PR 前后的真实函数加 mock API，原有 POST → 抽屉变成 POST → 三个全量列表 GET → 抽屉。以 POST 返回的运行或单条状态立即更新，后台定向校验，保留失败反馈和冲突处理。用同一轮批准同时核对抽屉、列表、徽标及请求序列。 |
| #958 定时执行记录搜索 | 无关键词时数据库分页，项目带关键词的路径先读出匹配 payload 的行再在 Go 侧过滤和分页。压测 100 个 job × 200 条历史，先用 `EXPLAIN ANALYZE` 和 CPU/内存核对瓶颈，再推动索引或字段投影。 |
| #12 / #158 Local | #12 的可观测性去重按事件在同一把锁内读取和重写整份 ledger，单 scope 有上限，scope 数无同等上限。本机以 5/100/1,000 个合成 scope 各签收 10 个新事件，中位数约 6.8/89/863 ms。#158 的 DSH 状态查询即使给了 slug 也会遍历和同步排序，100/1,000/5,000 条历史的 p95 约 2.2/34.9/242 ms。可按批登记去重与保留期，显式 slug 直接定位。上游 #1068 已标记 Local 为 legacy。 |

代码：[壳装载](web/app.js#L912-L955)、[全页渲染](web/app.js#L2022)、[Studio 缓存](internal/httpapi/studio.go#L96-L140)、[审批刷新](web/app/approvals.js#L347-L357)、[调度历史搜索](internal/repo/schedule_job_executions.go#L212-L244)、[Local 去重状态](local/.codebuddy/skills/agent-observability/scripts/core/state.py#L518-L553)、[DSH 状态遍历](local/dsh-plugin/index.js#L71-L89)。

### 仅上游已有的候选

| PR | 发现与下一轮验证 |
| --- | --- |
| [#1015](https://github.com/Tencent/LoopForge/pull/1015) | 工作区 turn 收尾清除 Git 临时凭据，旧 checkout helper 会额外枚举 remotes 和 branch refs，而清理只需目录。用上游源码和本机 Git 跑 1/10/50 个合成仓库，对应 3/30/150 次 Git 调用、约 40/381/1,976 ms 中位数。复用目录发现，保留凭据清理和 checkpoint 顺序。 |
| [#1077](https://github.com/Tencent/LoopForge/pull/1077) | 每次标准 Docker SOP CLI 执行增加一次任务用户的 `docker exec` 可用性探测，warm sandbox 也会做。同容器的冷安装受锁保护，最长可进入分钟级超时路径。记录探测、锁等候和安装分段时间，再按 sandbox 代际、任务用户和 PATH 复用成功结果，exec 缺失时重新探测。 |
| [#1096](https://github.com/Tencent/LoopForge/pull/1096) | 项目工作区 CLI 启动前读全平台仓库列表、排序、按项目过滤再比较摘要。已存 `project_id` 可由 `CatalogRepo` 下推过滤，保留排序、目录 hash 与现有 32 KiB 限制。 |
| [#1033](https://github.com/Tencent/LoopForge/pull/1033) | MCP 发现和调用会先取额外凭据，即使该 server 用普通认证。公开构建未注册额外来源，实际负载取决于部署适配器。以带计数测试的凭据来源覆盖普通、所需凭据和缺失三类请求，只在需要绑定时查找。 |

代码：[Git 清理](https://github.com/Tencent/LoopForge/blob/6f606cf2a7354f05d1dbf2085f4eb832b7e89ccb/internal/runtime/git_workspace_scrub.go#L30-L63)、[Docker CLI 探测](https://github.com/Tencent/LoopForge/blob/6f606cf2a7354f05d1dbf2085f4eb832b7e89ccb/internal/runtime/sandbox_docker.go#L168-L172)、[项目仓库上下文](https://github.com/Tencent/LoopForge/blob/6f606cf2a7354f05d1dbf2085f4eb832b7e89ccb/internal/httpapi/assistant_workspace_mode.go#L35-L64)、[MCP 绑定](https://github.com/Tencent/LoopForge/blob/6f606cf2a7354f05d1dbf2085f4eb832b7e89ccb/internal/httpapi/assistant_mcp.go#L397-L403)。

## 核对说明

CI 计时来自 GitHub job/step 元数据。产品侧实测来自单机或模拟环境。其余项按代码控制流和复杂度给出可验证的风险，未用线上监控估算普遍影响。此次审查未应用修复，也未运行完整平台套件。仓库声明的 Go 1.26.11 本机不可用，恢复探针用本机缓存的 Go 1.27.0 和临时测试 overlay 执行，三次通过。

可用以下命令复查窗口代码边界、代表性 PR 合并时间，以及最慢 job 的各步骤。完整 Actions 数据按三日一片分页，遇到计数不一致的时间片再细分。

```bash
git log --first-parent --oneline 09c7652..d630ae0
gh api repos/Tencent/LoopForge/pulls/36 --jq '{created_at,merged_at}'
gh api repos/Tencent/LoopForge/actions/runs/38506444939/attempts/1/jobs --jq '.jobs[] | select(.id == 115664949291) | {started_at,completed_at,steps}'
```

## 逐 PR 覆盖清单

同一 PR 可跨多个范围，含后端 **520** 个、Web **126** 个、Local **48** 个、CI/构建 **147** 个、文档 **172** 个、其他 **5** 个。路径以 Git 差异中的增删路径计数，未折叠重命名。#1109 用 GitHub 的文件列表补齐。单个 PR 可触发多个 workflow 或多次 push，因此 run 数不等于提交或重跑数。清单只在有单 job 达到 15 分钟的核对记录时显示具体时间。跨窗口的 8 月条目标明原创建时间。

<details>
<summary>GitHub 可见的 374 个 PR</summary>

| PR | 状态 | 创建 → 合并 / 关闭（北京时间） | 路径数与范围 | Actions runs / 慢 job | 合并提交 |
| --- | --- | --- | --- | --- | --- |
| [#7 chore(deps): Bump actions/checkout from 4.4.0 to 7.0.1](https://github.com/Tencent/LoopForge/pull/7) | 合并 | 08-04 17:48 → 09-17 16:11 | 2 · CI/构建 | 0 | `88e45cb` fork |
| [#8 chore(deps): Bump actions/setup-node from 4.4.0 to 7.0.0](https://github.com/Tencent/LoopForge/pull/8) | 合并 | 08-04 17:48 → 09-17 16:12 | 2 · CI/构建 | 0 | `f652ce9` fork |
| [#9 chore(deps-dev): Bump wheel from 0.45.1 to 0.48.0 in the python-deps group across 1 directory](https://github.com/Tencent/LoopForge/pull/9) | 关闭 | 08-04 17:49 → 09-17 17:34 | 1 · 其他 | 1 | — |
| [#11 ci: allow retrying releases from existing tags](https://github.com/Tencent/LoopForge/pull/11) | 合并 | 09-10 15:48 → 09-10 15:50 | 1 · CI/构建 | 1 | `1eb65c4` fork |
| [#12 feat(agent-observability): dashboard accuracy fixes + custom model pricing overrides](https://github.com/Tencent/LoopForge/pull/12) | 合并 | 09-15 10:05 → 09-15 17:48 | 106 · Local/CI/构建 | 9 | `0d7ef79` fork |
| [#13 chore(release): prepare v0.2.0](https://github.com/Tencent/LoopForge/pull/13) | 合并 | 09-16 10:41 → 09-16 10:44 | 5 · 文档 | 1 | `6b3049c` fork |
| [#14 feat: add optional Tencent Cloud MCP Initializr](https://github.com/Tencent/LoopForge/pull/14) | 合并 | 09-16 17:26 → 09-16 17:35 | 76 · Local/CI/构建/文档 | 1 | `766189d` fork |
| [#15 fix: keep executable bits when installing Classic hook scripts](https://github.com/Tencent/LoopForge/pull/15) | 合并 | 09-16 17:27 → 09-16 17:51 | 5 · 文档 | 2 | `0c7e211` fork |
| [#16 chore(release): prepare v0.2.1](https://github.com/Tencent/LoopForge/pull/16) | 合并 | 09-16 17:36 → 09-16 17:40 | 7 · Local/文档 | 1 | `285f252` fork |
| [#18 docs: highlight local agent observability](https://github.com/Tencent/LoopForge/pull/18) | 合并 | 09-17 10:54 → 09-17 17:02 | 3 · 文档 | 1 | `a3688df` fork |
| [#21 fix(agent-observability): do not crash when opentelemetry is missing](https://github.com/Tencent/LoopForge/pull/21) | 关闭 | 09-17 11:26 → 09-18 20:28 | 6 · Local | 1 | — |
| [#22 add cli commands lf](https://github.com/Tencent/LoopForge/pull/22) | 关闭 | 09-17 14:40 → 11-09 17:16 | 14 · Local/CI/构建/文档 | 1 | — |
| [#24 feat(portable): add Pi host support (adapter + Portable installer CLI)](https://github.com/Tencent/LoopForge/pull/24) | 合并 | 09-17 14:43 → 09-17 15:09 | 14 · Local/CI/构建/文档 | 1 | `af44e5d` fork |
| [#25 docs: add WeCom community group QR to README](https://github.com/Tencent/LoopForge/pull/25) | 合并 | 09-17 16:33 → 09-17 16:36 | 4 · 文档 | 1 | `1014a19` fork |
| [#26 ci: align tcmcp job action pins with checkout v7 and setup-node v7](https://github.com/Tencent/LoopForge/pull/26) | 合并 | 09-17 16:50 → 09-17 16:52 | 1 · CI/构建 | 1 | `0257602` fork |
| [#27 fix(build): drop the redundant wheel build dependency](https://github.com/Tencent/LoopForge/pull/27) | 合并 | 09-17 17:33 → 09-17 18:30 | 1 · 其他 | 1 | `1764109` fork |
| [#28 fix(agent-observability): degrade gracefully when opentelemetry is missing](https://github.com/Tencent/LoopForge/pull/28) | 关闭 | 09-17 19:33 → 09-17 19:37 | 105 · Local | 0 | — |
| [#29 fix(agent-observability): degrade gracefully when opentelemetry is missing](https://github.com/Tencent/LoopForge/pull/29) | 合并 | 09-17 19:41 → 09-17 19:51 | 3 · Local | 1 | `60af49f` fork |
| [#30 feat(portable): add OpenCode adapter](https://github.com/Tencent/LoopForge/pull/30) | 关闭 | 09-20 00:05 → 11-09 17:16 | 16 · Local/CI/构建/文档 | 4 | — |
| [#32 test(e2e): add end-to-end workflow coverage and a merge gate](https://github.com/Tencent/LoopForge/pull/32) | 合并 | 09-20 18:43 → 10-13 11:05 | 3 · Local | 2 | `136d37c` fork |
| [#33 fix: reject state file paths that escape the project root](https://github.com/Tencent/LoopForge/pull/33) | 关闭 | 09-20 19:58 → 11-09 17:16 | 2 · Local | 1 | — |
| [#34 fix(agent-observability): count stop events in cost and token totals](https://github.com/Tencent/LoopForge/pull/34) | 关闭 | 09-20 19:58 → 11-09 17:16 | 6 · Local | 1 | — |
| [#35 feat: add one-command local observability dashboard](https://github.com/Tencent/LoopForge/pull/35) | 关闭 | 09-22 14:44 → 11-09 17:16 | 10 · Local/文档 | 1 | — |
| [#36 fix(tcmcp): preserve node pool fallback errors](https://github.com/Tencent/LoopForge/pull/36) | 合并 | 09-28 04:30 → 11-12 10:49 | 2 · Local | 7 | `08e7110` fork |
| [#37 chore(deps-dev): Bump setuptools from 75.3.4 to 84.0.0](https://github.com/Tencent/LoopForge/pull/37) | 关闭 | 10-01 23:39 → 11-07 17:00 | 1 · 其他 | 1 | — |
| [#38 chore(deps): update Docker build-push action to v7 in release workflows](https://github.com/Tencent/LoopForge/pull/38) | 合并 | 10-01 23:39 → 11-07 17:00 | 2 · CI/构建 | 10 | `d14b62f` fork |
| [#39 chore(deps): update Docker Buildx action to v4 in release workflows](https://github.com/Tencent/LoopForge/pull/39) | 合并 | 10-01 23:39 → 11-07 17:00 | 2 · CI/构建 | 4 | `fd21531` fork |
| [#40 chore(deps): update Docker login action to v4 in release workflows](https://github.com/Tencent/LoopForge/pull/40) | 合并 | 10-01 23:39 → 11-07 17:00 | 2 · CI/构建 | 4 | `89085eb` fork |
| [#41 test(e2e): add deterministic end-to-end suite and required CI check](https://github.com/Tencent/LoopForge/pull/41) | 合并 | 10-11 22:17 → 10-11 22:50 | 17 · Local/CI/构建/文档 | 2 | `f112b7d` fork |
| [#42 test(e2e): cover all routes/modes, render dashboard in CI, fix issues found (P1 of #31)](https://github.com/Tencent/LoopForge/pull/42) | 合并 | 10-11 22:43 → 10-11 23:06 | 40 · Local/CI/构建/文档 | 4 | `96e38f9` fork |
| [#44 docs(readme): tighten intro, add promo video, table-based quick start](https://github.com/Tencent/LoopForge/pull/44) | 合并 | 10-11 23:27 → 10-11 23:58 | 4 · 文档 | 4 | `3c1b4e1` fork |
| [#45 docs(readme): add hero banner, badges, language switch and section nav](https://github.com/Tencent/LoopForge/pull/45) | 合并 | 10-12 00:04 → 10-12 00:09 | 4 · 文档 | 2 | `73374ef` fork |
| [#46 Add the self-hosted LoopForge platform; move the CLI into cli/](https://github.com/Tencent/LoopForge/pull/46) | 关闭 | 10-12 19:03 → 10-16 14:50 | 1851 · 后端/Web/Local/CI/构建/文档 | 96 | — |
| [#47 Redesign the web workspace; rooms work as a group-chat bot crew](https://github.com/Tencent/LoopForge/pull/47) | 关闭 · platform-integration | 10-14 11:28 → 10-16 14:50 | 38 · 后端/Web/CI/构建 | 3 / [49:58](https://github.com/Tencent/LoopForge/actions/runs/38506444939/job/115664949291) | — |
| [#48 Add audited self-hosted platform and retain Local workflows](https://github.com/Tencent/LoopForge/pull/48) | 合并 | 10-16 14:49 → 10-19 21:56 | 2787 · 后端/Web/Local/CI/构建/文档 | 39 | `fb4e331` fork |
| [#50 chore(deps): Bump golang.org/x/sys from 0.48.0 to 0.49.0](https://github.com/Tencent/LoopForge/pull/50) | 关闭 | 10-19 21:57 → 11-07 17:02 | 2 · 后端 | 3 | — |
| [#51 chore(deps): Bump github.com/klauspost/compress from 1.20.2 to 1.20.3](https://github.com/Tencent/LoopForge/pull/51) | 关闭 | 10-19 21:57 → 11-06 18:27 | 2 · 后端 | 6 | — |
| [#52 chore(deps-dev): Bump setuptools from 75.3.4 to 84.0.0 in /local](https://github.com/Tencent/LoopForge/pull/52) | 关闭 | 10-19 21:57 → 11-01 23:38 | 1 · Local | 3 | — |
| [#53 chore(deps): Bump golang.org/x/crypto from 0.57.0 to 0.58.0](https://github.com/Tencent/LoopForge/pull/53) | 关闭 | 10-19 21:57 → 11-07 17:02 | 2 · 后端 | 9 | — |
| [#54 chore(deps): update MySQL driver and refresh license notices](https://github.com/Tencent/LoopForge/pull/54) | 合并 | 10-19 21:57 → 11-07 17:00 | 3 · 后端 | 6 | `f53f14f` fork |
| [#55 chore(deps): update Go x modules and refresh license notices](https://github.com/Tencent/LoopForge/pull/55) | 合并 | 10-19 21:58 → 11-07 17:02 | 3 · 后端 | 12 | `57f12ad` fork |
| [#56 chore(deps-dev): Bump @playwright/test from 1.55.1 to 1.64.1 in /e2e](https://github.com/Tencent/LoopForge/pull/56) | 合并 | 10-19 21:58 → 11-07 16:56 | 2 · CI/构建 | 9 | `055bffa` fork |
| [#57 chore(deps): Bump actions/setup-node from 7.0.0 to 7.1.0 in the actions group across 1 directory](https://github.com/Tencent/LoopForge/pull/57) | 合并 | 10-19 21:58 → 11-07 16:57 | 4 · CI/构建 | 18 | `738642e` fork |
| [#58 chore(deps): Bump actions/upload-artifact from 4 to 7](https://github.com/Tencent/LoopForge/pull/58) | 合并 | 10-19 21:58 → 11-07 16:57 | 2 · CI/构建 | 6 | `0d80d43` fork |
| [#60 docs: restore community QR and add playable demo videos](https://github.com/Tencent/LoopForge/pull/60) | 合并 | 10-20 14:31 → 10-20 14:57 | 3 · 文档 | 3 | `53fe87b` fork |
| [#61 fix: confine runtime file writes and preserve synced issue status](https://github.com/Tencent/LoopForge/pull/61) | 合并 | 10-20 14:51 → 10-20 15:12 | 43 · 后端/Web/文档 | 3 | `91f2b42` fork |
| [#62 feat: add self-managed GitLab repository integration](https://github.com/Tencent/LoopForge/pull/62) | Draft | 10-27 18:08 | 97 · 后端/Web/CI/构建/文档 | 9 | — |
| [#63 fix(local): preserve manual gates without PyYAML](https://github.com/Tencent/LoopForge/pull/63) | 合并 | 11-01 10:17 → 11-12 14:32 | 2 · Local | 3 | `e0afaa2` fork |
| [#64 fix(deps): upgrade setuptools while preserving Python 3.8 support](https://github.com/Tencent/LoopForge/pull/64) | 合并 | 11-01 23:38 → 11-07 16:59 | 1 · Local | 6 | `2e53e84` fork |
| [#65 feat: sync chat home, conversation workspaces and delivery tracking](https://github.com/Tencent/LoopForge/pull/65) | 合并 | 11-03 17:25 → 11-03 22:24 | 1046 · 后端/Web/CI/构建/文档 | 36 | `752e4fe` fork |
| [#66 chore: remove integrations that only work on a private network](https://github.com/Tencent/LoopForge/pull/66) | 合并 | 11-03 23:15 → 11-03 23:25 | 129 · 后端/Web/Local/CI/构建/文档 | 3 | `0e3fee7` fork |
| [#67 Sync platform updates: shared AI schedules, role subagents and conversation delivery](https://github.com/Tencent/LoopForge/pull/67) | 合并 | 11-05 22:18 → 11-05 22:53 | 251 · 后端/Web/CI/构建/文档 | 6 / [18:45](https://github.com/Tencent/LoopForge/actions/runs/42043758668/job/127301089896) | `557482e` fork |
| [#68 fix: simplify GitHub connection and first-run deployment](https://github.com/Tencent/LoopForge/pull/68) | 合并 | 11-06 15:19 → 11-06 18:25 | 57 · 后端/Web/CI/构建/文档 | 18 | `59acccb` fork |
| [#69 docs(readme): redesign README around the chat workbench with fresh screenshots](https://github.com/Tencent/LoopForge/pull/69) | 合并 | 11-06 16:53 → 11-06 18:11 | 29 · 文档 | 15 / [16:12](https://github.com/Tencent/LoopForge/actions/runs/42197998118/job/127825994433) | `7c4f98b` fork |
| [#70 chore(release): prepare v0.2.3](https://github.com/Tencent/LoopForge/pull/70) | 合并 | 11-06 17:28 → 11-06 17:53 | 7 · 后端/Local/文档 | 6 | `836009a` fork |
| [#71 feat: prebuild the generic SOP sandbox image during install](https://github.com/Tencent/LoopForge/pull/71) | 关闭 | 11-06 17:44 → 11-06 22:19 | 7 · 后端/CI/构建/文档 | 3 | — |
| [#72 feat: choose a workspace branch and collect role deliverables](https://github.com/Tencent/LoopForge/pull/72) | 关闭 | 11-06 18:10 → 11-06 18:31 | 61 · 后端/Web/文档 | 6 / [17:09](https://github.com/Tencent/LoopForge/actions/runs/42202795668/job/127841960052) | — |
| [#73 docs(readme): embed the product video inline](https://github.com/Tencent/LoopForge/pull/73) | 合并 | 11-06 18:16 → 11-06 18:20 | 5 · 文档 | 3 | `46ecdf2` fork |
| [#74 Feature/workspace branch and role deliverables](https://github.com/Tencent/LoopForge/pull/74) | 合并 | 11-06 18:31 → 11-06 21:20 | 61 · 后端/Web/文档 | 9 | `c7114fc` fork |
| [#75 feat: 新用户注册时可自选加入项目，并支持自助加入开放项目](https://github.com/Tencent/LoopForge/pull/75) | 关闭 | 11-06 19:12 → 11-11 15:59 | 19 · 后端/Web/文档 | 6 | — |
| [#76 fix(auth): restrict SSO return_url to same-site hosts](https://github.com/Tencent/LoopForge/pull/76) | 合并 | 11-06 20:21 → 11-06 20:34 | 3 · 后端 | 3 | `1659128` fork |
| [#77 docs(readme): use the English product video in README.md](https://github.com/Tencent/LoopForge/pull/77) | 合并 | 11-06 20:25 → 11-06 20:33 | 1 · 文档 | 3 | `50dc89a` fork |
| [#78 fix: harden git remote URL handling](https://github.com/Tencent/LoopForge/pull/78) | 合并 | 11-06 20:34 → 11-06 20:43 | 9 · 后端 | 3 | `75b4080` fork |
| [#84 ci(release): publish the platform from vX.Y.Z tags](https://github.com/Tencent/LoopForge/pull/84) | 合并 | 11-06 22:31 → 11-06 22:40 | 9 · Local/CI/构建/文档 | 3 | `b4ea418` fork |
| [#95 test(e2e): wait for committed replies and ignore stalled first loads](https://github.com/Tencent/LoopForge/pull/95) | 合并 | 11-06 22:51 → 11-06 23:10 | 2 · CI/构建 | 3 / [17:31](https://github.com/Tencent/LoopForge/actions/runs/42237750215/job/127959730960) | `887d95e` fork |
| [#96 feat(deploy): start from the pinned GHCR release image (0.3.1)](https://github.com/Tencent/LoopForge/pull/96) | 合并 | 11-07 00:37 → 11-07 00:45 | 13 · Local/CI/构建/文档 | 3 | `798d006` fork |
| [#97 Sync runtime updates: native sandbox driver, unattended agents, personal credentials on platform runners](https://github.com/Tencent/LoopForge/pull/97) | 关闭 | 11-07 01:13 → 11-07 01:22 | 42 · 后端/Web/CI/构建/文档 | 3 | — |
| [#98 fix(install): listen on 0.0.0.0 without requiring LOOPFORGE_PUBLIC_URL](https://github.com/Tencent/LoopForge/pull/98) | 合并 | 11-07 01:14 → 11-07 01:30 | 9 · CI/构建/文档 | 3 | `48193c6` fork |
| [#99 fix(web,chatbot): copy GitHub device code on HTTP; rename remaining AgentFlow labels to LoopForge](https://github.com/Tencent/LoopForge/pull/99) | 合并 | 11-07 01:43 → 11-07 02:02 | 8 · Web/CI/构建 | 6 | `98aa9e7` fork |
| [#100 feat(runtime): install Claude Code on first use instead of bundling it (0.3.2)](https://github.com/Tencent/LoopForge/pull/100) | 合并 | 11-07 01:46 → 11-07 01:55 | 10 · 后端/CI/构建/文档 | 3 | `d9f7b84` fork |
| [#101 fix: show LoopForge instead of AgentFlow in user-visible text](https://github.com/Tencent/LoopForge/pull/101) | 合并 | 11-07 02:21 → 11-07 02:32 | 35 · 后端/Web/文档 | 3 | `dfd6f7e` fork |
| [#102 docs: keep bash scripts/install.sh as the quick start alongside ./start.sh](https://github.com/Tencent/LoopForge/pull/102) | 合并 | 11-07 02:25 → 11-07 02:33 | 3 · 文档 | 3 | `8b9e564` fork |
| [#103 fix(web): keep the credential wizard from stealing focus from a field the user picked](https://github.com/Tencent/LoopForge/pull/103) | 合并 | 11-07 02:26 → 11-07 02:37 | 2 · Web/CI/构建 | 3 | `c4b15d1` fork |
| [#104 [DO NOT MERGE] Republish 0.3.0 and 0.3.1 images without Claude Code](https://github.com/Tencent/LoopForge/pull/104) | 关闭 | 11-07 02:41 → 11-07 03:06 | 1 · CI/构建 | 4 / [17:46](https://github.com/Tencent/LoopForge/actions/runs/42273048049/job/128082266067) | — |
| [#105 fix: Codex and Claude Code SOP roles in Docker sandboxes; label installer addresses (0.3.3)](https://github.com/Tencent/LoopForge/pull/105) | 合并 | 11-07 03:29 → 11-07 03:40 | 8 · 后端/CI/构建/文档 | 3 | `ad5300a` fork |
| [#106 fix(web): refresh approvals and cover requirement and approval UI journeys](https://github.com/Tencent/LoopForge/pull/106) | 合并 | 11-07 03:51 → 11-12 15:17 | 5 · Web/CI/构建 | 6 | `c7e2f74` fork |
| [#107 docs: use ./start.sh as the quick start](https://github.com/Tencent/LoopForge/pull/107) | 合并 | 11-07 04:20 → 11-07 04:28 | 3 · 文档 | 3 | `95954d7` fork |
| [#108 fix: pick the CodeBuddy network environment from a list and link to its key page](https://github.com/Tencent/LoopForge/pull/108) | 合并 | 11-07 16:17 → 11-07 17:07 | 22 · 后端/Web/CI/构建/文档 | 6 | `ce0116d` fork |
| [#110 feat(platform): member accounts and a shared CodeBuddy key in 平台管理; arm64 images](https://github.com/Tencent/LoopForge/pull/110) | 合并 | 11-07 23:52 → 11-09 02:26 | 42 · 后端/Web/CI/构建/文档 | 18 | `3dab8fb` fork |
| [#116 feat: select a native sandbox per OS and run more agent CLIs](https://github.com/Tencent/LoopForge/pull/116) | 合并 | 11-08 12:53 → 11-09 15:19 | 85 · 后端/Web/CI/构建/文档 | 21 | `89d5ed4` fork |
| [#119 fix(release): ship ARM64 platform images in 0.3.4](https://github.com/Tencent/LoopForge/pull/119) | 合并 | 11-09 10:49 → 11-09 15:28 | 6 · 后端/CI/构建/文档 | 3 | `e47dcd0` fork |
| [#121 feat(platform): support DeepSeek as an opt-in SOP executor via Codex](https://github.com/Tencent/LoopForge/pull/121) | 开放 | 11-09 12:10 | 37 · 后端/Web/CI/构建/文档 | 36 | — |
| [#123 fix(ci): make Gitleaks downloads resilient to HTTP 504 and rate limits](https://github.com/Tencent/LoopForge/pull/123) | 合并 | 11-09 15:44 → 11-09 16:20 | 5 · Local/CI/构建 | 3 | `ffaed15` fork |
| [#129 fix(workspace): never report a failed or interrupted restore as ready](https://github.com/Tencent/LoopForge/pull/129) | 合并 | 11-09 18:04 → 11-11 01:59 | 4 · 后端 | 3 | `eb28d23` fork |
| [#130 fix(workspace): fail fast when a checkpoint operation meets an open terminal](https://github.com/Tencent/LoopForge/pull/130) | 合并 | 11-09 18:19 → 11-11 01:59 | 4 · 后端 | 3 | `1f072c2` fork |
| [#131 feat(chatbot): mark unread replies in the session sidebar like Codex](https://github.com/Tencent/LoopForge/pull/131) | 合并 | 11-09 18:22 → 11-11 01:59 | 18 · 后端/Web/文档 | 6 | `f6b9207` fork |
| [#132 fix(access): never emit http:// endpoints for token-bearing snippets on https](https://github.com/Tencent/LoopForge/pull/132) | 合并 | 11-09 18:40 → 11-12 19:36 | 3 · Web/CI/构建 | 9 | `7df6ada` fork |
| [#133 fix(runtime): keep index.lock while an alternate-path holder is live](https://github.com/Tencent/LoopForge/pull/133) | 合并 | 11-09 18:45 → 11-12 18:49 | 2 · 后端 | 9 | `6c8597e` fork |
| [#135 fix(web): restore the AI orchestrator build animations](https://github.com/Tencent/LoopForge/pull/135) | 合并 | 11-09 19:48 → 11-12 19:59 | 2 · Web/CI/构建 | 12 | `0721241` fork |
| [#136 fix(auth): do not trust identity headers from forwarded clients](https://github.com/Tencent/LoopForge/pull/136) | 合并 | 11-09 19:54 → 11-12 19:37 | 5 · 后端/文档 | 9 | `9bd41ca` fork |
| [#137 fix(repo): keep credentials out of repository clone URLs](https://github.com/Tencent/LoopForge/pull/137) | 合并 | 11-09 20:03 → 11-13 01:49 | 19 · 后端/文档 | 24 | `1c71aa0` fork |
| [#138 fix(workspace): reject a foreign remote branch when resuming a new work branch](https://github.com/Tencent/LoopForge/pull/138) | 合并 | 11-09 20:06 → 11-11 01:59 | 2 · 后端 | 3 | `43195ff` fork |
| [#139 fix(auth): define password-attempt limits per server and per request kind](https://github.com/Tencent/LoopForge/pull/139) | 合并 | 11-09 20:17 → 11-11 01:59 | 4 · 后端 | 3 | `3d35a52` fork |
| [#140 ci(platform): run repository-bound and second-user e2e API tests](https://github.com/Tencent/LoopForge/pull/140) | 合并 | 11-09 20:20 → 11-12 18:49 | 1 · CI/构建 | 6 | `441f897` fork |
| [#141 fix(chatbot): explain repository permissions in quick access before the last step](https://github.com/Tencent/LoopForge/pull/141) | 合并 | 11-09 20:27 → 11-11 01:59 | 4 · Web | 3 | `63c9e46` fork |
| [#143 fix: show the Pi mark in the chat CLI selector](https://github.com/Tencent/LoopForge/pull/143) | 合并 | 11-09 23:09 → 11-10 22:44 | 7 · Web | 3 | `ca77087` fork |
| [#144 feat: run parallel SOP branches together](https://github.com/Tencent/LoopForge/pull/144) | 合并 | 11-09 23:10 → 11-10 22:45 | 32 · 后端/CI/构建/文档 | 3 | `6f79c7a` fork |
| [#145 feat: make requirement clarification optional](https://github.com/Tencent/LoopForge/pull/145) | 合并 | 11-09 23:11 → 11-10 22:47 | 60 · 后端/Web/CI/构建/文档 | 3 | `00875a8` fork |
| [#146 feat: run scheduled tasks as normal chat turns on the chosen CLI](https://github.com/Tencent/LoopForge/pull/146) | 合并 | 11-09 23:11 → 11-10 22:48 | 30 · 后端/Web/CI/构建/文档 | 3 | `2b282af` fork |
| [#147 feat: choose the CLI for the SOP orchestration assistant](https://github.com/Tencent/LoopForge/pull/147) | 合并 | 11-09 23:12 → 11-10 22:48 | 7 · Web/文档 | 3 | `f24a669` fork |
| [#148 fix: explain why an issue cannot be deleted and hide unset stage times](https://github.com/Tencent/LoopForge/pull/148) | 合并 | 11-09 23:12 → 11-10 23:32 | 21 · 后端/Web/CI/构建/文档 | 6 | `f21b960` fork |
| [#149 fix(workspace): report opening progress and preserve failed workspaces](https://github.com/Tencent/LoopForge/pull/149) | 开放 | 11-10 00:02 | 42 · 后端/Web/文档 | 27 / [15:04](https://github.com/Tencent/LoopForge/actions/runs/43476347142/job/132061048762) | — |
| [#150 fix(workspace): report progress and stop hung clones and image builds (#124)](https://github.com/Tencent/LoopForge/pull/150) | 合并 | 11-10 00:03 → 11-10 20:23 | 17 · 后端/文档 | 9 | `d7df441` fork |
| [#151 chore: reserve DSH integration work](https://github.com/Tencent/LoopForge/pull/151) | 关闭 | 11-10 00:31 → 11-12 11:55 | 0 · 其他 | 3 | — |
| [#152 chore: reserve Hermes and DSH integration work](https://github.com/Tencent/LoopForge/pull/152) | 关闭 | 11-10 00:31 → 11-12 11:55 | 0 · 其他 | 3 | — |
| [#153 [closed for security] feat: self-hosted GitLab providers](https://github.com/Tencent/LoopForge/pull/153) | 关闭 | 11-10 09:48 → 11-10 09:52 | 9 · 后端 | 3 | — |
| [#154 [closed for security] feat: self-hosted GitLab providers](https://github.com/Tencent/LoopForge/pull/154) | 关闭 | 11-10 09:56 → 11-10 10:13 | 9 · 后端 | 3 | — |
| [#155 feat: support self-hosted GitLab providers (gitlab: prefix, multi-instance)](https://github.com/Tencent/LoopForge/pull/155) | 开放 | 11-10 10:15 | 17 · 后端/文档 | 27 | — |
| [#157 feat: ask before handing a requirement to an SOP](https://github.com/Tencent/LoopForge/pull/157) | 合并 | 11-10 11:15 → 11-12 20:21 | 39 · 后端/Web/CI/构建/文档 | 15 | `deca63a` fork |
| [#158 feat(local): support Hermes and DSH host integrations](https://github.com/Tencent/LoopForge/pull/158) | 合并 | 11-10 13:14 → 11-10 19:51 | 32 · Local/文档 | 3 | `e0ce2de` fork |
| [#159 feat(assistant): lightweight GitHub tools for Main Bot without opening a workspace](https://github.com/Tencent/LoopForge/pull/159) | 合并 | 11-10 16:45 → 11-10 19:29 | 4 · 后端 | 6 | `f258944` fork |
| [#161 feat(runtime): add ZCode as an agent CLI executor (#125)](https://github.com/Tencent/LoopForge/pull/161) | 合并 | 11-10 17:46 → 11-10 20:26 | 24 · 后端/Web | 6 | `4867a57` fork |
| [#162 feat: render sandbox dev servers natively in the side browser](https://github.com/Tencent/LoopForge/pull/162) | 合并 | 11-10 19:49 → 11-12 19:55 | 39 · 后端/Web | 9 | `41e6a07` fork |
| [#163 docs: add cloud server install guide, platform support and a Chinese deployment guide](https://github.com/Tencent/LoopForge/pull/163) | 开放 | 11-10 22:41 | 4 · 文档 | 6 | — |
| [#164 chore(deps): bump golang.org/x/sys from 0.49.0 to 0.50.0](https://github.com/Tencent/LoopForge/pull/164) | 开放 | 11-10 23:35 | 3 · 后端 | 3 | — |
| [#165 feat(runtime): let ZCode take a reasoning effort (#125)](https://github.com/Tencent/LoopForge/pull/165) | 开放 | 11-10 23:53 | 7 · 后端 | 3 | — |
| [#166 fix(deps): patch vulnerable chatbot test toolchain](https://github.com/Tencent/LoopForge/pull/166) | 合并 | 11-11 10:02 → 11-12 16:31 | 4 · Web | 6 | `39e1dac` fork |
| [#168 fix(web): publish a workflow only after its saved content changes](https://github.com/Tencent/LoopForge/pull/168) | 合并 | 11-11 15:19 → 11-11 19:32 | 14 · 后端/Web/CI/构建/文档 | 6 | `3d48656` fork |
| [#169 feat(web): consolidate settings navigation and merge asset pages](https://github.com/Tencent/LoopForge/pull/169) | 合并 | 11-11 15:24 → 11-11 20:08 | 52 · Web/CI/构建/文档 | 18 | `29fbcc8` fork |
| [#172 feat(runner): support native Windows execution and guided onboarding](https://github.com/Tencent/LoopForge/pull/172) | 关闭 | 11-11 15:52 → 11-15 09:28 | 99 · 后端/Web/CI/构建/文档 | 44 / [15:04](https://github.com/Tencent/LoopForge/actions/runs/43580321196/job/132403445540) | — |
| [#173 fix(image): include WeCom CLI for notifications](https://github.com/Tencent/LoopForge/pull/173) | 开放 | 11-11 15:56 | 7 · CI/构建/文档 | 17 / [16:42](https://github.com/Tencent/LoopForge/actions/runs/44548577897/job/135645773710) | — |
| [#174 Add a VS Code integration for LoopForge workflows](https://github.com/Tencent/LoopForge/pull/174) | 开放 | 11-11 16:04 | 14 · CI/构建/文档 | 15 | — |
| [#175 Support personal conversation workspaces](https://github.com/Tencent/LoopForge/pull/175) | 开放 | 11-11 16:36 | 26 · 后端 | 18 | — |
| [#176 fix(sop): provide a default sandbox and validate sandbox selection](https://github.com/Tencent/LoopForge/pull/176) | 合并 | 11-11 16:59 → 11-11 22:29 | 11 · 后端/Web/CI/构建/文档 | 9 | `e83d9c8` fork |
| [#177 feat: keep a personal model catalog for Codex, Claude Code, and agent CLIs](https://github.com/Tencent/LoopForge/pull/177) | 合并 | 11-11 17:38 → 11-11 19:33 | 13 · 后端/Web/文档 | 6 | `367b41d` fork |
| [#178 feat(web): animate running SOP node activity](https://github.com/Tencent/LoopForge/pull/178) | 合并 | 11-11 17:45 → 11-12 16:12 | 8 · 后端/Web/CI/构建/文档 | 6 | `5884832` fork |
| [#179 fix(deploy): update Go security patch and install missing prerequisites](https://github.com/Tencent/LoopForge/pull/179) | 合并 | 11-11 17:49 → 11-11 18:48 | 10 · 后端/CI/构建/文档 | 6 | `a529e25` fork |
| [#182 chore(release): prepare v0.3.5](https://github.com/Tencent/LoopForge/pull/182) | 合并 | 11-11 20:40 → 11-11 21:39 | 5 · CI/构建/文档 | 6 | `6b25d20` fork |
| [#183 feat(chat): install a missing CLI from the conversation failure](https://github.com/Tencent/LoopForge/pull/183) | 合并 | 11-11 21:04 → 11-12 23:11 | 19 · 后端/Web/文档 | 3 | `dfebabc` fork |
| [#184 fix(chatbot): clear the missing-key notice once quick access saves a key](https://github.com/Tencent/LoopForge/pull/184) | 合并 | 11-11 22:11 → 11-12 19:36 | 4 · Web | 3 | `911e393` fork |
| [#185 chore(release): prepare v0.3.6](https://github.com/Tencent/LoopForge/pull/185) | 合并 | 11-11 22:30 → 11-11 22:56 | 3 · CI/构建/文档 | 3 | `bdf8723` fork |
| [#186 ci: keep browser jobs running when runners cannot reach the Ubuntu archives](https://github.com/Tencent/LoopForge/pull/186) | 合并 | 11-11 23:04 → 11-11 23:33 | 3 · CI/构建 | 6 | `c87694a` fork |
| [#187 fix(preview): isolate archived previews from studio sessions](https://github.com/Tencent/LoopForge/pull/187) | 开放 | 11-11 23:39 | 26 · 后端/Web/CI/构建/文档 | 12 / [16:31](https://github.com/Tencent/LoopForge/actions/runs/44109549197/job/134131071493) | — |
| [#188 ci(release): verify installer upgrades before promoting platform releases](https://github.com/Tencent/LoopForge/pull/188) | 开放 | 11-12 00:13 | 16 · CI/构建/文档 | 3 | — |
| [#192 fix(deps): patch vulnerable webtest coverage toolchain](https://github.com/Tencent/LoopForge/pull/192) | 开放 | 11-12 10:11 | 2 · CI/构建 | 3 | — |
| [#194 ci: probe whether runners can still resolve the Ubuntu archives](https://github.com/Tencent/LoopForge/pull/194) | 关闭 | 11-12 11:10 → 11-12 15:07 | 1 · CI/构建 | 4 | — |
| [#196 fix(image): bundle wecom-cli so WeCom notification bots work out of the box](https://github.com/Tencent/LoopForge/pull/196) | 关闭 | 11-12 12:06 → 11-12 12:24 | 6 · CI/构建/文档 | 3 | — |
| [#199 fix(web): let a SOP role choose any built-in CLI](https://github.com/Tencent/LoopForge/pull/199) | 合并 | 11-12 14:58 → 11-13 13:00 | 10 · 后端/Web/文档 | 3 | `b751e09` fork |
| [#200 docs(readme): explain the benefits of cloud deployment](https://github.com/Tencent/LoopForge/pull/200) | 开放 | 11-12 15:25 | 2 · 文档 | 9 | — |
| [#202 chore(release): prepare v0.3.7](https://github.com/Tencent/LoopForge/pull/202) | 合并 | 11-12 16:16 → 11-12 16:19 | 4 · CI/构建/文档 | 3 | `caa89b7` fork |
| [#204 fix(web): remove the unused page-feedback button](https://github.com/Tencent/LoopForge/pull/204) | 开放 | 11-12 16:40 | 21 · 后端/Web/CI/构建/文档 | 6 | — |
| [#205 ci: mirror the platform image to TCR after publishing GHCR](https://github.com/Tencent/LoopForge/pull/205) | 合并 | 11-12 17:08 → 11-12 17:25 | 1 · CI/构建 | 6 | `35cc1eb` fork |
| [#206 feat(web): show the image commit in the corner](https://github.com/Tencent/LoopForge/pull/206) | 合并 | 11-12 17:12 → 11-12 19:58 | 8 · 后端/CI/构建 | 6 | `cf1348e` fork |
| [#207 feat(git): support SSH repositories with per-host user keys](https://github.com/Tencent/LoopForge/pull/207) | 合并 | 11-12 17:30 → 11-12 23:12 | 89 · 后端/Web/CI/构建/文档 | 6 | `e8d7e14` fork |
| [#208 fix(bind-cli): start the systemd user bus before installing the service](https://github.com/Tencent/LoopForge/pull/208) | 合并 | 11-12 17:31 → 11-13 02:17 | 2 · CI/构建 | 3 | `ad08f28` fork |
| [#209 fix(security): confine agent CLI config writes to home](https://github.com/Tencent/LoopForge/pull/209) | 合并 | 11-12 17:31 → 11-12 23:13 | 7 · 后端 | 12 | `b043c2d` fork |
| [#210 feat(issues): allow force deletion of settled execution history](https://github.com/Tencent/LoopForge/pull/210) | 合并 | 11-12 17:31 → 11-13 03:02 | 26 · 后端/Web/CI/构建/文档 | 6 | `a7c6f15` fork |
| [#211 feat(workspace): edit run workspaces while agents run](https://github.com/Tencent/LoopForge/pull/211) | 合并 | 11-12 17:31 → 11-13 03:01 | 31 · 后端/Web/CI/构建 | 9 | `03c3ab6` fork |
| [#212 fix(schedule): drop the length limit on scheduled task prompts](https://github.com/Tencent/LoopForge/pull/212) | 合并 | 11-12 17:32 → 11-13 02:17 | 8 · 后端/Web/文档 | 3 | `9d832c7` fork |
| [#213 docs: pull the platform image through the NJU GHCR cache](https://github.com/Tencent/LoopForge/pull/213) | 合并 | 11-12 17:48 → 11-12 20:03 | 11 · Local/CI/构建/文档 | 15 | `0ccfde7` fork |
| [#214 feat(local): support ZCode portable workflow adapter (#125)](https://github.com/Tencent/LoopForge/pull/214) | 开放 | 11-12 18:09 | 19 · Local/文档 | 12 / [17:18](https://github.com/Tencent/LoopForge/actions/runs/44259438595/job/134648923452) | — |
| [#216 chore(release): prepare v0.3.8](https://github.com/Tencent/LoopForge/pull/216) | 合并 | 11-12 20:35 → 11-12 20:47 | 5 · CI/构建/文档 | 3 | `39f4fe7` fork |
| [#217 docs: replace brand name LoopForge with NLForge in documentation](https://github.com/Tencent/LoopForge/pull/217) | 关闭 | 11-12 20:49 → 11-12 20:52 | 29 · Local/文档 | 3 | — |
| [#379 fix: bundle DeepSeek Harness, install agent CLIs in SOP sandboxes, redact their keys](https://github.com/Tencent/LoopForge/pull/379) | 合并 | 11-13 10:58 → 11-13 16:17 | 15 · 后端/CI/构建/文档 | 9 | `223bd41` fork |
| [#382 docs: add the Trendshift badge to the README](https://github.com/Tencent/LoopForge/pull/382) | 合并 | 11-13 11:05 → 11-13 11:12 | 2 · 文档 | 3 | `d85f605` fork |
| [#398 Add conditional routing to SOP workflows](https://github.com/Tencent/LoopForge/pull/398) | 开放 | 11-13 12:16 | 42 · 后端/Web/CI/构建/文档 | 18 | — |
| [#403 feat: add Qoder CLI executor](https://github.com/Tencent/LoopForge/pull/403) | 开放 | 11-13 12:41 | 39 · 后端/Web/CI/构建/文档 | 18 | — |
| [#413 Add node workspace checkpoints](https://github.com/Tencent/LoopForge/pull/413) | Draft | 11-13 13:19 | 63 · 后端/Web/文档 | 9 | — |
| [#418 Requirement clarify improvment](https://github.com/Tencent/LoopForge/pull/418) | 关闭 | 11-13 13:41 → 11-13 13:43 | 51 · 后端/Web/CI/构建/文档 | 0 | — |
| [#429 fix(chat): stream DeepSeek prompts and check CLI version](https://github.com/Tencent/LoopForge/pull/429) | 开放 | 11-13 14:19 | 6 · 后端 | 9 | — |
| [#463 fix(chat): keep the CLI of a steering message](https://github.com/Tencent/LoopForge/pull/463) | 开放 | 11-13 16:09 | 3 · 后端/文档 | 3 | — |
| [#468 feat(assistant): surface specific CLI failures and export turn diagnostics](https://github.com/Tencent/LoopForge/pull/468) | 合并 | 11-13 16:20 → 11-13 20:02 | 46 · 后端/Web/文档 | 9 | `2ddddc5` fork |
| [#478 feat: support project asset references in AI scheduled tasks](https://github.com/Tencent/LoopForge/pull/478) | 开放 | 11-13 17:00 | 17 · 后端/Web/CI/构建 | 21 / [15:20](https://github.com/Tencent/LoopForge/actions/runs/43461545113/job/132073211618) | — |
| [#486 ci: run the browser scroll test with the other real-Chromium tests](https://github.com/Tencent/LoopForge/pull/486) | 合并 | 11-13 17:32 → 11-13 17:52 | 1 · CI/构建 | 3 | `297d4dc` fork |
| [#491 fix(cursor): fold effort variants into one model row](https://github.com/Tencent/LoopForge/pull/491) | 合并 | 11-13 17:56 → 11-13 19:58 | 25 · 后端/Web | 6 | `8e74846` fork |
| [#494 feat(chat): run native agent CLIs in conversation workspaces on bound execution machines](https://github.com/Tencent/LoopForge/pull/494) | 合并 | 11-13 18:05 → 11-13 21:54 | 145 · 后端/Web/CI/构建/文档 | 12 | `cf1cf3a` fork |
| [#496 docs(readme): list all ten executors and their credential rules](https://github.com/Tencent/LoopForge/pull/496) | 合并 | 11-13 18:16 → 11-13 18:21 | 2 · 文档 | 3 | `d53c576` fork |
| [#503 Add GitLab personal access token support](https://github.com/Tencent/LoopForge/pull/503) | 开放 | 11-13 18:41 | 80 · 后端/Web/CI/构建/文档 | 17 | — |
| [#546 fix(schedule): make unattended AI loops reliable](https://github.com/Tencent/LoopForge/pull/546) | 合并 | 11-13 22:08 → 11-13 22:26 | 21 · 后端/文档 | 6 | `2441f23` fork |
| [#551 chore(release): prepare v0.3.9-rc.1](https://github.com/Tencent/LoopForge/pull/551) | 合并 | 11-13 22:32 → 11-13 22:48 | 3 · CI/构建/文档 | 3 | `31dff50` fork |
| [#558 fix(chat): replay the current mode's context when a resume falls back](https://github.com/Tencent/LoopForge/pull/558) | 合并 | 11-13 23:03 → 11-14 00:54 | 3 · 后端 | 3 | `1b60847` fork |
| [#559 Revert "chore(release): prepare v0.3.9-rc.1 (#551)"](https://github.com/Tencent/LoopForge/pull/559) | 合并 | 11-13 23:07 → 11-13 23:20 | 3 · CI/构建/文档 | 3 | `eb4d4d5` fork |
| [#580 fix(chat): keep native conversation roots where the Docker daemon can mount them](https://github.com/Tencent/LoopForge/pull/580) | 合并 | 11-14 00:26 → 11-14 00:54 | 8 · 后端/CI/构建/文档 | 3 | `3d3254f` fork |
| [#586 fix(seed): add openssh-client to the built-in generic-backend Dockerfile](https://github.com/Tencent/LoopForge/pull/586) | 合并 | 11-14 00:53 → 11-14 01:14 | 7 · 后端/CI/构建/文档 | 6 | `e5a6417` fork |
| [#597 fix(workspace): install Cursor CLI and pinned dsh in capability-dropped sandboxes](https://github.com/Tencent/LoopForge/pull/597) | 合并 | 11-14 01:27 → 11-14 01:41 | 2 · 后端/文档 | 3 | `e48ced5` fork |
| [#601 fix(chat): remove a conversation's shared native root when it is deleted](https://github.com/Tencent/LoopForge/pull/601) | 合并 | 11-14 01:41 → 11-14 02:02 | 5 · 后端/文档 | 3 | `1326982` fork |
| [#604 feat(sandbox): mount the platform toolchain into custom-image sandboxes](https://github.com/Tencent/LoopForge/pull/604) | 合并 | 11-14 01:49 → 11-14 02:51 | 10 · 后端/CI/构建/文档 | 12 | `1dde975` fork |
| [#605 test(runtime): disable plugin sync in native gateway acceptance](https://github.com/Tencent/LoopForge/pull/605) | 合并 | 11-14 01:53 → 11-14 02:10 | 1 · 后端 | 3 | `13dbfc4` fork |
| [#609 fix(runtime): skip CodeBuddy's built-in plugin marketplaces](https://github.com/Tencent/LoopForge/pull/609) | 合并 | 11-14 02:06 → 11-14 02:31 | 3 · 后端/文档 | 3 | `ca9325a` fork |
| [#628 fix(browser): capture a sharp frame when the settled screencast sends none](https://github.com/Tencent/LoopForge/pull/628) | 开放 | 11-14 02:49 | 6 · 后端/文档 | 9 | — |
| [#644 chore(release): prepare v0.3.9-rc.1](https://github.com/Tencent/LoopForge/pull/644) | 关闭 | 11-14 03:39 → 11-14 03:43 | 3 · CI/构建/文档 | 3 | — |
| [#703 【单测补充】test(domain): 覆盖 MCP 凭据脱敏、验证门禁拆分与执行机器辅助函数](https://github.com/Tencent/LoopForge/pull/703) | 开放 | 11-14 09:12 | 3 · 后端 | 3 | — |
| [#794 Classify terminal model errors from local executors](https://github.com/Tencent/LoopForge/pull/794) | 关闭 | 11-14 19:28 → 11-14 19:45 | 3 · 后端 | 3 / [15:08](https://github.com/Tencent/LoopForge/actions/runs/43590222331/job/132433189149) | — |
| [#796 Propagate model errors from remote runners](https://github.com/Tencent/LoopForge/pull/796) | 关闭 | 11-14 19:36 → 11-14 19:45 | 14 · 后端 | 3 | — |
| [#798 分类处理模型服务错误并恢复工作流执行](https://github.com/Tencent/LoopForge/pull/798) | 开放 | 11-14 19:44 | 49 · 后端/Web/CI/构建 | 24 | — |
| [#810 fix(assistant): keep a live generation through a slow heartbeat](https://github.com/Tencent/LoopForge/pull/810) | 合并 | 11-14 21:06 → 11-15 06:02 | 29 · 后端/文档 | 18 / [15:32](https://github.com/Tencent/LoopForge/actions/runs/43627345132/job/132546099734) | `dd56585` fork |
| [#829 fix: sync recent chat, SOP, and runner fixes](https://github.com/Tencent/LoopForge/pull/829) | 合并 | 11-14 23:31 → 11-15 04:29 | 24 · 后端/Web/CI/构建 | 15 | `500a5e9` fork |
| [#835 Hardening: ensure applyNativeToolSafety always enforces the full disallowed-tools list](https://github.com/Tencent/LoopForge/pull/835) | 开放 | 11-15 01:27 | 2 · 后端 | 15 | — |
| [#875 feat(runner): deliver Windows execution machine lifecycle](https://github.com/Tencent/LoopForge/pull/875) | Draft | 11-15 09:16 | 127 · 后端/CI/构建/文档 | 25 | — |
| [#876 fix(assistant): treat CLI default as unset for non-CodeBuddy engines](https://github.com/Tencent/LoopForge/pull/876) | 开放 | 11-15 09:42 | 2 · 后端 | 3 | — |
| [#877 fix(gitwork): count renamed files once in patch inventory](https://github.com/Tencent/LoopForge/pull/877) | 开放 | 11-15 11:47 | 2 · 后端 | 6 | — |
| [#878 fix(tooling): use a native pinned Gitleaks release](https://github.com/Tencent/LoopForge/pull/878) | 开放 | 11-15 11:47 | 1 · CI/构建 | 3 | — |
| [#879 test(gitwork): accept Git’s canonical workspace path](https://github.com/Tencent/LoopForge/pull/879) | 开放 | 11-15 11:49 | 1 · 后端 | 3 | — |
| [#880 fix(sandbox): skip build notice for prebuilt isolated images](https://github.com/Tencent/LoopForge/pull/880) | 开放 | 11-15 11:52 | 3 · 后端 | 3 | — |
| [#881 refactor(runtime): isolate process and filesystem platform adapters](https://github.com/Tencent/LoopForge/pull/881) | 开放 | 11-15 11:59 | 28 · 后端/CI/构建 | 7 / [15:12](https://github.com/Tencent/LoopForge/actions/runs/43674930865/job/132692876146) | — |
| [#882 feat(runner): enable Windows task runtimes and isolated SSH](https://github.com/Tencent/LoopForge/pull/882) | 开放 | 11-15 12:52 | 99 · 后端/CI/构建 | 7 | — |
| [#883 fix: explain missing MySQL telemetry adapter column](https://github.com/Tencent/LoopForge/pull/883) | 开放 | 11-15 15:54 | 2 · 后端 | 3 | — |
| [#886 ci(local): fix Python 3.8 and Playwright browser runs on Ubuntu 26.04 (#885)](https://github.com/Tencent/LoopForge/pull/886) | 关闭 | 11-15 19:00 → 11-18 15:09 | 2 · CI/构建 | 3 | — |
| [#887 fix(runtime): reject checkpoints with borrowed Git object stores](https://github.com/Tencent/LoopForge/pull/887) | 开放 | 11-15 22:49 | 3 · 后端/文档 | 6 | — |
| [#888 feat(web): unify CLI/model pickers in roles and clarification (#441)](https://github.com/Tencent/LoopForge/pull/888) | 开放 | 11-15 22:55 | 11 · Web/CI/构建/文档 | 9 / [15:47](https://github.com/Tencent/LoopForge/actions/runs/44113945769/job/134146736698) | — |
| [#891 docs(api): declare Project.runEval in the OpenAPI contract](https://github.com/Tencent/LoopForge/pull/891) | 开放 | 11-15 23:51 | 1 · 文档 | 3 | — |
| [#894 refine(sops): tighten built-in role prompts and align wizard presets](https://github.com/Tencent/LoopForge/pull/894) | 开放 | 11-16 00:41 | 3 · 后端/Web | 3 | — |
| [#901 fix(web): wrap long Base URLs in credential confirmation](https://github.com/Tencent/LoopForge/pull/901) | 开放 | 11-16 10:31 | 2 · Web/CI/构建 | 3 | — |
| [#902 fix(assistant): install shared skills on every conversation turn](https://github.com/Tencent/LoopForge/pull/902) | 合并 | 11-16 11:28 → 11-17 18:02 | 7 · 后端/Web | 24 | `1535886` fork |
| [#903 feat(cnb): add native CNB (cnb.cool) repository and PR support (#890)](https://github.com/Tencent/LoopForge/pull/903) | 开放 | 11-16 11:32 | 99 · 后端/Web/CI/构建/文档 | 9 | — |
| [#904 fix(runtime): support bearer auth for custom Claude Code gateways](https://github.com/Tencent/LoopForge/pull/904) | 开放 | 11-16 11:40 | 5 · 后端 | 3 | — |
| [#906 fix(local): register Classic Codex custom roles](https://github.com/Tencent/LoopForge/pull/906) | 开放 | 11-16 12:24 | 8 · Local | 3 | — |
| [#907 fix(schedule): keep oversized cron steps within their range](https://github.com/Tencent/LoopForge/pull/907) | 开放 | 11-16 12:25 | 2 · 后端 | 3 | — |
| [#908 fix(schedule): stop execution when the run claim cannot be saved](https://github.com/Tencent/LoopForge/pull/908) | 开放 | 11-16 12:26 | 2 · 后端 | 3 | — |
| [#909 fix(local): avoid self-matches in public-content validation](https://github.com/Tencent/LoopForge/pull/909) | 开放 | 11-16 12:27 | 1 · Local | 3 | — |
| [#910 fix(stream): retain live log and telemetry deltas on run-control updates](https://github.com/Tencent/LoopForge/pull/910) | 开放 | 11-16 12:28 | 2 · 后端 | 3 | — |
| [#911 fix(local): reject unsafe paths in installer state](https://github.com/Tencent/LoopForge/pull/911) | 开放 | 11-16 12:29 | 2 · Local | 3 | — |
| [#912 fix(schedule): find the next leap-day cron occurrence](https://github.com/Tencent/LoopForge/pull/912) | 开放 | 11-16 12:30 | 2 · 后端 | 3 | — |
| [#913 fix(local): install from a release with workflow assets](https://github.com/Tencent/LoopForge/pull/913) | 开放 | 11-16 12:32 | 2 · Local | 3 | — |
| [#914 fix(gitwork): return merge errors when no merge starts](https://github.com/Tencent/LoopForge/pull/914) | 开放 | 11-16 12:32 | 2 · 后端 | 3 | — |
| [#915 fix(gitwork): complete merges resolved to the current tree](https://github.com/Tencent/LoopForge/pull/915) | 开放 | 11-16 12:33 | 2 · 后端 | 3 | — |
| [#916 fix(insights): normalize source expiry checks to UTC](https://github.com/Tencent/LoopForge/pull/916) | 开放 | 11-16 12:33 | 2 · 后端 | 3 | — |
| [#917 fix(gitwork): preserve literal conflict filenames](https://github.com/Tencent/LoopForge/pull/917) | 开放 | 11-16 12:34 | 2 · 后端 | 6 | — |
| [#918 fix(runs): advance revisions across concurrent control writes](https://github.com/Tencent/LoopForge/pull/918) | 开放 | 11-16 12:34 | 2 · 后端 | 3 | — |
| [#919 fix(gitwork): merge the work branch by its full ref](https://github.com/Tencent/LoopForge/pull/919) | 开放 | 11-16 12:34 | 2 · 后端 | 3 | — |
| [#920 fix(stream): recover live run polling after a database read fails](https://github.com/Tencent/LoopForge/pull/920) | 开放 | 11-16 12:35 | 2 · 后端 | 3 | — |
| [#921 fix(procwatch): bound buffered output from unterminated lines](https://github.com/Tencent/LoopForge/pull/921) | 开放 | 11-16 12:36 | 2 · 后端 | 3 | — |
| [#922 fix(runeval): reject malformed workflow retry counts](https://github.com/Tencent/LoopForge/pull/922) | 开放 | 11-16 12:37 | 2 · 后端 | 3 | — |
| [#923 feat(chat): copy a message turn diagnostic ID](https://github.com/Tencent/LoopForge/pull/923) | 开放 | 11-16 12:38 | 5 · 后端/Web | 5 | — |
| [#924 fix(runtime): honor deadlines while waiting to install a CLI](https://github.com/Tencent/LoopForge/pull/924) | 开放 | 11-16 12:39 | 2 · 后端 | 3 | — |
| [#925 fix(runtime): repair incomplete pinned Claude Code installations](https://github.com/Tencent/LoopForge/pull/925) | 开放 | 11-16 12:40 | 2 · 后端 | 3 | — |
| [#926 fix(web): tint configured monochrome credential icons](https://github.com/Tencent/LoopForge/pull/926) | 开放 | 11-16 12:41 | 3 · Web/CI/构建 | 9 | — |
| [#927 fix(chat): report the CLI that ran each reply](https://github.com/Tencent/LoopForge/pull/927) | 开放 | 11-16 12:50 | 5 · Web | 3 | — |
| [#928 fix(notification): start retry backoff after a failed delivery attempt](https://github.com/Tencent/LoopForge/pull/928) | 开放 | 11-16 12:50 | 2 · 后端 | 3 | — |
| [#929 fix(otel): preserve parent links for protobuf spans](https://github.com/Tencent/LoopForge/pull/929) | 开放 | 11-16 12:51 | 2 · 后端 | 3 | — |
| [#930 fix(otel): retain numeric and boolean trace attributes](https://github.com/Tencent/LoopForge/pull/930) | 开放 | 11-16 12:52 | 3 · 后端 | 3 | — |
| [#931 fix(chat): refresh the model picker after credential changes](https://github.com/Tencent/LoopForge/pull/931) | 开放 | 11-16 12:53 | 2 · Web | 3 | — |
| [#932 fix(mcp): preserve structured results and textless tool errors](https://github.com/Tencent/LoopForge/pull/932) | 开放 | 11-16 12:53 | 2 · 后端 | 3 | — |
| [#933 fix(artifacts): skip ambiguous misplaced output recovery](https://github.com/Tencent/LoopForge/pull/933) | 开放 | 11-16 12:53 | 6 · 后端 | 3 | — |
| [#934 fix(local): replace install state atomically](https://github.com/Tencent/LoopForge/pull/934) | 开放 | 11-16 12:55 | 2 · Local | 3 | — |
| [#935 fix(mcp): carry session and protocol headers after initialization](https://github.com/Tencent/LoopForge/pull/935) | 开放 | 11-16 12:55 | 2 · 后端 | 3 | — |
| [#936 fix(mcp): follow tool catalog pagination cursors](https://github.com/Tencent/LoopForge/pull/936) | 开放 | 11-16 12:55 | 2 · 后端 | 3 | — |
| [#937 fix(runner): distinguish lease loss from a database failure](https://github.com/Tencent/LoopForge/pull/937) | 开放 | 11-16 12:55 | 2 · 后端 | 3 | — |
| [#938 fix(chat): scope workspace changes to their run](https://github.com/Tencent/LoopForge/pull/938) | 开放 | 11-16 12:55 | 3 · Web | 3 | — |
| [#939 fix(mcp): honor caller deadlines during gateway initialization](https://github.com/Tencent/LoopForge/pull/939) | 开放 | 11-16 12:56 | 2 · 后端 | 3 | — |
| [#940 fix(mcp): parse SSE data fields with optional spaces](https://github.com/Tencent/LoopForge/pull/940) | 开放 | 11-16 12:56 | 2 · 后端 | 3 | — |
| [#941 fix(chat): stop refetching unavailable text diffs](https://github.com/Tencent/LoopForge/pull/941) | 开放 | 11-16 12:56 | 2 · Web | 3 | — |
| [#942 fix(stream): keep run log and telemetry cursors from moving backward](https://github.com/Tencent/LoopForge/pull/942) | 开放 | 11-16 12:57 | 2 · 后端 | 3 | — |
| [#943 fix(local): accept Portable-only hosts in the curl installer](https://github.com/Tencent/LoopForge/pull/943) | 开放 | 11-16 12:59 | 2 · Local | 3 | — |
| [#944 fix(runner): split event uploads before control-plane limits](https://github.com/Tencent/LoopForge/pull/944) | 开放 | 11-16 13:00 | 4 · 后端 | 3 | — |
| [#945 fix(runner): retry transient task completion report failures](https://github.com/Tencent/LoopForge/pull/945) | 开放 | 11-16 13:00 | 2 · 后端 | 3 | — |
| [#946 fix(workspace): qualify combined diffs by repository](https://github.com/Tencent/LoopForge/pull/946) | 开放 | 11-16 13:00 | 5 · 后端/Web | 3 | — |
| [#947 feat(workspaces): open conversation workspaces by sandbox with agent-owned Git](https://github.com/Tencent/LoopForge/pull/947) | 合并 | 11-16 14:58 → 11-16 19:25 | 103 · 后端/Web/文档 | 15 | `777f16a` fork |
| [#948 fix(ci): keep Python 3.8 and Playwright working on Ubuntu 26.04 runners](https://github.com/Tencent/LoopForge/pull/948) | 合并 | 11-16 15:36 → 11-16 15:54 | 2 · CI/构建 | 3 | `00d4f17` fork |
| [#952 fix(local): stop installing Codex agent roles the runtime never dispatches](https://github.com/Tencent/LoopForge/pull/952) | 开放 | 11-16 17:04 | 8 · Local | 3 | — |
| [#953 refactor(runtime): remove workspace instruction masks and script helper](https://github.com/Tencent/LoopForge/pull/953) | 合并 | 11-16 17:14 → 11-16 19:00 | 44 · 后端/文档 | 6 | `c1fa0f7` fork |
| [#955 feat(engine): make the conversation workspace idle TTL configurable](https://github.com/Tencent/LoopForge/pull/955) | 开放 | 11-16 17:40 | 3 · 后端/文档 | 3 | — |
| [#956 feat(runtime): report when a command waits for an execution slot](https://github.com/Tencent/LoopForge/pull/956) | 开放 | 11-16 17:45 | 2 · 后端 | 3 | — |
| [#958 feat(schedule): execution history, session mode, conversations kept per job and row actions for scheduled tasks](https://github.com/Tencent/LoopForge/pull/958) | 合并 | 11-16 18:32 → 11-17 17:34 | 44 · 后端/Web/CI/构建/文档 | 24 | `dfea6da` fork |
| [#960 feat(chatbot): remove the one-click demo starter from the home page](https://github.com/Tencent/LoopForge/pull/960) | 合并 | 11-16 19:25 → 11-16 20:29 | 6 · Web/CI/构建 | 6 | `3e02e40` fork |
| [#961 chore(web): remove the page feedback widget from the open source build](https://github.com/Tencent/LoopForge/pull/961) | 开放 | 11-16 19:30 | 18 · 后端/Web/CI/构建/文档 | 3 | — |
| [#962 fix(delivery): do not mark a work item merged without a delivery MR](https://github.com/Tencent/LoopForge/pull/962) | 开放 | 11-16 19:46 | 3 · 后端 | 3 | — |
| [#964 fix(catalog): derive project role for template agent nodes missing from Agents list](https://github.com/Tencent/LoopForge/pull/964) | 开放 | 11-16 21:19 | 2 · 后端 | 3 | — |
| [#965 feat(issues): create requirements directly and clarify them in a chat](https://github.com/Tencent/LoopForge/pull/965) | 合并 | 11-16 21:31 → 11-17 00:57 | 27 · 后端/Web/CI/构建 | 15 | `9abc8f3` fork |
| [#966 feat(schedule): run AI jobs with the saver's key, fall back to the shared key](https://github.com/Tencent/LoopForge/pull/966) | 合并 | 11-16 21:37 → 11-17 02:09 | 23 · 后端/Web/CI/构建/文档 | 21 | `1e50be7` fork |
| [#967 feat(sop): add an MR node to the built-in code delivery SOPs](https://github.com/Tencent/LoopForge/pull/967) | 合并 | 11-16 22:36 → 11-17 02:09 | 23 · 后端/Web/文档 | 18 | `69e4bf0` fork |
| [#968 feat(web): restore reclaimed workspaces from run details](https://github.com/Tencent/LoopForge/pull/968) | 合并 | 11-16 22:46 → 11-17 00:10 | 10 · 后端/Web/CI/构建 | 6 | `86b0ee0` fork |
| [#970 fix(deploy): never fall back to plain HTTP for an installer served over HTTPS](https://github.com/Tencent/LoopForge/pull/970) | 关闭 | 11-17 00:14 → 11-17 00:19 | 6 · 后端/CI/构建/文档 | 3 | — |
| [#971 feat(runtime): add Hermes Agent as an agent CLI executor](https://github.com/Tencent/LoopForge/pull/971) | 关闭 | 11-17 00:14 → 11-17 00:19 | 22 · 后端/Web/文档 | 3 | — |
| [#972 feat(runner): add CLI authentication and maintenance controls](https://github.com/Tencent/LoopForge/pull/972) | 合并 | 11-17 00:57 → 11-17 19:46 | 81 · 后端/Web/CI/构建/文档 | 33 / [15:20](https://github.com/Tencent/LoopForge/actions/runs/43939242187/job/133551993401) | `236f1c5` fork |
| [#973 feat(assistant): request approval for external MCP writes](https://github.com/Tencent/LoopForge/pull/973) | 合并 | 11-17 01:07 → 11-17 19:47 | 27 · 后端/Web/文档 | 15 / [16:01](https://github.com/Tencent/LoopForge/actions/runs/44072867778/job/134003240808) | `eb2b8e8` fork |
| [#974 feat(workspaces): allow binding project SOP workspaces via workspaceRunId](https://github.com/Tencent/LoopForge/pull/974) | 合并 | 11-17 01:20 → 11-17 19:47 | 8 · 后端/文档 | 15 | `7cc5911` fork |
| [#975 perf(gitwork): find the nearest patch baseline without probing every remote ref](https://github.com/Tencent/LoopForge/pull/975) | 开放 | 11-17 01:22 | 2 · 后端 | 3 | — |
| [#976 fix: redact legacy repository credentials and harden workspace binding](https://github.com/Tencent/LoopForge/pull/976) | 开放 | 11-17 01:36 | 5 · 后端 | 3 | — |
| [#977 feat(web): complete English coverage for the project, run, runner and requirement screens](https://github.com/Tencent/LoopForge/pull/977) | 开放 | 11-17 08:44 | 3 · Web/CI/构建 | 3 | — |
| [#978 feat(chatbot): remember the chat model and approval mode](https://github.com/Tencent/LoopForge/pull/978) | 合并 | 11-17 13:00 → 11-17 18:36 | 6 · 后端/Web | 9 / [15:22](https://github.com/Tencent/LoopForge/actions/runs/44046283517/job/133912653740) | `b30f400` fork |
| [#979 feat(metrics): expose Prometheus metrics endpoint](https://github.com/Tencent/LoopForge/pull/979) | 开放 | 11-17 14:34 | 3 · 后端 | 3 | — |
| [#980 feat(workflow): editing lease to stop two editors overwriting each other](https://github.com/Tencent/LoopForge/pull/980) | 开放 | 11-17 14:41 | 7 · 后端 | 3 | — |
| [#981 fix(chatbot): drop the filename bar under chat image attachments](https://github.com/Tencent/LoopForge/pull/981) | 合并 | 11-17 14:45 → 11-17 18:36 | 2 · Web | 6 | `9909482` fork |
| [#982 feat(chatbot): show the model a subagent ran on](https://github.com/Tencent/LoopForge/pull/982) | 合并 | 11-17 14:50 → 11-17 18:36 | 5 · 后端/Web | 9 | `106ab91` fork |
| [#983 chore(web): remove the "Browse public MCP" link from the capability page](https://github.com/Tencent/LoopForge/pull/983) | 关闭 | 11-17 14:51 → 11-17 15:49 | 2 · Web | 3 | — |
| [#984 解耦 SOP 编排与执行环境管理](https://github.com/Tencent/LoopForge/pull/984) | 开放 | 11-17 15:15 | 81 · 后端/文档 | 3 | — |
| [#985 增加独立部署的沙箱管理器](https://github.com/Tencent/LoopForge/pull/985) | Draft | 11-17 15:15 | 83 · 后端/文档 | 3 | — |
| [#989 fix(workspace): restore snapshots over empty dirs and preserve git-tracked excluded paths](https://github.com/Tencent/LoopForge/pull/989) | 合并 | 11-17 15:32 → 11-17 16:40 | 15 · 后端 | 6 | `4bcd9fb` fork |
| [#990 Unify the chat workbench and application shell](https://github.com/Tencent/LoopForge/pull/990) | 开放 | 11-17 15:33 | 65 · 后端/Web/CI/构建/文档 | 12 | — |
| [#991 feat(sop): rebuild the SOP orchestrator on the assistant engine and chatbot](https://github.com/Tencent/LoopForge/pull/991) | 开放 | 11-17 16:22 | 33 · 后端/Web/CI/构建/文档 | 9 | — |
| [#992 fix(engine): 产物超过 64KB 时节点终态写入失败，run 永久停在运行中](https://github.com/Tencent/LoopForge/pull/992) | 开放 | 11-17 16:52 | 4 · 后端 | 3 | — |
| [#993 fix(runner): 预检失败不再上报 exit 0，避免日志出现「… (exit 0)」](https://github.com/Tencent/LoopForge/pull/993) | 开放 | 11-17 17:04 | 3 · 后端 | 3 | — |
| [#995 feat: support direct OpenAI Responses API for chat and clarification](https://github.com/Tencent/LoopForge/pull/995) | 开放 | 11-17 17:22 | 41 · 后端/Web/文档 | 3 | — |
| [#996 fix(assistant): pass oversized chat prompts on stdin instead of argv](https://github.com/Tencent/LoopForge/pull/996) | 合并 | 11-17 17:25 → 11-17 18:35 | 7 · 后端 | 3 | `5706684` fork |
| [#997 fix(runner): 领取任务时不再把整行排进排序缓冲，避免大 payload 下的 1038](https://github.com/Tencent/LoopForge/pull/997) | 开放 | 11-17 18:05 | 2 · 后端 | 3 | — |
| [#998 fix(engine): 通用模板 SOP 的节点重试报「agent … 不存在」，重连校验漏了模板角色回落](https://github.com/Tencent/LoopForge/pull/998) | 开放 | 11-17 18:28 | 2 · 后端 | 3 | — |
| [#999 fix(assistant): resolve the Claude Code model catalog](https://github.com/Tencent/LoopForge/pull/999) | 开放 | 11-17 19:04 | 3 · 后端 | 3 | — |
| [#1000 fix(chatbot): drop the CLI default model from the chat model picker](https://github.com/Tencent/LoopForge/pull/1000) | 开放 | 11-17 19:05 | 54 · 后端/Web/CI/构建/文档 | 3 / [15:59](https://github.com/Tencent/LoopForge/actions/runs/44070842619/job/133996353258) | — |
| [#1001 企业 IM 接入 R1：企微回归基线 + 飞书私聊跑通（方案文档与基线测试）](https://github.com/Tencent/LoopForge/pull/1001) | Draft | 11-17 19:51 | 4 · 后端/文档 | 3 | — |
| [#1002 chore(release): prepare v0.4.0](https://github.com/Tencent/LoopForge/pull/1002) | 合并 | 11-17 20:13 → 11-17 21:34 | 3 · CI/构建/文档 | 9 / [16:39](https://github.com/Tencent/LoopForge/actions/runs/44081200209/job/134031949036) | `6b7e9f4` fork |
| [#1003 test(httpapi): keep the CLI reload install test independent of preinstalled dsh](https://github.com/Tencent/LoopForge/pull/1003) | 开放 | 11-17 20:29 | 2 · 后端 | 3 | — |
| [#1004 fix(chatbot): keep queued messages when switching conversations](https://github.com/Tencent/LoopForge/pull/1004) | 合并 | 11-17 20:35 → 11-17 20:55 | 2 · Web | 6 / [15:39](https://github.com/Tencent/LoopForge/actions/runs/44084592530/job/134043587316) | `e270ef0` fork |
| [#1005 fix(assistant): let a turn keep the issue it linked from its @-mention](https://github.com/Tencent/LoopForge/pull/1005) | 合并 | 11-17 20:38 → 11-17 20:59 | 5 · 后端/文档 | 6 | `8f0eeea` fork |
| [#1006 fix(schedule): support reasoning effort for AI scheduled tasks](https://github.com/Tencent/LoopForge/pull/1006) | 开放 | 11-17 20:41 | 14 · 后端/Web/CI/构建/文档 | 3 | — |
| [#1007 fix(local): resolve the DSH agent working directory](https://github.com/Tencent/LoopForge/pull/1007) | 开放 | 11-17 21:05 | 2 · Local | 3 | — |
| [#1008 fix(api): require a model for Pi, OpenCode and Kimi roles](https://github.com/Tencent/LoopForge/pull/1008) | 开放 | 11-17 21:11 | 3 · 后端 | 3 | — |
| [#1009 feat(api): paginate persisted SOP run events](https://github.com/Tencent/LoopForge/pull/1009) | 开放 | 11-17 21:11 | 5 · 后端/文档 | 3 | — |
| [#1010 feat(installer): replay the SOP 「发起 MR」 migration on install and pull](https://github.com/Tencent/LoopForge/pull/1010) | 开放 | 11-17 21:29 | 6 · 后端/CI/构建/文档 | 3 | — |
| [#1011 test(chatbot): wait for the issue draft to be saved before reading it](https://github.com/Tencent/LoopForge/pull/1011) | 合并 | 11-17 21:29 → 11-17 21:31 | 1 · Web | 3 / [17:12](https://github.com/Tencent/LoopForge/actions/runs/44093221031/job/134073533542) | `92e995e` fork |
| [#1012 fix(install): mount the Docker VM socket on macOS](https://github.com/Tencent/LoopForge/pull/1012) | 开放 | 11-17 21:43 | 3 · CI/构建/文档 | 3 | — |
| [#1013 test(e2e): expect the three home starters left after #960](https://github.com/Tencent/LoopForge/pull/1013) | 合并 | 11-17 21:48 → 11-17 21:49 | 1 · CI/构建 | 3 | `e89cda9` fork |
| [#1014 test(e2e): expect three home starters in first-run](https://github.com/Tencent/LoopForge/pull/1014) | 关闭 | 11-17 21:48 → 11-17 21:51 | 1 · CI/构建 | 3 / [16:48](https://github.com/Tencent/LoopForge/actions/runs/44096448219/job/134084702539) | — |
| [#1015 feat(git): per-host HTTPS Git credentials for workspace agents](https://github.com/Tencent/LoopForge/pull/1015) | 合并 | 11-17 21:56 → 11-19 00:03 | 195 · 后端/Web/CI/构建/文档 | 39 / [18:16](https://github.com/Tencent/LoopForge/actions/runs/44224015093/job/134532613147) | `0541f88` 上游 |
| [#1016 feat(runner): support Antigravity login and harden credential refresh](https://github.com/Tencent/LoopForge/pull/1016) | 合并 | 11-17 21:58 → 11-19 00:03 | 61 · 后端/Web/CI/构建/文档 | 27 / [15:26](https://github.com/Tencent/LoopForge/actions/runs/44257282015/job/134641901829) | `218db9b` 上游 |
| [#1017 fix(workspace): recover a conversation after its instance dies](https://github.com/Tencent/LoopForge/pull/1017) | 开放 | 11-17 23:24 | 5 · 后端 | 3 | — |
| [#1018 chore(deps): bump golang.org/x/crypto from 0.58.0 to 0.59.0](https://github.com/Tencent/LoopForge/pull/1018) | 开放 | 11-17 23:35 | 2 · 后端 | 6 / [18:02](https://github.com/Tencent/LoopForge/actions/runs/44343192013/job/134938441104) | — |
| [#1019 chore(deps): bump github.com/klauspost/compress from 1.20.3 to 1.20.4](https://github.com/Tencent/LoopForge/pull/1019) | 开放 | 11-17 23:35 | 2 · 后端 | 6 / [19:00](https://github.com/Tencent/LoopForge/actions/runs/44343195465/job/134938451064) | — |
| [#1020 fix(assistant): report disk-full execution failures](https://github.com/Tencent/LoopForge/pull/1020) | 开放 | 11-18 00:13 | 4 · 后端 | 5 | — |
| [#1021 Coordinate Runner OAuth refresh and reject sandbox credential replacement](https://github.com/Tencent/LoopForge/pull/1021) | 合并 · feature/antigravity-account-login | 11-18 00:32 → 11-18 01:45 | 27 · 后端/CI/构建/文档 | 9 / [16:57](https://github.com/Tencent/LoopForge/actions/runs/44127019592/job/134193480878) | `09cff36` |
| [#1023 fix(assistant): explain unsupported OpenCode variants](https://github.com/Tencent/LoopForge/pull/1023) | 开放 | 11-18 01:17 | 3 · 后端 | 3 | — |
| [#1024 docs(api): index assistant and shared schedule routes](https://github.com/Tencent/LoopForge/pull/1024) | 开放 | 11-18 01:30 | 1 · 文档 | 3 | — |
| [#1025 fix(sandbox): explain missing isolated runtime images](https://github.com/Tencent/LoopForge/pull/1025) | 开放 | 11-18 01:37 | 2 · 后端 | 3 | — |
| [#1026 fix(models): preserve and configure per-model reasoning efforts](https://github.com/Tencent/LoopForge/pull/1026) | 合并 | 11-18 01:49 → 11-19 01:17 | 64 · 后端/Web/CI/构建/文档 | 39 / [17:18](https://github.com/Tencent/LoopForge/actions/runs/44344726704/job/134943928802) | `5c37c33` 上游 |
| [#1027 test(chatbot): use jsdom storage on Node 26](https://github.com/Tencent/LoopForge/pull/1027) | 开放 | 11-18 09:14 | 1 · Web | 3 | — |
| [#1028 feat(local): preview install and update changes with --dry-run](https://github.com/Tencent/LoopForge/pull/1028) | 开放 | 11-18 09:47 | 5 · Local/文档 | 3 | — |
| [#1030 feat(i18n): add comprehensive Korean localization support](https://github.com/Tencent/LoopForge/pull/1030) | 开放 | 11-18 10:36 | 19 · 后端/Web | 30 | — |
| [#1031 build: auto restart mysql](https://github.com/Tencent/LoopForge/pull/1031) | 开放 | 11-18 10:58 | 1 · CI/构建 | 3 / [15:37](https://github.com/Tencent/LoopForge/actions/runs/44227114931/job/134552716254) | — |
| [#1032 feat(runtime): custom run credentials for self-hosted deployments](https://github.com/Tencent/LoopForge/pull/1032) | 合并 | 11-18 11:23 → 11-18 16:36 | 11 · 后端 | 6 / [17:00](https://github.com/Tencent/LoopForge/actions/runs/44265090394/job/134667518476) | `48cdf0c` 上游 |
| [#1033 feat(mcp): credential-authenticated MCP servers](https://github.com/Tencent/LoopForge/pull/1033) | 合并 | 11-18 11:36 → 11-18 23:04 | 9 · 后端 | 21 / [19:43](https://github.com/Tencent/LoopForge/actions/runs/44273652059/job/134696204612) | `291c14d` 上游 |
| [#1034 fix(runtime): never write "args": null into the MCP config](https://github.com/Tencent/LoopForge/pull/1034) | 开放 | 11-18 13:04 | 2 · 后端 | 3 | — |
| [#1035 feat(cnb): support CNB repositories and pull requests](https://github.com/Tencent/LoopForge/pull/1035) | Draft | 11-18 14:21 | 99 · 后端/Web/CI/构建/文档 | 3 | — |
| [#1036 fix(web): restore SOP trigger card layout](https://github.com/Tencent/LoopForge/pull/1036) | 开放 | 11-18 14:56 | 1 · Web | 3 | — |
| [#1037 fix(assistant): list Claude Code models for the claude-code executor](https://github.com/Tencent/LoopForge/pull/1037) | 开放 | 11-18 15:08 | 2 · 后端 | 3 | — |
| [#1042 feat(assistant): add send preferences and preserve the CLI during steering](https://github.com/Tencent/LoopForge/pull/1042) | 合并 | 11-18 16:38 → 11-19 00:03 | 20 · 后端/Web/文档 | 15 / [16:22](https://github.com/Tencent/LoopForge/actions/runs/44268796022/job/134679877460) | `01ef8dd` 上游 |
| [#1043 feat(engine): let nodes request platform credentials as env vars](https://github.com/Tencent/LoopForge/pull/1043) | 合并 | 11-18 16:55 → 11-18 23:04 | 7 · 后端 | 6 / [16:58](https://github.com/Tencent/LoopForge/actions/runs/44271113032/job/134687663398) | `f8dcd1e` 上游 |
| [#1045 fix(gitwork): exclude diff3 ancestors from conflict sides](https://github.com/Tencent/LoopForge/pull/1045) | 开放 | 11-18 17:36 | 2 · 后端 | 3 | — |
| [#1046 feat(workspaces): enable personal conversation workspaces](https://github.com/Tencent/LoopForge/pull/1046) | 合并 | 11-18 18:03 → 11-19 01:17 | 31 · 后端/Web/文档 | 9 | `6f7edc7` 上游 |
| [#1047 fix(schedule): expand numeric cron steps through the field range](https://github.com/Tencent/LoopForge/pull/1047) | 开放 | 11-18 18:16 | 2 · 后端 | 3 | — |
| [#1048 fix(tracker): prefer open pull requests for reused branches](https://github.com/Tencent/LoopForge/pull/1048) | 开放 | 11-18 18:16 | 2 · 后端 | 3 | — |
| [#1049 fix(otel): resolve run context from span attributes](https://github.com/Tencent/LoopForge/pull/1049) | 开放 | 11-18 18:16 | 2 · 后端 | 3 | — |
| [#1050 fix(preview): avoid detecting vitest as a frontend build](https://github.com/Tencent/LoopForge/pull/1050) | 开放 | 11-18 18:16 | 2 · 后端 | 3 | — |
| [#1051 fix(runtime): normalize Gemini service URLs consistently](https://github.com/Tencent/LoopForge/pull/1051) | 开放 | 11-18 18:19 | 5 · 后端 | 3 | — |
| [#1052 fix(tracker): retain local SOP state during issue refreshes](https://github.com/Tencent/LoopForge/pull/1052) | 开放 | 11-18 18:20 | 3 · 后端 | 3 | — |
| [#1053 fix(preview): serve nested directory indexes and relative assets](https://github.com/Tencent/LoopForge/pull/1053) | 开放 | 11-18 18:22 | 2 · 后端 | 3 | — |
| [#1054 fix(runner): reject incomplete control-plane responses](https://github.com/Tencent/LoopForge/pull/1054) | 开放 | 11-18 18:27 | 2 · 后端 | 3 | — |
| [#1055 perf(engine): index workflow edges for reachability checks](https://github.com/Tencent/LoopForge/pull/1055) | 开放 | 11-18 18:33 | 2 · 后端 | 3 | — |
| [#1056 fix(domain): preserve case-sensitive Git branch identities](https://github.com/Tencent/LoopForge/pull/1056) | 开放 | 11-18 18:34 | 2 · 后端 | 3 | — |
| [#1057 fix(analytics): use nearest-rank delivery execution percentiles](https://github.com/Tencent/LoopForge/pull/1057) | 开放 | 11-18 18:34 | 2 · 后端 | 3 | — |
| [#1058 fix(runtime): parse skill frontmatter as YAML](https://github.com/Tencent/LoopForge/pull/1058) | 开放 | 11-18 18:35 | 2 · 后端 | 3 | — |
| [#1059 fix(runeval): count tool invocations once across lifecycle events](https://github.com/Tencent/LoopForge/pull/1059) | 开放 | 11-18 18:36 | 2 · 后端 | 3 | — |
| [#1060 fix(runeval): retain final telemetry in bounded evidence digest](https://github.com/Tencent/LoopForge/pull/1060) | 开放 | 11-18 18:37 | 2 · 后端 | 3 | — |
| [#1061 fix(schedule): search decoded execution history fields](https://github.com/Tencent/LoopForge/pull/1061) | 开放 | 11-18 18:37 | 2 · 后端 | 3 | — |
| [#1062 fix(schedule): recover only predecessor job occurrences](https://github.com/Tencent/LoopForge/pull/1062) | 开放 | 11-18 18:38 | 2 · 后端 | 3 | — |
| [#1063 fix(runtime): reject checkpoints with unreadable Git indexes](https://github.com/Tencent/LoopForge/pull/1063) | 开放 | 11-18 18:38 | 3 · 后端 | 3 | — |
| [#1064 fix(knowledge): fill candidate pages after filtering and retries](https://github.com/Tencent/LoopForge/pull/1064) | 开放 | 11-18 18:38 | 2 · 后端 | 6 | — |
| [#1065 fix(engine): retain token usage cursor high-water marks](https://github.com/Tencent/LoopForge/pull/1065) | 开放 | 11-18 18:38 | 2 · 后端 | 3 | — |
| [#1066 feat(platform): add Forgejo repository and PR delivery](https://github.com/Tencent/LoopForge/pull/1066) | 开放 | 11-18 19:32 | 76 · 后端/Web/CI/构建/文档 | 9 | — |
| [#1067 feat(mcp): add AI/ML API to the bundled catalog](https://github.com/Tencent/LoopForge/pull/1067) | 开放 | 11-18 21:33 | 2 · 后端 | 3 | — |
| [#1068 docs: mark LoopForge Local as legacy](https://github.com/Tencent/LoopForge/pull/1068) | 合并 | 11-18 23:08 → 11-19 01:52 | 5 · Local/文档 | 3 | `451d38c` 上游 |
| [#1070 feat(engine): bind approvals to artifacts and gate verification nodes](https://github.com/Tencent/LoopForge/pull/1070) | 开放 | 11-18 23:22 | 16 · 后端/文档 | 3 | — |
| [#1071 build: optional npm and Go module mirrors for source builds](https://github.com/Tencent/LoopForge/pull/1071) | 开放 | 11-18 23:22 | 4 · Local/CI/构建/文档 | 3 | — |
| [#1072 feat(market): let market items declare credential authentication](https://github.com/Tencent/LoopForge/pull/1072) | 合并 | 11-18 23:25 → 11-19 20:53 | 2 · 后端 | 3 | `6c4226f` 上游 |
| [#1073 docs(api): index the AI conversation domain and its v0.4.0 endpoints](https://github.com/Tencent/LoopForge/pull/1073) | 开放 | 11-19 00:05 | 1 · 文档 | 3 | — |
| [#1077 fix(runtime): provision missing CLIs in Docker SOPs](https://github.com/Tencent/LoopForge/pull/1077) | 合并 | 11-19 02:01 → 11-19 20:32 | 10 · 后端/Web/CI/构建/文档 | 9 / [18:21](https://github.com/Tencent/LoopForge/actions/runs/44496444215/job/135464705656) | `684e79d` 上游 |
| [#1080 fix(schedule): bound AI task runs and wait for SOP outcomes](https://github.com/Tencent/LoopForge/pull/1080) | 开放 | 11-19 10:07 | 25 · 后端/Web/CI/构建/文档 | 9 | — |
| [#1082 fix(models): align CLI catalogs and reasoning choices](https://github.com/Tencent/LoopForge/pull/1082) | 合并 | 11-19 11:10 → 11-19 21:36 | 69 · 后端/Web/CI/构建/文档 | 21 | `6f606cf` 上游 |
| [#1083 fix(chat): respect participant model catalogs and Codex providers](https://github.com/Tencent/LoopForge/pull/1083) | 开放 | 11-19 11:57 | 7 · 后端/Web | 3 | — |
| [#1088 fix(runner): preserve MCP connectivity and native Codex workspace cwd](https://github.com/Tencent/LoopForge/pull/1088) | 开放 | 11-19 13:53 | 3 · 后端 | 3 | — |
| [#1089 fix(chat): run Codex threads in the native sandbox workspace](https://github.com/Tencent/LoopForge/pull/1089) | 开放 | 11-19 14:53 | 2 · 后端 | 3 | — |
| [#1091 feat(verification): bind approvals to artifacts and enforce verification gates](https://github.com/Tencent/LoopForge/pull/1091) | 开放 | 11-19 15:35 | 47 · 后端/Local/CI/构建/文档 | 3 | — |
| [#1094 docs(readme): describe supervision as human on the loop](https://github.com/Tencent/LoopForge/pull/1094) | 合并 | 11-19 17:10 → 11-19 17:25 | 3 · 文档 | 6 / [17:31](https://github.com/Tencent/LoopForge/actions/runs/44494021215/job/135456507495) | `a664453` 上游 |
| [#1095 fix(mcp): explain platform tool argument envelopes](https://github.com/Tencent/LoopForge/pull/1095) | 开放 | 11-19 18:07 | 5 · 后端 | 3 | — |
| [#1096 feat(chat): refresh workspace repository context on demand](https://github.com/Tencent/LoopForge/pull/1096) | 合并 | 11-19 18:11 → 11-19 21:06 | 11 · 后端 | 9 / [17:28](https://github.com/Tencent/LoopForge/actions/runs/44513898507/job/135524825568) | `4175c96` 上游 |
| [#1097 fix(assistant): preserve committed terminal diagnostic outcome](https://github.com/Tencent/LoopForge/pull/1097) | 开放 | 11-19 18:27 | 5 · 后端 | 6 | — |
| [#1098 fix(flow): keep the canvas viewport usable when coordinates go non-finite](https://github.com/Tencent/LoopForge/pull/1098) | 开放 | 11-19 18:38 | 4 · Web/CI/构建 | 3 | — |
| [#1099 feat(flow): make designer cards resizable with a detail view](https://github.com/Tencent/LoopForge/pull/1099) | 开放 | 11-19 18:38 | 7 · 后端/Web/CI/构建 | 3 | — |
| [#1100 fix(assistant): preserve conversation process lifecycle evidence](https://github.com/Tencent/LoopForge/pull/1100) | 开放 | 11-19 18:47 | 6 · 后端 | 3 | — |
| [#1103 fix(local): require ripgrep for the forbidden-content scans](https://github.com/Tencent/LoopForge/pull/1103) | Draft | 11-19 19:58 | 3 · Local/CI/构建 | 3 | — |
| [#1104 feat: bind knowledge MCP services to workflow nodes and chat sessions](https://github.com/Tencent/LoopForge/pull/1104) | 开放 | 11-19 20:15 | 60 · 后端/Web/Local/CI/构建/文档 | 0 | — |
| [#1106 fix(platform): tell the model which wrapper a tool argument goes in](https://github.com/Tencent/LoopForge/pull/1106) | 开放 | 11-19 20:57 | 6 · 后端/Local/CI/构建/文档 | 3 | — |
| [#1108 fix(runner): install the Linux Docker helper on macOS](https://github.com/Tencent/LoopForge/pull/1108) | 开放 | 11-19 22:25 | 8 · 后端/CI/构建/文档 | 6 / [15:39](https://github.com/Tencent/LoopForge/actions/runs/44543533860/job/135627892016) | — |
| [#1109 fix(local): correct DSH skill invocation guidance](https://github.com/Tencent/LoopForge/pull/1109) | 开放 | 11-19 23:37 | 10 · Local/文档 | 3 | — |
| [#1110 fix(compose): expose the worker concurrency limit](https://github.com/Tencent/LoopForge/pull/1110) | 开放 | 11-19 23:47 | 3 · CI/构建/文档 | 3 | — |

</details>

<details>
<summary>Git 引用补齐的 284 个编号</summary>

GitHub 列表缺少这些条目的生命周期字段。标题来自 main 合并记录或 PR head 提交。日期为 head 提交时间，仅用于 Git 筛查。

| PR | 最近 head 提交（北京时间） | 路径数与范围 | Actions runs / 慢 job | main 对应提交 | Git 标题 |
| --- | --- | --- | --- | --- | --- |
| #115 | 11-09 17:17 | 12 · Local/CI/构建/文档 | 3 | `9bc2058` fork | feat: fail pull requests when coverage or a platform job drops |
| #191 | 11-12 14:45 | 5 · 后端 | 0 | `91f3fa5` fork | fix(runtime): clip tool messages and executor output on rune boundaries |
| #220 | 11-14 04:40 | 5 · 后端/文档 | 0 | — | Merge main into host-local clone URL rejection |
| #222 | 11-12 23:52 | 4 · Web/CI/构建 | 0 | `8d40b94` fork | [Loopforge 自动化] fix(web): resolve run machine labels without opening settings (refs #198) |
| #224 | 11-12 23:27 | 9 · 后端/Web | 0 | `d664330` fork | fix(runners): apply one rule to choosing the default machine |
| #225 | 11-13 19:32 | 6 · Local/CI/构建/文档 | 3 | `47261cc` fork | fix(install): reapply schema extensions when upgrading |
| #230 | 11-13 00:11 | 2 · 后端 | 0 | `f679fd0` fork | refactor(repo): drop unused analytics versions field and artifactHit type |
| #233 | 11-13 00:15 | 2 · Web/CI/构建 | 0 | `bbe979d` fork | fix(web): show the CodeBuddy network environment on runner cards (refs #198) |
| #235 | 11-13 00:23 | 2 · 后端 | 0 | `c4388b4` fork | refactor(httpapi): share JSON fence stripping and object parsing in stream.go |
| #237 | 11-13 01:22 | 5 · 后端 | 0 | — | test(httpapi): assert draft warnings list referencing assets after helper move |
| #239 | 11-13 01:54 | 2 · 后端 | 0 | `ddd6d88` fork | 【架构优化】合并 notification.ProcessOne 中重复的等待节点过期判断 |
| #241 | 11-13 02:13 | 3 · 后端 | 0 | `7adeef3` fork | 【架构优化】合并 gongfeng OAuth 中重复的 HTTP 请求与限长读取逻辑 |
| #247 | 11-13 02:20 | 3 · 后端 | 0 | — | test(mcpclient): cover caller context propagation and non-fatal init failure |
| #251 | 11-13 02:19 | 2 · 后端 | 0 | `b79dcb8` fork | refactor(cronexpr): 用字段规格表合并 Parse 中五段重复的解析与错误包装 |
| #252 | 11-13 02:19 | 2 · 后端 | 0 | — | refactor(cronexpr): table-driven Parse and extract parsePart |
| #253 | 11-13 02:25 | 4 · 后端 | 0 | `812ace2` fork | 【架构优化】合并 auth 中重复的 RIO 签名 GET 请求构造 |
| #254 | 11-13 02:19 | 2 · 后端 | 0 | — | refactor(cronexpr): table-driven Parse and split parseField |
| #256 | 11-13 02:25 | 4 · 后端 | 0 | — | test(auth): cover signed GET wiring in ExchangeCode and FetchPhoto |
| #257 | 11-13 02:19 | 2 · 后端 | 0 | — | refactor(cronexpr): table-driven Parse and extract parseRange |
| #259 | 11-13 02:20 | 2 · 后端 | 0 | `8bfa2e4` fork | refactor(runeval): extract elapsedMs and orderedNodeIDs from BuildFacts |
| #262 | 11-13 02:35 | 3 · 后端 | 0 | `2adda62` fork | 【架构优化】拆分 managedcli.updateLocked 中的版本闸门判断与候选安装校验 |
| #263 | 11-13 02:23 | 2 · 后端 | 0 | — | refactor(gatemcp): split decodeResult validation and share schema limits |
| #265 | 11-14 04:46 | 16 · 后端/Web | 0 | — | Merge main into trace ID copy |
| #266 | 11-13 02:32 | 2 · 后端 | 0 | `3768251` fork | refactor(httpapi): use builtin max instead of maxInt64/maxFloat/maxInt in space_lanes |
| #268 | 11-13 02:35 | 3 · 后端 | 0 | `449441d` fork | refactor(preview): share &lt;head&gt; insertion helpers across HTML injectors |
| #270 | 11-13 02:43 | 2 · 后端 | 0 | `5c07611` fork | refactor(procwatch): extract stop classification and failure message from Run |
| #272 | 11-13 02:55 | 3 · 后端 | 0 | `0b9a3e6` fork | refactor(knowledge): use slices.Contains for slice membership checks (#271) |
| #274 | 11-13 03:04 | 2 · 后端 | 0 | `73ee73a` fork | refactor(botinteraction): extract snapshotNode for workflow snapshot lookups |
| #277 | 11-13 03:13 | 2 · 后端 | 0 | `fb28569` fork | refactor(tracker): 合并重复的 Authorization 请求头判断 (#276) |
| #279 | 11-13 03:27 | 3 · 后端 | 0 | `f3665ac` fork | refactor(httpapi): extract trailingSlashEntity and use slices.Contains |
| #281 | 11-13 03:34 | 2 · 后端 | 0 | `62d1eca` fork | refactor(engine): drop custom max shadowing the builtin |
| #283 | 11-13 03:53 | 2 · 后端 | 0 | — | test(gitwork): cover repaired-conflict completion and fetch failure in mergeOriginBranch |
| #285 | 11-13 04:02 | 3 · 后端 | 0 | `bcfda66` fork | 【架构优化】拆分 mcpserver.ToolInputSchema 的契约路由查找与 query 参数收集并删除转发函数 |
| #288 | 11-13 04:03 | 2 · 后端 | 0 | — | refactor(observability): split HookEvent lifecycle mapping, hook execution summary and attributes |
| #290 | 11-13 04:23 | 2 · 后端 | 0 | `9bdb802` fork | 【架构优化】合并 tracker/github.go 中重复的 PR 已合并判断与 #编号 引用拼接 |
| #292 | 11-13 04:24 | 2 · 后端 | 0 | `053f87f` fork | refactor(schedule): split eligible-issue filter and repo-block check out of execute |
| #294 | 11-13 04:35 | 2 · 后端 | 0 | `d110053` fork | refactor(runner): replace hasCredentialBinding with slices.Contains |
| #296 | 11-13 04:46 | 2 · 后端 | 0 | `84abc3b` fork | refactor(engine): 合并 hasWaiting/hasFailed/cancelledNode 重复的节点状态查找 |
| #298 | 11-13 04:56 | 2 · 后端 | 0 | `1c6067c` fork | refactor(domain): split ValidateMCPServer into transport, scope and auth checks |
| #300 | 11-14 10:26 | 2 · 后端 | 0 | — | fix(scriptview): keep fail-closed cleanup for masks outside the workspace |
| #303 | 11-13 05:13 | 2 · 后端 | 0 | `00d0a01` fork | refactor(repo): remove unused containsFold helper (#302) |
| #304 | 11-13 20:21 | 13 · 后端/Web/文档 | 3 | `463b16c` fork | fix(clarify): stop swapping a Codex model for CodeBuddy in CodeBuddy-only entries |
| #306 | 11-13 05:24 | 3 · 后端 | 0 | `cedd727` fork | refactor(runtime): reuse dropArgWithValue in isolateCodeBuddyFlagsWithSource (#305) |
| #308 | 11-13 05:35 | 2 · 后端 | 0 | `6b0d090` fork | refactor(runtime): share process sandbox construction between seatbelt and bwrap (#307) |
| #310 | 11-13 05:45 | 2 · 后端 | 0 | `6ea376b` fork | refactor(domain): split ApprovalWaitReason helpers (#309) |
| #312 | 11-13 05:55 | 2 · 后端 | 0 | `6d8dd95` fork | refactor(httpapi): simplify hasBusinessUser with slices.Contains (#311) |
| #314 | 11-13 06:03 | 2 · 后端 | 0 | `c93521b` fork | refactor(preview): extract tar.zst writing from ArchiveDir into writeTarZst |
| #317 | 11-13 06:26 | 2 · 后端 | 0 | `a071725` fork | 【架构优化】从 runner.readRunnerWorkspace 中拆出浏览器操作分支 handleRunnerWorkspaceBrowser |
| #319 | 11-13 06:23 | 2 · 后端 | 0 | `e06ed20` fork | refactor(domain): extract default notification template selection |
| #321 | 11-13 06:38 | 2 · 后端 | 0 | `35e4989` fork | refactor(httpapi): simplify runner waiter notify with runnerWaiter.accepts |
| #323 | 11-13 06:44 | 2 · 后端 | 0 | `08f67c2` fork | refactor(repo): 合并 rooms.go 中重复的聊天消息解码与 ID/RoomID 回填逻辑 |
| #325 | 11-13 06:58 | 2 · 后端 | 0 | `f4897cb` fork | refactor(httpapi): merge duplicated generic template clone lookup |
| #327 | 11-13 07:03 | 2 · 后端 | 0 | `bd89008` fork | refactor(gitwork): split ParseNameStatusNumstat line parsing (#326) |
| #330 | 11-13 07:13 | 2 · 后端 | 0 | `d2e870d` fork | refactor(managedcli): split candidate into metadata fetch and mature version selection |
| #332 | 11-13 07:23 | 2 · 后端 | 0 | `ac162fd` fork | refactor(domain): extract analyticsMRKey from SummarizeDeliveries (#331) |
| #334 | 11-13 07:42 | 2 · 后端 | 0 | `8d1569f` fork | refactor(httpapi): extract workspaceEditAllowed from duplicated workspace edit checks |
| #336 | 11-13 07:43 | 2 · 后端 | 0 | `577a43a` fork | refactor(notification): extract wecomCLIEnv and wecomResultCode from CLIProvider.call (#335) |
| #338 | 11-13 07:53 | 2 · 后端 | 0 | `fa8d59d` fork | refactor(repo): 从 WorkflowMetricsFor 中拆出节点级指标累计 accumulateNodeMetrics |
| #340 | 11-13 08:02 | 2 · 后端 | 0 | `641acb1` fork | refactor(runner-cli): extract concurrencyFromEnv and parseLabels from main (#339) |
| #343 | 11-13 08:13 | 2 · 后端 | 0 | — | refactor(mcpserver): split validateInputValue into per-keyword helpers |
| #345 | 11-13 08:24 | 2 · 后端 | 0 | `a2b3577` fork | refactor(runtime): 拆出 applyDeepSeekCredential 并合并 Pi/DeepSeek 协议映射 |
| #347 | 11-13 08:34 | 4 · 后端 | 0 | `8999893` fork | refactor(httpapi): replace argvHasFlag with slices.Contains |
| #349 | 11-13 08:43 | 2 · 后端 | 0 | `c60f8a1` fork | refactor(repo): use slices.Contains in notificationEventSelected |
| #351 | 11-13 08:53 | 2 · 后端 | 0 | `a54cb29` fork | refactor(domain): split run timing end and interval merge out of computeRunTiming |
| #353 | 11-13 09:05 | 2 · 后端 | 0 | `b3a92d1` fork | refactor(domain): use slices.Contains in AnalyticsFilter.MatchesProject |
| #356 | 11-13 09:14 | 2 · 后端 | 0 | `8e03e08` fork | refactor(domain): use slices.Contains in IsAgentCLIKind and ValidateAgentEffort |
| #358 | 11-13 09:28 | 2 · 后端 | 0 | `c8c719e` fork | refactor(gitwork): extract bindSSHRemotes from StartSSH |
| #360 | 11-13 09:33 | 2 · 后端 | 0 | `c65d8f4` fork | refactor(runeval): split applyGateList into gate ID, remove and upsert helpers |
| #362 | 11-13 09:43 | 2 · 后端 | 0 | `ab3e4a7` fork | refactor(runtime): simplify hasArg with slices.Contains |
| #364 | 11-13 09:56 | 2 · 后端 | 0 | — | refactor(runtime): split sameDirectoryTree into collect and per-entry compare |
| #366 | 11-13 10:07 | 2 · 后端 | 0 | `da3d290` fork | refactor(httpapi): share SSE event cursor helpers in assistant_live.go |
| #369 | 11-13 10:20 | 2 · 后端 | 0 | `4328a59` fork | refactor(httpapi): merge duplicated rules/skills project filter into filterOwnedByProject |
| #371 | 11-13 10:22 | 2 · 后端 | 0 | `a1b0d9a` fork | refactor(domain): use slices.Contains in ScheduleJob.MatchesIssueStatus |
| #374 | 11-13 10:41 | 3 · 后端 | 0 | `c5c6d4a` fork | refactor(runtime): share sandbox env overlay and driver construction |
| #376 | 11-13 10:45 | 2 · 后端 | 0 | `8afb586` fork | refactor(engine): simplify hasRunEvent with slices.ContainsFunc |
| #378 | 11-13 10:53 | 2 · 后端 | 0 | `e206ff0` fork | refactor(preview): simplify containsIndex with slices.ContainsFunc |
| #381 | 11-13 11:05 | 2 · 后端 | 0 | `86494ff` fork | refactor(engine): split validate into validateNode and validateDAG |
| #384 | 11-13 11:14 | 2 · 后端 | 0 | `ff40451` fork | refactor(runtime): hasClosedFrontmatter 复用 splitFrontmatter |
| #387 | 11-13 11:46 | 2 · 后端 | 0 | `9e57b5a` fork | refactor(observability): share protobuf field walk in OTLP parser (#386) |
| #389 | 11-13 11:54 | 3 · 后端 | 0 | — | test(botinteraction): cover sendNotificationCard send, rejection and failure branches |
| #390 | 11-13 11:46 | 2 · 后端 | 0 | `27a9564` fork | refactor(preview): 合并 tar.zst 读取函数重复的成员校验与文件数上限判断 |
| #392 | 11-13 11:54 | 2 · 后端 | 0 | `80ec3cc` fork | refactor(httpapi): extract assistantSessionToolName for session MCP prefix stripping |
| #395 | 11-13 12:06 | 2 · 后端 | 0 | `e1fd1c0` fork | refactor(runtime): extract OpenCode/DeepSeek MCP entry rendering from WriteAgentCLIMCPConfig |
| #397 | 11-13 12:14 | 2 · 后端 | 0 | `8066462` fork | refactor(runtime): share checkpoint path exclusion check |
| #400 | 11-13 21:23 | 2 · 后端 | 0 | `5c0d1cc` fork | 【架构优化】mcpCallMayWrite 词表提为包级变量并去掉重复的工具名切分循环 |
| #402 | 11-13 12:34 | 8 · 后端 | 0 | `f26ce14` fork | refactor(repo): 合并重复的 SHA-256 十六进制摘要计算为 hexSHA256 |
| #405 | 11-13 12:47 | 2 · 后端 | 0 | `1447979` fork | refactor(engine): extract prepareLocalSnapshotRepository from prepareRepository |
| #407 | 11-13 12:56 | 2 · 后端 | 0 | `c8f54a9` fork | refactor(knowledge): split ParseDistilled front matter parsing |
| #409 | 11-13 13:05 | 1 · 后端 | 0 | `7a6be44` fork | refactor(domain): remove unused GroupMemberIDs/HasGroupMember |
| #412 | 11-13 13:15 | 2 · 后端 | 0 | `73b8218` fork | refactor(preview): merge duplicated positive int env parsing (#411) |
| #415 | 11-13 13:23 | 2 · 后端 | 0 | `21cdaa2` fork | refactor(runtime): extract codexMCPServerArgs from codexConfigArgs (#414) |
| #417 | 11-13 13:32 | 2 · 后端 | 0 | `41cbb0d` fork | refactor(runeval): use slices.Contains in appendUnique (#416) |
| #420 | 11-13 13:43 | 2 · 后端 | 0 | `e05b3ac` fork | refactor(domain): extract publicRunnerRepository from PublicRunnerTask (#419) |
| #423 | 11-13 13:53 | 2 · 后端 | 0 | `27c3648` fork | refactor(tracker): share single TAPD record fetch and kind upsert dispatch |
| #425 | 11-13 14:03 | 2 · 后端 | 0 | `ab901dd` fork | refactor(config): extract httpPortFromEnv and envURL from LoadEndpoints |
| #428 | 11-13 14:16 | 2 · 后端 | 0 | `5c1efe0` fork | refactor(botinteraction): extract frame key logging from dial |
| #432 | 11-13 14:26 | 2 · 后端 | 0 | `3150273` fork | refactor(runner): split collectTaskOutputs into focused helpers |
| #435 | 11-13 14:35 | 2 · 后端 | 0 | `e2495cf` fork | refactor(runtime): split path validation and create branch out of WriteWorkspaceFile |
| #437 | 11-13 14:44 | 2 · 后端 | 0 | `49c6d67` fork | refactor(repo): extract unionDuration from businessExecutionDuration |
| #439 | 11-13 14:57 | 2 · 后端 | 0 | `4394849` fork | refactor(httpapi): split assignee and query predicates out of filterIssues |
| #442 | 11-13 15:02 | 2 · 后端 | 0 | `3911c02` fork | refactor(scriptview): split parseConfig validation and share mask sort order |
| #445 | 11-13 15:18 | 6 · 后端 | 0 | — | refactor(httpapi): share asset reference lookups between catalog and assistant handlers |
| #448 | 11-13 15:23 | 2 · 后端 | 0 | `2da46d8` fork | refactor(gitwork): split baseline resolution out of RecordBaseline |
| #452 | 11-13 19:20 | 2 · 后端 | 3 | `47818f5` fork | 【架构优化】合并 gitwork/workbranch.go 中 push/pull 重复的非工作分支严格校验为 verifyOffWorkBranch |
| #456 | 11-13 15:46 | 2 · 后端 | 0 | `addd52a` fork | refactor(engine): extract observationActivity from applyObservation |
| #459 | 11-13 15:54 | 2 · 后端 | 0 | `b92762a` fork | refactor(engine): extract promptRepositoryLine and upstreamNodeOutputs from applyPromptPlaceholders |
| #464 | 11-13 19:31 | 2 · 后端 | 3 | `0e72ab1` fork | refactor(domain): reuse TerminalRunStatus in computeRunTiming |
| #465 | 11-13 19:51 | 3 · Web/文档 | 3 | `44a862c` fork | fix(chatbot): size the artifact preview/source switch like its toolbar buttons |
| #467 | 11-13 16:15 | 2 · 后端 | 0 | `dddc948` fork | refactor(engine): dedupe cloud pool runner listing and enablement check (#466) |
| #470 | 11-13 16:24 | 2 · 后端 | 0 | `0ab25ff` fork | refactor(runtime): extract restoreCheckpointEntry from RestoreWorkspaceCheckpointFrom |
| #472 | 11-13 16:35 | 2 · 后端 | 0 | `06a2f01` fork | refactor(runtime): extract stageArchivedArtifact from RestoreMissingWorkspaceArtifacts |
| #474 | 11-13 16:45 | 2 · 后端 | 0 | `691b5c4` fork | refactor(repo): 合并 analytics 中重复的累加器取用或新建逻辑 (#473) |
| #477 | 11-13 16:55 | 2 · 后端 | 0 | `dc84d68` fork | refactor(httpapi): split temp-dir scan and mtime filter out of recentDeckPathsIn |
| #481 | 11-13 17:05 | 2 · 后端 | 0 | `bd867a9` fork | refactor(repo): extract overlayIssueMRState from applyMRDeliveryStates |
| #482 | 11-14 04:44 | 7 · 后端 | 0 | — | Merge main into CLI-default model handling |
| #484 | 11-13 18:02 | 3 · 后端 | 0 | — | test(github): cover per-source check warnings for unparsable, truncated and unreachable APIs |
| #487 | 11-13 17:34 | 2 · 后端 | 0 | `0602a18` fork | refactor(runtime): split npm args and stage activation out of installNPMCLI |
| #489 | 11-13 20:21 | 2 · 后端 | 3 | `5d9fa25` fork | 【架构优化】合并 workbranch.go 中重复的默认提交身份注入为 withFallbackIdentity |
| #492 | 11-13 17:53 | 2 · 后端 | 0 | `821f385` fork | refactor(httpapi): extract validateInsightUsageAmounts from validateInsightExecution |
| #495 | 11-13 18:07 | 3 · 后端 | 0 | `4cfba04` fork | refactor(runtime): share Codex user provider argv between bindCommand and runner (#493) |
| #498 | 11-13 18:18 | 4 · 后端 | 0 | `3d8d971` fork | refactor(httpapi): reuse domain.TerminalRunStatus instead of roomRunTerminal |
| #499 | 11-13 19:06 | 2 · Web/CI/构建 | 0 | `2fac104` fork | [Loopforge 自动化] fix(web): list acceptance gates in SOP execution order |
| #501 | 11-13 18:25 | 9 · 后端 | 0 | `7c51f0e` fork | refactor(engine): 删除 isTerminalRunStatus 重复实现，复用 domain.TerminalRunStatus |
| #505 | 11-13 18:44 | 2 · 后端 | 0 | `4a8107f` fork | refactor(runner): share invocation repository fallback via invocationRepositories (#504) |
| #507 | 11-14 11:16 | 2 · 后端 | 0 | — | test(runner): move TestRunnerTaskCancelDesired into the platform cancel suite |
| #508 | 11-13 19:25 | 2 · 后端 | 0 | `c8fb2cd` fork | [Loopforge 自动化] refactor(repo): 合并 buildRequirementLanes 中重复的 SOP 绑定字段复制为 withSOPBinding (#502) |
| #510 | 11-14 02:52 | 3 · 后端 | 0 | — | Merge branch 'main' into refactor/browser-view-on-event-509 |
| #512 | 11-13 19:48 | 3 · 后端 | 0 | `4f8bc98` fork | 【架构优化】从 readSkillZipBytes 中拆出 zip 条目筛选与顶层目录剥离 skillZipEntries |
| #514 | 11-13 19:35 | 3 · 后端 | 0 | `6b2b549` fork | refactor(tracker): share layered Gongfeng MR payload decoding via visitGongfengMRPayload |
| #516 | 11-13 19:44 | 2 · 后端 | 0 | `8a8fcd2` fork | refactor(knowledge): extract cleanProjectSources and resolveSettingsToken from UpdateSettings (#515) |
| #518 | 11-13 19:55 | 2 · 后端 | 0 | `bb835c2` fork | refactor(auth): flatten NormalizePhotoSpec into a single-level switch (#517) |
| #520 | 11-13 20:07 | 2 · 后端 | 0 | `f443698` fork | refactor(runtime): split checkpoint entry header and file copy out of CreateWorkspaceCheckpointTo |
| #522 | 11-13 20:14 | 2 · 后端 | 0 | `f29682b` fork | refactor(runtime): 拆分 FoldCursorCLIModels 的扫描、普通 ID 并入家族与家族行构造 (#521) |
| #524 | 11-13 20:24 | 2 · 后端 | 0 | `eb00faa` fork | refactor(httpapi): extract assistantWrappedToolParams and assistantToolSearchNames from assistantToolDisplay (#523) |
| #526 | 11-13 20:43 | 2 · 后端 | 0 | `b8f2c05` fork | 【架构优化】从 resolveAssets 中拆出 roleMCPServerVisible 并复用 loadMCPFiles |
| #528 | 11-13 20:53 | 2 · 后端 | 0 | `c455800` fork | refactor(domain): extract analyzeDiagnosticTools from AnalyzeAssistantDiagnostics (#527) |
| #530 | 11-13 21:10 | 2 · 后端 | 0 | `4c077fa` fork | 【架构优化】从 buildLoopRetro 中拆出遥测信号统计 collectLoopRetroSignals (#529) |
| #532 | 11-14 04:38 | 6 · 后端/文档 | 0 | — | Merge main into schema gap check |
| #533 | 11-14 01:49 | 2 · 后端 | 3 / [15:01](https://github.com/Tencent/LoopForge/actions/runs/43476549178/job/132061750137) | `5ffdba2` fork | refactor(httpapi): use slices.Contains in sharedAgentReferences (#531) |
| #535 | 11-13 22:55 | 3 · 后端 | 0 | `5040c1b` fork | 【架构优化】从 assistantDraftWarnings 中拆出 SOP 删除前检查 workflowDeleteWarnings |
| #537 | 11-13 21:37 | 3 · 后端 | 0 | `d5a5b7f` fork | refactor(httpapi): extract stripUploadTopDir shared by skillFiles and mcpFiles (#536) |
| #539 | 11-13 21:47 | 2 · 后端 | 0 | `e34585a` fork | 【架构优化】用 slices.Contains 简化 marketItem 的别名查找循环 (#538) |
| #541 | 11-13 21:54 | 2 · 后端 | 0 | `0f99857` fork | 【架构优化】archivedWorkspaceFiles 复用 artifactOutputFiles 并拆出 hasHiddenPathSegment (#540) |
| #544 | 11-13 22:05 | 2 · Web | 0 | `17bdb29` fork | fix(web): keep runner command boxes from jumping on hover |
| #545 | 11-13 22:07 | 2 · 后端 | 0 | `4a1b3a0` fork | refactor(runner): extract conversation tool forward and reply delivery from readControls |
| #548 | 11-13 22:54 | 2 · 后端 | 0 | `42b92ff` fork | 【架构优化】从 nativeMetadata 中拆出原生事件里程碑分类 nativeMilestone |
| #550 | 11-13 22:54 | 2 · 后端 | 0 | `28dcf00` fork | 【架构优化】从 workbenchReferences 中拆出引用排序 sortAssistantReferences 与分页 pageAssistantReferences |
| #553 | 11-13 22:35 | 2 · 后端 | 0 | `10c60b3` fork | refactor(runtime): extract stdio JSON-RPC line read/write from runStdioMCP |
| #555 | 11-13 22:44 | 2 · 后端 | 0 | `5c98f7c` fork | refactor(domain): split MR dedup and state counting out of SummarizeDeliveries |
| #557 | 11-13 22:55 | 2 · 后端 | 0 | — | refactor(domain): split event scan and node-row fallback out of computeRunTiming |
| #562 | 11-13 23:10 | 2 · 后端 | 0 | `2131c5d` fork | refactor(runeval): extract bumpToolRepeat from BuildFacts |
| #564 | 11-14 06:58 | 13 · 后端/Web/CI/构建/文档 | 0 | — | docs(changelog): drop the 0.3.9-rc.1 section main removed |
| #565 | 11-13 23:39 | 2 · 后端 | 0 | — | test(engine): cover cleanupWorkspace routing cleaned runs to rearm |
| #567 | 11-13 23:49 | 5 · 后端 | 0 | `8c7c93c` fork | 【架构优化】合并 repo 包中 payload JSON 文本提取的方言分支为 sqlPayloadJSONText |
| #569 | 11-13 23:49 | 3 · 后端 | 0 | `3fea73a` fork | 【架构优化】合并 preview/archive.go 中四处重复的展开大小上限判断为 checkExpandedSize |
| #571 | 11-14 00:34 | 15 · 后端 | 0 | `05d5aaf` fork | 【架构优化】合并 engine.hasRunningNode 与 httpapi.runHasActiveNode 为 domain.Run.HasActiveNode |
| #573 | 11-14 02:52 | 3 · 后端 | 0 | — | Merge branch 'main' into refactor/gitwork-retry-push-572 |
| #575 | 11-14 00:16 | 2 · 后端 | 0 | `7b3bc01` fork | 【架构优化】合并 workspace_preparation 中重复的 UTF-8 安全尾部截断为 tailAtRuneBoundary |
| #578 | 11-14 00:13 | 2 · 后端 | 0 | `9aeef81` fork | refactor(repo): extract issue deletion task/run accumulation into addTask/addRun |
| #581 | 11-14 00:43 | 2 · 后端 | 0 | `ed63530` fork | 【架构优化】从 PrepareMCPCall 中拆出 MCP server 配置条目转换 mcpCallConfigServers |
| #584 | 11-14 00:43 | 2 · 后端 | 0 | `0c3d3ed` fork | refactor(tracker): extract gongfengSyncedIssue from pullGongfengIssues |
| #585 | 11-14 01:03 | 2 · 后端 | 0 | `e86d108` fork | 【架构优化】从 listSessions 中拆出会话去重与可见性过滤 visibleAssistantConversations |
| #588 | 11-14 01:20 | 2 · 后端 | 0 | — | test(tracker): cover upsertTAPDRecord rejecting occupied local IDs |
| #590 | 11-14 01:02 | 2 · 后端 | 0 | `e5cf42e` fork | refactor(domain): split command partition and git gate construction out of SplitVerificationGates |
| #592 | 11-14 01:39 | 3 · 后端 | 0 | — | test(engine): cover commitNodeSuccess when the store write fails |
| #594 | 11-14 01:14 | 2 · 后端 | 0 | `dab7ac7` fork | refactor(notification): extract renderAIReport from generateReport (#593) |
| #596 | 11-14 01:24 | 2 · 后端 | 0 | `57ae473` fork | refactor(gitwork): split ParseSSHRemote into splitSSHRemote and validSSHRepoPath (#595) |
| #599 | 11-14 01:33 | 2 · 后端 | 0 | `575fec4` fork | refactor(repo): extract nodeMetricStage from requirementPhaseMeasurements |
| #603 | 11-14 01:46 | 2 · 后端 | 0 | `ef94871` fork | refactor(repo): extract run notification binding selection from enqueueRunNotifications |
| #607 | 11-14 02:21 | 3 · 后端 | 0 | — | test(httpapi): cover assistantActivityLog event/byte/text limit branches |
| #611 | 11-14 02:07 | 2 · 后端 | 0 | `54bb632` fork | refactor(domain): split DetectFrontendProject walk into previewDetectSkipDir and frontendFileEvidence |
| #614 | 11-14 02:30 | 3 · 后端 | 0 | — | test(runner): cover workspace conversation factory failure cleanup and managed CLI inventory |
| #616 | 11-14 02:15 | 2 · 后端 | 0 | `b0ceea2` fork | refactor(httpapi): extract businessReportFilter from business domain report (#613) |
| #620 | 11-14 02:37 | 3 · 后端 | 0 | — | test(observability): pin classifySpan aliases, attribute priority and message clipping |
| #621 | 11-14 02:51 | 4 · 后端 | 0 | — | test(httpapi): cover protocol() wiring around inspectProtocolMetadata |
| #625 | 11-14 02:34 | 2 · 后端 | 0 | — | refactor(httpapi): split assistantLineEvents into message, partial and tool-result handlers |
| #626 | 11-14 02:41 | 2 · 后端 | 0 | `56cf1c9` fork | refactor(httpapi): merge assistantStreamFilter answer/thinking text accumulation and extract acceptTool (#624) |
| #629 | 11-14 03:02 | 3 · 后端 | 0 | `c174190` fork | 【架构优化】从 githubPRFiles 中拆出单文件 diff 截断与总预算分配 githubPRFilePatch |
| #631 | 11-14 03:10 | 3 · 后端 | 0 | `15e1880` fork | 【架构优化】从 prepareSandboxEnvironment 中拆出环境变量覆盖层组装 environmentOverlay |
| #633 | 11-14 03:08 | 2 · 后端 | 0 | `bda7b97` fork | refactor(httpapi): extract TAPD/Gongfeng read instructions from clarifyMCPInstructionsFor (#632) |
| #636 | 11-14 03:39 | 3 · 后端 | 0 | — | test(users): drive probeCLIModels through its route |
| #638 | 11-14 03:25 | 2 · 后端 | 0 | `e36c305` fork | 【架构优化】从 writeDownloadZIP 中拆出 tar 成员校验 downloadEntryName 与 ZIP 成员写出 copyDownloadMember (#637) |
| #642 | 11-14 03:31 | 2 · 后端 | 0 | `d31b084` fork | 【架构优化】合并 tracker 创建结果的对象/数组首元素解码为 decodeCreatedRecord (#639) |
| #643 | 11-14 03:37 | 2 · 后端 | 0 | `f12f59f` fork | 【架构优化】从 ValidateNotificationBinding 中拆出 validateNotificationTemplate 与 validateNotificationEvents (#640) |
| #647 | 11-14 03:47 | 2 · 后端 | 0 | `37d4d1b` fork | 【架构优化】合并 analytics 维度累加函数中重复的用量、状态与耗时累加逻辑 (#646) |
| #649 | 11-14 07:30 | 22 · 后端/Web/文档 | 0 | — | test(chat): pin the trailing-question trace ID re-stamp to its narrow case |
| #650 | 11-14 03:59 | 2 · 后端 | 0 | `0534934` fork | 【架构优化】合并 buildRequirementLanes 中重复的 workflow 解析与 lane 命名回退 (#648) |
| #651 | 11-14 14:11 | 20 · 后端/Web/文档 | 0 | — | test(gate): guard lcov filter path normalisation in diff_coverage |
| #654 | 11-14 04:20 | 2 · 后端 | 0 | — | 【架构优化】从 collectArtifactsMode 中拆出 contractArtifact 与 repositoryPatchArtifacts |
| #656 | 11-14 04:29 | 2 · 后端 | 0 | `4f6aba7` fork | 【架构优化】从 bindCommand 中拆出遥测环境变量注入 bindTelemetryEnvironment (#655) |
| #658 | 11-14 04:48 | 4 · 后端 | 0 | `7562385` fork | 【架构优化】合并 domain.workBranchSegment 与 engine.safeGitRefSegment 为 domain.SafeGitRefSegment (#657) |
| #662 | 11-14 05:05 | 2 · 后端 | 0 | `dfd9e2c` fork | refactor(runner): extract runtimeProviderSpecs and builtinRuntimes from detectRuntimes (#661) |
| #664 | 11-14 05:16 | 13 · 后端/Web/CI/构建 | 0 | — | feat(schedule): let AI scheduled tasks pick a reasoning effort |
| #666 | 11-14 05:26 | 2 · 后端 | 0 | `c1d9bff` fork | 【架构优化】合并 authCurrent 中重复的用户记录同步为 syncAuthUserRecord (#665) |
| #668 | 11-14 05:34 | 5 · 后端 | 0 | `c5a62aa` fork | 【架构优化】合并 repo 中 7 处重复的 execution_context JSON 取值表达式为 sqlExecutionContextJSONText (#667) |
| #670 | 11-14 05:46 | 2 · 后端 | 0 | — | 【架构优化】从 writeConversationAssets 中拆出 conversationSkills 与 conversationAssetHome |
| #672 | 11-14 05:55 | 2 · 后端 | 0 | — | 【架构优化】从 discoverAssistantEngineModels 中拆出地址拼接 assistantModelsEndpoint 与响应解析 decodeAssistantDiscoveredModels |
| #674 | 11-14 06:04 | 2 · 后端 | 0 | `6fb39a5` fork | 【架构优化】合并 githubIssueClose/githubPRClose 中重复的关闭前评论校验与发布 (#673) |
| #678 | 11-14 06:22 | 3 · 后端 | 0 | `f5c5963` fork | 【架构优化】runs/dispatch 复用 repositoryRunContext，删除两处重复的 RepositoryContext 字段拷贝 (#676) |
| #680 | 11-14 06:39 | 3 · 后端 | 0 | `8830005` fork | 【架构优化】合并 serveRunnerBinary/serveGateMCPBinary 中重复的 os/arch 参数解析与校验为 binaryDownloadPlatform (#679) |
| #682 | 11-14 06:54 | 2 · 后端 | 0 | `dcfbd81` fork | 【架构优化】从 ExtractTarZst 中拆出单个 tar 成员落盘 extractTarMember |
| #685 | 11-14 06:57 | 2 · 后端 | 0 | `c780be2` fork | 【架构优化】从 materializeInvocationForDriver 中拆出 materializeAgentRules 与 materializeMCPDefinitions (#684) |
| #687 | 11-14 07:05 | 2 · 后端 | 0 | `97b3cdc` fork | 【架构优化】从 alignBaseline 中拆出执行机器对齐结果判定 runnerAlignOutcome |
| #691 | 11-14 08:26 | 3 · 后端 | 0 | — | test(preview): assert tarBudget wiring in InspectTarZst/ExtractTarZst |
| #693 | 11-14 08:51 | 4 · 后端 | 0 | — | test(engine): wait for lock holders to release before checking keyedLocks leak |
| #695 | 11-14 08:49 | 3 · 后端 | 0 | — | test(httpapi): 补充 notificationBotModel 保留现值与 findNotificationTarget 精确匹配的测试 |
| #697 | 11-14 09:11 | 4 · 后端 | 0 | — | test(runtime): cover ApplyAgentCLICredential keeping argv per CLI and rejecting unknown kinds |
| #699 | 11-14 08:59 | 2 · 后端 | 0 | `27bde9e` fork | 【架构优化】从 githubPullsList 中拆出 PR 返回体类型 githubPullItem 与字段映射 out (#698) |
| #702 | 11-14 09:18 | 3 · 后端 | 0 | — | test(runtime): cover ensureScriptContainer helper owner wiring |
| #705 | 11-14 09:26 | 3 · 后端 | 0 | — | test(runtime): cover installZCodeApproval resume script, missing/unreadable/malformed config and caller error propagation |
| #707 | 11-14 10:05 | 3 · 后端 | 0 | — | test(domain): cover ParseGongfengRepoURL reference edge cases |
| #710 | 11-14 10:34 | 3 · 后端 | 0 | `f952b37` fork | [Loopforge 自动化] 【架构优化】从 validateAssistantDraft 中拆出定时任务草稿校验 validateAssistantTaskDraft |
| #711 | 11-14 10:35 | 3 · 后端 | 0 | — | test(httpapi): cover identity lookup failure in exchangeDeviceCode and poll |
| #714 | 11-14 10:34 | 2 · 后端 | 0 | `c300bcf` fork | 【架构优化】从 readWorkspaceTree 中拆出空目录剔除 pruneEmptyWorkspaceDirs |
| #715 | 11-14 11:03 | 3 · 后端 | 0 | `a1b33f8` fork | 【架构优化】从 generateBotReply 中拆出历史消息整理 botReplyMessages 与绑定交互判断 isBotBindingExchange |
| #718 | 11-14 11:10 | 3 · 后端 | 0 | `559611c` fork | 【架构优化】从 importTAPD 中拆出仓库列表整理 tapdImportRepositoryIDs 与导入后仓库回写 applyTAPDImportRepositories |
| #720 | 11-14 11:46 | 3 · 后端 | 0 | — | test(httpapi): pin roomNodeMessage author, timing and floor ordering edges |
| #722 | 11-14 14:27 | 11 · 后端 | 0 | — | test(gitwork): cover scratch-index edge cases of the undelivered-work check |
| #723 | 11-14 12:04 | 3 · 后端 | 0 | — | test(httpapi): cover fork/edit attachment claiming through the handlers |
| #725 | 11-14 11:37 | 2 · 后端 | 0 | `b9980ba` fork | 【架构优化】从 approvalReviewer 中拆出审查输入构造 approvalReviewInput |
| #728 | 11-14 12:24 | 3 · 后端 | 0 | `cdce425` fork | 【架构优化】合并 run_workspace.go 中五处重复的工作区不可用响应构造为 workspaceUnavailablePayload |
| #730 | 11-14 12:23 | 3 · 后端 | 0 | — | test(httpapi): cover applySelectedRecommendations apply/failure paths and persistence |
| #732 | 11-14 12:05 | 2 · 后端 | 0 | `51ca439` fork | 【架构优化】从 linkConversationRequirement 中拆出新需求构造 conversationRequirementIssue |
| #734 | 11-14 12:53 | 3 · 后端 | 0 | — | test(repo): cover saveIssueLocked inherit/closure edge cases |
| #736 | 11-14 12:54 | 3 · 后端 | 0 | — | test(engine): cover pushConversationOnRunner wiring |
| #738 | 11-14 13:09 | 3 · 后端 | 0 | `88d1819` fork | 【架构优化】从 BusinessDomainReport 中拆出业务域分组构造 newBusinessDomainGroups 与项目用量累加 addBusinessDomainProjectUsage |
| #740 | 11-14 13:11 | 3 · 后端 | 0 | — | test(domain): keep provided MR URLs and empty result shape in ResolveRunDeliveryMRs |
| #742 | 11-14 13:15 | 3 · 后端 | 0 | `acadf85` fork | 【架构优化】从 dockerAgentConfigMasks 中拆出工作区挂载目标准备 ensureDockerMaskTargets |
| #745 | 11-14 13:37 | 3 · 后端 | 0 | — | test(engine): guard IngestWorkerEnvelope observation/log split |
| #746 | 11-14 13:27 | 18 · 后端/Web/文档 | 0 | — | fix(gate): address diff_coverage review round 3 (test files, lcov size, paths, assistant schema) |
| #749 | 11-14 14:04 | 3 · 后端 | 0 | — | test(repo): guard per-row lease re-check in stageWorkspaceCheckpointChunks |
| #750 | 11-14 14:12 | 3 · 后端 | 0 | `01dfcfe` fork | 【架构优化】合并 workflows handler 中重复的沙箱档案校验与 revision conflict 响应 |
| #752 | 11-14 14:08 | 2 · 后端 | 0 | `8a8637a` fork | 【架构优化】从 gitIndexLockHeldUnder 中拆出单进程 fd 目录扫描 fdDirHoldsFile |
| #754 | 11-14 14:30 | 3 · 后端 | 0 | — | test(repo): cover KnowledgeArtifacts over-fetch and limit clamping |
| #756 | 11-14 14:46 | 4 · 后端 | 0 | — | test(managedcli): cover executeCommand failure output, watcher join and capture Stat errors |
| #758 | 11-14 15:16 | 4 · 后端 | 0 | — | test(httpapi): cover owner-allowed paths and bearer auth for requirePersonalMCPOwner |
| #760 | 11-14 15:10 | 3 · 后端 | 0 | — | test(engine): cover waitForPoolRunner timeout, late runner and bind failure |
| #762 | 11-14 15:36 | 4 · 后端 | 0 | — | test(botinteraction): pin notificationCard content-reason wiring |
| #765 | 11-14 16:04 | 3 · 后端 | 0 | — | test(auth): 钉住 ExchangeCode 编排的输入清洗、短路、时间与响应上限 |
| #768 | 11-14 17:01 | 3 · 后端 | 0 | — | test(repo): pin UpdateUserCredentials secret wiring per field |
| #770 | 11-14 17:55 | 3 · 后端 | 0 | — | test(httpapi): split PEM header literal so check-public stays green |
| #772 | 11-14 18:07 | 3 · 后端 | 0 | `947dfbb` fork | 【架构优化】合并 repo/analytics.go 中重复的趋势点取用/新建与趋势桶需求登记为 analyticsTrendEntry/addAnalyticsTrendIssue |
| #776 | 11-14 18:23 | 5 · 后端 | 0 | — | test(httpapi): guard stored-session project check, issue/branch notes and attachment authors in workbench context |
| #778 | 11-14 18:07 | 3 · 后端 | 0 | `b215f23` fork | 【架构优化】从 EnsureToolchainCLI 中拆出包暂存复制 installToolchainPackage 与启动器软链 linkToolchainLaunchers |
| #780 | 11-14 18:21 | 3 · 后端 | 0 | — | revert(engine): drop duplicate guard test; helper collided with runner_await_observe_guard_test.go |
| #782 | 11-14 18:08 | 3 · 后端 | 0 | — | test(runtime): guard dev server client proxy, keep-alive, dial context and cache wiring |
| #784 | 11-14 18:27 | 3 · 后端 | 0 | — | test(engine): cover materializeInputs patch-failure, symlink and child-SOP input paths |
| #786 | 11-14 18:37 | 3 · 后端 | 0 | — | test(engine): cover existing-branch, shared-merge failure and multi-repo paths of pushLocalConversationRepository |
| #788 | 11-14 19:18 | 3 · 后端 | 0 | — | test(runtime): pin Claude stream frame dispatch and event fields |
| #791 | 11-14 19:47 | 3 · 后端 | 0 | `ba1059d` fork | 【架构优化】从 markIssueDelivered 中拆出交付守卫判断 issueDeliverySkipped |
| #792 | 11-14 19:57 | 3 · 后端 | 0 | — | test(httpapi): drive insight source validators through the create handler |
| #797 | 11-14 20:28 | 3 · 后端 | 0 | — | ci: retrigger checks (core flaked on TestSSHRepositoryCheckoutAndDelivery, unrelated to this change) |
| #800 | 11-14 20:01 | 2 · 后端 | 0 | `0599fc3` fork | 【架构优化】从 antigravityStream.translate 中拆出 translateStep/responseStep/antigravityToolStep |
| #803 | 11-14 20:06 | 2 · 后端 | 0 | `ccd7499` fork | 【架构优化】从 PutAssistantTask 中拆出写入字段推算 stampAssistantTask 与完成通知判定 assistantTaskCompletion |
| #805 | 11-14 20:48 | 3 · 后端 | 0 | — | test(repo): guard mrMergeAnalytics run-issue lookup, unknown issues and query failure |
| #807 | 11-14 20:58 | 3 · 后端 | 0 | `a14dd44` fork | 【架构优化】从 WriteWorkspaceFile 中拆出覆盖写入的原子替换 replaceWorkspaceFile |
| #809 | 11-14 21:22 | 3 · 后端 | 0 | — | test(runtime): cover live-holder retry and uninspectable lock paths of recoverGitIndexLockConflict |
| #813 | 11-14 21:19 | 2 · 后端 | 0 | `6a12797` fork | 【架构优化】从 savePlatformCodeBuddyKey 中拆出共享 Key 模型清单归一化与校验 sharedCodeBuddyModels |
| #815 | 11-14 23:50 | 6 · 后端 | 0 | — | fix(runtime): parse only docker stdout when probing workspace architecture |
| #816 | 11-14 21:29 | 2 · 后端 | 0 | `fef89e1` fork | 【架构优化】从 downloadAttachment 中拆出转录附件上传者查找 transcriptAttachmentOwner |
| #818 | 11-14 22:47 | 4 · 后端 | 0 | — | test(httpapi): 补 callProxyTool 未传 arguments 时活动 query 记为空串的测试 |
| #820 | 11-14 22:47 | 4 · 后端 | 0 | `ce0a29f` fork | 【架构优化】从 syncTasks 中拆出 Issue 最新 Task 选取 latestTaskByIssue 与 Issue 回写 syncIssuesToLatestTask |
| #823 | 11-14 22:37 | 3 · 后端 | 0 | `a594c4e` fork | 【架构优化】合并 dispatchRoomIssue/dispatchToIssue 中重复的多 SOP 派发目标拼接为 roomDispatchTarget |
| #825 | 11-14 22:27 | 2 · 后端 | 0 | `08d7e2f` fork | 【架构优化】从 finishAt 中拆出活动节点判断 hasActiveNode 与残留执行节点收敛 cancelRunningNodes |
| #827 | 11-14 22:51 | 2 · 后端 | 0 | `ad33e14` fork | 【架构优化】从 extractJSONObject 中拆出 result 事件解码 streamResultEventValue 并合并重复的摘要兜底 clarifySummaryFallback |
| #832 | 11-15 00:53 | 3 · 后端 | 0 | — | test(auth): guard token create reasons, trimmed length and owner persistence for #832 |
| #833 | 11-15 00:33 | 3 · 后端 | 0 | — | test(httpapi): guard githubIssueLatestComments paging and failure paths |
| #837 | 11-15 02:46 | 3 · 后端 | 0 | — | test(httpapi): guard resolveSteerModel shared-key pick, effort target and 400 path |
| #840 | 11-15 03:42 | 3 · 后端 | 0 | — | test(httpapi): refuse a proposal typed into a user message in recordProposalDecision |
| #843 | 11-15 04:01 | 3 · 后端 | 0 | — | test(tracker): pin per-call-site TAPD sync_stories logs |
| #844 | 11-15 04:01 | 3 · 后端 | 0 | — | test(tracker): pin PullAll dispatch and error recording after split |
| #846 | 11-15 04:11 | 3 · 后端 | 0 | — | test(runtime): pin docker network discovery behind conversationDockerAccess |
| #848 | 11-15 04:17 | 3 · 后端 | 0 | — | test(gitwork): pin case folding, xml guard, protection order and bare dirs for IsTestArtifactOrCachePath split |
| #851 | 11-15 04:30 | 3 · 后端 | 0 | — | test(runner): cover renewTaskLease deadline cap, shutdown abort and loop expiry carry-over |
| #853 | 11-15 04:39 | 3 · 后端 | 0 | — | test(engine): cover per-repository delivery verification after split |
| #855 | 11-15 04:39 | 2 · 后端 | 0 | — | 【架构优化】从 handleDevServerHTTP 中拆出方法归一化 devServerMethod 与转发失败分类 devServerFailureState |
| #858 | 11-15 05:33 | 3 · 后端 | 0 | — | test(runner): guard syncWorkBranch dispatch and pushSharedTarget conflicts |
| #859 | 11-15 05:21 | 3 · 后端 | 0 | — | test(runtime): guard Codex item dispatch, output precedence and phases |
| #862 | 11-15 05:25 | 3 · 后端 | 0 | — | test(auth): cover readPhotoResponse status, size limits and no negative caching |
| #865 | 11-15 05:44 | 3 · 后端 | 0 | — | test(repo): cover member upsert, backfill payload/scope and undecodable issues |
| #866 | 11-15 05:52 | 3 · 后端 | 0 | — | test(httpapi): cover activity budget exact limits, replace rejections and add() call sites |
| #868 | 11-15 06:04 | 3 · 后端 | 0 | — | test(httpapi): pin publishWorkbenchArtifact wiring around extracted helpers |
| #870 | 11-15 05:53 | 2 · 后端 | 0 | — | refactor(engine): extract workspace save lease watcher from EndConversationWorkspaceUseAsync |
| #872 | 11-15 05:59 | 2 · 后端 | 0 | — | 【架构优化】从 putAssistantSession 中拆出已存会话字段继承 inheritStoredAssistantSession |

</details>
