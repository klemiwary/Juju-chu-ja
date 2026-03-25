## Version Control — 必須手順

このプロジェクトは **Jujutsu (jj)** でバージョン管理している。詳細なルールは `.claude/rules/jujutsu-rules.md` を参照。

**コード編集を開始する前に、必ず以下の手順を実行すること:**

1. `jj log --ignore-working-copy -r @` で現在の change を確認する
2. description が空かつ diff が空（empty）→ `jj describe -m "<description>"` で description を設定して作業開始
3. それ以外（すでに作業中 or 完了済み）→ `jj new -m "<description>"` で新しい change を作成
4. description は Conventional Commits 形式、英語で記述

**禁止事項:** `git` コマンドの直接使用（`jj git` サブコマンドおよび `gh` CLI は許可）
