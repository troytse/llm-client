# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 常用命令

```bash
# 开发模式 (Chrome MV3)
pnpm dev

# 生产构建
pnpm build
pnpm package

# Firefox 开发/构建
pnpm dev:firefox
pnpm build:firefox
pnpm package:firefox

# 代码检查
pnpm lint          # 检查代码风格
pnpm lint:fix      # 自动修复问题
pnpm format        # 格式化代码
pnpm format:fix    # 自动格式化

# 测试
pnpm test          # 交互式测试
pnpm test:ui       # Vitest UI 模式
pnpm test:run      # 运行所有测试
pnpm test:coverage # 生成覆盖率报告
pnpm test:watch    # 监视模式

# 清理构建产物
pnpm clean

# 生成 i18n 资源
pnpm generate:resources
```

## 项目架构概览

这是一个基于 WXT 框架的浏览器扩展项目，用于与本地 LLM 提供商（Ollama、LM Studio、llama.cpp）进行聊天。项目采用 multi-provider 架构，通过抽象层支持多个后端。

### 扩展入口点

- `src/entrypoints/sidepanel/index.tsx` - 侧边栏聊天 UI
- `src/entrypoints/options/index.tsx` - 设置页面
- `src/entrypoints/background.ts` - 后台服务工作器
- `src/entrypoints/content.ts` - 内容脚本
- `src/entrypoints/selection-button.content.tsx` - 文本选择按钮

### 数据流架构

```
Sidepanel UI → Runtime Port → Background Worker → ProviderFactory → Provider → Stream
     ↓                                                                            ↓
Dexie Chat Store ←──────────────────────────────────────────────────────────────┘
     ↓
Optional RAG Pipeline → Embeddings → Vector DB (Dexie)
```

1. Sidepanel 收集用户输入，通过 runtime port 发送到 background
2. Background 通过 `ProviderFactory.getProviderForModel(modelId)` 解析 provider
3. Provider 处理流式响应并回传到 UI
4. UI 更新状态并持久化到 Dexie IndexedDB
5. 可选的 RAG 管道检索本地上下文

## 核心模块

### Provider 抽象层 (`src/lib/providers/`)

- `types.ts` - Provider 接口定义 (`LLMProvider`, `ProviderConfig`, `ChatRequest`)
- `factory.ts` - 根据 modelId 或 config 创建 provider 实例
- `manager.ts` - Provider 配置和映射管理
- `registry.ts` - Provider 显示名称和图标
- `ollama.ts` / `lm-studio.ts` / `llama-cpp.ts` / `openai.ts` - 各 provider 实现

**重要**: Provider 路由通过 `model_provider_mappings` 存储，如果映射不存在则默认使用 Ollama。

### Background Handlers (`src/background/handlers/`)

- `handle-chat-with-model.ts` - 聊天流式处理
- `handle-chat-stream.ts` - 流式响应中继
- `handle-embed-chunks.ts` - 文件块嵌入处理
- `handle-*.ts` - 各种模型管理操作（pull、delete、unload 等）

### 存储 (`src/lib/`)

- `db.ts` - Dexie 聊天/会话/文件数据库（主存储）
- `plasmo-global-storage.ts` - 设置和 provider 配置存储
- `sqlite/db.ts` - SQLite 迁移路径（辅助存储，非主要运行时）
- `embeddings/db.ts` - 向量数据库

**注意**: 存在 Dexie + SQLite 双存储架构，SQLite 是迁移路径，主要运行时使用 Dexie。

### RAG 系统 (`src/lib/rag/core/`, `src/lib/embeddings/`)

浏览器优先的 RAG 模块契约：
- `interfaces.ts` - 核心接口 (`DocumentSource`, `Chunker`, `Embedder`, `VectorStore`, `Retriever`, `PromptAssembler`)
- `browser-*.ts` - 浏览器适配器实现
- `embedding-strategy.ts` - 嵌入回退链：provider-native → shared MiniLM → background warmup → Ollama fallback

### 消息传递

消息键定义在 `src/lib/constants/keys.ts`:

- `MESSAGE_KEYS.PROVIDER.*` - Provider 通用消息
- `MESSAGE_KEYS.OLLAMA.*` - Ollama 专用消息（遗留兼容）
- `STORAGE_KEYS.*` - 存储键

**注意**: 为保持向后兼容，遗留的 `ollama-*` 键仍被支持。新代码应使用 `PROVIDER.*` 键。

### Feature 模块

- `src/features/chat/` - 聊天 UI、hooks、RAG 管道
- `src/features/model/` - 模型配置、管理 UI
- `src/features/file-upload/` - 文件上传和处理器
- `src/features/knowledge/` - 知识库/RAG 设置

### 文件处理器 (`src/features/file-upload/processors/`)

支持的文件类型：
- 文本 (`.txt`, `.md`)
- PDF (`.pdf`)
- Word (`.docx`)
- CSV/TSV/PSV
- HTML

### 数据库迁移

- `src/lib/storage/provider-migration.ts` - 从 Ollama 遗留键迁移到 provider 通用键
- `src/background/migrations/rag-quality-migration.ts` - RAG 质量迁移
- Dexie 版本升级内嵌在 `src/lib/db.ts`

## 关键技术约束

1. **CSP 限制**: Chrome 扩展 CSP 限制某些 WASM/worker ML 路径
2. **Firefox 差异**: Firefox 缺少 Chrome DNR API 行为
3. **浏览器优先**: RAG 运行时是浏览器优先的，不假设 Node.js API
4. **本地优先**: 所有数据默认本地存储，无内置遥测

## 测试配置

- 测试框架: Vitest
- 环境: happy-dom
- 设置文件: `src/test/setup.ts`
- 覆盖率: v8 provider
- 别名: `@/*` 映射到 `./src/*`

## 类型路径

- `@/*` → `./src/*`
- Chrome types: `@types/chrome`

## 重要约定

1. **向后兼容**: 保持存储键向后兼容，除非包含迁移
2. **Provider 不可知**: 新代码避免假设 Ollama 专用行为
3. **本地优先**: 隐私和数据安全是产品行为，而非可选增强
4. **显式行为**: 避免隐藏的网络调用或静默遥测
5. **特性局部化**: 使用特性局部模块而非将逻辑分散到不相关文件夹

## 添加新 Provider 检查清单

1. 在 `src/lib/providers/<provider>.ts` 实现 `LLMProvider`
2. 在 `src/lib/providers/types.ts` 添加 provider id/type
3. 在 `src/lib/providers/manager.ts` 注册默认配置
4. 在 `src/lib/providers/factory.ts` 添加创建逻辑
5. 验证 provider 设置和模型列表分组 UI
6. 添加测试：模型列表、流式解析、错误处理

## 文档参考

- 架构指南: `docs/architecture.md`
- Provider 支持: `docs/providers.md`
- RAG 指南: `docs/rag.md`
- 浏览器优先 RAG 核心: `docs/rag-browser-core.md`

## Active Technologies
- TypeScript 5.x, React 18+, WXT Framework + React, Plasmo, Lucide Icons, Dexie (001-openai-api-provider)
- Plasmo Global Storage (provider configurations), Dexie IndexedDB (chats/messages) (001-openai-api-provider)

## Recent Changes
- 001-openai-api-provider: Added TypeScript 5.x, React 18+, WXT Framework + React, Plasmo, Lucide Icons, Dexie
