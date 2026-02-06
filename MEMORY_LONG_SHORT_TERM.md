# OpenClaw Memory 系统：长期记忆与短期记忆的区分

## 1. 概述

OpenClaw的Memory系统明确区分了**长期记忆（Long-term Memory）**和**短期记忆（Short-term Memory）**，这两种记忆类型在存储位置、生命周期、用途和转换机制上都有显著差异。

## 2. 记忆类型对比

### 2.1 长期记忆（Long-term Memory）

#### 特征
- **存储位置**: `MEMORY.md`, `memory.md`, `memory/*.md` 文件
- **数据源标识**: `source = "memory"`
- **生命周期**: **持久化、跨会话**
- **更新频率**: 通过Memory刷新机制主动提取
- **内容性质**: 经过筛选的、重要的、需要长期保留的信息

#### 存储结构
```
workspace/
├── MEMORY.md          # 主Memory文件
├── memory.md          # 备用Memory文件
└── memory/
    ├── 2024-01-15.md  # 按日期组织的Memory文件
    ├── 2024-01-16.md
    └── ...
```

#### 用途
- 存储用户偏好、重要决策、项目信息等需要长期记忆的内容
- 跨会话检索，即使会话被压缩或重置也能访问
- 作为Agent的"知识库"，提供上下文信息

### 2.2 短期记忆（Short-term Memory）

#### 特征
- **存储位置**: 会话文件（Session Transcripts），格式为 `.jsonl` 文件
- **数据源标识**: `source = "sessions"`
- **生命周期**: **临时、会话特定**
- **更新频率**: 每次消息交互自动追加
- **内容性质**: 完整的会话历史记录，包括所有对话内容

#### 存储结构
```
~/.local/share/openclaw/agents/{agentId}/sessions/
├── {sessionId}.jsonl           # 会话文件
├── {sessionId}-topic-{id}.jsonl # 带主题的会话文件
└── sessions.json                # 会话元数据
```

#### 会话文件格式
```jsonl
{"type":"header","sessionId":"...","createdAt":...}
{"type":"message","message":{"role":"user","content":"..."}}
{"type":"message","message":{"role":"assistant","content":"..."}}
...
```

#### 用途
- 存储当前会话的完整对话历史
- 提供会话上下文，支持多轮对话
- 通过压缩机制管理上下文长度

## 3. 数据流与转换机制

### 3.1 记忆写入流程

```
用户消息
    ↓
会话文件追加 (短期记忆)
    ├─> 写入 {sessionId}.jsonl
    └─> 自动索引到Memory系统 (如果sources包含"sessions")
    ↓
Agent处理
    ├─> 使用短期记忆作为上下文
    └─> 生成回复
    ↓
回复追加到会话文件 (短期记忆)
    ↓
检查是否需要Memory刷新
    ├─> 会话token接近上下文窗口限制?
    └─> 触发Memory刷新
        ├─> 运行特殊的Agent轮次
        ├─> Agent提取重要信息
        └─> 写入memory/YYYY-MM-DD.md (长期记忆)
```

### 3.2 Memory刷新机制（Memory Flush）

#### 触发条件
```typescript
// 从 memory-flush.ts
shouldRunMemoryFlush({
  totalTokens >= threshold
  // threshold = contextWindow - reserveTokensFloor - softThresholdTokens
  // 默认: 上下文窗口 - 保留token - 4000软阈值
})
```

#### 刷新流程
```
会话接近上下文窗口限制
    ↓
runMemoryFlushIfNeeded()
    ├─> 检查Memory刷新设置
    ├─> 检查工作空间是否可写
    └─> 检查是否达到阈值
    ↓
运行Memory刷新Agent轮次
    ├─> Prompt: "Pre-compaction memory flush. Store durable memories now..."
    ├─> System Prompt: "capture durable memories to disk"
    └─> Agent决定哪些信息需要保存
    ↓
Agent写入memory/YYYY-MM-DD.md
    ├─> 创建memory/目录（如果不存在）
    └─> 写入重要信息
    ↓
更新会话元数据
    ├─> memoryFlushAt: timestamp
    └─> memoryFlushCompactionCount: compactionCount
    ↓
触发Memory同步
    └─> 新文件被索引到Memory系统
```

#### 刷新Prompt示例
```typescript
// 默认Memory刷新Prompt
"Pre-compaction memory flush.
Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).
If nothing to store, reply with <SILENT_REPLY_TOKEN>."
```

### 3.3 会话压缩（Compaction）

#### 压缩机制
- **目的**: 减少会话上下文长度，避免超出模型上下文窗口
- **时机**: 当会话token数接近上下文窗口限制时自动触发
- **方式**: Pi Agent的compaction功能，智能压缩会话历史

#### 压缩与Memory刷新的关系
```
会话token接近限制
    ↓
先执行Memory刷新
    └─> 将重要信息保存到长期记忆
    ↓
然后执行压缩
    └─> 压缩会话历史，但保留在会话文件中
    ↓
压缩后的会话
    ├─> 短期记忆被压缩，但未删除
    └─> 长期记忆已保存，可跨会话访问
```

## 4. Memory搜索中的区分

### 4.1 数据源配置

```typescript
// 配置示例
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "sources": ["memory"]  // 只搜索长期记忆
        // 或
        "sources": ["memory", "sessions"]  // 搜索长期和短期记忆
      }
    }
  }
}
```

### 4.2 搜索结果标识

