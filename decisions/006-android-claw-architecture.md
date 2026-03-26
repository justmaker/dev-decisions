# Android Claw — 系統架構文件

- **日期**: 2026-03-26
- **狀態**: Draft v1
- **相關決策**: [DEC-006](006-android-claw-feasibility.md)

---

## 設計原則

1. **Phone-native, not server-lite** — 不是把 server 塞進手機，而是為手機重新設計 AI agent
2. **Provider-agnostic** — 使用者帶自己的 key，接任何 OpenAI-compatible / Anthropic / Google API
3. **Offline-capable** — 核心功能離線可用（本地 LLM + 本地 tool）
4. **Progressive complexity** — C 類使用者開箱即用，B 類使用者可深度設定
5. **Standard protocols** — MCP、OpenAI function calling、標準 API format，不發明自己的協議

---

## 系統堆疊總覽

```
┌─────────────────────────────────────────────────────────────────┐
│                         UI Layer                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ Chat UI  │ │ Files UI │ │ Settings │ │ Skill Store / Mgr │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Session Layer                               │
│  ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │ Conversation Mgr │ │ Context Mgr  │ │ Memory / Bond    │    │
│  │ (multi-session)  │ │ (budget,hist)│ │ (persistent)     │    │
│  └──────────────────┘ └──────────────┘ └──────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                      Brain Layer                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Provider Router                          │   │
│  │  ┌────────────┐ ┌──────────┐ ┌────────┐ ┌────────────┐  │   │
│  │  │ OpenAI-    │ │Anthropic │ │ Google │ │ Local LLM  │  │   │
│  │  │ Compatible │ │ Messages │ │ GenAI  │ │ (llama.cpp)│  │   │
│  │  │ (GHC,OR,..)│ │ (Claude) │ │(Gemini)│ │            │  │   │
│  │  └────────────┘ └──────────┘ └────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Action Layer                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐    │
│  │ Tool Engine  │ │ MCP Client   │ │ Skill Engine         │    │
│  │ (native)     │ │ (SSE/stdio)  │ │ (structured prompts) │    │
│  └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘    │
│         │                │                     │                 │
│  ┌──────┴────────────────┴─────────────────────┴───────────┐    │
│  │              Unified Tool Registry                       │    │
│  │  native tools + MCP tools + skill-defined tools          │    │
│  └──────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                     Security Layer                               │
│  ┌──────────┐ ┌───────────┐ ┌───────────┐ ┌────────────────┐   │
│  │ Perm     │ │ Path      │ │ Rate      │ │ Audit Log      │   │
│  │ Guard    │ │ Sandbox   │ │ Limiter   │ │ (all actions)  │   │
│  └──────────┘ └───────────┘ └───────────┘ └────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                   Platform Layer (Android)                        │
│  ┌────────┐ ┌──────┐ ┌─────┐ ┌──────┐ ┌───────┐ ┌──────────┐  │
│  │ Files  │ │Camera│ │ GPS │ │ Mic  │ │ Share │ │Biometric │  │
│  │(SAF/MS)│ │      │ │     │ │(STT) │ │Intent │ │          │  │
│  └────────┘ └──────┘ └─────┘ └──────┘ └───────┘ └──────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Platform Layer（Android 能力抽象）

這層把 Android 特有的能力包裝成統一介面，上層不需要知道 Android API 的細節。

### 1.1 檔案系統 (FileSystemProvider)

Android 的檔案存取是最複雜的部分，有三種機制要處理：

```
┌─────────────────────────────────────────────┐
│            FileSystemProvider                 │
│                                              │
│  ┌─────────────┐  App 私有空間               │
│  │ Internal    │  /data/data/app/files/      │
│  │ Storage     │  → 不需要權限，但其他 app     │
│  │             │    看不到                     │
│  └─────────────┘                              │
│                                              │
│  ┌─────────────┐  共享空間（需權限）           │
│  │ MediaStore  │  → 圖片存相簿、文件存下載      │
│  │ API         │  → Android 10+ 推薦方式      │
│  │             │  → 不需 MANAGE_EXTERNAL      │
│  └─────────────┘                              │
│                                              │
│  ┌─────────────┐  完整存取（需特殊權限）       │
│  │ SAF /       │  → Storage Access Framework  │
│  │ MANAGE_     │  → 或 MANAGE_EXTERNAL_STORAGE│
│  │ EXTERNAL    │  → B 類使用者可選開啟         │
│  └─────────────┘                              │
└─────────────────────────────────────────────┘
```

**策略：分層權限**
- **預設（C 類）**：MediaStore API — 圖片自動出現在相簿、文件自動出現在「檔案」app、不需要可怕的權限彈窗
- **進階（B 類）**：SAF 選擇特定目錄授權，或 MANAGE_EXTERNAL_STORAGE 全開
- **App 私有**：Skill 設定檔、memory、cache 等系統檔案放這裡

**支援的檔案類型：**

| 類型 | 讀 | 寫 | 實作 |
|------|---|---|------|
| 文字（.txt .md .json .csv） | ✅ | ✅ | 直接讀寫 |
| 圖片（.jpg .png .webp） | ✅ | ✅ | MediaStore + BitmapFactory |
| 影片（.mp4 .webm） | ✅ | ✅ | MediaStore |
| 音檔（.mp3 .wav .ogg） | ✅ | ✅ | MediaStore |
| 程式碼（.kt .py .js 等） | ✅ | ✅ | 存到 Documents/ 或 app 私有 |
| 壓縮檔（.zip） | ✅ | ✅ | java.util.zip |

### 1.2 感測器與硬體

```kotlin
// 統一介面，每個感測器是一個 Capability
interface DeviceCapability {
    val id: String
    val isAvailable: Boolean
    suspend fun activate(params: Map<String, Any>): CapabilityResult
}

