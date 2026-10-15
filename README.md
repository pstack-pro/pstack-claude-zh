# pstack-claude-zh

pstack 中文版（预览）

把 [pstack](https://github.com/cursor/plugins/tree/main/pstack)（poteto 的严谨 agent 工作流）移植成 Claude Code 插件。这是中文预览版，版本 0.15.25-zh.2（见 `.claude-plugin/plugin.json`），版本号形如 `<上游版本>-zh.N`。跟踪上游 pstack 0.15.25（[cursor/plugins@e5136d8](https://github.com/cursor/plugins/commit/e5136d8)），上游版本记在 `UPSTREAM.json`。

只用 Claude 模型（fable / opus / sonnet / haiku），不调用任何外部模型或 CLI。

## 这一版翻译了什么

54 个 skill 的 description 改成纯中文，句子里直接嵌入 slash 命令和英文触发短语（如 `'arena this'`），所以用中文或英文提问都能匹配；不再附整段英文原文，每次会话少加载一半 token。`test/routing-eval/` 里有模拟路由评测（不等同于真实的 Claude Code / pi 路由）。

skill 正文、playbook、references 和 agents 已全部译成中文（上游 111 个 Markdown 文件全部译完）。这是预览质量的机器辅助翻译，难免有生硬或不准的地方；代码块、命令、工具名、路径、链接和脚本要解析的字符串都保持英文，并有脚本逐项核对。移植时新增或改写的文件（`overrides/` 里的 `setup-pstack`、`readonly` agent，以及本仓库自己的适配说明）仍是英文。**如果觉得中文版不好用，请安装英文版：[pstack-claude](https://github.com/pstack-pro/pstack-claude) 或 [pstack-pi](https://github.com/pstack-pro/pstack-pi)。**

## 前置条件

- [Claude Code](https://code.claude.com)（在 2.1.289 上测试过）
- Node.js 18 或更高版本，插件的 hook 是 Node 脚本

## 安装 / 试用 / 卸载

在要启用 pstack 的项目根目录执行：

```bash
# 只在当前项目启用（写入 .claude/settings.local.json，默认被 gitignore，不会提交）
claude plugin marketplace add pstack-pro/pstack-claude-zh --scope local
claude plugin install pstack@pstack-claude-zh --scope local

# 更新到 GitHub 上的最新版本
claude plugin marketplace update pstack-claude-zh

# 从当前项目卸载
claude plugin uninstall pstack@pstack-claude-zh --scope local
claude plugin marketplace remove pstack-claude-zh --scope local
```

插件名仍是 `pstack`，命令仍是 `/pstack:<name>`。不要在同一个项目里同时启用本插件和英文版 [pstack-claude](https://github.com/pstack-pro/pstack-claude)，两边的插件名都是 `pstack`。

用 local scope 安装时，其他项目的 `claude plugin list` 里会显示 pstack，但状态是 disabled，不会加载。

想改代码，或者只想在一次会话里试用，可以 clone 到任意目录，然后用本地路径：

```bash
git clone https://github.com/pstack-pro/pstack-claude-zh.git <任意目录>

# 只试一次：仅这次会话加载，不写任何配置
claude --plugin-dir <任意目录>

# 或者从本地目录安装到当前项目（Claude Code 会保存成绝对路径）
claude plugin marketplace add <任意目录> --scope local
claude plugin install pstack@pstack-claude-zh --scope local
```

**同一台机器上，GitHub 和本地目录两种来源只能选一种。** marketplace 名字 `pstack-claude-zh` 是全局共享的：再用另一种来源添加，会替换掉所有项目共用的来源；移除时也会把这条全局登记一起删掉，导致其他项目里的 pstack 失效。要切换来源，先在所有启用了 pstack 的项目里卸载，再用新来源重新安装。

## 开始用

1. `/pstack:setup-pstack`：写入 `.pstack/models.md`，默认使用 `claude-tiers` 方案，可以按角色修改。
2. `/pstack:poteto-mode <任务>`：主入口，会自动选择 playbook。

`claude-tiers` 方案：

| 角色 | 模型 |
|---|---|
| hardest tasks | `fable` |
| 判断和写作（judgment、how/why 的综合、reflect 的评审） | `opus` |
| 写代码（feature、bug-fix、perf、refactoring、swarm） | `sonnet` |
| 调查（how explorer、why investigators） | `haiku` |
| 评审组（arena、architect、interrogate） | `opus, sonnet, haiku` |

推理强度不按角色设置，跟随会话的 `/effort`。改了 `.pstack/models.md` 之后，`/clear` 或开一个新会话，新的配置才会注入。

如果不想把模型配置提交到项目仓库，把 `.pstack/` 加进项目的 `.gitignore`。

## 和 Cursor 版的差异

转换后的 skill 正文基本保持原样，差异由下面几处补上：

- `CLAUDE-CODE.md`：把 Cursor 的工具和功能逐项映射到 Claude Code，例如 `Task` 对应 `Agent`，`readonly` 对应 `pstack:readonly`，`environment: "cloud"` 对应 `isolation: "worktree"`，`AskQuestion` 对应 `AskUserQuestion`。主会话通过 SessionStart hook 注入这份说明；pstack 的 agent 和 poteto-mode 会自己读取。
- `hooks/session-start.mjs`：把说明、`.pstack/models.md` 和当前 transcript 的路径注入新会话。
- `hooks/allow-plugin-read.mjs`：自动放行对插件目录内文件的 Read、Glob、Grep，因为 skill 需要读取自己的 playbook。插件目录外的路径和指向外部的符号链接，仍然走正常的权限流程。
- `agents/readonly.md`：禁用编辑类工具的通用 agent，对应 Cursor 的 `readonly: true`。
- 配置文件放在 `.pstack/models.md`，而不是 `.claude/` 下。Claude Code 把 `.claude/` 当作敏感目录，每次写入都要确认。

## 翻译是怎么维护的

译文的共同来源在 `i18n/zh/`，Claude 版和 pi 版用同一套：

- `descriptions.json`：每个 skill 一条，有英文原文、中文和 `source_sha256`。原文一变，哈希对不上，这条就算过期。
- `glossary.md`：术语表。改译法时改这里。
- `bodies/<上游路径>`：正文译文，按上游 pstack 的目录结构存放；`bodies.json` 记录每个文件翻译时英文原文的哈希，上游一改就标为过期。`scripts/sync.mjs` 先把译文叠到上游副本上，再做和英文版相同的改写（命令前缀、路径、工具名）。

`scripts/sync.mjs` 从上游生成 `skills/` 和 `agents/` 之后调用 `scripts/apply-zh.mjs`，把中文 description 写进 skill，并把其中的 slash 命令改成这一边的前缀 `/pstack:`。移植时改过描述的 skill 写在 `i18n/zh-port.json`，覆盖共同译文里的同名条目。这边目前覆盖的是 `setup-pstack`。

`node scripts/validate-zh.mjs` 核对译文是否原样保留了标识符、命令、工具名、路径、代码、链接和触发短语。过期条目默认算失败。

每周的 GitHub Action（`.github/workflows/sync-upstream.yml`）会同步上游，再跑 `node scripts/validate-zh.mjs --allow-stale`。过期只记成警告。出现其他错误时工作流失败，不会开 PR。PR 正文附上校验输出，并列出其中包含 `stale` 的行，这些行对应的条目需要重译。推送和 PR 上的 `.github/workflows/validate.yml` 不加 `--allow-stale`，过期会让检查失败。

## 跟进上游

```bash
node scripts/sync.mjs                        # 自动浅克隆 github.com/cursor/plugins 到系统临时目录
node scripts/sync.mjs <plugins 仓库>/pstack   # 或者指定本地已有的 checkout
PSTACK_SRC=<plugins 仓库>/pstack node scripts/sync.mjs

test/smoke.sh                                # 离线检查：不调用模型，不消耗额度
```

`sync.mjs` 会重新生成 `skills/` 和 `agents/`，然后叠加 `overrides/`（`setup-pstack` 和 `readonly` agent），再套用中文 description。上游版本记录在 `UPSTREAM.json`。

## 相关仓库

- 英文版 Claude Code 移植：<https://github.com/pstack-pro/pstack-claude>
- 英文版 pi package：<https://github.com/pstack-pro/pstack-pi>
- pi 中文版：<https://github.com/pstack-pro/pstack-pi-zh>
- 上游 pstack：<https://github.com/cursor/plugins/tree/main/pstack>

## 致谢与许可

pstack 原作者 Lauren Tan（[cursor/plugins](https://github.com/cursor/plugins)），MIT 许可，见 `LICENSE-pstack`。

移植作者 Hacken Tao，MIT 许可。

`i18n/zh/glossary.md` 的部分条目改编自 [fenfenxu/pstack-zh](https://github.com/fenfenxu/pstack-zh)（MIT）。

> 说明：`make-bot-ui` 为 Grok Bot 场景设计，在 Claude Code 或 pi 里一般用不到。为与上游保持一致，仍保留。
