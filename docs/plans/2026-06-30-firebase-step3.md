# デリログ Firebase Step 3 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** デリログに Google Sign-In とゲストモードを追加し、データ保存先を localStorage から Firestore に移行する。

**Architecture:** DataService オブジェクトがゲスト（localStorage）と Firestore を切り替える。アプリ本体のロジックは変更せず、DataService の境界で Date ↔ ISO 文字列の変換を行う。書き込みは毎秒保存ではなくイベント単位（upsertRecord/deleteRecord/upsertEarnings/saveCfg）で行う。

**Tech Stack:** Firebase compat SDK v10.12.5（CDN）、Firestore、Firebase Auth（Google Sign-In）、vanilla JavaScript、localStorage

## Global Constraints

- 対象ファイル: `projects/food-delivery/deli-public/index.html` のみ（新規ファイル不要）
- Firebase compat SDK を使用（モジュラー SDK への移行は将来対応）
- ビルドツールなし（CDN スクリプトのみ）
- Firestore オフライン永続化 API: `firebase.firestore().enablePersistence()`（compat 用）
- records の start/end はアプリ内部では Date オブジェクトのまま。DataService 境界で ISO 文字列に変換
- Firestore データ構造: `users/{uid}` ドキュメントに cfg フィールド、`users/{uid}/records/` と `users/{uid}/earnings/` がサブコレクション
- record の ID フィールド名: `id`（既存の `clientId` は移行時に `id` として扱う）
- git remote: `https://github.com/Kazutyon/Deli.git`（main ブランチ）
- commit 後は毎回 `git push origin main` で GitHub Pages に反映

---

## 現在のコード構造（把握必須）

```
index.html
  ├── [1-391]    CSS + head
  ├── [392-405]  Firebase SDK (app-compat + auth-compat) ← Firestore CDN を追加
  ├── [409-466]  ログイン画面 HTML (login-screen div) ← 削除
  ├── [468-799]  アプリ HTML (header, settings, tabs, views)
  └── [800-2375] JS
        ├── [800-817]   定数・状態変数
        ├── [829-881]   loadData() ← DataService に置き換え
        ├── [883-894]   saveLocalData() ← イベント単位 API に分解
        ├── [896-...]   各種関数（startDel, completeDel, deleteRecord 等）
        ├── [2292]      setInterval(()=>{saveLocalData();render();},1000) ← saveLocalData 削除
        ├── [2296-2319] scheduleEarningsSave / saveEarningsToGAS (GAS 連携)
        └── [2321-2373] Firebase Auth ブロック ← Google Sign-In に書き換え
```

**現在の saveLocalData() の呼び出し箇所（全件を毎回保存）:**
- `saveSettings()` 内
- `startDel()` 内
- `completeDel()` 内
- `deleteRecord()` 内
- 編集モーダル保存時
- earnings 更新時（複数箇所）
- `setInterval` 内（毎秒）

---

## Task 1: Firebase Firestore CDN 追加 + Google Sign-In + ゲストモード UI

**Files:**
- Modify: `index.html`（head の Firebase scripts、login-screen div、header-btns、Firebase Auth ブロック）

**Interfaces:**
- Produces: `window.DS_currentUser`（null = guest、Firebase User オブジェクト = ログイン済み）
- Produces: `window.doGoogleLogin()`、`window.doLogout()`
- Produces: `#auth-badge` 要素（ヘッダーのログイン状態表示）
- Produces: `#migration-banner` 要素（移行バナー、Task 4 で使用）

---

- [ ] **Step 1: Firestore compat CDN を追加**

`index.html` の line 394（`firebase-auth-compat.js` の直後）に1行追加:

```html
  <script src="https://www.gstatic.com/firebasejs/10.12.5/firebase-firestore-compat.js"></script>
```

その直後の `<script>` ブロック（firebase.initializeApp の下）に `enablePersistence()` を追加:

```html
  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyAQ9e3gvS_h5qFMMNC6lqrEg0h2Dza3Cs0",
      authDomain: "delirog.firebaseapp.com",
      projectId: "delirog",
      storageBucket: "delirog.firebasestorage.app",
      messagingSenderId: "395910993902",
      appId: "1:395910993902:web:48ee9300426fd9d026e0ff"
    };
    firebase.initializeApp(firebaseConfig);
    firebase.firestore().enablePersistence().catch(function(err) {
      if (err.code !== 'failed-precondition' && err.code !== 'unimplemented') {
        console.warn('[Firestore] persistence error:', err.code);
      }
    });
  </script>
```

---

- [ ] **Step 2: ログイン画面 HTML を削除**

