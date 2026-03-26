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

### 研究論文

| 論文 | 年份 | 方法 | 重點 |
|------|------|------|------|
| [Go-Game Image Recognition Based on Improved Pix2pix](https://pmc.ncbi.nlm.nih.gov/articles/PMC10871096/) | 2023 | cGAN (Pix2pix + CCMA + DDC) | 目前最高準確率 >99.99%，打敗 DenseNet/VGG-16/YOLOv5 |
| [A Robust Algorithm for Go Image Recognition](https://ieeexplore.ieee.org/document/8780588/) | 2018 | Hough Transform + 透視校正 | 不同視角、棋盤、光照都有效 |
| [Automatic Extraction of Go Game Positions](https://www.seewald.at/files/2007-04.pdf) | 2010 | Keypoint + Canny Stone Detector | 47 張測試圖，平均 1.31 個交叉點錯誤 |
| [GoCam](http://users.ics.aalto.fi/thirsima/gocam/gocam.pdf) | Helsinki Univ. | MATLAB | 開創性工作，第一個從影片自動錄棋 |
| [Imago 技術規格](http://tomasm.cz/imago_files/spec.pdf) | - | 傳統 CV | 配合 imago 開源專案的完整技術文件 |

#### 方法效果比較

- **傳統 CV（Hough + 色彩分析）**：門檻低、速度快，但對光照和角度變化敏感。大部分開源專案使用此方法。
- **Pix2pix / GAN**：2023 論文達最高準確率，image-to-image 一步到位。
- **專用小型 NN（Kifubara 做法）**：多個專用網路各司其職，比傳統 CV 更 robust，實務上最務實。
- **Haar Cascade（watchGo）**：古典 ML，精度不如 NN 但比純 Hough 穩定。

### 開源專案

#### 第一梯隊（最成熟）

| 專案 | Stars | 語言 | 方法 | 可借鏡 |
|------|-------|------|------|--------|
| [igoki](https://github.com/CmdrDats/igoki) | 172 | Clojure | CV + webcam | 即時偵測落子、錄成 SGF、接 online-go.com |
| [imago](https://github.com/tomasmcz/imago) | 87 | Python | 傳統 CV (PIL/pygame) | 有配套論文，可批次重建棋譜，開源界 SOTA |
| [gbr](https://github.com/skolchin/gbr) | 82 | Python | HoughCircles + Hough Lines | 圓形偵測找棋子、SGF 輸出、有 Jupyter 教學 |
| [img2sgf](https://github.com/hanysz/img2sgf) | 62 | Python | 先偵測圓形再偵測線條 | 處理棋子遮擋格線問題，附帶手動修正編輯器 |
| [watchGo](https://github.com/daniel-bandstra/watchGo) | 53 | Python | Haar Cascade ×3 | 預訓練 cascade XML（黑子、白子、空點）可直接用 |

#### 第二梯隊（有特色）

| 專案 | Stars | 語言 | 方法 | 可借鏡 |
|------|-------|------|------|--------|
| [kifu-recorder](https://github.com/leonardost/kifu-recorder) | 27 | Java | OpenCV (Android) | Android 手機即時錄棋 |
| [kifu-cam](https://github.com/hauensteina/kifu-cam) | 14 | ObjC/C++ | OpenCV (iOS) | 拍照輸出 SGF，可匯入 SmartGo / CrazyStone |
| [PhotoKifu](https://github.com/francoisbeaussier/photokifu) | 6 | C++ | OpenCV (iOS) | 棋盤+棋子偵測，輸出 SGF |
| [Gomrade](https://github.com/smolendawid/Gomrade) | 5 | Python | CV + ML + KataGo | 偵測後接 KataGo 語音播報走法，實體棋盤 vs AI |

#### 棋盤格點偵測（跨領域）

| 專案 | Stars | 方法 | 可借鏡 |
|------|-------|------|--------|
| [neural-chessboard](https://github.com/maciejczyzewski/neural-chessboard) | 313 | LAPS + PAMG | 格點偵測 99.5%、棋盤定位 95%，技術可直接移植 |
| [chess-board-detector](https://github.com/dimasikson/chess-board-detector) | - | Homography + CNN | 角點→透視校正→格子分割→CNN 分類 |
| [ChessboardDetect](https://github.com/Elucidation/ChessboardDetect) | - | 多種演算法比較 | Lucas-Kanade 追蹤做即時偵測 |

### 資料集

| 資料集 | 規模 | 格式 | 用途 |
|--------|------|------|------|
| [Gomrade](https://www.kaggle.com/datasets/davids1992/gomrade-dataset-go-baduk-images-with-labels) | ~2000 圖 / 30 萬標註 / 62 局棋 | 座標 + 棋子位置 | CNN classifier 訓練 |
| [Roboflow Go Positions](https://universe.roboflow.com/synthetic-data-3ol2y/go-positions) | 合成資料：黑 90K + 白 90K + 格點 1K 標註 | YOLO / COCO | YOLO fine-tune |

### 商業參考

| 產品 | 方法 | 重點 |
|------|------|------|
| **Kifu-Snap**（Remi Coulom / Crazy Stone 作者）| Hough + k-means | 最成熟商業方案，有 Android app + 線上版，接 Crazy Stone 分析 |
| **BadukAI** | NN | 支援任意尺寸棋盤片段，即時拍照修正 |
| **Kifubara** | 多個專用小型 NN | 號稱一秒辨識，接 KataGo 計分，傳統 CV 太脆弱所以轉 NN |

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
