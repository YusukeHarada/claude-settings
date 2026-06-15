# claude-settings

開発を通じて得たナレッジを整理・集約し、今後のすべてのプロジェクトで再活用するためのリポジトリ。

---

## ファイル構成

```
CLAUDE.md                  # 全プロジェクト共通の開発ガイドライン（Claude Code が自動参照）
docs/
  firebase-notes.md        # Firebase のハマりパターン・注意事項
  nextjs-notes.md          # Next.js のハマりパターン・注意事項
```

## 各ファイルの役割

### CLAUDE.md

Claude Code が全プロジェクトで自動参照するガイドライン。言語設定・ワークフロー・コードスタイル・Git 運用など、プロジェクト横断で共通するルールをまとめている。

### docs/firebase-notes.md

Firebase（Authentication / Firestore / Cloud Storage）でハマったパターンを「症状→原因→解決策」の形式で記録したもの。新しい知見は開発中に随時追記する。

主なトピック:
- Authentication: `signInWithPopup` の使い方、iOS Safari 対応、セッション管理
- Firestore: セキュリティルール、複合クエリ、`undefined` 書き込みエラー、トランザクション
- Cloud Storage: Admin SDK 経由の書き込みパターン
- CI/CD: 環境変数の空文字列問題

### docs/nextjs-notes.md

Next.js（App Router）＋ PWA 開発でハマったパターンを「症状→原因→解決策」の形式で記録したもの。

主なトピック:
- レイアウト: `sticky` が効かない原因、`min-h-0`、スクロールコンテナの設計
- iOS Safari: オートズーム、横スクロール、safe-area-inset、`getUserMedia`
- PWA: `viewport` の明示的エクスポート

## 運用ルール

- 新しい知見は `docs/` 以下の該当ファイルに追記する
- CLAUDE.md の変更・docs の追記はいずれも PR を出してレビュー後にマージする
- PR は自分でマージしない
