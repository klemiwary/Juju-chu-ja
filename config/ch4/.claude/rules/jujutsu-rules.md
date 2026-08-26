# Jujutsu (jj) Rules for AI Agents

jj の標準的なセマンティクスは AI エージェントにとって既知のものと想定する。ここに記述するのはそこから推察しえない、Jujutsu ネイティブのワークフロー、およびリポジトリ固有の設定やエイリアスを前提とした手順である。

---

## AI エージェントへの重要な注意

### コマンドの制約

以下は `settings.json` で明示的に禁止されている。これらを前提にしたプランを立ててはならない。どうしても必要な場合は、コマンドを提示してユーザー自身に実行してもらうこと。

| 禁止コマンド  | 理由                                                                  |
| ------------- | --------------------------------------------------------------------- |
| `git`（全般） | jj の状態を壊すおそれがある。`jj git ...` と `gh` は引き続き使用可。  |
| `jj resolve`  | マージエディタが開く。conflict はファイルを直接編集して解消する。     |
| `jj diffedit` | diff エディタが開く。                                                 |
| `jj arrange`  | 対話的な TUI。                                                        |

`jj split` は使用できるが、**必ず fileset と `-m` を渡すこと。引数なしでの実行、および `-i` と `--tool` の指定は、diff エディタが起動してしまうためいずれも禁止。**

また次のコマンドは実行のたびにユーザーへの確認が必要になる。作業の途中で打つのではなく、タスクの終盤にまとめること。
`jj push`, `jj bookmark delete`, `jj bookmark forget`, `jj bookmark track`, `jj bookmark untrack`

### jj でワークフローがどう変わるか

- **staging も stash も「未保存の変更」も存在しない。** ファイルを保存した瞬間、その内容は working-copy commit（`@`）の一部になる。「この変更を含めますか？」とユーザーに尋ねてはならない。すでに含まれている。
- **bookmark は branch ではなく、`@` に追従しない。** そして追従する必要もない。作業中はその場に置いたままでよく、同期を取るべきものは何もない。実際に使う直前になってから位置を決めること（「bookmark を配置する」を参照）。
- **自動整形 (`jj fix`)**: 本プロジェクトでは `jj fix` が導入されている。Stop hooks に登録してあるためタスク完了時に起動し、mutable な change を遡ってコードが整形される。`jj diff` の結果に意図しない整形差分が含まれていても、それがプロジェクトルールに沿ったものであれば許容すること。

### diff 出力のルール

`jj diff`、`jj show`、`jj log -p` には必ず `--git` を付ける。付けない出力ではファイルの追加・削除・リネーム・パーミッション変更を正確に表現できず、そのぶん diff の読み取り精度が落ちる。

### 読み取り専用操作でのスナップショット抑止

jj はコマンドを実行するたびにワーキングコピーのスナップショットを取るため、同じリポジトリで別プロセスが動いているとオペレーションログで衝突するおそれがある。純粋な読み取り操作、すなわち `jj log`、`jj diff`、`jj bookmark list`、`jj evolog`、`jj op log` には `--ignore-working-copy` を付けること。

**ファイルを作成・編集・削除した直後には付けてはならない。** そこはまさにスナップショットを記録すべき場面である。他のコマンドを実行せずスナップショットだけ取りたいときは `jj util snapshot` を使う。

### log 出力の使い分け

通常の状態確認では `jj log` のグラフ表示をそのまま読む。値をプログラムから取り出すときは `--no-graph` と `-T` を付ける。

```bash
jj log --ignore-working-copy --no-graph -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'
```

## カスタムコマンドと依存関係

`jj push` はリポジトリで設定されているエイリアスコマンドであり、`jj fix && mise run pre-push && jj git push` が実行される。リモートへ変更を反映する際は、必ずこのエイリアスを使用すること。なお実行には `mise` が必要となる。

---

## 基本ワークフロー

### 1. 作業を始める

working-copy commit（`@`）の状態に応じて動きを変える。

1. `jj status` を実行する。ここでは `--ignore-working-copy` を **付けない**。上のルールに対する唯一の例外である。直前の jj コマンド以降にユーザーがファイルを編集している可能性があり、スナップショットを飛ばすとその編集が存在しないものとして報告されてしまう。
2. **`@` が空なら**（description も diff も持たない）→ **そのまま再利用する。** `jj describe -m "<description>"` で description を設定し、その中で作業する。**新しい change を作ってはならない。** `jj new` で作られた change もそれ自体は空なので、このルールに該当する。
3. それ以外（すでに description もしくは diff がある）→ `jj new -m "<description>"` を実行する。
4. description は英語の Conventional Commits 形式で書く。

### 2. change を分割する

```bash
jj split -m "<切り出す change の description>" <path>...
```

diff エディタを開かせないのは fileset、説明文エディタを開かせないのは `-m` である。hunk 単位の分割にはそのエディタが必要なので、ユーザーに委ねること。

### 3. conflict を解消する

マーカーの入ったファイルを直接編集し、`jj status` で conflict が消えたことを確認する。ファイルを保存すること自体が解消であり、その後に `git add` に相当する操作は要らない。

jj の conflict マーカーは Git のものとは異なる。`%%%%%%%` のブロックはベースからの差分を `-`/`+` の行で表したもので、`+++++++` のブロックはもう一方のそのままの内容である。

### 4. bookmark を配置する

bookmark を動かすのは、push を頼まれたときと、push の手順が暗黙に含まれる形で PR の作成を頼まれたときだけ。それ以外では触れないこと。change の存在に bookmark は必須ではない。

中身を見ないまま `@` に向けてはならない。`jj new` の直後の `@` は description のない空の change であり、description がない commit の push は拒否される。`@` の祖先のうち diff と description の両方を持つ最も新しい change を対象とし、動かす前に読み返すこと。

```bash
jj log --ignore-working-copy --no-graph \
  -r 'heads(::@ & ~empty() & ~description(exact:""))' \
  -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'

jj bookmark create <name> -r 'heads(::@ & ~empty() & ~description(exact:""))'  # 初回
jj bookmark move <name> --to 'heads(::@ & ~empty() & ~description(exact:""))'  # 2 回目以降
```

**`jj bookmark advance` は使わないこと。** 対象を `revsets.bookmark-advance-to` から取るため、着地点がコマンドの記述ではなくリポジトリの設定で決まってしまう。呼び出す場所で対象を明示すること。

### 5. リモートと同期する

```bash
jj git fetch                      # リモートの最新状態を取得する
jj push -b <bookmark-name>    # bookmark を push する
```

---

## PR 作成ワークフロー

- ターゲットは指示がない限り常に `main`。確認は不要。
- **squash しない。** 変更履歴の追跡可能性を保つため、各 change はそのまま残す。
- タイトルは `--title` で明示的に渡す。bookmark の先頭 change の description から導いた Conventional Commits 形式にすること。`gh` が自動生成するタイトルに任せない。

```bash
jj git fetch
jj log --ignore-working-copy
jj bookmark list --all --ignore-working-copy   # 確認後、上記の手順で配置する
jj bookmark track <bookmark-name>@origin       # 未追跡の場合のみ
jj push -b <bookmark-name>                 # ローカルとリモートが一致していれば不要
gh pr create --base main --head <bookmark-name> --title "<type>: <summary>" --body "<body>"
```

---

## トラブルシューティング

**「The working copy is stale」** — 同じリポジトリで人間と AI が並行して作業したか、外部ツールがファイルを書き換えた場合に起きる。`jj workspace update-stale` を実行し、`jj status` で正常に戻ったことを確認する。
