# デリログ Firebase Step 3 設計書：Google ログイン + Firestore 移行

**作成日:** 2026-06-29
**最終更新:** 2026-06-29（Codex セカンドオピニオン反映）
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
| アーキテクチャ | DataService イベント単位 API パターン |
| Firebase プラン | Blaze（確認済み） |
| 内部データ形式 | start/end は ISO 文字列で統一（DataService が Firestore との変換を担う） |
| active タイマー | 端末ローカル（localStorage）保持のみ。Firestore には同期しない |

---

## 3. アーキテクチャ全体像

```
ゲストモード                   ログインモード
────────────                   ──────────────
localStorage                   Firestore（IndexedDB キャッシュ付き）
    ↑↓                             ↑↓
  DataService  ←── ログイン ───→  DataService
    ↑                               ↑
    └──────── アプリ本体 ───────────┘
              （呼び出し側は変更最小限）
```

**DataService の役割:** 保存先の差を吸収し、アプリ本体へは常に同じ形式でデータを返す。書き込みは全件保存ではなく**イベント単位 API**（1件追加・1件更新・1件削除）で行い、毎秒呼ばれる render() と Firestore 書き込みを分離する。

**SDK 追加（CDN 1行）:**
```html
<script src="https://www.gstatic.com/firebasejs/10.12.5/firebase-firestore-compat.js"></script>
```

---

## 4. 認証フロー

```
アプリ起動
    ↓
firebase.firestore().enablePersistence().catch(() => {})  ← compat SDK の正しい API、起動時1回のみ
    ↓
auth.getRedirectResult() を呼ぶ（redirect ログイン完了チェック）
    ├── result.user あり → onAuthStateChanged と同様にログイン処理
    └── なし → 通常起動
    ↓
onAuthStateChanged
    ├── ログイン済み → DataService.init(user) → Firestore からデータ読込 → アプリ表示
    │                   → localStorage にゲストデータがあれば移行バナーを表示
    └── 未ログイン  → DataService.init(null) → localStorage からデータ読込 → アプリ表示
                      ヘッダーに「👤 ゲスト（この端末のみ保存）」を表示

「ログイン」ボタン押下
    → signInWithPopup(GoogleAuthProvider)
        ├── 成功 → onAuthStateChanged が発火 → 上記フローへ
        └── auth/popup-blocked → signInWithRedirect(GoogleAuthProvider) にフォールバック
                                   → 次回起動時に getRedirectResult() で完了処理

「ログアウト」
    → メモリ上の records / earnings / cfg をクリア
    → DataService.init(null)（guest モードへ）
    → localStorage からデータ再読込（前ユーザーのデータは Firestore にのみ残る）
    → render()
```

**現行からの変更:**
- メール/パスワードログイン画面を削除
- アプリ起動時にログイン画面でブロックしない（ゲストで使える）
- ヘッダー右上にログイン状態バッジを追加
- `auth.getRedirectResult()` を起動時に呼ぶ処理を追加

---

## 5. Firestore データ構造

```
users/                         ← コレクション
  {uid}/                       ← ドキュメント（cfg フィールドをここに直接格納）
    weekStart: 1
    weeklyTargetHours: null
    weeklySalesTarget: null
    enabledVendors: ["uber","demae","rocket"]
    gasUrl: ""
    devMode: false

    records/                   ← サブコレクション（稼働記録、1件1ドキュメント）
      {recordId}               ← ID: "${Date.now()}_${random6}"
        id: "{recordId}"       ← ドキュメント ID と同じ値をフィールドにも持つ
        vendor: "uber"
        start: "2026-06-29T10:32:00.000Z"   ← ISO 文字列
        end:   "2026-06-29T11:20:00.000Z"
        duration: 3720                        ← 秒
        manual: false                         ← 手動追加フラグ

    earnings/                  ← サブコレクション（売上、1日1ドキュメント）
      {dayKey}                 ← "2026-06-29" 形式（localStorage キーと同じ）
        uber: 5400
        demae: 0
        rocket: { delivery: 2100, quest: 500 }
        cashLog: [{ amount: 800, note: "現金" }]
        counts: { uber: 8, demae: 0, rocket: 3 }
        sentAt: null
```

**設計の意図:**
- `users/{uid}` ドキュメント自体に cfg を格納し、Firestore のコレクション/ドキュメント交互配置規則に準拠
- `records` と `earnings` をサブコレクションにし、1ドキュメント 1MB 上限を回避
- `record.id` フィールドを明示してドキュメント ID と一致させる（編集・削除・GAS 重複排除に使用）
- `start` / `end` は ISO 文字列。Firestore・localStorage・JSON すべてと互換

---

