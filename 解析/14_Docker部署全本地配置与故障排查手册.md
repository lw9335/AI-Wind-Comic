# Wind Comic Docker 部署、全本地配置与故障排查手册

## 1. 三种部署目标

| 目标 | 应用 / DB | AI 模型 | 适用场景 |
|---|---|---|---|
| 流程演示 | Docker + SQLite | Mock | 开发、CI、零费用演示 |
| 本地应用 + 云模型 | Docker + SQLite/PG | 外部 Provider | 最容易获得真实质量 |
| 私有化 / 尽量本地 | Docker + PG/对象存储 | XVERSE、ComfyUI、LTX、Wav2Lip 等 | 数据敏感、具备 GPU 运维团队 |

“Docker 本地启动”只说明应用运行位置，不说明模型推理位置。

## 2. 当前 Docker 镜像

### 构建阶段

```mermaid
flowchart LR
    DEPS["deps: npm ci + native build toolchain"] --> BUILD["builder: npm run build"]
    BUILD --> RUN["runner: Next.js + node_modules + ffmpeg + fonts"]
```

### 运行设置

- Node 20 Alpine。
- `NODE_ENV=production`。
- 端口 3100。
- 安装系统 FFmpeg 和 DejaVu 字体。
- 使用非 root `nextjs` 用户。
- `/app/data` 声明为 Volume。
- 健康检查请求首页。

### 已知点

- runner 复制完整 node_modules，包含 devDependencies，镜像并非最小化。
- 仅有 DejaVu 不一定覆盖所有 CJK 字形；真实中文字幕应确认字体。
- Dockerfile 不启动 WebSocket 协作服务。
- `docker-compose.pg.yml` 只启动 PostgreSQL，不启动完整应用和 AI 模型。

## 3. 单容器启动

概念命令：

```powershell
docker build -t wind-comic:local .
docker run --name wind-comic `
  -p 3100:3100 `
  --env-file .env.local `
  -v wind-comic-data:/app/data `
  wind-comic:local
```

不要把真实 Key 写进 Dockerfile、镜像层或提交到 Git。

## 4. 最小环境变量

### 应用安全

```dotenv
JWT_SECRET=<至少高强度随机值>
SEED_DEMO_USER=0
```

公网环境建议关闭 demo 账号。若需要 demo 用户，应显式设置 `DEMO_PASSWORD`，不要依赖日志生成的一次性密码。

### LLM

```dotenv
OPENAI_API_KEY=...
OPENAI_BASE_URL=https://.../v1
OPENAI_MODEL=...
OPENAI_CREATIVE_MODEL=...
```

这里的 OpenAI 表示兼容协议，不一定是 OpenAI 官方服务。

### 队列

```dotenv
PIPELINE_QUEUE=1
```

生产建议开启，否则浏览器连接与整片任务耦合，整季生成还可能走进程内 fire-and-forget。

## 5. 数据库配置

### SQLite

```dotenv
DB_DRIVER=sqlite
```

必须挂载 `/app/data`。适合单实例；不要让多个容器同时写共享 SQLite 文件。

### PostgreSQL

```dotenv
DB_DRIVER=pg
DATABASE_URL=postgresql://user:password@postgres:5432/windcomic
```

切换前：

1. 执行迁移 / 校验脚本。
2. 验证核心表与索引。
3. 用 PG 数据、空 SQLite 做接口测试，捕获混读。
4. 特别测试项目详情、检查点、发布、质量和协作路径。

`npm run pg:migrate` 会从当前 SQLite Schema 动态导出 PostgreSQL DDL，并覆盖 `db/schema.pg.sql`；仓库中已有的静态 SQL 文件可能不是最新运行时结构，不能跳过动态导出与表清单核对。

## 6. 存储配置

### 本地

- 确保 Volume 可写。
- 容器用户 UID/GID 有权限。
- 备份数据库和媒体。
- 监控磁盘空间和临时文件。

### S3 / 对象存储

按项目支持的存储变量配置 endpoint、bucket、region、credentials。上线前运行 `npm run s3:smoke` 的等效验证。

