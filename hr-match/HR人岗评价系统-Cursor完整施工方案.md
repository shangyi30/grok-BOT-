# HR 人岗评价系统 · Cursor / Codex 完整施工方案（可直接执行）

> 把本文**与同目录《UX说明.md》《给Cursor或Codex的提示词.md》一并**交给 Cursor Agent / Codex。按顺序实现，不要改栈，不要做成文档 RAG 问答产品。页面交互以《UX说明.md》为准。
> 定稿日期：2026-09-24  
> 定位：用历史「客观背景 + HR 评价/适岗」当案例库，通过结构化近邻 + LLM few-shot（只接 API、不微调），给新候选人产出贴合指定 HR 口径的评价与适岗建议；公网可访问。

---

## 0. 一句话目标

构建公网 Web 应用：导入历史案例 → 配置风格卡与字段相似度 → 单人推断（评价+适岗+近邻依据）→ 跑批调准与回归对比。

**不是**：文档库 RAG 问答、模型微调、全套 ATS、多租户复杂权限。

---

## 1. 技术栈（锁定，勿改）

| 层 | 选型 |
|---|---|
| Monorepo | pnpm workspace |
| 前端 | Next.js App Router + TypeScript + Tailwind + shadcn/ui + TanStack Query |
| 后端 | Hono + `@hono/node-server` + Zod |
| 数据 | PostgreSQL + Prisma |
| LLM | OpenAI 兼容 SDK（`LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL`），provider 可换 |
| 近邻 | 结构化特征：`must_match` 过滤 + `weighted` 加权距离；**不建**文档 chunk / 向量库 |
| 部署 | Docker Compose：`web` + `api` + `db` + `caddy`（或 nginx）公网 HTTPS |
| 鉴权 | 邮箱密码登录；业务路由必须登录；Key 永不回前端 |

目录：

```
/
  apps/web/
  apps/api/
  packages/db/          # Prisma
  packages/shared/      # Zod DTO / 枚举
  docker-compose.yml
  Caddyfile             # 或 nginx.conf
  .env.example
  pnpm-workspace.yaml
  README.md
```

---

## 2. 产品页面与路由（App Router）

| 路由 | 用途 |
|---|---|
| `/login` | 邮箱密码登录 |
| `/cases` | 案例列表 |
| `/cases/import` | 上传 CSV/Excel → 表头映射 → 失败行预览 |
| `/config` | Tab：风格卡 \| 字段角色 \| 权重；顶栏显示 `configVersion` |
| `/infer` | 单人推断表单 |
| `/infer/[id]` | 推断结果 + 近邻对照 |
| `/batches` | 跑批列表 |
| `/batches/[id]` | 结果表（命中率/同方向率） |
| `/batches/[id]/rows/[rowId]` | 下钻近邻对照 + 金标并排 |

账号管理后置；MVP 可用 seed 脚本建第一个用户。

**主路径**：导入 → 配风格卡/字段角色 → 推断看近邻是否像 → 跑批看数 → 调权重/补样本 → 重跑对比。

---

## 3. 领域模型（Prisma）

### CaseSample（历史案例，一行一条）
- `id` String @id @default(cuid())
- `features` Json          // 客观条件：年龄、学历、年限、岗位、地域…
- `hrEvalText` String      // HR 评价原文（必填）
- `fitText` String         // 适岗定位原文（自由文本，必填）
- `fitJobs` String[]       // 可选；能拆岗位名再填
- `createdAt` DateTime
- `updatedAt` DateTime

### StyleCard
- `id`, `version` Int, `dimensions` Json, `tone` String, `bannedPhrases` String[]
- `isActive` Boolean
- `createdAt`

### SimilarityConfig
- `id`, `version` Int
- `fields` Json  // `[{ key, role: "must_match"|"weighted"|"display_only", weight?: number, enumMap?: {...} }]`
- `topN` Int @default(8)
- `isActive` Boolean

### InferenceRun
- `id`
- `inputFeatures` Json
- `status`  // done | failed（MVP 同步推断，一般直接 done）
- `evalSummary` String?
- `fitJobsTopK` Json?      // `[{ job, score }]`
- `styleCardSummary` String?
- `configVersion` String   // 建议 `"style:{v}+sim:{v}"`
- `rawLlmJson` Json?       // 可选调试
- `error` String?
- `createdAt`
- `createdByUserId`

