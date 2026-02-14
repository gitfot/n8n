# n8n 企业版功能解锁 & AI Builder 直连指南 (v2)

> 通过 5 处源码修改 + 环境变量配置，实现：
> 1. 解锁全部企业版功能（无需 License Key）
> 2. AI Builder 跳过 n8n 云端代理，直连你自己的 AI 接口

---

## 修改总览

| # | 目标 | 文件 | 一句话说明 |
|---|------|------|-----------|
| A | 解锁企业版 | `packages/cli/src/license.ts` | `isLicensed()` 永远返回 `true` |
| B | 显示 AI 按钮 | `packages/cli/src/services/frontend.service.ts` | 强制 `aiAssistant.setup` 和 `aiBuilder.setup` 为 `true` |
| C | 跳过云端认证 | `packages/cli/src/services/ai-workflow-builder.service.ts` | 有本地 Key 时不传 `client`，阻止向 n8n 官方请求 Proxy Token |
| D | 自定义模型/协议 | `packages/@n8n/ai-workflow-builder.ee/src/llm-config.ts` | 读取环境变量，支持 Anthropic / OpenAI 两种协议 |
| E | 屏蔽配额查询 | `packages/cli/src/controllers/ai.controller.ts` | `/ai/build/credits` 直接返回无限额度，不再请求云端 |

---

## A. 解锁全部企业版功能

`packages/cli/src/license.ts` — `License` 类的 `isLicensed` 方法

n8n 所有企业功能（共享、LDAP、SAML、AI Builder 等）都通过 `isLicensed(feature)` 做许可证校验。将其硬编码为 `true` 即可全部绕过。

```diff
 isLicensed(feature: BooleanLicenseFeature) {
-    return this.manager?.hasFeatureEnabled(feature) ?? false;
+    // return this.manager?.hasFeatureEnabled(feature) ?? false;
+    return true;
 }
```

**原理**：`this.manager` 是 `@n8n_io/license-sdk` 的实例，会向 n8n 许可证服务器验证。直接返回 `true` 跳过整个验证链路。

---

## B. 强制显示 AI 按钮

`packages/cli/src/services/frontend.service.ts` — `FrontendService` 的设置初始化

前端是否显示 AI 功能入口，取决于后端下发的 `settings.aiAssistant.setup` 和 `settings.aiBuilder.setup`。原逻辑需要配置了 `aiAssistant.baseUrl` 或 `aiBuilder.apiKey` 才为 `true`。强制设为 `true` 确保按钮始终可见。

```diff
 if (isAiAssistantEnabled) {
     this.settings.aiAssistant.enabled = isAiAssistantEnabled;
-    this.settings.aiAssistant.setup =
-        !!this.globalConfig.aiAssistant.baseUrl || !!process.env.N8N_AI_ANTHROPIC_KEY;
+    // this.settings.aiAssistant.setup =
+    //     !!this.globalConfig.aiAssistant.baseUrl || !!process.env.N8N_AI_ANTHROPIC_KEY;
+    this.settings.aiAssistant.setup = true;
 }

 // ... (中间省略 askAi / aiCredits 部分)

 if (isAiBuilderEnabled) {
     this.settings.aiBuilder.enabled = isAiBuilderEnabled;
-    this.settings.aiBuilder.setup =
-        !!this.globalConfig.aiAssistant.baseUrl || !!this.globalConfig.aiBuilder.apiKey;
+    // this.settings.aiBuilder.setup =
+    //     !!this.globalConfig.aiAssistant.baseUrl || !!this.globalConfig.aiBuilder.apiKey;
+    this.settings.aiBuilder.setup = true;
 }
```

**注意**：`enabled` 由许可证控制（已在步骤 A 解锁），`setup` 由配置控制（此处强制）。两者都为 `true` 前端才会渲染 AI 入口。

---

## C. 跳过云端 Proxy 认证

`packages/cli/src/services/ai-workflow-builder.service.ts` — `WorkflowBuilderService.init()`

`AiWorkflowBuilderService` 构造函数的第二个参数是 `client`（n8n 云端 Proxy 客户端）。传入 `client` 时，AI 请求会走 n8n 官方代理（需要有效订阅）。传入 `undefined` 时，会走本地环境变量配置的直连模式。

```diff
 this.service = new AiWorkflowBuilderService(
     nodeTypeDescriptions,
-    this.client,
+    process.env.N8N_AI_ANTHROPIC_KEY ? undefined : this.client,
     this.logger,
     // ...
 );
```

**调用链**：
1. `client` 存在 → `setupModels()` 走 `this.client.getApiProxyBaseUrl()` → 请求 n8n 官方代理
2. `client` 为 `undefined` → `setupModels()` 走 `else` 分支 → 读取 `process.env.N8N_AI_ANTHROPIC_KEY` 直连

---

## D. 支持自定义模型与协议切换

`packages/@n8n/ai-workflow-builder.ee/src/llm-config.ts` — `anthropicClaudeSonnet45` 和 `anthropicClaudeSonnet45Think`

原代码硬编码使用 Anthropic 协议 + `claude-sonnet-4-5-20250929` 模型。修改后读取环境变量，支持切换为 OpenAI 兼容协议和自定义模型名。

两个函数（普通模式 / Thinking 模式）都做了相同修改，逻辑如下：

