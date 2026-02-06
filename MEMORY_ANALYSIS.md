# OpenClaw Memory 系统详细解析

## 1. Memory 系统概述

Memory系统是OpenClaw的核心功能之一，用于存储和检索AI代理的长期记忆。它支持语义搜索、全文搜索，并能从Markdown文件中提取和索引信息。

### 1.1 核心功能
- **语义搜索**: 基于向量嵌入的相似度搜索
- **全文搜索**: 基于SQLite FTS的关键词搜索
- **混合搜索**: 结合向量和关键词搜索的结果
- **增量同步**: 只索引变更的文件
- **会话记忆**: 索引会话历史记录

### 1.2 数据源

Memory系统支持两种数据源，分别对应**长期记忆**和**短期记忆**：

#### 长期记忆（Long-term Memory）
- **存储位置**: `MEMORY.md`, `memory.md`, `memory/*.md`
- **数据源标识**: `source = "memory"`
- **特点**: 持久化、跨会话、经过筛选的重要信息
- **更新方式**: 通过Memory刷新机制主动提取

#### 短期记忆（Short-term Memory）
- **存储位置**: 会话文件（Session Transcripts），格式为 `.jsonl`
- **数据源标识**: `source = "sessions"`
- **特点**: 临时、会话特定、完整的对话历史
- **更新方式**: 每次消息交互自动追加

#### 自定义路径
- 用户指定的额外Markdown文件路径（可配置）

> **详细说明**: 关于长期记忆和短期记忆的详细区分，请参考 [MEMORY_LONG_SHORT_TERM.md](./MEMORY_LONG_SHORT_TERM.md)

## 2. Memory 架构

### 2.1 组件层次

```
Memory系统
├── Backend层
│   ├── builtin (MemoryIndexManager)
│   └── qmd (QmdMemoryManager)
├── Search Manager层
│   └── getMemorySearchManager() - 统一入口
├── Embedding层
│   ├── OpenAI Embeddings
│   ├── Gemini Embeddings
│   └── Local Embeddings (node-llama-cpp)
├── Storage层
│   ├── SQLite数据库
│   ├── sqlite-vec扩展 (向量搜索)
│   └── SQLite FTS (全文搜索)
└── Sync层
    ├── 文件监听 (chokidar)
    ├── 增量同步
    └── 批量嵌入
```

### 2.2 核心类与接口

#### 2.2.1 MemorySearchManager 接口
```typescript
interface MemorySearchManager {
  search(query: string, opts?: SearchOptions): Promise<MemorySearchResult[]>;
  readFile(params: ReadFileParams): Promise<ReadFileResult>;
  status(): MemoryProviderStatus;
  sync?(params?: SyncParams): Promise<void>;
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;
  close?(): Promise<void>;
}
```

#### 2.2.2 MemoryIndexManager (内置后端)
- **位置**: `src/memory/manager.ts`
- **功能**: 实现builtin后端的所有功能
- **特点**: 
  - SQLite数据库存储
  - 向量搜索 + FTS混合搜索
  - 增量同步
  - 文件监听

#### 2.2.3 QmdMemoryManager (QMD后端)
- **位置**: `src/memory/qmd-manager.ts`
- **功能**: 集成外部QMD工具
- **特点**: 
  - 委托给外部QMD进程
  - 支持多个索引集合
  - 自动降级到builtin

## 3. 数据流详解

### 3.1 Memory索引流程

```
文件变更检测
    ↓
文件监听 (chokidar) 或 手动同步
    ↓
文件列表收集
    ├─> listMemoryFiles() - Memory文件
    └─> listSessionFilesForAgent() - 会话文件
    ↓
文件哈希计算
    ├─> buildFileEntry() - 计算文件哈希
    └─> 检查是否需要重新索引
    ↓
Markdown分块
    ├─> chunkMarkdown() - 按token数分块
    └─> 生成chunks (startLine, endLine, text, hash)
    ↓
嵌入生成
    ├─> embedQuery() / embedBatch() - 生成向量
    ├─> 批量API调用 (OpenAI/Gemini)
    └─> 本地模型 (node-llama-cpp)
    ↓
数据库存储
    ├─> files表 - 文件元数据
    ├─> chunks表 - 文本块
    ├─> chunks_vec表 - 向量数据
    └─> chunks_fts表 - FTS索引
    ↓
索引完成
```

### 3.2 Memory搜索流程

