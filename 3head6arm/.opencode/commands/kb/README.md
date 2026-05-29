# 通用知识库命令系统（kb）

基于 opencode 的通用知识库构建命令系统，支持为任意项目独立构建知识库，互不干扰。

---

## 一、命令总览

| 命令 | 用途 | 何时使用 |
|------|------|---------|
| `/kb/init` | 初始化知识库 | **首次为项目建立 KB** 或 **新对话中恢复项目上下文** |
| `/kb/update` | 添加/更新知识 | 学完新概念、做出决策、有阶段性进展、需要修正已有内容 |
| `/kb/query` | 查询知识库 | 快速提问、检索已记录的知识、查看待解决问题 |
| `/kb/status` | 生成状态报告 | 定期回顾（建议每 3-5 次 update 后）、阶段性收束 |
| `/kb/archive` | 归档过时内容 | 内容已被替代、不再相关、阶段性清理 |

---

## 二、推荐工作流

### （一）开启新项目

```
/kb/init
→ 跟随引导完成项目配置
→ /kb/update 开始记录知识
```

### （二）日常使用

```
/kb/update          # 记录新知识
/kb/query           # 检索已有知识
/kb/update          # 继续记录
/kb/status          # 阶段性回顾
→ 根据报告补充知识或归档过时内容
```

### （三）会话恢复

```
/kb/init            # 新对话第一句话
→ 自动加载项目上下文
→ 继续上次工作
```

---

## 三、目录结构

```
.kb/                          # 知识库根目录（可配置）
├── kb.json                   # 元数据：项目名、创建日期、版本
├── context/                  # 会话恢复上下文（/init 加载区）
│   ├── project.md            # 项目背景（用户撰写）
│   ├── state.md              # 当前状态摘要（/status 自动更新）
│   └── conventions.md        # KB 写作约定（模板、命名规范）
├── docs/                     # 知识文档存储
│   ├── decisions/            # 决策记录（ADR）
│   ├── notes/                # 学习/探索笔记
│   ├── progress/             # 进度记录
│   └── specifications/       # 规格/标准文档
├── index/                    # 可检索索引（结构化数据）
│   ├── concepts.json         # 概念索引
│   ├── questions.json       # 待解决问题追踪
│   └── timeline.json        # 时间线
└── archive/                  # 已归档内容
```

---

## 四、文档类型

| 类型 | 目录 | 职责 | 包含内容 | 不包含内容 |
|------|------|------|---------|-----------|
| **学习笔记** | `docs/notes/` | 记录学到的概念、理论或技术 | 概念定义、理解、与项目关联 | 当前待办 |
| **决策记录** | `docs/decisions/` | 记录"为什么这样做" | 背景、选项对比、决策理由、后果 | 纯技术教程 |
| **进度记录** | `docs/progress/` | 记录当前阶段进展 | 阶段性成果、阻塞项、下步计划 | 概念深度讨论 |
| **规格文档** | `docs/specifications/` | 记录"做成什么样" | 接口定义、数据结构、约束条件 | 为什么这样设计 |

---

## 五、文件命名约定

**命名格式**

```
格式: YYYY-MM-DD-<kebab-case-slug>.md
```

**示例**

```
2025-06-15-error-signal-design.md
2025-06-15-误差信号方案.md
```

**命名规则**

1. 日期为文档创建日期
2. slug 简洁描述主题（英文 3-6 词，中文 4-8 字）
3. 同一天多个文档在 slug 后追加 -2, -3

---

## 六、操作化状态标记

| 标记 | 含义 |
|------|------|
| 🟢 已操作化 | 有明确计算/测量方法 |
| 🟡 待操作化 | 代理指标可用，但有已知误差 |
| 🔴 悬空 | 有名称但缺乏操作化定义 |

---

## 七、设计原理

### （一）五层提问范式（源自原命令集）

本系统继承并简化了原命令集的五层提问范式：

```
第零层：初始化        → /kb/init
第一层：假设检验      → （融入 /kb/update 的学习笔记类型）
第二层：定向知识获取  → /kb/update
第三层：操作化检验    → /kb/update（学以致用引导）
第四层：整合收束      → /kb/status
第五层：反事实压力测试 → （/kb/status 的薄弱环节诊断）
```

### （二）核心设计原则

| 原则 | 说明 |
|------|------|
| **零硬编码** | 所有项目信息从 `kb.json` 和 `context/` 读取，命令不含项目特定内容 |
| **跨平台优先** | 使用 opencode 内置工具，不依赖 shell 命令 |
| **索引驱动** | 核心操作通过 `index/*.json` 完成 |
| **自动持久化** | `/kb/update` 自动保存，无需手动操作 |
| **渐进式加载** | `/kb/init` 先轻量加载，按需深入检索 |
| **人机协作** | 关键操作引导式交互，禁止"记忆参数" |

---

## 八、数据流关系

```
/kb/init
  首次 → 创建 kb.json, context/, index/, docs/
  恢复 → 加载 project.md, state.md, kb.json

/kb/update
  读取 ← kb.json, index/concepts.json
  写入 → docs/, index/concepts.json, questions.json, timeline.json

/kb/query
  读取 ← index/*.json, docs/**

/kb/status
  读取 ← kb.json, docs/, index/*.json, context/
  写入 → context/state.md

/kb/archive
  读取 ← index/*.json, docs/
  写入 → archive/, 更新 index/*.json
```

---

## 九、安装到全局

**（一）复制到全局命令目录**

如果想在任何项目中使用这套命令：

```bash
mkdir -p ~/.config/opencode/commands
cp -r 3head6arm/commands/kb/* ~/.config/opencode/commands/
```

命令名将变为 `/init`, `/update`, `/query`, `/status`, `/archive`。

**（二）保留 kb/ 前缀**

如果要保留 `kb/` 前缀（避免与全局 init 等冲突）：

```bash
mkdir -p ~/.config/opencode/commands/kb
cp 3head6arm/commands/kb/*.md ~/.config/opencode/commands/kb/
```

---

## 十、已知限制

1. `/kb/archive` 执行文件移动时，需要文件系统支持跨目录移动操作
2. 索引文件（JSON）损坏时，系统会尝试部分处理但建议定期备份
3. 中文 slug 在部分文件系统上可能有兼容性问题，建议优先使用英文 slug

---

## 十一、许可证

MIT