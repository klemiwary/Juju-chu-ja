---
name: jujutsu
description: この Skill は、このプロジェクトで AI エージェントがバージョン管理システムとして Jujutsu (`jj`) を安全かつ一貫して使うための詳細手順をまとめたもの。 常設ルールは `AGENTS.md` にあり、この Skill には具体的な操作手順と判断基準を記す。
---

# Jujutsu 運用スキル

## この Skill を使う場面

- 現在の作業状態や履歴を確認したい
- 新しい change を始めたい
- rebase / squash / split / restore を行いたい
- conflict を解決したい
- bookmark を作成・移動・削除したい
- revset で対象を指定したい
- PR を作成したい
- stale / immutable などのエラーに対処したい

## 前提

このプロジェクトでは VCS として Jujutsu を使う。

- raw `git` コマンドは原則使用禁止
- 例外は `jj git ...` と `gh` CLI
- 読み取り専用の `git log` なども使わず、`jj` コマンドで代替する

## 用語のマッピング

Git の用語をそのまま持ち込まないこと。

| Git 用語 | Jujutsu での扱い |
| --- | --- |
| commit（作業単位） | change |
| branch | bookmark |
| HEAD | `@`（working copy） |
| staging | その概念はない |
| unstaged / uncommitted | その概念はない |
| stash | 基本的に不要。必要なら `jj new` |
| `git add` | 不要 |
| `git commit --amend` | 不要 |

### 重要な考え方

1. ファイルを保存した時点で変更は現在の change に含まれる
2. change は作業単位であり、revision はそのスナップショット
3. 親を変更すると子孫 change は自動 rebase される
4. conflict は first-class な状態として記録される
5. すべての操作は operation log に残るので、必要なら `jj undo` や `jj op restore` で戻せる

## diff と log の基本方針

### diff

`jj diff` / `jj show` / `jj log -p` では、常に `--git` を付けること。

```bash
jj diff --git
jj diff --git -r @-
jj show --git
jj log -p --git
```

禁止例:

```bash
jj diff
jj diff -r @-
jj show
jj log -p
```

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

# ℹ️ スナップショットだけを取りたい場合
jj util snapshot
```

**判断基準**: 直前にファイルの作成・編集・削除を行った場合は `--ignore-working-copy` を付けない。それ以外の「既知の状態を確認するだけ」の場面では付与する。

### log

通常の確認ではグラフ付きの `jj log` を使う。
機械的に抽出したいときだけ `--no-graph` と `-T` を使う。

```bash
jj log --ignore-working-copy
jj log --ignore-working-copy --no-graph -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'
jj log --ignore-working-copy --no-graph -T 'commit_id.short() ++ " " ++ bookmarks ++ "\n"' -r 'bookmarks()'
```

## 基本ワークフロー

### 1. 状態確認

```bash
jj status
jj log --ignore-working-copy
jj diff --git
jj diff --git --ignore-working-copy -r @-
jj evolog --ignore-working-copy
jj op log --ignore-working-copy
```

用途:

- `jj status`: working copy の状態確認
- `jj log --ignore-working-copy`: 履歴とスタック構造の把握
- `jj diff --git`: 現在の差分確認（変更直後）
- `jj diff --git --ignore-working-copy -r @-`: 一つ前の change の差分確認（読み取り専用）
- `jj evolog --ignore-working-copy`: 現在の change の変遷確認
- `jj op log --ignore-working-copy`: 操作履歴の確認

### 2. 作業の開始

現在の `@` の状態を見て、`describe` にするか `new` にするかを決める。

手順:

1. `jj log --ignore-working-copy -r @` で現在の change を確認する
2. description が空で、かつ diff も空なら、その `@` を使って作業開始する
3. その場合は `jj describe -m "<description>"` を使う
4. すでに作業中、または description / diff があるなら `jj new -m "<description>"` を使う
5. description は Conventional Commits 形式に従い、英語で記述する

例:

```bash
jj log --ignore-working-copy -r @
jj describe -m "feat: add search form"
```

または:

```bash
jj new -m "fix: handle empty input"
```

### 3. 変更操作後の conflict 確認

`jj rebase`、`jj new`、`jj squash` などの変更操作の直後は、必ず `jj status` を実行すること。

```bash
jj rebase -s @ -d main
jj status
```

`jj` は conflict が起きても操作を止めない。
そのため、`jj status` を見ずに先へ進むと conflict を抱えたまま作業を続けてしまう危険がある。

conflict の兆候の例:

```text
The change has 2 conflicts:
  src/main.rs    2-sided conflict