## 6. DataService インターフェース

```js
const DataService = {
  _mode: 'guest',   // 'guest' | 'firestore'
  _uid: null,
  _db: null,

  init(user) {
    this._mode = user ? 'firestore' : 'guest';
    this._uid = user ? user.uid : null;
    this._db  = user ? firebase.firestore() : null;
    // enablePersistence は起動時に1回済み。ここでは呼ばない
  },

  mode() { return this._mode; },

  _userRef() { return this._db.collection('users').doc(this._uid); },

  // ── 読み込み（起動時・ログイン切替時に使用）────────────────
  // 戻り値は常に同じ形式
  // records: Array<{id, vendor, start(ISO), end(ISO), duration, manual}>
  // earnings: Object<dayKey, {...}>
  // cfg: Object
  async loadRecords()  { /* guest: localStorage / firestore: 全件取得 */ },
  async loadEarnings() { /* guest: localStorage / firestore: 全件取得 */ },
  async loadCfg()      { /* guest: localStorage / firestore: uid ドキュメント取得 */ },

  // ── イベント単位書き込み（毎秒 render() とは切り離す）──────
  // guest では localStorage を丸ごと更新（既存動作と同じ）
  // firestore では1件だけ書く
  async upsertRecord(record)           { /* setDoc(userRef/records/id, record) */ },
  async deleteRecord(recordId)         { /* deleteDoc(userRef/records/id) */ },
  async upsertEarnings(dayKey, data)   { /* setDoc(userRef/earnings/dayKey, data) */ },
  async saveCfg(cfg)                   { /* updateDoc(userRef, cfg) */ },

  // ── Firestore 保存失敗時の挙動 ──────────────────────────────
  // オフラインキューに任せ、アプリは成功扱いで継続
  // 致命的エラー（書込み禁止など）のみトースト表示
  // 配達中に記録が止まらないことを最優先

  // ── 移行 ────────────────────────────────────────────────────
  hasMigrationData() {
    return !!(localStorage.getItem('fdc_records') || localStorage.getItem('fdc_earnings'));
  },
  async migrate()     { /* 下記セクション7参照 */ },
  clearLocalData() {
    ['fdc_cfg','fdc_records','fdc_earnings','fdc_active',
     'fdc_cash','fdc_counts','fdc_sent'].forEach(k => localStorage.removeItem(k));
  },
};
```

**既存コードへの変更箇所（最小限）:**
- `startWork(vendor)` → `DataService.upsertRecord(newRecord)`
- `endWork()` → `DataService.upsertRecord({...record, end, duration})`
- `deleteRecord(id)` → `DataService.deleteRecord(id)`
- `editRecord(id, patch)` → `DataService.upsertRecord({...record, ...patch})`
- `saveEarnings()` の呼び出し箇所 → `DataService.upsertEarnings(dayKey, data)`
- `saveCfg()` → `DataService.saveCfg(cfg)`
- `loadData()` → `DataService.loadRecords()` / `loadEarnings()` / `loadCfg()`
- 毎秒の `saveLocalData()` は削除（イベント単位保存に置き換えられるため不要）

---

## 7. データ移行フロー（localStorage → Firestore）

```
ログイン成功 → Firestore からデータ読込完了
    ↓
DataService.hasMigrationData() === true
    ↓
【分岐】Firestore に既存データ（records が1件以上）があるか？
    ├── なし → 移行バナー:
    │           「ゲストとして記録したデータがあります。アカウントに引き継ぎますか？」
    │           [引き継ぐ]  [削除して新規スタート]  [あとで]
    └── あり → 追加確認バナー:
                「クラウドにも既存データがあります。ゲストデータを追加しますか？」
                「既存のクラウドデータに同日データがある場合は上書きされます」
                [追加する]  [ゲストデータを削除]  [あとで]

「引き継ぐ」/「追加する」の処理（再実行可能に設計）
    1. localStorage の records を500件ずつバッチ書き込み（Firestore 上限対策）
    2. localStorage の earnings を全件 setDoc（dayKey が同じなら上書き）
    3. localStorage の cfg を updateDoc（存在しなければ setDoc）
    4. Firestore から records / earnings を再読み込み
    5. 件数が localStorage の件数以上ならば clearLocalData()
    6. 成功トースト → render()
    ※ 途中でエラーが出た場合はエラートースト。localStorage はそのまま保持
    ※ 再ログイン時に再実行できる（冪等性あり）

「削除して新規スタート」/ 「ゲストデータを削除」
    → clearLocalData() → バナー非表示

「あとで」
    → sessionStorage.setItem('migrationDismissed', '1')
    → 同一セッション中は再表示しない（次回起動時は再表示）
```

