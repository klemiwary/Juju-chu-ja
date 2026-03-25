# Jujutsu (jj) Rules for AI Agents

このプロジェクトではバージョン管理に **Jujutsu (jj)** を使用する。AI エージェントは以下のルールを厳守し、**`git` コマンドは原則使用禁止**とする（`jj git` サブコマンドおよび `gh` CLI を除く）。

---

## AI エージェント向け重要注意

### 用語のマッピング

Jujutsu と Git では用語の意味が異なる。AI は以下の用語の使い分けを徹底すること。

| Git 用語                     | Jujutsu 用語                                 |
| ---------------------------- | -------------------------------------------- |
| commit（名詞として）         | change                                       |
| branch                       | bookmark                                     |
| staging                      | （この概念は存在しない）                     |
| unstaged / uncommitted       | （この概念は存在しない）                     |
| HEAD                         | `@`（working copy）                          |
| stash                        | （この概念は存在しない。`jj new` で代替）    |
| `git add`                    | （不要。自動スナップショット）               |
| `git commit --amend`         | （不要。`@` への変更は自動的に反映される）   |

### Jujutsu の Git との根本的な違い

1. **「未保存の変更」は存在しない**: ファイルを保存した瞬間に現在の change に自動的に含まれる。「この変更を含めますか？」という確認は不要である。
2. **change と revision**:
   - **change**: 作業単位。一意で不変の **change ID** を持ち、内容は可変（mutable）。
   - **revision**: change のスナップショット。編集のたびに新しい revision が作られるが、change ID は不変。
3. **自動 rebase**: change の親を変更すると、子孫の change が自動的に rebase される。手動で rebase チェーンを管理する必要はない。
4. **first-class conflict**: conflict が発生しても操作は中断されず、conflict を含む change として記録される。後から解決可能。
5. **operation log**: すべての操作が記録されており、`jj undo` / `jj op restore` で任意の時点に戻れる。失敗を恐れず操作してよい。

### diff 出力のルール

**`jj diff`、`jj show`、`jj log -p` を実行する際は、常に `--git` フラグを付けること。**

このオプション指定により差分出力が自動的に Git 形式になる。

```bash
# 正しい
jj diff --git
jj diff --git -r @-
jj diff --git --from main --to @

# 禁止
jj diff           # --git なしは禁止
jj diff -r @-     # --git なしは禁止
```

Git 形式の diff では、ファイルの追加・削除・リネーム・パーミッション変更も正確に表現され、AI エージェントによる差分の解析精度が大幅に向上する。

### 読み取り専用操作でのスナップショット抑止

jj はコマンド実行のたびに作業コピーのスナップショットを自動作成する。AI が不用意に状態確認を行うと、他のプロセスでの操作によって operation log の競合が起きるリスクがあるため、**ファイルを変更していない状態**での純粋な読み取り操作には `--ignore-working-copy` を付与すること。

```bash
# ✅ 読み取り専用：ファイル変更を伴わない調査時
jj log --ignore-working-copy
jj log --ignore-working-copy -r 'main..@'
jj diff --git --ignore-working-copy -r @-
jj bookmark list --ignore-working-copy

# ❌ 付けてはいけない場面：ファイル変更直後の状態確認
#    （最新のスナップショットを反映させる必要があるため）
jj status          # 変更直後は --ignore-working-copy を付けない
jj diff --git      # 変更直後は --ignore-working-copy を付けない

# ℹ️  スナップショットだけを取りたい場合
jj util snapshot
```

**判断基準**: 直前にファイルの作成・編集・削除を行った場合は `--ignore-working-copy` を付けない。それ以外の「既知の状態を確認するだけ」の場面では付与する。

### ログ出力の使い分け

通常の状態確認では `jj log` のグラフ表示をそのまま使用する。特定の情報を機械的に抽出する必要がある場合は `--no-graph` と `--template`（`-T`）を活用する。

```bash
# 通常の確認（グラフ付き）
jj log --ignore-working-copy

# 特定情報の抽出（機械可読形式）
jj log --ignore-working-copy --no-graph -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'
jj log --ignore-working-copy --no-graph -T 'commit_id.short() ++ " " ++ bookmarks ++ "\n"' -r 'bookmarks()'
```

