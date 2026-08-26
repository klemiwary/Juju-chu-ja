---
name: jujutsu
description: 複雑な revset、conflict、bookmark、履歴の書き換えや復旧、リモートとの同期、PR の作成など、単純ではない Jujutsu 操作に関する、このリポジトリ固有の手順。
---

# Jujutsu の操作

常時適用される方針は `AGENTS.md` に記載している。この Skill では、標準的な Jujutsu の挙動からは導けない、このリポジトリ固有の判断基準と手順のみを扱う。

## この Skill を使用する場面

- 単純ではない revset の記述や評価
- conflict の解消
- bookmark の作成、移動、追跡、追跡解除、削除
- rebase、squash、restore、abandon、undo、操作の復元
- diff エディタを使わない change の分割
- stale working copy や immutable revision のエラーへの対処
- fetch、push、PR の作成

通常の状態確認や一般的な作業の開始には、この Skill は必要ない。`AGENTS.md` に直接従うこと。

## リポジトリ固有の制約

- `git` コマンドを直接実行してはならない。ただし `jj git ...` と `gh` は使用してよい。
- `jj resolve`、`jj diffedit`、`jj arrange` を実行してはならない。これらは対話型インターフェースを必要とする。`jj split` は fileset と `-m` を渡す形式に限り使用してよい。引数なしでの実行、`-i` と `--tool` の指定は禁止（「ファイル単位で change を分割する」を参照）。非対話型コマンドで安全に完了できない作業については、その部分をユーザーに実行してもらうこと。
- `jj diff`、`jj show`、`jj log -p` には、必ず `--git` を付ける。
- `jj new`、`jj rebase`、`jj squash`、`jj restore`、`jj abandon` など、履歴を変更する操作を行った後は、作業を続ける前に `jj status` を実行する。
- PR の準備時に stacked change を squash してはならない。

# 自動フォーマット

`.codex/hooks.json` の Stop hook は、タスクの終了時に `jj fix` を実行する。これにより、複数の可変 change にまたがるファイルが遡及的にフォーマットされることがある。フォーマットによる差分だけであり、リポジトリのフォーマット規則に適合している場合は、その差分を受け入れること。

## スナップショットの規律

`--ignore-working-copy` を参照専用の確認、すなわち `jj log`、`jj diff`、`jj bookmark list`、`jj evolog`、`jj op log` に付けてよいのは、作業コピーがすでにスナップショット済みだと確認できている場合に限る。このオプションは新しいスナップショットの作成を抑止するため、古い状態を返す可能性がある。

作業の開始時、およびファイルを作成、編集、削除した直後には付けないこと。そこはまさに最新のファイルシステムの状態を取り込むべき場面である。

## リポジトリの push エイリアス

変更を公開するときは `jj git push` ではなく `jj push` を使う。リポジトリで設定されているエイリアスコマンドであり、`jj fix && mise run pre-push && jj git push` が実行される。なお実行には `mise` が必要となる。

## ファイル単位で change を分割する

```bash
jj split -m "<切り出す change の description>" <path>...
```

diff エディタを開かせないのは fileset、テキストエディタを開かせないのは `-m` である。hunk 単位の分割にはそのエディタが必要なので、ユーザーに委ねること。

## conflict を解消する

conflict マーカーを直接編集して意図した最終内容にし、`jj status` で conflict が残っていないことを確認する。ファイルを保存すること自体が解消である。`jj resolve` を実行してはならず、編集後に `git add` 相当の操作も要らない。

## 公開する revision に bookmark を配置する

通常の実装中に bookmark を移動してはならない。ユーザーから push または PR の作成を依頼された場合にのみ、bookmark を配置する。PR の作成依頼に push が暗黙的に含まれる場合も同様である。

`@` が公開対象の revision だと決めつけてはならない。`jj new` の実行後は空の change である可能性がある。`@` の祖先のうち内容と description の両方を持つ最新の revision を対象とし、bookmark を配置する前に出力結果を確認すること。

```bash
jj log --ignore-working-copy --no-graph \
  -r 'heads(::@ & ~empty() & ~description(exact:""))' \
  -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'

jj bookmark create <name> -r 'heads(::@ & ~empty() & ~description(exact:""))'
jj bookmark move <name> --to 'heads(::@ & ~empty() & ~description(exact:""))'
jj status
```

新しい bookmark には `create`、既存の bookmark には `move` を使用する。`jj bookmark advance` は使用せず、呼び出し箇所で対象 revision を明示する。

## リモートおよび PR のワークフロー

特に指示がない限り、PR のベースには `main` を使用し、stack 内の各 change を維持する。remote bookmark が未追跡なら `jj bookmark track <name>@origin` で追跡を開始し、local と remote が一致していれば push は不要。PR タイトルは Conventional Commits 形式で明示的に渡し、本文は日本語で書く。

```bash
jj git fetch
jj log --ignore-working-copy
jj bookmark list --all --ignore-working-copy   # 確認後、前述の手順で配置する
jj push -b <name>
gh pr create --base main --head <name> --title "<type>: <summary>" --body "<body>"
```

## 復旧とトラブルシューティング

`jj undo`、`jj op restore`、`jj restore`、`jj abandon` を実行する前に、影響を受ける操作または revision を確認し、それが意図した対象と正確に一致することを確かめる。対象を推測したまま、ユーザーの作業を破棄したり書き換えたりしてはならない。

**stale working copy** — `jj workspace update-stale` を実行し、続けて `jj status` で確認する。

**immutable revision** — immutability を迂回してはならない。対象 revision を確認し、意図した可変 change を選んで、その change 上で操作をやり直す。
