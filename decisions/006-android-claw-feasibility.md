# DEC-006: Android Claw — 手機端 AI Agent 可行性與路線選擇

- **狀態**: 討論中
- **日期**: 2026-03-26
- **相關 repo**: (尚未建立，預計 justmaker/android-claw)
- **參與者**: Rex

## 背景

目前 Claude / Gemini / ChatGPT 等主流 AI app 在手機上的能力嚴重受限：
- **無法直接產生檔案到手機路徑**（圖、文、影、音）
- **不支援 MCP / Tool / Skill 等 AI agent 生態**
- **只能接自家 Provider**，無法用 GHC 訂閱 / OpenRouter / Google AI Studio 等多元推論資源
- **沒有 agent 能力**：不能自動化、不能組合多步驟操作

而很多使用者沒有 24/7 server，想直接在手機上跑 AI agent 做到：
- 用自然語言讀寫檔案（產出報告、圖片存相簿、處理影音）
- 產品規劃、市場調查、寫程式
- 接多家 LLM provider
- 使用 MCP server / tool / skill 生態

### 目標使用者

| 類型 | 特徵 | 數量潛力 |
|------|------|---------|
| **B 類（技術人）** | 有 API key、會設定，想要深度 agent 功能 | 深度使用者，口碑傳播 |
| **C 類（一般人）** | 不懂技術，裝 app 就要能用 | 量大，需求相對單純 |

## 市場調查（2026-03-26）

### 已存在的方案

