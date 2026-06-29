# デリログ Firebase Step 3 設計書：Google ログイン + Firestore 移行

**作成日:** 2026-06-29
**最終更新:** 2026-06-29（ChatGPT セカンドオピニオン反映）
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
| 内部データ形式 | アプリ内部では start/end を ISO 文字列で統一（Firestore との変換は DataService が担う） |

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

**方針:** `DataService` という薄いラッパーオブジェクトを追加する。アプリ本体（`records`/`earnings`/`cfg` を読み書きしている既存コード）は DataService 経由で読み書きするように切り替えるだけで、ロジック本体は変更しない。DataService がゲスト/Firestore の違いを吸収し、アプリ本体へは**常に同じ形式**でデータを返す。

**SDK 追加（CDN 1行）:**
```html
<script src="https://www.gstatic.com/firebasejs/10.12.5/firebase-firestore-compat.js"></script>
```

---

## 4. 認証フロー

```
アプリ起動
    ↓
enableIndexedDbPersistence() を1回だけ呼ぶ（Firestore 初期化直後）
    ↓
onAuthStateChanged
    ├── ログイン済み → DataService を Firestore モードで初期化 → アプリ表示
    │                   → localStorage にゲストデータがあれば移行バナー表示
    └── 未ログイン  → DataService を guest モードで初期化 → アプリ表示
                      ヘッダーに「👤 ゲスト（保存先: この端末のみ）」を表示

「ログイン」ボタン押下
    → signInWithPopup(GoogleAuthProvider)
        ├── 成功 → onAuthStateChanged が発火 → 上記フローへ
        └── auth/popup-blocked → signInWithRedirect にフォールバック
        └── その他エラー → エラートースト表示、ゲストモード継続

「ログアウト」
    → signOut() → ゲストモードに戻る（Firestore データはサーバーに残る）
```

**現行からの変更:**
- メール/パスワードログイン画面を削除
- アプリ起動時にログイン画面でブロックしない（ゲストで使える）
- ヘッダー右上にログイン状態アイコン（未ログイン: 👤 / ログイン済み: イニシャル円アイコン）を追加

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
        start: "2026-06-29T10:32:00.000Z"   ← ISO 文字列で保存
        end:   "2026-06-29T11:20:00.000Z"
        duration: 3720                        ← 秒

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
- `start` / `end` は ISO 文字列で保存。Firestore Timestamp より localStorage・JSON との互換性が高い
- `recordId` は `${Date.now()}_${Math.random().toString(36).slice(2,8)}` で生成（移行データにも必ず付与）

---

## 6. DataService インターフェース

```js
// アプリ起動時（Firestore 初期化直後）に1回だけ呼ぶ
firebase.firestore().enableIndexedDbPersistence().catch(() => {});

const DataService = {
  _mode: 'guest',          // 'guest' | 'firestore'
  _uid: null,

  init(user) {
    this._mode = user ? 'firestore' : 'guest';
    this._uid = user ? user.uid : null;
    // enableIndexedDbPersistence はここでは呼ばない（起動時に1回済み）
  },

  mode() { return this._mode; },

  // ── 読み込み ──────────────────────────────────────────
  // 戻り値は常に同じ形式（records: 配列 / earnings: dayKeyオブジェクト / cfg: オブジェクト）
  async loadRecords() { ... },    // guest: localStorage / firestore: Firestore
  async loadEarnings() { ... },
  async loadCfg() { ... },

  // ── 書き込み ──────────────────────────────────────────
  // Firestore 保存失敗時はオフラインキューに任せる。アプリは成功扱いで継続
  saveRecords(records) { ... },
  saveEarnings(earnings) { ... },
  saveCfg(cfg) { ... },

  // ── 移行 ──────────────────────────────────────────────
  hasMigrationData() {
    return !!(localStorage.getItem('fdc_records') || localStorage.getItem('fdc_earnings'));
  },
  async migrate() { ... },    // 全件書き込み → 成功確認 → clearLocalData()
  clearLocalData() {
    ['fdc_cfg','fdc_records','fdc_earnings','fdc_active',
     'fdc_cash','fdc_counts','fdc_sent'].forEach(k => localStorage.removeItem(k));
  },
};
```

**DataService が保証するデータ形式（アプリ本体側から見える形は常に同じ）:**
- `records`: `Array<{id, vendor, start(ISO文字列), end(ISO文字列), duration}>`
- `earnings`: `Object<dayKey, {uber, demae, rocket, cashLog, counts, sentAt}>`
- `cfg`: `Object`

---

## 7. データ移行フロー（localStorage → Firestore）

