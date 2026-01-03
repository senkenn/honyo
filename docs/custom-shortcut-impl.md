# Custom Keyboard Shortcut - 実装計画

## ファイル構成

```
src/
├── config/
│   └── types.ts          # 変更: ShortcutConfig 型追加
├── keyboard/
│   ├── handler.ts        # 変更: ショートカット判定ロジックを汎用化
│   └── shortcut.ts       # 新規: ショートカット設定・マッチング
└── ui/
    └── settings.ts       # 変更: IPC ハンドラ追加

settings.html             # 変更: Shortcut タブ追加
```

---

## 各ファイルの変更内容

### 1. `src/config/types.ts`

**追加:**

```typescript
export interface ShortcutConfig {
  keycode: number;        // uiohook-napi のキーコード (例: 46 = C)
  ctrlKey: boolean;
  metaKey: boolean;
  altKey: boolean;
  shiftKey: boolean;
  doublePress: boolean;   // ダブルプレス有効/無効
}
```

**Config に追加:**

```typescript
export interface Config {
  // ... 既存
  shortcut?: ShortcutConfig;  // 未設定時はデフォルト
}
```

---

### 2. `src/keyboard/shortcut.ts` (新規)

**責務:** ショートカット設定の管理とキーイベントのマッチング

```typescript
import { UiohookKey, UiohookKeyboardEvent } from 'uiohook-napi';
import { ShortcutConfig } from '../config/types.ts';

// デフォルト: ダブル Cmd+C (Mac) / Ctrl+C (Win/Linux)
export function getDefaultShortcut(): ShortcutConfig {
  return {
    keycode: UiohookKey.C,
    ctrlKey: process.platform !== 'darwin',
    metaKey: process.platform === 'darwin',
    altKey: false,
    shiftKey: false,
    doublePress: true,
  };
}

// キーイベントがショートカットにマッチするか判定
export function matchesShortcut(
  event: UiohookKeyboardEvent,
  config: ShortcutConfig
): boolean {
  return (
    event.keycode === config.keycode &&
    event.ctrlKey === config.ctrlKey &&
    event.metaKey === config.metaKey &&
    event.altKey === config.altKey &&
    event.shiftKey === config.shiftKey
  );
}

// 表示用文字列生成 (例: "Cmd+C", "Ctrl+Shift+T")
export function formatShortcut(config: ShortcutConfig): string {
  const parts: string[] = [];
  if (config.ctrlKey) parts.push('Ctrl');
  if (config.metaKey) parts.push('Cmd');
  if (config.altKey) parts.push('Alt');
  if (config.shiftKey) parts.push('Shift');
  parts.push(getKeyName(config.keycode));
  return parts.join('+');
}

// キーコード → 表示名
function getKeyName(keycode: number): string {
  // UiohookKey の逆引きマップを作成
  const keyMap: Record<number, string> = {};
  for (const [name, code] of Object.entries(UiohookKey)) {
    if (typeof code === 'number') {
      keyMap[code] = name;
    }
  }
  return keyMap[keycode] ?? `Key${keycode}`;
}
```

---

### 3. `src/keyboard/handler.ts`

**変更点:**

1. ハードコードされた `UiohookKey.C` と `metaKey/ctrlKey` を `matchesShortcut()` に置換
2. `doublePress` フラグに基づいてダブルプレス判定をスキップ

```typescript
// 変更前
const isC = e.keycode === UiohookKey.C;
const meta = process.platform === 'darwin' ? e.metaKey : e.ctrlKey;
if (!(isC && meta)) return;

// 変更後
import { matchesShortcut, getDefaultShortcut } from './shortcut.ts';

const config = getConfig();
const shortcut = config.shortcut ?? getDefaultShortcut();

if (!matchesShortcut(e, shortcut)) return;

// ダブルプレス判定
if (shortcut.doublePress) {
  const now = Date.now();
  n = now - t < 800 ? n + 1 : 1;
  t = now;
  if (n !== 2) return;
  n = 0;
}

// トリガー発火
```

---

### 4. `src/ui/settings.ts`

