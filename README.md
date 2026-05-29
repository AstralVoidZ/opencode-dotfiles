# opencode-dotfiles

> opencode 个人配置集，非代码项目。

---

## 一、项目概述

### （一）仓库定位

本仓库为 [opencode](https://opencode.ai) 个人调优配置集，涵盖模型分工、代理定义、自定义命令、主题配色等。

### （二）目录结构

```
opencode-dotfiles/
├── 3head6arm/
│   ├── .opencode/                       # opencode 运行时配置
│   │   ├── opencode.jsonc               # 主配置
│   │   ├── agents/                      # 自定义代理
│   │   │   ├── build++.md               # DeepSeek 特化 Build Agent
│   │   │   └── plan++.md                # DeepSeek 特化 Plan Agent
│   │   ├── commands/                    # 自定义命令
│   │   │   ├── kb/                      # 通用知识库命令集
│   │   │   ├── review-cfg.md            # 配置文件审查
│   │   │   └── review-doc.md            # 技术文档审阅
│   │   ├── themes/                      # 自定义主题
│   │   │   ├── immortal-cultivation.json
│   │   │   ├── three-body.json
│   │   │   └── wandering-earth.json
│   │   ├── dcp.jsonc                    # 动态上下文压缩配置
│   │   ├── tui.jsonc                    # 终端 UI 配置
│   │   └── vibeguard.config.json        # 密钥扫描规则
│   └── .wakatime.cfg
├── references/                          # 参考手册
│   ├── opencode_agent_config.md         # 代理配置参考手册
│   └── deepseek_agent_guide.md         # DeepSeek-V4-Pro Agent 开发指南
└── LICENSE
```

---

## 二、配置说明

### （一）主配置结构（opencode.jsonc）

| 区块 | 用途 |
|------|------|
| `model` / `small_model` | 默认调用的主模型和小模型 |
| `mcp` | MCP 服务器配置（MiniMax 等） |
| `agent` | 各代理的参数（temperature、top_p、max_tokens、steps） |
| `provider` | 模型提供商的 API 配置和 variants 定义 |
| `plugin` | 插件列表（opencode-dcp、opencode-vibeguard 等） |
| `permission` | 工具权限默认策略 |
| `compaction` | 上下文压缩控制 |

### （二）自定义代理 agents/

| 代理 | 模型 | 用途 |
|------|------|------|
| `build++` | DeepSeek-V4-Pro（variant: thinking） | 深度编程 |
| `plan++` | DeepSeek-V4-Pro（variant: deep-thinking） | 深度规划 |

### （三）自定义命令 commands/

| 命令 | 文件 | 用途 |
|------|------|------|
| `/review-cfg` | `review-cfg.md` | 审查配置文件合法性、语义、修正、调优 |
| `/review-doc` | `review-doc.md` | 审查技术文档结构与可读性 |
| `/kb/init` | `kb/init.md` | 初始化知识库或恢复会话 |
| `/kb/update` | `kb/update.md` | 添加学习笔记/决策/进度/规格 |
| `/kb/query` | `kb/query.md` | 搜索文档/概念/问题 |
| `/kb/status` | `kb/status.md` | 生成知识库状态报告 |
| `/kb/archive` | `kb/archive.md` | 归档过时文档 |

### （四）主题 themes/

| 主题 | 说明 |
|------|------|
| `immortal-cultivation` | 凡人修仙传风格 |
| `three-body` | 三体风格 |
| `wandering-earth` | 流浪地球风格 |

### （五）参考手册 references/

| 文件 | 用途 |
|------|------|
| `opencode_agent_config.md` | 代理配置参考手册，含模型参数约束、权限矩阵、误配案例 |
| `deepseek_agent_guide.md` | DeepSeek-V4-Pro Agent 开发指南，含 Skill-Document 架构、Variant 配置、Prompt 工程范式 |

---

## 三、模型分工

### （一）MiniMax 系列

| 代理 | 模型 | temp | top_p | max_tokens | 定位 |
|------|------|------|-------|------------|------|
| `build` | `minimax-cn-coding-plan/MiniMax-M2.7` | 1.0 | 0.95 | 32768 | 主力执行 |
| `plan` | `minimax-cn-coding-plan/MiniMax-M2.7` | 0.5 | 0.9 | 32768 | 任务规划 |

### （二）DeepSeek 系列

| 代理 | 模型 | variant | max_tokens | 定位 |
|------|------|----------|------------|------|
| `build++` | `deepseek/deepseek-v4-pro` | thinking | 32768 | 深度编程 |
| `plan++` | `deepseek/deepseek-v4-pro` | deep-thinking | 32768 | 深度规划 |

### （三）其他

| 代理 | 模型 | temp | top_p | max_tokens | 定位 |
|------|------|------|-------|------------|------|
| `small_model` | `zhipuai/glm-4.7-flash` | — | — | — | 小模型备用 |

---

## 四、隐私

所有密钥使用 `{env:VAR_NAME}`占位，禁止明文写入。

---

## 五、其他

### （一）免责声明

本作出品，人工智能。
如有雷同，纯属巧合。

### （二）License

[MIT](LICENSE) — 免费、开源、可自由使用，无任何担保。
