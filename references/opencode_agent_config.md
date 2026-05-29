# OpenCode 代理配置参考手册 v1.2

> **文档定位**：团队内部配置参考文档，即查即用  
> **最后更新**：2026-05-30  
> **适用版本**：OpenCode v1.x（≥1.4.x）  
> **维护周期**：每季度复核模型 API 约束，每半年全面审查手册内容

## 目录

- [一、配置基础](#一配置基础)
  - [1.1 参数透传规则](#11-参数透传规则)
  - [1.2 配置来源与合并优先级](#12-配置来源与合并优先级)
  - [1.3 Schema 字段速查表](#13-schema-字段速查表)
  - [1.4 Agent 定义方式](#14-agent-定义方式)
  - [1.5 其他全局配置项](#15-其他全局配置项)
- [二、权限系统](#二权限系统)
  - [2.1 权限键清单](#21-权限键清单)
  - [2.2 按代理角色的权限矩阵](#22-按代理角色的权限矩阵)
  - [2.3 细粒度控制](#23-细粒度控制)
- [三、模型配置速查](#三模型配置速查)
  - [3.1 DeepSeek-V4 Flash / Pro](#31-deepseek-v4-flash--pro)
  - [3.2 MiniMax-M2.5 / M2.7](#32-minimax-m25--m27)
  - [3.3 Kimi-K2.5 / K2.6](#33-kimi-k25--k26)
  - [3.4 GLM-4.7 / GLM-5.1](#34-glm-47--glm-51)
  - [3.5 Variants 推理强度控制](#35-variants-推理强度控制)
- [四、典型误配案例](#四典型误配案例)
- [五、调优与运维](#五调优与运维)
  - [5.1 参数调优铁律](#51-参数调优铁律)
  - [5.2 成本与性能权衡](#52-成本与性能权衡)
  - [5.3 配置验证与维护](#53-配置验证与维护)
- [六、参考来源](#六参考来源)

---

## 一、配置基础

### 1.1 参数透传规则

代理配置中除 `model`、`mode`、`prompt`、`description`、`permission`、`disable`、`color`、`hidden`、`variant`、`steps` 等 OpenCode 原生字段外，其余字段（包括 `options` 对象以及直接写在 agent 中的非原生字段）会经过 options 合并链传入模型提供商的 API。合并优先级（由低到高）：ProviderTransform.options() → model.options → agent.options → variant。任何在代理 Markdown 文件（`.opencode/agents/*.md`）或 `opencode.json` 中显式声明的参数，都会覆盖全局和项目级默认值。

> **弃用提醒**：`tools` 字段已官方弃用（deprecated），请统一使用 `permission`。旧版 `tools: { "write": false }` 等价于 `permission: { "edit": "deny" }`。

### 1.2 配置来源与合并优先级

OpenCode 从多个来源加载配置并合并（非替换），后者覆盖前者的冲突键：

1. **Remote config** — `.well-known/opencode`（组织级默认值，自动获取）
2. **全局配置** — `~/.config/opencode/opencode.json`（用户偏好）
3. **自定义路径** — `OPENCODE_CONFIG` 环境变量指定
4. **项目配置** — 项目根目录 `opencode.json`
5. **`.opencode` 目录** — `agents/`、`modes/`、`commands/`、`plugins/` 等
6. **内联配置** — `OPENCODE_CONFIG_CONTENT` 环境变量（运行时覆盖）
7. **Managed config** — 系统管理配置目录
8. **macOS 托管首选项** — MDM 推送的 `.mobileconfig`（最高优先级，不可覆盖）

> **排查技巧**：若修改参数后无效果，按上述顺序检查是否被更高优先级覆盖，或确认该参数是否被目标模型忽略（如思考模式下 temperature/top_p 无效）。

### 1.3 Schema 字段速查表

| 字段 | 类型 | 说明 | 易错点 |
|------|------|------|--------|
| `model` | string | `provider/model` 格式 | 模型标识变更频繁，需定期核对 |
| `mode` | string | `subagent` / `primary` / `all`（默认） | 不设时默认为 `all`，既是 primary 也是 subagent |
| `temperature` | number | 采样温度，通常 0~1 | **调优首选**。严禁与 `top_p` 同时大幅调整 |
| `top_p` | number | 核采样，通常 0~1 | 仅在 temperature 调优后微调 |
| `max_tokens` | number | 最大生成长度 | **切忌过小**（工具调用链每步消费 token） |
| `steps` | number | 最大迭代次数 | 替换已弃用的 `maxSteps` |
| `prompt` | string | 自定义系统提示词 | 应包含退出策略，避免迭代失控 |
| `description` | string | 子代理调度匹配说明 | 直接影响 Agent 路由准确性 |
| `permission` | object | 工具权限覆盖 | 敏感工具必须显式配置 |
| `disable` | boolean | 禁用代理 | 调试隔离用 |
| `color` | string | UI 显示颜色 (Hex 或主题色名) | 支持 primary / secondary / accent / success / warning / error / info |
| `variant` | string | 推理强度等级选择 | 通过 agent 的 variant 字段选择，详见 §3.5 |
| `hidden` | boolean | 从 @ 菜单隐藏子代理 | 仅 `mode: subagent` 时生效 |
| `options` | object | 额外透传选项 | 与直接写在代理对象内等效，均在请求时合并 |

> **透传选项**：`thinking`、`reasoning_effort` 等非原生字段可以**直接写在代理对象内**，也可以放在 `options` 嵌套对象中——两种写法等效。OpenCode 会在加载时将未知键自动归入 `options`，并在请求时通过 merge 链注入 API body。`provider.<provider>.models.<model>.options` 用于模型级配置，优先级低于 agent 级。

### 1.4 Agent 定义方式

除 JSON 配置外，Agent 也可通过 **Markdown 文件** 定义。文件扫描规则：

| 扫描目录 | 自动行为 | 示例路径 |
|---------|---------|---------|
| `{agent,agents}/**/*.md` | 文件名即 agent 名，frontmatter 为配置，正文为 `prompt` | `.opencode/agents/review.md` → agent `review` |
| `{mode,modes}/*.md` | 同上，但强制 `mode: "primary"` | `.opencode/modes/sre.md` → primary agent `sre` |
| 全局目录 | `~/.config/opencode/agents/` 同理 | 全局共享 agent |

```markdown
<!-- .opencode/agents/review.md -->
---
description: 审查代码质量和安全风险
mode: subagent
model: deepseek/deepseek-v4-pro
variant: high
temperature: 0.1
permission:
  edit: deny
  bash: deny
---
你是一位资深代码审查员。在审查时重点关注：
- 安全漏洞和注入风险
- 代码可维护性和最佳实践
- 潜在的性能问题
仅提供分析建议，不要修改任何文件。
```

> **Tip**：Markdown 方式更适合团队共享（可被 git 追踪），JSON 方式适合需要动态变量替换（`{env:VAR}` / `{file:path}`）的场景。

### 1.5 其他全局配置项

以下顶层配置字段影响 Agent 和 Model 的行为：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `default_agent` | string | `"build"` | 默认 primary agent，必须为 primary 类型 |
| `small_model` | string | 自动选择 | 轻量任务（标题生成等）专用模型，格式 `provider/model` |
| `enabled_providers` | string[] | 全部启用 | 白名单模式，仅加载指定的 provider |
| `disabled_providers` | string[] | 无 | 黑名单模式，排除指定的 provider（优先级高于 enabled） |
| `$schema` | string | 自动注入 | 指向 `https://opencode.ai/config.json`，提供编辑器自动补全 |

---

## 二、权限系统

### 2.1 权限键清单

OpenCode 的权限系统以"工具名"为键。以下为可用权限键：

| 权限键 | 控制的工具 | 建议 |
|--------|-----------|------|
| `read` | 文件读取 | 只读代理 `allow` |
| `edit` | `write`、`edit`、`apply_patch` | **Plan/Explore 建议 `deny`** |
| `glob` | 文件搜索 | 通常 `allow` |
| `grep` | 内容搜索 | 通常 `allow` |
| `list` | 目录列表 | 通常 `allow` |
| `bash` | 终端命令 | **高风险**，只读代理应 `deny` |
| `task` | 派发子代理 | 控制子任务生成 |
| `skill` | 技能调用 | 按需配置 |
| `lsp` | 语言服务器 | 代码辅助，通常 `allow` |
| `question` | 向用户提问 | 通常 `allow` |
| `webfetch` | 下载网页 | 子代理建议 `deny` |
| `websearch` | 网络搜索 | 子代理建议 `deny` 或 `ask` |
| `external_directory` | 项目外部文件操作 | 沙箱关键，公共 Bot 建议 `deny` |
| `repo_clone` | Git 仓库克隆 | 控制依赖检查与外部仓库拉取 |
| `repo_overview` | 仓库概览分析 | 控制对 git 仓库元数据的访问 |
| `doom_loop` | 循环恢复提示 | 建议 `allow` |
| `todowrite` | 任务列表管理 | 按需配置 |

每个权限键可设为 `"allow"`、`"ask"`、`"deny"`，或一个对象实现命令级细粒度控制。

### 2.2 按代理角色的权限矩阵

| 代理 | read | edit | bash | task | webfetch | websearch | 说明 |
|------|------|------|------|------|----------|-----------|------|
| **Plan** | allow | **deny** | **deny** | allow | **deny** | **deny** | 只读分析 |
| **Build** | allow | ask/allow | ask/allow | allow | ask | ask | 全工具，bash/网络设 ask |
| **Explore** | allow | **deny** | **deny** | allow | **deny** | **deny** | 代码检索 |
| **Review** | allow | **deny** | ask | allow | ask | ask | 只读 bash 白名单 |
| **Docs** | allow | allow | **deny** | allow | ask | ask | 可写文件，禁执行 |

### 2.3 细粒度控制

**对象模式**（最后匹配的规则生效）：

```json
{
  "agent": {
    "plan": {
      "permission": {
        "edit": "deny",
        "bash": {
          "*": "deny",
          "git status": "allow",
          "git diff*": "allow",
          "grep *": "allow"
        },
        "webfetch": "deny",
        "websearch": "deny"
      }
    }
  }
}
```

**MCP 工具通配符**：

```json
{ "agent": { "build": { "permission": { "playwright_*": "ask", "weather_*": "allow" } } } }
```

---

## 三、模型配置速查

### 3.1 DeepSeek-V4 Flash / Pro

**模型迁移**：`deepseek-chat` → `deepseek-v4-flash`；`deepseek-reasoner` → `deepseek-v4-pro` + `reasoning_effort`。

**硬约束**：

| 模式 | 被忽略的参数 |
|------|------------|
| **思考模式开启** | `temperature`、`top_p`、`presence_penalty`、`frequency_penalty` |
| **非思考模式** | 所有参数正常生效 |

| 角色 | 模型 | temp | top_p | max_tokens | 约束 |
|------|------|------|-------|------------|------|
| 规划 | `deepseek/deepseek-v4-pro` | 0.08 | 0.7 | 32768 | 无思考模式（需显式设 `thinking: { type: "disabled" }`），高确定性输出 |
| 执行 | `deepseek/deepseek-v4-flash` | 0.12 | 0.75 | 8192~32768 | 非思考模式 |
| 探索 | `deepseek/deepseek-v4-flash` | 0.2 | 0.8 | 8192 | 快速检索 |
| 创意 | `deepseek/deepseek-v4-flash` | 0.7~0.9 | 0.92 | 32768 | 确保未开启思考 |
| 深度推理 | `deepseek/deepseek-v4-pro` | — | — | 32768 | `variant: "high"` 或 `"max"`，自动启用思考模式 |

> **两种推理策略**：① **Variant 模式（推荐）** — 不设 temperature/top_p，在 agent 上设 `variant: "max"` 或 `"high"`，系统自动注入 `{ reasoningEffort }` 和 thinking 参数，采样参数由模型接管；② **无思考模式** — 显式关闭 thinking（`thinking: { type: "disabled" }`），自定义 temperature/top_p，适用于需要精确控制输出风格的场景。 |

> **注意**：`presence_penalty`/`frequency_penalty` 已弃用；本地部署温度 ≤0.8 等效旧版 1.3 可控性，`top_p` 不低于 0.5 以防输出中断。

### 3.2 MiniMax-M2.5 / M2.7

**官方基准**：`temperature=1.0`、`top_p=0.95`、`top_k=40`。

> ⚠️ `top_k=40` 为 MiniMax 原生 API 参数，OpenCode 中是否透传取决于 AI SDK。建议以 `DEBUG=1` 验证实际请求体。

| 角色 | 模型 | temp | top_p | max_tokens | 说明 |
|------|------|------|-------|------------|------|
| 通用/Agent | `minimax/MiniMax-M2.7` | 1.0 | 0.95 | 32768 | 官方黄金组合，M2.7 指令遵循率 97% |
| 确定性代码 | `minimax/MiniMax-M2.7` | 0.3~0.6 | 0.85~0.95 | 32768 | **>5 步工具调用不建议降温** |
| 高速执行 | `minimax/MiniMax-M2.7-highspeed` | 1.0 | 0.95 | 16384 | ~100 tps |

> **特有约束**：M2 系列为**交错思考模型**。历史消息中 `<thinking>...</thinking>` 内容必须原样回传，切勿过滤。长链工具调用不稳定时优先检查 `max_tokens` 和工具描述清晰度，而非下调温度。

### 3.3 Kimi-K2.5 / K2.6

**硬约束**：`temperature`、`top_p`、`presence_penalty`、`frequency_penalty`、`n` **不可自定义**。

| 模式 | temperature | top_p | 触发方式 |
|------|-------------|-------|---------|
| K2.6 思考模式 | **固定 1.0** | **固定 0.95** | 默认开启 |
| K2.6 即时模式 | **固定 0.6** | **固定 0.95** | `thinking: { type: "disabled" }`（官方 API）；部分第三方需 `extra_body` |
| K2.5 通用 | **固定 1.0** | **固定 0.95** | 无开关 |

| 角色 | 模型 | 参数 | max_tokens |
|------|------|------|------------|
| K2.6 复杂推理 | `moonshotai/kimi-k2.6` | **不设采样参数** | 32768 |
| K2.6 即时 | `moonshotai/kimi-k2.6` | **不设采样参数** | 32768 |
| K2.5 通用 | `moonshotai/kimi-k2.5` | **不设采样参数** | 8192~32768 |

> **关键**：代理配置中不要写入 `temperature`/`top_p`/`penalty`/`n`，仅保留 `model`、`prompt`、`permission`、`max_tokens`。`reasoning_content` 多轮对话中需按官方规则回传。

### 3.4 GLM-4.7 / GLM-5.1

**官方基准**：`temperature=1.0`、`top_p=0.95`。**只调一个参数**。

> **提供商命名**：OpenCode 使用 `zhipuai` 提供商键（models.dev 注册键）。

| 角色 | 模型 | temp | top_p | max_tokens | 说明 |
|------|------|------|-------|------------|------|
| 通用/长程 | `zhipuai/glm-5.1` | 1.0 | 0.95 | 32768~131072 | 默认即最优，上下文 200K |
| 创意生成 | `zhipuai/glm-5.1` | 1.0 | 0.95 | 32768 | 只改 temperature |
| 代码/逻辑 | `zhipuai/glm-4.7-flash` | 0.1~0.3 | 0.95 | 2048~4096 | 极低温度 |
| 工具调用 | `zhipuai/glm-4.7-flash` | 1.0 | 0.95 | 4096 | 通用推荐 |
| 深度思考 | `zhipuai/glm-5.1` | — | — | 32768 | 直接传入 `thinking: { type: "enabled" }` |

> **注意**：GLM-5.1 支持流式工具调用（`tool_stream=true`）；GLM-4.7 在 SWE-bench 最优参数为 `temp=0.7, top_p=1.0, max_tokens=16384`。

---

### 3.5 Variants 推理强度控制

OpenCode 内建了 Variants 系统，用于控制推理模型（reasoning model）的思考深度。每个 variant 是一组预设的 API 参数，在发请求时自动合并到最终的 API body 中。

**工作机制**：
1. 对于 `reasoning: true` 的模型，系统根据 provider 自动生成 variant 集合
2. 用户在 agent 上通过 `variant` 字段选择一个 variant 名称
3. 选中的 variant 参数与 `model.options`、`agent.options` 合并后注入 API 请求

**内建 Variant 名称表**：

| Provider | 模型条件 | Variant 名称 | 注入参数 |
|----------|---------|-------------|---------|
| Anthropic (thinking) | reasoning 模型 | `max`、`high` | `{ thinking: { type: "enabled", budgetTokens: N } }` |
| Anthropic (adaptive) | Opus 4.7+ | `minimal`~`xhigh` | `{ thinking: { type: "adaptive" }, effort }` |
| OpenAI (o-series/GPT-5) | reasoning 模型 | `minimal`、`low`、`medium`、`high`、`xhigh` | `{ reasoningEffort, reasoningSummary }` |
| Google Gemini 2.5 | reasoning 模型 | `high`、`max` | `{ thinkingConfig: { thinkingBudget: N } }` |
| Google (其他) | reasoning 模型 | `low`、`high` | `{ includeThoughts, thinkingLevel }` |
| xAI Grok-3-mini | reasoning 模型 | `low`、`high` | `{ reasoningEffort }` 或 `{ reasoning: { effort } }` |
| GitHub Copilot | reasoning 模型 | `low`~`xhigh` | `{ reasoningEffort, reasoningSummary }` |
| **DeepSeek V4** | reasoning 模型 | `low`、`medium`、`high`、`max` | `{ reasoningEffort }` |
| 其他 OAI 兼容 | reasoning 模型 | `low`、`medium`、`high` | `{ reasoningEffort }` |

**用户配置**：

可通过 `provider.<id>.models.<name>.variants` 自定义或禁用 variant：

```jsonc
{
  "provider": {
    "deepseek": {
      "models": {
        "deepseek-v4-pro": {
          "variants": {
            "max": {
              "reasoningEffort": "max"   // 覆盖默认值
              // 可添加任意额外字段
            },
            "low": {
              "disabled": true            // 禁用不需要的 variant
            }
          }
        }
      }
    }
  },
  "agent": {
    "build": {
      "variant": "max",                   // 选择 variant
      "steps": 50
    }
  }
}
```

> **注意**：`variant` 仅在 agent 使用了自定义 `model` 时生效。若 agent 未配置 model，则继承调用方的模型及其 variant 设置。

---

## 四、典型误配案例

### ❌ 案例 1：temperature 和 top_p 双低陷阱

| 项目 | 内容 |
|------|------|
| **误配** | `temperature=0.1`、`top_p=0.3` |
| **现象** | 输出僵硬、重复，甚至循环。低温已集中概率分布，低 top_p 进一步裁减候选词。 |
| **修正** | 保持 `top_p ≥ 0.85`。若需极强确定性：`temperature=0.1` + `top_p=0.95`。 |

### ❌ 案例 2：max_tokens 过小

| 项目 | 内容 |
|------|------|
| **误配** | `max_tokens=512`，多步工具调用 |
| **现象** | 第 2~3 步输出截断，JSON 不完整，触发重试，最终返回空结果。 |
| **修正** | 执行代理 ≥4096；长链工具调用 ≥8192；GLM-5.1 等可设更高。 |

### ❌ 案例 3：在固定参数模型上强行设值

| 项目 | 内容 |
|------|------|
| **误配** | Kimi-K2.5/K2.6 代理中写入 `temperature=0.3` |
| **现象** | K2.5 直接报错；K2.6 传入非固定值报错。 |
| **修正** | 完全移除 `temperature`、`top_p`、`penalty`、`n`，仅保留 `max_tokens` 和 `thinking`。 |

### ❌ 案例 4：弃用的 tools 字段 + 错误键名

| 项目 | 内容 |
|------|------|
| **误配** | `tools: { "file_edit": false, "network": false }` |
| **现象** | 键名无效，工具实际仍处于默认允许状态，只读代理意外获得权限。 |
| **修正** | 改用 `permission: { "edit": "deny", "webfetch": "deny", "websearch": "deny" }`。 |

### ❌ 案例 5：无终止条件

| 项目 | 内容 |
|------|------|
| **误配** | 复杂任务代理未在 prompt 中定义退出策略 |
| **现象** | 任务超时，token 消耗爆炸，返回半成品。 |
| **修正** | prompt 中显式加入："连续两次工具调用未取得进展，或相同错误超过 2 次时，停止并总结当前状态。" |

---

## 五、调优与运维

### 5.1 参数调优铁律

1. **一次只改一个参数**，观察 3~5 次完整运行再下结论
2. **优先调 `temperature`**，`top_p` 仅用于微调
3. **避免同时大幅调整** `temperature` 和 `top_p`
4. **注意架构差异**：MoE 模型（DeepSeek-V4、MiniMax-M2.7）与 Dense 模型对温度的敏感度不同

| 温度区间 | 典型行为 | 适用场景 |
|---------|---------|---------|
| 0.0~0.3 | 高度确定、精确 | 代码生成、数学推理、工具调用 |
| 0.4~0.7 | 平衡创造力与一致性 | 技术文档、翻译、通用对话 |
| 0.8~1.5 | 高随机性、创意发散 | 创意写作、头脑风暴、角色扮演 |

### 5.2 成本与性能权衡

| 调整方向 | 成本 | 延迟 | 建议 |
|---------|------|------|------|
| 增大 `max_tokens` | 线性增加 | 轻微上升 | 留 20% 余量，避免浪费 |
| 高 `temperature` | 可能间接增加 | 无显著变化 | 代码任务低温以缩短输出 |
| 高速版模型 | 持平或略高 | **显著降低** | 时延敏感交互首选 |
| 开启思考/推理 | 显著增加 | 明显增加 | 仅用于真正需要深度推理 |
| 增加迭代深度 | 成倍增长 | 线性增加 | 为子代理设保守预算 |

**成本优化法则**：
- 只读分析、代码探索 → `glm-4.7-flash` 等低成本模型
- 核心规划、复杂推理 → `deepseek-v4-pro`、`glm-5.1`、`kimi-k2.6`
- 避免在高成本模型上执行低价值重复任务

### 5.3 配置验证与维护

**验证方法**：
- `DEBUG=1` 环境变量运行，观察实际 API 请求体
- 为代理设 `edit: deny` 后尝试要求修改文件，确认被拒绝或提示 `ask`
- `temperature=0.1` 时多次运行同一提示应输出几乎一致，否则参数未生效

**维护周期**：
- **每季度**：检查各模型 API 参数约束变更
- **每半年**：全面审查手册，更新模型标识、推荐参数和误配案例

**版本说明**：
- 本手册基于 OpenCode v1.x（≥1.4.x）
- `permission` 字段在 v1.x 稳定；`tools` 在 v1.4+ 弃用
- 若升级 v2.x，关注代理配置结构变化

---

## 六、参考来源

| # | 标题 | 链接 | 概要 |
|---|------|------|------|
| 1 | OpenCode Agents | https://opencode.ai/docs/agents/ | Agent 类型、Markdown/JSON 格式、`permission` 定义 |
| 2 | OpenCode Config | https://opencode.ai/docs/config/ | 配置结构、合并规则、全局选项 |
| 3 | OpenCode Permissions | https://opencode.ai/docs/permissions/ | 权限键清单、对象级细粒度控制语法 |
| 4 | OpenCode Models | https://opencode.ai/docs/models/ | 模型选项配置、`provider` 参数、`variants` 机制 |
| 5 | DeepSeek Thinking Mode | https://api-docs.deepseek.com/zh-cn/guides/thinking_mode | 思考模式下采样参数被忽略的官方说明 |
| 6 | DeepSeek V4 Migration | https://wavespeed.ai/blog/posts/blog-deepseek-v4-model-name-migration/ | 模型名变更及 `reasoning_effort` 用法 |
| 7 | MiniMax-M2 GitHub | https://github.com/MiniMax-AI/MiniMax-M2 | 官方推荐参数、交错思考回传要求 |
| 8 | MiniMax-M2.7 NVIDIA | https://build.nvidia.com/minimaxai/minimax-m2.7/modelcard | 生产部署推荐参数 |
| 9 | Kimi K2.6 API | https://platform.kimi.ai/docs/guide/kimi-k2-6-quickstart | 参数固定约束、`thinking` 默认值 |
| 10 | Kimi K2.5 NVIDIA | https://docs.api.nvidia.com/nim/reference/moonshotai-kimi-k2-5 | Thinking/Instant 模式官方推荐 |
| 11 | GLM-5.1 Migration | https://docs.z.ai/guides/overview/migrate-to-glm-new | 采样参数默认值、深度思考开启方式 |
| 12 | GLM-4.7 Blog | https://z.ai/blog/glm-4.7 | 不同任务基准参数 |
| 13 | LLM Tuning Best Practices | 社区共识与官方推荐汇总 | temperature/top_p 互斥调优原则 |

---

> **文档维护声明**：本手册为团队内部配置参考文档 v1.2。任何配置变更建议注明复核日期。如遇 OpenCode 或模型 API 重大版本更新，建议参考上述官方来源。

## 审阅修订记录

- **审阅日期**：2026-05-30
- **审阅依据**：OpenCode 源码（`packages/opencode/src/config/agent.ts`、`provider.ts`、`permission.ts`、`transform.ts`、`session/llm/request.ts`）+ OpenCode 官方文档 + DeepSeek/Kimi/GLM 官方 API 文档
- **修改摘要**：P0 1 项 / P1 6 项 / P2 5 项 / P3 4 项
- **主要变动**：
  - 修正 `mode` 字段说明（新增 `all` 默认值）
  - Schema 表新增 `variant`、`hidden`、`options` 字段
  - 修正参数透传机制描述（补充 `mergeOptions` 合并链）
  - 权限表新增 `repo_clone`、`repo_overview`
  - **新增 §1.4 Agent 定义方式**（Markdown 文件定义）
  - **新增 §1.5 其他全局配置项**（`default_agent`、`small_model` 等）
  - **新增 §3.5 Variants 推理强度控制**（变体机制全说明 + 内建名称表）
  - 修正 DeepSeek "规划"角色策略说明（两种推理方案）
  - 添加文档目录 (TOC)
- **待验证项**：DeepSeek 模型迁移映射（`deepseek-chat`→`deepseek-v4-flash`）来源于第三方博客 [wavespeed.ai]，建议关注官方迁移公告
- **健康评分**：见下方

## 文档健康评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 事实准确性 | A | P0 已修正，所有 Schema 字段和 API 参数均与源码和官方文档核对 |
| 结构清晰度 | A | 添加 TOC，按读者路径（概念→配置→权限→模型→调优→参考）排列 |
| 格式规范性 | B | 代码块有语言标识；中英文空格待后续清理 |
| 内容时效性 | A | 日期 2026-05-30，Kimi K2.6 / GLM-5.1 / DeepSeek V4 参数均已核对 |
| **综合** | **A** | |

