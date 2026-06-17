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
  persistence: [indexedDBLocalPersistence, browserLocalPersistence],
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
`signInWithRedirect` を使わず `signInWithPopup` に統一する。iOS Safari だけでなく Android の一部ブラウザや WebView でも `signInWithRedirect` はストレージ制限によって失敗することがある。プラットフォームで分岐せず、常に `signInWithPopup` を使うのが最もシンプルで安全。

```typescript
// プラットフォームで分岐しない。常に signInWithPopup を使う
await signInWithPopup(auth, new GoogleAuthProvider())
```

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
    router.push('/')
  })
  return () => unsubscribe()
}, [router])
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

authDomain ミスマッチが起きる根本原因はカスタムドメインの設定不足なので、Firebase Hosting でカスタムドメインを設定するか、`authDomain` は常に環境変数のまま使うのが最もシンプル。

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

**解決策 A（データ量が少ない場合）**: `orderBy` をクエリから外し、クライアントサイドでソートする。インデックス作成が不要で即動く。

```typescript
const q = query(collection(db, 'items'), where('userId', '==', userId))
const snap = await getDocs(q)
const items = snap.docs
  .map(d => ({ id: d.id, ...d.data() }))
  .sort((a, b) => b.createdAt.toMillis() - a.createdAt.toMillis())
```

**解決策 B（データ量が多い場合）**: Firestore コンソールで複合インデックスを作成する。エラーメッセージに記載されているリンクから直接作成できる。

```
Firebase Console → Firestore → インデックス → 複合インデックスを追加
フィールド: userId (昇順) + createdAt (降順)
```

データ量が少ないうちはAで十分。件数が増えてクライアントソートが遅くなったらBに移行する。

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

## Next.js + Firebase 統合

### Next.js App Router で Firebase Client SDK が SSR 時に初期化エラーになる

**症状**
`npm run build` や SSR 時に `auth/invalid-api-key` や `INTERNAL ASSERTION FAILED: Expected a class definition` エラーが出る。ローカルの `npm run dev` では問題ない。

**原因**
Next.js App Router は `"use client"` を付けたコンポーネントもビルド時にサーバー側でレンダリングする。Firebase Client SDK はブラウザ API（`window`、`indexedDB` 等）に依存しているため、Node.js 環境で実行すると内部アサーションが失敗する。

**解決策**
Firebase に依存するコンポーネントを `dynamic(() => import(...), { ssr: false })` でラップし、サーバー上での実行を防ぐ。このとき呼び出し元のコンポーネントも `"use client"` が必要（Server Component では `ssr: false` を使えない）。

```typescript
// app/(app)/layout.tsx
"use client"; // ← dynamic ssr:false を使うために必要

import dynamic from "next/dynamic";

const AppClientLayout = dynamic(
  () => import("@/components/layout/AppClientLayout"),
  { ssr: false }
);

export default function AppLayout({ children }: { children: React.ReactNode }) {
  return <AppClientLayout>{children}</AppClientLayout>;
}
```

```typescript
// components/layout/AppClientLayout.tsx
"use client";

// Firebase の Context Provider や Auth Guard などをここに集約する
export default function AppClientLayout({ children }) {
  return (
    <AuthProvider>
      <AppGuard>{children}</AppGuard>
    </AuthProvider>
  );
}
```

---

### `onSnapshot` クエリが `where` 句なしだとセキュリティルールで弾かれる

**症状**
ユーザーはログイン済みなのに `onSnapshot` のエラーコールバックが "Missing or insufficient permissions" で発火し、データが表示されない。

**原因**
Firestore のセキュリティルールが `resource.data.userId == request.auth.uid` で保護されている場合、コレクション全体を対象にしたクエリ（`where` なし）は「このクエリは権限のないドキュメントを返す可能性がある」と判定されて拒否される。ドキュメントを個別に読んだとき通るルールでも、クエリレベルでは別評価になる。

**解決策**
`onSnapshot` のクエリには必ず `where("userId", "==", userId)` を付けて、セキュリティルールが評価できる形に絞り込む。

```typescript
// NG: コレクション全体を読もうとしてルールに弾かれる
const q = collection(db, "tasks");

// OK: userId で絞り込めばルールが通る
const q = query(
  collection(db, "tasks"),
  where("userId", "==", userId)
);

onSnapshot(q, (snap) => { /* ... */ });
```