```typescript
type MemorySearchResult = {
  path: string;           // 文件路径
  startLine: number;      // 起始行号
  endLine: number;        // 结束行号
  score: number;          // 相似度分数
  snippet: string;        // 文本片段
  source: MemorySource;   // "memory" | "sessions"
  citation?: string;      // 引用信息
}
```

### 4.3 搜索行为差异

#### 长期记忆搜索
- **路径格式**: `MEMORY.md`, `memory/2024-01-15.md`
- **特点**: 
  - 跨会话可用
  - 内容经过筛选，质量较高
  - 适合检索用户偏好、项目信息等

#### 短期记忆搜索
- **路径格式**: `sessions/{sessionId}.jsonl`
- **特点**: 
  - 仅当前会话可用（除非明确配置）
  - 包含完整对话历史
  - 适合检索当前会话的上下文

## 5. 数据库中的区分

### 5.1 文件表（files表）

```sql
CREATE TABLE files (
  path TEXT NOT NULL,
  source TEXT NOT NULL,  -- 'memory' | 'sessions'
  hash TEXT NOT NULL,
  mtime_ms INTEGER NOT NULL,
  size INTEGER NOT NULL,
  PRIMARY KEY (path, source)
);
```

- **source = "memory"**: 长期记忆文件
- **source = "sessions"**: 短期记忆（会话）文件

### 5.2 块表（chunks表）

```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL,  -- 'memory' | 'sessions'
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  text TEXT NOT NULL,
  hash TEXT NOT NULL,
  FOREIGN KEY (path, source) REFERENCES files(path, source)
);
```

每个chunk都标记了其来源（source），便于区分和过滤。

## 6. 配置选项

### 6.1 启用会话记忆索引

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "sources": ["memory", "sessions"],
        "experimental": {
          "sessionMemory": true
        }
      }
    }
  }
}
```

### 6.2 Memory刷新配置

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "prompt": "Pre-compaction memory flush. Store durable memories...",
          "systemPrompt": "Pre-compaction memory flush turn..."
        },
        "reserveTokensFloor": 2000
      }
    }
  }
}
```

## 7. 使用场景

### 7.1 长期记忆适用场景
- ✅ 用户偏好设置（喜欢的颜色、风格等）
- ✅ 项目重要信息（项目目标、关键决策）
- ✅ 个人习惯（工作时间、常用工具）
- ✅ 需要跨会话记住的信息

### 7.2 短期记忆适用场景
- ✅ 当前对话的上下文
- ✅ 临时讨论的话题
- ✅ 会话特定的信息
- ✅ 不需要长期保留的对话内容

## 8. 最佳实践

### 8.1 何时使用长期记忆
1. **重要信息**: 需要长期记住的信息应写入memory文件
2. **用户偏好**: 用户明确表达的偏好应保存
3. **项目信息**: 项目相关的关键信息应记录

### 8.2 何时依赖短期记忆
1. **对话上下文**: 当前对话的上下文自动在会话文件中
2. **临时信息**: 不需要长期保留的信息可留在会话中
3. **会话特定**: 只在当前会话中有用的信息

### 8.3 Memory刷新策略
1. **主动刷新**: 在重要对话后，可以手动触发Memory刷新
2. **自动刷新**: 系统会在压缩前自动触发刷新
3. **定期整理**: 定期检查memory文件，整理和更新内容

## 9. 技术实现细节

### 9.1 长期记忆索引
```typescript
// 从 sync-memory-files.ts
async function syncMemoryFiles() {
  // 1. 列出所有Memory文件
  const files = await listMemoryFiles(workspaceDir, extraPaths);
  
  // 2. 检查文件哈希
  const fileEntries = await buildFileEntry(files);
  
  // 3. 如果哈希变化，重新索引
  if (hashChanged) {
    await indexFile(entry);
  }
}
```

### 9.2 短期记忆索引
```typescript
// 从 sync-session-files.ts
async function syncSessionFiles() {
  // 1. 列出所有会话文件
  const files = await listSessionFilesForAgent(agentId);
  
  // 2. 检查dirtyFiles集合（增量更新）
  if (dirtyFiles.has(file) || needsFullReindex) {
    await indexFile(entry);
  }
  
  // 3. 增量读取会话文件
  const delta = readDeltaFromFile(file, lastSize);
}
```

### 9.3 增量同步机制
- **Memory文件**: 通过文件哈希检测变更
- **会话文件**: 通过增量字节数（deltaBytes）和消息数（deltaMessages）检测变更

## 10. 总结

OpenClaw的Memory系统通过以下方式区分长期记忆和短期记忆：

1. **存储位置**:
   - 长期记忆: `MEMORY.md`, `memory/*.md`
   - 短期记忆: 会话文件（`.jsonl`）

2. **数据源标识**:
   - 长期记忆: `source = "memory"`
   - 短期记忆: `source = "sessions"`

3. **生命周期**:
   - 长期记忆: 持久化、跨会话
   - 短期记忆: 临时、会话特定

4. **转换机制**:
   - Memory刷新: 从短期记忆提取重要信息到长期记忆
   - 会话压缩: 压缩短期记忆，但保留在会话文件中

5. **搜索配置**:
   - 可以单独搜索长期记忆或短期记忆
   - 也可以同时搜索两者

这种设计使得Agent既能记住重要的长期信息，又能利用当前会话的上下文，实现了灵活而强大的记忆管理。