---

## 基本ワークフロー

### 1. 状態・差分確認

```bash
jj status                                  # 作業コピーの状態確認（変更直後はそのまま実行）
jj log --ignore-working-copy               # 履歴をグラフ表示
jj diff --git                              # 現在の作業コピーの差分（変更直後）
jj diff --git --ignore-working-copy -r @-  # 一つ前の change の差分（読み取り専用）
jj evolog --ignore-working-copy            # 現在の change の変遷履歴
jj op log --ignore-working-copy            # 操作履歴の表示
```

### 2. 作業の開始

現在の change (`@`) の状態に応じて操作を変えること。判断手順は以下の通り。

1. `jj log --ignore-working-copy -r @` で現在の change を確認する。
2. description が空かつ diff が空（empty）なら → `jj describe -m "<description>"` のように description を設定して作業開始。
3. それ以外（すでに作業中 or 完了済み）→ `jj new -m "<description>"` で新しい change を作成。
4. description は Conventional Commits 形式に従い、英語で記述すること。

### 3. 変更操作後の conflict 確認

**`jj rebase`、`jj new`、`jj squash` などの変更操作を行った後は、必ず `jj status` で conflict の有無を確認すること。** Jujutsu は conflict が発生しても操作を中断しないため、気づかずに作業を進めてしまうリスクがある。

```bash
# 変更操作の後は必ず実行
jj status

# 出力に以下のような行が含まれていたら conflict が存在する：
#   The change has 2 conflicts:
#     src/main.rs    2-sided conflict
```

conflict を検知した場合は、作業を進める前に解決すること（手順は「conflict 解決」セクションを参照）。

### 4. ブックマーク操作

bookmark は Git の branch と違って、手動で動かす必要がある。

```bash
jj bookmark create <name> -r @          # 新しいブックマークを作成（-r で対象の revision を指定）
jj bookmark move <name> -t @            # 既存のブックマークを現在の change に移動
jj bookmark list --ignore-working-copy  # ブックマーク一覧
jj bookmark delete <name>               # ブックマークを削除
```

### 5. 変更の分割・復元

```bash
# change を複数に分割（スコープが大きすぎる場合の修正に使用）
jj split -r <revision>

# 特定ファイルを別の revision から復元
jj restore --from <revision> <path>

# change を以前の状態に戻す（evolog で確認した過去バージョンを使用）
jj evolog --ignore-working-copy -r <change-id>    # 過去バージョンを確認
jj restore --from <change-id>/1 --to <change-id>  # 1つ前の状態に復元
```

> **補足**: `<change-id>/n` 記法は `xyz/0` が最新版、`xyz/1` が一つ前のバージョンを指す。`jj evolog` で変遷履歴を確認してから使用すること。

### 6. 履歴の修正・取り消し

- **`jj undo`**: 操作を誤った場合は躊躇なく使用して直前の状態に戻ること。
- **`jj op restore <operation-id>`**: 特定の操作時点まで戻す。`jj op log` で操作 ID を確認できる。
- **`jj abandon @`**: 現在の change 自体を破棄する。

### 7. conflict 解決

Jujutsu では conflict が発生しても操作は中断されず、conflict マーカーが挿入された状態で change が記録される。AI は以下の手順で解決すること。

1. `jj status` で conflict しているファイルを確認する。
2. 対象ファイルを開き、conflict マーカーを含む箇所を**直接編集して正しい状態に書き換える**。

Jujutsu の conflict マーカーは Git とは異なる形式を使用する：

```
<<<<<<<
%%%%%%%
-removed line
+added line
+++++++
content from the other side
>>>>>>>
```

- `%%%%%%%` ブロック: diff 形式。ベースからの変更を `-`/`+` で表現する。
- `+++++++` ブロック: スナップショット形式。もう一方の内容をそのまま表示する。

3. ファイルを保存する。`jj` が自動的に解消を検知するため、`git add` に相当する操作は不要。
4. `jj status` で conflict が消えたことを確認する。

### 8. リモートとの同期

