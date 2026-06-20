# {プロジェクト名} — CLAUDE.md

## プロジェクト概要

{1〜3行でアプリの目的・対象ユーザーを説明する}

---

## 開発コマンド

```bash
npm run dev      # 開発サーバー起動
npm run build    # 本番ビルド
npm run lint     # ESLint
npm test         # ユニットテスト（Vitest）
npm run test:e2e # E2E テスト（Playwright）
```

---

## 技術スタック

| 役割 | ライブラリ / サービス |
|---|---|
| フレームワーク | Next.js {バージョン}（App Router） |
| UI | React {バージョン} + TypeScript + Tailwind CSS v4 |
| フォーム | react-hook-form + zod |
| 認証 | Firebase Authentication（Google） |
| DB | Cloud Firestore |
| ホスティング | Vercel |
| テスト | Vitest + React Testing Library |

---

## ディレクトリ構成

```
src/
├── app/
│   ├── (auth)/          # 認証不要ページ（ログイン等）
│   └── (app)/           # 認証必須ページ
├── components/
│   ├── ui/              # 汎用 UI コンポーネント
│   └── {機能名}/        # 機能別コンポーネント
├── hooks/               # カスタムフック
├── lib/
│   ├── firebase/        # Firebase 初期化・Firestore アクセス層
│   └── utils/           # ドメインロジック（純粋関数）
├── types/               # 型定義
└── contexts/            # React Context
```

---

## 環境変数

`.env.local` に以下が必要（Firebase Console → プロジェクト設定から取得）:

```
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=
```

{Admin SDK を使う場合は追加}
```
FIREBASE_SERVICE_ACCOUNT_KEY=   # JSON を文字列化したもの
```

---

## データモデル（Firestore）

```
{コレクション名}/{ドキュメントID}
  {フィールド名}: {型}   # {説明}
```

### 型定義

```typescript
type {ModelName} = {
  id: string
  userId: string
  // ...
  createdAt: string    // ISO 文字列（Timestamp は converter で変換）
}
```

---

## 画面構成

| パス | 画面名 | 概要 |
|---|---|---|
| `/` | {画面名} | {説明} |
| `/{path}` | {画面名} | {説明} |

---

## アーキテクチャ上の決定事項

{このプロジェクト固有の設計判断を記録する。「なぜそう決めたか」が重要}

### 例
- **認証方式**: {セッションクッキー or クライアント Auth の理由}
- **データ設計**: {正規化/非正規化の判断とその理由}
- **{その他の判断}**: {理由}

---

## ビジネスロジック

{このプロジェクト固有の計算・判定ロジックを記録する}

- `src/lib/utils/{ファイル名}.ts` — {ロジックの説明}

---

## 注意事項・ハマりパターン

{このプロジェクト固有のハマりパターンを「症状→原因→解決策」の形式で記録する}
{汎用的な知見は ~/develop/claude-settings/docs/ に移植すること}

### {症状の一言タイトル}

**症状**
{どんな問題が起きたか}

**原因**
{なぜ起きたか}

**解決策**
{どう直したか}

---

## Claudeへの制約

{このプロジェクト固有の「やってはいけないこと」を記録する}

- {例: 本番DBに直接書き込まない}
- {例: xxx パッケージを追加しない（理由）}
- {例: yyy ファイルを直接編集しない（自動生成ファイルのため）}

---

## スコープ管理

### 実装済み
- [ ] {機能名}

### 未実装（将来対応）
- {機能名}: {理由・優先度}