`index.html` の `<!-- ===== ログイン画面 ===== -->` から `<!-- ===== /ログイン画面 ===== -->` まで（line 409〜466 相当）を**丸ごと削除**する。

---

- [ ] **Step 3: ヘッダーにログイン状態バッジを追加**

`index.html` の `<div class="header-btns">` ブロック（現在 add/settings/logout の3ボタン）を以下に置き換える:

```html
    <div class="header-btns">
      <button class="hbtn add" onclick="openModal(null)" title="追加" aria-label="追加"></button>
      <button class="hbtn settings" onclick="toggleSettings()" title="設定" aria-label="設定"></button>
      <button id="auth-badge" onclick="handleAuthBadgeClick()"
        style="background:none;border:1px solid #e2e8f0;border-radius:999px;
               padding:4px 10px;font-size:12px;font-weight:700;cursor:pointer;color:#6b7280;
               white-space:nowrap;max-width:120px;overflow:hidden;text-overflow:ellipsis;">
        👤 ゲスト
      </button>
    </div>
```

---

- [ ] **Step 4: 移行バナー用プレースホルダー HTML を追加**

`<div id="sync-bar" ...>` の直前（`<div class="app">` の直後）に追加:

```html
  <div id="migration-banner" style="display:none;
    background:#fef3c7;border:1px solid #f59e0b;border-radius:10px;
    padding:10px 12px;margin-bottom:10px;font-size:13px;">
    <div id="migration-banner-msg"></div>
    <div style="margin-top:8px;display:flex;gap:8px;flex-wrap:wrap;">
      <button id="migration-btn-ok"
        style="background:#21a85b;color:#fff;border:none;border-radius:6px;
               padding:5px 12px;font-size:12px;font-weight:700;cursor:pointer;"></button>
      <button id="migration-btn-delete"
        style="background:#ef4444;color:#fff;border:none;border-radius:6px;
               padding:5px 12px;font-size:12px;font-weight:700;cursor:pointer;">削除して新規スタート</button>
      <button id="migration-btn-later"
        style="background:none;border:1px solid #d1d5db;border-radius:6px;
               padding:5px 12px;font-size:12px;color:#6b7280;cursor:pointer;">あとで</button>
    </div>
  </div>
```

---

- [ ] **Step 5: Firebase Auth ブロックを Google Sign-In に書き換え**

`index.html` の `// ===== Firebase Auth =====` から `// ===== /Firebase Auth =====` まで（line 2321〜2372 相当）を**丸ごと以下に置き換える**:

```js
// ===== Firebase Auth =====
window.DS_currentUser = null;

(function() {
  const auth = firebase.auth();
  const provider = new firebase.auth.GoogleAuthProvider();

  // redirect ログイン完了チェック（起動時1回）
  auth.getRedirectResult().then(function(result) {
    // onAuthStateChanged が発火するので追加処理不要
  }).catch(function(err) {
    console.warn('[Auth] getRedirectResult error:', err.code);
  });

  // ログイン状態変化
  auth.onAuthStateChanged(function(user) {
    window.DS_currentUser = user;
    updateAuthBadge(user);
    if (user) {
      // Firestore モードで DataService を再初期化してデータ読み直し
      DS.init(user).then(function() {
        render();
        showMigrationBannerIfNeeded();
      });
    } else {
      // guest モードで DataService を再初期化
      DS.init(null).then(function() {
        render();
      });
    }
  });

  window.handleAuthBadgeClick = function() {
    if (window.DS_currentUser) {
      // ログイン済み → メニュー
      const menu = [
        { label: 'ログアウト', fn: doLogout },
        { label: 'アカウントを削除', fn: doDeleteAccount }
      ];
      const choice = confirm(
        window.DS_currentUser.displayName + ' としてログイン中\n\nOK: ログアウト\nキャンセル: 閉じる'
      );
      if (choice) doLogout();
    } else {
      // ゲスト → Google ログイン
      doGoogleLogin();
    }
  };

  window.doGoogleLogin = function() {
    auth.signInWithPopup(provider).catch(function(err) {
      if (err.code === 'auth/popup-blocked' || err.code === 'auth/popup-cancelled-by-user') {
        auth.signInWithRedirect(provider);
      } else if (err.code !== 'auth/cancelled-popup-request') {
        showToast('ログインに失敗しました: ' + err.code);
      }
    });
  };

  window.doLogout = function() {
    if (!confirm('ログアウトしますか？')) return;
    auth.signOut();
    // onAuthStateChanged → DS.init(null) で guest モードに戻る
  };

  window.doDeleteAccount = async function() {
    if (!window.DS_currentUser) return;
    if (!confirm('アカウントとすべてのデータを削除します。元に戻せません。\n本当に削除しますか？')) return;
    try {
      await DS.deleteAllUserData();
      await window.DS_currentUser.delete();
      showToast('アカウントを削除しました');
    } catch(err) {
      if (err.code === 'auth/requires-recent-login') {
        showToast('セキュリティのため、一度ログアウトして再ログインしてから削除してください');
      } else {
        showToast('削除に失敗しました: ' + err.code);
      }
    }
  };
})();

function updateAuthBadge(user) {
  const badge = document.getElementById('auth-badge');
  if (!badge) return;
  if (user) {
    const initial = (user.displayName || user.email || '?')[0].toUpperCase();
    badge.textContent = initial;
    badge.title = user.displayName || user.email;
    badge.style.background = '#21a85b';
    badge.style.color = '#fff';
    badge.style.border = 'none';
  } else {
    badge.textContent = '👤 ゲスト';
    badge.style.background = 'none';
    badge.style.color = '#6b7280';
    badge.style.border = '1px solid #e2e8f0';
    badge.title = 'ログインしてクラウド保存';
  }
}

function showToast(msg) {
  let t = document.getElementById('_toast');
  if (!t) {
    t = document.createElement('div');
    t.id = '_toast';
    t.style.cssText = 'position:fixed;bottom:80px;left:50%;transform:translateX(-50%);' +
      'background:#111827;color:#fff;padding:8px 16px;border-radius:8px;font-size:13px;' +
      'z-index:999;pointer-events:none;opacity:0;transition:opacity .2s;';
    document.body.appendChild(t);
  }
  t.textContent = msg;
  t.style.opacity = '1';
  clearTimeout(t._timer);
  t._timer = setTimeout(function() { t.style.opacity = '0'; }, 3000);
}
// ===== /Firebase Auth =====
```

