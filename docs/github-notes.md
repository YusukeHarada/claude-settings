# GitHub 運用ノート

GitHub CLI・Git 運用で詰まったこと・決めごとを記録する。

---

## GitHub CLI スコープ

### Private リポジトリで `gh` コマンドが失敗する

**症状**
`gh pr create` や `gh pr list` が "Could not resolve to a Repository" 等のエラーで失敗する。

**原因**
`public_repo` スコープでは Private リポジトリを操作できない。

**解決策**
`repo` スコープで再認証する。

```bash
gh auth status              # 現在のスコープを確認（'repo' が含まれていれば OK）
gh auth refresh --scopes repo  # スコープを追加して再認証
```

スコープと権限の対応:

| スコープ | Public 読み | Public 書き | Private 読み | Private 書き |
|---|---|---|---|---|
| なし | 可 | 不可 | 不可 | 不可 |
| `public_repo` | 可 | 可 | 不可 | 不可 |
| `repo` | 可 | 可 | 可 | 可 ✓ |

- 初回セットアップ時は必ず `repo` スコープで認証する
- すべての操作は YusukeHarada 名義として GitHub に記録される

---

## PR の作成

### PR body テンプレート

`gh pr create` 時に使う標準フォーマット。Claude Code が毎回一貫したフォーマットで作成できるよう定義しておく。

```bash
gh pr create --title "タイトル（70文字以内）" --body "$(cat <<'EOF'
## 変更内容
<!-- なぜこの変更が必要か -->

## 確認済み
- [ ] `npx tsc --noEmit` 通過
- [ ] `npm run test:run` 通過
- [ ] `npm run build` 通過
EOF
)"
```

### ドラフト PR で誤レビュー依頼を防ぐ

CI 確認前に誰かがレビューしてしまうのを防ぐため、ドラフトで作成してから ready に変更する運用も有効。

```bash
gh pr create --draft --title "タイトル" --body "..."
gh pr ready <番号>  # CI 確認後にドラフト解除
```

---

## PR のマージ

### squash merge を使う

`main` の履歴を「1PR = 1コミット」に保つため、squash merge を標準とする。

```bash
gh pr merge <番号> --squash --delete-branch
```

### ブランチの自動削除

GitHub の Settings → General → "Automatically delete head branches" を ON にしておくと、マージ後のブランチ掃除が不要になる。

---

## PR の確認・CI

```bash
# PR 詳細とレビューコメントを確認
gh pr view <番号> --comments

# CI の状態を確認
gh pr checks <番号>

# ブランチ間の差分を確認
gh pr diff <番号>

# Issue をブランチと紐付けて作業開始
gh issue develop <番号> --checkout
```

---

## ブランチ管理

### リモート追跡参照の掃除

マージ済みリモートブランチのローカル追跡参照を定期的に削除する。

```bash
git fetch --prune
```

---

## GitHub Actions

### Actions 無料枠

GitHub Free プランの無料枠: 月 2,000 分。
使用量は GitHub → Settings → Billing & plans → Actions で確認できる。

### 権限の最小化

`GITHUB_TOKEN` のデフォルト権限は Organization 設定に依存するため、ワークフローで明示する。
`contents: write` は強力な権限なので、本当に必要な場合のみ付与する。

```yaml
# PR 作成・CI 確認のみ（リポジトリへの書き込み不要）
permissions:
  contents: read       # チェックアウトのみ
  pull-requests: write # PR 作成・コメント

# リリース成果物のアップロード等、リポジトリへの書き込みが必要な場合のみ
permissions:
  contents: write
  pull-requests: write
```
