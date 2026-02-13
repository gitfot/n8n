# AGENTS.md

本文件提供了在 n8n 仓库中开展工作的指导。

## Project Overview

n8n 是一个使用 TypeScript 编写的工作流自动化平台，采用由 pnpm
workspaces 管理的 monorepo 结构。它由 Node.js 后端、Vue.js 前端，以及可扩展的基于节点的工作流引擎组成。

## General Guidelines

- 始终使用 pnpm
- 我们使用 Linear 作为工单追踪系统
- 我们使用 Posthog 作为功能开关系统
- 开始处理新工单时，需从最新的 master 创建新分支，并使用 Linear 工单中指定的分支名
- 在 Linear 中为工单创建新分支时，使用 Linear 建议的分支名
- 在 MD 文件中需要可视化内容时，使用 mermaid 图

## Essential Commands

### Building
使用 `pnpm build` 构建所有包。务必将构建命令输出重定向到文件：

```bash
pnpm build > build.log 2>&1
```

你可以查看构建日志文件的最后几行来检查错误：
```bash
tail -n 20 build.log
```

### Testing
- `pnpm test` - 运行所有测试
- `pnpm test:affected` - 基于自上次提交以来的变更运行测试

运行某个特定测试文件时，需要进入该测试所在目录并执行：`pnpm test <test-file>`。

切换目录时，使用 `pushd` 进入目录，使用 `popd` 返回上一个目录。不确定时，使用 `pwd` 检查当前目录。

### Code Quality
- `pnpm lint` - 代码规范检查
- `pnpm typecheck` - 运行类型检查

提交代码前始终运行 lint 和 typecheck，以确保质量。
请在你当前工作的具体包目录内执行这些命令（例如：`cd packages/cli && pnpm lint`）。仅在准备最终 PR 时运行全仓库检查。
当你的变更影响到类型定义、`@n8n/api-types` 中的接口，或跨包依赖时，请先构建系统，再运行 lint 和 typecheck。

## Architecture Overview

**Monorepo 结构：** 使用 pnpm workspaces，并由 Turbo 协调构建

### Package Structure

该 monorepo 由以下核心包组成：

- **`packages/@n8n/api-types`**：前后端共享的 TypeScript 接口
- **`packages/workflow`**：核心工作流接口与类型
- **`packages/core`**：工作流执行引擎
- **`packages/cli`**：Express 服务器、REST API 和 CLI 命令
- **`packages/editor-ui`**：Vue 3 前端应用
- **`packages/@n8n/i18n`**：UI 文本的国际化支持
- **`packages/nodes-base`**：内置集成节点
- **`packages/@n8n/nodes-langchain`**：AI/LangChain 节点
- **`@n8n/design-system`**：用于 UI 一致性的 Vue 组件库
- **`@n8n/config`**：集中式配置管理

## Technology Stack

- **前端：** Vue 3 + TypeScript + Vite + Pinia + Storybook UI Library
- **后端：** Node.js + TypeScript + Express + TypeORM
- **测试：** Jest（单元）+ Playwright（E2E）
- **数据库：** TypeORM，支持 SQLite/PostgreSQL
- **代码质量：** Biome（格式化）+ ESLint + lefthook git hooks

### Key Architectural Patterns

1. **依赖注入**：使用 `@n8n/di` 作为 IoC 容器
2. **Controller-Service-Repository**：后端采用类 MVC 模式
3. **事件驱动**：使用内部事件总线实现解耦通信
4. **基于上下文的执行**：不同节点类型使用不同上下文
5. **状态管理**：前端使用 Pinia stores
6. **设计系统**：可复用组件与设计令牌集中在 `@n8n/design-system`，所有纯 Vue 组件应放在这里，以确保一致性和复用性

## Key Development Patterns

- 每个包都有独立构建配置，可独立开发
- 开发期间支持全栈热重载
- 节点开发使用专用的 `node-dev` CLI 工具
- 工作流测试基于 JSON，用于集成测试
- AI 功能有专门的开发流程（`pnpm dev:ai`）

### Workflow Traversal Utilities

`n8n-workflow` 包从 `packages/workflow/src/common/` 导出图遍历工具。
请使用这些工具，而不是自定义遍历逻辑。

**关键概念：** `workflow.connections` 按 **源节点（source node）** 建立索引。
如果要查找父节点，请先使用 `mapConnectionsByDestination()` 进行反转。

```typescript
import { getParentNodes, getChildNodes, mapConnectionsByDestination } from 'n8n-workflow';

// 查找父节点（前驱）- 需要先反转连接关系
const connectionsByDestination = mapConnectionsByDestination(workflow.connections);
const parents = getParentNodes(connectionsByDestination, 'NodeName', 'main', 1);

// 查找子节点（后继）- 直接使用 connections
const children = getChildNodes(workflow.connections, 'NodeName', 'main', 1);
```

### TypeScript Best Practices
- **绝不要使用 `any` 类型** - 请使用合适类型或 `unknown`
- **避免使用 `as` 进行类型断言** - 请改用类型守卫或类型谓词（测试代码中可使用 `as`）
- **在 `@n8n/api-types` 包中定义共享接口** - 用于前后端通信

### Error Handling
- 不要在 CLI 和节点中使用 `ApplicationError` 抛错，因为它已被弃用。
  请改用 `UnexpectedError`、`OperationalError` 或 `UserError`。
- 从各包中对应的错误类模块进行导入

### Frontend Development
- **所有 UI 文本都必须使用 i18n** - 在 `@n8n/i18n` 包中添加翻译
- **直接使用 CSS 变量** - 不要将间距硬编码为 px 值
- **`data-testid` 必须是单个值**（不能有空格或多个值）

实现 CSS 时，请参考 @packages/frontend/CLAUDE.md 中关于 CSS 变量和样式约定的指南。

### Testing Guidelines
- **运行测试时始终在对应包目录内执行**
- **在单元测试中 mock 所有外部依赖**
- **编写单元测试前先与用户确认测试用例**
- **提交前类型检查至关重要** - 始终运行 `pnpm typecheck`
- **修改 pinia store 时**，检查是否存在未使用的 computed 属性

我们使用的测试工具如下：
- 对节点及其他后端组件的测试，使用 Jest 进行单元测试。示例见 `packages/nodes-base/nodes/**/*test*`。
- 服务端 mock 使用 `nock`
- 前端测试使用 `vitest`
- E2E 测试使用 Playwright。运行命令：`pnpm --filter=n8n-playwright test:local`。
  详情见 `packages/testing/playwright/README.md`。

### Common Development Tasks

实现功能时：
1. 在 `packages/@n8n/api-types` 中定义 API 类型
2. 在 `packages/cli` 模块中实现后端逻辑，遵循
   `@packages/cli/scripts/backend-module/backend-module-guide.md`
3. 通过 controller 添加 API 端点
4. 在 `packages/editor-ui` 更新前端，并加入 i18n 支持
5. 使用合适的 mock 编写测试
6. 运行 `pnpm typecheck` 验证类型

## Github Guidelines
- 创建 PR 时，请遵循
  `.github/pull_request_template.md` 和
  `.github/pull_request_title_conventions.md` 中的规范。
- 使用 `gh pr create --draft` 创建草稿 PR。
- 在 PR 描述中始终引用对应的 Linear 工单，格式为
  `https://linear.app/n8n/issue/[TICKET-ID]`
- 如果 Linear 工单中提到了 GitHub issue，务必一并链接。