---

- [ ] **Step 6: 動作確認**

ブラウザで `index.html` を開いて以下を確認:
- ログイン画面が出ない（ゲストとして直接アプリが表示される）
- ヘッダー右上に「👤 ゲスト」バッジが表示される
- バッジをタップすると Google ログインポップアップが出る
- ログイン後、バッジがイニシャル（緑背景）に変わる
- ログアウトするとバッジが「👤 ゲスト」に戻る
- コンソールにエラーがない

---

- [ ] **Step 7: commit & push**

```bash
cd projects/food-delivery/deli-public
git add index.html
git commit -m "feat: add Google Sign-In and guest mode, remove email/password login"
git push origin main
```

---

## Task 2: DataService（guest モード）+ イベント単位保存に切り替え

**Files:**
- Modify: `index.html`（JS section）

**Interfaces:**
- Consumes: Task 1 の `window.DS_currentUser`
- Produces: `window.DS`（DataService オブジェクト）
  - `DS.init(user)` → Promise\<void\>
  - `DS.upsertRecord(record)` → Promise\<void\>（record は Date オブジェクト含む）
  - `DS.deleteRecord(id)` → Promise\<void\>
  - `DS.upsertEarnings(dayKey, data)` → Promise\<void\>
  - `DS.saveCfg(cfg)` → Promise\<void\>
  - `DS.hasMigrationData()` → boolean
  - `DS.migrate()` → Promise\<void\>
  - `DS.clearLocalData()` → void
  - `DS.deleteAllUserData()` → Promise\<void\>

---

- [ ] **Step 1: DataService オブジェクトを loadData() の直前に挿入**

`function loadData() {` の直前に以下を追加:

