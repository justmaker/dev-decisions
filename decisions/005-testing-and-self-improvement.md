# DEC-005: 辨識測試基礎建設與自我改進 Pipeline

- **狀態**: 討論中
- **日期**: 2026-03-26
- **相關 repo**: justmaker/go-board-recognition
- **參與者**: Rex, Kael

## 背景

go-board-recognition 目前零測試覆蓋。每次調整演算法（alpha.1 → alpha.7、各實驗分支）都只能靠手動拍照比對，無法量化改進或偵測退化。

Rex 提供了兩張螢幕翻拍的死活題照片作為初始測試標的，並希望建立一個流程：
- 不斷新增測試照片
- 手動修正後的結果自動成為 ground truth
- 演算法可以持續驗證、持續進步

這本質上是一個 **人機協作的持續改進迴圈**。

## 初始測試圖片

| 檔名 | 類型 | 特徵 |
|------|------|------|
| `screen_sparse_01.jpg` | 螢幕翻拍 | 稀疏局部盤面，少量黑子 + 標記符號 |
| `screen_dense_01.jpg` | 螢幕翻拍 | 密集盤面，黑白子大量混合（右下征子形態） |

預計後續新增的場景：
- 實體棋盤（木質、塑膠、磁性）
- 不同光源（自然光、日光燈、側光）
- 不同角度（正上方、斜拍）
- 不同棋盤大小（9x9, 13x13, 19x19）
- 局部盤面 vs 完整盤面

## 設計

### 目錄結構

```
go-board-recognition/
├── test_images/
│   ├── screen_sparse_01.jpg          # 原始測試圖片
│   ├── screen_dense_01.jpg
│   └── ...                           # 未來新增
├── test_data/
│   ├── ground_truth/
│   │   ├── screen_sparse_01.json     # 人工標註的正確答案
│   │   ├── screen_dense_01.json
│   │   └── ...
│   └── results/                      # 各版本辨識結果（gitignore，CI 產生）
│       ├── alpha.7/
│       ├── hybrid.1/
│       └── ...
├── packages/core/
│   ├── lib/src/
│   │   └── board_state.dart          # 加入 toJson / fromJson
│   └── test/
│       ├── recognition_regression_test.dart   # 回歸測試
│       └── recognition_benchmark_test.dart    # 準確率基準測試
└── tools/
    └── label_export.dart             # 從手動修正匯出 ground truth 的工具
```

### Ground Truth 格式

```json
{
  "image": "screen_sparse_01.jpg",
  "labeled_by": "rex",
  "labeled_at": "2026-03-26T18:00:00+08:00",
  "board": {
    "rows": 19,
    "cols": 19,
    "is_partial": false,
    "stones": [
      { "row": 3, "col": 15, "color": "black" },
      { "row": 4, "col": 14, "color": "white" },
      ...
    ]
  },
  "notes": "螢幕翻拍，死活題，稀疏局部"
}
```

設計考量：
- 只記錄非空交叉點（stones），減少檔案大小且易讀
- `labeled_by` 追蹤標註來源（手動 vs 自動修正）
- `notes` 記錄場景特徵，方便日後分類分析

### 準確率指標

| 指標 | 定義 | 意義 |
|------|------|------|
| **Total Accuracy** | 正確交叉點數 / 總交叉點數 | 整體表現 |
| **Black Precision** | 正確辨識為黑 / 辨識為黑的總數 | 黑子誤判率 |
| **Black Recall** | 正確辨識為黑 / 實際黑子總數 | 黑子漏判率 |
| **White Precision** | 同上，白子 | |
| **White Recall** | 同上，白子 | |
| **Empty Accuracy** | 正確判空 / 實際空點 | 幽靈子問題 |
| **Per-Image Score** | 每張圖的 Total Accuracy | 找出特定弱點場景 |

### 回歸測試流程

```
dart test packages/core/test/recognition_regression_test.dart
```

測試邏輯：
1. 載入每張 `test_images/*.jpg`
2. 跑 `BoardRecognition().recognizeFromImage()`
3. 載入對應 `test_data/ground_truth/*.json`
4. 逐交叉點比對
5. 輸出準確率報告
6. 如果任何圖片的 Total Accuracy 低於門檻 → 測試失敗

**門檻策略：** 一開始不設門檻（先收集 baseline），等有足夠資料後設定合理門檻（例如：已知最佳結果 - 2%）。

## 自我改進迴圈

這是整個計畫的核心：讓每次手動修正都成為未來改進的養分。

### 迴圈流程