// 實作
class CameraCapability : DeviceCapability   // 拍照、錄影
class LocationCapability : DeviceCapability // GPS 定位
class MicrophoneCapability : DeviceCapability // 錄音、STT
class BiometricCapability : DeviceCapability  // 指紋/臉部確認
class ShareCapability : DeviceCapability      // 接收其他 app 的分享
class NotificationCapability : DeviceCapability // 推播
class ClipboardCapability : DeviceCapability  // 剪貼簿
```

### 1.3 背景執行策略

這是 Android 上最大的技術挑戰。

```
┌──────────────────────────────────────────────┐
│          Background Execution Strategy        │
│                                              │
│  前景操作（使用者正在用 app）                  │
│  → 正常執行，沒有限制                         │
│                                              │
│  短期背景（< 10 分鐘）                        │
│  → WorkManager (OneTimeWorkRequest)          │
│  → 適合：API 呼叫、檔案處理                   │
│                                              │
│  持續背景（使用者想讓 agent 跑一段時間）        │
│  → Foreground Service + 常駐通知              │
│  → 使用者明確啟動「Agent 模式」               │
│  → 通知顯示：「🦞 Android Claw 運行中」       │
│                                              │
│  定期任務                                     │
│  → WorkManager (PeriodicWorkRequest)         │
│  → 最小間隔 15 分鐘（Android 限制）           │
│  → 適合：cron-like 排程                       │
│                                              │
│  ❌ 不做的事                                  │
│  → 不嘗試繞過 Doze mode                      │
│  → 不用 AlarmManager exact alarm（會被 OEM 殺）│
│  → 不假裝永遠在背景跑                         │
└──────────────────────────────────────────────┘
```

**設計哲學**：接受 Android 的限制，而不是跟它對抗。Agent 在前景時全力運作，背景時做有限的事，而不是假裝是 server。

---

## Layer 2: Security Layer

參考 PocketClaw 的 4 層設計，但加上 B/C 類使用者的區分：

```
┌─────────────────────────────────────────────┐
│              Security Model                  │
│                                              │
│  ┌─────────────────────────────────┐         │
│  │ PermissionGuard                 │         │
│  │                                 │         │
│  │ L0 (Auto)    讀取、搜尋、分析    │         │
│  │ L1 (Grant)   寫檔案、啟動 app   │         │
│  │ L2 (Confirm) 發訊息、刪檔案     │         │
│  │ L3 (Biometric) 付款、敏感操作   │         │
│  └─────────────────────────────────┘         │
│                                              │
│  ┌─────────────────────────────────┐         │
│  │ Trust Profiles                  │         │
│  │                                 │         │
│  │ 🟢 Casual (C類預設)             │         │
│  │   所有 L1+ 操作都要確認          │         │
│  │                                 │         │
│  │ 🟡 Standard (B類預設)           │         │
│  │   L1 首次授權後自動，L2+ 確認    │         │
│  │                                 │         │
│  │ 🔴 Power (B類手動開啟)          │         │
│  │   L0-L1 自動，L2 確認，L3 生物   │         │
│  └─────────────────────────────────┘         │
│                                              │
│  PathSandbox  — 可存取路徑白名單              │
│  RateLimiter  — 每回合/每分鐘 tool 呼叫上限   │
│  AuditLog     — 所有 tool 操作可檢視          │
└─────────────────────────────────────────────┘
```

---

## Layer 3: Action Layer（Tool + MCP + Skill）

這是核心差異化所在。

### 3.1 Unified Tool Registry

所有 tool 不管來源（native / MCP / skill-defined），都統一註冊到同一個 registry：

```
┌────────────────────────────────────────────────────┐
│              Unified Tool Registry                  │
│                                                    │
│  Tool 來源 A: Native Tools（內建）                  │
│  ┌─────────┬──────────┬───────────┬──────────┐     │
│  │file_read│file_write│file_list  │file_delete│     │
│  │web_fetch│web_search│clipboard  │app_launch │     │
│  │camera   │location  │mic_record │share      │     │
│  │schedule │notify    │screenshot │...        │     │
│  └─────────┴──────────┴───────────┴──────────┘     │
│                                                    │
│  Tool 來源 B: MCP Server 動態工具                   │
│  ┌──────────────────────────────────────────┐      │
│  │ 從 MCP server 的 tools/list 動態載入      │      │
│  │ 例：mcp-atlassian → jira_search,         │      │
│  │      confluence_read, ...                 │      │
│  │ 例：mcp-sauron → gitlab_mr_create, ...   │      │
│  └──────────────────────────────────────────┘      │
│                                                    │
│  Tool 來源 C: Skill 定義的虛擬工具                  │
│  ┌──────────────────────────────────────────┐      │
│  │ Skill 可以定義 workflow = 多個 tool 組合   │      │
│  │ 例：「產品分析」skill = web_search         │      │
│  │      → file_write(report.md)              │      │
│  │      → notify(完成)                        │      │
│  └──────────────────────────────────────────┘      │
│                                                    │
│  統一介面：                                         │
│  ┌──────────────────────────────────────────┐      │
│  │ interface Tool {                          │      │
│  │   id: String                              │      │
│  │   name: String                            │      │
│  │   description: String                     │      │
│  │   parameters: JsonSchema                  │      │
│  │   riskLevel: RiskLevel                    │      │
│  │   source: ToolSource (NATIVE|MCP|SKILL)   │      │
│  │   execute(args: JsonObject): ToolResult   │      │
│  │ }                                         │      │
│  └──────────────────────────────────────────┘      │
└────────────────────────────────────────────────────┘
```

**關鍵設計：使用標準 function calling**

不用 PocketClaw 的 `[T:tool_id:args]` 自定義格式。直接用 OpenAI function calling 標準：

```json
{
  "type": "function",
  "function": {
    "name": "file_write",
    "description": "寫入檔案到指定路徑",
    "parameters": {
      "type": "object",
      "properties": {
        "path": { "type": "string" },
        "content": { "type": "string" }
      },
      "required": ["path", "content"]
    }
  }
}
```

每個 Provider adapter 負責把標準 tool schema 轉成自己的格式（OpenAI 原生支援、Anthropic 有 tool_use、Google 有 function_declarations）。LLM 回傳的 tool call 也統一解析。

### 3.2 MCP Client on Android

這是技術挑戰最大的部分。

```
┌──────────────────────────────────────────────────┐
│           MCP Client Architecture                 │
│                                                  │
│  Transport 層：                                   │
│                                                  │
│  ┌──────────────────┐  首選：HTTP+SSE             │
│  │ SSE Transport    │  → 連到遠端 MCP server      │
│  │ (Remote)         │  → 支援 mcp-atlassian 等    │
│  │                  │  → 最自然的 Android 方式     │
│  └──────────────────┘                             │
│                                                  │
│  ┌──────────────────┐  進階：本地 process          │
│  │ Embedded Runtime │  → WebView/QuickJS 跑       │
│  │ (Local)          │    輕量 MCP server           │
│  │                  │  → 不需要 Node.js            │
│  │                  │  → 限制：只有純 JS 的 server  │
│  └──────────────────┘                             │
│                                                  │
│  ┌──────────────────┐  B 類：Termux bridge        │
│  │ Termux Bridge    │  → 如果使用者有裝 Termux     │
│  │ (Power User)     │  → 可以跑完整的 Node.js MCP  │
│  │                  │  → 透過 localhost 通訊       │
│  └──────────────────┘                             │
│                                                  │
│  Protocol 層：                                    │
│  ┌──────────────────────────────────────────┐     │
│  │ MCP JSON-RPC Handler                     │     │
│  │ - initialize / tools/list / tools/call   │     │
│  │ - resources/list / resources/read        │     │
│  │ - prompts/list / prompts/get             │     │
│  └──────────────────────────────────────────┘     │
│                                                  │
│  Integration：                                    │
│  ┌──────────────────────────────────────────┐     │
│  │ MCP tools 自動註冊到 Unified Tool Registry│     │
│  │ MCP resources 可被 Context Mgr 引用       │     │
│  │ MCP prompts 可被 Skill Engine 使用        │     │
│  └──────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

