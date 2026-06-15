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

---

### `where()` + `orderBy()` の複合クエリでデータが表示されない

**症状**
`onSnapshot` がエラーコールバックを起動し、データが空配列になる。コンソールには「インデックスが必要」旨のエラーが出る。

**原因**
`where('userId', '==', x)` と `orderBy('createdAt', 'desc')` を組み合わせると Firestore の複合インデックスが必要になる。インデックス未作成の場合、クエリが例外を投げる。

**解決策**
`orderBy` をクエリから外し、クライアントサイドでソートする。

```typescript
// NG: 複合インデックスが必要
const q = query(
  collection(db, 'items'),
  where('userId', '==', userId),
  orderBy('createdAt', 'desc')
)

// OK: クライアントでソート
const q = query(collection(db, 'items'), where('userId', '==', userId))
const snap = await getDocs(q)
const items = snap.docs
  .map(d => ({ id: d.id, ...d.data() }))
  .sort((a, b) => b.createdAt.toMillis() - a.createdAt.toMillis())
```

---

### optional フィールドが `undefined` のとき `addDoc`/`updateDoc` が失敗する

**症状**
`addDoc` や `updateDoc` が "Unsupported field value: undefined" エラーで失敗する。

**原因**
Firestore は `undefined` 値を受け付けない。TypeScript の optional フィールド（`memo?: string` 等）が未入力の場合、`undefined` がそのまま渡される。

**解決策 A（推奨）**: `initializeFirestore` の `ignoreUndefinedProperties` オプションを有効にする。

```typescript
import { initializeFirestore, getFirestore } from 'firebase/firestore'

function createFirestore() {
  try {
    return initializeFirestore(app, { ignoreUndefinedProperties: true })
  } catch {
    return getFirestore(app) // 初期化済みの場合
  }
}
export const db = createFirestore()
```

**解決策 B**: 書き込み前に `undefined` キーを除去するヘルパーを使う。

```typescript
function omitUndefined(obj: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(Object.entries(obj).filter(([, v]) => v !== undefined))
}

await addDoc(collection(db, 'items'), omitUndefined({ ...data, userId }))
```

---

## Authentication（応用）

### `signInWithRedirect` + `getRedirectResult` の組み合わせは信頼性が低い

**症状**
ログイン後のリダイレクトが完了しない、またはログインページをループし続ける。

**原因**
`getRedirectResult` はリダイレクト直後のページロードでのみ結果を返す。Next.js の SSR やナビゲーションのタイミングによって呼び忘れが発生する。

**解決策**
`getRedirectResult` を廃止し、`onAuthStateChanged` でユーザー状態の変化を待つ。

```typescript
useEffect(() => {
  const unsubscribe = onAuthStateChanged(auth, async (user) => {
    if (!user) { setLoading(false); return; }
    // ログイン済みなら次の処理へ
    router.push('/')
  })
  return () => unsubscribe()
}, [router])
```

---

### `signInWithRedirect` は iOS Safari だけでなく Android でも問題が出る

**症状**
Android の一部ブラウザやアプリ内 WebView でもログインが失敗する。

**原因**
`signInWithRedirect` は iOS Safari の ITP 以外にも Android WebView でストレージへのアクセス制限がかかる環境がある。

**解決策**
`signInWithPopup` に統一する。iOS Safari でも Android でも `signInWithPopup` の方が動作が安定する。プラットフォーム分岐は不要。

```typescript
// プラットフォームで分岐しない。常に signInWithPopup を使う
await signInWithPopup(auth, new GoogleAuthProvider())
```

---

### `authDomain` を `window.location.hostname` に変えると localhost で壊れる

**症状**
本番環境での authDomain ミスマッチを解消しようとして `authDomain` を `window.location.hostname` に変えると、`localhost` でのローカル開発時にログインが失敗する。

**原因**
`window.location.hostname` が `'localhost'` になり、Firebase Console に登録された authDomain と一致しなくなる。

**解決策**
`localhost` の場合はそのまま環境変数の値を使う。

```typescript
authDomain:
  typeof window !== 'undefined' && window.location.hostname !== 'localhost'
    ? window.location.hostname
    : process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN!,
```

ただし、authDomain ミスマッチが起きる場合はカスタムドメインで Firebase Hosting を使うか、`authDomain` は常に環境変数のまま使うのが最もシンプル。

---

## Cloud Storage

### Storage への書き込みをクライアントから直接行うとセッション認証と競合する

**症状**
セッションクッキー認証（Admin SDK）を使うアプリでクライアントから Storage に直接書き込もうとするとルール設定が複雑になる。

**原因**
クライアントからの Storage 書き込みは Firebase Auth の ID トークンで認証するが、セッションクッキー認証を使うアプリではクライアントに常に有効な ID トークンが存在しないことがある。

**解決策**
Storage の書き込みは Admin SDK 経由の API ルートに限定し、Storage ルールでクライアントからの書き込みを禁止する。

```javascript
// storage.rules: クライアントからの write を全て禁止
match /images/{userId}/{allPaths=**} {
  allow read: if true;
  allow write: if false; // Admin SDK のみ書き込み可
}
```

```typescript
// app/api/upload/route.ts: Admin SDK で書き込む
import { adminStorage } from '@/lib/firebase/admin'

export async function POST(request: NextRequest) {
  // セッションクッキーを検証してから Storage に書き込む
  const decoded = await adminAuth().verifySessionCookie(sessionCookie, true)
  const bucket = adminStorage().bucket()
  await bucket.file(path).save(buffer, { contentType: 'image/webp' })
  await bucket.file(path).makePublic()
}
```

---

## CI/CD

### CI 環境で Firebase 環境変数が空文字列になると初期化エラー

**症状**
CI ビルドで `auth/invalid-api-key` エラーが発生する。ローカルでは問題ない。

**原因**
CI では環境変数が `undefined` ではなく `''`（空文字列）で渡されることがある。`??` は `null`/`undefined` のみフォールバックし、空文字列はスルーするため、空の値が Firebase に渡される。

**解決策**
フォールバックに `??` でなく `||` を使う。

```typescript
// NG: '' はフォールバックされない
apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY ?? 'placeholder-api-key',

// OK: '' もフォールバックされる
apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY || 'placeholder-api-key',
```
