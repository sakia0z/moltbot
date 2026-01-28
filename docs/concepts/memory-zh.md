---
summary: "Moltbot 记忆系统详解（中文）"
read_when:
  - 想要了解 Moltbot 的记忆系统是如何工作的
  - 需要配置和使用记忆功能
  - 想要理解记忆搜索和索引机制
---

# Moltbot 记忆系统详解

## 概述

Moltbot 的记忆系统是一个基于 **Markdown 文件** 的持久化存储方案，而不是依赖于 RAM（内存）。所有的记忆都以普通的 Markdown 文件形式存储在 agent 工作空间中（默认为 `~/clawd`），这些文件是真实的数据源。

**核心理念**：模型只"记住"写入磁盘的内容。如果没有写入文件，信息就会随着会话结束而丢失。

## 一、架构与存储方式

### 1.1 记忆文件布局

默认的工作空间使用两层记忆结构：

```
~/clawd/
├── MEMORY.md                    # 长期记忆（精心整理的重要信息）
└── memory/
    ├── 2026-01-27.md           # 每日日志（按日期追加）
    ├── 2026-01-28.md
    └── ...
```

**文件说明**：

- **`memory/YYYY-MM-DD.md`**（每日日志）
  - 追加式写入，记录当天的对话和事件
  - 会话开始时会读取今天和昨天的日志
  - 适合记录日常笔记和运行时上下文

- **`MEMORY.md`**（长期记忆）
  - 精心整理的长期记忆文件
  - 存储决策、偏好和持久性事实
  - **仅在主私有会话中加载**（不在群组上下文中加载）
  - 适合存储重要的、需要长期保留的信息

### 1.2 记忆搜索索引

除了 Markdown 文件，Moltbot 还维护一个 SQLite 数据库来支持高效的语义搜索：

```
~/.clawdbot/memory/<agentId>.sqlite
```

**索引内容**：
- 文本块（chunks）及其元数据
- 向量嵌入（embeddings）
- 全文搜索索引（FTS5）
- 嵌入缓存

**重要**：索引是从 Markdown 文件 **派生** 而来的，可以随时重建。Markdown 文件始终是真实数据源。

## 二、核心组件

### 2.1 源代码结构

记忆系统的核心代码位于 `/src/memory/` 目录：

| 文件 | 功能 |
|------|------|
| `manager.ts` | 主要的记忆搜索管理器（协调索引、搜索、缓存） |
| `memory-schema.ts` | SQLite 数据库模式定义 |
| `embeddings.ts` | 嵌入提供商接口 |
| `embeddings-openai.ts` | OpenAI 嵌入实现 |
| `embeddings-gemini.ts` | Gemini 嵌入实现 |
| `hybrid.ts` | 混合搜索（BM25 + 向量） |
| `manager-search.ts` | 向量和关键词搜索实现 |
| `batch-openai.ts` | OpenAI 批量嵌入 API |
| `batch-gemini.ts` | Gemini 批量嵌入 API |
| `sync-memory-files.ts` | 文件监视和同步逻辑 |
| `sqlite-vec.ts` | SQLite 向量加速扩展 |

### 2.2 Agent 工具

记忆系统为 agent 提供了两个主要工具（位于 `/src/agents/tools/memory-tool.ts`）：

- **`memory_search`**：语义搜索记忆文件
  - 返回相关片段及其文件路径和行号
  - 支持混合搜索（向量 + 关键词）
  - 自动处理嵌入和索引

- **`memory_get`**：读取特定记忆文件
  - 根据路径读取完整文件内容
  - 支持行范围过滤
  - 仅限访问 `MEMORY.md` 和 `memory/` 目录

### 2.3 自动记忆刷新

在会话接近上下文压缩（compaction）时，Moltbot 会触发一个 **静默的 agent 转换**，提醒模型在上下文被压缩前写入持久化记忆。

**位置**：`/src/auto-reply/reply/memory-flush.ts`

**特点**：
- 默认情况下对用户不可见（返回 `NO_REPLY`）
- 每个压缩周期只运行一次
- 如果工作空间是只读的，则跳过