```typescript
const customModel = process.env.N8N_AI_MODEL_NAME;
const provider = process.env.N8N_AI_PROVIDER ?? 'anthropic';     // 默认 anthropic
const customBaseUrl = process.env.N8N_AI_ASSISTANT_BASE_URL;

if (provider === 'openai') {
    // 使用 ChatOpenAI（OpenAI 兼容协议）
    const openaiBaseUrl = customBaseUrl?.replace(/\/+$/, '') ?? config.baseUrl;
    return new ChatOpenAI({
        model: customModel ?? 'claude-sonnet-4-5-20250929',
        apiKey: config.apiKey,
        temperature: 0,
        maxTokens: -1,
        configuration: {
            baseURL: openaiBaseUrl,
            // ...
        },
    });
}

// 默认：使用 ChatAnthropic（Anthropic 原生协议）
const baseUrl = customBaseUrl?.replace(/\/+$/, '') ?? config.baseUrl;
const model = new ChatAnthropic({
    model: customModel ?? 'claude-sonnet-4-5-20250929',
    apiKey: config.apiKey,
    anthropicApiUrl: baseUrl,
    // ...
});
```

**关键细节**：
- `customBaseUrl` 会自动去除末尾斜杠（`replace(/\/+$/, '')`）
- Anthropic 模式下 SDK 自动补全 `/v1/messages`
- OpenAI 模式下 SDK 自动补全 `/chat/completions`
- Thinking 模式在 OpenAI 协议下退化为普通模式（OpenAI 协议不支持 `thinking` 参数）

---

## E. 屏蔽远端配额查询

`packages/cli/src/controllers/ai.controller.ts` — `AiController.getBuilderCredits()`

前端会调用 `GET /ai/build/credits` 查询 AI Builder 剩余额度。原逻辑请求 n8n 云端，直连模式下必然失败。改为直接返回"无限额度"。

```diff
 @Get('/build/credits')
 async getBuilderCredits(
-    req: AuthenticatedRequest,
-    _: Response,
+    _req: AuthenticatedRequest,
+    _res: Response,
 ): Promise<AiAssistantSDK.BuilderInstanceCreditsResponse> {
-    try {
-        return await this.workflowBuilderService.getBuilderInstanceCredits(req.user);
-    } catch (e) {
-        assert(e instanceof Error);
-        throw new InternalServerError(e.message, e);
-    }
+    return {
+        creditsQuota: -1,      // -1 表示无限
+        creditsClaimed: 0,
+    };
 }
```

---

## 环境变量

| 变量 | 必填 | 默认值 | 说明 |
|------|:----:|--------|------|
| `N8N_AI_ENABLED` | 是 | — | 全局开启 AI 模块，不设则 AI 功能完全不加载 |
| `N8N_AI_ANTHROPIC_KEY` | 是 | — | 你的 API Key。**同时作为直连模式的开关**：有值 → 跳过云端代理（见步骤 C） |
| `N8N_AI_ASSISTANT_BASE_URL` | 是 | — | AI 接口基础地址，如 `https://api.anthropic.com`。末尾斜杠会自动去除 |
| `N8N_AI_MODEL_NAME` | 否 | `claude-sonnet-4-5-20250929` | 自定义模型名称 |
| `N8N_AI_PROVIDER` | 否 | `anthropic` | 协议类型：`anthropic` 或 `openai` |

### Base URL 配置示例

```bash
# 直连 Anthropic 官方
N8N_AI_ASSISTANT_BASE_URL=https://api.anthropic.com
N8N_AI_PROVIDER=anthropic

# 通过 OpenAI 兼容接口（如 OpenRouter、vLLM、one-api 等）
N8N_AI_ASSISTANT_BASE_URL=https://openrouter.ai/api/v1
N8N_AI_PROVIDER=openai
N8N_AI_MODEL_NAME=anthropic/claude-sonnet-4-5

# 通过 Anthropic 兼容的中转服务
N8N_AI_ASSISTANT_BASE_URL=https://your-proxy.com
N8N_AI_PROVIDER=anthropic
N8N_AI_MODEL_NAME=claude-sonnet-4-5-20250929
```

> **路径补全规则**：Anthropic SDK 自动追加 `/v1/messages`，OpenAI SDK 自动追加 `/chat/completions`。
> 所以 `N8N_AI_ASSISTANT_BASE_URL` 只需填到域名或 `/v1` 级别，不要包含具体端点路径。

---

## 编译与启动

```bash
# 1. 安装依赖
pnpm install

# 2. 编译（修改源码后必须重新 build）
pnpm build

# 3. 设置环境变量并启动
export N8N_AI_ENABLED=true
export N8N_AI_ANTHROPIC_KEY=sk-xxx
export N8N_AI_ASSISTANT_BASE_URL=https://api.anthropic.com
export N8N_AI_PROVIDER=anthropic
# export N8N_AI_MODEL_NAME=claude-sonnet-4-5-20250929  # 可选

pnpm start
```

---

## 验证

1. 打开 `http://localhost:5678`
2. 画布右下角应出现 AI 星星图标
3. 点击进入 Build 标签页，输入任意指令
4. 后端日志应出现 `[AI Builder] Initializing model...`，确认直连成功
5. 如果使用了自定义模型，日志中应显示对应的模型名称

### 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| AI 按钮不显示 | 步骤 A 或 B 未生效 | 确认 `pnpm build` 成功，检查 `license.ts` 和 `frontend.service.ts` |
| 点击 Build 报 "credits" 错误 | 步骤 E 未生效 | 检查 `ai.controller.ts` 是否已修改 |
| 请求发往 n8n 官方 | `N8N_AI_ANTHROPIC_KEY` 未设置 | 步骤 C 依赖此变量判断是否跳过云端 |
| 模型调用 404 | Base URL 或 Provider 不匹配 | 检查 URL 是否多了 `/v1/messages` 等后缀 |
| Thinking 模式无效 | 使用了 `openai` provider | OpenAI 协议不支持 thinking 参数，属于预期行为 |