```
ログイン成功 → onAuthStateChanged 発火
    ↓
DataService.hasMigrationData() === true
    ↓
【分岐】Firestore に既存データ（uid のレコード）があるか？
    ├── なし → 通常の移行バナーを表示:
    │           「ゲストとして記録したデータがあります。アカウントに引き継ぎますか？」
    │           [引き継ぐ]  [削除して新規スタート]  [あとで]
    └── あり → 追加確認バナーを表示:
                「クラウドにも既存データがあります。ゲストデータを追加しますか？」
                [追加する]  [ゲストデータを削除]  [あとで]

「引き継ぐ」/「追加する」
    ├── fdc_records の全件 → users/{uid}/records/ に setDoc
    ├── fdc_earnings の全件 → users/{uid}/earnings/ に setDoc（既存 dayKey は上書き）
    ├── fdc_cfg → users/{uid}/cfg に setDoc
    ├── Firestore から再読み込みして件数を確認
    └── 成功確認後のみ clearLocalData() → バナー非表示 → render()
        （書き込み失敗時はエラートーストを出し localStorage を保持したまま）

「削除して新規スタート」/「ゲストデータを削除」
    └── clearLocalData() → バナー非表示

「あとで」
    └── sessionStorage.setItem('migrationDismissed', '1')
        → 同一セッション中は再表示しない（次回起動時は再表示）
```

---

## 8. オフライン同期

- `enableIndexedDbPersistence()` はアプリ起動時に**1回だけ**呼ぶ（ログイン状態に関係なく）
- 複数タブで `failed-precondition` が出ることがあるが、エラーは無視して続行（Firestore は動く）
- 電波なし → Firestore の書き込みが IndexedDB に蓄積（アプリは普通に動く）
- 電波復帰 → Firestore SDK が自動でサーバーと同期（アプリ側のコード不要）
- Firestore 保存エラー時はオフラインキューに任せ、アプリは成功扱いで継続する（配達中に記録が止まらないことを優先）
- ゲストモードは localStorage のみ。同期しない（仕様）

---

## 9. UI 変更点

### ヘッダー
- 右上にログイン状態バッジを追加
  - 未ログイン: `👤 ゲスト` ボタン（タップでログインポップアップ）+ 小文字で「この端末のみ保存」
  - ログイン済み: Google アカウントのイニシャル円アイコン（タップでメニュー: ログアウト）

### ログイン画面
- 現行のフルスクリーンログインを**削除**
- 代わりにヘッダーボタンからポップアップ

### 移行バナー
- ヘッダー直下に黄色バナーで表示
- 移行完了・スキップ・「あとで」（sessionStorage）で非表示

### 削除するもの
- `login-screen` div（フルスクリーンログイン）
- メールアドレス・パスワード入力フォーム
- `auth.signInWithEmailAndPassword()` の呼び出し

---

## 10. Firestore Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

- ログインユーザーは自分の `users/{uid}` 以下のみ読み書き可
- 他人の uid にはアクセス不可
- Ver1 ではこのルールで十分。本番強化（レートリミット・フィールドバリデーション）は将来対応

---

## 11. コスト見積もり（Firestore）

| 操作 | 1ユーザー/日 | 100ユーザー/月 | 無料枠 |
|------|-------------|----------------|--------|
| writes | ~20件 | ~60,000件 | 20,000件/日 |
| reads | ~50件 | ~150,000件 | 50,000件/日 |
| deletes | ~2件 | ~6,000件 | 20,000件/日 |

100ユーザーでも無料枠内に収まる見込み。

**将来の最適化課題（今は実装しない）:**
- 起動時に全件読まず直近30日だけ読み込む
- 履歴タブで必要な期間だけクエリで絞る
- 月別集計を別ドキュメントにキャッシュする

---

## 12. 実装上の注意点

- `enableIndexedDbPersistence()` はアプリ起動時1回のみ（DataService.init() では呼ばない）
- `records` の `start` / `end` はアプリ内部・Firestore ともに ISO 文字列で統一。`new Date(record.start)` は引き続き使える
- Google Sign-In ポップアップは iOS Safari / 一部 WebView でブロックされることがある。`auth/popup-blocked` エラー時は `signInWithRedirect` にフォールバック
- GAS debounce 自動保存は DataService の `saveEarnings` に統合
- Firestore 書き込みエラーはオフラインキューに任せる（エラーをアプリに伝播させない）。致命的エラーのみトースト表示
- 移行後の `clearLocalData()` は Firestore 書き込み成功確認後のみ実行

---

## 13. かずさん側の事前作業

- Firebase コンソール → Authentication → Sign-in method → Google を有効化
- 承認済みドメインに `kazutyon.github.io` を追加
- Firebase コンソール → Firestore Database を作成（本番モードで開始）
- Firestore → ルール タブに上記 Security Rules を貼り付けて公開
- Firebase コンソール → 使用量アラートを設定（月額 $5 程度を目安に）

---

## 14. 実装しないもの（将来対応）

- GitHub / Apple / Twitter などの追加プロバイダー
- Anonymous Auth（ゲスト→ログイン自動引き継ぎ）
- 有料プランの課金ゲート
- Firestore Security Rules の本番強化（レートリミット・フィールドバリデーション）
- モジュラー SDK（v9）への移行
- saveRecords の差分保存（addRecord / updateRecord / deleteRecord）
- earnings の日付範囲クエリ最適化
- Google Sheets との双方向同期（Firestore が正本、Sheets はエクスポート先）

---

## 15. 参照ファイル

- 現行アプリ: `projects/food-delivery/deli-public/index.html`
- v2 設計書: `docs/superpowers/specs/2026-06-29-derilog-v2-design.md`
- 現在地: `projects/food-delivery/CURRENT.md`
