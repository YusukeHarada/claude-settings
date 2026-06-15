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

---

## 各ファイルの役割

### CLAUDE.md

プロジェクト横断で共通するルールをまとめたガイドライン。言語設定・ワークフロー・コードスタイル・Git 運用・アーキテクチャ方針・iOS Safari 対応などを記載している。

このファイルの内容は `~/.claude/CLAUDE.md`（グローバル設定）に反映することで全プロジェクトに適用される。グローバルの内容と乖離してきたら同期する。

### docs/firebase-notes.md

Firebase（Authentication / Firestore / Cloud Storage）でハマったパターンを
「症状→原因→解決策」の形式で記録したもの。

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

---

## 使い方

### 新しいプロジェクトを始めるとき

`CLAUDE.md` の内容は Claude Code が自動参照するため、特別な操作は不要。
新プロジェクトの `CLAUDE.md` を書くときは、以下を参考にする。

1. `docs/` の該当ノートを開き、使う技術スタックに関するハマりパターンを確認する
2. プロジェクト固有の設定（データモデル・ディレクトリ構成・画面構成など）はそのプロジェクトの `CLAUDE.md` に書く
3. 全プロジェクト共通のルールはここにだけ書き、各プロジェクトに重複させない

### 開発中にハマったとき

解決したら、同じ問題で次も時間を使わないよう `docs/` に追記する。

- Firebase 関連 → `docs/firebase-notes.md`
- Next.js 関連 → `docs/nextjs-notes.md`
- 新しい技術領域 → `docs/xxx-notes.md` を新規作成

追記形式は「症状→原因→解決策」に統一する（既存ファイルを参照）。

### 複数プロジェクトに共通するルールが見つかったとき

特定のプロジェクトの `CLAUDE.md` にしか書かれていないルールで、
他のプロジェクトでも使えると判断したものは、このリポジトリの
`CLAUDE.md` または `docs/` に移植する。

移植の判断基準:
- 2つ以上のプロジェクトで同じパターンが登場した
- 技術スタックに依存せず、広く適用できる

---

## 運用ルール

- 変更はすべて feature ブランチで行い、PR を出してレビュー後にマージする
- PR は自分でマージしない
- `docs/` への追記は既存の内容と重複しないことを確認してから行う
