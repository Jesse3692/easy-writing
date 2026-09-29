# 竞品拆书技术分析：InkOS 实现

> 调研日期：2026-09-29
> 对象：`workspace/novel/inkos`（InkOS 源码，提示词已另行提取至 `inkos/InkOS提示词提取.md`）
> 目的：弄清竞品"拆书"的完整技术实现，为 easy-writing 其他模块借鉴。

InkOS 没有叫"拆书"的功能，而是把它拆成了**三条独立链路**，共享一套本地检索内核。
方法层全部放在运行时可注入的 Skill 文本里（`packages/core/skills/inkos-long-story-analysis/SKILL.md` 仅 18 行），
代码只负责产物形状与协议——方法论本身用户可改。

| 链路 | 入口 | 产物 | 正文是否受影响 |
|---|---|---|---|
| A. 参考素材 | `ingest_material` 工具 | `.inkos/materials/*.md` 可检索素材卡 | 否（只是参考） |
| B. 整本导入 | `import_chapters` 工具 | 真实章节 + 逆向重建的设定/状态 | 是（变成可续写的书） |
| C. 文风仿写 | `generateStyleGuide` | `story/style_guide.md` | 否（写作时注入提示词） |

---

## 一、链路 A：素材级拆解（不进正典）

### 1.1 落地：ingest_material

文件：`packages/core/src/materials/ingest.ts`

- 支持 URL / 上传文件两种来源；PDF 用 `unpdf` 提取文本（扫描件直接报错提示需 OCR），HTML 剥标签取正文。
- 大小上限 18MB；URL 抓取 20 秒超时、限制 http/https 协议。
- 产出一对文件：
  - `{时间戳}-{slug}.md`：正文 + 自带 Metadata 头（kind / purpose / source / mime_type / char_count）；
  - 同名 `.json` manifest：zod schema 严格校验。
- `commitAtomicFileSet` 原子写入（tmp + rename），id 带时间戳防碰撞。
- 工具描述里写死："素材永远只是素材，不得改动正典、章节、剧本状态"。

### 1.2 检索：FTS5 本地检索内核（最值钱的基础设施）

文件：`packages/core/src/retrieval/local-search.ts`（281 行）+ `materials/retrieve.ts`

- `node:sqlite`（Node 原生模块，零外部依赖）+ **SQLite FTS5**：
  - 外部内容表（`content=` 指回主表），触发器同步增删改；
  - `bm25()` 排序，标题字段权重 5.0；
- 中文分词：`Intl.Segmenter`（ICU）按词切 + **相邻汉字二元组补丁**（ICU 会把短中文复合词切碎，bigram 保住召回），连字符复合词额外整词入索引；
- markdown 按**空行分块、标题向下继承**，每块记录 `charStart / charEnd`；
- 每文档存 **sha256 内容哈希**，`replaceScope` 增量 upsert，内容没变的段落跳过写库；
- 数据库被明确定位为"**可重建的投影**"——源 md 文件永远权威，DB 删了可整库重刷；
- 一个内核服务三处：故事记忆、归档素材、Skill 引用（scope 隔离）。

返回结果带 `path:charStart-charEnd` 证据指针，工具回执里提示模型"要用整段就自己读该文件"。

### 1.3 绑定：manage_book_reference

文件：`packages/core/src/references/book-references.ts`

- 把已落地素材挂到某本书，manifest 是一个 `reference_bindings.json`（materialId + `uses[]` + note），原子写。
- 两个讲究的细节：
  - `uses`（用途标签）**保留用户原话**（如"开篇机制、调查节奏"），不映射到固定分类体系；
  - 绑定不复制正文；解绑不删素材（素材存一份，多本书可共享）。
- 读取时校验 manifest 路径必须与 materialId 一致（防篡改指向）。

### 1.4 写作时消费：三级漏斗

文件：`packages/core/src/references/reference-context.ts` + `agents/composer.ts:446`

写第 N 章时的选取流程：

1. **切段**：所有绑定素材按标题切成 section（anchor slug 化并去重）；
2. **关键词召回**：候选列表（只含元信息，不含正文）交给模型；
3. **模型精选**：一次结构化工具调用（返回 `selectedIndices` 整数数组）挑出对本章有用的段落；
4. 只有选中段落才进上下文，标记 `protection: "compressible"`（上下文超预算时可被压缩）。

reason 字段固定写入：*"Reference guidance only; it cannot override author intent, canon, or current state."*
（参考只是指导，不能覆盖作者意图、正典与当前状态。）