策略建议：

- 多实例必须以对象存储为主。
- 数据库保存 object key，不只保存短期签名 URL。
- 上传失败不能在多实例生产中静默退回单实例本地。

## 7. 外部 Provider 配置族

| 模态 | 典型变量族 |
|---|---|
| LLM | `OPENAI_*`、`OPENROUTER_*`、`MINIMAX_*`、`XVERSE_*` |
| 图片 | `OPENAI_IMAGE_*`、`GEMINI_*`、Midjourney、MiniMax、`COMFYUI_*` |
| 视频 | `KELING_*`、`VIDU_*`、`VEO_*`、`LTX_*`、MiniMax 等 |
| TTS | MiniMax、VectorEngine 等 |
| Lip-sync | `LIPSYNC_*`、Sync.so、Hailuo 等 |
| 跨实例事件 | `REDIS_URL` |
| Stripe | Secret、Webhook Secret、Price / Portal 配置 |

以 `.env.example` 和实际 Provider 文件为准，不要凭 README 旧模型名配置。

## 8. Mock 模式

```dotenv
MOCK_ENGINES=1
PIPELINE_QUEUE=1
```

Mock 模式应做到：

- 不调用任何外网模型；
- 输出确定性图片 / 视频 / TTS 占位；
- 可验证认证、预算、队列、SSE、检查点和 UI；
- 不用于判断模型质量、Lip-sync 或真实成本。

测试时仍应提供测试专用 JWT Secret，避免启动安全逻辑差异。

## 9. 本地 XVERSE

宿主机启动 OpenAI-compatible XVERSE 服务后：

```dotenv
XVERSE_ENABLED=true
XVERSE_FALLBACK=true
XVERSE_BASE_URL=http://host.docker.internal:8000/v1
XVERSE_MODEL=xverse/XVERSE-Ent-A5.7B
XVERSE_FAST_MODEL=xverse/XVERSE-Ent-A4.2B
```

### 网络说明

- 应用和模型在同一 Compose 网络：使用服务名，如 `http://xverse:8000/v1`。
- 模型在 Windows 宿主机、应用在 Docker Desktop：通常用 `host.docker.internal`。
- 容器内 `localhost` 只指当前容器。

### 验证

1. 从容器内部访问 `/v1/models` 或兼容接口。
2. 调 `/api/test-llm`，但必须登录且注意会使用真实模型。
3. 看 Provider 健康页和实际调用日志。
4. 确认 Writer / Director 真实选择 XVERSE，而不是悄悄回退云模型。

## 10. 本地 ComfyUI

