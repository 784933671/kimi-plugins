# kimi-plugins

自托管的 **Kimi Code** 插件市场，提供 UI/UX、Vue、JavaScript 和项目专用 API 能力包。由 zCode 插件库 [784933671/agents](https://github.com/784933671/agents) 迁移而来，遵循 [Kimi Code 插件规范](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html)。

## 安装

### 方式一：浏览整个市场（推荐）

在 Kimi Code 中执行：

```text
/plugins marketplace /path/to/kimi-plugins/marketplace.json
```

如果本仓库已推送到 GitHub，也可以直接使用 marketplace.json 的 raw URL：

```text
/plugins marketplace https://raw.githubusercontent.com/<owner>/<repo>/main/marketplace.json
```

### 方式二：安装单个插件

```text
/plugins install /path/to/kimi-plugins/plugins/ui-ux-craft
```

安装后运行 `/reload` 或 `/new` 使插件生效。用 `/plugins list` 查看已安装插件，`/plugins info <id>` 查看详情与诊断。

## 插件列表

| 插件 | 版本 | 说明 |
|------|------|------|
| `ui-ux-craft` | 1.1.1 | UI 设计、UI 修复、UX 功能测试与可访问性审查 |
| `vue-development` | 1.2.2 | Vue 3 / Composition API / Pinia / Vite / Element Plus / Vant 业务开发 |
| `javascript-development` | 1.0.1 | JavaScript ES2023+ / async-await / ESM / Node.js / 浏览器 API |
| `xuegong-system` | 1.2.1 | 学工系统 Apifox MCP 接入 |

### ui-ux-craft

- **Agents**：`ui-designer`（设计决策）、`ui-fixer`（最小范围修复）、`ui-ux-tester`（流程测试）、`accessibility-tester`（WCAG 审计）
- **Skills**：`ui-fix-playbook`、`ux-test-design`、`wcag-essentials`
- **命令**：`/ui-ux-craft:ui-walkthrough`（UI 走查）、`/ui-ux-craft:accessibility-scan`（无障碍扫描）

### vue-development

- **Agents**：`vue-expert`
- **Skills**：`vue`、`vue-best-practices`、`vue-component-design`、`vue-router-best-practices`、`pinia`、`vite`、`vue-api-integration`、`element-plus-forms`、`vant-forms`、`frontend-debugging`
- **命令**：`/vue-development:component-scaffold`（组件脚手架）、`/vue-development:vue-audit`（项目审计）

### javascript-development

- **Skills**：`javascript-pro`

### xuegong-system

- **Agents**：`xuegong-api-expert`
- **命令**：`/xuegong-system:xuegong-check`（MCP 连通性自检）
- **MCP**：`xuegong`（Apifox MCP Server，stdio）

**使用前必须配置 Apifox 访问令牌**：插件的 MCP 服务器从环境变量读取令牌，启动 Kimi Code 前在 shell 中导出：

```sh
export APIFOX_ACCESS_TOKEN=<你的 Apifox 访问令牌>
```

令牌在 Apifox → 个人设置 → API 访问令牌 中生成，需具备目标项目的读取权限。默认项目 ID 为 `7802011`；如需修改，安装后编辑 `~/.kimi-code/plugins/managed/xuegong-system/kimi.plugin.json` 中 `mcpServers.xuegong.args` 的 `--project-id`，然后 `/reload`。

## 仓库结构

```text
marketplace.json            # Kimi Code 自定义市场清单（version: "2"）
plugins/
  <plugin-id>/
    kimi.plugin.json        # 插件清单（Kimi Code 格式）
    agents/*.md             # 自定义子代理（自动发现）
    commands/*.md           # 斜杠命令（以 <plugin-id>:<command> 调用）
    skills/<skill>/SKILL.md # Agent Skills
```

## License

各插件目录下的 LICENSE 文件独立生效；其余内容遵循 MIT。