```js
// ===== DataService =====
window.DS = (function() {
  let _mode = 'guest';  // 'guest' | 'firestore'
  let _uid  = null;
  let _db   = null;

  function _userRef() { return _db.collection('users').doc(_uid); }
  function _recCol()  { return _userRef().collection('records'); }
  function _ernCol()  { return _userRef().collection('earnings'); }

  // Date オブジェクト → ISO 文字列（保存用）
  function _recToStore(r) {
    return {
      id:       r.id || r.clientId || (Date.now()+'_'+Math.random().toString(36).slice(2,8)),
      clientId: r.clientId || r.id || '',
      vendor:   r.vendor,
      start:    r.start instanceof Date ? r.start.toISOString() : r.start,
      end:      r.end   instanceof Date ? r.end.toISOString()   : r.end,
      duration: r.duration || 0,
      manual:   r.manual || false,
      pending:  r.pending || false
    };
  }

  // ISO 文字列 → Date オブジェクト（読み込み用）
  function _recFromStore(r) {
    return { ...r, start: new Date(r.start), end: new Date(r.end) };
  }

  return {
    mode() { return _mode; },

    // user: Firebase User | null
    async init(user) {
      _mode = user ? 'firestore' : 'guest';
      _uid  = user ? user.uid : null;
      _db   = user ? firebase.firestore() : null;

      // メモリ上のデータをリセット
      records  = [];
      earnings = {};
      active   = null; activeStart = null;
      try { localStorage.removeItem('fdc_active'); } catch(e) {}

      // データ読み込み
      try {
        cfg      = await this.loadCfg();
        records  = await this.loadRecords();
        earnings = await this.loadEarnings();
        // fdc_active（タイマー）は常に localStorage から
        const a = localStorage.getItem('fdc_active');
        if (a) { const ad = JSON.parse(a); active = ad.vendor; activeStart = new Date(ad.start); }
      } catch(e) { console.error('[DS.init] load error:', e); }

      // 設定パネルの UI を更新
      _applyConfigToUI();
    },

    // ── 読み込み ──────────────────────────────────────────────────
    async loadCfg() {
      if (_mode === 'guest') {
        const c = localStorage.getItem('fdc_cfg');
        const saved = c ? JSON.parse(c) : {};
        const base = {
          weekStart: 1, weeklyTargetHours: null, weeklySalesTarget: null,
          enabledVendors: ['uber','demae','rocket'], gasUrl: '', devMode: false
        };
        const merged = Object.assign(base, saved);
        // 移行処理
        if (typeof merged.weekMax === 'number' && merged.weeklyTargetHours === null) {
          merged.weeklyTargetHours = merged.weekMax > 0 ? merged.weekMax : null;
          delete merged.weekMax;
        }
        if (merged.gasUrl && merged.gasUrl.includes('AKfycbwfK9kRpmAC6w0FaWc1WOAGtZkwn')) merged.gasUrl = '';
        if (!Array.isArray(merged.enabledVendors)) merged.enabledVendors = ['uber','demae','rocket'];
        return merged;
      } else {
        const snap = await _userRef().get();
        if (!snap.exists) return cfg; // 初回: デフォルト値を返す
        const data = snap.data();
        return {
          weekStart:          data.weekStart          ?? 1,
          weeklyTargetHours:  data.weeklyTargetHours  ?? null,
          weeklySalesTarget:  data.weeklySalesTarget  ?? null,
          enabledVendors:     data.enabledVendors     ?? ['uber','demae','rocket'],
          gasUrl:             data.gasUrl             ?? '',
          devMode:            data.devMode            ?? false
        };
      }
    },

    async loadRecords() {
      if (_mode === 'guest') {
        const r = localStorage.getItem('fdc_records');
        if (!r) return [];
        return JSON.parse(r).map(_recFromStore);
      } else {
        const snap = await _recCol().orderBy('start').get();
        return snap.docs.map(d => _recFromStore(d.data()));
      }
    },

    async loadEarnings() {
      if (_mode === 'guest') {
        const e = localStorage.getItem('fdc_earnings');
        const base = e ? JSON.parse(e) : {};
        // legacy キーをマージ
        const legacyCash   = JSON.parse(localStorage.getItem('fdc_cash')   || '{}');
        const legacyCounts = JSON.parse(localStorage.getItem('fdc_counts') || '{}');
        const legacySent   = JSON.parse(localStorage.getItem('fdc_sent')   || '{}');
        const dayKeys = new Set([
          ...Object.keys(base),
          ...Object.keys(legacyCash),
          ...Object.keys(legacyCounts),
          ...Object.keys(legacySent)
        ]);
        dayKeys.forEach(dk => {
          if (!base[dk]) base[dk] = {};
          const d = base[dk];
          if (!Array.isArray(d.cashLog) && Array.isArray(legacyCash[dk])) d.cashLog = legacyCash[dk];
          if (!d.counts && legacyCounts[dk]) d.counts = legacyCounts[dk];
          if ((typeof d.sentAt === 'undefined' || d.sentAt === null) && legacySent[dk]) d.sentAt = legacySent[dk];
        });
        return base;
      } else {
        const snap = await _ernCol().get();
        const result = {};
        snap.docs.forEach(d => { result[d.id] = d.data(); });
        return result;
      }
    },

    // ── イベント単位書き込み ──────────────────────────────────────
    async upsertRecord(record) {
      const stored = _recToStore(record);
      // id を record に書き戻し（新規生成された場合）
      record.id = stored.id;
      if (_mode === 'guest') {
        const idx = records.findIndex(r => (r.id || r.clientId) === stored.id);
        if (idx >= 0) records[idx] = { ...records[idx], ...record };
        // records 配列は呼び出し元が既に更新済みなのでここでは localStorage のみ更新
        try {
          localStorage.setItem('fdc_records', JSON.stringify(records.map(_recToStore)));
        } catch(e) {}
      } else {
        try {
          await _recCol().doc(stored.id).set(stored);
        } catch(e) { console.warn('[DS] upsertRecord error (offline ok):', e.code); }
      }
    },

    async deleteRecord(id) {
      if (_mode === 'guest') {
        try {
          localStorage.setItem('fdc_records', JSON.stringify(records.map(_recToStore)));
        } catch(e) {}
      } else {
        try {
          await _recCol().doc(id).delete();
        } catch(e) { console.warn('[DS] deleteRecord error:', e.code); }
      }
    },

    async upsertEarnings(dayKey, data) {
      if (_mode === 'guest') {
        try {
          localStorage.setItem('fdc_earnings', JSON.stringify(earnings));
        } catch(e) {}
      } else {
        try {
          await _ernCol().doc(dayKey).set(data);
        } catch(e) { console.warn('[DS] upsertEarnings error (offline ok):', e.code); }
      }
    },

    async saveCfg(cfgObj) {
      if (_mode === 'guest') {
        try {
          localStorage.setItem('fdc_cfg', JSON.stringify(cfgObj));
        } catch(e) {}
      } else {
        try {
          await _userRef().set(cfgObj, { merge: true });
        } catch(e) { console.warn('[DS] saveCfg error:', e.code); }
      }
    },

    // ── 移行 ────────────────────────────────────────────────────
    hasMigrationData() {
      return !!(localStorage.getItem('fdc_records') || localStorage.getItem('fdc_earnings'));
    },

    async migrate() {
      // Step 1: records を 500 件ずつバッチ書き込み
      const localRecs = JSON.parse(localStorage.getItem('fdc_records') || '[]');
      const BATCH_SIZE = 500;
      for (let i = 0; i < localRecs.length; i += BATCH_SIZE) {
        const batch = _db.batch();
        localRecs.slice(i, i + BATCH_SIZE).forEach(function(r) {
          const stored = _recToStore({ ...r, start: new Date(r.start), end: new Date(r.end) });
          batch.set(_recCol().doc(stored.id), stored);
        });
        await batch.commit();
      }
      // Step 2: earnings を全件 setDoc
      const localErn = JSON.parse(localStorage.getItem('fdc_earnings') || '{}');
      const ernBatch = _db.batch();
      Object.keys(localErn).forEach(function(dk) {
        ernBatch.set(_ernCol().doc(dk), localErn[dk]);
      });
      await ernBatch.commit();
      // Step 3: cfg を保存
      const localCfg = JSON.parse(localStorage.getItem('fdc_cfg') || '{}');
      await _userRef().set(localCfg, { merge: true });
      // Step 4: 再読み込みして件数確認
      const snap = await _recCol().get();
      if (snap.size >= localRecs.length) {
        this.clearLocalData();
      } else {
        throw new Error('件数不一致: local=' + localRecs.length + ', firestore=' + snap.size);
      }
    },

    clearLocalData() {
      ['fdc_cfg','fdc_records','fdc_earnings','fdc_active',
       'fdc_cash','fdc_counts','fdc_sent'].forEach(function(k) {
        localStorage.removeItem(k);
      });
    },

    async deleteAllUserData() {
      if (!_uid || !_db) return;
      // records を全件削除
      const recSnap = await _recCol().get();
      const recBatch = _db.batch();
      recSnap.docs.forEach(d => recBatch.delete(d.ref));
      await recBatch.commit();
      // earnings を全件削除
      const ernSnap = await _ernCol().get();
      const ernBatch = _db.batch();
      ernSnap.docs.forEach(d => ernBatch.delete(d.ref));
      await ernBatch.commit();
      // cfg（uid ドキュメント自体）を削除
      await _userRef().delete();
    },

    // 現在 Firestore にデータが存在するか
    async hasFirestoreData() {
      if (_mode !== 'firestore') return false;
      const snap = await _recCol().limit(1).get();
      return !snap.empty;
    }
  };
})();
// ===== /DataService =====
```

