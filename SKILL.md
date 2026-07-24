---
name: autopilot
description: Autonomous long-haul execution for when you have ALREADY set goals and a rough plan and want maximum progress without babysitting — overnight, during a meeting, while AFK. Runs TDD. Parks anything that needs the user's decision into a DECISIONS log and KEEPS WORKING on independent tasks instead of stopping; only halts for irreversible / security-risk operations or when genuinely fully blocked. Parallelizes highly-independent tasks across subagents. Manages its own context budget on long shifts — offloads heavy reads/analysis to subagents (returns conclusions, not file dumps) and rebuilds state from disk after compaction. Honors per-project authorization declared at invocation (DB access level, manageable ports, etc.). When functional goals are essentially complete, auto-switches to a non-functional polish pass (simplify / reuse / efficiency / dead code) dispatched to ISOLATED subagents (reusing /code-review and /simplify) so self-authored code is judged without self-justification bias. Anti-over-engineering discipline (YAGNI, no premature abstraction/optimization) enforced throughout. Ends with a structured handoff report. Invoke with /autopilot. Trigger words — autonomous, unattended, overnight, while I sleep, keep working, hands-off, AFK, babysit, 我去睡觉了, 继续推进.
---

# Autopilot — 无人值守自主推进

当你已经标好**目标 + 大致计划**、希望 agent 在你离开时（睡觉 / 开会 / AFK）尽量多推进时，用 `/autopilot` 触发。核心一句话：

> **默认搁置需要用户拍板的决策、继续干别的；只有不可逆 / 安全风险操作、或确实全部被卡住时才真正停下。**

## 黄金规则（最重要）

- **全程不要向用户提问**（用户不在）。任何"需要拍板"的点 → 写进项目的 `DECISIONS.md`，按推荐默认先推进或先跳过，然后**立刻继续下一个独立任务**。
- **只有两种情况结束回合**：
  1. 所有功能目标基本完成 + Polish 做完 + 报告已生成；
  2. 遇到 HARD STOP 操作，或剩下所有任务都被卡住、确实无活可干。
- **推进清单里的每一项**，不只挑容易做的；独立度高的任务可并行推进（见 Execute）。
- 用户回来看到的，要么是「完成报告 + 待拍板清单」，要么是「被硬决策卡住不得不停」——两种情况推进量都最大化。

## 决策分类清单（PARK vs HARD STOP）

### PARK — 写进 DECISIONS.md，按默认推进，继续干别的
- 命名、标签、文案、错误提示、按钮文字
- API 字段结构 / 返回格式（选 RESTful 常规默认，备注备选）
- "要不要顺带加 X？" 之类的范围扩展 → 默认**不加**，记录
- 有合理默认的歧义需求
- 等价技术方案里明显更低风险的那个

每条 DECISIONS 含：`编号 / 优先级 / 上下文 / 选项 / 我的推荐 / 影响 / 是否可逆 / 当前已按什么推进`（模板见 `templates/DECISIONS.md`）。

### HARD STOP — 结束回合，把决策摆到用户面前
- 不可逆 / 破坏性：删库（DROP）、删生产资源、force-push、批量删数据、无法回滚的 schema 变更 / migrate（普通 INSERT / UPDATE 写入**不**在此列，见 DB 授权）
- 安全风险且有外部影响：改鉴权、暴露密钥、对外开端口、放宽权限
- **任何数据删除**（DELETE / DROP / TRUNCATE 等），以及**超出本次声明授权范围**的敏感操作（见下方「项目专属授权」）
- **push 到用户未授权的分支**：push 默认禁止；仅当用户明确点名某个 / 某些分支时才允许，且**只能 push 被点名的那些分支**（用户授权 `feature-a`，却去 push `main` 或其他分支 → 禁止）。force-push 始终 HARD STOP，除非用户**单独**明确授权。
- **覆盖不可再生 / 昂贵再生的产物**：实验数据、训练好的模型 / checkpoint、历史快照、导出 dump、生成的大文件（索引 / embedding / 缓存）等——默认**新建目录或版本副本、保留旧版**；必须覆盖且确认旧内容无用才可，拿不准 → HARD STOP（见「通用边界·非破坏性写入」）。
- 两条路都很贵、且无法自信二选一的
- 剩下所有任务都被 parked 决策卡住、没有别的可推进时

## 反过度设计原则（cross-cutting，贯穿 Execute 与 Polish）

自主推进时最大的诱惑是"反正没人盯着、多建点能力以防万一"。明确抵制——**写能解决当前问题的最简代码**：

