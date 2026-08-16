# Wind Comic 后端、AI 服务与 Agent 编排完整面试问答

> 用法：先背“项目口径”和 30 秒版本，再按轮次练习。回答遵循“结论 → 设计 → 边界 → 改进”，不要把尚未实现的生产方案说成现状。

---

## 一、重要对象与代码速查

| 名称 | 定位 | 关键文件 |
|---|---|---|
| `create-stream` | 创作 HTTP / SSE 入口 | `app/api/create-stream/route.ts` |
| `runCreatePipeline` | 完整成片应用主链 | `lib/create-pipeline.ts` |
| `HybridOrchestrator` | AI 制片领域总控 | `services/hybrid-orchestrator.ts` |
| `DirectorPlan` | 导演规划契约 | `types/agents.ts` |
| `Script` | 编剧结构化输出 | `types/agents.ts` |
| `Storyboard[]` | 镜头计划 / 分镜资产 | `types/agents.ts` |
| `VideoClip[]` | 镜头视频结果 | `types/agents.ts` |
| `EditResult` | 剪辑、时间线、成片结果 | `types/agents.ts` |
| `DirectorReview` | 质量复审与返工意见 | `types/agents.ts` |
| `pipeline_jobs` | 持久任务状态机 | `lib/repos/pipeline-job-repo.ts` |
| Project Asset | 阶段检查点与事实载体 | `lib/pipeline-checkpoints.ts` |
| Provider Registry | 多模型能力路由 | `lib/*-providers/` |
| Workflow Graph | 可配置 DAG | `lib/agent-workflow-core.ts` |
| Workflow Engine | 拓扑分层执行 | `lib/workflow-engine.ts` |

## 二、必须守住的真实性边界

### 可以说“已实现”

- Next.js 全栈后端、JWT、SSE。
- 从创意到计划、剧本、视觉、分镜、视频、编辑、复审的主链。
- 多 LLM / 图片 / 视频 / TTS Provider 接入与一定程度的回退。
- 数据库队列、心跳、重试、死信和孤儿恢复。
- 资产检查点、缺失镜头补跑和定点返工。
- SQLite 默认、PostgreSQL 驱动可选。
- 自研 DAG 校验、拓扑分层、同层并发、dry-run / real-run。
- Prompt 规则安全、预算拦截、成本归因、质量事件。

### 应说“配置后可用 / 具备接入点”

- 本地 XVERSE LLM。
- ComfyUI 本地图片工作流（含可选 ControlNet 草图硬锁，预备态）。
- LTX / Wav2Lip 等自托管服务。
- PostgreSQL 多实例部署。
- Redis 跨实例事件桥接。

### 应说“生产化建议”，不能说已完成

- 完全离线、全模态本地高质量生成。
- 独立分布式 Worker 集群和强 SLA。
- Temporal / LangGraph 持久状态图。
- 全路由已经形式化证明的多租户隔离。
- 完整自动媒体合规审核。
- 所有 Provider 统一幂等和账单对账。

---

# 第一轮：项目与业务

## Q1：请介绍一下 Wind Comic 项目。

**参考回答：**

Wind Comic 是一个 AI 漫剧生产平台，用户输入一句故事创意后，系统将其拆成导演策划、剧本、角色和场景设计、分镜、视频、配音剪辑与最终复审。后端基于 Next.js Route Handlers，默认 SQLite、可切 PostgreSQL；核心是 `runCreatePipeline` 和 `HybridOrchestrator`。系统通过多 Provider 接入 LLM、图片、视频和 TTS，并用项目资产做检查点，因此长任务失败后可以只补缺失阶段或镜头，而不是全量重跑。

## Q2：这个项目最核心的技术挑战是什么？

**参考回答：**

不是调用某一个模型，而是治理一个分钟级、会真实计费、会部分成功的多模态流水线。主要挑战是跨阶段人物与画风一致性、外部 Provider 的超时和降级、镜头级幂等恢复、SSE 长连接体验、媒体文件的确定性合成，以及成本和质量之间的平衡。

## Q3：为什么要拆成多个 Agent 岗位？

**参考回答：**

不同阶段的输入输出、Prompt、模型能力和失败策略不同。导演输出方案，编剧输出结构化剧本，视觉岗位产出参考资产，分镜师定义镜头，视频制作调用昂贵视频模型，剪辑师完成 TTS 和 FFmpeg。拆分岗位后，每步可以独立持久化、观测、降级和返工，也便于替换 Provider。

## Q4：它是全本地系统吗？

**参考回答：**

应用、SQLite、资产和 FFmpeg 可以本地运行，但默认模型推理大多使用外部 API。代码支持本地 XVERSE、ComfyUI、自托管 LTX / Lip-sync 等接入点，`MOCK_ENGINES=1` 还能全本地跑测试，但 Mock 不代表生产质量。要完全离线还要补齐本地 TTS、GPU 服务、模型部署和存储方案。

## Q5：为什么产品要输出中间资产，而不是只给最终视频？

**参考回答：**

