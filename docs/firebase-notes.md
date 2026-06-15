# Firebase 知見集

ハマったこと・注意事項を症状→原因→解決策の形式で記録する。
新しい知見は開発中に随時追記すること。

---

## Authentication

### `initializeAuth` で `signInWithPopup` が `auth/argument-error` で失敗する

**症状**
`initializeAuth` を使うと `signInWithPopup` が `auth/argument-error` で失敗する。

**原因**
`getAuth` はデフォルトで `browserPopupRedirectResolver` を含むが、`initializeAuth` は含まない。

**解決策**
`initializeAuth` を使う場合は `browserPopupRedirectResolver` を明示的に渡す。

```typescript
import { initializeAuth, browserPopupRedirectResolver } from 'firebase/auth';

const auth = initializeAuth(app, {
  persistence: browserLocalPersistence,
  popupRedirectResolver: browserPopupRedirectResolver,
});
```

---

### iOSでログイン後に「missing initial state」エラーが発生する

**症状**
iOS Safari でログインすると `missing initial state` エラーが出てログインできない。

**原因**
`signInWithRedirect` を使うと、iOS Safari の ITP がリダイレクト後に `sessionStorage` を消去するため。

**解決策**
`signInWithRedirect` を使わず `signInWithPopup` に統一する。

---

### ページ読み込み時にログイン状態が一瞬 null になる

**症状**
リロード直後にログイン済みユーザーが一瞬 null になり、未ログイン扱いの画面がチラつく。

**原因**
`onAuthStateChanged` のコールバック外で `setLoading(false)` を呼んでいるため、認証状態の解決前に描画が走る。

**解決策**
`setLoading(false)` はコールバック内で呼ぶ。

```typescript
useEffect(() => {
  const unsubscribe = onAuthStateChanged(auth, (user) => {
    setUser(user);
    setLoading(false); // ← コールバック内で解決する
  });
  return unsubscribe;
}, []);
```

---

### Next.js で「Firebase App already exists」エラーが発生する

**症状**
Next.js の開発サーバーでホットリロードのたびに `Firebase App named '[DEFAULT]' already exists` エラーが出る。

**原因**
HMR や SSR で `initializeApp` が複数回呼ばれるため。

**解決策**
`getApps()` ガードを付けて二重初期化を防ぐ。

```typescript
const app = getApps().length ? getApp() : initializeApp(firebaseConfig);
```

---

## Firestore

### サブコレクションへのアクセスがセキュリティルールで弾かれる

**症状**
親コレクションのルールを設定したのにサブコレクションへのアクセスが拒否される。

**原因**
Firestore のセキュリティルールは親から引き継がれない。サブコレクションには別途 `match` が必要。

**解決策**
サブコレクションに明示的にルールを追加する。

```javascript
match /items/{itemId} {
  allow read, write: if request.auth.uid == resource.data.userId;

  match /history/{historyId} {
    allow read, write: if request.auth.uid == get(/databases/$(database)/documents/items/$(itemId)).data.userId;
  }
}
```

---

### `create` 時にセキュリティルールでエラーが発生する

**症状**
ドキュメント作成時に `resource.data` を参照するルールでエラーになる。

**原因**
`create` 時は `resource` が `null` のため `resource.data` を参照できない。

**解決策**
`create` と `update/delete` でルールを分離し、作成時は `request.resource.data` を使う。

```javascript
match /items/{itemId} {
  allow create: if request.auth.uid == request.resource.data.userId;
  allow read, update, delete: if request.auth.uid == resource.data.userId;
}
```

---

### CollectionGroup クエリでセキュリティルールが通らない

**症状**
CollectionGroup クエリを使うとセキュリティルールで弾かれる。

**原因**
親ドキュメントを経由せずにクエリするため、親の `userId` を参照できない。

**解決策**
サブコレクションのドキュメントに `userId` を直接保存する。

---

### 複数ドキュメントの同時書き込みで不整合が起きる

**症状**
複数タブや同時アクセス時にデータが壊れる。

**原因**
複数ドキュメントを個別に書き込むと、途中で失敗した場合に中途半端な状態になる。

**解決策**
複数ドキュメントの同時書き込みは必ずトランザクションを使う。

---

### UI 全体で `.toDate()` の呼び忘れによるランタイムエラーが散発する

**症状**
Firestore から取得した日時を使う箇所でランタイムエラーが散発する。

**原因**
Firestore の `Timestamp` 型をそのまま UI に渡しており、`.toDate()` の呼び忘れが発生する。

**解決策**
converter で取得時に ISO 文字列に変換し、UI 側では `.toDate()` を呼ばない設計にする。

---

### `onSnapshot` リスナーが重複して不要な読み取りが発生する

**症状**
Firestore の読み取りコストが想定より高く、メモリリークも発生する。

**原因**
各コンポーネントで直接 `onSnapshot` を呼んでいるため、リスナーが重複している。

**解決策**
`onSnapshot` リスナーは Context で一度だけ張り、各コンポーネントは Context 経由でデータを受け取る。

---

### セキュリティルールを編集しても反映されない

**症状**
Firestore コンソールでルールを編集したのに動作が変わらない。

**原因**
ローカルファイルを編集しただけでデプロイしていない。

**解決策**
編集後は必ずデプロイする。

```bash
firebase deploy --only firestore:rules
```