---

## 二、链路 B：整本导入拆解（逆向重建，最重的一条）

入口：`pipeline/runner.ts:1662` `importChapters`，两步走。
支持：章节自动切分（默认正则匹配"第X章 / 第X回 / Chapter N"，可传自定义 JS regex）、
`resumeFrom` 断点续传、continuation（原线续写）/ series（同宇宙新故事）两种模式、`acquireBookLock` 全程持锁。

### 2.1 Step 1：从全书反推基础设定（一次付清，重试不重付）

**① 语义编译超长输入**（`agents/import-context.ts` `compileImportSource`）：

- 先估算全书 token（`semanticInputBudget`），不超预算直接原文直送；
- 超了就按预算分块，逐块让 worker agent（挂 story-import + continuation-writing 技能）"语义编译"成可追溯 Markdown；
- 最后拼上完整章节目录组成资料包；编译痕迹落 `story/runtime/import-context-trace.json`（模式、章数、块数）。

**② 四阶段结构化提取**（`agents/architect.ts:188` `generateFoundationFromImport`）：

每阶段一次 tool call（`submitStructured`），后一阶段吃前一阶段产物，温度 0.5
（比正常生成 foundation 的 0.8 低——拆解要克制）：

| 阶段 | 产物 | 关键约束 |
|---|---|---|
| ① 框架 | `story_frame` + `volume_map` | 输出上限 8192 token |
| ② 规则 | `book_rules`（可读 md + BookRulesSchema 结构化 JSON 双份）+ 初始伏笔 hooks | |
| ③ 人物索引 | cast index | **只许提交姓名和主/次级别**，"不把无名岗位群体扩写成虚构传记" |
| ④ 人物卡 | 逐个角色卡 | 中文 150–250 字，限定写"当下动机、已知信息、关系压力、能力边界" |

护栏提示词（值得原文保留）：

> 既成事实从资料包推导；未来剧情和结局服从明确的续写指令。不能仅凭文风或悬念推导必须发生的未来事件，未指定的发展保留开放。压缩资料包是证据，不是臆造缺失正典的许可。

**③ 落盘**（`architect.ts:142` `writeFoundationFiles`）：
原子写 `story/outline/story_frame.md`、`story/outline/volume_map.md`、`story/book_rules.md` + `book_rules.json`、`story/roles/{主要|次要}角色/*.md`；
然后重置回放种子（current_state / pending_hooks 置初始种子，删除旧 summaries / memory.db / snapshots）。
失败重试时 foundation 草稿已落盘，跳过 Architect 直接进 Step 2（不重付这一次大调用）。

### 2.2 Step 2：顺序回放重建状态

逐章调 `writer.settleChapterState`（`agents/writer.ts:214`）：

- **原文一字不动**，只让 Settler（`agents/settler-prompts.ts`）把"正文**明确事实**投影为**增量**运行时 truth（delta）"，明令"不把计划当成已发生事件"；
- 每章带 `baselineChapter` 状态快照，逐章累积 `current_state.md` / `pending_hooks.md` / `chapter_summaries.md` / `memory.db`；
- 失败可 `resumeFrom` 从任意章续跑。

核心思想：**导入后的"设定与状态"不是一次性猜出来的，是逐章重放出来的**——跑完后这本书天然可续写，Writer/Auditor 拿到的 truth 与真实正文严格对账。

---

## 三、链路 C：文风拆解（仿写）

文件：`agents/style-guide.ts`（46 行）——三条链路里最短的一条。

- 一次 worker agent 调用：注入 long-story-analysis + imitation-writing 两个技能全文；
- 系统提示只有一句："把参考文本编译为**有证据、可执行的**文风指南，只返回 Markdown"；温度 0.3；
- 产物落 `story/style_guide.md`，之后 Writer 每章提示词里作为独立权威块注入（`agents/writer-prompts.ts` 的"文风指南" block）。

拆解镜头定义在 `skills/inkos-long-story-analysis/references/analysis-lens.md`，**八层**：

1. Promise — 读者对"继续读下去能获得什么"的预期；
2. Story engine — 可复用的压力-选择-后果-奖励循环；
3. Structure — 开局契约、弧线转场、卷目标、中点变化、回报埋点；
4. Character — 欲望、恐惧、策略、关系移动、不可逆选择；
5. Information — 每阶段读者/主角/对手各自知道什么；
6. Scene craft — 入场压力、具体动作、证据、转折、后果、离场张力；
7. Prose behavior — 视角距离、句势、对话潜台词、细节挑选、段落节奏；
8. Long-horizon state — 事实、承诺、钩子、未偿代价、连续性负担。