```
用户查询
    ↓
getMemorySearchManager()
    ├─> resolveMemoryBackendConfig() - 解析后端配置
    ├─> 如果是qmd → QmdMemoryManager
    └─> 如果是builtin → MemoryIndexManager
    ↓
manager.search(query, opts)
    ├─> 混合搜索模式?
    │   ├─> 是 → 并行执行向量搜索和FTS搜索
    │   │   ├─> searchVector() - 向量搜索
    │   │   └─> searchKeyword() - 关键词搜索
    │   │   ↓
    │   └─> mergeHybridResults() - 合并结果
    │
    └─> 单一模式
        └─> searchVector() 或 searchKeyword()
    ↓
结果排序和过滤
    ├─> 按score排序
    ├─> 过滤minScore
    └─> 限制maxResults
    ↓
生成snippets
    ├─> 提取文本片段
    └─> 添加引用信息（如果启用）
    ↓
返回结果
```

### 3.3 嵌入生成流程

#### 3.3.1 嵌入提供者选择

```
createEmbeddingProvider(options)
    ├─> provider === "auto"?
    │   ├─> 是 → 自动选择
    │   │   ├─> 尝试local (如果配置了)
    │   │   ├─> 尝试openai (如果有API key)
    │   │   └─> 尝试gemini (如果有API key)
    │   │
    │   └─> 否 → 使用指定provider
    │
    ├─> provider === "openai"
    │   └─> createOpenAiEmbeddingProvider()
    │       ├─> 检查API key
    │       ├─> 创建OpenAI客户端
    │       └─> 返回provider
    │
    ├─> provider === "gemini"
    │   └─> createGeminiEmbeddingProvider()
    │       ├─> 检查API key
    │       ├─> 创建Gemini客户端
    │       └─> 返回provider
    │
    └─> provider === "local"
        └─> createLocalEmbeddingProvider()
            ├─> 加载node-llama-cpp
            ├─> 加载模型
            └─> 返回provider
```

#### 3.3.2 批量嵌入处理

```
需要嵌入的文本列表
    ↓
检查批量模式
    ├─> batch.enabled === true?
    │   ├─> 是 → 批量API调用
    │   │   ├─> OpenAI: runOpenAiEmbeddingBatches()
    │   │   │   ├─> 创建批量请求
    │   │   │   ├─> 提交到OpenAI Batch API
    │   │   │   ├─> 轮询状态 (pollIntervalMs)
    │   │   │   └─> 获取结果
    │   │   │
    │   │   └─> Gemini: runGeminiEmbeddingBatches()
    │   │       ├─> 分批处理 (每批maxTokens)
    │   │       └─> 并发调用 (concurrency)
    │   │
    │   └─> 否 → 直接API调用
    │       └─> embedBatch() - 并发调用
    ↓
嵌入结果
    ├─> 归一化向量
    └─> 存储到数据库
```

### 3.4 同步机制

#### 3.4.1 文件监听同步

```
启动MemoryIndexManager
    ↓
初始化文件监听器 (chokidar)
    ├─> 监听Memory文件目录
    └─> 监听会话文件目录
    ↓
文件变更事件
    ├─> add/change事件
    │   └─> 标记为dirty
    │
    └─> unlink事件
        └─> 标记为删除
    ↓
防抖处理 (watchDebounceMs)
    ├─> 延迟执行同步
    └─> 合并多次变更
    ↓
触发同步
    └─> sync() - 增量同步
```

#### 3.4.2 增量同步流程

```
sync(reason, force, progress)
    ↓
检查是否需要全量重建
    ├─> force === true?
    ├─> 数据库元数据不匹配?
    └─> 向量维度变化?
    ↓
同步Memory文件
    ├─> syncMemoryFiles()
    │   ├─> listMemoryFiles() - 列出所有文件
    │   ├─> buildFileEntry() - 构建文件条目
    │   ├─> 检查哈希是否变化
    │   ├─> 如果变化 → indexFile()
    │   └─> 删除不存在的文件记录
    │
    └─> syncSessionFiles()
        ├─> listSessionFilesForAgent() - 列出会话文件
        ├─> 检查dirtyFiles集合
        ├─> 增量读取会话文件 (deltaBytes/deltaMessages)
        ├─> 如果变化 → indexFile()
        └─> 删除不存在的文件记录
    ↓
indexFile(entry)
    ├─> 读取文件内容
    ├─> chunkMarkdown() - 分块
    ├─> 检查chunk哈希
    ├─> 生成嵌入 (如果需要)
    ├─> 存储到数据库
    └─> 更新FTS索引
```

#### 3.4.3 会话文件增量读取

```
会话文件变更检测
    ↓
计算增量
    ├─> 读取文件大小变化 (deltaBytes)
    ├─> 读取消息数量变化 (deltaMessages)
    └─> 记录lastSize
    ↓
增量读取
    ├─> 从lastSize位置读取
    ├─> 解析新增的Markdown内容
    └─> 只索引新增的chunks
    ↓
更新索引
```

### 3.5 混合搜索算法

#### 3.5.1 向量搜索

