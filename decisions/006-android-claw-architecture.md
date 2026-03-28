# EnjoyClaw — 系統架構文件

- **日期**: 2026-03-26（初版）/ 2026-03-28（v2 — Open Questions 定案）
- **狀態**: Draft v2
- **相關決策**: [DEC-006](006-android-claw-feasibility.md)

---

## Open Questions — 已定案

| # | Question | 決定 | 備註 |
|---|----------|------|------|
| 1 | App 名稱 | **EnjoyClaw** | |
| 2 | 上架策略 | **先 GitHub APK** | Google Play 以後再說 |
| 3 | MCP Gateway | **手機本地** | SSE transport on localhost |
| 4 | 本地 LLM | **不做** | 純雲端 Provider |
| 5 | Server Sync | **不需要** | 手機完全獨立 |

## 關鍵技術決定

| 項目 | 決定 | 理由 |
|------|------|------|
| 目標 Android 版本 | **API 35/36（Android 15/16）** | 自用為主，裝置都很新 |
| UI 框架 | **Flutter (Dart)** | 未來可跨 iOS |
| Provider | **Anthropic (Claude) + GitHub Copilot + OpenAI (ChatGPT)** | 三大主力 |
| MCP Server 位置 | **先手機本地**，之後再開發混合/遠端 | |
| 認證 | **OS 瀏覽器 OAuth flow**（Custom Tabs / AppAuth） | |
| 跟 OpenClaw 關係 | **完全獨立** | OpenClaw 不是 open source，且目標是無 PC 場景 |

---

## 設計原則

1. **Phone-native, not server-lite** — 不是把 server 塞進手機，而是為手機重新設計 AI agent
2. **Provider-first** — 純雲端 Provider，不做本地 LLM
3. **Cross-platform ready** — Flutter 架構，未來可跨 iOS
4. **Progressive complexity** — C 類使用者開箱即用，B 類使用者可深度設定
5. **Standard protocols** — MCP、OpenAI function calling、標準 API format，不發明自己的協議

---

## 系統堆疊總覽

```
┌─────────────────────────────────────────────────────────────────┐
│                         UI Layer (Flutter)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ Chat UI  │ │ Files UI │ │ Settings │ │ Skill Store / Mgr │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Session Layer                               │
│  ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │ Conversation Mgr │ │ Context Mgr  │ │ Memory           │    │
│  │ (multi-session)  │ │ (budget,hist)│ │ (persistent)     │    │
│  └──────────────────┘ └──────────────┘ └──────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                      Brain Layer                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Provider Router                          │   │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────────────────┐  │   │
│  │  │ OpenAI-    │ │Anthropic │ │ OpenAI (ChatGPT)       │  │   │
│  │  │ Compatible │ │ Messages │ │                        │  │   │
│  │  │ (GHC)      │ │ (Claude) │ │                        │  │   │
│  │  └────────────┘ └──────────┘ └────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Action Layer                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐    │
│  │ Tool Engine  │ │ MCP Client   │ │ Skill Engine         │    │
│  │ (native)     │ │ (SSE local)  │ │ (structured prompts) │    │
│  └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘    │
│         │                │                     │                 │
│  ┌──────┴────────────────┴─────────────────────┴───────────┐    │
│  │              Unified Tool Registry                       │    │
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

```
┌─────────────────────────────────────────────┐
│            FileSystemProvider                 │
│                                              │
│  ┌─────────────┐  App 私有空間               │
│  │ Internal    │  /data/data/app/files/      │
│  │ Storage     │  → 不需要權限               │
│  └─────────────┘                              │
│                                              │
│  ┌─────────────┐  共享空間（需權限）           │
│  │ MediaStore  │  → 圖片存相簿、文件存下載      │
│  │ API         │  → Android 10+ 推薦方式      │
│  └─────────────┘                              │
│                                              │
│  ┌─────────────┐  完整存取（需特殊權限）       │
│  │ SAF /       │  → Storage Access Framework  │
│  │ MANAGE_     │  → B 類使用者可選開啟         │
│  │ EXTERNAL    │                              │
│  └─────────────┘                              │
└─────────────────────────────────────────────┘
```

### 1.2 感測器與硬體

```dart
// 統一介面
abstract class DeviceCapability {
  String get id;
  bool get isAvailable;
  Future<CapabilityResult> activate(Map<String, dynamic> params);
}

// 實作
class CameraCapability extends DeviceCapability   // 拍照、錄影
class LocationCapability extends DeviceCapability // GPS 定位
class MicrophoneCapability extends DeviceCapability // 錄音、STT
class BiometricCapability extends DeviceCapability  // 指紋/臉部確認
class ShareCapability extends DeviceCapability      // 接收其他 app 的分享
class NotificationCapability extends DeviceCapability // 推播
class ClipboardCapability extends DeviceCapability  // 剪貼簿
```

### 1.3 背景執行策略

```
前景操作 → 正常執行，沒有限制
短期背景（< 10 min）→ WorkManager (OneTimeWorkRequest)
持續背景 → Foreground Service + 常駐通知「🦞 EnjoyClaw 運行中」
定期任務 → WorkManager (PeriodicWorkRequest)，最小間隔 15 分鐘

❌ 不嘗試繞過 Doze mode
❌ 不用 AlarmManager exact alarm
❌ 不假裝永遠在背景跑
```

---

## Layer 2: Security Layer

```
4 級權限 × 3 種信任 Profile