**MCP 在 Android 上的現實限制：**
- stdio transport 需要本地 process → Android 上不容易跑 Node.js
- **SSE transport 是唯一適合 Android 的方式**（像呼叫 REST API 一樣）
- 大部分 MCP server 還是 stdio-only → 需要一個 **MCP Gateway**（server 端的 stdio→SSE proxy）
- 或者：使用者的 server/NAS 上跑 MCP server，手機端透過 SSE 連過去

**可行的 MCP 策略：**

| MCP Server 位置 | Transport | 適合 |
|----------------|-----------|------|
| 雲端（Cloudflare Workers 等） | SSE | C 類 — 預設方式 |
| 使用者的 server/NAS | SSE（需 proxy） | B 類 — 連自己的基礎設施 |
| 手機本地（QuickJS 跑純 JS server） | 嵌入式 | 離線場景 |
| Termux（完整 Node.js） | localhost SSE | Power User |

### 3.3 Skill Engine

```
┌──────────────────────────────────────────────────┐
│               Skill Engine                        │
│                                                  │
│  Skill = 結構化的知識 + 工具組合 + 行為指引         │
│                                                  │
│  ┌──────────────────────────────────────────┐     │
│  │ skill.json                               │     │
│  │ {                                        │     │
│  │   "name": "product-analysis",            │     │
│  │   "description": "產品規劃與市場調查",     │     │
│  │   "tools_required": ["web_search",       │     │
│  │     "file_write", "web_fetch"],           │     │
│  │   "system_prompt": "你是產品分析師...",    │     │
│  │   "steps": [                              │     │
│  │     "搜尋目標市場資料",                    │     │
│  │     "分析競品",                            │     │
│  │     "產出 report.md 到 Documents/"        │     │
│  │   ],                                     │     │
│  │   "output_format": "markdown"             │     │
│  │ }                                        │     │
│  └──────────────────────────────────────────┘     │
│                                                  │
│  來源：                                           │
│  📦 內建 Skills（預裝 5-10 個常用）               │
│  📥 Skill Store（社群分享、一鍵安裝）              │
│  ✏️ 使用者自建（「幫我建一個 XX skill」）          │
│  🔗 MCP prompts 自動轉 Skill                     │
└──────────────────────────────────────────────────┘
```

