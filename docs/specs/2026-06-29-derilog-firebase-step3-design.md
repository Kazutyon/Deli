# デリログ Firebase Step 3 設計書：Google ログイン + Firestore 移行

**作成日:** 2026-06-29
**対象ファイル:** `projects/food-delivery/deli-public/index.html`
**ステータス:** 設計確定・実装待ち

---

## 1. 背景・目的

デリログを一般公開・将来的に販売できるプロダクトへと育てるため、データをクラウド（Firestore）に移行する。複数ユーザーが各自のデータを持てる構造を作りながら、ログインなしでも試せるゲストモードで新規ユーザーの離脱を防ぐ。

---

## 2. 決定事項サマリー

| 項目 | 決定 |
|------|------|
| ログイン方法 | Google Sign-In のみ（メール/パスワード廃止） |
| 未ログイン | ゲストモードで利用可（localStorage のみ、同期なし） |
| データ保存 | ゲスト: localStorage / ログイン: Firestore |
| オフライン対応 | Firestore の IndexedDB 永続化（自動同期） |
| 既存データ移行 | ログイン後に移行バナーを表示、ワンタップで引き継ぎ |
| SDK | Firebase compat SDK のまま（firebase-firestore-compat を CDN 追加） |
| アーキテクチャ | DataService 薄いラッパーパターン |
| Firebase プラン | Blaze（確認済み） |

---

## 3. アーキテクチャ全体像

```
ゲストモード                   ログインモード
────────────                   ──────────────
localStorage                   Firestore（IndexedDB オフラインキャッシュ付き）
    ↑↓                             ↑↓
  DataService  ←── ログイン ───→  DataService
    ↑                               ↑
    └──────── アプリ本体 ───────────┘
              （既存コードそのまま）
```

**方針:** `DataService` という薄いラッパーオブジェクトを追加する。アプリ本体（`records`/`earnings`/`cfg` を読み書きしている既存コード）は DataService 経由で読み書きするように切り替えるだけで、ロジック本体は変更しない。

**SDK 追加（CDN 1行）:**
```html
<script src="https://www.gstatic.com/firebasejs/10.12.5/firebase-firestore-compat.js"></script>
```

---

## 4. 認証フロー

```
アプリ起動
    ↓
onAuthStateChanged
    ├── ログイン済み → DataService を Firestore モードで初期化 → アプリ表示
    └── 未ログイン  → DataService を guest モードで初期化 → アプリ表示
                      ヘッダーに「☁ ログイン」ボタンを表示

「ログイン」ボタン押下
    → signInWithPopup(GoogleAuthProvider)
        ├── 成功
        │    ├── localStorage に fdc_records / fdc_earnings があれば移行バナー表示
        │    └── なければそのまま Firestore モードへ切り替え
        └── 失敗 → エラートースト表示、ゲストモード継続

「ログアウト」
    → signOut() → ゲストモードに戻る（Firestore データはサーバーに残る）
```

**現行からの変更:**
- メール/パスワードログイン画面を削除
- アプリ起動時にログイン画面でブロックしない（ゲストで使える）
- ヘッダー右上にログイン状態アイコン（未ログイン: 👤 / ログイン済み: Google アバター or イニシャル）を追加

---

## 5. Firestore データ構造

```
users/
  {uid}/
    cfg                    ← ドキュメント（設定 1件）
      weekStart: 1
      weeklyTargetHours: null
      weeklySalesTarget: null
      enabledVendors: ["uber","demae","rocket"]
      gasUrl: ""
      devMode: false

    records/               ← サブコレクション（稼働記録、1件1ドキュメント）
      {recordId}
        vendor: "uber"
        start: Timestamp
        end: Timestamp
        duration: 3720     ← 秒

    earnings/              ← サブコレクション（売上、1日1ドキュメント）
      {dayKey}             ← "2026-06-29" 形式（既存 localStorage キーと同じ）
        uber: 5400
        demae: 0
        rocket: { delivery: 2100, quest: 500 }
        cashLog: [{ amount: 800, note: "現金" }]
        counts: { uber: 8, demae: 0, rocket: 3 }
        sentAt: null
```

**設計の意図:**
- `records` と `earnings` をサブコレクションにすることで 1ドキュメント 1MB 上限を回避
- `dayKey` 形式を localStorage と統一し、移行コードをシンプルに保つ
- `recordId` は `Date.now().toString()` + ランダムサフィックスで生成（既存ロジック流用可）

---

## 6. DataService インターフェース