ComfyUI 是可选的本地图片工作流引擎，不是默认出图路径。Wind Comic 通过它做角色 IP-Adapter 一致性和可选的 ControlNet 草图硬锁。ControlNet 不是又一个云厂商：它是挂在扩散模型旁的辅助网络，用预处理器（本项目用 Canny）从分镜草图抽出边缘条件图，硬锁构图、机位和主体位置，比把草图当普通参考图更硬；未配置 `COMFYUI_CONTROLNET_MODEL` 时不会走这条路径。IP-Adapter 用冻结 CLIP 抽角色参考图特征，再经解耦交叉注意力锁身份。原理参考：[ControlNet](https://zhuanlan.zhihu.com/p/660924126)、[IP-Adapter](https://zhuanlan.zhihu.com/p/3472288872)、[CLIP](https://blog.csdn.net/weixin_47228643/article/details/136690837)；项目口径见 [00 总览 2.1](./00_WindComic项目解析总览.md) 和 [02 图片 Provider](./02_AI服务与Provider路由机制.md)。

```dotenv
COMFYUI_ENABLED=true
COMFYUI_URL=http://host.docker.internal:8188
```

需要额外确认：

- Workflow 是否部署。
- 模型、LoRA、ControlNet 名称匹配。
- 输出目录可被 API 访问或结果能回传。
- 并发和显存足够。
- ComfyUI 失败时是否允许回退云端；私有化环境应禁用外网 fallback。

## 11. 本地视频与 Lip-sync

LTX 可通过 `LTX_BASE_URL` 指向自托管服务；Lip-sync 可通过 `LIPSYNC_API_URL` 指向 Wav2Lip 类 HTTP 服务。

### GPU 容量

全本地的瓶颈通常不是 Next.js，而是：

- 模型权重磁盘；
- GPU 显存；
- 同时加载图片、视频、LLM 的显存碎片；
- 视频推理时长；
- Worker 并发导致 OOM。

推荐各模型独立服务和队列，Next.js 只通过 HTTP 调用，不把 GPU 推理塞入 Web 容器。

## 12. 本地 TTS 缺口

当前开箱 Provider 更偏外部 TTS 和 Mock。要实现高质量全本地，需要额外适配本地 TTS，并提供：

- 多角色音色；
- 中文及目标语言；
- 情绪 / 语速；
- 返回真实时长；
- 商用模型授权；
- 声音克隆同意日志。

没有这一步，所谓“全本地”可能只能生成无声视频或测试占位音频。

## 13. Redis 事件桥

```dotenv
REDIS_URL=redis://redis:6379
```

Event Bus 使用本地 EventEmitter，并在配置 Redis 时跨实例 Pub/Sub。Redis 只解决实时传播，不替代：

- 数据库任务状态；
- pipeline_job_events 回放；
- 人工 Gate 的持久等待记录；
- Exactly-once 业务语义。

## 14. WebSocket 协作

实时 Yjs 协作需额外启动：

```powershell
npm run dev:ws
```

生产应独立容器化 `scripts/ws-server.mjs`，配置：

- WebSocket 地址；
- JWT / 项目权限验证；
- 粘性会话或共享 Yjs 持久化；
- 反向代理 Upgrade；
- 连接和文档大小限制。

只启动 Next.js 容器不代表多人协作完整可用。

## 15. 反向代理

SSE 和长视频上传需要：

- 关闭代理缓冲；
- 增大读取超时；
- 允许 chunked response；
- 设置合理 Body 大小；
- WebSocket Upgrade；
- 正确传递 `X-Forwarded-Proto`；
- HTTPS。

若使用请求内生成模式，代理超时必须覆盖完整流水线；更推荐队列模式降低此依赖。

## 16. 健康检查分层

| 检查 | 证明什么 | 不证明什么 |
|---|---|---|
| `/` 返回 200 | Next 进程活着 | DB / 模型 / FFmpeg 正常 |
| runtime readiness | 必需配置和运行依赖 | Provider 一定可生成 |
| provider health | Key / 网络 / 模型状态 | 每个请求质量 |
| DB smoke | 数据库可读写 | 全 API 无混读 |
| ffmpeg smoke | 可执行与编码器可用 | 所有滤镜 / 字体可用 |
| end-to-end Mock | 编排可运行 | 真实模型质量 |
| 小额真实 smoke | 关键 Provider 可用 | 高并发稳定性 |

## 17. 启动后检查清单

1. `docker ps` 显示 healthy。
2. 首页与 `/api/runtime/readiness` 可访问。
3. 登录成功，Cookie 安全属性正确。
4. 新项目写入正确数据库。
5. `/app/data` 重启后仍保留。
6. FFmpeg / ffprobe 可执行。
7. CJK 字幕字体存在。
8. Provider Health 与预期配置一致。
9. Mock 创建能完整到 complete。
10. 队列任务断开浏览器后仍继续。

## 18. 故障：登录页提示 DEMO_PASSWORD

原因：页面展示 demo 邮箱，但密码由环境变量提供，仓库不再内置明文密码。

处理：

- 查看部署者提供的 `DEMO_PASSWORD`；
- 或注册新账号；
- 公网环境关闭 demo seed。

这不表示应用依赖外部登录服务，认证和用户库仍可本地。

## 19. 故障：容器健康但生成失败

按层排查：

```mermaid
flowchart TD
    HEALTH["Container healthy"] --> AUTH["登录 / 预算"]
    AUTH --> DB["Project / Job write"]
    DB --> LLM["LLM config"]
    LLM --> IMG["Image provider"]
    IMG --> VID["Video provider"]
    VID --> AUDIO["TTS / Lip-sync"]
    AUDIO --> FF["FFmpeg / font / disk"]
```

首页健康只检查 Web 进程，不会检查所有 AI 服务。

## 20. 故障：容器访问不到宿主机模型

- 不要使用 `localhost`。
- 使用 `host.docker.internal` 或同 Compose 服务名。
- 检查模型是否监听 `0.0.0.0`，而不是只监听 127.0.0.1。
- 检查 Windows 防火墙。
- 从容器内部 curl / wget 目标地址。

## 21. 故障：任务一直 queued

检查：

- `PIPELINE_QUEUE=1` 是否启用；
- instrumentation 是否执行；
- Worker 日志是否有 started；
- DB_DRIVER 与 Job 所在数据库一致；
- claim SQL 是否报错；
- 当前 active 是否达到 2；
- job 是否超过 24 小时或 attempts 已耗尽。

## 22. 故障：任务一直 running

- heartbeat 是否每 15 秒更新；
- Provider 是否在无上限 polling；
- SSE 心跳不等于 Job heartbeat；
- Worker 进程是否已死但数据库未扫描；
- 90 秒后是否回到 queued；
- 是否存在外部 task ID 可恢复。

## 23. 故障：重启后项目重复镜头

检查：

- 是否是旧版本资产竞态遗留；
- project/type/shot 是否存在重复行；
- upsert 是否走同一事务；
- 查询是否按 updated_at 取最新；
- voice-retake 等追加历史不要误删。

## 24. 故障：PG 模式数据消失

高概率是某条接口仍直接读 SQLite。验证方法：

1. 检查目标 Route 是否 import `db`。
2. 比较 PG 和 `data/qfmj.db` 中同 ID 行。
3. 查 Repository 是否使用 `getDbDriver`。
4. 不要通过复制一份数据到 SQLite 掩盖问题，应修访问路径。

## 25. 故障：字幕乱码 / 缺字

- 容器检查字体列表。
- 安装 Noto Sans CJK 等许可明确字体。
- 确认 FFmpeg build 有 libass。
- 检查 ASS 文件编码为 UTF-8。
- 验证字体名与文件内部 family 一致。

## 26. 故障：磁盘持续增长

来源可能包括：

- Provider 下载文件；
- FFmpeg 临时文件；
- 多版本镜头 / voice retake；
- final exports；
- SQLite WAL；
- 未被 DB 引用的 S3 / 本地孤儿。

建立保留策略前不要粗暴递归删除 `/app/data`。应按资产引用、项目归属和创建时间做清理，并先支持 dry-run。

## 27. 备份与恢复

### SQLite

- 使用一致性备份方法，不在活跃写入时简单复制半个 WAL 状态。
- 同时备份媒体 Volume。

### PostgreSQL + S3

- 数据库做 PITR / 定期 dump。
- S3 开版本或生命周期。
- 定期校验 DB asset object key 都存在。
- 恢复演练必须验证成片可播放，不只是表行数。

## 28. 当前环境验证说明

本次分析时，名为 `wind-comic` 的容器在 3100 端口处于 healthy。仓库宿主机未安装 node_modules，因此宿主机 `npm test` 无法找到 Vitest；运行镜像虽然包含 Vitest，但没有复制 tests 目录，所以镜像内也无法执行测试。这是环境 / 镜像职责问题，不代表源码测试通过或失败。

## 29. 源码索引

- [`Dockerfile`](../Dockerfile)
- [`docker-compose.pg.yml`](../docker-compose.pg.yml)
- [`.env.example`](../.env.example)
- [`instrumentation.ts`](../instrumentation.ts)
- [`lib/event-bus.ts`](../lib/event-bus.ts)
- [`scripts/ws-server.mjs`](../scripts/ws-server.mjs)
- [`lib/storage.ts`](../lib/storage.ts)
- [`app/api/runtime/readiness/route.ts`](../app/api/runtime/readiness/route.ts)