### NeighborHit
- `id`, `runId`, `caseSampleId`, `rank` Int, `score` Float
- `keyFeatures` Json       // 参与计算的字段快照对比

### EvalRun（跑批）
- `id`, `status`, `configVersion`
- `metrics` Json?          // `{ fitHitRate, evalAgreeRate, n }`
- `createdAt`, `createdByUserId`

### EvalRow
- `id`, `evalRunId`, `caseSampleId` // 留出集金标样本
- `inferenceRunId` String?
- `fitHit` Boolean?
- `evalAgree` Boolean?     // 与金标同方向（MVP 可用简单规则或二次 LLM 判定，先实现规则/人工可改）
- `goldHrEvalText` String
- `goldFitText` String

### User / Session
- User: id, email, passwordHash, createdAt
- Session: id, userId, tokenHash, expiresAt（或用 JWT，二选一；推荐 httpOnly Cookie + 服务端 session）

---

## 4. API 契约（全部需登录，除 login）

### Auth
- `POST /auth/login` `{ email, password }` → Set-Cookie
- `POST /auth/logout`
- `GET /auth/me`

### Cases
- `GET /case-samples?q=&page=`
- `POST /case-samples` 单条
- `POST /case-samples/import`  
  Body: 文件或 `{ rows:[{ features, hrEvalText, fitText, fitJobs? }] }`  
  Resp: `{ successCount, failedRows:[{ row, reason }] }`

### Config
- `GET /style-cards/active` · `PUT /style-cards/active`（更新升 version）
- `GET /similarity-config` · `PUT /similarity-config`（更新升 version）

### Inference（同步返回完整 run）
- `POST /inference` `{ features }` →  
  `{ id, evalSummary, fitJobsTopK, styleCardSummary, configVersion, neighbors:[{ caseSampleId, rank, score, keyFeatures, hrEvalText, fitText, features }] }`
- `GET /inference/:id`

### Eval / Batches
- `POST /eval/runs` `{ holdoutCaseIds?: string[] }` 或默认随机留出  
  → 创建 EvalRun，对每条留出样本跑推断，算 metrics
- `GET /eval/runs/:id` 含 rows 摘要
- `GET /eval/runs/:id/rows/:rowId` 下钻（neighbors + 金标字段）

**前端请求**：浏览器只打同源 `/api/*`（Caddy 反代到 api），**不要**把 api 3001 暴露公网。

---

## 5. 近邻算法（必须实现）

1. 读取 active `SimilarityConfig.fields`
2. 对库中每个 CaseSample：
   - 任一 `must_match` 字段不同档 → **剔除**
   - 对 `weighted` 字段累计距离（越小越像）：
     - 数值（年龄/年限）：归一化绝对差 × weight
     - 枚举（学历/岗位/地域）：同档 0，近档中，远档大（用 enumMap）
   - `display_only` 不进公式
3. 排序取 `topN`（默认 8）写入 NeighborHit
4. **默认权重（无样例前）**：岗位 0.35，年限 0.25，学历 0.2，年龄 0.15，其余均分  
   地域/目标岗若存在：优先 `must_match`

配置改了只重跑，不改表结构。

---

## 6. LLM 推断（只 API，不微调）

Prompt 组成：
1. System：StyleCard（维度、口吻、禁用空话）+「只能依据近邻示范，禁止编造近邻中没有的事实」
2. Few-shot：Top-N 的 `features + hrEvalText + fitText`
3. User：新候选人 `features`
4. 强制 JSON Schema 输出：
```json
{
  "evalSummary": "string",
  "fitJobsTopK": [{ "job": "string", "score": 0-1 }],
  "keyFeatureNotes": ["string"]
}
```

解析失败 → InferenceRun.status=failed，返回可重试错误。  
无 LLM Key 时允许 `MOCK_LLM=1` 用近邻评价拼接假结果，README 写明切换方式。

---

## 6.5 UX 配套（必须同时阅读）

同目录《UX说明.md》为界面与状态机权威说明。实现前端时：
- 优先三屏严格按 UX 说明：`/cases/import` 导入映射、`/infer/[id]` 近邻对照、`/batches/[id]` + rows 下钻并排
- 登录/列表/配置可用简洁中后台骨架，但须满足 UX 全局布局与状态表
- **映射页只绑列角色枚举，禁止写死表头文案**
- 近邻对照必须展示关键字段高亮 + `configVersion` + 风格卡摘要
- 跑批必须可下钻金标，并支持两次 run 并排/切换对比