- **YAGNI**：只做当前目标 / 计划真正需要的。不建"以后可能要用"的通用化、扩展点、配置开关、插件化。没有第二个**真实**用例，就别抽象。
- **三次法则**：抽象要等同一模式真正重复约 3 次再提取。**重复 < 错误的抽象**——抽错的通用层比重复更难改。Polish 阶段做"复用"时尤其要克制。
- **不过早优化**：没有**测出来**的性能问题（profiling / 慢测试 / 真实瓶颈）就不动性能。Polish 阶段的"提效"只针对实测热点，不为了"更快"而牺牲可读性。
- **重构只保行为**：refactor 不夹带新功能、不悄悄改架构。要加功能就单开任务走流程。
- **最小可行改动**：能小改解决的别推倒重来、别顺手"升级"周边代码。
- **跟随既有代码风格**：周围是简单过程式，就别引入 DI / 多层抽象 / 设计模式堆叠。匹配现有 altitude。

## 上下文卫生（cross-cutting，长任务的生命线）

本 skill 跑的是长 shift，主智能体的上下文会越攒越满、越往后质量越差。**主动给主上下文减负**和落盘同等重要——不是可选优化，是长任务能不能跑到最后的决定因素。

- **派子智能体当"减压阀"，只回收结论**：凡是要读一大堆文件 / 翻长日志 / 跑大块分析的任务，**派给只读子智能体（如 Explore）去做，主智能体只拿回一段结论**，不把原始内容灌进自己上下文。判定标准——"这步自己干会不会让 context 暴涨？"会 → 派出去。这是子智能体除"并行 / 隔离 review"之外的**第三用途**，也是防爆 context 最直接的一招。
- **读策略：定位而非整读**：先用 Grep / Glob 定位，再针对性 Read 某几段；别整文件 `Read`、别把构建 / 测试 / 日志的完整输出贴进上下文——要的是结论（绿 / 红、报错行、关键差异），不是原文。
- **把状态外化到文件，别靠记**：进度走 TaskList，决策走 `DECISIONS.md`，过程走 `SHIFT_REPORT.md` 草稿。**别把"做到哪了 / 还剩什么"装在脑子里**——脑内状态会在压缩后被冲掉，文件不会。
- **压缩恢复协议**：session 被压缩或重启后，**先重建状态再动手**：读 TaskList → `DECISIONS.md` → `SHIFT_REPORT.md` 草稿，理清"已完成 / 进行中 / 待办 / 已 park 的决策"，再继续。落盘就是为这一刻，别白落。
- **察觉变重就主动减负**：当开始需要翻很久之前的内容、或反复读同一文件时，说明 context 已经偏重——主动把当前进度落盘、把后续笨重步骤派给子智能体，别硬撑到爆。

## 五个阶段

### 1. Orient 定向
- 找用户标好的目标 + 计划：优先看项目里的 `PLAN.md` / `TODO.md` / `.autopilot/`、对话上文、现有 TaskList。
- 找到 → 进入执行。
- 找不到但能合理推断 → 按推断目标推进，并把"我推断的目标"作为一条 DECISIONS 记下。
- 完全找不到且无法推断 → 合法的 HARD STOP（没方向没法干）。
- 用 TaskCreate 建一份任务清单当实时进度账本。

### 2. Execute 执行
- 严格 TDD：先写 / 改测试 → 跑红 → 实现 → 跑绿 → 重构。
- **目标：尽最大可能把工作清单里每一项都往前推**，不只挑容易的；卡住的 park 后跳下一个，回头再说。
- **只做当前需要的**——遵循上面的反过度设计原则（YAGNI、最小改动、不过早优化），绝不自作主张加扩展点 / 通用化 / "以后可能要用"的开关。拿不准要不要做某事，默认不做，进 DECISIONS。
- **独立任务派子智能体推进**：清单里若有**高独立性**的任务（改动文件 / 运行状态基本不重叠），派多个子智能体（Agent 工具）同时推进；主智能体负责分发、聚合、集成，最后跑集成测试。派子智能体一举三得：**并行提速 + 任务间上下文隔离（互不污染）+ 给主上下文减压**（见「上下文卫生」）。
  - **独立性判断**：任务共享文件 / 共享状态 / 有先后依赖 → **串行**，别并行。只有文件与状态都不重叠才算独立。
  - **隔离**：git 仓库里并行**改代码**用 `isolation: 'worktree'`（各自独立工作树，互不冲突）；非 git 仓库或文件有重叠 → 串行。只读的探查 / review 可随意并行。
  - **规则一致**：每个子智能体同样遵守 TDD、反过度设计、PARK/HARD STOP；子智能体遇到的决策或 HARD STOP，由主智能体统一汇入 `DECISIONS.md`。
  - **天然隔离**：子智能体上下文互不可见——**每个任务的细节不污染其他任务、也不污染主智能体的上下文**，顺带也减弱"自己 review 自己"的偏见。
