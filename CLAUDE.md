# Personal preferences (all projects)

## Language
- 返答は日本語
- コード・コメント・変数名は英語

## Workflow
- 作業前に実装計画を提示し、承認を得てから着手する
- 仕様の曖昧さや不明点は計画時点で確認する（実装中に勝手に決めない）
- 完了後に何をしたかサマリーを出す
- テストが通っても、コミットはユーザーの承認後に行う

## Development approach
- 仕様を先に決め、それに沿ったテストを設計してから実装する
- コード変更後は必ずテストを実行して通過を確認する
- 不要な抽象化・将来のための設計はしない

## テスト方針

- 対象: ビジネスロジック（ドメインロジック・純粋関数など）
- 対象外: UI コンポーネント（`npm run dev` で目視確認）
- 外部サービス（Firebase・API 等）はモックで分離する

## Code style
- インデント：4スペース（言語問わず）
- コメントは WHY が非自明な場合のみ（WHAT は書かない）
- エラーハンドリングは境界（ユーザー入力・外部 API）のみ
- TypeScript strict mode を常に維持（`any` 禁止）
- UIテキストは日本語

## Architecture（Next.js + Firebase）
- Firestoreアクセスは専用レイヤーに集約し、コンポーネントから直接叩かない
- Firestoreのコレクションパス文字列は定数として一元管理する（タイポ防止）
- ドメインロジック（計算・集計など）は純粋関数として実装しテスト対象にする
- Firebase Admin SDK はサーバーサイド（APIルート）専用。クライアントバンドルに含めない
- Firestore の日時フィールドは `serverTimestamp()` で書き込む（クライアント時刻はタイムゾーンズレのリスクがある）
- Firestore 取得時は `FirestoreDataConverter` で型変換し、UI側で `.toDate()` を呼ばない設計にする
- **開発順序**: まずローカルデータで Context を実装し、UI・ロジックを確認してから Firebase 実装に切り替える。DBの問題でUIの確認が詰まるのを防ぐ。詳細は `docs/firebase-notes.md` の「ローカルファースト開発」を参照

- コード変更後は以下の順で確認してからコミットする:

  ```bash
  npx tsc --noEmit
  npm run test:run
  npm run build
  ```

## Git
- feature ブランチは `main` から作成する。命名規則: `feature/xxx`、`fix/xxx`
- コミットメッセージは「何をしたか」より「なぜしたか」を重視、日本語でよい
- PR タイトルは 70 文字以内
- PR は自分でマージしない。ユーザーがレビュー・マージする
- マージは squash merge を使う（`gh pr merge --squash --delete-branch`）

## GitHub CLI
- GitHub のリモート操作は `gh` CLI を使う（MCP は使わない）
- Private リポジトリ操作には `repo` スコープが必要。詳細は `docs/github-notes.md` を参照

```bash
# PR 作成（body テンプレートは docs/github-notes.md を参照）
gh pr create --title "タイトル" --body "..."

# PR 確認・CI チェック
gh pr list
gh pr view <番号> --comments
gh pr checks <番号>
gh pr diff <番号>

# Issue 一覧・作成
gh issue list
gh issue create --title "タイトル" --body "本文"

# 認証確認・再認証
gh auth status
gh auth login
```

GitHub Actions でワークフローに権限を明示する。

```yaml
# PR 作成・CI 確認のみ（リポジトリへの書き込み不要）
permissions:
  contents: read
  pull-requests: write

# リリース成果物のアップロード等、リポジトリへの書き込みが必要な場合のみ
permissions:
  contents: write
  pull-requests: write
```

## Output style
- 結論を先に書き、理由・背景を後に続ける
- コードブロックは積極的に使う
- ファイルに残すドキュメントでは太字（**）を多用しない

## iOS Safari 対応 / レイアウト（Next.js モバイルファースト）

詳細は `docs/nextjs-notes.md` を参照。

## Qiita
- 記事の公開は Qiita CLI を使う（`npx qiita preview` で確認後、`npx qiita publish` で公開）
- 記事ファイルは `public/` ディレクトリに配置する
