# 数据流与请求链路

> 从用户点击"优化"按钮到 LLM 返回结果的全链路追踪，涵盖文本/图像两种模式、流式响应、Electron 代理和变量替换。

---

## 文本优化全链路

```mermaid
sequenceDiagram
    participant User as 👤 用户
    participant Comp as Vue 组件
    participant Store as Pinia Store
    participant PS as PromptService
    participant TM as TemplateManager
    participant LS as LLMService
    participant MM as ModelManager
    participant Reg as TextAdapterRegistry
    participant Ad as Adapter (e.g. DeepSeek)
    participant API as DeepSeek API

    User->>Comp: 点击"优化"
    Comp->>Store: optimize()
    Store->>PS: optimizePrompt(request)

    Note over PS: validateInput(prompt, modelKey)

    PS->>MM: getModel(modelKey)
    MM-->>PS: TextModelConfig

    PS->>TM: getTemplate(templateId)
    TM-->>PS: TemplateProcessor

    Note over PS: 渲染模板<br/>填充 system/user prompt<br/>替换变量 {{var}}

    PS->>LS: sendMessageStructured(messages, provider)
    LS->>MM: getModel(provider)
    MM-->>LS: TextModelConfig

    LS->>LS: validateModelConfig(config)<br/>validateMessages(messages)

    LS->>Reg: getAdapter(providerMeta.id)
    Reg-->>LS: Adapter instance

    LS->>Ad: sendMessage(messages, config)
    Ad->>Ad: validateMessages<br/>validateConfig

    Ad->>API: HTTP POST /v1/chat/completions
    API-->>Ad: { choices: [{ message: { content: "..." } }] }

    Ad->>Ad: processThinkTags(content)

    Ad-->>LS: LLMResponse { content }
    LS-->>PS: response.content

    PS->>PS: validateResponse(response)
    PS->>PS: 记录历史 (HistoryManager)

    PS-->>Store: 优化后的提示词
    Store-->>Comp: 更新视图
    Comp-->>User: 显示结果
```

---

## 流式响应处理

```mermaid
sequenceDiagram
    participant Store as Pinia Store
    participant LS as LLMService
    participant Ad as Adapter
    participant API as LLM API

    Store->>LS: sendMessageStream(messages, callbacks)
    LS->>Ad: sendMessageStream(messages, config, callbacks)

    Ad->>API: HTTP POST (stream: true)

    loop SSE 流式返回
        API-->>Ad: data: {"choices":[{"delta":{"content":"token"}}]}
        Ad->>Ad: 解析 SSE chunk
        Ad->>LS: callbacks.onToken(token)
        LS->>Store: callbacks.onToken(token)
        Note over Store: 逐个追加 token 到显示内容
    end

    API-->>Ad: data: [DONE]
    Ad->>LS: callbacks.onComplete(fullContent)
    LS->>Store: callbacks.onComplete(fullContent)
    Note over Store: 标记完成，启用后续操作
```

### StreamHandlers 接口

```typescript
interface StreamHandlers {
  onToken?: (token: string) => void;         // 每个 token 到达时
  onComplete?: (fullContent: string) => void; // 流结束
  onError?: (error: Error) => void;          // 错误处理
  onThinkingStart?: () => void;              // thinking 开始（DeepSeek/Grok）
  onThinkingToken?: (token: string) => void; // thinking token
  onThinkingEnd?: () => void;                // thinking 结束
}
```

---

## 图像生成链路

```mermaid
sequenceDiagram
    participant User as 👤 用户
    participant Store as Pinia Store
    participant IS as ImageService
    participant Reg as ImageAdapterRegistry
    participant Ad as ImageAdapter (e.g. OpenAI)
    participant API as Image API

    User->>Store: 点击"生成图片"
    Store->>IS: generateImage(request, config)

    IS->>Reg: getAdapter(providerId)
    Reg-->>IS: ImageAdapter

    IS->>Ad: generate(request, config)
    Ad->>Ad: validateRequest(request, config)

    Note over Ad: 规范化输入<br/>- 图片 resize/base64 转换<br/>- 参数映射

    Ad->>API: HTTP POST /v1/images/generations
    API-->>Ad: { data: [{ url: "...", b64_json: "..." }] }

    Ad-->>IS: ImageResult { images: [...] }
    IS->>IS: 存储图片到 ImageAssetStorage

    IS-->>Store: ImageResult
    Store-->>User: 显示生成的图片
```

