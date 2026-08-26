## バージョン管理 — 必須手順

このプロジェクトはバージョン管理に Jujutsu（`jj`）を使用する。用語とコマンドは `jj` に統一し、`git` コマンドを直接実行してはならない。ただし `jj git ...` と `gh` は使用してよい。

編集を始める前に `jj status` を実行し、working copy をスナップショットに保存したうえで、working-copy commit（`@`）の状態を確認する。`@` に description と diff のどちらもない場合に限り、そのまま再利用する。それ以外の場合は、新しい change を作成する。change description は Conventional Commits に従い、英語で記述する。

`jj diff`、`jj show`、`jj log -p` には、必ず `--git` を付ける。`--ignore-working-copy` は、working copy がスナップショット済みだと確認できている場合に限り、変更を伴わない参照目的の操作にのみ使用する。

`jj split`、`jj resolve`、`jj diffedit`、`jj arrange` を対話モードで実行してはならない。履歴を変更する操作を行った後は `jj status` を実行し、conflict があれば解消してから作業を続ける。

変更を公開するとき以外は、bookmark を移動しない。特に指示がない限り、PR のターゲットは `main` とする。また、stacked change を squash してはならない。

複雑な revset、conflict、bookmark の操作、履歴の書き換えや復旧、fetch、push、PR の作成については、`.agents/skills/jujutsu/SKILL.md` にある `jujutsu` Skill を参照すること。