L0 (Auto)     讀取、搜尋、分析
L1 (Grant)    寫檔案、啟動 app
L2 (Confirm)  發訊息、刪檔案
L3 (Biometric) 付款、敏感操作

🟢 Casual   — 所有 L1+ 操作都要確認
🟡 Standard — L1 首次授權後自動，L2+ 確認
🔴 Power    — L0-L1 自動，L2 確認，L3 生物辨識
```

---

## Layer 3: Action Layer（Tool + MCP + Skill）

### 3.1 Unified Tool Registry

所有 tool（native / MCP / skill-defined）統一註冊，使用標準 OpenAI function calling 格式。

### 3.2 MCP Client on Android

- **Transport**: SSE（HTTP + Server-Sent Events）
- **先做本地 MCP server**（localhost）
- 未來可擴展到遠端 MCP server

### 3.3 Skill Engine

```dart
// skill.json
{
  "name": "product-analysis",
  "description": "產品規劃與市場調查",
  "tools_required": ["web_search", "file_write", "web_fetch"],
  "system_prompt": "你是產品分析師...",
  "steps": ["搜尋目標市場資料", "分析競品", "產出 report.md"],
  "output_format": "markdown"
}
```

來源：📦 內建 Skills、📥 Skill Store、✏️ 使用者自建

---

## Layer 4: Brain Layer（Provider Router）

```dart
abstract class ProviderAdapter {
  String get id;
  String get displayName;
  List<Model> get supportedModels;
  bool get supportsToolCalling;
  bool get supportsVision;
  bool get supportsStreaming;

  Stream<ChatChunk> chat(List<Message> messages, List<ToolSchema>? tools, ChatConfig config);
}
```

### MVP Provider（3 個）

| Provider | Adapter | 認證方式 |
|----------|---------|---------|
| **Anthropic (Claude)** | AnthropicAdapter | API Key |
| **GitHub Copilot** | OpenAICompatAdapter | OAuth (Custom Tabs) |
| **OpenAI (ChatGPT)** | OpenAIAdapter | API Key |

OpenAICompatAdapter 未來可擴展接 OpenRouter、Groq 等。

### 路由策略

使用者手動選 Provider/Model。自動路由留到 Phase 2+。

---

## Layer 5: Session Layer

- 多對話管理（像 ChatGPT 左側欄）
- Context budget 管理（自動摘要過長 history）
- Memory（跨對話持久知識，存 sqflite）
- **無 Server Sync** — 手機完全獨立

---

## Layer 6: UI Layer (Flutter)

- 🏠 Chat Screen — 對話列表 + 聊天介面 + Tool 執行狀態
- 📁 Files Screen — AI 產生的檔案瀏覽/預覽/分享
- ✨ Skills Screen — 已安裝 + Skill Store + 自建
- ⚙️ Settings Screen — Provider 設定、MCP、安全等級、Audit Log
- 📲 Share Intent Receiver — 接收其他 app 分享的內容
- 🔔 Notification Actions — 快速回覆

---

## 技術棧

| 層級 | 技術選擇 | 理由 |
|------|---------|------|
| 語言 | **Dart** | Flutter 原生 |
| UI | **Flutter** | 跨平台（未來 iOS） |
| 網路 | **dio** | SSE streaming 支援好 |
| 資料庫 | **sqflite / drift** | 對話、memory、audit log 持久化 |
| 背景 | **workmanager (Flutter plugin)** | 背景任務 |
| JSON | **json_serializable / freezed** | 型別安全 |
| 圖片 | **cached_network_image** | 圖片載入/快取 |
| DI | **riverpod / get_it** | 狀態管理 + DI |
| 測試 | **flutter_test + integration_test** | 標準 |
| MCP | **自建 MCP client (SSE)** | 標準 MCP protocol |
| OAuth | **flutter_appauth** | Custom Tabs OAuth |

---

## 開發分期

### Phase 0: Skeleton（第 1 週）
- [ ] 建 Flutter repo + 專案結構
- [ ] 基本 UI 骨架（Chat + Settings）
- [ ] OpenAICompatAdapter（接 GHC）— 能對話就好
- [ ] 驗證：打開 app → 填 API key → 能聊天

### Phase 1: Tool System（第 2-4 週）
- [ ] ToolRegistry + ToolExecutor + PermissionGuard
- [ ] 標準 function calling 整合
- [ ] Native tools: file_read, file_write, file_list, web_search, web_fetch
- [ ] FileSystemProvider（MediaStore 路線）
- [ ] AnthropicAdapter + OpenAIAdapter
- [ ] 驗證：「搜尋 X 並存成報告」能動

### Phase 2: MCP + Skills（第 4-8 週）
- [ ] MCP JSON-RPC client（SSE transport, localhost）
- [ ] MCP tools 自動註冊
- [ ] Skill Engine + 內建 Skills
- [ ] MCP Server 管理 UI
- [ ] 驗證：連到本地 MCP server，使用它的 tool

### Phase 3: Polish（第 8-12 週）
- [ ] 多媒體 tools（圖片、影片、音檔）
- [ ] Memory 系統
- [ ] 語音輸入（STT）
- [ ] Share Intent 接收
- [ ] C 類使用者 onboarding 流程
- [ ] GitHub Release + APK 自動建置