**追加:** ショートカット設定の IPC ハンドラ

```typescript
import { ShortcutConfig } from '../config/types.ts';

// setupSettingsIPC() に追加
ipcMain.on('load-shortcut', event => {
  const config = getConfig();
  event.reply('shortcut-loaded', config.shortcut);
});

ipcMain.on('save-shortcut', (event, shortcut: ShortcutConfig) => {
  updateConfig({ shortcut });
  event.reply('shortcut-saved', true);
});

// 録音モード用: レンダラーでキー入力を受け取り、メインプロセスに送信
ipcMain.on('record-shortcut-key', (event, keyEvent: UiohookKeyboardEvent) => {
  // キーイベントから ShortcutConfig を生成
  const shortcut: Partial<ShortcutConfig> = {
    keycode: keyEvent.keycode,
    ctrlKey: keyEvent.ctrlKey,
    metaKey: keyEvent.metaKey,
    altKey: keyEvent.altKey,
    shiftKey: keyEvent.shiftKey,
  };
  event.reply('shortcut-recorded', shortcut);
});
```

---

### 5. `settings.html`

**追加:** General タブ内にショートカット設定 UI

```html
<h3>Shortcut</h3>

<div class="form-group">
  <label>Translation Trigger</label>
  <div style="display: flex; gap: 10px; align-items: center;">
    <input type="text" id="shortcut-display" readonly
           placeholder="Click 'Record' to set" style="flex: 1;" />
    <button id="record-shortcut-btn" onclick="startRecording()">Record</button>
  </div>
  <div class="help-text">Press the key combination you want to use</div>
</div>

<div class="checkbox-group">
  <input type="checkbox" id="double-press" />
  <label for="double-press" class="checkbox-label">
    Require double press
  </label>
</div>
```

**JavaScript 追加:**

```javascript
let isRecording = false;

function startRecording() {
  isRecording = true;
  document.getElementById('shortcut-display').value = 'Press keys...';
  document.getElementById('record-shortcut-btn').textContent = 'Cancel';
  document.getElementById('record-shortcut-btn').onclick = cancelRecording;

  // キー入力をリッスン
  document.addEventListener('keydown', recordKey, { once: true });
}

function recordKey(e) {
  e.preventDefault();
  if (!isRecording) return;

  // Escape でキャンセル
  if (e.key === 'Escape') {
    cancelRecording();
    return;
  }

  // モディファイア単体は無視
  if (['Control', 'Meta', 'Alt', 'Shift'].includes(e.key)) {
    document.addEventListener('keydown', recordKey, { once: true });
    return;
  }

  // キーイベントを送信
  const keyEvent = {
    keycode: getUiohookKeycode(e.code), // DOM code → uiohook keycode
    ctrlKey: e.ctrlKey,
    metaKey: e.metaKey,
    altKey: e.altKey,
    shiftKey: e.shiftKey,
  };

  ipcRenderer.send('record-shortcut-key', keyEvent);
  finishRecording();
}

// DOM KeyboardEvent.code → uiohook keycode 変換テーブル
function getUiohookKeycode(code) {
  const map = {
    'KeyA': 30, 'KeyB': 48, 'KeyC': 46, /* ... */
    'F1': 59, 'F2': 60, /* ... */
    'Escape': 1,
  };
  return map[code] ?? 0;
}
```

---

## 実装順序

1. **`src/config/types.ts`** - 型定義追加
2. **`src/keyboard/shortcut.ts`** - ユーティリティ関数作成
3. **`src/keyboard/handler.ts`** - 既存ロジック置換
4. **`src/ui/settings.ts`** - IPC 追加
5. **`settings.html`** - UI 追加

---

## 注意点

- **キーコード変換:** DOM の `KeyboardEvent.code` と uiohook-napi の `keycode` は異なる体系。変換テーブルが必要
- **モディファイアの扱い:** 録音時、モディファイアキー単体の押下は無視し、通常キーが押されるまで待機
- **設定未保存時のデフォルト:** `config.shortcut` が `undefined` の場合は `getDefaultShortcut()` を使用