```js
const DataService = {
  _mode: 'guest',          // 'guest' | 'firestore'
  _uid: null,

  init(user) {
    this._mode = user ? 'firestore' : 'guest';
    this._uid = user ? user.uid : null;
    if (user) {
      firebase.firestore().enableIndexedDbPersistence().catch(() => {});
    }
  },

  mode() { return this._mode; },

  // ── 読み込み ──────────────────────────────────────────
  async loadRecords() { ... },    // guest: localStorage / firestore: Firestore
  async loadEarnings() { ... },
  async loadCfg() { ... },

  // ── 書き込み ──────────────────────────────────────────
  saveRecords(records) { ... },
  saveEarnings(earnings) { ... },
  saveCfg(cfg) { ... },

  // ── 移行 ──────────────────────────────────────────────
  hasMigrationData() {
    return !!(localStorage.getItem('fdc_records') || localStorage.getItem('fdc_earnings'));
  },
  async migrate() { ... },        // localStorage → Firestore に全件書き込み
  clearLocalData() {
    ['fdc_cfg','fdc_records','fdc_earnings','fdc_active',
     'fdc_cash','fdc_counts','fdc_sent'].forEach(k => localStorage.removeItem(k));
  },
};
```

**既存コードへの影響:**
- `loadData()` の中身を `DataService.loadRecords()` 等に差し替え
- `saveRecords()` / `saveEarnings()` / `saveCfg()` を DataService 経由に変更
- それ以外のアプリロジック（render・計算・タブ切り替え等）は**無変更**

---

## 7. データ移行フロー（localStorage → Firestore）

```
ゲストとして使っていた状態でログイン
    ↓
DataService.hasMigrationData() === true
    ↓
ヘッダー下に移行バナー表示:
「ゲストとして記録したデータがあります。アカウントに引き継ぎますか？」
[引き継ぐ]  [削除して新規スタート]  [あとで]
    ↓
「引き継ぐ」
    ├── fdc_records の全件 → users/{uid}/records/ に setDoc
    ├── fdc_earnings の全件 → users/{uid}/earnings/ に setDoc
    ├── fdc_cfg → users/{uid}/cfg に setDoc
    └── 完了 → clearLocalData() → バナー非表示 → render()

「削除して新規スタート」
    └── clearLocalData() → バナー非表示

「あとで」
    └── バナーを閉じる（次回ログイン時に再表示）
```

---

## 8. オフライン同期

- `DataService.init()` の中で `enableIndexedDbPersistence()` を呼ぶ（Firestore モード時のみ）
- 電波なし → Firestore の書き込みが IndexedDB に蓄積（アプリは普通に動く）
- 電波復帰 → Firestore SDK が自動でサーバーと同期（アプリ側のコード不要）
- ゲストモードは localStorage のみ。同期しない（仕様）

---

## 9. UI 変更点

### ヘッダー
- 右上にログイン状態バッジを追加
  - 未ログイン: `👤 ゲスト` ボタン（タップでログインポップアップ）
  - ログイン済み: Google アカウントのイニシャル円アイコン（タップでメニュー: ログアウト）

### ログイン画面
- 現行のフルスクリーンログインを**削除**
- 代わりにヘッダーボタンからポップアップ

### 移行バナー
- ヘッダー直下に黄色バナーで表示（移行が完了/スキップされたら非表示）

### 削除するもの
- `login-screen` div（フルスクリーンログイン）
- メールアドレス・パスワード入力フォーム
- `auth.signInWithEmailAndPassword()` の呼び出し

---

## 10. コスト見積もり（Firestore）

| 操作 | 1ユーザー/日 | 100ユーザー/月 | 無料枠 |
|------|-------------|----------------|--------|
| writes | ~20件 | ~60,000件 | 20,000件/日 |
| reads | ~50件 | ~150,000件 | 50,000件/日 |
| deletes | ~2件 | ~6,000件 | 20,000件/日 |

100ユーザーでも無料枠内に収まる見込み。ユーザーが増えた時点でモニタリングして対応。

---

## 11. 実装上の注意点

- `enableIndexedDbPersistence()` は同一ブラウザの複数タブで同時に使うと警告が出る（`failed-precondition`）。エラーは無視して続行する（Firestore は動く）
- `records` の `start` / `end` は Firestore では `Timestamp` 型。読み込み時に `Date` に変換が必要（既存の `new Date(x.start)` パターンを流用）
- Google Sign-In ポップアップは iOS Safari / 一部 WebView でブロックされることがある。`signInWithPopup` がエラー（`auth/popup-blocked`）を返した場合は `signInWithRedirect` にフォールバックする
- GAS debounce 自動保存は DataService の `saveEarnings` に統合（既存の saveEarnings を DataService に委譲）

---

## 12. かずさん側の事前作業

- Firebase コンソール → Authentication → Sign-in method → Google を有効化
- 承認済みドメインに `kazutyon.github.io`（GitHub Pages のドメイン）を追加

---

## 13. 実装しないもの（将来対応）

- GitHub / Apple / Twitter などの追加プロバイダー
- ゲストアカウントのデータ自動保持（Anonymous Auth）
- 有料プランの課金ゲート
- Firestore Security Rules の本番強化（今は `auth != null` のみ）
- モジュラー SDK への移行

---

## 14. 参照ファイル

- 現行アプリ: `projects/food-delivery/deli-public/index.html`
- v2 設計書: `docs/superpowers/specs/2026-06-29-derilog-v2-design.md`
- 現在地: `projects/food-delivery/CURRENT.md`