---

- [ ] **Step 2: loadData() を削除し、DS.init() に置き換え**

現在の `function loadData() { ... }` 関数全体（line 829〜881 相当）を**削除**する。

`_applyConfigToUI()` 関数を DataService の直後（または同じ `<script>` 内の任意の位置）に追加:

```js
function _applyConfigToUI() {
  const wsEl = document.getElementById('week-start');
  if (wsEl) wsEl.value = cfg.weekStart;
  const wthEl = document.getElementById('weekly-target-hours');
  if (wthEl) wthEl.value = cfg.weeklyTargetHours != null ? cfg.weeklyTargetHours : '';
  const wstEl = document.getElementById('weekly-sales-target');
  if (wstEl) wstEl.value = cfg.weeklySalesTarget != null ? cfg.weeklySalesTarget : '';
  ['uber','demae','rocket'].forEach(function(v) {
    const el = document.getElementById('chk-vendor-' + v);
    if (el) el.checked = (cfg.enabledVendors || ['uber','demae','rocket']).includes(v);
  });
  const gasEl = document.getElementById('gas-url');
  if (gasEl) gasEl.value = cfg.gasUrl || '';
  renderVendorGrid();
}
```

---

- [ ] **Step 3: saveLocalData() を削除し、saveSettings() を更新**

