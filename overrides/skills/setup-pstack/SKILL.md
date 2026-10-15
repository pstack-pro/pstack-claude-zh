---
name: setup-pstack
description: Configure which Claude model pstack uses per role, for this project only. Proposes the claude-tiers preset (fable for the hardest tasks, opus for judgment, sonnet for code, haiku for exploration) and writes .pstack/models.md, which the pstack SessionStart hook injects. Use for /pstack:setup-pstack, "configure pstack models", or changing pstack's model choices.
---

# Setup pstack（Claude Code）

在项目根目录写 `.pstack/models.md`。SessionStart hook 注入每个新会话，skill 选子代理模型时也会读。只影响本项目。

跟用户对话时用下面的中文提问和选项，别自己编英文。选项值保持英文：`fable` / `opus` / `sonnet` / `haiku` / `inherit-parent`。

每个角色都跑在 Claude 上。合法取值只有 `fable`、`opus`、`sonnet`、`haiku`，以及 `inherit-parent`（省略 `model`，子代理用会话模型）。**绝不**配置外部模型、CLI 或其他厂商的密钥。

## 预设：claude-tiers

更强的模型更吃订阅用量。最强的只放在真正改变结果的地方，写代码用更省的模型（pstack 会起很多写代码的子代理），探索用最快的。

```
feature, refactoring: sonnet
bug-fix: sonnet
perf-issue: sonnet
hillclimb: sonnet
judgment and prose: opus
hardest tasks: fable
how explorer: haiku
how explainer: opus
why investigators: haiku
why synthesizer: opus
reflect tooling: sonnet
reflect judgment, divergent, synthesizer: opus
arena runners: opus, sonnet, haiku
arena cross-judge pool: opus, sonnet, haiku
swarm workers: sonnet
architect runners: opus, sonnet, haiku
interrogate reviewers: opus, sonnet, haiku
```

## 步骤

### 1. 读当前配置

若已有 `.pstack/models.md`，读它，把其中的取值当作当前选择。否则从预设起步。角色不在预设里的行是退役角色，丢掉。

### 2. 套用用户请求

请求里已经点名模型或规则时（例如「判断也用 fable」或「不要 fable」），在预设上覆盖。映射里最强的模型仍应只给 `hardest tasks`，除非用户另有要求。

### 3. 展示并确认

用表格列出每个角色及其模型。说明评审组角色（arena runners、architect runners、interrogate reviewers）每个条目起一个子代理，所以列表长度就是数量；同一家 Claude 分档组成的评审组，交叉检查会弱于不同厂商。用 `AskUserQuestion` 确认。题干：`按这张表写入 .pstack/models.md？`；选项至少包括 `就这样`、`改几个角色`（再列出 `fable`、`opus`、`sonnet`、`haiku`、`inherit-parent`）。没法交互时直接采用并说明。

推理强度不按角色设置。子代理用会话的 `/effort`，或 agent 定义里的 effort。用户问起预算时提一次即可。

### 4. 写文件

写入 `.pstack/models.md`（没有 `.pstack/` 就建），整文件覆盖，保证重跑幂等：

```
# pstack model configuration for this project. One line per role. Delete a line to fall back to the skill default.
# Values: fable, opus, sonnet, haiku, or inherit-parent (session model).
# preset: claude-tiers
<one line per role>
```

文件内容里的英文注释与角色标签保持英文。**绝不**提交该文件，也**绝不**跑会改历史的 git 命令。问一次要不要把 `.pstack/models.md` 加进 `.gitignore`。题干：`要把 .pstack/models.md 加进 .gitignore 吗？`；选项：`加`、`不加`。没法交互时别改，回复里提一句就行。

### 5. 确认

告诉用户文件写好了。下一次子代理就会读到；hook 在新会话（或 `/clear` 后）注入。再跑本 skill 会更新；删掉就回默认。

### 6. 可选：提供验证 skill

看看项目有没有能真去点应用做验证的办法（`verify-*` skill，或现成 harness）。没有就问一次：`要不要给这个项目做一份验证 skill？代理可以像用户一样操作应用，证明改动有效。用 /pstack:create-verification-skill 生成。`；选项：`做一份`、`先不用`。选前者就跑该 skill，选后者继续，别再催。