中间资产同时服务于用户可控性、恢复和成本治理。用户可以先看剧本、角色和分镜；失败时从最近资产继续；导演审核也能定位到单镜头返工。若只存最终视频，任何后期失败都可能导致整条昂贵链路重跑。

---

# 第二轮：后端架构

## Q6：后端是怎么分层的？

**参考回答：**

接口层在 `app/api`，负责鉴权、参数、安全、预算和 SSE；应用层 `create-pipeline` 负责编排、落盘、恢复与事件；领域层 `HybridOrchestrator` 管理岗位 Prompt、上下文一致性和返工；基础设施层是 Provider、FFmpeg 和 Event Bus；持久层通过 Repository 与 `DbDriver` 访问 SQLite 或 PostgreSQL。

## Q7：为什么用 Next.js 单体，而不是前后端分离？

**参考回答：**

早期单体能共享 TypeScript 类型、减少部署单元，并方便访问本地文件、SQLite 和 FFmpeg，适合快速产品化。缺点是 Web、Worker 和媒体任务共享进程，扩容与资源隔离受限。并发增长后优先把 Worker 拆出去，而不是一开始就把所有模块微服务化。

## Q8：创建接口为什么使用 SSE？

**参考回答：**

创建是服务器单向持续推送的长任务，SSE 比轮询延迟低、比 WebSocket 简单，浏览器原生支持事件流。它用来推送阶段、进度、资产、心跳和完成事件。但 SSE 不是事实存储，断线恢复仍依赖任务和资产表。

## Q9：请求内执行和队列执行有什么区别？

**参考回答：**

默认模式在 HTTP 请求内运行 `runCreatePipeline`，事件直接写 SSE；`PIPELINE_QUEUE=1` 时请求先写 `pipeline_jobs`，进程内 Worker 异步认领，SSE 从 Event Bus 和数据库日志接收事件。队列模式能让浏览器断开后继续执行，也能处理进程重启，但当前 Worker 默认仍与 Web 同部署单元。

## Q10：任务状态机如何设计？

**参考回答：**

状态是 queued、running、done、failed。claim 时用条件更新保证只有一个 Worker 成功；失败且未耗尽次数时重新 queued，最多尝试三次；Worker 定期心跳，约 90 秒无心跳的 running 任务作为孤儿恢复；超过 24 小时则失败。死信可以人工重投。

## Q11：如何防止多个 Worker 领取同一任务？

**参考回答：**

先读取候选任务，再执行 `UPDATE ... WHERE id=? AND state='queued'`。只有 changes 为 1 的 Worker 获得任务，其他 Worker 更新失败后重新 claim。这是乐观并发控制。更高要求下可以使用 `FOR UPDATE SKIP LOCKED` 或租约 token。

## Q12：为什么不能用一个数据库事务包住整条 AI 流水线？

**参考回答：**

模型和视频生成持续数分钟，长事务会长期持锁、占连接，失败回滚还会丢掉已经付费生成的有效资产。项目使用阶段提交：每步完成就持久化，失败后从检查点继续。这是一种面向外部副作用的 Saga / 最终一致性思路。

## Q13：Event Bus 的作用是什么？

**参考回答：**

它把 Worker 产生的流水线、通知、评论或关卡事件实时分发给订阅者。单进程 EventEmitter 延迟低；多实例需要 Redis 等桥接。可靠恢复不能只靠 Pub/Sub，还要保留数据库事件或最终状态查询。

## Q14：容器部署有哪些持久化重点？

**参考回答：**

需要持久化 `/app/data` 中的 SQLite 和媒体，保护 JWT / Provider Secret，并保证 FFmpeg 字体和编码环境。多实例不应共享 SQLite 文件和容器本地媒体，而应迁 PostgreSQL 和对象存储。仓库的 PostgreSQL compose 只启动数据库，不是完整应用和模型集群。

---

# 第三轮：数据库与检查点

## Q15：SQLite 和 PostgreSQL 如何切换？

**参考回答：**

默认 `DB_DRIVER=sqlite`，通过 better-sqlite3 访问本地文件；设置 `DB_DRIVER=pg` 和 `DATABASE_URL` 后使用 `PgDriver`。业务 Repository 依赖异步 `DbDriver` 接口，SQL 占位符由方言层转换。迁移风险在于历史代码可能仍直接调用同步 SQLite API，需要逐路径收口和集成测试。

## Q16：项目资产表承担了什么职责？

**参考回答：**

它保存 plan、script、style bible、角色、场景、分镜、视频、timeline、final video、review 和质量报告等阶段产物。它既是用户可见资产，也是流水线检查点，支持版本、确认、stale 和 URL 等语义。

## Q17：断点续跑怎么实现？

**参考回答：**

启动时从项目资产重建 plan、script、角色场景、分镜计划、已渲染分镜、已生成 clip、剪辑结果和 review。每到一个阶段先判断检查点是否可用；图片和视频按 shot ID 找缺失项，只生成 pending 列表，然后与已有结果合并。

## Q18：如何定义“幂等”？

**参考回答：**

当前主要是业务资产幂等：同一阶段或镜头有可用资产就复用。更严格的外部幂等还应保存 Provider request ID、输入哈希、模型版本和幂等键，避免 submit 已成功但响应丢失后重复计费。