現在の `function saveLocalData() { ... }` 関数全体（line 883〜894 相当）を**削除**する。

`saveSettings()` の末尾の `saveLocalData();` を `DS.saveCfg(cfg);` に変更:

```js
function saveSettings() {
  cfg.weekStart = parseInt(document.getElementById('week-start').value);
  const wthEl = document.getElementById('weekly-target-hours');
  cfg.weeklyTargetHours = wthEl && wthEl.value !== '' ? (parseInt(wthEl.value) || null) : null;
  const wstEl = document.getElementById('weekly-sales-target');
  cfg.weeklySalesTarget = wstEl && wstEl.value !== '' ? (parseInt(wstEl.value) || null) : null;
  cfg.enabledVendors = ['uber','demae','rocket'].filter(function(v) {
    const el = document.getElementById('chk-vendor-' + v);
    return el && el.checked;
  });
  const gasEl = document.getElementById('gas-url');
  if (gasEl) cfg.gasUrl = gasEl.value;
  DS.saveCfg(cfg);
  updateSyncBar();
}
```

---

- [ ] **Step 4: startDel() / completeDel() / deleteRecord() を DataService に切り替え**

`startDel(v)` 内の `saveLocalData(); render();` を以下に変更:

```js
  active = v; activeStart = new Date();
  DS.upsertRecord(rec).catch(function(){});  // 業者切替で追加された rec がある場合
  render();
```

※ active 自体は setInterval が localStorage に保存するので追加処理不要。

`completeDel()` 内の `saveLocalData(); render();` を以下に変更（`records.push(rec)` の後）:

```js
    active = null; activeStart = null;
    try { localStorage.removeItem('fdc_active'); } catch(e) {}
    DS.upsertRecord(rec).catch(function(){});
    render();
```

`deleteRecord(idx)` 関数全体を以下に置き換え:

```js
async function deleteRecord(idx) {
  if (!confirm('この記録を削除しますか？')) return;
  const rec = records[idx];
  const recId = rec.id || rec.clientId;
  if (cfg.gasUrl && rec.id) {
    await postToGAS({ action:'delete', id:rec.id });
    setTimeout(syncFromGAS, 1500);
  }
  records.splice(idx, 1);
  DS.deleteRecord(recId).catch(function(){});
  render();
}
```

---

- [ ] **Step 5: 編集モーダル保存時を DataService に切り替え**

編集モーダルの保存処理（`saveLocalData(); closeModal();` の箇所）を探し、以下に変更:

```js
    DS.upsertRecord(records[editIdx]).catch(function(){});
    closeModal();
    render();
```

---

- [ ] **Step 6: earnings 更新を DataService に切り替え**

`scheduleEarningsSave(dayKey)` 関数を以下に変更（GAS 連携はそのまま）:

```js
function scheduleEarningsSave(dayKey) {
  clearTimeout(_earningsSaveTimer);
  _earningsSaveTimer = setTimeout(function() {
    DS.upsertEarnings(dayKey, earnings[dayKey] || {}).catch(function(){});
    if (cfg.gasUrl) saveEarningsToGAS(dayKey);
  }, 2000);
}
```

earnings をリセットする箇所（`delete earnings[dayKey]; saveLocalData();`）を:

```js
  delete earnings[dayKey];
  DS.upsertEarnings(dayKey, {}).catch(function(){});
```

inline oninput の `saveLocalData()` 呼び出しは `scheduleEarningsSave('${dayKey}')` に統合されているので変更不要。

---

- [ ] **Step 7: setInterval から saveLocalData を削除**

`setInterval(()=>{saveLocalData();render();},1000);` を以下に変更:

```js
setInterval(function() {
  // fdc_active（タイマー中断回復用）のみ毎秒保存
  if (active) {
    try {
      localStorage.setItem('fdc_active', JSON.stringify({ vendor: active, start: activeStart.toISOString() }));
    } catch(e) {}
  }
  render();
}, 1000);
```

---

- [ ] **Step 8: 起動時の DS.init() 呼び出しに変更**