```
查询文本
    ↓
生成查询向量
    └─> embedQuery(query)
    ↓
SQLite向量搜索
    ├─> 加载sqlite-vec扩展
    ├─> 执行向量相似度查询
    └─> 返回top-k结果 (candidateMultiplier * maxResults)
    ↓
计算向量分数
    └─> 余弦相似度 (0-1)
```

#### 3.5.2 关键词搜索 (FTS)

```
查询文本
    ↓
构建FTS查询
    ├─> buildFtsQuery()
    │   ├─> 提取token
    │   └─> 构建AND查询
    │
    └─> SQLite FTS查询
        └─> BM25排序
    ↓
计算文本分数
    ├─> bm25RankToScore()
    └─> 转换为0-1分数
```

#### 3.5.3 结果合并

```
向量结果 + 关键词结果
    ↓
按ID合并
    ├─> 相同ID的结果合并
    │   ├─> 保留较高的snippet
    │   └─> 合并分数
    │
    └─> 不同ID的结果添加
    ↓
计算最终分数
    ├─> score = vectorWeight * vectorScore + textWeight * textScore
    └─> 归一化权重
    ↓
排序和过滤
    ├─> 按score降序排序
    ├─> 过滤minScore
    └─> 限制maxResults
```

## 4. 数据库架构

### 4.1 表结构

#### 4.1.1 files表
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

#### 4.1.2 chunks表
```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL,
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  text TEXT NOT NULL,
  hash TEXT NOT NULL,
  FOREIGN KEY (path, source) REFERENCES files(path, source)
);
```

#### 4.1.3 chunks_vec表 (向量数据)
```sql
CREATE TABLE chunks_vec (
  id TEXT PRIMARY KEY,
  embedding BLOB NOT NULL,  -- Float32Array序列化
  FOREIGN KEY (id) REFERENCES chunks(id)
);
```

#### 4.1.4 chunks_fts表 (全文搜索)
```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  path,
  source,
  text,
  model,
  content='chunks',
  content_rowid='rowid'
);
```

#### 4.1.5 embedding_cache表 (嵌入缓存)
```sql
CREATE TABLE embedding_cache (
  text_hash TEXT PRIMARY KEY,
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  embedding BLOB NOT NULL,
  created_at INTEGER NOT NULL
);
```

### 4.2 索引策略

- **files表**: (path, source) 主键
- **chunks表**: (path, source) 外键索引
- **chunks_vec表**: 向量索引 (sqlite-vec)
- **chunks_fts表**: FTS5虚拟表索引

## 5. 配置系统

### 5.1 Memory配置层次

```
全局配置 (memory)
├── backend: "builtin" | "qmd"
├── citations: "auto" | "on" | "off"
└── qmd: QmdConfig

Agent默认配置 (agents.defaults.memorySearch)
├── enabled: boolean
├── sources: ["memory"] | ["sessions"] | ["memory", "sessions"]
├── provider: "openai" | "gemini" | "local" | "auto"
├── model: string
├── remote: RemoteConfig
├── local: LocalConfig
├── store: StoreConfig
├── chunking: ChunkingConfig
├── sync: SyncConfig
├── query: QueryConfig
└── cache: CacheConfig

Agent特定配置 (agents.[agentId].memorySearch)
└── 覆盖默认配置
```

### 5.2 关键配置项

#### 5.2.1 嵌入提供者配置
```typescript
provider: "openai" | "gemini" | "local" | "auto"
model: string  // 模型名称
remote: {
  baseUrl?: string
  apiKey?: string
  headers?: Record<string, string>
  batch?: {
    enabled: boolean
    wait: boolean
    concurrency: number
    pollIntervalMs: number
    timeoutMinutes: number
  }
}
local: {
  modelPath?: string
  modelCacheDir?: string
}
fallback: "openai" | "gemini" | "local" | "none"
```

#### 5.2.2 分块配置
```typescript
chunking: {
  tokens: number      // 默认400
  overlap: number     // 默认80
}
```

#### 5.2.3 同步配置
```typescript
sync: {
  onSessionStart: boolean    // 会话开始时同步
  onSearch: boolean          // 搜索时同步
  watch: boolean             // 文件监听
  watchDebounceMs: number    // 防抖时间
  intervalMinutes: number    // 定时同步间隔
  sessions: {
    deltaBytes: number       // 增量字节阈值
    deltaMessages: number    // 增量消息阈值
  }
}
```

#### 5.2.4 查询配置
```typescript
query: {
  maxResults: number         // 默认6
  minScore: number            // 默认0.35
  hybrid: {
    enabled: boolean         // 默认true
    vectorWeight: number     // 默认0.7
    textWeight: number       // 默认0.3
    candidateMultiplier: number  // 默认4
  }
}
```

## 6. Memory工具集成

### 6.1 memory_search工具

