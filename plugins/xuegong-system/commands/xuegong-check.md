---
description: 检查学工系统 MCP 是否连通、鉴权是否通过。
argument-hint: "[可选: 重试次数或备注]"
---

# 学工系统 MCP 自检

你是学工系统接口助手，负责对 `xuegong` MCP 服务器做一次端到端的可用性自检。目标是用一次最小接口调用，区分"连接层"和"鉴权层"两类问题，给出明确的可执行结论。

## 自检参数

$ARGUMENTS

## 自检流程

### 1. 探测 MCP 连接

确认 `xuegong` MCP 服务器处于可用状态：

- 清点可用的 `mcp__xuegong__*` 工具（如 `read_project_oas`、`read_project_oas_ref_resources`、`refresh_project_oas`）
- 若**没有任何 `mcp__xuegong__*` 工具**，直接判定为 `❌ MCP 未连接`，跳到"输出 / 未连接"小节

### 2. 发起最小鉴权调用

调用 `read_project_oas` 读取默认模块的 OpenAPI Spec。这是最轻、且必须鉴权的接口，足以验证 token 是否有效。

观察返回结果：

- **成功返回 Spec（任意非鉴权错误）** → 鉴权通过
- **返回 401 / Unauthorized / 鉴权失败 / token 相关错误** → 连接通但 token 无效
- **连接级错误（超时、进程退出、Connection closed）** → 连接层问题

### 3. 必要时复验

若第 2 步返回内容为空或疑似缓存，可调用 `refresh_project_oas` 强制重新拉取，再回到第 2 步判定。

## 输出

按结果三选一输出，并在末尾给出可执行的下一步。

### ✅ MCP 连通 + 鉴权通过

- 服务器状态：connected
- token：有效
- 已读取到的接口模块：`<列出从 Spec 看到的模块/分组数量或名称>`
- 结论：可以正常使用学工系统 MCP 工具查询接口

### ⚠️ MCP 连通但鉴权失败

- 服务器状态：进程已启动，但接口返回鉴权错误
- token：无效 / 过期 / 无目标项目权限
- 下一步：
  1. 确认已在 shell 中 `export APIFOX_ACCESS_TOKEN=<你的令牌>`（需具备目标项目读取权限），并从该 shell 启动 Kimi Code
  2. 如需修改项目 ID，编辑 `~/.kimi-code/plugins/managed/xuegong-system/kimi.plugin.json` 中 `mcpServers.xuegong.args` 里的 `--project-id`
  3. 运行 `/reload` 或新建会话，再次运行 `/xuegong-system:xuegong-check`

### ❌ MCP 未连接

- 服务器状态：未在工具列表中出现，或 `/mcp` 显示连接失败
- 下一步：
  1. 运行 `/mcp` 查看 `xuegong` 的状态和错误信息；用 `/plugins info xuegong-system` 查看插件诊断
  2. 常见原因：
     - `command not found` / `spawn npx ENOENT` → `npx` 不在 PATH，需先安装 Node.js
     - 启动超时 → 在服务器配置中加 `startupTimeoutMs`（如 `60000`）
     - 鉴权环境变量缺失 → 确认已 `export APIFOX_ACCESS_TOKEN` 再启动 Kimi Code
     - 服务器被禁用 → 执行 `/plugins mcp enable xuegong-system xuegong` 后 `/reload`
  3. 修复后 `/reload` 或新建会话，再次运行 `/xuegong-system:xuegong-check`

## 原则

- 只跑**一次**最小接口调用，不批量请求
- 不猜测错误：错误信息要来自实际返回
- 区分"连不上"和"鉴权不过"两类，它们修法不同
- 末尾必须给出下一步动作，不要停在"有问题"