## 三、工作原理详解

### 3.1 索引流程

1. **文件监视**：监视 `MEMORY.md` 和 `memory/*.md` 文件的变化
2. **文本分块**：将 Markdown 文件切分为约 400 token 的块，重叠 80 token
3. **生成嵌入**：为每个文本块生成向量嵌入
4. **存储索引**：将块、嵌入和元数据存入 SQLite 数据库
5. **全文索引**：同时建立 FTS5 全文搜索索引

```
Markdown 文件
    ↓
文本分块 (~400 tokens, 80 token 重叠)
    ↓
生成嵌入向量
    ↓
存入 SQLite (chunks + embeddings + FTS5)
```

### 3.2 搜索流程

当执行 `memory_search` 查询时：

1. **向量搜索**：使用查询文本的嵌入向量查找最相似的块
2. **关键词搜索**（如果启用混合搜索）：使用 BM25 算法进行全文搜索
3. **结果合并**：根据配置的权重合并两种搜索结果
4. **返回结果**：返回排名靠前的片段，包含：
   - 片段文本（约 700 字符）
   - 文件路径
   - 行号范围
   - 相似度分数
   - 使用的提供商和模型

### 3.3 混合搜索算法

混合搜索结合了两种检索策略：

**向量搜索**（语义匹配）：
- 擅长理解"意思相同"的查询
- 可以处理释义和同义词
- 例如："Mac Studio 网关主机" vs "运行网关的机器"

**BM25 关键词搜索**（精确匹配）：
- 擅长查找精确的标记（token）
- 对 ID、代码符号、错误字符串特别有效
- 例如：commit hash `a828e60`、配置键 `memorySearch.query.hybrid`

**合并算法**：

1. 从两种搜索中各检索 `maxResults * candidateMultiplier` 个候选结果
2. 将 BM25 排名转换为 0-1 分数：`textScore = 1 / (1 + max(0, bm25Rank))`
3. 按照配置的权重合并：`finalScore = vectorWeight * vectorScore + textWeight * textScore`
4. 返回综合得分最高的结果

**默认配置**：
```json5
{
  vectorWeight: 0.7,    // 向量搜索权重 70%
  textWeight: 0.3,      // 关键词搜索权重 30%
  candidateMultiplier: 4
}
```

## 四、嵌入模型提供商

Moltbot 支持多种嵌入提供商：

### 4.1 OpenAI（推荐）

**优点**：
- 速度快，质量高
- 支持批量 API（成本低 50%）
- 大规模索引时最快

**配置示例**：
```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          batch: {
            enabled: true,
            concurrency: 2
          }
        }
      }
    }
  }
}
```

**批量 API 优势**：
- 对于大型回填（backfill），OpenAI 通常是最快的选项
- 批量 API 价格优惠（比同步请求便宜）
- 可以一次提交大量嵌入请求，由 OpenAI 异步处理

### 4.2 Gemini

**配置示例**：
```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "gemini",
        model: "gemini-embedding-001",
        remote: {
          apiKey: "YOUR_GEMINI_API_KEY"
        }
      }
    }
  }
}
```

### 4.3 本地模型

**优点**：
- 完全离线运行
- 无 API 成本
- 数据隐私

**配置示例**：
```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
        local: {
          modelPath: "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf"
        }
      }
    }
  }
}
```

**注意事项**：
- 需要运行 `pnpm approve-builds` 并选择 `node-llama-cpp`
- 首次使用时会自动下载模型（约 0.6 GB）
- 需要更多本地计算资源

### 4.4 自动回退机制

Moltbot 支持多级回退：

```
provider: "auto" → 尝试以下顺序：
  1. 本地模型（如果已配置且存在）
  2. OpenAI（如果有 API 密钥）
  3. Gemini（如果有 API 密钥）
  4. 禁用（如果都不可用）
```

## 五、自动记忆刷新（Memory Flush）

### 5.1 什么是记忆刷新？

当会话接近自动压缩时，Moltbot 会触发一个静默的 agent 转换，提醒模型在上下文被压缩前保存重要信息。

