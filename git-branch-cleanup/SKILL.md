---
name: git-branch-cleanup
description: |
  リモートリポジトリの不要なブランチをリストアップするスキル。
  デフォルトブランチにmerge済みで、merge後に新たなコミットがないブランチ（削除してよいブランチ）と、
  まだ作業中のブランチ（未mergeのもの）を分類してリストアップする。

  以下のような場面で必ず使用する：
  - 「不要なブランチをリストアップして」「ブランチを整理したい」「古いブランチを消したい」
  - 「merge済みのブランチを調べて」「ブランチの棚卸しをしたい」
  - 「リモートブランチをクリーンアップしたい」「どのブランチが消せるか調べて」
  ユーザーがブランチの整理・削除・棚卸しに言及した場合は、明示的に依頼されていなくてもこのスキルを使うこと。
---

# git-branch-cleanup

リモートリポジトリの不要なブランチをリストアップし、削除候補と作業中ブランチを分類して報告するスキル。

## 実行手順

### 1. デフォルトブランチを確認

まず作業前にリポジトリのデフォルトブランチを確認する。`origin/HEAD` がローカルにキャッシュされていれば高速に取得できる。

```bash
# 推奨：ローカルキャッシュから取得（ネットワーク不要）
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's!.*/!!')

# origin/HEAD が未設定の場合はリモートに問い合わせてフォールバック
if [ -z "$DEFAULT_BRANCH" ]; then
  DEFAULT_BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
fi

echo "デフォルトブランチ: $DEFAULT_BRANCH"
```

以降の手順では `origin/main` の代わりに `origin/$DEFAULT_BRANCH` を使う。

### 2. リモートブランチ情報を最新化

```bash
git fetch --prune
```

### 3. ブランチを分類

```bash
# デフォルトブランチにmerge済みのブランチ（origin/HEAD・デフォルトブランチ自身を除く）
git branch -r --merged origin/$DEFAULT_BRANCH | grep -v "origin/HEAD\|origin/$DEFAULT_BRANCH"

# デフォルトブランチに未mergeのブランチ
git branch -r --no-merged origin/$DEFAULT_BRANCH | grep -v "origin/HEAD\|origin/$DEFAULT_BRANCH"
```

### 4. merge済みブランチのデフォルトブランチ以降の差分コミット数を確認

merge済みブランチでも、merge後に新たなコミットが追加されている場合がある。
そのため、差分コミット数を確認する。

```bash
git rev-list origin/$DEFAULT_BRANCH..origin/<branch> --count
```

- count = 0 → **削除してよいブランチ**（デフォルトブランチにmerge済みで後続コミットなし）
- count > 0 → 要確認（merge済みだが追加コミットあり）

### 5. 各ブランチの最新コミット情報を取得

`%ai` が日時、`%an` が author name、`%s` がコミットメッセージ（subject）に対応する。

```bash
git log origin/<branch> -1 --format="%ai %an %s"
```

### 6. 未mergeブランチの差分コミット数も確認

```bash
git rev-list origin/$DEFAULT_BRANCH..origin/<branch> --count
```

コミット数が多い・最終更新が古いかどうかで「要確認」「放棄の可能性あり」を判断する。

### 7. 未mergeブランチのPR情報を取得

`gh` コマンドが利用できる場合は、未mergeブランチに対応するopen PR（draft含む）を取得する。

```bash
# 全てのopen PRを一度に取得（draft含む）
gh pr list --state open --json number,headRefName,isDraft
```

取得したJSONの `headRefName` と未mergeブランチ名（`origin/` プレフィックスを除いた部分）を照合し、該当するPR番号を各ブランチに紐付ける。
`gh` コマンドが利用できない場合はこの手順をスキップし、PR情報の列は `-` で統一する。

---

## 出力フォーマット

以下の形式で結果をまとめる。デフォルトブランチ名を見出しに明記する。

### 削除してよいブランチ N本

| ブランチ名 | 最新コミット日時 | 最新コミットAuthor | 最新コミットメッセージ |
|---|---|---|---|
| feature/xxx | YYYY-MM-DD | author name | コミットメッセージ |

一括削除コマンド：

```bash
git push origin --delete \
  feature/xxx \
  fix/yyy
```

### 残すべきブランチ N本（`<デフォルトブランチ>` に未merge）

#### 要確認（最近のコミット・コミット数多い）

| ブランチ名 | 未mergeコミット数 | 最新コミット日時 | 最新コミットAuthor | 最新コミットメッセージ | PR番号 |
|---|---|---|---|---|---|
| feature/yyy | 26 | YYYY-MM-DD | author name | コミットメッセージ | #123 |

#### 古くて放棄の可能性あり（目安：最終更新から1年以上）

| ブランチ名 | 未mergeコミット数 | 最新コミット日時 | 最新コミットAuthor | 最新コミットメッセージ | PR番号 |
|---|---|---|---|---|---|
| feature/zzz | 3 | YYYY-MM-DD | author name | コミットメッセージ | - |

---

## 注意事項

- 以下のブランチはリストに含めない
  - デフォルトブランチ自身
  - `origin/HEAD`
  - `master` ブランチ
  - `main` ブランチ
  - `develop` ブランチ
- merge済みでも差分コミット数 > 0 のブランチは削除候補に含めない
- ユーザーから保存先を指示された場合は、結果を指定のmarkdownファイルにも書き出す
- 削除実行はユーザーの確認を得てから行う