| 專案 | Stars | 路線 | 評價 |
|------|-------|------|------|
| [PocketClaw](https://github.com/HenryZ838978/pocketclaw) | 11⭐ | Kotlin 原生 APK | 最接近目標，但只有 DashScope provider，無 MCP |
| [openclaw-android](https://github.com/iyeoh88-svg/openclaw-android) | 80⭐ | Termux 安裝腳本 | 功能完整但 UX 差，C 類不可能用 |
| [Clawbot](https://github.com/AbuZar-Ansarii/Clawbot) | 196⭐ | Termux 教學指南 | 把舊手機變 agent，同 Termux 限制 |
| [AutoGLM-TERMUX](https://github.com/eraycc/AutoGLM-TERMUX) | 184⭐ | Termux + 智譜 AI | 可語音控制，但綁定智譜生態 |
| [acornix](https://github.com/gonzaroman/acornix) | 81⭐ | Python on Termux | 可控制裝置，但同 Termux 限制 |

### 關鍵發現

**沒有任何現有方案同時做到：多 Provider + MCP + 原生 Android 體驗 + C 類使用者友善。**

這是一個明確的市場空缺。

## 選項

### 選項 A: Fork PocketClaw，加多 Provider + MCP

**PocketClaw 深度 Code Review 結果（95 個 .kt 檔）：**

能省的工作：
- Android 權限分級 L0-L3（2-3 週）
- 檔案沙盒 PathSandbox（1 週）
- Tool 執行框架 Registry + Executor + Parser（2-3 週）
- Rate Limiter + Audit Log（1 週）
- 基本 UI Chat + Settings + Skills（3-4 週）
- 本地 LLM 推理 llama.cpp + ONNX（2-3 週）
- **合計約省 12-16 週**

仍需自己做的：
- 多 Provider 支援（3-4 週）— 目前只有 DashScope，硬編碼
- MCP Client（4-6 週）— 完全沒有
- 多媒體處理（2-3 週）— 只支援純文字
- Multi-session（3-4 週）— 完全沒有

隱藏問題：
- ⚠️ **Tool 呼叫用自定義 `[T:tool_id:args]` 格式**，非標準 function calling，換 Provider 後穩定性堪憂
- ⚠️ **Provider 是硬編碼**，不是 plugin 架構
- ⚠️ **11 stars, 0 forks, 一人開發**，棄坑風險高
- ⚠️ Prompt 全是簡體中文

- 優點: 省 3-4 月基礎建設、架構已在實機驗證過
- 缺點: 核心架構（tool format、provider）跟需求差太遠，改到後來等於重寫；繼承技術債風險
- 預估時間: 2-3 個月到可用
- 風險: ⭐⭐⭐（中）— 最大風險是「改到一半發現不如重寫」

### 選項 B: 從零做 Kotlin 原生 App

- 優點: 架構從 Day 1 完全為需求設計（多 Provider、MCP、multi-session）；乾淨 codebase；學習價值最高
- 缺點: 所有東西從頭寫；Android 生態坑多（Doze mode、OEM 省電、Scoped Storage、不同機型行為）；UI 打磨耗時
- 預估時間: 4-6 個月到可釋出
- 風險: ⭐⭐⭐⭐（高）— 可能太累放棄，或 Android 的坑讓進度遠超預期

### 選項 C: Termux 路線（先驗證需求）

- 優點: 30 分鐘到 1 小時就能跑；功能完整（就是 OpenClaw）；零開發成本
- 缺點: C 類使用者完全不可能用；背景被 Android 殺（heartbeat 會斷）；Scoped Storage 限制無法自由存取檔案；不是產品，只是 workaround
- 預估時間: 1 小時安裝，1-2 天調教
- 風險: ⭐（低）— 但也沒有產品產出

### 選項 D: Kotlin 原生 App，參考 PocketClaw 架構但不 Fork，AI 加速開發

取 A 和 B 的優點：

- 用 PocketClaw 作為**架構參考**（Tool 系統設計、安全機制、檔案存取模式）
- 自己的 codebase，從 Day 1 支援多 Provider + MCP
- 用 OpenClaw + Claude/Copilot 等 AI coding agent 加速 Kotlin 開發
- MVP 分階段：
  - Phase 1（4-6 週）：多 Provider + 檔案讀寫 + Chat UI + Tool 框架
  - Phase 2（6-10 週）：MCP Client + Skill 系統 + 多媒體 + 語音
  - Phase 3（10-14 週）：C 類使用者友善化（設定精靈、Skill 市集、通知整合、分享整合）

- 優點: 架構自主；有現成參考不用從零摸索 Android agent 模式；AI 加速開發能大幅縮短時間
- 缺點: 仍需投入可觀時間；Android 平台的坑不會因為有參考就消失
- 預估時間: MVP 4-6 週，完整產品 3-4 個月
- 風險: ⭐⭐⭐（中）

## Android 平台關鍵技術挑戰

無論哪條路線，以下問題都必須面對：

| 挑戰 | 說明 | 嚴重度 |
|------|------|--------|
| **背景執行** | Doze mode + App Standby + OEM 省電 = 背景 process 隨時被殺 | 🔴 高 |
| **Scoped Storage** | Android 11+ 限制檔案存取，不能自由讀寫任意路徑 | 🟡 中（可用 MediaStore API 繞過部分限制） |
| **MCP on Mobile** | MCP server 通常是本地 process（stdio），Android 上跑 Node.js process 不實際；需走 SSE transport 或自己實現 | 🔴 高 |
| **OEM 碎片化** | Samsung / Xiaomi / OPPO / vivo 各有不同的省電策略和權限行為 | 🟡 中 |
| **電池消耗** | 持續的 API 呼叫 + tool 執行 = 耗電 | 🟡 中 |

## Android 獨有優勢（Server 做不到的）

| 能力 | 應用場景 |
|------|---------|
| 📷 相機 | 拍照 → AI 分析、OCR、掃碼 |
| 📍 GPS 定位 | 位置感知的任務（附近餐廳、天氣） |
| 🔔 原生推播通知 | 任務完成通知，比 Discord bot 更即時 |
| 🎤 麥克風 | 語音輸入、即時轉錄 |
| 📲 分享意圖 | 從任何 app「分享到」AI agent |
| 🔐 生物辨識 | 敏感操作指紋/臉部確認 |

## 決定

**尚未決定。** 待 Rex 確認方向。

初步傾向：**選項 D**（參考但不 Fork，AI 加速開發），理由：
1. 市場空缺明確——沒有現有方案同時滿足多 Provider + MCP + 原生體驗
2. 有 PocketClaw 作為架構參考，不需要從零摸索
3. AI coding agent 可大幅加速 Kotlin 開發
4. 可先用選項 C（Termux）花 1 小時驗證核心假設

## 後續行動

- [ ] Rex 決定是否啟動此專案
- [ ] 決定路線（A/B/C/D）
- [ ] 如果走 D：繪製架構文件（Provider 層、Tool 層、MCP Client、UI）
- [ ] 建立 justmaker/android-claw repo
- [ ] 可選：先用 Termux 跑一週驗證「手機上跑 AI agent」的真實價值
