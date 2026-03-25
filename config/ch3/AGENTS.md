## Jujutsu 運用ルール

このリポジトリではバージョン管理に Jujutsu (`jj`) を使う。  
Git ではなく、常に Jujutsu の概念とコマンドで作業すること。
詳細な手順については `.codex/skills/jujutsu/SKILL.md` を参照。

### 最重要ルール

- raw `git` コマンドは**原則使用禁止**
  - 例外: `jj git ...` と `gh` CLI
- `jj split` / `jj resolve` / `jj diffedit` は実行禁止（対話的コマンドのため）
- 読み取り専用であっても `git log` などは使わず、`jj log` などで代替する
- `jj diff` / `jj show` / `jj log -p` では常に `--git` を付ける
- `jj rebase` / `jj new` / `jj squash` などの変更操作の直後は、必ず `jj status` を実行して conflict の有無を確認する
- 特に指示がない限り、PR のベースは `main`
- stacked changes を squash してはならない

### 用語の扱い

Git の用語ではなく、Jujutsu の用語で考えること。

- commit（作業単位） → change
- branch → bookmark
- HEAD → `@`（working copy）
- staging / unstaged / uncommitted という概念はない
- `git add` は不要
- `git commit --amend` は不要
- stash の代わりに必要なら `jj new` を使う

### 作業開始の原則

**コード編集を開始する前に、必ず以下の手順を実行すること:**

1. `jj log --ignore-working-copy -r @` で現在の change を確認する
2. description が空かつ diff が空（empty）→  `jj describe -m "<description>"` で description を設定して作業開始
3. それ以外（すでに作業中 or 完了済み）→  `jj new -m "<description>"` で新しい change を作成
4. description は Conventional Commits 形式、英語で記述

### 基本確認コマンド

必要に応じて以下を優先して使うこと。

```bash
jj status
jj log --ignore-working-copy
jj diff --git --ignore-working-copy
jj evolog --ignore-working-copy
jj op log --ignore-working-copy
````

**補足説明**: AI エージェント自身がファイルを変更した後にそれを確認する場合を除き、純粋な読み取り操作には `--ignore-working-copy` を付与することを推奨。

## conflict の原則

Jujutsu では conflict が起きても操作は中断されない。
そのため、変更操作後に `jj status` を確認せず先へ進んではならない。

conflict があれば、ファイルを直接編集して解消し、再度 `jj status` で確認すること。
`git add` に相当する操作は不要。

## リモート・PR の原則

- リモート同期には `jj git fetch` / `jj git push -b <bookmark-name>` を使う
- PR 作成には `gh pr create --base main --head <bookmark-name>` を使う
- bookmark は branch と違い、自動では移動しない。必要に応じて明示的に操作すること

## 補足

このプロジェクトで Jujutsu に関する詳細な操作を行う場合、必ず Skill `jujutsu` を参照すること。  
Jujutsu 固有の判断が必要な場面では、Skill `jujutsu` を確認する前に作業を進めてはならない。

対象例:
- revset
- conflict 解決
- bookmark 操作
- rebase / squash / split / restore / abandon / undo / op restore
- stale / immutable などのエラー対応
- fetch / push / PR 作成
