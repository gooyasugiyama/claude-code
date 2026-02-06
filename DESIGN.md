# SleepTimer for Mac — 設計ドキュメント

## 概要

メニューバー（タスクトレイ）に常駐し、指定時間後にMacを自動スリープさせるユーティリティアプリ。

## 技術スタック

| 項目 | 選定 | 理由 |
|------|------|------|
| 言語 | Swift | macOSネイティブAPI との親和性が最も高い |
| UI フレームワーク | SwiftUI | メニューバーアプリのPopover UIを簡潔に実装可能 |
| 最低対応OS | macOS 13 (Ventura) | `MenuBarExtra` APIが macOS 13+ で利用可能 |
| ビルドシステム | Swift Package Manager | Xcode不要でCLIビルドも可能、CI/CD と相性が良い |

## アーキテクチャ

```
┌──────────────────────────────────────────────┐
│                  macOS Menu Bar               │
│  ┌──────────┐                                │
│  │ 🌙 (icon)│  ← NSStatusItem / MenuBarExtra │
│  └────┬─────┘                                │
│       │ click                                │
│  ┌────▼─────────────────────────────┐        │
│  │        Popover / Menu            │        │
│  │                                  │        │
│  │  ┌──────────┐  ┌──────────┐     │        │
│  │  │  15 min  │  │  30 min  │     │        │
│  │  └──────────┘  └──────────┘     │        │
│  │  ┌──────────┐  ┌──────────┐     │        │
│  │  │  1 hour  │  │  2 hours │     │        │
│  │  └──────────┘  └──────────┘     │        │
│  │                                  │        │
│  │  ── Timer Active ──────────     │        │
│  │  残り: 14:32   [キャンセル]      │        │
│  │                                  │        │
│  │  [終了]                          │        │
│  └──────────────────────────────────┘        │
└──────────────────────────────────────────────┘
```

## ファイル構成

```
SleepTimer/
├── Package.swift               # SPM パッケージ定義
└── Sources/
    └── SleepTimer/
        ├── SleepTimerApp.swift  # @main エントリポイント、MenuBarExtra定義
        ├── MenuBarView.swift    # メニューバーPopoverのUI (SwiftUI View)
        ├── TimerManager.swift   # タイマーロジック (ObservableObject)
        └── SleepController.swift # スリープ実行のシステムコマンド呼び出し
```

## コンポーネント詳細

### 1. SleepTimerApp (エントリポイント)

```swift
@main
struct SleepTimerApp: App {
    @StateObject private var timerManager = TimerManager()

    var body: some Scene {
        MenuBarExtra {
            MenuBarView(timerManager: timerManager)
        } label: {
            // タイマー非動作時: 月アイコン
            // タイマー動作時: 残り時間を表示
        }
        .menuBarExtraStyle(.window)  // Popoverスタイル
    }
}
```

- `MenuBarExtra` (macOS 13+) を使用してメニューバーに常駐
- `.menuBarExtraStyle(.window)` でPopoverウィンドウとして表示
- Dockにアイコンを表示しない（`Info.plist` で `LSUIElement = true`）

### 2. MenuBarView (UI)

**状態: タイマー未設定時**
- 4つのボタンをグリッド配置: 15分 / 30分 / 1時間 / 2時間
- アプリ終了ボタン

**状態: タイマー動作中**
- 残り時間のカウントダウン表示 (mm:ss)
- キャンセルボタン
- アプリ終了ボタン

### 3. TimerManager (タイマーロジック)

```swift
@MainActor
class TimerManager: ObservableObject {
    @Published var remainingSeconds: Int = 0
    @Published var isActive: Bool = false

    private var timer: Timer?

    func start(minutes: Int) { ... }
    func cancel() { ... }
    private func tick() { ... }
    private func onTimerComplete() { ... }
}
```

- `Timer.scheduledTimer` で1秒ごとにカウントダウン
- 残り時間が0になったら `SleepController.sleep()` を呼び出し
- タイマーはキャンセル可能

### 4. SleepController (スリープ実行)

```swift
struct SleepController {
    static func sleep() {
        // pmset sleepnow コマンドを実行
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/pmset")
        process.arguments = ["sleepnow"]
        try? process.run()
    }
}
```

- `pmset sleepnow` コマンドでMacをスリープ状態にする
- 代替手段: `IOPMSleepSystem()` (IOKit) — より低レベルだが権限管理が複雑
- `pmset sleepnow` は管理者権限不要で最もシンプル

## UI デザイン

### メニューバーアイコン
- タイマー非動作時: `moon.zzz` (SF Symbols)
- タイマー動作時: `timer` (SF Symbols) + 残り時間テキスト

### Popover ウィンドウ (約 200x250pt)

```
┌─────────────────────────┐
│    Sleep Timer          │
│                         │
│  ┌─────────┐ ┌────────┐│
│  │  15 min │ │ 30 min ││
│  └─────────┘ └────────┘│
│  ┌─────────┐ ┌────────┐│
│  │ 1 hour  │ │2 hours ││
│  └─────────┘ └────────┘│
│                         │
│         [Quit]          │
└─────────────────────────┘
```

タイマー動作中:

```
┌─────────────────────────┐
│    Sleep Timer          │
│                         │
│   Sleeping in...        │
│      14:32              │
│                         │
│     [Cancel]            │
│                         │
│         [Quit]          │
└─────────────────────────┘
```

## スリープ方法の比較

| 方法 | メリット | デメリット |
|------|---------|-----------|
| `pmset sleepnow` (Process) | シンプル、権限不要 | 外部プロセス呼び出し |
| `IOPMSleepSystem()` (IOKit) | ネイティブAPI | IOKitインポート必要、やや複雑 |
| AppleScript (`tell app "Finder" to sleep`) | 簡単 | Sandboxで制限される可能性 |

**選定: `pmset sleepnow`** — 最もシンプルで信頼性が高い。

## App Sandbox / 権限

- App Sandbox: **無効** (pmset実行のため。Mac App Store配布しない前提)
- もしSandbox有効にする場合は `IOPMSleepSystem()` を使い、`com.apple.security.device.usb` entitlement が必要になる可能性がある

## ビルドと実行

```bash
# ビルド
cd SleepTimer
swift build

# 実行
swift run

# リリースビルド
swift build -c release
```

## 将来の拡張案（スコープ外）

- カスタム時間入力
- 通知（スリープ1分前に警告）
- ログイン時自動起動 (LaunchAgent)
- Sparkle によるアプリ自動更新