---

## Layer 4: Brain Layer（Provider Router）

```
┌──────────────────────────────────────────────────────┐
│                 Provider Router                       │
│                                                      │
│  負責：                                               │
│  1. 統一 API 介面（上層不知道用的是哪個 Provider）     │
│  2. Tool schema 格式轉換                              │
│  3. Streaming response 統一處理                       │
│  4. 錯誤處理 + fallback                               │
│  5. Token 計數 + 費用追蹤                             │
│                                                      │
│  ┌──────────────────────────────────────────────┐     │
│  │ interface ProviderAdapter {                   │     │
│  │   id: String                                  │     │
│  │   displayName: String                         │     │
│  │   supportedModels: List<Model>                │     │
│  │   supportsToolCalling: Boolean                │     │
│  │   supportsVision: Boolean                     │     │
│  │   supportsStreaming: Boolean                   │     │
│  │                                               │     │
│  │   fun chat(                                   │     │
│  │     messages: List<Message>,                  │     │
│  │     tools: List<ToolSchema>?,                 │     │
│  │     config: ChatConfig                        │     │
│  │   ): Flow<ChatChunk>                          │     │
│  │ }                                             │     │
│  └──────────────────────────────────────────────┘     │
│                                                      │
│  實作：                                               │
│  ┌─────────────────────┐                              │
│  │ OpenAICompatAdapter  │ ← GHC, OpenRouter,          │
│  │                     │   Groq, Together, ...        │
│  │ baseUrl 可設定       │   任何 OpenAI-compatible     │
│  └─────────────────────┘                              │
│  ┌─────────────────────┐                              │
│  │ AnthropicAdapter     │ ← Claude (Messages API)     │
│  │ tool_use format     │                              │
│  └─────────────────────┘                              │
│  ┌─────────────────────┐                              │
│  │ GoogleGenAIAdapter   │ ← Gemini                    │
│  │ function_declarations│  (AI Studio / Vertex)       │
│  └─────────────────────┘                              │
│  ┌─────────────────────┐                              │
│  │ LocalLLMAdapter      │ ← llama.cpp on-device       │
│  │ GGUF models         │   (Qwen, Phi, Gemma 等)     │
│  └─────────────────────┘                              │
│                                                      │
│  路由策略：                                           │
│  ┌──────────────────────────────────────────────┐     │
│  │ 使用者選擇 → 該 Provider 處理                 │     │
│  │ 自動模式 → 依任務類型選最適合的 model          │     │
│  │   - 簡單問答 → 本地 LLM（免費、快、離線）     │     │
│  │   - 需要 tool → 雲端 model（tool calling 穩） │     │
│  │   - 需要視覺 → 支援 vision 的 model          │     │
│  │ Fallback → 主要 provider 失敗時自動切換       │     │
│  └──────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

**為什麼 OpenAICompatAdapter 是最重要的？**

因為 GHC（GitHub Copilot）、OpenRouter、Groq、Together、DeepSeek、Mistral 等全部走 OpenAI-compatible API。一個 adapter 就能接幾十個 provider。使用者只需要填：`baseUrl` + `apiKey` + 選 `model`。

---

## Layer 5: Session Layer

```
┌──────────────────────────────────────────────────┐
│               Session Layer                       │
│                                                  │
│  ┌──────────────────────────────────┐             │
│  │ Conversation Manager             │             │
│  │                                  │             │
│  │ 📱 多對話支援（像 ChatGPT 左側欄）│             │
│  │ 每個對話獨立 context + history    │             │
│  │ 可以為對話指定不同 model/provider │             │
│  │ 對話持久化到 Room DB              │             │
│  └──────────────────────────────────┘             │
│                                                  │
│  ┌──────────────────────────────────┐             │
│  │ Context Manager                   │             │
│  │                                  │             │
│  │ Token budget 管理：               │             │
│  │ system_prompt + tools + memory    │             │
│  │ + history + user_message          │             │
│  │ ≤ model max_tokens                │             │
│  │                                  │             │
│  │ 自動摘要過長的 history            │             │
│  │ 自動裁剪 tool output              │             │
│  └──────────────────────────────────┘             │
│                                                  │
│  ┌──────────────────────────────────┐             │
│  │ Memory (Persistent Knowledge)     │             │
│  │                                  │             │
│  │ 跨對話的使用者知識：               │             │
│  │ - 偏好（「我喜歡用 markdown」）    │             │
│  │ - 事實（「我的專案在 GitHub」）    │             │
│  │ - 習慣（「早上查天氣」）           │             │
│  │                                  │             │
│  │ 儲存：Room DB + optional RAG      │             │
│  │ (on-device embedding)             │             │
│  └──────────────────────────────────┘             │
└──────────────────────────────────────────────────┘
```

---

## Layer 6: UI Layer

```
┌──────────────────────────────────────────────────┐
│             UI Layer (Jetpack Compose)             │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ 🏠 Home / Chat Screen              │           │
│  │  - 對話列表（左側/底部導航）        │           │
│  │  - 聊天介面（文字+圖片+檔案+語音）  │           │
│  │  - Tool 執行狀態顯示               │           │
│  │  - 確認對話框（L1+ 操作）          │           │
│  └────────────────────────────────────┘           │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ 📁 Files Screen                    │           │
│  │  - 瀏覽 AI 產生的檔案              │           │
│  │  - 預覽（文字/圖片/影片）          │           │
│  │  - 分享到其他 app                  │           │
│  └────────────────────────────────────┘           │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ ✨ Skills Screen                   │           │
│  │  - 已安裝 skills                   │           │
│  │  - Skill Store（社群/推薦）        │           │
│  │  - 自建 skill                      │           │
│  └────────────────────────────────────┘           │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ ⚙️ Settings Screen                 │           │
│  │  - Provider 設定（API key, model） │           │
│  │  - MCP Server 管理                 │           │
│  │  - 安全等級（Casual/Standard/Power）│           │
│  │  - 檔案存取範圍                    │           │
│  │  - Audit Log 檢視                  │           │
│  │  - 背景模式設定                    │           │
│  └────────────────────────────────────┘           │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ 📲 Share Intent Receiver           │           │
│  │  - 接收其他 app 分享的內容          │           │
│  │  - 自動帶入對話：「分析這張圖」     │           │
│  └────────────────────────────────────┘           │
│                                                  │
│  ┌────────────────────────────────────┐           │
│  │ 🔔 Notification Actions            │           │
│  │  - 「Agent 完成任務」通知           │           │
│  │  - 快速回覆（不用開 app）          │           │
│  └────────────────────────────────────┘           │
└──────────────────────────────────────────────────┘
```

---

## 資料流：一次完整的使用者互動

```
使用者：「幫我搜尋 QNAP NAS 2026 新品，整理成報告存到下載」

    ↓ UI Layer