```
┌─────────────────────────────────────────────────────────┐
│                     使用者拍照                            │
│                        │                                │
│                        ▼                                │
│                  辨識引擎處理                             │
│                        │                                │
│                        ▼                                │
│              顯示結果 + 信心度標示                         │
│                        │                                │
│                ┌───────┴───────┐                        │
│                │               │                        │
│             結果正確         結果有誤                      │
│                │               │                        │
│                ▼               ▼                        │
│          直接使用          手動修正                       │
│                               │                        │
│                               ▼                        │
│                    匯出為 ground truth                   │
│                    (JSON + 原始圖片)                      │
│                               │                        │
│                               ▼                        │
│                     提交到 test_data/                    │
│                               │                        │
│                               ▼                        │
│              CI 跑回歸測試 → 更新準確率報告                 │
│                               │                        │
│                               ▼                        │
│                   開發者分析弱點場景                       │
│                               │                        │
│                               ▼                        │
│                    調整演算法 / 訓練模型                   │
│                               │                        │
│                               ▼                        │
│                  CI 驗證新版不退化 ✓                      │
│                               │                        │
│                               └──── 回到頂部 ────────────┘
└─────────────────────────────────────────────────────────┘
```

### Phase 1: Ground Truth 標註（手動）

**目標：** 建立初始 ground truth，驗證格式和流程。

1. **BoardState 加入 JSON 序列化**
   - `BoardState.toJson()` / `BoardState.fromJson()`
   - 用於 ground truth 儲存和載入

2. **手動修正 → 匯出 ground truth**
   - 在 `feat/manual-correction` 的 ResultScreen 加一個「儲存為測試資料」按鈕
   - 匯出修正後的 board state 為 JSON（上述格式）
   - 同時保存原始圖片路徑

3. **初始標註**
   - 用 app 跑 `screen_sparse_01.jpg` 和 `screen_dense_01.jpg`
   - 手動修正到 100% 正確
   - 匯出 ground truth JSON

### Phase 2: 回歸測試

**目標：** 任何辨識演算法的變動都能自動驗證。

1. **寫 recognition_regression_test.dart**
   - 對每張測試圖跑辨識
   - 比對 ground truth
   - 輸出準確率報告（console table）

2. **CI 整合**
   - GitHub Actions 在 push 時跑回歸測試
   - 準確率報告作為 CI artifact 保存
   - 問題：`opencv_dart` 需要 native library，CI 環境可能需要特殊設定

3. **基準記錄**
   - 每個版本的準確率寫入 `test_data/results/{version}.json`
   - 可以追蹤歷史趨勢

### Phase 3: 自動化標註流程

**目標：** 讓新增測試圖片的摩擦力降到最低。

1. **App 內一鍵匯出**
   - 手動修正完 → 點「存為 ground truth」
   - 自動產生 JSON + 複製圖片到固定目錄
   - 使用者只需 `git add` + `git commit`

2. **批次驗證工具**
   - CLI 工具：`dart run tools/benchmark.dart`
   - 掃描所有 test_images，跑辨識，比對 ground truth
   - 輸出彩色報告（綠色=進步，紅色=退化）

### Phase 4: 持續改進基礎

**目標：** 為未來 ML 訓練和演算法優化鋪路。

1. **交叉點圖片切片**
   - 從 warped image 切出每個交叉點的小圖（例如 32x32）
   - 配合 ground truth 標籤 → 直接可用的 CNN 訓練資料
   - 格式相容 Gomrade Dataset

2. **弱點分析**
   - 自動分類錯誤模式：「黑→白」「白→空」「空→黑」等
   - 統計哪種場景（光源、角度、棋盤材質）最容易出錯
   - 指引下一步優化方向

3. **A/B 測試框架**
   - 同一組圖片，跑兩個版本的辨識
   - 自動比對輸出差異
   - 用於評估實驗分支（hybrid vs haar vs alpha.7）

## 改動範圍估算

| Phase | 改動 | 預估量 |
|-------|------|--------|
| Phase 1 | BoardState JSON + 修正匯出 UI | ~100 行 |
| Phase 2 | 回歸測試 + CI 設定 | ~200 行 |
| Phase 3 | 一鍵匯出 + 批次工具 | ~150 行 |
| Phase 4 | 切片工具 + 弱點分析 | ~300 行 |

## 先決條件

- `feat/manual-correction` 的手動修正 UI 已完成 ✅
- 測試圖片已收集到 `test_images/` ✅（2 張）
- `opencv_dart` 在 CI 環境能跑（需驗證）

## 開放問題

1. **CI 環境的 opencv_dart** — `opencv_dart` 需要 native binary，GitHub Actions Linux runner 可能需要額外安裝步驟。如果太麻煩，Phase 2 的回歸測試可以先只跑 local。
2. **圖片大小** — 測試圖片（每張 ~130KB）直接 commit 到 repo 還是用 Git LFS？目前量少可以直接 commit，超過 50 張再考慮 LFS。
3. **局部盤面 ground truth** — 兩張測試圖都是螢幕翻拍，可能只辨識出局部。Ground truth 要記錄完整 19x19 還是只記錄辨識出的範圍？建議記錄辨識出的範圍 + `is_partial` flag。
4. **非棋子標記** — 螢幕翻拍的死活題上有三角形、數字等標記，辨識引擎可能把它們誤判為棋子。Ground truth 應該反映「真實棋子位置」，標記視為雜訊。

## 決定

待 Rex 確認方向後更新。
