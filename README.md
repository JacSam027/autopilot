# autopilot

> Claude Code skill —— 你标好目标和计划后，让 agent 在你离开时（睡觉 / 开会 / AFK）自主推进、最大化进度；只在不可逆 / 安全风险、或确实全卡住时才停下等你。

## 解决什么问题

Claude Code 是回合制的：agent 一旦停下来问你，这一轮就结束了，没法"同时"干别的。所以"关键处停下等我拍板"和"继续干别的不要等我"在字面上是矛盾的。

**autopilot 用「决策停车场」破这个矛盾**：需要你拍板的事，绝大多数先**记进 DECISIONS 日志、按推荐默认推进、然后立刻继续下一个独立任务**；只有不可逆 / 安全风险操作、或确实全部被卡住时，才真正停下。你回来看到的，要么是「完成报告 + 待拍板清单」，要么是「被硬决策卡住不得不停」——两种情况推进量都拉满。

## 核心特性

- **决策停车场（PARK vs HARD STOP）** —— 不为每个小决定打断你，分类搁置、继续推进
- **隔离 review** —— Polish 阶段把 review / 重构派给**子智能体开新上下文**，洗掉"自己 review 自己"的自洽倾向
- **反过度设计** —— YAGNI、三次法则再抽象、不过早优化、最小可行改动
- **独立任务并行** —— 高独立度任务派多个子智能体同时推进
- **项目专属授权** —— 调用时声明 DB 权限 / 端口等，严格在声明范围内操作
- **数据安全** —— DB 默认允许读写、**禁止删除**；临时数据跑完整个流程后，再提示你是否清理
- **push 受控** —— 默认禁止 push；仅当你明确点名分支时才允许，且**严格只动被点名的分支**（force-push 仍需单独授权）
- **TDD 全程 + Playwright** 做 E2E 与逻辑排查
- **增量落盘** —— 进度 / 决策实时写文件，session 中断也不丢
- **结构化汇报** —— 自动生成含待拍板清单的工作报告

## 安装

把本仓库放进 Claude Code 的 skills 目录（任选一种）。

**macOS / Linux（软链，推荐）：**
```bash
git clone https://github.com/JacSam027/autopilot.git ~/autopilot
ln -s ~/autopilot ~/.claude/skills/autopilot
```

**Windows（PowerShell，需开发者模式或管理员权限建软链）：**
```powershell
git clone https://github.com/JacSam027/autopilot.git $HOME\autopilot
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\autopilot" -Target "$HOME\autopilot"
```

**或直接复制**整个目录到 `~/.claude/skills/autopilot/`（无需软链权限）。

装好后**重启一次 Claude Code 会话**，`/autopilot` 即可触发。

## 使用

### 1. 先标好目标 + 大致计划
在项目里放 `PLAN.md` / `TODO.md`，或在对话里说清。Orient 阶段会优先读这些；找不到会推断并记录，实在推断不出才停下。

### 2.（可选）填测试账号
复制 `accounts.local.example.md` 为 `accounts.local.md`，填入测试 / 超管账号。该文件已被 `.gitignore` 忽略，不会提交。

### 3. 触发
```
/autopilot
项目专属授权：8006 端口占用程序可启用 / 重启 / 关闭。本地 PC 测试。
（DB 默认允许读写、禁止删除；临时数据跑完再问是否清理。如需更严可声明「只读」。）
（push 默认禁止；需要 push 时请点名分支，如「可 push 到 feature-x」，且只动被点名的分支。）
目标见 PLAN.md。独立度高的任务可并行。
```

> 项目专属授权按项目变；**没声明的敏感操作一律走 HARD STOP**，不会擅自动。

### 4. 回来看汇报
agent 会在项目里生成 `SHIFT_REPORT.md`，并在对话里给出摘要，含：完成任务 / 修改内容 / 测试结果 / 问题与解决 / 待拍板决策清单 / 非功能性 review 发现。

## 五个阶段

| 阶段 | 做什么 |
|---|---|
| **Orient 定向** | 读你标好的目标 + 计划，建任务清单 |
| **Execute 执行** | TDD 推进；撞决策就 PARK 继续 或 HARD STOP；独立任务可并行；卡住止损跳走 |
| **Verify 验证** | 跑测试；用 playwright 做 E2E 和逻辑排查（本地 PC） |
| **Polish 打磨** | 功能基本完成后，对本次改动做**子智能体隔离 review**（复用 `/code-review`、`/simplify`） |
| **Report 汇报** | 生成结构化工作报告 |

## 关键设计

### 决策分类（PARK vs HARD STOP）
- **PARK**：命名 / 文案 / API 结构 / 范围扩展 / 有合理默认的歧义 → 记进 `DECISIONS.md`，按默认推进，继续干别的
- **HARD STOP**：删生产数据、不可逆改动、安全风险、超出声明授权、或全部被卡住 → 才停下等你

### 隔离 review（防自洽倾向）
同一上下文里，agent review 自己写的代码会下意识为自己找理由。Polish 阶段强制派子智能体开**新上下文**：一个只看代码挑毛病（**不传**实现理由，避免它锚定你的解释）、另一个照单改；主智能体不得因"我当初有理由"就一律否决 findings。

### 反过度设计
写能解决当前问题的最简代码：YAGNI、抽象等模式重复约 3 次再做、不过早优化、重构只保行为、最小改动、跟随既有风格。**双向平衡**——既揪重复，也揪"为了复用而过度抽象"。

### git / push 策略
默认禁止 push（自动跑完只做本地 commit）。只有你**明确点名**某个 / 某些分支时才允许 push，且**严格只动被点名的分支**——授权范围外的分支一律走 HARD STOP。force-push 需**单独**明确授权。这样既能在你离开时把活干完、又不擅自把改动推到你不想要的分支或远端。

## 目录结构

```
autopilot/
├── SKILL.md                    # 主文件（agent 读这个）
├── README.md                   # 本文件
├── accounts.local.example.md   # 测试账号模板（复制为 accounts.local.md 填真实账号，已 gitignore）
├── .gitignore                  # 排除 accounts.local.md 等敏感 / 运行时文件
└── templates/
    ├── DECISIONS.md            # 决策停车场模板（运行时在项目侧生成）
    └── SHIFT_REPORT.md         # 工作汇报模板
```

## 依赖与相关
- Claude Code 的 **Agent 工具**（子智能体）、**TaskList**
- 复用 `playwright-skill`、`/code-review`、`/simplify`、`/verify` 等已有 skill
