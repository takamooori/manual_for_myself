# Git 作業フロー チートシート

## 0. 基本サイクル

```
最新化 → ブランチ作成 → 作業・コミット → push → PR → マージ → 後片付け
```

---

## 1. 作業開始時（最新化してからブランチを切る）

```bash
git switch main          # main へ移動
git pull                 # リモートの最新を取り込む
git switch -c feat/xxx   # 新しいブランチを作って移動
```

| 状況 | コマンド |
|---|---|
| 既存ブランチへ移動 | `git switch <branch>` |
| ブランチ一覧 | `git branch -a` |
| 今どこにいるか確認 | `git status` |
| リモートの最新ブランチを取得 | `git fetch --prune` |

**ブランチ名の慣例**

| 種別 | 例 |
|---|---|
| 機能追加 | `feat/occlusion-csv-export` |
| バグ修正 | `fix/tf-timestamp` |
| リファクタ | `refactor/ekf-node` |
| ドキュメント | `docs/readme-setup` |

---

## 2. 作業中

```bash
git status               # 変更ファイル確認
git diff                 # 未ステージの差分
git diff --staged        # ステージ済みの差分

git add <file>           # 個別に追加（推奨）
git add -p               # 変更を部分的に選んで追加
git add .                # 全部追加（ノートブック出力に注意）

git commit -m "feat: 遮蔽率のCSV出力を追加"
```

**コミットメッセージ prefix**

`feat:` 機能 / `fix:` 修正 / `refactor:` 整理 / `docs:` 文書 / `chore:` 雑務 / `test:` テスト

---

## 3. push と PR

```bash
git push -u origin feat/xxx   # 初回（-u で追跡設定）
git push                      # 2回目以降
```

PR は GitHub 上、または CLI で:

```bash
gh pr create --base main --head feat/xxx --title "..." --body "..."
gh pr view --web             # ブラウザで開く
gh pr status                 # 自分のPR状況
```

---

## 4. マージ後の後片付け

```bash
git switch main
git pull                     # マージ結果を取り込む
git branch -d feat/xxx       # ローカルブランチ削除
git fetch --prune            # 消えたリモートブランチの参照を掃除
git branch -d feat/xxx       # ブランチの削除
           -D                # 強制
git push origin --delete feat/xxx  # リモートへの反映
```

> `-d` は未マージだと拒否される（安全）。強制削除は `-D`。

---

## 5. 作業が長引いたとき（main の変更を取り込む）

```bash
git switch feat/xxx
git fetch origin
git merge origin/main        # 履歴を残す（安全・一般的）
# または
git rebase origin/main       # 履歴を直線化（push 済みなら要注意）
```

競合したら:

```bash
# ファイルを編集して解決
git add <解決したファイル>
git commit                   # merge の場合
git rebase --continue        # rebase の場合
git merge --abort / git rebase --abort   # やめる
```

---

## 6. 一時退避・やり直し

| やりたいこと | コマンド |
|---|---|
| 作業を一時退避 | `git stash push -m "wip"` |
| 退避一覧 | `git stash list` |
| 戻す | `git stash pop` |
| 直前コミットのメッセージ修正 | `git commit --amend` |
| 直前コミットを取り消し（変更は残す） | `git reset --soft HEAD^` |
| ステージを解除 | `git restore --staged <file>` |
| ファイルの変更を破棄 | `git restore <file>` |
| 公開済みコミットを打ち消す | `git revert <commit>` |

> `reset --hard` は変更が消えるので、push 済みブランチでは使わない。

---

## 7. 履歴確認

```bash
git log --oneline --graph --decorate --all   # 履歴を俯瞰
git log -p <file>                            # ファイルの変更履歴
git show <commit>                            # 特定コミットの内容
git blame <file>                             # 行ごとの最終変更者
```

---

## 8. 初期設定（1回だけ）

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false        # pull はマージ方式
```

エイリアス例:

```bash
git config --global alias.st status
git config --global alias.sw switch
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## 9. ノートブック運用メモ

- コミット前に出力をクリアする
- 自動化する場合は `nbstripout` を導入

```bash
pip install nbstripout
nbstripout --install     # リポジトリに filter を設定
```

---

## 10. PR テンプレート

```markdown
## PR Type
- [ ] Feature
- [ ] Bug fix
- [ ] Refactor
- [ ] Documentation
- [ ] Other

## Overview

## Detail

## Test

## Attention
```

`.github/pull_request_template.md` に置くと自動で反映される。