### 5.2 工作机制

1. **触发条件**：当会话 token 数接近 `contextWindow - reserveTokensFloor - softThresholdTokens`
2. **提示模型**：发送系统提示和用户提示，提醒保存记忆
3. **静默执行**：默认期望返回 `NO_REPLY`，用户看不到这个过程
4. **写入记忆**：模型可以写入 `memory/YYYY-MM-DD.md` 或更新 `MEMORY.md`
5. **继续会话**：完成后继续正常对话

### 5.3 配置选项

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

**参数说明**：
- `enabled`：是否启用自动记忆刷新
- `softThresholdTokens`：提前多少 token 触发刷新
- `systemPrompt`：附加到系统提示的内容
- `prompt`：发送给 agent 的用户提示

## 六、配置详解

### 6.1 完整配置示例

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        // 基本设置
        enabled: true,                              // 启用记忆搜索
        provider: "openai",                         // 嵌入提供商
        model: "text-embedding-3-small",           // 嵌入模型
        fallback: "openai",                         // 回退提供商
        
        // 远程提供商配置
        remote: {
          apiKey: "YOUR_API_KEY",                   // API 密钥
          baseUrl: "https://api.openai.com/v1",     // API 端点
          headers: {},                               // 自定义请求头
          batch: {
            enabled: true,                           // 启用批量 API
            concurrency: 2,                          // 并发批量作业数
            wait: true,                              // 等待批量完成
            pollIntervalMs: 5000,                   // 轮询间隔
            timeoutMinutes: 30                      // 超时时间
          }
        },
        
        // 本地模型配置
        local: {
          modelPath: "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
          modelCacheDir: "~/.cache/moltbot/models"
        },
        
        // 查询配置
        query: {
          maxResults: 10,                            // 最大返回结果数
          hybrid: {
            enabled: true,                           // 启用混合搜索
            vectorWeight: 0.7,                       // 向量搜索权重
            textWeight: 0.3,                         // 关键词搜索权重
            candidateMultiplier: 4                   // 候选结果倍数
          }
        },
        
        // 缓存配置
        cache: {
          enabled: true,                             // 启用嵌入缓存
          maxEntries: 50000                         // 最大缓存条目数
        },
        
        // 同步配置
        sync: {
          watch: true,                               // 监视文件变化
          debounceMs: 1500,                         // 防抖延迟
          sessions: {
            deltaBytes: 100000,                     // 会话增量字节阈值
            deltaMessages: 50                       // 会话增量消息阈值
          }
        },
        
        // 存储配置
        store: {
          path: "~/.clawdbot/memory/{agentId}.sqlite",
          vector: {
            enabled: true,                           // 启用向量加速
            extensionPath: "/path/to/sqlite-vec"   // 扩展路径（可选）
          }
        },
        
        // 实验性功能
        experimental: {
          sessionMemory: false                      // 索引会话记录
        },
        
        // 内存源
        sources: ["memory"]                         // 或 ["memory", "sessions"]
      }
    }
  }
}
```

### 6.2 自定义 OpenAI 兼容端点

可以使用任何 OpenAI 兼容的 API 端点（如 OpenRouter、vLLM）：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_CUSTOM_API_KEY",
          headers: {
            "X-Organization": "org-id",
            "X-Project": "project-id"
          }
        }
      }
    }
  }
}
```

## 七、CLI 工具使用

### 7.1 查看索引状态

```bash
# 查看所有 agent 的记忆状态
moltbot memory status

# 查看特定 agent 的状态
moltbot memory status --agent main

# 深度检查（探测向量和嵌入可用性）
moltbot memory status --deep

# 深度检查并在索引过期时重建
moltbot memory status --deep --index

# 详细输出
moltbot memory status --deep --index --verbose
```

### 7.2 手动索引

```bash
# 为所有 agent 重建索引
moltbot memory index

# 为特定 agent 重建索引
moltbot memory index --agent main

# 详细输出（显示每个阶段的详情）
moltbot memory index --verbose
```

### 7.3 搜索记忆

