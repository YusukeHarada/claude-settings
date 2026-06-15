# Qiita 執筆・公開ワークフロー

PR のマージを「公開スイッチ」とした運用。GitHub Actions が自動で `npx qiita publish` を実行する。

---

## 執筆の開始

```bash
./qnew.sh "記事のタイトル"
```

以下を自動処理する:
1. `main` を最新化
2. `article/yyyymmdd_title` ブランチを作成
3. `private: false` で記事ファイルを作成
4. リモートへ Push

---

## 執筆・確認（Draft PR）

GitHub 上で **Draft Pull Request** を作成し、執筆を進める。

```bash
npx qiita preview   # http://localhost:8888 でプレビュー確認
git add public/記事ファイル.md
git commit -m "コミットメッセージ"
git push
```

この段階では Qiita に一切投稿・更新されない。

---

## 公開（PR をマージ）

記事が完成したら GitHub 上で PR を **Merge** する。

- `main` ブランチへのマージを検知して GitHub Actions が起動
- `npx qiita publish` が実行され、Qiita に一般公開記事として投稿される

---

## マージ後の整理

```bash
git checkout main
git pull origin main
git branch -d [作業ブランチ名]
```

---

## 公開済み記事の修正

### 方法1: PR 経由（推奨）

```bash
git checkout main && git pull origin main
git checkout -b fix/記事名-修正内容

# public/ 以下のファイルを編集
npx qiita preview   # 確認

git add public/対象ファイル.md
git commit -m "記事名: 〇〇を修正"
git push origin fix/記事名-修正内容
# → GitHub 上で PR を作成してマージ → GitHub Actions が Qiita に反映
```

### 方法2: main へ直接 push（軽微な修正のみ）

```bash
git checkout main && git pull origin main
# ファイルを編集
git add public/対象ファイル.md
git commit -m "記事名: typo修正"
git push origin main
# → push をトリガーに GitHub Actions が Qiita へ反映（約1〜2分）
```

記事ファイルに `id:` フィールドが入っていれば、既存記事の更新として処理される。

---

## Qiita と GitHub の同期

Qiita Web 上で記事を削除した場合は、速やかに同期する。

```bash
npx qiita pull   # Qiita の最新状態をローカルに取得
# 該当ファイルを削除してコミット・Push
```

---

## 運用ルール

| 項目 | 内容 |
|---|---|
| 公開設定 | 全記事デフォルト `private: false`（一般公開） |
| 公開トリガー | GitHub 上での PR マージ |
| 直接 push | 軽微な修正のみ。新規記事は必ず PR 経由 |
| 記事削除 | Qiita Web で削除後、`npx qiita pull` でローカルも同期 |

---

## フロントマター仕様

```yaml
title: ''               # 記事のタイトル
tags:
  - ''                  # タグ（最大5つ）
private: false          # true: 限定共有 / false: 公開
updated_at: ''          # 投稿時に自動更新
id: null                # 投稿時に自動設定（更新時はここで記事を識別）
organization_url_name: null  # 関連 Organization の URL 名
slide: false            # true: スライドモード ON
ignorePublish: false    # true: publish コマンドで無視される
```

記事テンプレート: `~/develop/claude-settings/docs/qiita-article-template.md`