Chat Screen 收到文字輸入

    ↓ Session Layer
Context Manager 組裝 prompt：
  system_prompt（含 tool definitions）
  + memory（使用者偏好：繁體中文、markdown）
  + recent history
  + user message

    ↓ Brain Layer
Provider Router → 使用者選的 Provider（例如 GHC claude-sonnet-4.6）
  → 送出 chat completion with tools

    ↓ LLM 回覆
{
  tool_calls: [
    { name: "web_search", args: { query: "QNAP NAS 2026 新品" } }
  ]
}

    ↓ Action Layer
Unified Tool Registry 找到 web_search（native tool）
  → Security Layer: L0 auto-approve
  → 執行搜尋，回傳結果

    ↓ Brain Layer（第二輪）
LLM 看到搜尋結果，決定：
{
  tool_calls: [
    { name: "file_write", args: {
        path: "Download/QNAP-NAS-2026-新品報告.md",
        content: "# QNAP NAS 2026 新品整理\n\n..."
    }}
  ]
}

    ↓ Action Layer
file_write → Security Layer: L1
  → Trust Profile: Standard → 首次授權確認
  → UI 彈出確認對話框
  → 使用者確認
  → MediaStore API 寫入 Downloads/
  → 檔案出現在「檔案」app

    ↓ Brain Layer（第三輪）
