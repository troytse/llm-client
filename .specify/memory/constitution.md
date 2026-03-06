<!--
Sync Impact Report:
  - Version change: initial → 1.0.0
  - Modified principles: none (initial creation)
  - Added sections: Core Principles (2), Governance
  - Removed sections: none
  - Templates requiring updates:
    ✅ plan-template.md (Constitution Check section verified)
    ✅ spec-template.md (requirements alignment verified)
    ✅ tasks-template.md (task categorization verified)
  - Follow-up TODOs: none
-->

# llm-client Constitution

## Core Principles

### 一、中文回复原则

所有交流、文档、代码注释和说明必须使用中文。具体要求：

- 用户交互的所有回复必须使用中文
- 代码注释必须使用中文解释逻辑和用途
- 文档说明必须使用中文
- 错误提示和用户可见的消息必须使用中文
- 技术术语和代码标识符保持原样（如 API、HTTP、JSON、变量名等）

**理由**：为中文用户提供一致的语言体验，降低理解成本。

### 二、最小实现原则

实现必须遵循最小可用实现（MVI），避免过度工程化：

- 只实现规格说明中明确要求的功能
- 不添加未明确要求的功能（即使"可能有用"）
- 避免不必要的抽象层和辅助函数
- 代码行数保持最少
- 避免为了"未来扩展"而添加当前不需要的代码

**理由**：保持代码简洁、易于维护，降低测试成本和 bug 引入风险。如有额外需求，应通过新的规格说明或任务提出。

### 三、本地优先原则

优先使用本地解决方案，减少外部依赖：

- 数据默认存储在本地（IndexedDB、浏览器存储）
- 避免内置遥测或数据收集
- 用户控制数据和端点
- 隐私和安全是默认行为，而非可选增强

**理由**：尊重用户隐私，减少对云服务的依赖，保持离线可用性。

### 四、供应商无关原则

代码不应假设特定供应商的专属行为：

- 使用抽象层（Provider 接口）处理供应商差异
- 新供应商实现遵循现有模式
- 避免在通用代码中硬编码供应商名称
- 遗留键（如 `ollama-*`）逐步迁移到通用命名

**理由**：简化供应商集成，降低新供应商的开发成本。

### 五、显式行为原则

系统行为应透明、可预测、可审计：

- 避免隐藏的网络调用或静默操作
- 错误消息清晰、可操作
- 用户可见所有数据流和配置
- 日志和状态明确

**理由**：用户应知道系统在做什么，便于调试和信任验证。

## 技术约束

### 技术栈

- **框架**: WXT (Chrome Extension MV3、Firefox Extension)
- **语言**: TypeScript 5.x, React 18+
- **存储**: Dexie IndexedDB (主要运行时存储), Plasmo Global Storage (配置和设置)
- **测试**: Vitest, happy-dom

### 浏览器限制

- **CSP**: 必须遵守内容安全策略，限制某些 WASM/worker ML 路径
- **Firefox**: 可能缺少某些 Chrome DNR API 行为
- **运行时**: 扩展上下文（sidepanel + background worker），不假设 Node.js API

### 数据持久化

- 主要存储：Dexie IndexedDB
- 辅助存储：SQLite 作为迁移路径存在，非主要运行时
- 版本迁移：Dexie 版本升级内嵌在 `src/lib/db.ts` 中

## Governance

### 修改流程

宪章修改必须：

1. 提出明确的修改原因和影响范围
2. 更新相关文档和模板
3. 保持向后兼容（除非是重大版本更新）
4. 遵循语义化版本控制（MAJOR.MINOR.PATCH）

### 版本控制

- **MAJOR**: 向后不兼容的治理原则删除或重新定义
- **MINOR**: 新原则/章节添加或实质性扩展指导
- **PATCH**: 澄清、措辞修正、非语义细化

### 合规审查

- 所有代码审查必须验证是否符合宪章
- 复杂度增加必须有正当理由
- 使用 `CLAUDE.md` 中的运行时开发指导

**Version**: 1.0.0 | **Ratified**: 2026-03-06 | **Last Amended**: 2026-03-06
