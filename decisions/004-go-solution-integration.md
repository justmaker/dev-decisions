# DEC-004: go-board-recognition 與 go-solution 整合計畫

- **狀態**: 討論中
- **日期**: 2026-03-26
- **相關 repo**: justmaker/go-board-recognition, justmaker/go-solution
- **參與者**: Rex, Kael

## 背景

go-board-recognition 已從 go-solution 獨立出來，辨識演算法持續演進（alpha.1 → alpha.7 + 多個實驗分支），兩邊的 `board_recognition.dart` 已分歧。目前兩邊各自維護一份辨識邏輯和 `BoardState` 模型，改 A 不會更新 B，長期必然造成重複勞動和行為不一致。

需要決定如何讓 go-solution 使用 go-board-recognition 的辨識結果，實現「拍照 → 辨識 → 解題」的完整流程。

## 現況分析

### BoardState 差異

| 欄位 | go-solution | go-board-recognition (core) |
|------|------------|---------------------------|
| 尺寸 | `boardSize`（單一 int，正方形） | `rows` / `cols`（支援非正方形局部盤面） |
| 遊戲邏輯 | `playMove()` 含提吃 + 自殺步判定 | 無 |
| 移動歷史 | `moveHistory: List<BoardPosition>` | 無 |
| 局部盤面 | 不支援 | `isPartial` + `realEdges` |
| 向後相容 | — | `boardSize` getter 回傳 `rows` |

### 辨識 API 差異

| | go-solution | go-board-recognition |
|---|---|---|
| 入口 | `BoardRecognition().recognizeFromImage(path)` | 同名，但多 `onLog`、`keepWarpedImage` 參數 |
| 回傳 | `Future<BoardState>` | `Future<RecognitionResult>`（含 boardState + debugInfo + warpedImage） |
| 行數 | 1146 行 | 1200 行（已分歧，有更多局部盤面和 Otsu 改進） |
| 依賴 | `opencv_dart` + Flutter `debugPrint` | `opencv_dart`（零 Flutter 依賴） |

### KataGo 整合端

go-solution 的 `KataGoEngine` 透過以下方式使用 BoardState：
- `board.boardSize` — 傳給 `_engine.analyze(boardSize: ...)`
- `board.grid` — 逐格掃描轉成 GTP moves
- `board.komi` — 傳給引擎
- `board.nextPlayer` — 決定 pass 順序
- `playMove()` — UI 點擊下子後重新分析

KataGo 只支援正方形棋盤（9/13/19），所以局部盤面（非正方形）不能直接送進分析。

## 選項

### 選項 A: go-solution 直接依賴 go_board_core（替換辨識）

go-solution 用 git dependency 引入 go_board_core，刪除自己的 `board_recognition.dart`，加一層薄 adapter 轉換 BoardState。

```yaml
# go-solution pubspec.yaml
dependencies:
  go_board_core:
    git:
      url: https://github.com/justmaker/go-board-recognition.git
      path: packages/core
```

**整合方式：**
1. go-solution 呼叫 `BoardRecognition().recognizeFromImage()` → `RecognitionResult`
2. Adapter 從 `RecognitionResult.boardState` 提取 grid，建構 go-solution 自己的 BoardState
3. go-solution 保留自己的 BoardState（含 playMove、moveHistory）

**改動範圍：**
- go-solution: 刪 `board_recognition.dart`（1146 行），加 adapter（~30 行），改 pubspec
- go-board-recognition: 不動

- 優點: 最小改動量；辨識邏輯統一；go-solution 保留完整遊戲邏輯不受影響
- 缺點: 兩份 BoardState 定義仍然存在（只是辨識統一了）；adapter 增加間接層

### 選項 B: 擴展 go_board_core 為共用基礎（統一 BoardState）

把遊戲邏輯（`playMove`、`moveHistory`、提吃判定）搬進 go_board_core 的 BoardState，兩邊都 import 同一份。

**改動範圍：**
- go_board_core: BoardState 加入 `playMove()`、`moveHistory`、`copyWithNextPlayer()`
- go-solution: 刪 `models/board_state.dart` + `services/board_recognition.dart`，全部改 import go_board_core
- go-board-recognition: 不動（core 本身在擴展）

- 優點: 單一真相來源；零 adapter；長期最乾淨
- 缺點: core 變重（加入遊戲規則）；改動量較大；兩邊同時升級有風險

### 選項 C: 三層架構（core + game + apps）

```
go_board_core      — 辨識 + 基礎 BoardState（只有 grid、getStone、setStone）
go_board_game      — 遊戲邏輯（playMove、提吃、moveHistory、SGF）
go-solution        — 依賴 core + game + katago
go-board-recognition app — 依賴 core（驗證用 app）
```

- 優點: 關注點分離最好；core 維持最小；game 可被任何圍棋 app 共用
- 缺點: 多一個 package 要維護；目前只有兩個消費者，過度設計的風險

## 建議

**選項 A（adapter 模式）** — 以最小改動量達成目標。

