# DEC-001: 棋盤辨識獨立 App 規劃

- **狀態**: 已決定 — 選項 B（獨立 monorepo）
- **決定日期**: 2026-03-26
- **日期**: 2026-03-26
- **相關 repo**: justmaker/go-solution
- **參與者**: Rex, Kael

## 背景

go-solution 中的棋盤辨識（`board_recognition.dart`，1146 行）精準度持續有問題。每次調參都需要跑整個 Flutter app + KataGo 才能驗證，feedback loop 太長。Rex 想把辨識功能拆出來做成獨立 Android app，專注改善精準度。

## 選項

### 選項 A: 在 go-solution 內繼續迭代

- 優點: 不用搬遷程式碼
- 缺點: 每次測試都要跑完整 app，迭代慢；辨識邏輯跟分析邏輯耦合

### 選項 B: 獨立 repo + 獨立 Android app

- 優點: 快速迭代（拍照 → 辨識 → 即時看結果）；可以加 debug overlay 標出格線和棋子判定；日後可當 library 被 go-solution 引用
- 缺點: 需要搬遷程式碼；初期有重複

### 選項 C: 獨立驗證工具（不做完整 app）

- 優點: 最小工作量
- 缺點: 沒有相機即時預覽，只能測靜態圖片

## 決定

選項 B：獨立 monorepo（`justmaker/go-board-recognition`）+ 獨立 Android 驗證 app。

## 後續行動

- [x] 確認 repo 結構（見 DEC-002 monorepo 策略）
- [x] 從 go-solution 提取 board_recognition.dart 核心邏輯 → `packages/core/`
- [x] 建立 Android app 骨架，支援拍照 + 相簿 + debug overlay → `apps/android/`
- [x] Debug APK build 驗證通過（2026-03-26）
- [x] GitHub Actions CI + Release workflow（push tag → build APK → pre-release）
- [x] 白子偵測修復 — 螢幕拍照低對比度（alpha.2）
- [x] 局部盤面自動偵測 — 邊緣留白分析判斷裁切邊 vs 真實棋盤邊（alpha.3）
- [x] 格線裁切修復 — 局部盤面只保留有偵測證據的格線範圍（alpha.4）
- [x] 棋子偵測重構 — 移除硬編碼閾值，改用 Otsu + stdV 自適應分類（alpha.6）
- [ ] 手機實測 Otsu 演算法效果
- [ ] go-solution 改為依賴 go_board_core，消除重複
- [ ] 下階段規劃（見 DEC-003）

## 進度記錄

| 版本 | 日期 | 變更 |
|------|------|------|
| v0.1.0-alpha.1 | 2026-03-26 | 初始版本：完整辨識 pipeline + debug overlay |
| v0.1.0-alpha.2 | 2026-03-26 | 修復螢幕拍照白子偵測（V range adaptive scaling） |
| v0.1.0-alpha.3 | 2026-03-26 | 支援局部盤面（邊緣留白分析 + rows/cols 非正方形） |
| v0.1.0-alpha.4 | 2026-03-26 | 修復局部盤面格線過度延伸 + 邊緣假黑子 |
| v0.1.0-alpha.5 | 2026-03-26 | 降低白子門檻（過渡版本） |
| v0.1.0-alpha.6 | 2026-03-26 | 棋子偵測改用 Otsu + stdV 主導分類，零硬編碼閾值 |
| v0.1.0-alpha.7 | 2026-03-26 | 棋子偵測改用 V 距離為主、stdV 為輔 |
| v0.1.0-hc.1 | 2026-03-26 | HoughCircles 幾何圓形偵測（實驗分支） |
| v0.1.0-hybrid.1 | 2026-03-26 | HoughCircles 定位 + V-distance 分類混合策略 |
| v0.1.0-haar.1 | 2026-03-26 | watchGo Haar Cascade 預訓練分類器 |
| v0.1.0-mc.1 | 2026-03-26 | 手動修正 UI — 點擊修正棋子 |
| v0.1.0-sgf.1 | 2026-03-26 | SGF 匯出 — 辨識結果分享為 SGF 檔 |
| v0.1.0-cam.1 | 2026-03-26 | 相機即時預覽 — 格線 overlay + 拍照 |
| v0.1.0-hist.1 | 2026-03-26 | 歷史紀錄 — 自動存檔 + 回顧瀏覽 |

## 方案對照表

| 版本代號 | Branch | 方法 | 分類 | 說明 |
|----------|--------|------|------|------|
| alpha.7 | `main` | V-distance + stdV | 傳統 CV | 以 \|V - boardV\| 為主要信號，Otsu 自適應閾值 |
| hc.1 | `feat/hough-circles` | HoughCircles | 傳統 CV | 幾何圓形偵測找棋子位置，灰度值分類黑白 |
| hybrid.1 | `feat/hybrid` | HoughCircles + V-distance | 傳統 CV | 結合圓形偵測定位與色彩距離分類，交叉驗證 |
| haar.1 | `feat/haar-cascade` | Haar Cascade | 古典 ML | watchGo 預訓練 XML（黑子/白子/空點分類器） |
| mc.1 | `feat/manual-correction` | 手動修正 UI | 人機協作 | 辨識後讓使用者點擊修正錯誤棋子 |
| sgf.1 | `feat/sgf-export` | SGF 匯出 | 功能擴展 | 辨識結果匯出為 SGF 檔，可分享至圍棋軟體 |
| cam.1 | `feat/camera-preview` | 相機即時預覽 | 功能擴展 | 拍照前顯示格線 overlay，支援 9/13/19 路切換 |
| hist.1 | `feat/history` | 歷史紀錄 | 功能擴展 | 自動儲存辨識紀錄，支援回顧與刪除 |
