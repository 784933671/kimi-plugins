# AGENTS.md

这是一个面向 **Kimi Code** 的自托管插件市场仓库，不是业务应用源码。模型在本仓库工作时，应把它理解为"Kimi Code 插件市场"：维护根目录的 `marketplace.json`，并在 `plugins/` 下按能力包组织可安装插件。

## 仓库结构

- `marketplace.json` — Kimi Code 自定义市场清单，格式为 `{"version": "2", "plugins": [{"id", "displayName", "description", "source"}]}`，`source` 使用 `./plugins/<id>` 相对路径
- `plugins/<id>/kimi.plugin.json` — 插件清单（Kimi Code 格式，字段见下）
- `plugins/<id>/agents/*.md` — 自定义子代理，`agents/` 目录存在时自动发现，无需在清单声明
- `plugins/<id>/commands/*.md` — 斜杠命令，必须在清单的 `commands` 字段声明 `./commands/`；安装后以 `<plugin-id>:<command>` 调用，正文用 `$ARGUMENTS` 接收参数
- `plugins/<id>/skills/<skill>/SKILL.md` — Agent Skills，必须在清单的 `skills` 字段声明 `./skills/`

## kimi.plugin.json 规则

- `name` 即插件 id，必须匹配 `[a-z0-9][a-z0-9_-]{0,63}`
- 展示信息放在 `interface`（`displayName`、`shortDescription`、`longDescription`、`developerName`、`websiteURL`）
- 所有路径字段（`skills`、`commands`、`agents`、`systemPromptPath`、`mcpServers` 内的 `./` 命令）必须位于插件根目录内
- MCP 服务器在清单的 `mcpServers` 字段声明，server 名用 ASCII（工具名形如 `mcp__<server>__<tool>`）；stdio 服务器由 `command` 字段推断，不要写 `type`
- 不支持的运行时字段（`tools`、`apps`、`inject`、`configFile` 等）会被忽略并记入诊断，不要添加

## Agent 文件规则

- frontmatter 字段：`name`（kebab-case）、`description`（必填，指导主 Agent 委派）、`whenToUse`、`tools`、`disallowedTools`、`subagents`
- 工具名必须使用 Kimi Code 命名：`Read`/`Grep`/`Glob`/`Bash`/`Edit`/`Write`/`FetchURL`/`WebSearch`/`TodoList`/`Agent` 等；MCP 工具用 glob 如 `mcp__github__*`
- 不要写 Claude Code 专有字段（`model`、`color`）或工具名（`WebFetch`、`TodoWrite`、`Task`）

## 维护原则

- 优先复用现有插件、代理、技能，不重复创建相同能力
- 新能力放在语义最贴近的插件中；跨领域能力才新增插件
- 代理适合较重的流程、审查、修复、测试和跨步骤任务；技能适合轻量知识、规范、API 模式
- 新增/删除插件或改名时，同步更新根 `marketplace.json` 和 `README.md` 的插件列表
- 修改插件内容后，按语义化版本更新该插件 `kimi.plugin.json` 的 `version`
- 敏感信息（token、密钥）一律通过环境变量注入（如 `APIFOX_ACCESS_TOKEN`），不得写入清单或文档

## 验证

修改插件后执行：

```sh
# JSON 合法性
python3 -m json.tool marketplace.json > /dev/null
python3 -m json.tool plugins/<id>/kimi.plugin.json > /dev/null
```

然后在 Kimi Code 中用 `/plugins install ./plugins/<id>` 本地安装，并用 `/plugins info <id>` 确认无诊断告警。
