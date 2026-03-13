# Bug Report: WebChat 图片导致非视觉模型（hunter-alpha）卡死

**日期**：2026-03-14  
**报告者**：小虾 🦞  
**严重程度**：中高（导致对话完全卡住，需要 /reset 恢复）  
**影响范围**：WebChat 通道 + 非视觉模型（hunter-alpha 等 `input: ["text"]` 的模型）

---

## 问题描述

当用户通过 **WebChat** 上传图片后，如果后续切换到**不支持图片的模型**（如 hunter-alpha），模型在读取会话上下文时会卡住/失败。

**复现步骤**：
1. 在 WebChat 中上传一张图片
2. 使用 healer-alpha（支持图片）处理图片，得到回复
3. 发送新消息，切换到 hunter-alpha（不支持图片）
4. Hunter-alpha 尝试读取包含图片数据的会话历史 → **卡死**

---

## 根因分析

### 1. WebChat 与飞书的图片处理方式完全不同

#### WebChat 通道
- 图片在前端被转换为 **base64** 编码
- 通过 `chat.send` RPC 的 `attachments` 参数发送
- 网关调用 `parseMessageWithAttachments()` 解析：
  ```javascript
  // gateway-cli-Bmg642Lj.js L10987
  const parsed = await parseMessageWithAttachments(inboundMessage, normalizedAttachments, {
      maxBytes: 5e6,
      log: context.logGateway
  });
  ```
- 解析后的图片以 `{ type: "image", data: base64String, mimeType }` 格式嵌入用户消息的 content 数组
- **原始 base64 数据直接持久化到会话历史文件**

#### 飞书（Lark）通道
- 图片被**下载到本地文件系统**：
  ```javascript
  // openclaw-lark/src/messaging/inbound/media-resolver.js
  const result = await downloadMessageResourceFeishu({
      cfg, messageId, fileKey: res.fileKey, type: resourceType, accountId,
  });
  const saved = await core.channel.media.saveMediaBuffer(
      result.buffer, contentType, 'inbound', maxBytes, fileName
  );
  ```
- 上下文中只存储**占位符** `<media:image>`
- 文件路径存储在 `MediaPath`/`MediaUrl` 字段
- **不嵌入原始图片数据到消息内容**

### 2. 模型能力定义

```javascript
// auth-profiles-iXW75sRj.js L3001-3014
{
    id: "openrouter/hunter-alpha",
    name: "Hunter Alpha",
    reasoning: true,
    input: ["text"],          // ❌ 仅支持文本
    cost: OPENROUTER_DEFAULT_COST,
    contextWindow: 1048576,
    maxTokens: 65536
},
{
    id: "openrouter/healer-alpha",
    name: "Healer Alpha",
    reasoning: true,
    input: ["text", "image"], // ✅ 支持文本和图片
    cost: OPENROUTER_DEFAULT_COST,
    contextWindow: 262144,
    maxTokens: 65536
}
```

### 3. 图片清理机制存在但不完善

代码中有 `pruneProcessedHistoryImages()` 函数用于清理历史中的图片：

```javascript
// auth-profiles-iXW75sRj.js L103869-103896
const PRUNED_HISTORY_IMAGE_MARKER = "[image data removed - already processed by model]";

function pruneProcessedHistoryImages(messages) {
    let lastAssistantIndex = -1;
    for (let i = messages.length - 1; i >= 0; i--)
        if (messages[i]?.role === "assistant") {
            lastAssistantIndex = i;
            break;
        }
    if (lastAssistantIndex < 0) return false;
    let didMutate = false;
    for (let i = 0; i < lastAssistantIndex; i++) {  // ⚠️ 只清理最后助手回复之前的消息
        const message = messages[i];
        if (!message || message.role !== "user" && message.role !== "toolResult"
            || !Array.isArray(message.content)) continue;
        for (let j = 0; j < message.content.length; j++) {
            const block = message.content[j];
            if (!block || typeof block !== "object") continue;
            if (block.type !== "image") continue;
            message.content[j] = {
                type: "text",
                text: PRUNED_HISTORY_IMAGE_MARKER
            };
            didMutate = true;
        }
    }
    return didMutate;
}
```

