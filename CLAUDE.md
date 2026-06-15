# Personal preferences (all projects)

## Language
- 返答は日本語
- コード・コメント・変数名は英語

## Workflow
- 作業前に実装計画を提示し、承認を得てから着手する
- 仕様の曖昧さや不明点は計画時点で確認する（実装中に勝手に決めない）
- 完了後に何をしたかサマリーを出す

## Development approach
- 仕様を先に決め、それに沿ったテストを設計してから実装する
- コード変更後は必ずテストを実行して通過を確認する
- 不要な抽象化・将来のための設計はしない

## テスト方針

- 対象: ビジネスロジック（`src/lib/`）
- 対象外: UI コンポーネント（`npm run dev` で目視確認）
- 外部サービス（Firebase・API 等）は `__mocks__/` でモック化

## Code style
- インデント：4スペース（言語問わず）
- コメントは WHY が非自明な場合のみ（WHAT は書かない）
- エラーハンドリングは境界（ユーザー入力・外部 API）のみ
- TypeScript strict mode を常に維持（`any` 禁止）
- UIテキストは日本語

## Architecture
- Firestoreアクセスは専用レイヤーに集約し、コンポーネントから直接叩かない
- Firestoreのコレクションパス文字列は定数として一元管理する（タイポ防止）
- ドメインロジック（計算・集計など）は純粋関数として実装しテスト対象にする
- Firebase Admin SDK はサーバーサイド（APIルート）専用。クライアントバンドルに含めない
- コード変更後は以下の順で確認してからコミットする:

  ```bash
  npx tsc --noEmit
  npm run test:run
  npm run build
  ```

## Git
- feature ブランチは `main` から作成する
- コミットメッセージは「何をしたか」より「なぜしたか」を重視、日本語でよい
- PR タイトルは 70 文字以内
- PR は自分でマージしない。ユーザーがレビュー・マージする

## GitHub CLI
- GitHub のリモート操作は `gh` CLI を使う（MCP は使わない）

```bash
# PR 作成
gh pr create --title "タイトル" --body "本文"

# PR 一覧
gh pr list

# Issue 一覧・作成
gh issue list
gh issue create --title "タイトル" --body "本文"

# 認証確認・再認証
gh auth status
gh auth login
```

- 初回セットアップ時は `repo` スコープで認証する（privateリポジトリを操作するために必要）
- すべての操作は YusukeHarada 名義として GitHub に記録される

### Private リポジトリの注意点

`public_repo` スコープでは読み書きできない。必ず `repo` スコープを使う。

| スコープ | Public | Private |
|---|---|---|
| `public_repo` | 読み書き可 | 不可 |
| `repo` | 読み書き可 | 読み書き可 ✓ |

```bash
gh auth status              # 現在のスコープを確認（'repo' が含まれていれば OK）
gh auth refresh --scopes repo  # スコープを追加して再認証
```

GitHub Actions の無料枠はプランによって異なる（GitHub Free: 月 2,000 分）。
使用量は GitHub → Settings → Billing & plans → Actions で確認できる。

GitHub Actions で PR 作成・書き込みが必要なワークフローには権限を明示する。

```yaml
permissions:
  contents: write
  pull-requests: write
```

## Output style
- 結論を先に書き、理由・背景を後に続ける
- コードブロックは積極的に使う
- ファイルに残すドキュメントでは太字（**）を多用しない

## iOS Safari 対応（Next.js PWA）

詳細は `docs/nextjs-notes.md` を参照。主な注意点：

- `<input>` / `<select>` / `<textarea>` の `font-size` は 16px 以上（未満だとオートズーム）
- `overflow-x: clip` を使う（`hidden` だとピンチズームが無効になる）
- `overscroll-behavior-x: none` で横ドリフトを防止する
- `viewportFit: "cover"` は `env(safe-area-inset-*)` を実装しない限り設定しない
- flex 内の `<Link>` に `block w-full`、`truncate` 要素に `min-w-0`
- `getUserMedia` はユーザー操作の同期コンテキストで呼ぶ

## レイアウトパターン（Next.js モバイルファースト）

詳細は `docs/nextjs-notes.md` を参照。主な注意点：

- `html`/`body` を `h-dvh` にしてビューポート高さを固定する
- `<main>` は `overflow-y-auto` のスクロールコンテナにし、`min-h-0` を必ず付ける
- BottomNav は `fixed bottom-0 sm:hidden` で配置し、`<main>` に `pb-16` を付ける
- `viewport` は `app/layout.tsx` で明示的にエクスポートする（Next.js 15 以降の要件）

## Qiita
- 記事の公開は Qiita CLI を使う（`npx qiita preview` で確認後、`npx qiita publish` で公開）
- 記事ファイルは `public/` ディレクトリに配置する