LLM：「報告已存到下載資料夾：QNAP-NAS-2026-新品報告.md 📄」

    ↓ UI Layer
顯示回覆 + 可點擊的檔案連結
```

---

## 技術棧

| 層級 | 技術選擇 | 理由 |
|------|---------|------|
| 語言 | **Kotlin** | Android 官方語言，生態最好 |
| UI | **Jetpack Compose** | 現代、宣告式、Google 力推 |
| 網路 | **OkHttp + Retrofit** | Android 標準，SSE streaming 支援好 |
| 資料庫 | **Room** | 對話、memory、audit log 持久化 |
| 背景 | **WorkManager** | Google 推薦的背景任務方案 |
| JSON | **kotlinx.serialization** | 效能好、Kotlin 原生 |
| 本地 LLM | **llama.cpp (Android NDK)** | 成熟、GGUF 模型生態豐富 |
| 圖片 | **Coil** | Compose 原生圖片載入 |
| DI | **Hilt** | Google 推薦的 DI 框架 |
| 測試 | **JUnit + Compose Testing** | 標準 |

---

## 開發分期

### Phase 0: Skeleton（第 1 週）
- [ ] 建 repo + Gradle 專案結構
- [ ] 基本 UI 骨架（Chat + Settings）
- [ ] OpenAICompatAdapter（接 GHC）— 能對話就好
- [ ] 驗證：打開 app → 填 API key → 能聊天

### Phase 1: Tool System（第 2-4 週）
- [ ] ToolRegistry + ToolExecutor + PermissionGuard
- [ ] 標準 function calling 整合
- [ ] Native tools: file_read, file_write, file_list, web_search, web_fetch
- [ ] FileSystemProvider（MediaStore 路線）
- [ ] 驗證：「搜尋 X 並存成報告」能動

### Phase 2: Multi-Provider（第 4-6 週）
- [ ] AnthropicAdapter（Claude）
- [ ] GoogleGenAIAdapter（Gemini）
- [ ] Provider 設定 UI
- [ ] 驗證：同一個 app 可以切換 3 種 provider

### Phase 3: MCP（第 6-10 週）
- [ ] MCP JSON-RPC client（SSE transport）
- [ ] MCP tools 自動註冊
- [ ] MCP Server 管理 UI
- [ ] 驗證：連到一個 MCP server，使用它的 tool

### Phase 4: Polish（第 10-14 週）
- [ ] Skill Engine + 內建 Skills
- [ ] 多媒體 tools（圖片、影片、音檔）
- [ ] Memory 系統
- [ ] 語音輸入
- [ ] Share Intent 接收
- [ ] C 類使用者 onboarding 流程
- [ ] Skill Store（如果有社群了）

---

## Open Questions

1. **App 名稱？** Android Claw? PocketAgent? 其他？
2. **是否上 Google Play？** 還是先 GitHub APK？
3. **MCP Gateway 誰來跑？** 需要一個公共的 stdio→SSE proxy 嗎？
4. **本地 LLM 是否為 MVP 必要？** 還是先只做雲端 provider？
5. **是否需要 server sync？** 手機和 desktop OpenClaw 之間的 memory / skill 同步？