```bash
# 语义搜索
moltbot memory search "release checklist"

# 搜索特定 agent 的记忆
moltbot memory search "release checklist" --agent main
```

## 八、高级功能

### 8.1 SQLite 向量加速（sqlite-vec）

当 sqlite-vec 扩展可用时，Moltbot 会在数据库内执行向量距离查询，这比在 JavaScript 中加载所有嵌入要快得多。

**配置**：
```json5
{
  store: {
    vector: {
      enabled: true,                              // 默认启用
      extensionPath: "/path/to/sqlite-vec"       // 可选：自定义扩展路径
    }
  }
}
```

**注意**：
- 如果扩展缺失或加载失败，会自动回退到 JavaScript 实现
- 没有功能损失，只是性能差异

### 8.2 嵌入缓存

为了避免重复嵌入相同的文本（特别是在频繁更新的会话记录中），Moltbot 会缓存文本块的嵌入向量。

**工作原理**：
1. 计算文本块的哈希值
2. 检查缓存表是否已有嵌入
3. 如果有，直接使用；如果没有，生成新嵌入并缓存
4. 当缓存条目超过 `maxEntries` 时，使用 LRU 策略清理

**配置**：
```json5
{
  cache: {
    enabled: true,
    maxEntries: 50000
  }
}
```

### 8.3 会话记忆搜索（实验性）

可以选择性地索引会话记录，使 `memory_search` 也能搜索历史对话。

**配置**：
```json5
{
  memorySearch: {
    experimental: {
      sessionMemory: true
    },
    sources: ["memory", "sessions"],
    sync: {
      sessions: {
        deltaBytes: 100000,      // 触发索引的增量字节数
        deltaMessages: 50        // 触发索引的增量消息数
      }
    }
  }
}
```

**注意事项**：
- 默认关闭（实验性功能）
- 会话更新是异步索引的，结果可能略有延迟
- `memory_search` 返回会话片段，但 `memory_get` 仍限于记忆文件
- 会话日志存储在 `~/.clawdbot/agents/<agentId>/sessions/*.jsonl`
- 任何有文件系统访问权限的进程/用户都可以读取会话日志

### 8.4 本地嵌入模型自动下载

使用本地嵌入模型时，如果模型文件不存在，`node-llama-cpp` 会自动下载。

**默认模型**：
- `hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf`
- 大小：约 0.6 GB

**下载位置**：
- 默认：`~/.cache/node-llama-cpp/models/`
- 可配置：`local.modelCacheDir`

**首次使用**：
1. 运行 `pnpm approve-builds`
2. 选择 `node-llama-cpp`
3. 运行 `pnpm rebuild node-llama-cpp`
4. 首次搜索时会自动下载模型

## 九、使用最佳实践

### 9.1 何时写入记忆

- **决策和偏好** → `MEMORY.md`
- **日常笔记和运行上下文** → `memory/YYYY-MM-DD.md`
- **"记住这个"** → 立即写入文件（不要只保存在 RAM 中）
- **重要的持久化信息** → 明确要求 bot 写入记忆

### 9.2 记忆内容建议

**好的记忆条目**：
```markdown
## 2026-01-28

- 用户偏好使用中文交流
- 项目使用 TypeScript + Vitest 进行测试
- 记忆系统使用 SQLite + 向量嵌入实现语义搜索
```

**不好的记忆条目**：
```markdown
## 2026-01-28

- 今天做了一些事
- 配置了东西
- 修复了问题
```

**关键原则**：
- 具体明确，包含足够的上下文
- 自包含（standalone），以后单独阅读也能理解
- 包含相关实体（人名、项目名、文件名等）
- 避免过于简短或模糊的描述

### 9.3 优化搜索质量

**提高召回率**：
- 启用混合搜索（结合向量和关键词）
- 增加 `maxResults` 数量
- 使用更高质量的嵌入模型

**提高精确度**：
- 写入更具体、自包含的记忆条目
- 使用一致的命名和术语
- 为重要实体创建专门的记忆条目

**优化性能**：
- 启用嵌入缓存
- 使用批量 API 进行大规模索引
- 启用 sqlite-vec 向量加速