---

## 8. オフライン同期

- `firebase.firestore().enablePersistence()` をアプリ起動時に**1回だけ**呼ぶ（compat SDK の正しい API）
- 複数タブで `failed-precondition` が出ることがあるが、エラーは catch して無視（Firestore は動く）
- 電波なし → 書き込みが IndexedDB に蓄積（アプリは普通に動く）
- 電波復帰 → Firestore SDK が自動でサーバーと同期
- **共有端末での注意:** ログアウトしてもオフラインキャッシュはブラウザに残る。Ver1 では許容し、将来課題とする
- ゲストモードは localStorage のみ。同期しない（仕様）

---

## 9. UI 変更点

### ヘッダー
| 状態 | 表示 |
|------|------|
| 未ログイン | `👤 ゲスト` ボタン + 小文字で「この端末のみ保存」 |
| ログイン済み | Google アカウントのイニシャル円アイコン（タップでメニュー） |

**ログイン済みメニュー:** ログアウト / アカウントを削除

### 移行バナー
- ヘッダー直下に黄色バナーで表示
- 完了・スキップ・「あとで」（sessionStorage）で非表示

### アカウント削除フロー
```
設定 → 「アカウントを削除」
    → 確認ダイアログ「すべてのデータが削除されます。元に戻せません」
    → 承認
    → Firestore: users/{uid} 以下を全削除（records / earnings / cfg）
    → auth.currentUser.delete()
    → ゲストモードへ移行
```

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
- 他人の uid にはアクセス不可（`request.auth.uid == userId` で保証）
- Ver1 ではこのルールで十分。フィールドバリデーション・レートリミットは将来対応

---

## 11. コスト見積もり（Firestore）

| 操作 | 1ユーザー/日 | 100ユーザー/月 | 無料枠 |
|------|-------------|----------------|--------|
| writes | ~20件 | ~60,000件 | 20,000件/日 |
| reads | 起動時全件 + α | 利用期間に比例して増加 | 50,000件/日 |
| deletes | ~2件 | ~6,000件 | 20,000件/日 |

**注意:** 長期利用ユーザーは起動時の全件 reads が増加する。100 ユーザー・短期利用なら無料枠内だが、将来的には期間クエリが必要。

**将来の最適化（今は実装しない）:**
- 起動時に直近 30 日だけ読み込む
- 履歴タブで必要な期間だけクエリで絞る
- 月別集計を `users/{uid}` ドキュメントにキャッシュする

---

## 12. 実装上の注意点

- compat SDK のオフライン永続化 API は `enablePersistence()`（`enableIndexedDbPersistence()` はモジュラー SDK 用）
- `start` / `end` はアプリ内部・Firestore ともに ISO 文字列で統一。`new Date(record.start)` は引き続き使える
- Google Sign-In ポップアップは iOS Safari / 一部 WebView でブロックされる。`auth/popup-blocked` 時は `signInWithRedirect` にフォールバックし、次回起動時に `auth.getRedirectResult()` で完了処理
- `getRedirectResult()` 返却時にどの Google アカウントでログインするか表示する
- GAS debounce 自動保存は `DataService.upsertEarnings()` に統合
- ログアウト時はメモリ上の records / earnings / cfg を必ずクリアしてから guest モードへ移行（前ユーザーのデータが画面に残らないようにする）
- `fdc_active`（稼働中タイマー）は端末ローカル（localStorage）のみ保持。Firestore には同期しない

---

## 13. かずさん側の事前作業

- Firebase コンソール → Authentication → Sign-in method → Google を有効化
- 承認済みドメインに `kazutyon.github.io` を追加
- Firebase コンソール → Firestore Database を作成（本番モードで開始）
- Firestore → ルール タブにセクション 10 の Security Rules を貼り付けて公開
- Firebase コンソール → 使用量アラートを設定（月額 $5 を目安に）

---

## 14. 実装しないもの（将来対応）

- GitHub / Apple / Twitter などの追加プロバイダー
- Anonymous Auth（ゲスト→ログイン自動引き継ぎ）
- 有料プランの課金ゲート
- Firestore Security Rules の追加強化（レートリミット・フィールドバリデーション）
- モジュラー SDK（v9）への移行
- upsertEarnings の日付範囲クエリ最適化
- 共有端末でのオフラインキャッシュ自動削除
- Google Sheets との双方向同期（Firestore が正本、Sheets はエクスポート先）

---

## 15. 参照ファイル

- 現行アプリ: `projects/food-delivery/deli-public/index.html`
- v2 設計書: `docs/superpowers/specs/2026-06-29-derilog-v2-design.md`
- 現在地: `projects/food-delivery/CURRENT.md`
