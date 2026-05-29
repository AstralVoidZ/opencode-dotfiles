# DeepSeek-V4-Pro Agent 开发指南 v1.0

> **文档定位**：开发 DeepSeek-V4-Pro 特化 Agent 的参考文档  
> **最后更新**：2026-05-30  
> **适用场景**：OpenCode Markdown Agent / deepcode-cli AGENTS.md  
> **核心参考**：deepcode-cli 源码分析 + OpenCode 代理配置手册 v1.2

---

## 目录

- [一、模型基础行为](#一模型基础行为)
  - [1.1 Thinking Mode 机制](#11-thinking-mode-机制)
  - [1.2 API 参数约束](#12-api-参数约束)
  - [1.3 reasoning_content 回传](#13-reasoning_content-回传)
- [二、Variant 配置体系](#二variant-配置体系)
  - [2.1 内建 Variant 速查](#21-内建-variant-速查)
  - [2.2 自定义 Variant](#22-自定义-variant)
  - [2.3 Variant 合并优先级](#23-variant-合并优先级)
- [三、Prompt 工程范式](#三prompt-工程范式)
  - [3.1 Skill-Document 架构](#31-skill-document-架构)
  - [3.2 核心 Skill 文档](#32-核心-skill-文档)
  - [3.3 Prompt Stack 顺序](#33-prompt-stack-顺序)
  - [3.4 输出格式约束](#34-输出格式约束)
- [四、Tool Schema 设计](#四tool-schema-设计)
  - [4.1 权限 scoping（sideEffects）](#41-权限-scopingsideeffects)
  - [4.2 参数描述规范](#42-参数描述规范)
- [五、Agent 设计模式](#五agent-设计模式)
  - [5.1 Build Agent 模式](#51-build-agent-模式)
  - [5.2 Plan Agent 模式](#52-plan-agent-模式)
  - [5.3 Drift Guard 机制](#53-drift-guard-机制)
- [六、典型误配案例](#六典型误配案例)
- [七、OpenCode vs deepcode-cli 对照](#七opencode-vs-deepcode-cli-对照)

---

## 一、模型基础行为

### 1.1 Thinking Mode 机制

DeepSeek-V4-Pro 内置推理模型，思考模式通过以下 API 参数控制：

```typescript
// 启用思考模式（推荐用于复杂推理）
{
  thinking: { type: "enabled" },
  extra_body: { reasoning_effort: "high" | "max" }
}

// 禁用思考模式（用于精确控制输出风格）
{
  thinking: { type: "disabled" }
}
```

**variants 对应关系**（OpenCode 内建）：

| Variant 名称 | 注入参数 | 适用场景 |
|-------------|---------|---------|
| `low` | `reasoningEffort: "low"` | 简单任务，快速响应 |
| `medium` | `reasoningEffort: "medium"` | 中等复杂度 |
| `high` | `reasoningEffort: "high"` | 复杂推理（默认） |
| `max` | `reasoningEffort: "max"` | 深度分析 |

> **参考**：deepcode-cli 在启用思考模式时默认 `reasoningEffort` 为 `"max"`（`buildThinkingRequestOptions` 函数签名默认值）

### 1.2 API 参数约束

**思考模式开启时，被模型忽略的参数**：

| 参数 | 状态 |
|------|------|
| `temperature` | **模型忽略** |
| `top_p` | **模型忽略** |
| `presence_penalty` | **模型忽略** |
| `frequency_penalty` | **模型忽略** |

> **参考**：DeepSeek 思考模式下采样参数由模型接管，外传值不生效。

**非思考模式**：所有参数正常生效。

| 角色 | 推荐 temp | 推荐 top_p | max_tokens | 说明 |
|------|----------|-----------|------------|------|
| 高确定性规划 | 0.08 | 0.7 | 32768 | 关闭思考，精确控制 |
| 深度推理 | — | — | 32768 | variant 模式，模型接管 |
| 快速执行 | 0.12 | 0.75 | 8192~32768 | 非思考模式 |

### 1.3 reasoning_content 回传

DeepSeek-V4-Pro 在思考模式中会返回 `reasoning_content` 字段，包含模型的内部推理过程。

**OpenCode 中的处理**（`opencode/session/llm/request.ts`）：
- 推理内容会被附加到 assistant message 的 `reasoning_content` 属性
- 多轮对话中需按模型规则回传，否则影响后续推理质量

> **注意**：MiniMax 的交错思考模型需原样回传 `<thinking>` 标签，DeepSeek 无此要求。

---

## 二、Variant 配置体系

### 2.1 内建 Variant 速查

OpenCode 为 DeepSeek V4 自动生成的 variant 集合（`opencode/provider.ts`）：

```json
{
  "deepseek": {
    "models": {
      "deepseek-v4-pro": {
        "variants": {
          "low": { "reasoningEffort": "low" },
          "medium": { "reasoningEffort": "medium" },
          "high": { "reasoningEffort": "high" },
          "max": { "reasoningEffort": "max" }
        }
      },
      "deepseek-v4-flash": {
        "variants": {
          "low": { "reasoningEffort": "low" },
          "medium": { "reasoningEffort": "medium" },
          "high": { "reasoningEffort": "high" },
          "max": { "reasoningEffort": "max" }
        }
      }
    }
  }
}
```

### 2.2 自定义 Variant

在 `opencode.jsonc` 的 `provider` 中自定义或覆盖：

```jsonc
{
  "provider": {
    "deepseek": {
      "options": {
        "apiKey": "{env:DEEPSEEK_API_KEY}",
        "setCacheKey": true
      },
      "models": {
        "deepseek-v4-pro": {
          "variants": {
            "ultra": {
              "reasoningEffort": "max",
              "thinking": { "type": "enabled" }
            },
            "quick": {
              "reasoningEffort": "low",
              "thinking": { "type": "enabled" }
            },
            "disabled": {
              "disabled": true
            }
          }
        }
      }
    }
  }
}
```

### 2.3 Variant 合并优先级

**参考**（`opencode/session/llm/request.ts`）：

```
mergeOptions(base, model.options, agent.options, variant)
```

| 优先级 | 来源 | 说明 |
|-------|------|------|
| 1 (最低) | `ProviderTransform.options()` | provider 全局选项 |
| 2 | `model.options` | 模型级配置 |
| 3 | `agent.options` | 代理级别配置 |
| 4 (最高) | `variant` | 推理强度变体 |

**关键约束**：
- variant 只在 agent **自己配置了 model** 时生效
- 继承主代理 model 时，variant 设置无效
- 思考模式下 variant 的 `reasoningEffort` 会覆盖 agent.options 中的任何同名参数

---

## 三、Prompt 工程范式

### 3.1 Skill-Document 架构

deepcode-cli 实践验证的架构，prompt stack 按顺序叠加：

```
1. SYSTEM_PROMPT_BASE     ← 简短身份定义（1-2 句）
2. Tool Documentation      ← 详尽的 JSON Schema（含 sideEffects）
3. Skill Documents         ← XML 标签包裹的行为约束
4. Runtime Context         ← workspace info
5. AGENTS.md              ← 追加的系统指令，最后注入
```

**核心原则**：AGENTS.md 是**追加增强**而非**自我包含**。不要在 AGENTS.md 里重复 tool description，让 skill documents 承载行为约束。

### 3.2 核心 Skill 文档

| Skill | 文件 | 用途 | 核心机制 |
|-------|------|------|---------|
| **Karpathy Guidelines** | `karpathy-guidelines.md` | 减少常见 LLM 编程错误 | Simplicity first + Surgical changes + Goal-driven |
| **Agent Drift Guard** | `agent-drift-guard.md` | 检测和纠正执行漂移 | 三级 severity（L1 自我修正→L2 暂停确认→L3 硬边界） |
| **Plan-and-Execute** | `plan-and-execute.md` | 结构化任务规划 | `UpdatePlan` 工具 + `[ ]/[>]/[x]/[!]` 状态机 |

#### karpathy-guidelines.md 要点

```markdown
## 1. Think Before Coding
- Don't assume. Don't hide confusion. Surface tradeoffs.
- 如果不确定，停下来问

## 2. Simplicity First
- Minimum code that solves the problem. Nothing speculative.
- 如果能 50 行解决不要写 200 行

## 3. Surgical Changes
- Touch only what you must. Clean up only your own mess.
- 不改无关代码，不做未请求的 refactor

## 4. Goal-Driven Execution
- 每个任务有 verify 步骤
- 多步任务列出：1. [Step] → verify: [check]
```

#### agent-drift-guard.md 要点

```markdown
## Drift Signals（漂移信号）
- 探索广泛后才打开最相关的文件
- 做用户未请求的额外工作（脚本、文档、cleanup）
- 任务范围扩大后继续执行原计划

## Severity Levels
- L1（轻微）：自动修正，缩小到最小下一步
- L2（中等）：暂停确认
- L3（硬边界/风险）：请求许可

## Self-Check Loop
- 执行后问："这个步骤是否直接推进请求结果？"
```

#### plan-and-execute.md 要点

```markdown
## Task State Symbols
- `[ ]` - 待处理
- `[>]` - 进行中
- `[x]` - 已完成
- `[!]` - 被阻塞

## 执行规则
- 调用 UpdatePlan 时传递完整任务列表，不是 partial diff
- 同时只有一个任务为 `[>]`
- 遇到错误保持 `[>]` 创建新任务解决阻塞
```

### 3.3 Prompt Stack 顺序

在 OpenCode Markdown Agent 中实现 skill-document 架构：

```yaml
---
description: DeepSeek-V4-Pro 特化构建代理
model: deepseek/deepseek-v4-pro
variant: thinking
steps: 40
---

你是 DeepCode，一位由 DeepSeek-V4-Pro 驱动的专家级软件工程代理。

<karpathy-guidelines>
[嵌入 karpathy-guidelines.md 内容]
</karpathy-guidelines>

<agent-drift-guard>
[嵌入 agent-drift-guard.md 内容]
</agent-drift-guard>

<plan-and-execute>
[嵌入 plan-and-execute.md 内容]
</plan-and-execute>

## 输出格式

简单任务：直接输出 + 验证结果
复杂任务：用 ```text thinking 分析，用 ```text output 结构化
任务列表：使用 [ ]/[>]/[x]/[!] 状态标记

## 核心约束
1. Simplicity First：不写 speculative code
2. Surgical Changes：每行变更追溯到用户请求
3. Goal-Driven：每任务有 verify 步骤
4. Drift Response：根据 severity 级别响应
```

### 3.4 输出格式约束

| 任务类型 | 格式 | 说明 |
|---------|------|------|
| 简单任务 | 直接输出 | 无需结构化 |
| 复杂架构分析 | ` ```thinking ` 双块 | 分析过程单独区块 |
| 实现方案 | ` ```output ` 结构化 | summary + artifacts + verification |
| 任务追踪 | UpdatePlan 工具 | `[ ]/[>]/[x]/[!]` 状态机 |

**deepcode-cli 双块结构 vs OpenCode build++ 双块结构**：

| 工具 | Thinking 区块 | Output 区块 |
|------|-------------|------------|
| deepcode-cli | `<analysis>` | `<summary>` |
| OpenCode build++ | ` ```text thinking ` | ` ```text output ` |

---

## 四、Tool Schema 设计

### 4.1 权限 scoping（sideEffects）

deepcode-cli 的 `bash.sideEffects` 设计原则：

| SideEffect | 含义 | 使用场景 |
|-----------|------|---------|
| `read-in-cwd` | 读取当前工作目录内文件 | `cat`, `head`, `rg` |
| `read-out-cwd` | 读取当前工作目录外文件 | `cat /etc/hosts` |
| `write-in-cwd` | 写入当前工作目录内文件 | `echo >`, `tee` |
| `write-out-cwd` | 写入当前工作目录外文件 | 系统文件写入 |
| `delete-in-cwd` | 删除当前工作目录内文件 | `rm` |
| `delete-out-cwd` | 删除当前工作目录外文件 | 危险操作 |
| `query-git-log` | 查询 git 历史 | `git log`, `git show`, `git blame` |
| `mutate-git-log` | 修改 git 历史 | `git commit`, `git reset`, `git rebase` |
| `network` | 网络访问 | `curl`, `wget` |
| `unknown` | 无法分类 | 兜底 |

**OpenCode 中的等价实现**：

```json
{
  "permission": {
    "bash": {
      "*": "ask",
      "git status*": "allow",
      "git log*": "allow",
      "git diff*": "allow",
      "npm *": "allow",
      "pnpm *": "allow",
      "cargo *": "allow",
      "python -m pytest*": "allow"
    }
  }
}
```

### 4.2 参数描述规范

deepcode-cli 工具 schema 的设计规范：

1. **每个参数必须有精确语义描述**：
   ```typescript
   {
     "description": "Clear, concise description of what this command does in active voice."
   }
   ```

2. **description 中不使用"complex"或"risk"等主观词汇**：
   ```typescript
   // Good
   "description": "Find and delete all .tmp files recursively"
   // Bad
   "description": "A complex and risky operation to delete temp files"
   ```

3. **bash 命令需声明 sideEffects**：
   ```typescript
   {
     "sideEffects": {
       "type": "array",
       "description": "Permission scopes required by this bash command.",
       "items": {
         "enum": ["read-in-cwd", "read-out-cwd", "write-in-cwd", ...]
       }
     }
   }
   ```

4. **复杂命令添加上下文说明**：
   ```typescript
   "description": "Fetch JSON from URL and extract data array elements"
   ```

---

## 五、Agent 设计模式

### 5.1 Build Agent 模式

**用途**：执行已批准的方案、编写代码、运行测试、验证结果

> **Note**：variant `thinking` 为自定义名称，需在 provider 中已定义；OpenCode 内建 variant 为 `low`/`medium`/`high`/`max`。

**frontmatter 配置**：
```yaml
---
description: 完整构建执行代理
mode: primary
model: deepseek/deepseek-v4-pro
variant: thinking
steps: 40
permission:
  read: allow
  edit: allow
  bash:
    "*": "ask"
    "npm *": "allow"
    "pnpm *": "allow"
    "cargo *": "allow"
    "python -m pytest*": "allow"
    "git status*": "allow"
    "git diff*": "allow"
---
```

**prompt 结构**：
1. 身份定义（一句话）
2. Karpathy Guidelines（约束）
3. Agent Drift Guard（约束）
4. Plan-and-Execute（任务管理）
5. 输出格式说明
6. 验证协议

### 5.2 Plan Agent 模式

**用途**：复杂重构分析、功能设计、依赖分析，执行前必须制定方案

> **Note**：variant `deep-thinking` 为自定义名称，需在 provider 中已定义；OpenCode 内建 variant 为 `low`/`medium`/`high`/`max`。

**frontmatter 配置**：
```yaml
---
description: 深度规划与架构分析代理
mode: primary
model: deepseek/deepseek-v4-pro
variant: deep-thinking
steps: 25
permission:
  read: allow
  edit: deny
  bash:
    "*": "ask"
    "git status*": "allow"
    "git log*": "allow"
    "git diff*": "allow"
---
```

**关键约束**：
- `edit: deny` — 禁止修改任何文件
- 方案输出必须包含完整文件内容
- 使用思考区块进行深度推理
- 交接时明确发出信号

### 5.3 Drift Guard 机制

**Self-Check Loop（执行后自检）**：

```
Before first meaningful action:
  - Requested outcome
  - Allowed scope
  - Forbidden scope
  - Smallest useful next step

After each non-trivial step:
  - Did this step directly help deliver the requested outcome?
  - Did I learn something that changes scope, or only implementation?
  - Am I about to do more than the user asked?

After a user correction:
  - Remove the old broader plan
  - Do not defend the discarded work
  - Continue from the narrowed scope
```

**Decision Rules（决策规则）**：
1. 优先最直接的 artifact（打开相关文件而非扫描整个仓库）
2. 优先最小化完整修复（解决被问的问题而非改进相关系统）
3. 优先内部修正而非用户中断（能缩小范围时缩小，需要时再问）
4. 将重复的用户约束视为优先级信号（重复指令意味着当前执行风格不对齐）
5. 分离工作类别（代码变更、调查、生产修复、清理、文档是不同的任务）

---

## 六、典型误配案例

### ❌ 案例 1：思考模式下设置 temperature/top_p

| 项目 | 内容 |
|------|------|
| **误配** | `temperature: 0.5, top_p: 0.8` + `variant: "thinking"` |
| **现象** | 参数被模型忽略，输出风格不受控制 |
| **修正** | 移除 temperature/top_p，让 variant 的 reasoningEffort 接管 |

### ❌ 案例 2：variant 无效（继承模型场景）

| 项目 | 内容 |
|------|------|
| **误配** | agent 未配置 model，直接使用 `variant: "max"` |
| **现象** | variant 不生效，模型使用默认推理强度 |
| **修正** | agent 必须显式配置 model，variant 才生效 |

### ❌ 案例 3：Build Agent 无终止条件

| 项目 | 内容 |
|------|------|
| **误配** | 复杂任务未定义退出策略 |
| **现象** | 任务超时，token 消耗爆炸 |
| **修正** | prompt 中加入："连续两次工具调用未取得进展时，停止并总结当前状态" |

### ❌ 案例 4：Plan Agent 使用 Build 模式权限

| 项目 | 内容 |
|------|------|
| **误配** | Plan Agent 配置 `edit: allow` |
| **现象** | 规划代理可修改文件，违反只读架构师定位 |
| **修正** | Plan Agent 必须 `edit: deny` |

### ❌ 案例 5：Skill 文档堆砌过多约束

| 项目 | 内容 |
|------|------|
| **误配** | 将所有规则直接写在 prompt 中，不使用 skill 分离 |
| **现象** | Agent 难以内化，输出质量不稳定 |
| **修正** | 使用 skill-document 架构，XML 标签包裹，让模型按需读取 |

---

## 七、OpenCode vs deepcode-cli 对照

| 维度 | OpenCode | deepcode-cli |
|------|----------|--------------|
| **Agent 定义** | Markdown (`agents/*.md`) 或 JSON | Markdown (`AGENTS.md`) |
| **Skill 加载** | `<skill-name>-skill` XML 标签 | `<skill-name>-skill path="...">` XML 标签 |
| **任务状态** | plan_exit 工具 | UpdatePlan 工具 |
| **输出格式** | ` ```thinking ` / ` ```output ` | `<analysis>` / `<summary>` |
| **Variant 注入** | OpenCode 自动合并 | 手动构建 thinkingRequestOptions |
| **权限控制** | permission 对象 | sideEffects 数组 |
| **AGENTS.md 位置** | `.opencode/agents/` 扫描 | `./.deepcode/AGENTS.md` → `./AGENTS.md` → `~/.deepcode/AGENTS.md` |

---

## 参考来源

| # | 标题 | 来源 | 概要 |
|---|------|------|------|
| 1 | OpenCode 代理配置手册 v1.2 | `references/opencode_agent_config.md` | 配置参数参考 |
| 2 | deepcode-cli AGENTS.md 加载逻辑 | `deepcode-cli/src/session.ts` | 优先级：`./.deepcode/AGENTS.md` → `./AGENTS.md` → `~/.deepcode/AGENTS.md` |
| 3 | deepcode-cli Thinking API | `deepcode-cli/src/common/openai-thinking.ts` | `buildThinkingRequestOptions()` 实现 |
| 4 | deepcode-cli karpathy-guidelines | `deepcode-cli/templates/skills/karpathy-guidelines.md` | Simplicity first + Surgical changes + Goal-driven |
| 5 | deepcode-cli agent-drift-guard | `deepcode-cli/templates/skills/agent-drift-guard.md` | L1/L2/L3 三级 severity 漂移检测 |
| 6 | deepcode-cli plan-and-execute | `deepcode-cli/templates/skills/plan-and-execute.md` | `[ ]/[>]/[x]/[!]` 状态机 |
| 7 | OpenCode Variant 合并 | `opencode/packages/opencode/src/session/llm/request.ts` | `mergeOptions(base, model.options, agent.options, variant)` |
| 8 | OpenCode subagent-permissions | `opencode/packages/opencode/src/agent/subagent-permissions.ts` | 权限派生规则 |
| 9 | OpenCode plan-mode prompt | `opencode/packages/opencode/src/session/prompt/plan-mode.txt` | 五阶段工作流 |

---

> **Note**：本指南为 DeepSeek-V4-Pro Agent 开发参考，任何配置变更建议结合实际效果迭代验证。思考模式参数以 DeepSeek 官方最新文档为准。

---

## 审阅修订记录

- **审阅日期**：2026-05-30
- **修改摘要**：P0 0 项 / P1 2 项 / P2 4 项 / P3 3 项
- **主要变动**：
  - P1：5.1/5.2 节 Build/Plan Agent frontmatter 中添加 `variant: thinking`/`deep-thinking` 为自定义名称的 Note 说明
  - P2：3.3 节代码块标识 `markdown` → `yaml`；3.4 节/7 节表格代码片段补充语言标识；参考来源路径添加 `/src/` 前缀并移除行号
  - P3：移除"3677 节点"具体统计；"最优架构" → "实践验证的架构"；文档维护声明格式统一为 `> **Note**`
- **待验证项**：variant `thinking`/`deep-thinking` 是否已在 3head6arm 的 provider 配置中定义
- **健康评分**：B+

---

## 修改摘要表

| # | 位置 | before | after | 级别 |
|---|------|--------|-------|------|
| 1 | L6 | "（3677 节点）" | 移除 | P3 |
| 2 | L65 | "deepcode-cli 默认使用 `"max"`" | "启用思考模式时默认 `"max`" | P2 |
| 3 | L191 | "验证过的最优架构" | "实践验证的架构" | P3 |
| 4 | L267 | ` ```markdown ` | ` ```yaml ` | P2 |
| 5 | L292 | ` ```thinking ` / ` ```output ` | ` ```text thinking ` / ` ```text output ` | P2 |
| 6 | L316 | 无语言标识 | ` ```text output ` | P2 |
| 7 | L401 | Build Agent frontmatter 后无说明 | 添加 variant 自定义名称 Note | P1 |
| 8 | L435 | Plan Agent frontmatter 后无说明 | 添加 variant 自定义名称 Note | P1 |
| 9 | L571 | `> **文档维护声明**` | `> **Note**` | P3 |
| 10 | 参考来源表 | 路径不一致 | 统一为 `模块/src/文件名` | P2 |