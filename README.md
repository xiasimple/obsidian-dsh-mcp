# Obsidian × DeepSeek Harness（DSH）MCP 接入

通过 **Model Context Protocol（MCP）** 把 Obsidian 笔记库接进 **DeepSeek Harness**，
让 DeepSeek 模型能直接**读、写、搜索、整理**你的 Obsidian 笔记。

模型最终会看到一组 `mcp__obsidian__*` 工具（共 14 个），与 Claude Code / Codex 使用的
server-qualified 命名形状一致。

## 原理

```
DeepSeek Harness (DSH)
   └── dsh-mcp-client            # DSH 内置的 MCP 客户端桥，1 个 server = 1 条插件配置
         └── obsidian-mcp-server # cyanheads 的 MCP server（stdio，npx 启动，14 个工具）
               └── Local REST API 插件 (Obsidian, HTTPS 127.0.0.1:27124)
                     └── 你的 vault（Markdown 文件）
```

- [`dsh-mcp-client`](https://www.npmjs.com/package/@deepseek-ai/dsh-mcp-client)：DSH 官方 MCP 桥接插件，把 MCP 工具注册成原生工具。
- [`obsidian-mcp-server`](https://github.com/cyanheads/obsidian-mcp-server)：包装 Obsidian Local REST API 的 TypeScript MCP server。
- [`Local REST API`](https://github.com/coddingtonbear/obsidian-local-rest-api)：Obsidian 社区插件，暴露本地 REST 接口。

## 前置条件

- DeepSeek Harness（DSH）已运行。
- Obsidian + **Local REST API** 插件 **v4.0.0 – v5.x**（⚠️ v6 移除了 section 写入所需的 wire 格式，不兼容）。
- Node.js **≥ 24**（`obsidian-mcp-server` 要求）。

## 接入步骤

### 1. 安装并启用 Local REST API 插件

Obsidian → 设置 → 第三方插件 → 关闭安全模式 → 浏览 → 搜 **Local REST API** → 安装 → 启用。

装完重载一次 Obsidian（Ctrl+P → `Reload app without saving`），插件会生成 API key 并监听 HTTPS 端口 `27124`。

### 2. 拿到 API key

Local REST API 插件设置页里会自动生成一个 API key（存在 vault 的
`.obsidian/plugins/obsidian-local-rest-api/data.json` 中，字段 `apiKey`）。

### 3. 写入 DSH 配置

把下面这段追加到你的 DSH profile 的 `cordis.patch.yml`（每个 MCP server 一条 `insert`）：

```yaml
- insert:
    - id: mcp-obsidian
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: obsidian
        transport: stdio
        command: npx
        args: ['-y', 'obsidian-mcp-server@latest']
        env:
          MCP_TRANSPORT_TYPE: 'stdio'
          # 推荐：从进程环境读取，不要把 key 写进仓库
          OBSIDIAN_API_KEY: !!js process.env.OBSIDIAN_API_KEY
          # Local REST API 的 HTTPS 端口（自签名证书，默认关闭校验）
          OBSIDIAN_BASE_URL: 'https://127.0.0.1:27124'
```

> `OBSIDIAN_API_KEY` 两种填法二选一：
> - **推荐**：在启动 DSH 前设置系统环境变量 `OBSIDIAN_API_KEY`，配置里用 `!!js process.env.OBSIDIAN_API_KEY` 读取；
> - **本地快速验证**：直接把 key 字符串填进去（`OBSIDIAN_API_KEY: '你的key'`），**但千万别把这种文件提交到 Git**。

完整模板见 [`cordis.patch.yml`](./cordis.patch.yml)。

### 4. 重载 DSH

编辑 `cordis.patch.yml` 会触发 HMR 热加载，无需重启进程；`serverName` 不变时工具名保持稳定。

### 5. 验证

重载后模型工具列表里会出现 `mcp__obsidian__*`，可先跑一个只读操作确认连通，例如
`mcp__obsidian__list_notes` 或 `mcp__obsidian__get_note`。

## 工具列表（14 个）

| 工具 | 说明 |
|:-----|:-----|
| `obsidian_get_note` | 读笔记：raw 内容 / 完整结构（含 frontmatter、tags）/ 文档地图 / 单节 |
| `obsidian_list_notes` | 列出某路径下的笔记与子目录 |
| `obsidian_list_tags` | 列出 vault 全部标签及使用次数 |
| `obsidian_list_commands` | 列出 Obsidian 命令面板命令（需 `OBSIDIAN_ENABLE_COMMANDS=true`） |
| `obsidian_search_notes` | 全文 / JSONLogic / Omnisearch 三种搜索，游标分页 |
| `obsidian_write_note` | 新建笔记，或按节替换（整文件覆盖需显式 `overwrite: true`） |
| `obsidian_append_to_note` | 追加内容；带 `section` 时追加到指定标题/块/frontmatter 字段 |
| `obsidian_patch_note` | 对标题/块/frontmatter 字段做 append / prepend / replace 精准编辑 |
| `obsidian_replace_in_note` | 正文内搜索替换（字面 / 正则、大小写、全词、捕获组） |
| `obsidian_manage_frontmatter` | 单个 frontmatter 键的 get / set / delete |
| `obsidian_manage_tags` | 增删查标签（frontmatter `tags:` 数组 / 行内 `#tag`） |
| `obsidian_delete_note` | 删除笔记（客户端支持时要求人工确认） |
| `obsidian_open_in_ui` | 在 Obsidian 界面中打开文件 |
| `obsidian_execute_command` | 按 ID 执行 Obsidian 命令（需 `OBSIDIAN_ENABLE_COMMANDS=true`） |

## 可选：路径级权限控制

`obsidian-mcp-server` 支持用环境变量圈定读写范围，默认不设 = 整个 vault 可读写：

| 变量 | 作用 |
|:-----|:-----|
| `OBSIDIAN_READ_PATHS` | 只读白名单（逗号分隔，前缀匹配、隐式递归） |
| `OBSIDIAN_WRITE_PATHS` | 只写白名单（写路径默认也可读） |
| `OBSIDIAN_READ_ONLY=true` | 全局只读开关，卸载所有写工具 |

## 安全

- **不要把 `OBSIDIAN_API_KEY` 提交进 Git**；用环境变量注入。
- Local REST API 使用自签名证书，`obsidian-mcp-server` 默认 `OBSIDIAN_VERIFY_SSL=false`，仅限本机 `127.0.0.1` 使用。
- 需要时用 `OBSIDIAN_READ_ONLY` / `OBSIDIAN_READ_PATHS` / `OBSIDIAN_WRITE_PATHS` 收缩权限。

## License

配置与文档：MIT。依赖的 `obsidian-mcp-server` 为 Apache-2.0，`dsh-mcp-client` 随 DeepSeek Harness 发布。
