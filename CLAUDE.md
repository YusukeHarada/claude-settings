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

## Code style
- インデント：4スペース（言語問わず）
- コメントは WHY が非自明な場合のみ（WHAT は書かない）
- エラーハンドリングは境界（ユーザー入力・外部 API）のみ
- TypeScript strict mode を常に維持（`any` 禁止）

## Architecture（Next.js + Firebase）
- Firestoreアクセスは `src/lib/firestore/` に集約し、コンポーネントから直接叩かない
- ドメインロジック（計算・集計など）は `src/lib/utils/` に純粋関数として実装しテスト対象にする
- コード変更後は以下の順で確認してからコミットする:

  ```bash
  npx tsc --noEmit
  npm run test:run
  npm run build
  ```

## iOS Safari 対応（Next.js PWA）

- `viewport` は `app/layout.tsx` で明示的にエクスポートする（未設定だと iOS Safari が 980px ビューポートで描画することがある）
- `<input>` / `<select>` / `<textarea>` の `font-size` は 16px（`text-base`）以上にする（14px 未満だとフォーカス時にオートズームし、フォーカスを外しても戻らないことがある）
- `overflow-x: hidden` を `html`/`body` に設定するとピンチズームが完全に無効になる。横オーバーフロー制御には `overflow-x: clip` を使う
- `overscroll-behavior-x: none` を `html`/`body` に設定して横ドリフトを防止する
- `viewportFit: "cover"` は `env(safe-area-inset-*)` の padding を実装しない限り設定しない（有害）
- flex コンテナ内の `<Link>` には `block w-full` を付ける（デフォルトの `display: inline` だと iOS Safari で幅計算が狂う）
- flex 内で `truncate` を使う要素には `min-w-0` を付ける

## レイアウトパターン（モバイルファースト）

- `html`/`body` を `h-dvh` にしてビューポート高さを固定する
- `<main>` は `overflow-y-auto` のスクロールコンテナにし、`min-h-0` を必ず付ける（flex child のデフォルト `min-height: auto` を打ち消すため）
- BottomNav は `fixed bottom-0 sm:hidden` で配置し、`<main>` に `pb-16` を付けてコンテンツが隠れないようにする
- `getUserMedia`（バーコードスキャン等）は iOS Safari の制約でユーザー操作の同期コンテキストで呼ぶ必要がある（非同期に遅延させると失敗する）

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

## Output style
- 結論を先に書き、理由・背景を後に続ける
- コードブロックは積極的に使う
- ファイルに残すドキュメントでは太字（**）を多用しない

## Qiita
- 記事の公開は Qiita CLI を使う（`npx qiita preview` で確認後、`npx qiita publish` で公開）
- 記事ファイルは `public/` ディレクトリに配置する
