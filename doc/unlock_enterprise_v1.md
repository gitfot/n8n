# n8n 功能解锁与 AI Builder 直连指南

本文档总结了如何通过修改源代码解锁 n8n 的企业版功能（Enterprise Features），并强制 AI Builder 跳过云服务直接通过环境变量调用第三方 AI 接口。

## 1. 源代码修改 (核心逻辑)

### A. 全局许可证解锁 (Unlock All Enterprise Features)
修改文件：`packages/cli/src/license.ts`
将 `isLicensed` 方法硬编码为返回 `true`。这会绕过所有企业版功能（共享、LDAP、SAML 等）的许可证校验。

```typescript
// packages/cli/src/license.ts
isLicensed(feature: BooleanLicenseFeature) {
    return true; 
}
```

### B. AI Builder UI 强制显化
修改文件：`packages/cli/src/services/frontend.service.ts`
强制将 `setup` 标记设为 `true`，确保前端 AI 按钮始终可见。

```typescript
// packages/cli/src/services/frontend.service.ts
this.settings.aiAssistant.setup = true;
this.settings.aiBuilder.setup = true;
```

### C. 绕过云服务 Token 校验 (Bypass Proxy Auth)
修改文件：`packages/cli/src/services/ai-workflow-builder.service.ts`
修改 `AiWorkflowBuilderService` 的初始化，当配置了本地 Key 时，禁止向 n8n 官方请求 Proxy Token。

```typescript
// packages/cli/src/services/ai-workflow-builder.service.ts
this.service = new AiWorkflowBuilderService(
    nodeTypeDescriptions,
    process.env.N8N_AI_ANTHROPIC_KEY ? undefined : this.client, // 如果有本地 Key 则不传 client
    // ...
);
```

### D. 支持自定义模型与协议切换
修改文件：`packages/@n8n/ai-workflow-builder.ee/src/llm-config.ts`
在 `anthropicClaudeSonnet45` 方法中添加了对环境变量 `N8N_AI_MODEL_NAME` 和 `N8N_AI_PROVIDER` 的支持，允许切换 `openai` 或 `anthropic` 协议。

### E. 屏蔽远端配额查询
修改文件：`packages/cli/src/controllers/ai.controller.ts`
将 `/ai/build/credits` 等接口拦截，直接返回无限额度数据，避免请求云端接口失败导致 UI 报错。

---

## 2. 必备环境变量配置

启动时请设置以下变量以实现直连：

| 变量名 | 推荐值 | 说明 |
| :--- | :--- | :--- |
| `N8N_AI_ENABLED` | `true` | 全局开启 AI 模块 |
| `N8N_AI_ANTHROPIC_KEY` | `your-api-key` | **必须**：设置后将自动切换为直连模式，跳过云端认证 |
| `N8N_AI_ASSISTANT_BASE_URL` | `https://api.xxx.com` | **必须**：你的 AI 接口地址 |
| `N8N_AI_MODEL_NAME` | `kimi-k2.5` | 自定义模型名称或接入点 ID |
| `N8N_AI_PROVIDER` | `anthropic` | 协议类型：`anthropic` (Claude) 或 `openai` |

### 路径补全说明：
*   **Anthropic 模式**: 自动补全 `/v1/messages`
*   **OpenAI 模式**: 自动补全 `/chat/completions`
请根据你的 API 供应商路径调整 `N8N_AI_ASSISTANT_BASE_URL`。

---

## 3. 编译流程

```bash
# 1. 安装依赖
pnpm install

# 2. 重新编译 (核心逻辑修改后必须 build)
pnpm build

# 3. 启动并注入环境变量
export N8N_AI_ENABLED=true
export N8N_AI_ASSISTANT_BASE_URL=https://...
export N8N_AI_ANTHROPIC_KEY=...
export N8N_AI_MODEL_NAME=...
export N8N_AI_PROVIDER=anthropic
pnpm start
```

---

## 4. 功能确认
1. 访问 `http://localhost:5678`。
2. 画布右下角应出现 **星星 (AI)** 图标。
3. 点击图标进入 **Build** 标签页，输入指令。
4. 观察后端日志 `[AI Builder] Initializing model...` 确认直连成功。