```

### 4. bookmark 操作

bookmark は Git の branch と違い、自動では動かない。必要なら明示的に操作する。

```bash
jj bookmark create <name> -r @
jj bookmark move <name> -t @
jj bookmark list --ignore-working-copy
jj bookmark delete <name>
```

用途:

- 新しい bookmark を作る
- 既存 bookmark を現在の change に移す
- 一覧を確認する
- 不要な bookmark を削除する

### 5. change の分割・復元

#### 分割

change が大きくなりすぎたときは分割する。

```bash
jj split -r <revision>
```

#### 復元

別の revision の状態を一部取り戻したいときは `restore` を使う。

```bash
jj restore --from <revision> <path>
```

#### 過去バージョンへの復元

`evolog` を見て、同じ change の過去状態へ戻すこともできる。

```bash
jj evolog --ignore-working-copy -r <change-id>
jj restore --from <change-id>/1 --to <change-id>
```

補足:

- `<change-id>/0` は最新版
- `<change-id>/1` は一つ前の版
- 実行前に `jj evolog` で確認すること

### 6. 履歴の修正・取り消し

```bash
jj undo
jj op restore <operation-id>
jj abandon @
```

用途:

- `jj undo`: 直前の操作を取り消す
- `jj op restore <operation-id>`: 任意の操作時点に戻す
- `jj abandon @`: 現在の change を破棄する

失敗を恐れず操作してよいが、意図が曖昧なときは `jj op log` で確認してから戻す。

## conflict 解決

Jujutsu では conflict が発生しても、その状態のまま change が記録される。
解決は次の手順で行う。

1. `jj status` で conflict 対象ファイルを確認する
2. ファイルを開き、conflict マーカーを含む箇所を直接編集する
3. 正しい内容に整えて保存する
4. `jj status` で conflict が消えたことを確認する

conflict マーカーの形式:

```text
<<<<<<<
%%%%%%%
-removed line
+added line
+++++++
content from the other side
>>>>>>>
```

意味:

- `%%%%%%%` ブロック: ベースからの差分
- `+++++++` ブロック: もう一方の内容そのもの

注意:

- Git のような `git add` は不要
- 保存すれば `jj` が自動的に解消を検知する

## リモートとの同期

```bash
jj git fetch
jj git push -b <bookmark-name>
```

- `jj git fetch`: リモートの最新状態を取得
- `jj git push -b <bookmark-name>`: bookmark を push

## revset チートシート

### 基本記法

| 記法                  | 意味                       |
| ------------------- | ------------------------ |
| `@`                 | 現在の change               |
| `@-`                | 1つ前の change              |
| `@--`               | 2つ前の change              |
| `<bookmark>`        | bookmark 名               |
| `<bookmark>@origin` | リモートの bookmark           |
| `main..@`           | `main` から現在までの change 集合 |
| `empty()`           | 空の change                |
| `<change-id>/n`     | 同じ change の n 世代前の版      |

### 便利なパターン

| パターン                                | 用途                  |
| ----------------------------------- | ------------------- |
| `trunk()..@`                        | main から現在までのスタック全体  |
| `mine() & mutable() & ~empty()`     | 自分の作業中の change 一覧   |
| `conflict()`                        | conflict を含む change |
| `bookmarks() & ~remote_bookmarks()` | 未 push の bookmark   |

例:

```bash
jj log -r 'trunk()..@'
jj log -r 'conflict()'
jj log -r 'mine() & mutable() & ~empty()'
```

## PR 作成ワークフロー

### 基本ルール

- 特に指示がない限り、PR のベースは `main`
- 複数 change を squash してはならない
- 次のことは毎回確認しなくてよい

  * 変更を含めるか → 常に現在の change に含まれている
  * `main` をベースにするか → 明示がなければ `main`
  * squash するか → しない

### 手順

```bash
jj git fetch
jj log --ignore-working-copy
jj bookmark list --ignore-working-copy
jj bookmark track <bookmark-name>@origin
jj git push -b <bookmark-name>
gh pr create --base main --head <bookmark-name>
```

補足:

- 先に bookmark の有無を確認する
- 未追跡なら `track` する
- push 後に `gh` で PR を作成する

## トラブルシューティング

### 「The working copy is stale」

人間と AI が並行して同じリポジトリを触った場合や、外部ツールがファイルを書き換えた場合に起こる。

対処:

```bash
jj workspace update-stale
jj status
```

### 「Commit XXXX is immutable」

`describe` や `squash` の対象が `main@origin` やその祖先など、immutable な revision に含まれている。

対処:

1. `-r` で指定した対象が正しいか確認する
2. mutable な change を対象にやり直す
3. 必要なら `jj log` で現在位置を確認する

## 最後の注意事項

1. `jj` には staging の概念がない
2. 作業ディレクトリの変更は自動追跡される
3. 変更操作後の `jj status` は必須
4. `.git` と共存できるが、操作は `jj` 経由で行う
5. `git` コマンドは状態を壊すリスクがあるため原則使わない
6. 迷ったら `jj log` / `jj status` / `jj op log` を見てから動く
