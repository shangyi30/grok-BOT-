# HR 人岗评价系统 · UX 说明（给 Cursor / Codex）

> 与《Cursor完整施工方案》配套。栈、路由、API 契约**不动**；本文只补界面结构、状态机与交互约束。
> 日期：2026-09-24
> 优先三屏：导入映射、推断近邻对照、跑批下钻并排。其余登录/列表/配置用简洁中后台骨架即可。

---

## 0. 全局

### 布局
- 未登录：仅 `/login`，无侧栏。
- 已登录：左侧导航 + 顶栏。
  - 侧栏：案例库 · 配置 · 单人推断 · 跑批
  - 顶栏右侧：当前用户邮箱 · 退出
- 组件库：shadcn/ui（Button / Table / Tabs / Card / Badge / Alert / Dialog / Select / Textarea）
- 密度：中后台紧凑；表格行高舒适可扫读。

### 全局状态
| 状态 | 表现 |
|---|---|
| 未登录访问业务页 | 跳转 `/login` |
| API 401 | 清会话 → `/login` |
| API 5xx / 网络错 | Toast 错误 + 页面内可重试 |
| 同步推断进行中 | 主按钮 loading、禁止重复提交 |
| 空列表 | 文案 + 主 CTA（可不做插画） |

### 导航对应路由（已锁）
`/cases` · `/cases/import` · `/config` · `/infer` · `/infer/[id]` · `/batches` · `/batches/[id]` · `/batches/[id]/rows/[rowId]` · `/login`

---

## 1. 优先屏 A：`/cases/import` 导入映射

### 目标
表头 → 列角色映射；失败行可见原因；成功数明确。

### 线框（自上而下）
```
[面包屑: 案例库 / 导入]
标题: 导入案例
步骤条: ① 上传  →  ② 映射  →  ③ 结果

── 步骤① 上传 ──
[拖拽区 / 选择 CSV·Excel]  支持 .csv .xlsx
预览前 5 行原始表头

── 步骤② 映射 ──
表: | 原表头 | 映射到（下拉） | 预览样例值 |
下拉选项（列角色枚举，禁止写死表头文案）:
  - features.<key>（占位: age / education / yearsOfExp / currentJob / targetJob / region）
  - hrEvalText
  - fitText
  - fitJobs（可选，多值逗号分隔）
  - （跳过）
必填未映射: hrEvalText、fitText、至少一个 must_match 相关 features
  → 底部「开始导入」禁用 + 红字提示

── 步骤③ 结果 ──
成功: Badge「成功 N 条」
失败: Table | 行号 | 原因 |  [仅失败可导出 CSV]
主按钮: 返回案例列表
次按钮: 再导一批
```

### 状态机
1. `idle` → 选文件 → `parsing`
2. `parsing` 成功 → `mapping`（带 headers + sampleRows）
3. `mapping` 点导入 → `importing`（按钮 loading）
4. 响应 `{ successCount, failedRows }` → `done`
5. 解析失败 → `error`（可重选文件）

### 交互约束
- **映射页只绑列角色枚举**，表头文案来自文件，UI 不写死中文列名。
- 同一目标字段不可被两列同时映射（冲突时后选覆盖前提示）。
- `failedRows` 为空时不展示失败表。
- 导入中离开页：ConfirmDialog 确认。

---

## 2. 优先屏 B：`/infer` + `/infer/[id]` 近邻对照

### `/infer` 表单
```
标题: 单人推断
表单字段 = 当前 SimilarityConfig 中的 features keys（占位列）
[提交推断] → POST /inference → 跳转 /infer/[id]
提交中: 按钮 loading，防双击
```

### `/infer/[id]` 线框（核心）
```
顶栏信息条:
  configVersion 徽章（如 style:3+sim:2）
  状态 Badge（done / failed）
  失败时: error 文案 + [重试]

左栏（约 40%）候选摘要:
  Card: 输入 features 键值表
  Card: evalSummary
  Card: fitJobsTopK 列表（job + score 进度条）
  Card: styleCardSummary

右栏（约 60%）近邻对照:
  标题: 近邻案例 Top-N
  每一行可展开的 Case:
    头: #rank · score · [展开]
    展开后「并排对照」两列表格:
      | 字段 | 候选人 | 案例 |
      差异字段行: 背景高亮（黄/橙浅底）
      同值字段: 默认样式
    下方只读: 案例 hrEvalText / fitText（金标参考）
```