### 图像适配器继承结构

```
AbstractImageProviderAdapter (抽象基类)
  ├── OpenAIAdapter       (13KB)
  ├── SiliconflowAdapter  (14KB)
  ├── DashScopeAdapter    (13KB)
  ├── SeedreamAdapter     (12KB)
  ├── GrokAdapter         (11KB)
  ├── CloudflareAdapter   (8KB)
  ├── ModelScopeAdapter   (8KB)
  ├── OpenRouterAdapter   (8KB)
  ├── GeminiAdapter       (8KB)
  └── OllamaAdapter       (7KB)
```

---

## Electron 代理层

在桌面端，渲染进程不能直接发送 HTTP 请求（受 CORS/安全策略限制）。项目通过 IPC 机制将请求转发到主进程：

```mermaid
sequenceDiagram
    participant Renderer as 渲染进程 (UI)
    participant Proxy as ElectronLLMProxy<br/>(core/src/services/llm/electron-proxy.ts)
    participant Main as 主进程
    participant API as LLM API

    Note over Renderer: isRunningInElectron() === true

    Renderer->>Proxy: sendMessage(messages, config)
    Proxy->>Main: ipcRenderer.invoke("llm:sendMessage", { messages, config })
    Main->>API: HTTP Request (Node.js undici)
    API-->>Main: Response
    Main-->>Proxy: ipcRenderer response
    Proxy-->>Renderer: LLMResponse
```

同样的代理模式也应用于：
- `ImageService` → `electron-proxy.ts`
- `PromptService` → `electron-proxy.ts`
- `FavoriteManager` → `electron-proxy.ts`
- `TemplateManager` → `electron-proxy.ts` (语言检测)
- `ModelManager` → `electron-proxy.ts` (配置读写)

### 代理选择逻辑

```typescript
// LLMService 中的判断
if (isRunningInElectron()) {
  const proxy = new ElectronLLMProxy();
  return proxy.sendMessage(messages, config);  // 走 IPC
} else {
  const adapter = registry.getAdapter(provider.id);
  return adapter.sendMessage(messages, config); // 直接请求
}
```

---

## 变量替换引擎

项目使用 **Mustache** 模板引擎处理提示词中的变量：

```
原始提示词：
"你是一个{{role}}，请用{{tone}}的语气回复用户关于{{topic}}的问题。"

变量值：
{ role: "技术顾问", tone: "专业", topic: "React 性能优化" }

渲染结果：
"你是一个技术顾问，请用专业的语气回复用户关于React 性能优化的问题。"
```

### Smart Fill（智能填充）

`packages/core/src/services/variable-value-generation/` 提供 AI 驱动的变量值自动生成：

```
用户输入："role=律师, 帮我生成 tone 和 topic 的值"
  → LLM 调用 → { tone: "严谨专业", topic: "合同纠纷", ... }
```

### 变量提取

`packages/core/src/services/variable-extraction/` 从提示词中自动识别 `{{variable}}` 模式并提取变量列表，用于构建变量配置界面。

---

## 前端数据持久化流程

```mermaid
sequenceDiagram
    participant User as 用户操作
    participant Store as Pinia Store
    participant Service as 领域服务
    participant Provider as StorageProvider
    participant DB as IndexedDB / 文件系统

    User->>Store: 保存模型配置
    Store->>Service: ModelManager.saveModel(config)
    Service->>Service: 序列化 + 验证

    Service->>Provider: set("model_configs", config)
    Provider->>DB: db.modelConfigs.put(record)

    Note over DB: Dexie: 自动索引<br/>File: JSON 写入

    DB-->>Provider: 写入成功
    Provider-->>Service: void
    Service-->>Store: 更新完成
    Store-->>User: UI 确认

    Note over User,DB: --- 下次启动 ---

    User->>Store: 应用启动
    Store->>Service: ModelManager.loadModels()
    Service->>Provider: get("model_configs")
    Provider->>DB: db.modelConfigs.toArray()
    DB-->>Provider: ModelConfigRecord[]
    Provider-->>Service: 反序列化
    Service-->>Store: ModelConfig[]
    Store-->>User: 恢复上次的模型列表
```

持久化遵循 **写时序列化、读时反序列化** 原则，`ModelManager.converter` 模块负责存储格式与运行时格式互转。