---

### `onSnapshot` のエラーが無音で消えてデータが表示されない

**症状**
`onSnapshot` を設定してもデータが表示されず、コンソールにも何も出ない。UI もエラー表示のままにならない。

**原因**
`onSnapshot` の第 2 引数（エラーコールバック）を省略すると、セキュリティルールエラーやクエリエラーが無音で捨てられる。

**解決策**
エラーコールバックを必ず渡し、Context の `error` state に反映して UI でも表示する。

```typescript
// NG: エラーコールバックなし → エラーが消える
onSnapshot(q, (snap) => {
  setTasks(snap.docs.map((d) => ({ id: d.id, ...d.data() } as Task)));
});

// OK: エラーコールバックを渡して state に反映する
onSnapshot(
  q,
  (snap) => {
    setTasks(snap.docs.map((d) => ({ id: d.id, ...d.data() } as Task)));
    setLoading(false);
  },
  (err) => {
    setError(err.message);
    setLoading(false);
  }
);
```

エラーを Context に持たせて UI に表示することで、問題の発見が早くなる。

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

---

## ローカルファースト開発

### Firebase を後回しにしてUI・ロジックを先に固める

**なぜやるか**
Firebase の設定ミス・セキュリティルール・`onSnapshot` のエラー処理など、DB 起因の問題がUIの確認を詰まらせる。ロジックとUIが正しくても「DBのせいで動かない」状態になりやすいため、まずローカルデータで動作確認してから Firebase に切り替える。

**前提: Context がインターフェースになっている**

コンポーネントが Context しか触らない設計（推奨）であれば、Context の中身を差し替えるだけで切り替えられる。

```
コンポーネント → Context（差し替えポイント）→ Firebase / ローカルデータ
```

**ローカル版 Context の実装**

`onSnapshot` の代わりに `useState` でデータを管理する。インターフェース（`tasks`, `loading`, `error`）は Firebase 版と同じに保つ。

```typescript
// contexts/TasksContext.local.tsx
"use client";

import { createContext, useContext, useState } from "react";
import type { Task } from "@/types/task";

type TasksState = { tasks: Task[]; loading: boolean; error: string | null };
const TasksContext = createContext<TasksState>({ tasks: [], loading: false, error: null });

const SAMPLE_TASKS: Task[] = [
    {
        id: "1",
        userId: "local",
        title: "サンプルタスク",
        description: "",
        completed: false,
        priority: "medium",
        categoryId: null,
        dueDate: null,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
    },
];

export function LocalTasksProvider({ children }: { children: React.ReactNode }) {
    const [tasks] = useState<Task[]>(SAMPLE_TASKS);
    return (
        <TasksContext.Provider value={{ tasks, loading: false, error: null }}>
            {children}
        </TasksContext.Provider>
    );
}

export function useTasks() {
    return useContext(TasksContext);
}
```

**切り替え方法**

環境変数で Provider を切り替える。`.env.local` に `NEXT_PUBLIC_DATA_SOURCE=local` を追加するだけで有効になる。

```typescript
// components/layout/AppClientLayout.tsx
import { TasksProvider } from "@/contexts/TasksContext";
import { LocalTasksProvider } from "@/contexts/TasksContext.local";

const Tasks = process.env.NEXT_PUBLIC_DATA_SOURCE === "local"
    ? LocalTasksProvider
    : TasksProvider;

export default function AppClientLayout({ children }) {
    return (
        <AuthProvider>
            <Tasks>
                {children}
            </Tasks>
        </AuthProvider>
    );
}
```

```bash
# .env.local
NEXT_PUBLIC_DATA_SOURCE=local   # ローカルデータで動作確認
# NEXT_PUBLIC_DATA_SOURCE=      # コメントアウトで Firebase に切り替え
```

**開発フロー**

1. ローカル版 Context を書いてサンプルデータを入れる
2. UI・ロジック・バリデーションを確認する（Firebase 設定不要）
3. 問題なければ Firebase 版 Context を実装して切り替える
4. Firebase 特有の問題（セキュリティルール・`onSnapshot` エラー処理）をここで対処する
