# 『じゅじゅちゅ！&nbsp; jj new で始める Jujuts × AI ワークフロー』サポートページ

このリポジトリは、『じゅじゅちゅ！&nbsp; jj new で始める Jujuts × AI ワークフロー』のサンプルコードおよび正誤表を提供するものです。

<img src="./images/jujuchu-covers.png" alt="りあクト！三部作 表紙画像" />

<br>

## ■ 書籍の紹介

### 概要

『じゅじゅちゅ！&nbsp; jj new で始める Jujuts × AI ワークフロー』は人気上昇中の新世代 VCS「**[Jujutsu](https://www.jj-vcs.dev/)**」の本格的な入門書です。  
内容は基本的な使い方だけでなく、Git と比較しながらのメンタルモデルの構築や AI エージェントと協働するための Tips、Jujutsu を実際に使っていくときに行き当たりがちな問題の対処法などに渡ります。

この 1 冊ですぐ Jujutsu を始めることができ、AI 支援開発に応用できるようになっています。

### 立ち読み用無料サンプル

53 ページ（本書 148 ページ中）の無料でお読みいただけるサンプル PDF を用意しています。ご自由にご覧ください。

- [無料サンプル（53 ページ）](./jujuchu-sample.pdf)

<br>

## ■ サンプルコード

本文中に掲載されている設定のソースコードについては、このリポジトリの以下の場所に格納されています。

- 第 3 章に掲載の各種設定： [`./config/ch3/`](/klemiwary/Juju-chu-ja/tree/main/config/ch3)
- 第 4 章に掲載の各種設定： [`./config/ch4/`](/klemiwary/Juju-chu-ja/tree/main/config/ch4)

<br>

## ■ 正誤表・更新情報

電子版は随時アップデートをかけていますので、購入サイトから最新版をダウンロードしてください。  
紙本に対応する正誤表・更新情報につきましては、お手持ちの書籍の奥付に記載されている情報をご確認いただき、以下の刷番に対応したページをご参照ください。

- [正誤表・更新情報](./errata.md)

<br>

## ■ 電子版の閲覧について

本書の電子版は **PDF** と **EPUB** の 2 つのファイル形式で提供されています。閲覧の方法については[「よりよい環境で電子版をお読みいただくために」](./ebook-tips.md)をご覧ください。

<br>

## ■ 目次

### 【① 言語・環境編】

#### 第 1 章　Jujutsu ってどんなツール？

- 1-1. Git は AI 支援開発と相性が悪い？
  - 冗長な 3 ステートモデル
  - コンテキスト切り替えのコストの高さ
  - 履歴の書き換えが複雑で危険
  - コマンドの副作用が大きく取り消しにくい
- 1-2. AI 時代に評価される Jujutsu の特徴
  - 作業ディレクトリが即 commit される
  - あらゆる操作が undo 可能
  - Conflict をただの状態として扱う

#### 第 2 章　使ってみよう Jujutsu

- 2-1. Jujutsu の環境構築
- 2-2. Jujutsu 体験ツアー
  - 2-2-1. リポジトリの初期化
  - 2-2-2. リポジトリの状態を確認する
  - 2-2-3. Change の操作
  - 2-2-4. リモートとやりとりする
- 2-3. Jujutsu の 3 つのログ
  - 2-3-1. Revision Log（`jj log`）
  - 2-3-2. Evolution Log（`jj evolog`）
  - 2-3-3. Operation Log（`jj op log`）
- 2-4. Jujutsu のメンタルモデル
  - 2-4-1. Change とは何か
  - 2-4-2. Branch と Bookmark のちがい
  - 2-4-3. 匿名 Branch で作業する
- 2-5. よく使う `jj` コマンド一覧

#### 第 3 章　実践 Jujutsu × AI ワークフロー

- 3-1. AI エージェントに Jujutsu を使わせる
  - 3-1-1. AI エージェントに Jujutsu を使わせる
  - 3-1-2. `jj` コマンドの Permissions 設定
  - 3-1-3. Hooks で `jj fix` を走らせる
- 3-2. 実例で見る Jujutsu × AI による開発プロセス
  - 3-2-1. AI が作成した change の粒度を整える
  - 3-2-2. push して PR を作成する
  - 3-2-2. workspace を使った並行開発

#### 第 4 章　Jujutsu の高度な術式

- 4-1. ターゲットを指定する冴えたやりかた
  - 4-1-1. Revset で Revision をスマートに指定する
  - 4-1-2. Fileset でファイルをスマートに指定する
- 4-2. 知ってると便利な裏技的コマンド
  - 4-2-1. `jj absorb`
  - 4-2-2. `jj arrange`
  - 4-2-3. `jj bookmark advance`
- 4-3. Git Hooks の代替戦術
- 4-4. Jujutsu の UI ツール
  - 4-4-1. jjui
  - 4-4-2. JJ View

#### 第 5 章　Jujutsu 問題解決ガイド

- 5-1.　FAQ
  - 5-1-1. Git との比較
    - ● Git にできて Jujutsu にできないことは？
    - ● merge コマンドはないの？
    - ● pull コマンドはないの？
    - ● Git の cherry-pick 相当のことがしたい
  - 5-2-2. ニッチな操作・設定について
    - ● `@` を移動させずに、任意の時点でのファイルの内容を確認できる？
    - ● change の分割を時系列で行いたい
    - ● 一時的なログやダンプなど、履歴に含めたくないファイルがある
    - ● Jujutsu のリポジトリ用設定を、そのリポジトリ自身に登録したい
    - ● track 中の bookmark の名前を変更したい
  - 5-2-3. Jujutsu のトリビア
    - ● Jujutsu は Git のラッパーツールなの？
    - ● Jujutsu の作者ってどんな人？
    - ● プロダクト名の意味は「呪術」「柔術」どっち？
- 5-2.　トラブルシューティング
  - ● ただの push が知らないうちに force push になっている
  - ● 『Error: The working copy is stale』という謎のエラーが出る
  - ● いつのまにか change に divergent という注釈がついていた
  - ● 画像や動画ファイルを Jujutsu が追跡してくれない
  - ● PR をマージ後に fetch したら `@` が迷子になる
  - ● まだ作業中のリモートの bookmark を GitHub 上から削除してしまった
  - ● Claude Code の Permissions 設定で `jj log` を allow にしていても実行許可を求められる

---

© 2026 Yuka Ooka / [Klemiwary Books](https://klemiwary.com/)