`loadData();` を呼んでいる箇所（現在 `updateSyncBar();` の前後）を探し、削除する。
代わりに `DS.init(null)` を呼ぶ。ただし Task 1 で追加した `onAuthStateChanged` が `DS.init()` を呼ぶので、起動時の明示的呼び出しは**不要**。

`updateSyncBar();` と `if(cfg.gasUrl) syncFromGAS();` の前にあった `loadData();` 行を削除するだけでよい。

---

- [ ] **Step 9: 動作確認**

ブラウザで以下を確認:
- ゲストモードで全機能が動作する（稼働記録・売上入力・履歴・分析）
- 稼働開始 → タブを閉じる → 再度開く → タイマーが復元される
- 記録の編集・削除が動作する
- 設定が保存される
- コンソールにエラーがない

---

- [ ] **Step 10: commit & push**

```bash
git add index.html
git commit -m "feat: replace saveLocalData with DataService event-based API (guest mode)"
git push origin main
```

---

## Task 3: DataService Firestore モード動作確認

> DataService の Firestore 実装は Task 2 で追加済み。このタスクは Firestore モードの動作テストと onAuthStateChanged の統合確認。

**Files:**
- Modify: `index.html`（Task 2 で追加した DataService の Firestore 実装を検証・修正）

**Interfaces:**
- Consumes: Task 2 の `DS`、Task 1 の `onAuthStateChanged`

---

- [ ] **Step 1: Firebase コンソールで事前設定（かずさん作業）**

以下を完了させてから次のステップへ進む:
1. Firebase コンソール → Authentication → Sign-in method → Google を有効化
2. 承認済みドメインに `kazutyon.github.io` を追加
3. Firebase コンソール → Firestore Database を作成（本番モードで開始）
4. Firestore → ルール タブに以下を貼り付けて公開:

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

---

- [ ] **Step 2: GitHub Pages にプッシュ済みのコードでログインテスト**

`https://kazutyon.github.io/Deli/` を開き:

1. 「👤 ゲスト」をタップ → Google ログイン
2. ログイン後、稼働を開始 → 完了
3. Firebase コンソール → Firestore → `users/{uid}/records/` に記録が保存されていることを確認
4. 売上を入力
5. Firestore → `users/{uid}/earnings/{dayKey}` に保存されていることを確認
6. 設定を変更
7. Firestore → `users/{uid}` ドキュメントに cfg フィールドが保存されていることを確認

---

- [ ] **Step 3: オフライン動作確認**

1. Chrome DevTools → Network → Offline に設定
2. 稼働記録を追加
3. Network を Online に戻す
4. Firestore コンソールで記録が同期されていることを確認

---

- [ ] **Step 4: ログアウト確認**

1. ログアウト
2. ゲストモードに戻り、前のユーザーのデータが表示されないことを確認
3. ゲストとして新しいデータを入力 → localStorage に保存されることを確認

---

- [ ] **Step 5: エラーがなければ commit（変更があれば）**

```bash
git add index.html
git commit -m "test: verify Firestore mode integration"
git push origin main
```

---

## Task 4: 移行フロー実装

**Files:**
- Modify: `index.html`（JS section）

**Interfaces:**
- Consumes: Task 1 の `#migration-banner` HTML、Task 2 の `DS.hasMigrationData()`、`DS.migrate()`、`DS.hasFirestoreData()`

---

- [ ] **Step 1: showMigrationBannerIfNeeded() 関数を追加**

Firebase Auth ブロックの末尾（`// ===== /Firebase Auth =====` の直前）に追加:

```js
async function showMigrationBannerIfNeeded() {
  if (!DS.hasMigrationData()) return;
  if (sessionStorage.getItem('migrationDismissed')) return;

  const banner = document.getElementById('migration-banner');
  const msg    = document.getElementById('migration-banner-msg');
  const btnOk  = document.getElementById('migration-btn-ok');
  const btnDel = document.getElementById('migration-btn-delete');
  const btnLater = document.getElementById('migration-btn-later');
  if (!banner) return;

  const hasCloud = await DS.hasFirestoreData();
  if (hasCloud) {
    msg.textContent = 'クラウドにも既存データがあります。ゲストデータをアカウントに追加しますか？（同じ日のデータは上書きされます）';
    btnOk.textContent = '追加する';
  } else {
    msg.textContent = 'ゲストとして記録したデータがあります。アカウントに引き継ぎますか？';
    btnOk.textContent = '引き継ぐ';
  }

  banner.style.display = 'block';

  btnOk.onclick = async function() {
    btnOk.disabled = true;
    btnOk.textContent = '移行中...';
    try {
      await DS.migrate();
      // migrate 内で clearLocalData 済み
      records  = await DS.loadRecords();
      earnings = await DS.loadEarnings();
      banner.style.display = 'none';
      showToast('データを引き継ぎました');
      render();
    } catch(e) {
      btnOk.disabled = false;
      btnOk.textContent = hasCloud ? '追加する' : '引き継ぐ';
      showToast('移行に失敗しました。もう一度お試しください: ' + e.message);
    }
  };

  btnDel.onclick = function() {
    if (!confirm('ゲストデータを削除します。元に戻せません。よろしいですか？')) return;
    DS.clearLocalData();
    banner.style.display = 'none';
    showToast('ゲストデータを削除しました');
  };

  btnLater.onclick = function() {
    sessionStorage.setItem('migrationDismissed', '1');
    banner.style.display = 'none';
  };
}
```

