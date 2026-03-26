# DEC-003: 棋盤辨識下階段規劃

- **狀態**: 討論中
- **日期**: 2026-03-26
- **相關 repo**: justmaker/go-board-recognition
- **參與者**: Rex, Kael

## 背景

alpha.6 完成了基礎辨識 pipeline，包含局部盤面支援和 Otsu 自適應分類。但辨識品質在不同環境下仍不穩定，且缺少實用功能（匯出、修正、解題）。需要規劃下階段方向。

## 現有技術限制

1. **棋子分類** — Otsu + stdV 在光源不均或棋盤材質特殊時可能失靈
2. **棋盤定位** — warp 品質直接影響後續所有步驟
3. **無測試覆蓋** — 改 A 可能炸 B，無法確保穩定
4. **純驗證工具** — 辨識完無法做任何事（無匯出、無解題）

## 可行方向

### 方向 A: 功能補齊（短期可交付）

1. **SGF 匯出** — 辨識結果匯出為 SGF 檔，可丟進任何圍棋軟體
2. **手動修正 UI** — 點擊翻轉棋子顏色、新增/刪除棋子
3. **辨識信心度** — 每個交叉點給 confidence score，低信心用不同顏色標示

- 優點: 工作量小，讓 app 立即實用
- 缺點: 不解決根本的辨識品質問題

### 方向 B: 辨識品質提升（傳統 CV 路線）

1. **HoughCircles 輔助** — 用圓形偵測找棋子位置，與現有 stdV 分類交叉驗證
2. **LAPS 格點偵測** — 參考 [neural-chessboard](https://github.com/maciejczyzewski/neural-chessboard) 的 lattice point search，提升格線定位精度
3. **整合測試** — 收集測試圖片（螢幕拍照、實體棋盤），建立 regression test

- 優點: 不需要訓練資料，可在現有架構上增量改進
- 缺點: 傳統 CV 有天花板，調參永遠調不完

### 方向 C: ML 輔助（中期品質跳級）

1. **交叉點 CNN classifier** — 在 [Gomrade Dataset](https://www.kaggle.com/datasets/davids1992/gomrade-dataset-go-baduk-images-with-labels)（~2000 張圖，30 萬標註）上訓練輕量 CNN，取代 Otsu 分類
2. **YOLOv8 石頭偵測** — 在 [Roboflow Go Positions](https://universe.roboflow.com/synthetic-data-3ol2y/go-positions) 資料集上 fine-tune，直接偵測棋子
3. **Pix2pix image-to-image** — 2023 論文方法，一張照片直接輸出棋盤狀態，號稱 >99.99%

- 優點: 品質可以跳一個層級，不依賴手調參數
- 缺點: 需要訓練環境、模型部署到手機端需要 TFLite/ONNX 轉換

### 方向 D: go-solution 整合

1. 辨識結果直接送進解題引擎（KataGo）
2. 顯示最佳下法和變化圖
3. 這是整個 app 的殺手功能

- 優點: 完成 拍照→辨識→解題 的完整流程
- 缺點: 依賴辨識品質夠好才有意義

## 外部資源參考

### 開源專案

| 專案 | Stars | 方法 | 可借鏡 |
|------|-------|------|--------|
| [gbr](https://github.com/skolchin/gbr) | 82 | HoughCircles + Hough Lines | 圓形偵測找棋子 |
| [imago](https://github.com/tomasmcz/imago) | 87 | 傳統 CV + 論文 | 批次處理、棋譜重建 |
| [watchGo](https://github.com/daniel-bandstra/watchGo) | 53 | Haar Cascade | 預訓練 cascade XML 可直接用 |
| [igoki](https://github.com/CmdrDats/igoki) | 172 | CV + webcam | 即時偵測落子、錄成 SGF |
| [neural-chessboard](https://github.com/maciejczyzewski/neural-chessboard) | 313 | LAPS | 格點偵測 99.5% 精度 |
| [kifu-recorder](https://github.com/leonardost/kifu-recorder) | 27 | OpenCV (Android) | Android 手機即時錄棋 |

### 資料集

| 資料集 | 規模 | 用途 |
|--------|------|------|
| [Gomrade](https://www.kaggle.com/datasets/davids1992/gomrade-dataset-go-baduk-images-with-labels) | ~2000 圖 / 30 萬標註 | CNN classifier 訓練 |
| [Roboflow Go Positions](https://universe.roboflow.com/synthetic-data-3ol2y/go-positions) | 合成資料 9 萬+ 標註 | YOLO fine-tune |

### 商業參考

- **Kifu-Snap**（Crazy Stone 作者）— Hough + k-means，最成熟商業方案
- **Kifubara** — 多個專用小型 NN，號稱一秒辨識

## 建議優先順序

**Phase 1（立即）**: A1 SGF 匯出 + A2 手動修正 → 讓 app 可用
**Phase 2（短期）**: B1 HoughCircles + B3 整合測試 → 穩定品質
**Phase 3（中期）**: C1 CNN classifier 或 C2 YOLO → 品質跳級
**Phase 4（目標）**: D go-solution 整合 → 完整產品

## 決定

待 Rex 確認方向後更新。

## 後續行動

- [ ] Rex 確認優先方向
- [ ] 根據方向規劃具體實作計畫