理由：
1. **風險最低** — go-solution 的遊戲邏輯（playMove、提吃、KataGo 座標轉換）已經穩定跑著，不該碰
2. **立即可做** — 不需要重構 go_board_core，現在就能整合
3. **局部盤面問題** — go_board_core 支援非正方形棋盤，但 KataGo 不支援。Adapter 是處理這種邊界情況的正確位置
4. **日後升級** — 如果兩份 BoardState 的維護成本真的變高，再做選項 B 也不遲。YAGNI 原則

選項 B 是長期最乾淨的結局，但現在做的話改動量大且有回歸風險。先 A 再 B 是合理的漸進路徑。

## 實作計畫（選項 A）

### Phase 1: Adapter 層建立

**目標：** go-solution 使用 go_board_core 的辨識，保留自己的 BoardState。

**步驟：**

1. **go-solution pubspec 加入 go_board_core dependency**
   ```yaml
   go_board_core:
     git:
       url: https://github.com/justmaker/go-board-recognition.git
       path: packages/core
   ```

2. **新增 adapter**（`lib/services/recognition_adapter.dart`，~40 行）
   ```dart
   import 'package:go_board_core/go_board_core.dart' as core;
   import '../models/board_state.dart';

   class RecognitionAdapter {
     final core.BoardRecognition _recognition;

     RecognitionAdapter({core.LogCallback? onLog})
         : _recognition = core.BoardRecognition(onLog: onLog, keepWarpedImage: true);

     Future<BoardState> recognizeFromImage(String imagePath) async {
       final result = await _recognition.recognizeFromImage(imagePath);
       return _convert(result.boardState);
     }

     BoardState _convert(core.BoardState src) {
       // 非正方形局部盤面 → 取較大邊作為 boardSize（KataGo 需要正方形）
       final size = src.rows == src.cols ? src.rows : max(src.rows, src.cols);
       final grid = List.generate(size, (r) =>
         List.generate(size, (c) =>
           (r < src.rows && c < src.cols)
             ? _convertStone(src.grid[r][c])
             : StoneColor.empty,
         ),
       );
       return BoardState(boardSize: size, grid: grid);
     }
   }
   ```

3. **替換 AnalysisScreen 的辨識呼叫**
   - `BoardRecognition()` → `RecognitionAdapter()`
   - 回傳型別不變（還是 go-solution 的 `BoardState`）

4. **刪除 go-solution 的 `board_recognition.dart`**（1146 行）

5. **移除 go-solution pubspec 中的 `opencv_dart` 依賴**（辨識已由 go_board_core 處理）
   - 注意：需確認 go-solution 沒有其他地方直接使用 opencv_dart

### Phase 2: 辨識品質同步

**目標：** go-board-recognition 的改進自動流入 go-solution。

**步驟：**

1. **go-solution 的 go_board_core dependency 指向特定 tag 或 commit**（避免 breaking change）
   ```yaml
   go_board_core:
     git:
       url: https://github.com/justmaker/go-board-recognition.git
       path: packages/core
       ref: v0.1.0-alpha.7  # 鎖定版本
   ```

2. **建立升級流程**：在 go-board-recognition 確認新版辨識穩定後，更新 go-solution 的 ref

### Phase 3: UI 整合（拍照 → 辨識 → 解題）

**目標：** go-solution 內可直接拍照 → 使用 go_board_core 辨識 → KataGo 分析。

**步驟：**

1. **AnalysisScreen 辨識流程改用 RecognitionAdapter**（Phase 1 已完成）
2. **辨識 debug overlay**（可選）：顯示 go_board_core 的 `RecognitionDebugInfo`，讓使用者確認辨識結果
3. **手動修正整合**（可選）：辨識後讓使用者修正再送 KataGo，借鏡 `feat/manual-correction` 的 UI

### Phase 4: 驗收條件

- [ ] go-solution 拍照後使用 go_board_core 辨識（非自己的 board_recognition.dart）
- [ ] KataGo 分析結果與原本一致（regression test 用同一張測試圖比對）
- [ ] go-solution 的 `board_recognition.dart` 已刪除
- [ ] go-solution 不再直接依賴 `opencv_dart`（由 go_board_core transitive dependency 提供）
- [ ] 局部盤面（非正方形）辨識結果能正確 pad 為正方形送進 KataGo

## 開放問題

1. **局部盤面怎麼處理？** 辨識結果如果是 9x11 的局部盤面，要 pad 成 19x19 還是找最近的標準大小（9/13/19）？Adapter 需要決定策略。
2. **StoneColor enum 重複** — 兩個 package 各定義了一份相同的 `StoneColor`。Adapter 需要轉換，或者 go-solution 直接 import go_board_core 的版本。
3. **opencv_dart 版本衝突** — 如果 go-solution 保留自己的 `opencv_dart` 依賴，可能與 go_board_core 的版本衝突。Phase 1 步驟 5 應確認是否能完全移除。
4. **長期是否走向選項 B？** 如果 adapter 的轉換成本越來越高（新功能要兩邊同步），就該認真考慮統一 BoardState。

## 決定

待 Rex 確認方向後更新。