---

- [ ] **Step 2: 動作確認**

1. ゲストとして稼働記録を追加（localStorage にデータを作る）
2. Google ログイン
3. 移行バナーが表示されることを確認
4. 「引き継ぐ」を押す → Firestore に記録が移行されることを確認
5. ローカルの fdc_records が消えていることを確認（DevTools → Application → localStorage）
6. ログアウト → ゲストとして再入力 → 再ログイン → バナーが再表示されることを確認
7. 「あとで」を押す → 同一セッション中はバナーが出ないことを確認
8. ページをリロード → バナーが再表示されることを確認

---

- [ ] **Step 3: commit & push**

```bash
git add index.html
git commit -m "feat: add migration banner (localStorage -> Firestore)"
git push origin main
```

---

## Task 5: アカウント削除フロー

**Files:**
- Modify: `index.html`（設定パネル HTML + `doDeleteAccount()` は Task 1 で実装済み）

**Interfaces:**
- Consumes: Task 1 の `window.doDeleteAccount()`、Task 2 の `DS.deleteAllUserData()`

---

- [ ] **Step 1: 設定パネルにアカウント削除ボタンを追加**

設定パネルの閉じるボタン（`toggleSettings()` を呼ぶボタン）の直前に追加:

```html
      <div class="setting-row" style="margin-top:16px;border-top:1px solid #fecaca;padding-top:12px;">
        <button onclick="doDeleteAccount()"
          style="width:100%;padding:10px;background:#fef2f2;color:#dc2626;
                 border:1px solid #fecaca;border-radius:8px;font-size:13px;
                 font-weight:700;cursor:pointer;">
          🗑️ アカウントを削除
        </button>
      </div>
```

---

- [ ] **Step 2: 動作確認**

1. Google ログイン
2. 設定パネルを開く → 「アカウントを削除」ボタンが表示されることを確認
3. ボタンを押す → 確認ダイアログ → キャンセル → 何も起きないことを確認
4. （任意）削除を実行 → Firestore データが消えること、ゲストモードに戻ることを確認

---

- [ ] **Step 3: commit & push**

```bash
git add index.html
git commit -m "feat: add account deletion flow"
git push origin main
```

---

## 自己レビューチェック

### 仕様カバレッジ

| 仕様項目 | 対応タスク |
|----------|-----------|
| Google Sign-In のみ（メール/パスワード廃止） | Task 1 |
| ゲストモード（localStorage のみ） | Task 1, 2 |
| `enablePersistence()`（compat SDK） | Task 1 |
| `getRedirectResult()` | Task 1 |
| DataService イベント単位 API | Task 2 |
| `users/{uid}` に cfg 直接格納 | Task 2 |
| records サブコレクション + ISO 文字列 | Task 2 |
| earnings サブコレクション + dayKey | Task 2 |
| ログアウト時のメモリ破棄 | Task 1（onAuthStateChanged → DS.init(null)） |
| Firestore Security Rules | Task 3 |
| 移行バナー（引き継ぐ/削除/あとで） | Task 4 |
| バッチ移行・成功確認後に削除 | Task 4（DS.migrate()） |
| sessionStorage で「あとで」 | Task 4 |
| 二重登録防止（hasFirestoreData チェック） | Task 4 |
| アカウント削除 | Task 1（doDeleteAccount）, Task 5 |
| fdc_active は localStorage のみ | Task 2 |
| ヘッダーのゲスト表示 | Task 1 |
| showToast | Task 1 |

### 残っている作業（実装後）

- Firebase コンソールの事前設定（かずさん作業、Task 3 Step 1）
- GitHub Pages の動作確認（Google ログインは HTTPS 必須）
- 使用量アラートの設定

---

## セッション終了前チェック（作業AIへ）

各タスク完了後に必ず実施:
- [ ] `projects/food-delivery/LOG.md` に実施内容を追記
- [ ] `projects/food-delivery/CURRENT.md` の「次にやること」を更新
- [ ] `git push origin main` で GitHub Pages に反映済み