## Q19：如果资产生成成功但写库失败怎么办？

**参考回答：**

这是最难的双写窗口。当前恢复可能把它视为缺失并重做。改进方案是调用前先创建 generation attempt 记录，保存 idempotency key；拿到远端 task ID 立即落库；完成后以事务关联资产；恢复时先查询远端任务或临时对象，而不是直接重新生成。

---

# 第四轮：Agent 与编排

## Q20：`HybridOrchestrator` 的定位是什么？

**参考回答：**

它是一次项目生产的有状态领域总控，维护用户、项目、语言、风格、画幅、参考图和 Agent 状态，为各岗位构建 Prompt，调用文本和媒体 Provider，并管理连续性与返工。`runCreatePipeline` 决定应用流程和持久化，orchestrator 决定每个岗位怎样产出。

## Q21：Agent 之间如何通信？

**参考回答：**

不通过自由聊天，而是通过 `DirectorPlan`、`Script`、角色场景结果、`Storyboard[]`、`VideoClip[]` 和 `EditResult` 等结构化对象交接。共享约束还保存在 orchestrator 上，并把关键结果落项目资产用于恢复。

## Q22：为什么说它不是 LangGraph Multi-Agent？

**参考回答：**

主链由普通 TypeScript 函数显式调用，绝大多数下一步预先确定；没有 LangGraph Graph State、checkpoint saver 或消息式 Supervisor 动态路由。项目确实有“多个岗位 Agent”和自研 DAG，但本质是类型化的确定性流水线。

## Q23：这种确定性编排有什么优势？

**参考回答：**

调用路径可预测、成本上界容易控制、阶段可持久化、错误可定位、人工关卡容易插入，特别适合视频生成这种昂贵工具。缺点是灵活性较弱，新增分支往往要改流程代码或 DAG Runner。

## Q24：角色和场景为什么能并行？

**参考回答：**

两者都依赖剧本 / 导演方案，但相互不依赖，所以在自动模式下用 `Promise.all` 降低总延迟。启用人工关卡时可能串行，以便逐阶段确认。并行前提是不会共享可变 Provider 会话或破坏上下文。

## Q25：风格一致性如何维护？

**参考回答：**

先生成 Style Bible / Key Art，再把它与角色参考、场景参考、锁定角色、Seed 图等注入分镜和视频请求。Prompt 中还重复传递色板、服装、光照和世界观约束。仅靠一句风格词无法保证跨镜头一致，因此参考资产比聊天记忆更重要。

## Q26：为什么分镜规划和渲染要拆开？

**参考回答：**

文本分镜便宜、修改快，图片渲染更慢且有费用。拆开后可以先确认镜头叙事，再批量生成图片；恢复时也能只补缺图镜头。这是“先低成本决策，再高成本执行”的治理原则。

## Q27：导演审核怎么形成闭环？

**参考回答：**

Director 前置生成 plan，末尾 `runDirectorReview` 对剧本、视频和 editResult 做检查。若不通过，`executeReviewFeedback` 将意见映射到目标分镜 / clip，调用单镜头重生等能力，再进行二次审核。审核轮数有限，防止自治循环导致成本失控。

## Q28：人工关卡的价值是什么？

**参考回答：**

它让用户在剧本、角色或分镜阶段确认后才进入更昂贵的图片和视频环节，核心价值是方向控制和成本止损。生产上关卡等待状态应持久化，跨进程事件只能作为唤醒手段。

## Q29：如何继续拆分四千多行的 orchestrator？

**参考回答：**

保留一个薄 Facade，把各岗位拆为 Role Agent，把重试和成本收口到 Model Gateway，把角色 / 风格一致性拆为 Continuity Service，把审核返工拆为 Repair Planner，把 FFmpeg 与媒体下载放到 Media Service。Writer 和 Editor 已经出现独立文件，是合理方向。

---

# 第五轮：DAG 工作流

## Q30：自研 DAG 如何建模？

**参考回答：**

节点包含 id、kind、dependsOn 和 config，kind 覆盖导演、编剧、风格、角色、场景、分镜、视频、剪辑、制片和 custom。图在执行前检查重复 ID、未知 kind、缺失依赖、自依赖和环。

## Q31：拓扑排序怎么做？

**参考回答：**

使用 Kahn 算法：计算入度，取全部入度 0 节点作为一层，移除它们的出边，再生成下一层。最终处理节点数不足说明有环。复杂度是 `O(V+E)`。

## Q32：为什么要返回拓扑层，而不是一个线性序列？

**参考回答：**

同层节点彼此无依赖，可以并行。例如 Writer 与 Style Bible、Character 与 Scene。引擎层间串行、层内 `Promise.all`，既保证依赖又降低延迟。

## Q33：工作流 Runner Registry 有什么用？

**参考回答：**

图只声明做什么，Runner 决定怎么做。它使引擎核心与业务实现解耦，也支持 dry-run 与真实运行。真实运行时用单次 Runner 绑定当前 `HybridOrchestrator`，避免把携带用户状态的实例放在全局注册表造成并发污染。

