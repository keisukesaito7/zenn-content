---
title: "[gitmoji-cli] コミットメッセージで簡単に絵文字を使う方法"
emoji: "✨"
type: "tech"
topics: ["Git", "GitHub", "gitmoji"]
published: true
---

## 概要

`gitmoji-cli` を使うと、コミットメッセージに絵文字を簡単に含められるようになります。
本記事では、`gitmoji-cli`の導入方法と使い方について、簡単に紹介します。

## gitmoji-cli のインストール

[GitHub](https://github.com/carloscuesta/gitmoji)のインストール手順に則って、`gitmoji-cli`を導入していきます。

```
$ npm i -g gitmoji-cli
```

はい。これで終わりです。インストールできたか確認してみましょう。

```
$ gitmoji -v
#=> 4.1.0
```

このようにバージョンが表示されていればインストール成功です！

:::message
もし`gitmoji -v`でバージョンを確認したとき、`command not found`となってしまった場合は、一度ターミナルを再起動してみてください。
:::

## 使ってみる

使う手順としては、以下の通りです。

```
# ファイルに何か変更を加える
$ git add .
$ gitmoji -c
```

通常では`git commit`とするところを、`gitmoji -c`とします。
すると、ターミナルで絵文字が選択できるようになります（検索も可能）。

[![Image from Gyazo](https://i.gyazo.com/fbb9445e67360bc647db4ac3f2566116.gif)](https://gyazo.com/fbb9445e67360bc647db4ac3f2566116)

絵文字を選択したら、次はコミットのタイトル、最後にコミットのメッセージを入力し、完了です。

[![Image from Gyazo](https://i.gyazo.com/362d60ade52ac21e8c6649c003eb7355.png)](https://gyazo.com/362d60ade52ac21e8c6649c003eb7355)

GitHub で見ると、こんな感じです。

## おわりに

以上が、`gitmoji-cli`のインストールと簡単な使い方です。
絵文字で見やすくなりますし、コミットの種類も意識しながら実装できそうで良さそうです。

コミットメッセージに絵文字を含めたい方（主に初学者）の方は、ぜひ試してみて下さい！