### 交互约束
- **关键字段高亮**：以 `neighbors[].keyFeatures` 或候选人 vs 案例 features 差集为准；`must_match` / `weighted` 字段优先高亮。
- **`configVersion` 必须常显**，不可藏在折叠里。
- 抽检目标：用户只靠对照表 + 风格卡摘要 + 金标文本判断「像不像」。
- failed：不渲染假 neighbors；明确失败原因。

---

## 3. 优先屏 C：跑批 `/batches` · `/batches/[id]` · `.../rows/[rowId]`

### `/batches` 列表
```
[新建跑批] → POST /eval/runs → 进入详情
Table: 时间 | configVersion | 状态 | fitHitRate | evalAgreeRate | n
```

### `/batches/[id]` 结果总览
```
顶栏指标卡 ×3: 适岗 Top-K 命中率 | 评价同方向率 | 样本数 n
副信息: configVersion · 创建时间 · [再次跑批]

并排对比（可选）:
  Select「对比另一 run」→ 右侧多一列指标卡（两次 evalRunId 并排）
  无第二次时隐藏对比列

结果表:
  | 案例摘要 | fitHit | evalAgree | 操作 |
  行点击或「对照」→ /batches/[id]/rows/[rowId]
  离谱样本: fitHit=false 或 evalAgree=false 行用警示 Badge
```

### `/batches/[id]/rows/[rowId]` 下钻
```
面包屑: 跑批 / {id} / 行 {rowId}

金标区（只读）:
  goldHrEvalText · goldFitText

模型输出区:
  本行关联 Inference 的 evalSummary / fitJobsTopK

近邻对照: 复用「优先屏 B」右栏组件（同一 NeighborCompare）
常显: configVersion

并排两次 run（若带 ?compareRunId=）:
  左右分栏各一套「模型输出 + 近邻摘要」
  顶标注 evalRunId A / B
```

### 交互约束
- 命中率 / 同方向率来自 `metrics`，缺省显示「—」而非 0（避免误解）。
- 改配置后必须 **新 run**（新 evalRunId）；对比是选两个已有 run，不是原地覆盖。
- 下钻必须能对金标；禁止只有汇总数字无法点进样本。

---

## 4. 骨架屏（简洁即可）

### `/login`
居中 Card：邮箱、密码、登录。错误 Inline Alert。无注册入口（MVP）。

### `/cases`
Table 案例列表（features 摘要列 + hrEval 截断）；顶栏 [导入]；搜索框可选。空态 CTA → 导入。

### `/config`
Tabs：`风格卡` | `字段角色` | `权重`
- 顶栏常显将生成的 `configVersion` 预览（保存后刷新）
- 字段角色：表格 key · role 下拉（must_match / weighted / display_only）· weight（仅 weighted 可编）
- 风格卡：dimensions / tone / bannedPhrases
- 保存 → PUT → Toast「已升版本」

---

## 5. 组件建议（给前端文件树）

| 组件 | 用途 |
|---|---|
| `AppShell` | 侧栏+顶栏 |
| `ColumnMapper` | 导入步骤② |
| `ImportResultPanel` | successCount + failedRows |
| `NeighborCompare` | 并排字段高亮 + 金标文本（infer 与 batch row 共用） |
| `ConfigVersionBadge` | 全局展示 configVersion |
| `MetricsCards` | 跑批命中/同方向/n |
| `RunCompareSelect` | 选择第二次 evalRun 并排 |

---

## 6. Cursor 实现检查清单（UX DoD）

- [ ] 导入三步完整；失败行有原因；成功数可见
- [ ] 近邻页：字段高亮 + styleCardSummary + configVersion
- [ ] 跑批：指标卡 + 行下钻 + 可选两次 run 并排
- [ ] 映射只绑枚举；占位 features 可替换表头而不改布局
- [ ] 推断提交防重复；失败态无假数据

---

## 7. 非目标

- 不做账号管理页、不做移动端专项、不做复杂数据可视化大屏。
- Figma 高保真可后补；本文线框 + 状态足够灌页面。Cursor 以本文 + 施工方案第 7/8 节为准。
