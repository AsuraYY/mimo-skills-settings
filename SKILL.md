---
name: mimo-skills-settings
description: 配置 MiMo Desktop 的 skill 安装与斜杠命令对齐。用于：从本地路径安装/适配 skill 到 MiMoCode 目录；检查并修复 skill 目录结构（Junction、locales、frontmatter）；保证加号更多中的 skill 与斜杠命令一一对应；清理覆盖内建命令的自定义 command 以及重复调用同一 skill 的斜杠命令。当用户要求安装 skill、调整 skill 结构、对齐斜杠与加号、或给出本地 skill 路径要求适配 MiMo 时使用。
---

# MiMo Skills Settings

统一管理 MiMo Desktop 的 skill 安装结构，以及「加号更多」skill 库与「斜杠命令」的一致性。

## 硬性规则（必须遵守）

1. **绝不覆盖应用自带行为**
   - 不得创建与内建斜杠同名的 `command/<name>.md`（如 `/loop`、`/deep-research`、`/compose-next`、`/goal`、`/dream`、`/distill`、`/init`、`/review` 等）。
   - 不得修改 engine-config、builtin 内置 skill 的内容。
   - 若发现已有 command 文件覆盖了内建命令，删除该文件，保留内建实现。

2. **安装本地 skill 时不改动 skill 核心内容**
   - 只做目录结构适配（复制、补 locales、校验 frontmatter），不改写 SKILL.md 正文、章节、脚本等实际内容。
   - 优先 **复制**，不用 Junction/符号链接作为 MiMo 侧的 skill 目录。

3. **斜杠命令与加号更多保持一致**
   - 加号中出现的每个 skill，在斜杠中都应有对应入口。
   - 同一 skill 不允许存在多个重复的自定义斜杠命令。

## 目录约定（mimo-skill-authoring）

MiMoCode 是 **唯一写入目标**：

| 类型 | 路径 |
|------|------|
| 全局 skill | `~/.config/mimocode/skills/<id>/` |
| 项目 skill | `<project>/.mimocode/skills/<id>/` |
| 全局斜杠 command | `~/.config/mimocode/command/<name>.md` |

品牌根 **不要写入、也不要当作已安装位置**（MiMo Desktop 不扫描）：

- `~/.claude/skills`、`~/.codex/skills`、`~/.agents/skills`、`~/.opencode/skills`

### 合格 skill 目录结构

```
<id>/
├── SKILL.md          # frontmatter: name（=目录名，仅字母数字连字符）、description 非空
├── locales/
│   ├── zh-CN.json    # 仅 displayName + brief
│   └── en-US.json
└── （其余原有文件保持不动）
```

`locales/*.json` 示例（以 skill 目录名 `docx-official` 为例）：

```json
{
  "displayName": "docx-official",
  "brief": "一句话说明"
}
```

**displayName 统一规则**：所有语言的 `displayName` 必须与 skill 文件夹名称（即 skill id / frontmatter `name`）完全一致，不因语言而改写、翻译或美化。文件夹叫什么，displayName 就是什么，保持全局统一。

禁止把稳定 `name` 或 Agent 用的 `description` 写进 locales。

### 合格斜杠 command 格式

`~/.config/mimocode/command/<skill-id>.md`：

```markdown
---
description: <displayName>：<brief>
agent: build
---

加载并使用 `<skill-id>` 技能。若未自动加载，先通过 Skill 工具加载 `<skill-id>`，再严格按该技能的说明回应。

用户请求：$ARGUMENTS
```

说明：

- 文件名即斜杠名：`docx-official.md` → `/docx-official`
- 已有内建斜杠的 skill（至少：`loop`、`deep-research`、`compose-next`）**不要**再建 command 文件
- 切勿用短别名 command 去重复调用已有 command 的同一 skill

## 工作流程 A：从本地路径安装 skill

用户提供本地 skill 目录路径时：

1. **读取源目录**：确认存在 `SKILL.md`；解析 frontmatter 的 `name`、`description`。
2. **校验 ID**：`name` 必须匹配 `^[A-Za-z0-9-]+$`；与目标目录名一致。不合法则向用户说明，不要静默改名装入。
3. **确定落点**：默认全局 `~/.config/mimocode/skills/<id>/`；用户明确要求项目级时用 `<project>/.mimocode/skills/<id>/`。
4. **处理已有目标**：
   - 若目标是 Junction/ReparsePoint：用 `cmd /c rmdir` **只删链接**（勿跟随删除目标），再复制。
   - 若目标已是真实目录：向用户确认是覆盖还是跳过。
5. **复制**：完整复制源目录到目标（保持原文件内容，不做删改）。
6. **补齐 locales**（仅当缺失时）：
   - `displayName` **一律使用 skill 文件夹名称（skill id）**，所有语言文件保持相同值，不做翻译或改写
   - 从 SKILL.md 的 description 提炼简短 `brief`（可按语言分别撰写）
   - 写入 `locales/zh-CN.json`、`locales/en-US.json`（只有两个字段）
7. **同步斜杠**：
   - 若 `~/.config/mimocode/command/<id>.md` 不存在，且 `<id>` 不在内建斜杠名单中 → 按上面模板创建
   - 若已存在同 skill 的其他别名 command → 删除重复，只留一个
8. **校验**：目标非 ReparsePoint；frontmatter 有效；locales 存在；command 与 skill 一一对应。

源路径若在品牌根，仍 **复制** 到 MiMoCode 路径；不要在品牌根新建，也不要只留 Junction。

## 工作流程 B：全量对齐检查

用户要求「对齐斜杠和加号」或「检查 skill」时：

1. **枚举加号 skill 库**（三个扫描根，跳过 `@*` 命名空间目录）：
   - `~/.config/mimocode/skills/`
   - `~\AppData\Roaming\Xiaomi MiMo\engine-config\skills\`（只读检查）
   - `~\.local\share\mimocode\builtin_skills\desktop-*/skills\`（只读检查）
2. **枚举斜杠 command**：`~/.config/mimocode/command/*.md`
3. **发现问题则修复**：
   - 覆盖内建斜杠 → 删除该 command 文件
   - 多个 command 调用同一 skill → 只保留与 skill-id 同名的一个，删别名
   - skill 有、斜杠无 → 按模板补 command（跳过内建斜杠名单）
   - skill 目录是 Junction 指向品牌根 → 转为真实副本
   - 缺 locales / frontmatter 非法 → 按约定补齐（不改正文）
   - locales 中 `displayName` 与文件夹名称不一致 → 改为文件夹名称（所有语言统一）
4. **空壳目录**（有目录无 SKILL.md）：报告给用户，**不要**擅自编造 skill；删除与否由用户决定。
5. **输出对照表**：加号数量、斜杠数量、已修复项、跳过项（内建/空壳）。

## 内建斜杠名单（不可创建同名 command）

至少包括：`loop`、`loops`、`deep-research`、`compose-next`、`init`、`review`、`goal`、`dream`、`distill`、`rebuild`、`context-limit`、`sessions`、`models`、`agents`、`skills`、`login`、`connect`、`help`、`exit` 等应用/会话/内建 prompt 命令。

不确定某名字是否内建时：先对照 `mimocode-docs` 的 `@reference/commands.md`，确认后再决定是否建 command。

## 完成时汇报

每次操作结束时简要说明：

- 做了什么（安装/修复/删除/新建）
- 路径
- 加号与斜杠当前是否一一对应
- 需要用户知晓的例外（内建跳过、空壳保留、需新开对话生效等）

命令与 skill 热加载；若界面未刷新，提醒用户新开对话。