若 UX 说明与本文路由/契约冲突：路由与 API 以本文为准，交互表现以 UX 说明为准。

---

## 7. 前端实现要点

- shadcn 中后台布局；未登录跳 `/login`
- TanStack Query；推断页展示：评价、适岗 Top-K、风格卡摘要、`configVersion`、近邻并排（关键字段高亮）
- 导入页：列映射 UI（表头→features key / hrEvalText / fitText）；展示 successCount + failedRows
- 配置页：三种角色可勾选；改完保存升版本
- 跑批页：指标卡片 → 行表 → 下钻；支持选择两次 evalRun 并排对比（至少支持先后两次结果切换对比）

---

## 8. UX / 交互验收（必须满足）

1. 导入：映射清晰；失败行可见原因；成功数明确  
2. 近邻对照：关键字段高亮 + 风格卡摘要 + 配置版本可见；能判断像不像  
3. 跑批：命中/同方向率 → 下钻金标对照 → 改配置重跑可对比  
4. 总验收：适岗命中、评价同方向、依据可溯源、配置可回归  
5. 公网：未登录 401/跳转；页面无 LLM Key；api 不直接公网暴露

---

## 9. 公网部署（Compose）

服务：
- `web`：内部 3000
- `api`：内部 3001（**不** publish 到宿主机公网）
- `db`：5432 仅 compose 网络
- `caddy`：80/443 对外；反代 `/`→web，`/api/*`→api

环境变量（示例 `.env.example`）：
```
DATABASE_URL=postgresql://...
LLM_API_KEY=
LLM_BASE_URL=
LLM_MODEL=
SESSION_SECRET=
APP_ORIGIN=https://your.domain
MOCK_LLM=0
```

安全：HTTPS 强制；日志不打印简历全文；密码哈希（argon2/bcrypt）。

---

## 10. 分步实施顺序（按这个做）

1. 脚手架：pnpm monorepo + compose db + `/health` + 空 web  
2. Prisma 全表 migrate + seed 管理员用户 + 默认 StyleCard/SimilarityConfig  
3. Auth 中间件  
4. Case CRUD + import  
5. Similarity 近邻服务 + 单元测试（几条假数据）  
6. Inference（mock LLM 先通，再接真 API）  
7. Eval 跑批 + metrics  
8. 前端六页接契约  
9. Caddy HTTPS 配置与 README 部署步骤  
10. 用假数据跑通主路径；有真实表头再改列映射默认值

---

## 11. 样例数据缺失时的占位 features schema

在 `packages/shared` 先定义可扩展 keys（导入映射可改名）：

```ts
// 占位，真实表头到达后只改映射，不改架构
age: number
education: string   // 枚举
yearsOfExp: number
currentJob: string
targetJob?: string
region?: string
```

`hrEvalText` / `fitText` 固定两列。

---

## 12. README 必须写清

- 本地：`docker compose up -d db` → `pnpm install` → migrate → `pnpm dev:api` / `pnpm dev:web`
- 公网：配域名与 `.env` → `docker compose up -d`
- 如何 seed 用户、如何开关 `MOCK_LLM`
- 主路径截图级文字说明（导入→配置→推断→跑批）

---

## 13. 明确禁止

- 不要做成「上传 PDF/知识库问答」RAG
- 不要微调/训练管线
- 不要 Redis/K8s/微服务拆太碎
- 不要把 LLM Key 放进 `NEXT_PUBLIC_*`
- 不要在没登录时开放导入/推断/跑批

---

## 14. 完成定义（Definition of Done）

- [ ] compose 一键起，公网 HTTPS 可登录  
- [ ] 能导入 ≥20 条案例（可用假 CSV）  
- [ ] 单人推断返回评价 + 适岗 Top-K + 近邻依据 + configVersion  
- [ ] 跑批产出命中率/同方向率，可下钻  
- [ ] 改权重/风格卡后重跑，版本号变化且可对比  
- [ ] 无 Key 时 MOCK 可演示；有 Key 时真调用可切换  

每完成一步输出：改动文件列表、运行命令、当前阻塞。不要推远程除非用户明确要求。
