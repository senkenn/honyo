# Custom Keyboard Shortcut - Design Document

## Overview

ユーザーが翻訳トリガーのキーボードショートカットをカスタマイズできるようにする。

## 決定事項

### 実装方針

**方針2: uiohook-napi を拡張**

- 既存の `uiohook-napi` を使用したグローバルキーイベント検出を拡張
- Electron の `globalShortcut` は使用しない

**理由:**
- ダブルプレス検出は差別化ポイント（他アプリにない体験）
- 既存コードを活かせる
- globalShortcut は他アプリとの競合リスクがある

### UI

**録音方式（Record Shortcut）**

- 設定画面でキー入力を待ち受け
- ユーザーが押したキーをそのまま記録
- テキスト入力ではない

### ショートカット仕様

| 項目 | 仕様 |
|------|------|
| ダブルプレス間隔 | 800ms（固定） |
| 単一キー | 対応（例: `F1`, `Escape`） |
| モディファイア付き | 対応（例: `Ctrl+Shift+T`） |
| シーケンス | 非対応（`Ctrl+K Ctrl+C` 形式は不要） |
| ダブルプレス | オプション（オン/オフ可能） |

### 現在の実装（参考）

```typescript
// src/keyboard/handler.ts
const isC = e.keycode === UiohookKey.C;
const meta = process.platform === 'darwin' ? e.metaKey : e.ctrlKey;
if (!(isC && meta)) return;

const now = Date.now();
n = now - t < 800 ? n + 1 : 1;
t = now;

if (n === 2) {
  // トリガー発火
}
```

## 決定済み

- [x] シーケンス（`Ctrl+K Ctrl+C` 形式）→ **非対応**（不要）
- [x] ダブルプレスのオン/オフ → **設定可能にする**
- [x] デフォルトショートカット → **現状維持**（ダブル Cmd+C / Ctrl+C）

## 設計

### 設定項目

- **キー**: 任意の単一キー（例: C, T, F1, Escape）
- **モディファイア**: Ctrl / Cmd(Meta) / Alt / Shift の組み合わせ
- **ダブルプレス**: オン/オフ

### プラットフォーム差分

- uiohook-napi がキーコードを抽象化（例: C は全 OS で 46）
- モディファイアは録音したままを保存（Mac: metaKey、Linux/Win: ctrlKey）
- CommandOrControl のような抽象化は不要。各 OS でユーザーが設定する

### デフォルト

現状維持: ダブル Cmd+C (Mac) / ダブル Ctrl+C (Windows/Linux)

### 録音 UI フロー

1. 「ショートカットを変更」ボタンをクリック
2. 入力待ち状態（「キーを押してください...」）
3. ユーザーがキーを押す
4. 押されたキー + モディファイアを記録
5. 表示用文字列に変換（例: `Cmd+C`）して保存

## Waylandでグローバルショートカットを実現する方法：

  1. XDG Desktop Portal API (org.freedesktop.portal.GlobalShortcuts)
    - Wayland公式のグローバルショートカット機構
    - ユーザーが明示的にショートカットを許可する必要あり
    - GNOME 44+、KDE Plasma 5.27+ でサポート
    - Node.js から使うには D-Bus バインディングが必要（dbus-next 等）
  2. libinput 直接アクセス
    - /dev/input/* を直接読む
    - inputグループ必須 + ログアウト/ログイン必要
    - Waylandのセキュリティモデルをバイパスする形になる
    - Node.js では node-libinput 等を使う
  3. KDE/GNOME固有のD-Bus API
    - 各DE独自のショートカット登録機構を使う
    - ポータビリティが低い

現実的な選択肢： Portal APIが最もクリーンだが、実装コストが高い。uiohook-napiを捨てて別ライブラリに移行するか、X11前提で割り切るか、の判断が必要。