- 每完成一个任务：更新 TaskList、增量写 `SHIFT_REPORT.md` 草稿、有决策就追加 `DECISIONS.md`。**状态必须落盘**——session 可能被压缩或中途挂掉。
- 撞到决策 → 按 PARK / HARD STOP 分类处理。
- **卡住止损**：单个任务连续约 3 次尝试无进展 → park「blocked on X，试过 A/B/C」，跳到下一个任务，**不空转**。

### 3. Verify 验证
- 跑全量测试（单元 + 集成），确保全绿。**测试都在本地 PC 跑**，不碰远程 / 生产环境。
- 对**已构建的功能**，用 `playwright-skill`：
  - 自动探测本地 dev server；用 `accounts.local.md`（本项目 `.autopilot/accounts.local.md` 优先，否则本 skill 目录里的）的测试账号 / 超管账号登录。仓库里只提供 `accounts.local.example.md` 模板——复制为 `accounts.local.md` 填入真实账号（该文件已 gitignore）。
  - **不只跑主流程 E2E，也可用于逻辑排查 / 复现 bug**：驱动前端到特定状态、观察实际行为来定位问题、截图取证。
  - 密码等敏感信息**绝不写进报告或日志**。

### 4. Polish 打磨（功能目标基本达成后主动切换）
- 进入条件：所有非阻塞任务已完成，剩下的要么是 parked 决策、要么是 polish 候选。
- 范围：**本次 shift 动过的代码**（`git diff`，或自己维护的 touched-files 清单）。
- review 清单：正确性 bug / 复用（消除重复）/ 精简 / 死代码 / 运行效率 / **过度设计（过早抽象、YAGNI 违规）** / 命名。
  - 注意这是**双向**的：既要揪出"重复/啰嗦"，也要揪出"为了复用而过度抽象、为了灵活而加的用不上的扩展点"。两者都是债。

**🔒 强制隔离原则（重要）——绝不在写代码的同一上下文里 review 或重构自己写的代码。**
同一上下文里，智能体天然有「**自洽倾向**」：会下意识为自己的实现找理由、回避否定自己之前的判断，review 会失真。因此 Polish 阶段的所有 review、重构、精简、复用、性能优化，**必须派发给子智能体（Agent 工具）开全新上下文执行**——这也避免已经很长的主上下文再被整段 diff 的阅读撑爆（见「上下文卫生」）：

1. **Review（子智能体 · 独立上下文）**：派一个子智能体，只给它「diff / touched 文件路径 + 客观验收标准 + 上面的 review 清单」。**不要**把你的实现理由 / 设计思路塞进它的 prompt——那会让它锚定你的解释。让它直接读代码、独立评判。可以让它在内部调用 `/code-review` skill。返回 findings。（同时提醒它：消除重复与警惕过早抽象要平衡——别为了"复用"反而推荐过度抽象。）
2. **Apply（子智能体 · 独立上下文）**：基于 findings 派**另一个**子智能体执行修复 / 重构（也可让它用 `/simplify`）。能安全改的就改并跑测试；拿不准、或改动大有风险的 → 记进 DECISIONS，不强行改。
3. **防偏见自检**：子智能体返回的 findings，主智能体**不得**因为"我当初有理由"就一律否决——要么照改、要么把"不改的理由"也写进 DECISIONS 让用户裁。否则隔离就白做了。
4. **并行规则**：多个子智能体改同一批文件时**串行**避免冲突；只读的 review 可并行。需要并行改代码可用 `isolation: 'worktree'`（仅 git 仓库）。

### 5. Report 汇报
- 在项目里生成 `SHIFT_REPORT.md`（模板见 `templates/SHIFT_REPORT.md`），含：
  1. 本次完成的任务列表
  2. 每项任务的修改内容与实现方式
  3. 测试执行情况与结果
  4. 遇到的问题及解决方案
  5. 当前项目状态
  6. 下一步建议计划
  7. 未完成任务的原因与预计工作量
  8. **待你拍板的决策清单**（DECISIONS.md 摘要，按优先级，每条带推荐）
  9. **非功能性 review 发现 + 已做的精简 / 优化**（标注哪些来自隔离子智能体）
  10. **待清理数据清单 + 是否删除**：列出本次产生的临时 / 测试数据（DB 行、临时记录等），**逐项问用户是否删除**，未确认不删。