## Q34：节点失败如何处理？

**参考回答：**

支持 abort 或 continue。依赖失败的下游会跳过；continue 时不相关分支仍可执行。真实成片通常用 abort，因为缺核心视觉资产继续调用视频模型既贵又没有意义。

## Q35：DAG 能替代 `runCreatePipeline` 吗？

**参考回答：**

当前不能完全替代。DAG 已有拓扑和 Runner，但完整主链在检查点、人工关卡、镜头补跑、资产保存和复审上更成熟。DAG 目前更适合可视化实验和能力组合，未来可逐步吸收主链状态机能力。

## Q36：要把 DAG 做成生产级还缺什么？

**参考回答：**

需要持久 run / node attempt、幂等、超时重试、人工审批等待、版本快照、跨进程租约、条件边、资源队列、成本预算和 Trace。不能只有一个内存 Promise 调度器。

---

# 第六轮：AI Provider 与模型治理

## Q37：多 LLM 路由怎么做？

**参考回答：**

先构建主模型、同网关备选、OpenRouter 和 MiniMax fallback 的候选链。调用时跳过健康冷却中的端点，对 429、5xx 和网络超时有限重试，清理部分模型的 think 标签并解析 JSON。余额不足按 host 冷却，避免连续无效请求。

## Q38：Provider Registry 的设计模式是什么？

**参考回答：**

插件 / 策略模式。每个 Provider 声明 id、priority、supports、availability 和 generate；注册表自动发现并过滤不可用项，按指定 Provider 或优先级选择，再将不同厂商结果归一化。业务流程依赖统一契约，不直接耦合厂商 SDK。

## Q39：所有视频 Provider 真能无差别切换吗？

**参考回答：**

不能。它们在文生 / 图生、首尾帧、时长、分辨率、多主体、异步协议和成本上不同。统一接口只能抽象公共部分，仍需 capability 判断和请求适配；降级时还可能牺牲某些能力。

## Q40：如何分类 Provider 错误？

**参考回答：**

429、5xx、网络超时属于瞬时错误，可以退避重试；无效 Key、欠费、模型不存在、能力不支持属于致命错误，应快速失败并暂时摘除 Provider。分类错误能避免对不可恢复问题浪费重试。

## Q41：为什么要 Mock Provider？

**参考回答：**

真实图片和视频 API 慢、贵且不稳定，无法作为每次 CI 的依赖。Mock 提供确定性占位资产，用于验证认证、SSE、队列、检查点、DAG 和 UI。它只能证明编排正确，不能证明真实模型质量。

## Q42：如何实现全本地 AI？

**参考回答：**

LLM 可部署 XVERSE 为 OpenAI-compatible 服务，图片接 ComfyUI，视频接自托管 LTX，Lip-sync 接 Wav2Lip，本地保留 FFmpeg；还需要补本地 TTS、GPU 队列、模型卷和监控。容器访问宿主机服务要用正确网络地址，不能默认 `localhost`。ComfyUI 若要按分镜草图硬锁构图，还需额外装 ControlNet（Canny）模型，见下一题。

## Q42a：文档里的「可选 ControlNet（Canny）按分镜草图硬锁构图」是什么意思？

**参考回答：**

