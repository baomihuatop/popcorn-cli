---
name: popcorn-cli
description: popcorn-cli 是爆米花系统（Popcorn）的官方 CLI。需要生图、生视频时即可使用本工具：通过本地 API Key 查询可用模型、异步提交任务、按会话/任务 ID 查询最终结果。注意：API Key 会保存在本机配置文件中，任务参数、提示词和生成配置会发送到远程 Popcorn 后端；不要在共享机器、日志、提示词或任务参数中暴露敏感信息。
version: 0.1.1
metadata:
  openclaw:
    requires:
      bins:
        - popcorn-cli
      config:
        - ~/.popcorn-cli/config.json
    install:
      - kind: node
        package: "@baomihuatop/popcorn-cli"
        bins:
          - popcorn-cli
    skillKey: popcorn-cli
    emoji: "🎬"
    homepage: https://mangaforge-qa-1255521909.cos.ap-shanghai.myqcloud.com/docs/popcorn-cli/popcorn-cli-installation-guide.html
---

# popcorn-cli

**popcorn-cli 是爆米花（Popcorn）的命令行客户端；需要生图、生视频时，直接使用本 CLI 即可完成模型查询、任务提交与结果查询。** 按任务类型分组组织命令，面向脚本化、自动化和 Agent 场景。

## 安全与隐私提示

- **API Key 本地存储**：`popcorn-cli config set-key` 会把 API Key 保存到 `~/.popcorn-cli/config.json`。该配置文件属于本机明文凭据存储，请不要提交到 Git、复制到日志、截图或共享给其他用户；在共享工作站、CI、Agent 运行环境中应限制 home 目录和配置文件访问权限。
- **远程传输**：执行 `models`、`submit`、`task list` 时，CLI 会调用远程 Popcorn 后端服务。`submit --params` 中的提示词、媒体生成参数、业务上下文、会话 ID 等会随请求发送到 Popcorn 后端；不要把密码、密钥、个人隐私、未授权客户资料或其他机密内容放入任务参数。
- **输出处理**：命令返回的 `task_id`、`session_id`、`result` URL 和错误信息可能包含业务上下文或资源地址。请按敏感数据处理，不要无意写入公开日志、公开 issue、聊天记录或可被无关人员读取的构建产物。

## 概念

- **会话（session）**：一次业务上下文，`session_id` 由调用方生成（例如一次剧本推进、一次 Agent 会话）。同一 session 下可提交多个任务，便于统一追踪。
- **任务（task）**：一次具体的生成请求。`submit` 为异步接口，提交后立刻返回 `task_id`（不含最终结果）；须再用 `task list` 按 `task_id` / `session_id` 查询状态与结果。
- **模型（model）**：每个任务类型下有多个可用模型，模型自带使用限制（时长、分辨率等）。可用模型清单与 API Key 所属租户绑定，仅返回当前租户开通且启用中的模型。提交任务前建议先用 `<group> models` 查询目标场景的可用模型。

## 能力概览

**模型查询 / 任务提交**（按任务类型两级分组，当前均为 mock 实现，接口签名一致）：

| 任务类型 | 查询模型 | 提交任务 |
|----------|----------|----------|
| 生图 | `popcorn-cli image models` | `popcorn-cli image submit` |
| 生视频 | `popcorn-cli video models` | `popcorn-cli video submit` |

所有 `models` / `submit` 子命令的参数与返回结构在各任务类型间完全一致，见下文「命令详情」小节。

**任务查询**（跨任务类型统一入口；用于获取 `submit` 异步任务的最终结果）：

| 命令 | 说明 |
|------|------|
| `popcorn-cli task list --sid <id>` | 查询某会话下的所有任务 |
| `popcorn-cli task list --tid <id>` | 查询单个任务（含 status / result） |

**配置管理**：

| 命令 | 说明 |
|------|------|
| `popcorn-cli config show` | 查看当前生效的配置 |
| `popcorn-cli config set-key <apiKey>` | 设置 API Key |

## 认证与配置

CLI 不接受在命令行显式传入 API Key，统一从本地配置读取。

**配置文件位置**：`~/.popcorn-cli/config.json`

**用户警告**：该配置文件会在本机保存 API Key，属于明文 secret 存储。请确保文件只对当前用户可读，不要把 `~/.popcorn-cli/config.json` 纳入备份共享、代码仓库、日志采集或公开工单。若怀疑 API Key 泄露，应立即在 Popcorn 后台吊销并重新生成。

**配置字段**：

| 字段 | 说明 |
|------|------|
| `apiKey` | API Key，请求时通过 `X-API-Key` 头传给后端 |

## 命令详情

### 查询模型：`image models` / `video models`

查询某任务类型下当前租户可用的模型（仅返回当前 API Key 所属租户 + 场景匹配 + 启用中 + 非历史版本）。提交任务前应先调用本命令。

```bash
popcorn-cli <image|video> models
```