- 报告写完后，把摘要 + 待清理清单输出到对话里，**就待清理数据明确询问用户**，再结束回合。

## 权限与边界

**项目专属授权（调用时由用户声明，严格在声明范围内操作）**
调用 `/autopilot` 时用户可能声明本项目特有的权限。**只在声明范围内行动；声明之外、且属敏感 / 不可逆的操作 → HARD STOP。** 常见声明项：
- **数据库（默认策略，可由调用时声明覆盖）**：
  - ✅ 允许：读（SELECT）、写（INSERT / UPDATE）
  - ❌ 禁止 → HARD STOP：任何删除（DELETE / DROP / TRUNCATE / 批量删）、schema 变更 / migrate
  - 🕓 临时数据：测试等过程产生的临时数据**中途不要删**，记进「待清理」清单（落盘到 SHIFT_REPORT 草稿）；**完整流程跑完后**在汇报里提示用户，由用户决定是否删除——未获明确确认前不删。
  - 调用时可声明更严（如「只读」）。**默认不放开删除权限**，除非用户明确授权。
- **可管理的端口**：如声明「8006 端口占用程序的启用 / 重启」→ 你可对该端口对应进程 **启动 / 重启 / 关闭**。只动声明过的端口，并记录做了什么、为什么。
- **可 push 的分支**：默认禁止 push。若用户明确声明「可 push 到 `feature-x`」之类 → 允许 push，但**仅限用户点名的分支**，超出范围一律 HARD STOP。force-push 不含在内，需**单独**明确授权。
- 其他声明（可调用的内网服务、可用账号范围等）同样照此办理。

**通用边界（始终适用）**
- **只在本地 PC 测试 / 起服务**，不碰远程 / 生产环境。
- **不要**：改远端 CI 配置、删生产数据、对外暴露密钥——这些走 HARD STOP。
- **git**：可以本地 commit（遵循仓库的提交规范）。
- **push 默认禁止，按分支白名单放行**：默认不 push。仅当用户**明确点名**某个 / 某些分支时才允许 push，且**只能 push 用户点名的那些分支**——授权范围外的分支一律 HARD STOP（用户授权了 `feature-a` 却去 push `main` 或其他分支 = 禁止）。force-push 始终 HARD STOP，除非用户**单独**明确授权。
- **回滚友好**：尽量让每一步都可逆（小步 commit、新代码先加再删旧的），减少 HARD STOP 的触发。
- **非破坏性写入（不可再生产物只新增、不覆盖）**：很多产物**再生成本高或根本无法复现**——实验数据、训练好的模型 / checkpoint、历史数据快照、导出 / dump、生成的大文件（索引 / embedding / 缓存）、报告快照等。覆盖它们 = 在一次普通 `save` / `write` 里**静默销毁旧内容**，等同于删除，但容易绕过 DELETE 类红线。
  - **默认新建、不覆盖**：写新版本就开**新目录 / 新文件 / 带版本号或时间戳的副本**，旧的原地保留。例：跑实验 → `runs/exp-2026-07-24/`、`models/v2/`，不覆盖 `runs/last/` 或既有权重文件。
  - **覆盖前先问"能否廉价、确定性再生"**：能（如重新生成的测试产物、从代码重建的文件）→ 可覆盖；**不能确定 → 当作不可再生，新建**。
  - **必须覆盖时**：只有覆盖是唯一办法、且旧内容确实不再需要才覆盖；拿不准或旧内容可能还有用 → 记进 DECISIONS，按"新建保留旧版"推进，**不强行覆盖**。

## 相关 skill 与工具
- `playwright-skill` — 浏览器 E2E + 逻辑排查 / 复现 bug
- **Agent 工具（subagent）** — 三用途：① Execute 阶段并行推进独立任务（兼任务间上下文隔离）；② Polish 阶段开独立上下文做隔离 review，洗掉自洽倾向；③ 「上下文卫生」减压阀——把笨重的读 / 分析派出去，只回收结论，防爆主上下文
- **只读子智能体（如 Explore）** — 返回结论而非文件堆；上下文卫生里派去读大堆文件 / 翻长日志时的首选
- `/code-review`、`/simplify` — 在子智能体内部调用，做正确性 / 复用 / 精简 / 效率 review
- `/verify` — 端到端验证改动确实生效