```bash
jj git fetch                       # リモートの最新状態を取得
jj git push -b <bookmark-name>    # bookmark をリモートに push
```

---

## revision 指定の記法（revset）

| 記法                  | 意味                                         |
| --------------------- | -------------------------------------------- |
| `@`                   | 現在の change                                |
| `@-`                  | 1つ前の change                               |
| `@--`                 | 2つ前の change                               |
| `<bookmark>`          | bookmark 名で指定                            |
| `<bookmark>@origin`   | リモートの bookmark                          |
| `main..@`             | `main` から現在の変更までの全 change セット  |
| `empty()`             | 内容が空の change                            |
| `<change-id>/n`       | change の n 世代前のバージョン（0 が最新）   |

---

## 便利に使える revset パターン

| パターン                                    | 用途                             |
| ------------------------------------------- | -------------------------------- |
| `trunk()..@`                                | main から現在までのスタック全体  |
| `mine() & mutable() & ~empty()`             | 自分の作業中の change 一覧      |
| `conflict()`                                | コンフリクトを持つ change        |
| `bookmarks() & ~remote_bookmarks()`         | 未 push のブックマーク           |

```bash
# 使用例
jj log --ignore-working-copy -r 'trunk()..@'
jj log --ignore-working-copy -r 'conflict()'
```

---

## PR 作成ワークフロー

### 基本ルール

1. **ターゲットは常に `main`** — 特に指示がない限り、PR のターゲットブランチは `main` とする。確認は不要。
2. **squash は禁止** — 複数の change を 1 つにまとめてはならない。変更履歴の追跡可能性を維持するため、各 change はそのまま保持する。
3. **確認不要な事項** — 以下について毎回ユーザーに確認する必要はない：
   - 「変更を含めますか？」→ 常に含まれている
   - 「main にマージしますか？」→ 特に指示がなければ main
   - 「squash しますか？」→ しない

### PR 作成手順

```bash
# 1. リモートの最新状態を取得
jj git fetch

# 2. 現在の状態を確認
jj log --ignore-working-copy

# 3. bookmark が設定されていることを確認
jj bookmark list --ignore-working-copy

# 4. bookmark が未追跡の場合は追跡を開始する
jj bookmark track <bookmark-name>@origin

# 5. リモートに push
jj git push -b <bookmark-name>

# 6. GitHub CLI で PR 作成
gh pr create --base main --head <bookmark-name>
```

---

## トラブルシューティング

### 「The working copy is stale」エラー

人間と AI が同じリポジトリで並行作業している場合や、外部ツールがファイルを変更した場合に発生する。このエラーが出た場合は以下を実行して同期する。

```bash
jj workspace update-stale
```

その後、`jj status` で作業コピーの状態が正常であることを確認すること。

### 「Commit XXXX is immutable」エラー

`describe` や `squash` を実行した際にこのエラーが出た場合、操作対象が保護されたリビジョン（`main@origin` とその祖先）に含まれている。`-r` オプションで指定しているリビジョンが正しいか確認し、mutable な change に対して操作をやり直すこと。

---

## 注意事項

1. **自動保存**: jj は作業ディレクトリの変更を自動的に追跡する。明示的な `add` は不要。
2. **イミュータブル履歴**: デフォルトでは `trunk()` とその祖先が不変。ローカルの mutable な change は自由に編集可能。
3. **conflict の扱い**: jj は conflict を含む change も記録可能。後から解決できるが、**変更操作後は必ず `jj status` で確認すること**。
4. **Git 互換**: `.git` ディレクトリと共存可能。`jj git push/fetch` で Git リモートと連携。
5. **change ID の転送**: change ID は Git commit ヘッダー（`change-id`）としてリモートにも転送される。
6. **glob パターン**: revset 等の文字列パターンはデフォルトで glob として解釈される。部分一致には `substring:` プレフィックスを使用する。
7. **git コマンドの使用制限**: `git` コマンドは状態を壊すリスクがあるため原則禁止。`gh` CLI は内部的に `git` を使用するが、これは許可する。読み取り専用の `git` 操作（`git log` 等）も `jj log` で代替すること。