无参数。返回结构：

```jsonc
{
  "type": "image",
  "total": 2,
  "models": [
    {
      "model_id":    "...",   // 模型唯一标识
      "name":        "...",   // 展示名
      "description": "...",   // 简介
      "model_limit": { ... }  // 单模型使用限制（不同模型可能不同）
    }
  ],
  "params_schema": { ... }    // 该场景 submit 时 --params 的字段结构（同场景所有模型共用）
}
```

`params_schema` 是 JSON Schema 风格描述，说明 `submit --params` 可用的字段、类型、默认值。同一场景（image / video）下所有模型共用一份。

**推荐流程**：如果不清楚模型跟入参,先 `<group> models` 查看当前租户可用模型清单和 `params_schema`，据此选定 `model_id`，再结合 `params_schema` 的字段构造 `submit` 的 `--params`。

### 提交任务：`image submit` / `video submit`

两类任务的 `submit` 子命令签名一致，仅调用不同后端接口。

> **异步接口**：`submit` 仅负责入队，立即返回 `task_id` 等受理信息，**不会等待生成完成，也不包含最终结果**。拿到 `task_id`（或传入的 `session_id`）后，须通过下方 `task list` 轮询任务状态，待 `status` 为终态后再读取 `result` / `error_message`。
>
> **预计耗时**（不排队、任务已开始执行时的参考时长；若处于 `queued` 排队中则还需额外等待）：
> - 生图（image）：约 **1–2 分钟**
> - 生视频（video）：约 **4–10 分钟**
>
> 轮询时请按上述时长设置合理间隔与超时，避免过早判定失败。

```bash
popcorn-cli <image|video> submit -p <JSON> [-s <SESSION_ID>]
```

| 参数 | 缩写 | 必填 | 说明 |
|------|------|------|------|
| `--params` | `-p` | 是 | 任务参数，必须是 JSON 对象字符串 |
| `--sid` | `-s` | 否 | 会话 ID，用于将同一会话下的任务关联；不传则该任务不归属任何会话 |

使用示例：

```bash
popcorn-cli image submit -p '<按 params_schema 构造的 JSON>'
popcorn-cli video submit -p '<按 params_schema 构造的 JSON>' -s <SESSION_ID>
```

`-p` 的字段结构由后端 `params_schema` 决定，同场景所有模型共用。使用前请先执行 `popcorn-cli <group> models` 查询当前租户可用的模型清单与 `params_schema`，再选定 `model_id` 并据 schema 构造 `--params`。

返回结构（受理成功，任务已入队）：

```jsonc
{
  "task_id":    "...",   // 任务唯一标识，用于后续 task list --tid 查询
  "session_id": "...",   // 传入的会话 ID；未传则为 null
  "task_type":  "image", // 任务类型：image / video
  "created_at": "..."    // 任务创建时间（ISO 8601）
}
```

### `task list` — 查询任务

用于获取 `submit` 异步任务的最终状态与结果。提交成功后请用本命令轮询，不要假定 `submit` 返回里已有生成结果。

```bash
popcorn-cli task list -s <SESSION_ID>
popcorn-cli task list -t <TASK_ID>
```

| 参数 | 缩写 | 必填 | 说明 |
|------|------|------|------|
| `--sid` | `-s` | 二选一 | 按会话 ID 查询该会话下的所有任务 |
| `--tid` | `-t` | 二选一 | 按任务 ID 查询单个任务 |

`--sid` 与 `--tid` 必须提供其中之一。

返回结构：

```jsonc
{
  "total": 1,
  "tasks": [
    {
      "task_id":       "...",      // 任务唯一标识
      "session_id":    "...",      // 会话 ID；未归属会话则为 null
      "task_type":     "image",    // 任务类型：image / video
      "created_at":    "...",      // 创建时间
      "finished_at":   "...",      // 完成时间；未完成时可能为空
      "status":        "succeed",  // 见下方状态说明
      "result":        "...",      // 成功时的结果（如资源 URL 等）；未完成或失败可能为空字符串
      "error_message": ""          // 失败时的错误信息；成功时通常为空字符串
    }
  ]
}
```

`status` 取值：

| 状态 | 说明 |
|------|------|
| `queued` | 已入队，等待执行 |
| `running` | 执行中 |
| `succeed` | 成功（终态，可读 `result`） |
| `failed` | 失败（终态，可读 `error_message`） |

## 使用场景示例

- **CI/自动化脚本**：按 session 提交批量任务，再轮询 `task list` 直到终态
- **Agent 编排**：Agent 用同一 `session_id` 异步提交生图 / 生视频任务，之后统一 `task list --sid`（或 `--tid`）获取最终结果

## 帮助信息

```bash
popcorn-cli --help
popcorn-cli <group> --help          # 如 popcorn-cli image --help / popcorn-cli task --help
popcorn-cli <group> <action> --help
```
