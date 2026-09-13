# self-teaching-skill

可跨 agent 复用的 skill 集合。目前包含一个 skill：

| Skill                                                 | 解决的问题                                                            |
| ----------------------------------------------------- | ---------------------------------------------------------------- |
| [`self-teaching-mode`](./self-teaching-mode/SKILL.md) | 当用户以**自我教学**为目的让 AI 生成代码时，强制 AI 先侦查项目、确认用户的前置基础、写教学性注释、真实验证并清理环境 |

> 这个 skill 提炼自一次真实的会话：用户是"看得懂 Python 基础语法、但 FastAPI 忘得差不多了"的学习者，
> 要求 AI 在空项目里生成一个 FastAPI + MySQL 增删改查示例。
> 过程中暴露出的关键点——**先问清基础、注释只讲"为什么"、起临时 MySQL 实例真跑一遍、
> 验证完把临时环境清干净**——都被固化成了这个 skill 的硬性流程。

---

## 目录结构

```
my_skills/
├── README.md                          # 本文件
└── self-teaching-mode/
    └── SKILL.md                       # skill 本体（唯一的源文件）
```

**一份 `SKILL.md`，三种 agent 通用**——它遵循开放的 [Agent Skills](https://agentskills.io/specification)
标准，只用了两个必需字段 `name` 和 `description`，因此不依赖任何厂商私有字段。

关于语言：`SKILL.md` 正文为**英文**（适配各家 LLM，不绑定某一种语言环境），
但 `description` 里同时保留了英文和中文触发词，中文提问同样能命中；本 README 保持中文，因为
**LLM 加载的只有 `SKILL.md`**，README 是写给人看的。

---

## 兼容性

| Agent           | 支持的形态                            | 用户级安装路径                            | 仓库级安装路径                          |
| --------------- | -------------------------------- | ---------------------------------- | -------------------------------- |
| **Claude Code** | 目录 + `SKILL.md`                  | `~/.claude/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| **Codex**       | 目录 + `SKILL.md`（Agent Skills 标准） | `~/.agents/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` |
| **Reasonix**    | **单文件** `<name>.md`              | `~/.reasonix/skills/<name>.md`     | `.reasonix/skills/<name>.md`     |

> Claude Code 注意：`skills/` 目录下**不支持单文件** `.md`，必须是 `<name>/SKILL.md` 的目录形式，
> 且 frontmatter 里的 `name` 必须与目录名完全一致。
> 
> Codex 注意：`~/.codex/prompts/*.md` 那套自定义 prompt 已被官方标记为**弃用**，
> 只支持显式 `/prompts:<name>` 调用；现在推荐使用上面的 skills 机制。

---

## 安装

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -r self-teaching-mode ~/.claude/skills/
```

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse .\self-teaching-mode "$env:USERPROFILE\.claude\skills\"
```

### Codex

```bash
mkdir -p ~/.agents/skills
cp -r self-teaching-mode ~/.agents/skills/
```

只想在某个仓库里生效的话，把 `self-teaching-mode/` 复制到仓库根的 `.agents/skills/` 下，
跟代码一起提交，团队就都有了。

### Reasonix

Reasonix 读的是**单文件**，所以要把 `SKILL.md` 复制成 `<name>.md`：

```bash
mkdir -p ~/.reasonix/skills
cp self-teaching-mode/SKILL.md ~/.reasonix/skills/self-teaching-mode.md
```

> 注意这是**副本**不是链接：改了仓库里的 `SKILL.md` 之后，要重新执行一次这条 `cp` 才会生效。

### 兜底：agent 不支持 skills 怎么办

把 `SKILL.md` 的正文（去掉 frontmatter）粘贴进项目根的 `AGENTS.md`（Codex / 通用）
或 `CLAUDE.md`（Claude Code）即可——只是会常驻上下文，不如 skill 的按需加载省 token。

---

## 怎么触发

- **自动触发**：`description` 里写的是触发条件，英文如 "I want to learn X"、"teach me from scratch"，
  中文如"我想学 X""帮我写个能看懂的示例""我是新手""加注释讲给我听"，都会被匹配到。
- **手动触发**：
  - Claude Code：输入 `/self-teaching-mode`
  - Codex：在对话里点名该 skill，或用 `/skills` 选择
  - Reasonix：输入 `/self-teaching-mode`

### 验证是否生效

对 agent 说：

> 我是新手，帮我写一个用 Python 读 CSV 并统计的脚本，我想学一下。

**生效的表现**：它会先反问你基础（比如"用过 `pandas` 吗？列表推导式熟不熟？"），
而不是直接把代码贴给你。**没生效的表现**：上来就是一整段代码——说明 skill 没被加载，
检查目录名与 frontmatter `name` 是否一致，然后重启 agent（Codex 在会话启动时扫描 skills）。

---

## 这个 skill 的核心约定

四步硬性流程，**每一步都不许跳**：

1. **先侦查项目现状** —— 现有文件、已装依赖及**版本**、本机可用的服务与工具（数据库在不在跑、Docker 可用吗、端口占用），再动手。
2. **确认前置基础** —— 推导出本次学习内容必需的前置知识清单，用**选择题**逐项确认会不会；缺前置就先补最小示例，答"不知道"就按零基础处理。
3. **生成代码 + 教学注释** —— 只给用户不具备的知识点加注；注释解释**为什么**而非复述"这行做什么"；
   新概念首次出现就地解释，同一概念不重复；解释用用户的语言、代码与术语保持原文；标出需用户替换的占位值。
4. **验证 → 清理 → 交付说明** —— 能用真实依赖就用真实依赖跑通；用不了就自建临时环境（临时实例/目录/端口）验证，
   **验证完立即清理**并确认用户原环境状态未变；必须真跑并覆盖错误分支；
   交付说明要含文件职责表 + 阅读顺序 + 环境准备步骤 + 针对本次内容的常见疑问 + "我没验证到什么、为什么" + 进阶练习清单。

配套的**反模式表**和**红线清单**在 `SKILL.md` 里，可以直接拿来自查。

---

## 自定义

- **改触发条件**：编辑 `SKILL.md` 的 `description`。写清"什么时候用"，不要在里面概括流程——描述是路由契约，写得太笼统会导致该触发时不触发。
  ⚠️ `description` 是 **YAML plain scalar**：值里**不能出现 `: `（冒号加空格）**，否则整个 frontmatter 解析失败，
  skill 会被**静默丢弃**（症状是 agent 里能看到 skill 名字却没有描述，或干脆不触发）。要写"中文触发词"这类引导语，
  用 `—`、`=` 或括号代替冒号。改完请用真正的 YAML 解析器确认，别靠肉眼（需先 `pip install pyyaml`）：
  `python -c "import re,yaml,pathlib; t=pathlib.Path('self-teaching-mode/SKILL.md').read_text(encoding='utf-8'); print(yaml.safe_load(re.match(r'^---\n(.*?)\n---\n', t, re.S).group(1)))"`
- **改流程**：直接编辑正文；它是普通 Markdown，不需要重新编译或打包。
- **改语言**：正文目前是英文；想换回中文只需改写正文，加不加中英双语触发词的取舍见下面的说明。
  `description` 里同时保留英文与中文触发词，是为了让中英文提问都能命中——如果只面向单一语言的用户群，可以精简成一种。
- **加平台私有字段**：只有 Claude Code 认 `allowed-tools`、Reasonix 认 `runAs`/`scope`。
  为了三平台通用，本 skill 刻意只用 `name` + `description`；如需添加，请确认目标 agent 会忽略未知字段。

---

## 说明

- 各 agent 的 skill 发现路径与字段支持仍在演进，路径请以官方文档为准
  （[Claude Code](https://code.claude.com/docs/en/skills)、[Agent Skills 规范](https://agentskills.io/specification)）。
- 本目录内容可自由修改、分发。
