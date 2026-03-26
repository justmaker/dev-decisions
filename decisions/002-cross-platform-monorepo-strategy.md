# DEC-002: 跨平台 Monorepo 策略

- **狀態**: 討論中
- **日期**: 2026-03-26
- **相關 repo**: justmaker/go-solution
- **參與者**: Rex, Kael

## 背景

Rex 日後有跨平台計畫（iOS / macOS / Windows）。過去經驗是一個 repo 做多平台 build，改好一個平台另一個就掛了。需要決定 repo 策略。

## 選項

### 選項 A: 一個 repo 直接多平台 build（現狀）

- 優點: 簡單，共用度高
- 缺點: 改 A 平台容易壞 B；CI/CD 複雜

### 選項 B: Branch 分平台

- 優點: 平台間不互相影響
- 缺點: 共用邏輯幾乎沒有，cherry-pick 地獄，維護成本極高

### 選項 C: 多個獨立 repo

- 優點: 完全隔離
- 缺點: 共用邏輯要手動同步，容易 drift

### 選項 D: Monorepo + packages（melos）

- 優點: 平台間不互打；核心邏輯自然共用；各 app 獨立 build；Flutter 生態標準做法
- 缺點: 初期需要花時間拆分 packages

建議結構：
```
go-board-recognition/
├── packages/
│   ├── core/                  ← 純 Dart 邏輯（零平台依賴）
│   ├── opencv_bridge/         ← OpenCV 封裝層
│   └── platform_camera/       ← 相機抽象層（各平台實作）
├── apps/
│   ├── android/
│   ├── ios/
│   └── desktop/
└── melos.yaml
```

## 決定

傾向選項 D（Monorepo + packages），但待確認。

## 後續行動

- [ ] 確認是否採用 melos
- [ ] 規劃 packages 拆分方式
- [ ] 先建 Android app，驗證結構可行性
