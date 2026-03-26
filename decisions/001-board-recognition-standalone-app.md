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
- [ ] 在手機上實測辨識效果
- [ ] 根據 debug overlay 調整辨識參數
- [ ] go-solution 改為依賴 go_board_core，消除重複