## 十、故障排查

### 10.1 记忆搜索不工作

**检查清单**：
1. 确认 `memorySearch.enabled = true`
2. 检查嵌入提供商配置和 API 密钥
3. 运行 `moltbot memory status --deep` 查看详细状态
4. 查看日志中的错误信息

**常见问题**：
- API 密钥未配置或无效
- 本地模型未下载或编译失败
- 防火墙阻止 API 请求

### 10.2 索引过期或不更新

**解决方法**：
```bash
# 手动重建索引
moltbot memory index --verbose

# 检查文件监视是否正常
moltbot memory status --deep --index
```

**可能原因**：
- 文件监视被禁用（`sync.watch = false`）
- 文件系统不支持监视事件
- 索引数据库损坏

### 10.3 搜索结果质量差

**改进方法**：
1. 启用混合搜索（如果未启用）
2. 调整权重比例（尝试不同的 `vectorWeight` 和 `textWeight`）
3. 增加候选结果池（`candidateMultiplier`）
4. 改进记忆条目的质量和具体性
5. 考虑使用更好的嵌入模型

## 十一、与其他系统集成

### 11.1 Git 版本控制

记忆文件是纯文本 Markdown，可以直接用 Git 管理：

```bash
cd ~/clawd
git init
git add MEMORY.md memory/
git commit -m "Initial memory snapshot"
```

**优势**：
- 可追踪记忆变化历史
- 支持多设备同步
- 可以回滚到历史状态

### 11.2 备份策略

**推荐备份内容**：
- `~/clawd/MEMORY.md`
- `~/clawd/memory/`
- `~/.clawdbot/memory/*.sqlite`（可选，可重建）

**备份方式**：
- 定期 Git 提交并推送到远程仓库
- 使用 rsync 或其他备份工具
- 云同步服务（如 Dropbox、iCloud）

### 11.3 多设备同步

可以使用 Git 或文件同步服务在多个设备间同步记忆：

```bash
# 设备 A
cd ~/clawd
git add . && git commit -m "Update memory"
git push

# 设备 B
cd ~/clawd
git pull
moltbot memory index  # 重建索引
```

**注意**：SQLite 索引文件不需要同步，每个设备应该有自己的索引。

## 十二、未来发展方向

根据研究文档（`docs/experiments/research/memory.md`），记忆系统可能的演进方向：

1. **实体中心的记忆**：
   - 为重要实体（人、项目、概念）创建专门的记忆页面
   - 支持实体链接和关系

2. **置信度和证据跟踪**：
   - 为观点和偏好添加置信度评分
   - 跟踪支持和反驳证据

3. **定期反思和整理**：
   - 自动从每日日志中提取重要事实
   - 定期更新和整理长期记忆

4. **时间感知查询**：
   - 支持"上周发生了什么"类型的查询
   - 时间范围过滤

5. **更先进的检索算法**：
   - 考虑使用 HNSW 或 SuCo 等高级 ANN 算法
   - 改进大规模记忆的检索性能

## 总结

Moltbot 的记忆系统是一个设计精良、功能强大的持久化方案：

**核心优势**：
- ✅ **人类可读**：纯 Markdown 文件，可直接编辑和审查
- ✅ **可靠持久**：基于文件系统，不依赖 RAM 或云服务
- ✅ **语义搜索**：支持向量嵌入和混合搜索
- ✅ **灵活配置**：支持多种嵌入提供商和高级选项
- ✅ **自动管理**：文件监视、自动索引、记忆刷新
- ✅ **性能优化**：嵌入缓存、批量 API、向量加速

**使用建议**：
- 养成明确写入重要信息的习惯
- 利用语义搜索找回历史信息
- 根据需求选择合适的嵌入提供商
- 定期备份记忆文件

**了解更多**：
- 英文文档：[Memory](/concepts/memory)
- CLI 参考：[moltbot memory](/cli/memory)
- 会话管理：[Session management + compaction](/reference/session-management-compaction)
- 插件系统：[Plugins](/plugins)