ControlNet 是挂在扩散模型旁边的辅助网络：参考图经预处理器抽出空间条件（边缘、姿态、深度等），再注入生成过程，让出图按构图/姿态走，而不是只听 Prompt。Canny 属于边缘线条类，相当于描线填色——锁构图、机位、人物站位，不锁颜色和画风。短剧里分镜图常作为图生视频首帧，构图一漂后面整镜都会偏。原理通俗解释见 Rocky Ding：[深入浅出完整解析 ControlNet 核心基础知识](https://zhuanlan.zhihu.com/p/660924126)。

结构上是锁定副本（冻住底模型）+ 可训练副本（学空间条件）+ 零卷积（训练初期不打扰底模型）。它比把草图当参考图更硬：软参考是口头说“请跟构图”，模型可以不听；ControlNet 用边缘图改 conditioning，线条位置更难被改掉。也不要和 IP-Adapter 混：IP-Adapter 锁“这人长什么样”，ControlNet 锁“人站在画面哪里”；二者可叠加。

项目口径是**配置后可用 / 预备态**，不是默认路径。需要 `COMFYUI_ENABLED`、`COMFYUI_CONTROLNET_MODEL`、自托管 ComfyUI 和 `comfyui_controlnet_aux`。有草图时优先硬锁，失败回落 IP-Adapter 再回落云端；未配置时草图只当普通参考图，行为与升级前一致。代码在 `lib/storyboard-sketch.ts` 和 `services/comfyui.service.ts` 的 `buildControlNetWorkflow`。

## Q43：为什么要统一 Model Gateway？

**参考回答：**

目前公共 LLM Client 与 orchestrator 内部调用逻辑有重复。统一后可集中管理重试、熔断、预算、Trace、Prompt 哈希、模型版本、用量和 Secret，也能避免不同岗位策略漂移。

---

# 第七轮：媒体工程

## Q44：为什么最终还需要 FFmpeg？

**参考回答：**

AI Provider 生成的是概率性素材，最终交付需要确定性操作：统一编码、分辨率和帧率，拼接镜头、转场、字幕、音频混合和响度标准化。FFmpeg 是把“素材”变成“可播放成片”的最后执行层。

## Q45：怎么判断生成的视频是真的可用？

**参考回答：**

不能只检查 URL 或 HTTP 200，要下载后用 ffprobe 验证容器、编码、时长、音视频轨和可解码性，并检查文件大小、比例和分辨率。损坏素材应在进入最终拼接前被隔离或降级。

## Q46：TTS 与字幕如何对齐？

**参考回答：**

剧本提供角色、对白和顺序，TTS 生成音频后获取或探测实际时长，再构建时间线和字幕区间；镜头过短时可调整停留时长、语速或剪辑。不能只按字符数硬估算，否则容易音画错位。

## Q47：对口型有哪些降级策略？

**参考回答：**

优先使用支持当前输入类型的 Lip-sync Provider；外部 / 自托管视频口型失败时，可保留原视频只混 TTS，或对静态脸使用 local-2d。降级必须写质量事件，因为“有声音”不等于“口型同步成功”。

---

# 第八轮：安全、成本和可观测性

## Q48：Prompt Injection 怎么防？

**参考回答：**

入口做产品范围、注入模式、有害内容和 PII 处理，再给岗位 System Prompt 加安全前缀；模型输出还要结构校验。规则只是一层，生产还需媒体输出审核、权限隔离和出站工具白名单。

## Q49：预算如何控制？

**参考回答：**

调用前根据镜头和 Provider 粗估成本，与用户当月已花费和硬上限比较，超预算直接拒绝；执行中把真实 cost event 写入 `cost_log`，事后按 LLM、图片、视频、TTS、Lip-sync 分类归因。估算负责前置决策，真实记账负责账单和分析。

## Q50：如何避免重复计费？

**参考回答：**

已有资产检查点能避免大部分全量重跑。进一步要持久化 Provider task ID、请求幂等键、Prompt 哈希和 attempt 状态；超时后先查询远端，不立即重新 submit；统一各层重试责任，避免客户端、Provider 和 Job 三层乘法放大。

## Q51：如何做可观测性？

**参考回答：**

以 projectId / traceId 串联 job attempt、pipeline step、Provider call 和 asset。记录模型、耗时、重试、远端 request ID、费用、输入摘要哈希、输出资产和质量事件；SSE 提供用户体验，Sentry / 日志提供错误，任务表和成本表提供事实。Prompt 与 Key 要脱敏。

## Q52：如何衡量这个系统是否变好？

**参考回答：**

技术指标看各阶段 P50/P95 延迟、Provider 成功率、重试率、恢复率和单片成本；质量指标看角色一致性、可解码率、审核一次通过率和返工镜头数；产品指标看成片完成率、用户修改率、导出率和复用率。不能只看模型调用成功率。

## Q53：多租户安全最容易漏在哪里？

**参考回答：**

JWT 只确认身份，最容易漏的是按 projectId / workflowId 查询时没有 owner 条件、SSE 频道未校验订阅者、静态媒体 URL 绕过权限和管理员接口只检查登录。需要逐路由对象级授权测试。

---

# 第九轮：故障题与系统设计题

## Q54：浏览器关闭后任务怎么办？

**参考回答：**

队列模式下任务由 Worker 独立执行，浏览器关闭不影响；重连后通过任务事件和项目状态恢复 UI。默认请求内模式对连接更敏感，但已经持久化的资产可以用于再次续跑。

## Q55：生成到第七个镜头时服务重启怎么办？

**参考回答：**

Worker 心跳停止，重启后 instrumentation 启动 Worker，孤儿扫描把任务重新 queued。新执行实例从资产加载前六个成功镜头，只把第七个及后续缺失项加入 pending，避免重做和重复付费。

## Q56：某 Provider 连续返回 429 怎么办？

**参考回答：**

在 Provider 层识别 429，按 Retry-After 或指数退避，限制重试次数并更新健康状态，之后切备用 Provider。任务层不要立刻无脑全量重跑；还应按用户 / Provider 做并发限流，避免队列持续冲击。

## Q57：两个应用实例怎么保证任务不双跑？

**参考回答：**

共享 PostgreSQL，claim 使用原子条件更新或 `SKIP LOCKED`；running 状态附带 worker / lease token 和 heartbeat，只有持有租约者能完成。事件使用 Redis / Outbox 跨实例传播，本地文件迁对象存储。SQLite + EventEmitter 不适合这个场景。

## Q58：如何设计更可靠的 Provider 调用表？

**参考回答：**

建立 generation_attempt：id、project、stage、shot、provider、model、input_hash、idempotency_key、remote_request_id、state、attempt、cost、started/heartbeat/completed、error。先落 pending，再 submit 并保存远端 ID；回调 / polling 更新状态；资产表引用成功 attempt。这样支持恢复、对账和审计。

## Q59：如何做灰度模型切换？

**参考回答：**

把模型配置版本化，按用户或项目做稳定哈希分流，记录每次实际 Provider / model / prompt version。比较成功率、成本、延迟、导演审核和用户接受率；异常时通过配置回滚，不修改历史运行快照。

## Q60：如果让你三个月生产化，你怎么排期？

**参考回答：**

第一阶段做安全和数据底座：对象级授权、Secret、对象存储、备份和 Provider 请求审计。第二阶段拆 Web / Worker、完成 PostgreSQL 和多实例事件、压测队列与 FFmpeg。第三阶段统一 Model Gateway、幂等对账、可观测性和质量评测，再基于真实数据优化路由，不会先大规模重写 Agent 框架。

---

# 第十轮：反问与深挖

## Q61：这个项目最值得重构的地方是什么？

**参考回答：**

一是 `HybridOrchestrator` 过大，应拆 Role Agent、Continuity、Repair Planner 和 Model Gateway；二是统一公共 LLM Client 与内部调用；三是把队列 Worker 从 Web 部署拆出；四是把外部 request ID 和幂等状态持久化。重构要保持现有检查点资产兼容。

## Q62：为什么不直接换成 LangGraph？

**参考回答：**

LangGraph 可增强状态图、节点恢复和动态路由，但迁移成本不只在画图，还要映射现有资产、SSE、人工关卡、Provider 和镜头级恢复。当前确定性主链对高成本媒体任务是合理的；若动态分支和审批明显增加，可以先把自研 DAG 状态持久化，再评估局部引入，而不是为了框架重写。

## Q63：这个系统里 AI Agent 真正创造的价值是什么？

**参考回答：**

价值不是“用了八个 Agent”这个标签，而是把影视岗位知识转成阶段 Prompt、结构化契约和质量闭环，让用户能从一句创意得到可编辑中间资产和成片。工程上则把概率性模型调用变成可追踪、可恢复、可定点返工的生产过程。

## Q64：项目最大的业务风险是什么？

**参考回答：**

单位成片成本和质量稳定性。如果角色漂移、视频失败或返工率高，Provider 成本会吞噬毛利；如果过度使用低价 fallback，用户又不愿导出。需要用镜头级质量、成本归因和用户接受率共同优化，而不是只追求模型参数。

---

# 第十一轮：业务、数据与生产事故追问

## Q65：这个产品真正的核心业务对象是什么？

**参考回答：**

不是一次模型调用，而是 Project 和它的版本化资产。项目聚合剧本、角色、场景、分镜、视频、时间线、成本、审核和发布；Project Asset 既是用户可编辑产物，也是检查点。模型可以替换，资产才能复用、返工和交付。

## Q66：为什么试拍第一镜有业务价值？

**参考回答：**

试拍把画风和角色方向错误暴露在全量图片、视频调用之前。用户接受后，试拍图还能复用为第一镜和全片风格 Seed，减少一次生成费用并提高预期一致性。它本质是低成本决策 Gate。

## Q67：单集和系列的模型有什么差异？

**参考回答：**

每一集仍是独立 Project，通过 series_id 和 episode_number 归组；series_anchors 保存画风、角色和上集尾帧。第 N 集还注入前集 recap。这样复用单集流水线，但当前“记忆”仍偏摘要和视觉锚，不是完整剧情知识图谱。

## Q68：模板市场和全局资产库的区别是什么？

**参考回答：**

模板封装一次起片所需的风格、题材、节奏和 payload；全局资产是角色、场景、风格、道具等可跨项目复用的原子资产。模板可以引用资产，但两者生命周期和授权不同。

## Q69：Cameo IP 是完整版权交易平台吗？

**参考回答：**

不是。它已有角色 Token、可见性、view/remix/commercial 许可、建议版税和授权审批记录，但没有完整结算、合同、税务和争议处理。准确说法是角色 IP 授权与使用记录地基。

## Q70：发布流程为什么要区分 packaged、manual 和 published？

**参考回答：**

packaged 只表示发布素材包已生成；manual 表示平台无公开 API 或未配置凭据，需要用户手动上传；published 才表示真实上传成功。若混为一谈，系统会向用户伪报发布成功，也无法做真实转化统计。

## Q71：数据库中最重要的技术债是什么？

**参考回答：**

第一是 PostgreSQL 驱动已经存在，但不少核心路径仍直接读取 SQLite，可能混读；第二是项目字段和资产存在双事实源；第三是 project_assets 同时承担当前资产 upsert 和重录历史 append 两种语义。优先级高于普通表结构美化。

## Q72：为什么 `project_assets` 不能简单加一个唯一索引？

**参考回答：**

分镜和视频希望 `(project,type,shot)` 唯一，但 voice-retake 等历史故意允许同镜多条。如果加全表唯一约束会破坏追加历史。正确方向是拆当前资产与 asset revision / generation attempt。

## Q73：怎么发现 SQLite / PG 混读？

**参考回答：**

测试时只向 PG 写数据并保持 SQLite 空，运行项目详情、检查点、发布和质量接口。若接口查不到项目或拼出不一致结果，说明存在裸 `db.prepare`。两个库数据完全相同反而容易掩盖问题。

## Q74：项目状态为什么需要正式状态机？

**参考回答：**

当前多个模块直接写 draft、active、completed、failed、archived 等字符串，可能出现非法跳转或 UI 与 Job 不一致。应集中定义 transition、actor、reason 和时间，例如 running 不能被普通 PATCH 直接改 completed。

## Q75：如果 finalVideoUrl 非空，能否判断出片成功？

**参考回答：**

不能。Editor 最低降级可能返回首个有效 clip。还要检查 videoCount、实际时长、音轨、hasVoiceover、hasBgm、degradedShots、audioWarnings 和 ffprobe。技术完成与可交付质量必须分级。

## Q76：为什么需要 Delivery Grade？

**参考回答：**

项目大量采用诚实降级：fallbackScript、Ken Burns、无口型、简单拼接、手动发布包。只有 completed 无法表达质量。A/B/C/D 等交付等级能区分全部真实能力成功、轻降级可发布、需人工确认和仅流程占位。

## Q77：预算 Guard 和限流有什么不同？

**参考回答：**

预算限制金额，限流限制单位时间和并发。一个用户可能仍在预算内，却一秒提交 100 个任务压垮 Worker 和 Provider。因此需要用户速率、Provider 并发、项目互斥和队列长度限制。

## Q78：为什么模型调用应该有 reservation？

**参考回答：**

只在开始前估算，多个并发任务都可能看到同一剩余额度并同时放行。Reservation 先冻结预计金额，attempt 完成后按真实费用结算和释放，能防止并发穿透预算。

## Q79：Web 和 Worker 为什么要拆部署？

**参考回答：**

FFmpeg、下载和视频任务会占 CPU、内存和长连接；与 Web 同进程时滚动发布会中断任务，扩 Web 副本也会意外增加 Worker。拆部署后可以独立扩容、限流和优雅停机，同时继续共享代码与数据库。

## Q80：Redis 是否能保证任务可靠？

**参考回答：**

当前 Redis 主要桥接 Event Bus，解决跨实例实时通知和 Gate 放行，不替代数据库 Job、事件回放和资产检查点。Pub/Sub 消息可能瞬时丢失，最终事实仍在 DB。

## Q81：Agent 契约中最值得重构的字段是什么？

**参考回答：**

Character Designer 生产 `character`，消费方却访问可选 name；Review targetRole 又放宽为 string；Gate editedData 是 any。这些都是边界漂移。应先对公开阶段输出做运行时 Schema，而不是先清理所有内部变量。

## Q82：Micro-beat 比普通 Prompt 好在哪里？

**参考回答：**

它把一个镜头拆成带起止秒的动作、机位、对白、表情和速度段，减少视频模型自行补全动作造成的漂移。还可以校验各段连续且总时长匹配镜头。

## Q83：为什么 `cut` 和 `continuous` 会影响视频生成？

**参考回答：**

continuous 表示同场动作延续，适合把上一镜真实尾帧传给下一镜；cut 表示换场或硬切，应使用自己的分镜。所有镜头都链尾帧会把上一场景污染到新场景。

## Q84：如何避免 TTS 把对白截断？

**参考回答：**

不能只按字数估时。要生成后获取真实音频时长，再与镜头实际时长和转场重叠比较，必要时调整语速、延长镜头或给出警告。角色音色、情绪韵律和时间适配应分开处理。

## Q85：为什么生产镜像不应包含测试依赖？

**参考回答：**

当前 runner 复制完整 node_modules，Vitest、Playwright 等 devDependencies 也进入镜像，增加体积和漏洞面。应在 builder / CI 运行测试，runner 使用 standalone 或 production prune。

## Q86：怎么证明 PostgreSQL 真的可用？

**参考回答：**

不能只跑建表或单个 Repository smoke。要用空 SQLite + PG 数据跑完整创建、恢复、项目详情、协作、发布和删除，并做事务、并发 claim、asset upsert 和 JSON/BLOB 测试。

## Q87：如何设计 Agent 的运行时 Schema？

**参考回答：**

每个阶段定义 input/output Schema 和 schemaVersion，模型输出先 parse、normalize、validate，再进入下游；fallback 也必须通过同一 Schema。资产保存时附 Provider、model、Prompt version、input hash 和 generatedBy。

## Q88：发生线上重复计费怎么排查？

**参考回答：**

按 project/shot 查 Job attempts、Provider submit 日志、remote task ID、资产写入时间和 cost_log。重点找“远端受理但本地未保存 task ID”的窗口，以及 client/provider/job 多层重试。没有 generation attempt 账本时对账会很困难。

## Q89：这个项目应该优先加更多模型还是修基础设施？

**参考回答：**

当前 Provider 已很多，优先修数据单一事实源、PG 混读、外部幂等、Web/Worker 分离、对象存储和降级透明。这些直接提高可用成片率和成本可控性；继续加模型会扩大集成和故障面。

## Q90：如果只能选一个北极星指标，你选什么？

**参考回答：**

我会选“单位可交付成片成本”，它不是最低生成成本，而是总模型与人工成本除以真正通过质量门禁、被用户接受并导出的成片数。它同时约束成功率、质量、返工和费用。

---

## 十二、30 秒 / 60 秒速答卡

### 30 秒项目介绍

> Wind Comic 是一个 Next.js AI 漫剧生产平台，核心用 `runCreatePipeline` 和 `HybridOrchestrator` 把导演、编剧、角色场景、分镜、视频、剪辑和复审串成类型化流水线。系统通过多 Provider 接入不同模态，每阶段结果落项目资产，因此支持 SSE 进度、数据库任务重试、断点续跑和镜头级返工。它不是自由聊天式 Multi-Agent，而是面向高成本媒体生产的确定性 Agent 编排。

### 60 秒架构介绍

> 接口层先完成 JWT 鉴权、输入安全和预算预估，然后默认直接运行流水线，或在 `PIPELINE_QUEUE=1` 时写数据库任务让 Worker 执行。应用层 `runCreatePipeline` 管阶段顺序、资产持久化、恢复和 SSE；领域层 `HybridOrchestrator` 管 Prompt、角色风格连续性、Provider 选择和审核返工；能力层分别接 LLM、图片、视频、TTS、Lip-sync 和 FFmpeg。项目默认 SQLite，可切 PostgreSQL。视频按镜头保存，重启后从资产检查点恢复，只补缺失镜头。当前应用与数据可本地，但真实模型默认多为外部，仓库只是提供了 XVERSE、ComfyUI、LTX 等本地接入点。

### 30 秒可靠性

> 长任务用数据库状态机管理 queued、running、done 和 failed，Worker 有心跳、孤儿恢复、三次重试和死信；每个阶段和镜头都保存资产，所以 Job 重试不是全量重跑。Provider 层做超时、错误分类、健康冷却和降级，最终视频再由 ffprobe / FFmpeg 验证。SSE 只负责实时体验，数据库和资产才是事实来源。

### 30 秒 DAG

> 可配置工作流由节点 kind、dependsOn 和 config 组成，执行前校验重复 ID、缺失依赖和环，用 Kahn 算法生成拓扑层，层间串行、层内 Promise.all 并行。Runner Registry 解耦图与 Agent 实现，支持 dry-run 和绑定当前 orchestrator 的 real-run。当前它更适合流程实验，完整检查点仍由固定成片主链负责。

---

## 十三、面试时的高频误区

| 错误说法 | 推荐说法 |
|---|---|
| “这是全本地 AI 平台” | “应用和数据本地，模型默认外部，支持部分本地 Provider” |
| “用了 LangGraph 多 Agent” | “自研确定性岗位编排 + 轻量 DAG” |
| “SSE 保证任务可靠” | “SSE 做推送，DB Job 和资产保证恢复” |
| “失败自动回滚” | “阶段提交，保留有效资产并局部重算” |
| “Provider 可以无缝替换” | “公共契约统一，但能力、质量、成本仍需适配” |
| “有 JWT，所以多租户安全” | “JWT 身份 + 每个资源的对象级授权” |
| “PostgreSQL 已完全生产验证” | “有双驱动和 Schema，仍需清理旧同步路径并压测” |
| “审核 Agent 会无限优化到通过” | “有限轮复审和定点返工，防止成本失控” |

---

## 十四、最后总结

回答这个项目时，最值得强调的不是模型列表，而是四个工程判断：

1. 用结构化岗位契约控制多模态生产，而不是让 Agent 自由发散。
2. 用阶段资产和镜头级检查点应对长任务与部分成功。
3. 用 Provider 路由、预算和有限返工约束外部模型成本。
4. 用 FFmpeg、任务状态机和数据库事实把概率性 AI 变成可交付系统。

只要始终守住“已实现 / 可选配置 / 未来改进”的边界，这个项目就能被讲成一个有真实工程深度的 AI 应用，而不是堆砌 Agent 名词。

## 十五、配套专题

- [项目总览](./00_WindComic项目解析总览.md)
- [后端与数据层](./01_后端架构与数据层深度解析.md)
- [AI Provider 路由](./02_AI服务与Provider路由机制.md)
- [Agent 与 HybridOrchestrator](./03_Agent体系与HybridOrchestrator编排.md)
- [DAG 工作流引擎](./04_可配置DAG工作流引擎解析.md)
- [一句话到成片调用链](./05_一句话到成片完整调用链.md)
- [可靠性、安全与成本](./06_可靠性安全成本与生产化边界.md)
- [业务域与产品流程](./08_业务域产品流程与商业逻辑深度解析.md)
- [API 与权限参考](./09_API接口认证权限与SSE协议详细参考.md)
- [数据库与资产状态机](./10_数据库表结构资产版本与状态机源码级解析.md)
- [Agent 字段级契约](./11_Agent数据契约Prompt上下文与返工机制源码级解析.md)
- [媒体与 FFmpeg 链路](./12_图片视频音频FFmpeg与存储链路深度解析.md)
- [问题与技术债清单](./13_已知问题技术债风险清单与改进方案.md)
- [部署与故障排查](./14_Docker部署全本地配置与故障排查手册.md)
- [测试与验收](./15_测试体系质量门禁与验收用例详细解析.md)