**问题**：
- 该函数只清理 `lastAssistantIndex` **之前**的消息
- 在最后助手回复**之后**的用户消息中的图片**不会被清理**
- 原始 base64 数据已持久化到会话文件，重新加载时仍会存在

### 4. 非视觉模型的图片处理流程

当 hunter-alpha 运行时：

```javascript
// auth-profiles-iXW75sRj.js L104059-104077
function modelSupportsImages(model) {
    return model.input?.includes("image") ?? false;  // hunter-alpha 返回 false
}

async function detectAndLoadPromptImages(params) {
    if (!modelSupportsImages(params.model)) return {
        images: [],  // 返回空数组，不处理新图片
        // ...
    };
    // ... 检测并加载图片
}
```

虽然 `detectAndLoadPromptImages` 会为非视觉模型返回空图片数组，但这**不影响已存储在会话历史中的图片数据块**。当 API 调用发送包含 `{ type: "image", data: "..." }` 块的消息历史给 OpenRouter 时，hunter-alpha 模型无法处理这些内容，导致请求失败或卡死。

---

## 相关源码位置

| 文件 | 行号 | 说明 |
|------|------|------|
| `gateway-cli-Bmg642Lj.js` | 10987 | `chat.send` 处理器，解析附件 |
| `push-apns-D-8zMaiO.js` | 332 | `parseMessageWithAttachments`，图片解析 |
| `auth-profiles-iXW75sRj.js` | 3001-3014 | hunter/healer 模型定义 |
| `auth-profiles-iXW75sRj.js` | 103875 | `pruneProcessedHistoryImages` 函数 |
| `auth-profiles-iXW75sRj.js` | 104059 | `modelSupportsImages` 检查 |
| `auth-profiles-iXW75sRj.js` | 105534 | 图片清理调用点 |
| `auth-profiles-iXW75sRj.js` | 105580 | 图片传递给 `activeSession.prompt()` |
| `openclaw-lark/src/messaging/inbound/media-resolver.js` | 全文 | 飞书图片下载和存储 |

---

## 建议修复方案

### 方案 A（推荐）：WebChat 图片保存为本地文件

参考飞书通道的实现，将 WebChat 上传的图片：
1. 保存到本地文件系统（使用 `core.channel.media.saveMediaBuffer`）
2. 在用户消息 content 中存储**文件路径或占位符**，而非 base64 数据
3. 图片仅在调用视觉模型时加载，非视觉模型跳过

**优点**：与飞书通道行为一致，从根本上解决问题  
**缺点**：需要修改 WebChat 的图片处理流程

### 方案 B：增强图片清理机制

修改 `pruneProcessedHistoryImages`：
1. 不仅清理最后助手回复之前的消息，也清理之后的消息
2. 在模型切换时主动触发清理
3. 添加 `modelSupportsImages` 检查，非视觉模型自动过滤历史中的图片块

**优点**：改动较小  
**缺点**：治标不治本，base64 数据仍在会话文件中

### 方案 C：API 调用前过滤

在调用模型 API 之前，检查模型是否支持图片：
- 如果不支持，将历史消息中的 image 块替换为文本占位符
- 在 `runEmbeddedPiAgent` 或更底层的消息序列化处添加过滤

**优点**：集中处理，覆盖面广  
**缺点**：需要确保所有 API 路径都经过过滤

---

## 临时解决方案

1. **`/reset`**：清空会话上下文
2. **切换回视觉模型**：使用 `/model openrouter/healer-alpha`
3. **避免混合使用**：不要在同一个会话中交替使用视觉/非视觉模型

---

## 附加信息

- **测试环境**：WSL2 (Ubuntu), OpenClaw 最新版
- **通道**：webchat（通过 control-ui）
- **涉及模型**：openrouter/hunter-alpha, openrouter/healer-alpha
- **飞书通道表现正常**：同一用户在飞书发图片不会导致此问题