#### 6.1.1 工具定义
```typescript
{
  name: "memory_search",
  description: "语义搜索MEMORY.md和memory/*.md文件",
  parameters: {
    query: string,
    maxResults?: number,
    minScore?: number
  }
}
```

#### 6.1.2 执行流程
```
Agent调用memory_search工具
    ↓
createMemorySearchTool().execute()
    ├─> getMemorySearchManager() - 获取管理器
    ├─> manager.search() - 执行搜索
    ├─> 处理引用 (如果启用)
    └─> 返回结果
    ↓
结果注入Agent上下文
    └─> Agent使用结果生成回复
```

#### 6.1.3 引用处理
- **auto模式**: 直接聊天显示引用，群组/频道隐藏
- **on模式**: 总是显示引用
- **off模式**: 不显示引用

### 6.2 memory_get工具

#### 6.2.1 工具定义
```typescript
{
  name: "memory_get",
  description: "读取MEMORY.md或memory/*.md的指定行",
  parameters: {
    path: string,
    from?: number,
    lines?: number
  }
}
```

#### 6.2.2 执行流程
```
Agent调用memory_get工具
    ↓
createMemoryGetTool().execute()
    ├─> manager.readFile() - 读取文件
    └─> 返回指定行的文本
    ↓
结果注入Agent上下文
```

## 7. Memory刷新机制

### 7.1 触发条件

```
会话token使用量检查
    ├─> totalTokens >= threshold?
    │   ├─> threshold = contextWindow - reserveTokensFloor - softThresholdTokens
    │   └─> 检查是否已刷新 (compactionCount)
    │
    └─> 触发Memory刷新
```

### 7.2 刷新流程

```
runMemoryFlushIfNeeded()
    ↓
检查条件
    ├─> Memory刷新是否启用?
    ├─> 工作空间是否可写?
    ├─> 是否达到阈值?
    └─> 是否已刷新?
    ↓
运行Memory刷新Agent轮次
    ├─> 使用特殊prompt
    │   └─> "Pre-compaction memory flush. Store durable memories..."
    ├─> 运行Agent
    └─> Agent写入memory/*.md文件
    ↓
更新会话元数据
    ├─> memoryFlushAt: timestamp
    └─> memoryFlushCompactionCount: compactionCount
    ↓
触发Memory同步
    └─> 新文件被索引
```

## 8. 性能优化策略

### 8.1 索引优化

1. **增量同步**: 只索引变更的文件
2. **哈希检查**: 文件内容哈希避免重复索引
3. **批量嵌入**: 批量API调用减少网络开销
4. **并发控制**: 限制并发索引任务数
5. **缓存**: 嵌入结果缓存避免重复计算

### 8.2 搜索优化

1. **混合搜索**: 结合向量和关键词搜索
2. **候选扩展**: 返回更多候选结果再过滤
3. **分数归一化**: 统一分数范围便于合并
4. **结果限制**: 早期限制结果数量

### 8.3 存储优化

1. **向量压缩**: Float32Array序列化
2. **索引优化**: 适当的数据库索引
3. **清理策略**: 定期清理过期缓存

## 9. 错误处理与降级

### 9.1 嵌入提供者降级

```
首选提供者失败
    ↓
检查fallback配置
    ├─> fallback === "none"?
    │   └─> 返回错误
    │
    └─> 尝试fallback提供者
        ├─> 记录降级原因
        └─> 使用fallback提供者
```

### 9.2 QMD后端降级

```
QmdMemoryManager失败
    ↓
FallbackMemoryManager
    ├─> 记录错误
    ├─> 关闭QMD管理器
    └─> 切换到builtin后端
    ↓
后续请求使用builtin
```

### 9.3 搜索错误处理

```
搜索失败
    ├─> 记录错误日志
    ├─> 返回空结果
    └─> 不阻塞Agent执行
```

## 10. 扩展点

### 10.1 自定义后端

实现`MemorySearchManager`接口：
```typescript
class CustomMemoryManager implements MemorySearchManager {
  async search(...) { ... }
  async readFile(...) { ... }
  status() { ... }
  // ...
}
```

### 10.2 自定义嵌入提供者

实现`EmbeddingProvider`接口：
```typescript
const customProvider: EmbeddingProvider = {
  id: "custom",
  model: "custom-model",
  embedQuery: async (text) => { ... },
  embedBatch: async (texts) => { ... }
}
```

## 11. 总结

Memory系统是OpenClaw的核心功能，提供了：

1. **强大的搜索能力**: 语义搜索 + 全文搜索
2. **灵活的配置**: 多层次的配置系统
3. **高性能**: 增量同步、批量处理、缓存
4. **可扩展性**: 支持多种后端和嵌入提供者
5. **智能刷新**: 自动在压缩前保存重要信息

系统设计考虑了性能、可扩展性和用户体验，是一个成熟的Memory管理解决方案。