输出纪律（同样在 lens 文件里）：

> 重要论断必须锚定章节或引文；分离直接观察与推断；结尾给可迁移原则、失败条件与重组选项；禁止输出"变相改写"的原文。

仿写护栏（`skills/inkos-imitation-writing/SKILL.md`）：不搬人名、措辞、场景顺序、标志性组合；
参考供给 craft evidence 而非故事内容；用户的预设、人物、事实、方向始终权威。

---

## 四、可借鉴点清单（映射到 easy-writing）

| # | 技术 | InkOS 做法 | easy-writing 落点 |
|---|---|---|---|
| 1 | **FTS5 检索内核**（最推荐） | 281 行、零外部依赖、一个内核服务三处（素材/记忆/Skill 引用） | Rust 侧用 rusqlite 复刻（外部内容表 + ICU 分词 + Han bigram + content-hash 增量），给素材库、设定库、历史拆书结果做统一检索——现在拆完的书是"死数据"，接上检索即可变成写作时可查的证据库 |
| 2 | **分阶段提取** | 框架→规则→人物索引→人物卡，逐级收窄、每级输出限额 | 单章拆解目前是 400 段一次性吐全部 JSON；做全书级深拆（卷纲、节奏曲线、人物弧）时照搬分阶段，稳定性远高于一次大 JSON |
| 3 | **语义编译超长输入** | token 预算→分块逐块编译→超预算重编译一轮→仍超则硬失败 | 全书报告现在只喂各章 summary briefs（信息损失大），这是长书报告的直接升级路径 |
| 4 | **证据指针贯穿** | `path:charStart-charEnd` + "观察 vs 推断分离" | `paragraphInsightIds` 段落映射已同构；差的是把指针传给下游（写作时引用"第 X 章第 Y 段是这么处理钩子的"），而不只是当高亮 |
| 5 | **权威模型** | "参考/拆解产物永远不是正典"写进每个注入点的 reason | 以后做"参考拆书结果续写"时，这条护栏必须一起搬 |
| 6 | **断点续传 + 幂等** | foundation 草稿落盘保留、`resumeFrom` 章级续传、书本锁、原子写 | 拆书已有 per-chapter 状态机；长任务（整本导入、批量拆书）可补"草稿保留、重试不重付"这一层 |
| 7 | **Skill 与代码分层** | 代码只管产物形状与协议，方法论全在 SKILL.md（运行时注入、用户可改） | 与 `promptText('breakdown',…)` 用户可改提示词库思路一致；可进一步把"分析镜头"也做成可加载资源 |
| 8 | **结构化输出协议** | 要求原生工具参数（数组不许塞字符串）、结果工具只调一次、schema 严格校验 | `parseAiJson` 容错解析可保留，但拆解类 JSON 可考虑改走原生结构化输出通道 |

---

## 五、关键文件索引

```
inkos/packages/core/
├─ skills/
│  ├─ inkos-long-story-analysis/SKILL.md        # 长篇拆稿技能（18 行）+ references/analysis-lens.md 八层镜头
│  ├─ inkos-short-story-analysis/SKILL.md       # 短篇拆稿（情绪链/证据链/反转机制）
│  ├─ inkos-imitation-writing/SKILL.md          # 仿写护栏
│  └─ inkos-story-import/SKILL.md               # 导入/重建/母本区分意图路由
├─ src/
│  ├─ materials/ingest.ts                       # ingest_material 落地管线
│  ├─ materials/retrieve.ts                     # 素材检索（三级漏斗第一级）
│  ├─ retrieval/local-search.ts                 # FTS5 检索内核（281 行）
│  ├─ references/book-references.ts             # 绑定 manifest
│  ├─ references/reference-context.ts           # 写作时参考段落选取
│  ├─ agents/composer.ts:446                    # selectReferenceSections 模型精选
│  ├─ agents/import-context.ts                  # 超长输入语义编译
│  ├─ agents/architect.ts:188                   # 四阶段逆向 foundation
│  ├─ agents/writer.ts:214                      # settleChapterState 顺序回放
│  ├─ agents/settler-prompts.ts                 # 状态投影提示词
│  ├─ agents/style-guide.ts                     # 文风指南编译（46 行）
│  ├─ agent/agent-tools.ts:823-1130             # ingest/retrieve/book_reference/import_chapters 工具协议
│  └─ pipeline/runner.ts:1662                   # importChapters 主流程
```
