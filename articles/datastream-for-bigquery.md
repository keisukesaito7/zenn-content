---
title: "[GCP Datastream] AWS RDS から BigQuery へのレプリケーションを試してみた"
emoji: "📝"
type: "tech"
topics: ["GCP", "BigQuery", "datastream", "AWS", "RDS"]
published: false
---

## 概要

PostgreSQL や MySQL などの運用データベースから、GCP のデータウェアハウスである BigQuery に、直接かんたんにデータをレプリケートできる [Datastream for BigQuery](https://cloud.google.com/datastream-for-bigquery) のプレビュー版が提供されました。
BigQuery のスキーマ定義、BigQuery に適した形へのデータ変換、データを BigQuery に送信する日バッチ処理などが不要になることが期待されたので、試してみました。

## 構成

本記事では以下の構成で GCP Datastream for BigQuery を試しました。

- AWS EC2 : RDS の踏み台サーバー
- AWS RDS (MySQL) : ソース
- GCP BigQuery : レプリカ
- GCP Datastream : RDS のデータを BigQuery にレプリケート

## 手順

[Datastream 公式ドキュメント](https://cloud.google.com/datastream/docs/how-to?hl=ja)に沿って進めました。
ソースの構成について Datastream 指定の設定に変える必要がありましたが、[AWS での操作手順](https://cloud.google.com/datastream/docs/configure-your-source-mysql-database?hl=ja)も細かく書かれており、基本的にはつまづかずに進められました。

(バイナリログについてだけ知見が少なかったので自分用にメモ)

:::details バイナリログについて

バイナリログとは？

- データベースに行われた変更が記録される
- レプリケーション、データリカバリで使用される
- レプリケーションは、ソース側のバイナリログがレプリカ側に送信され、ソース側と同じデータ変更を行うことで実現する

このバイナリログには、以下 3 つの記録形式があります。

- SQL ステートメントベース : STATEMENT
- テーブルの行ベース : ROW
- 上記 2 つの混合 : MIXED

各形式にトレードオフがありますが、GCP Datastream では、ソースの MySQL のバイナリログ形式が `ROW` であることが要求されています。
そのため、RDS のデフォルトパラメータグループを以下のように変更しました。

![](/images/datastream-for-bigquery/rds-parameter-group.png)

バイナリログについての詳細は以下のリンクから
https://dev.mysql.com/doc/refman/8.0/ja/binary-log.html
:::
## 接続プロファイルを作成

Datastream と RDS の接続設定をします。
マニュアルでいうと[この部分です](https://cloud.google.com/datastream/docs/create-connection-profiles?hl=ja)。

フォワード SSH トンネルもサポートされており、踏み台サーバー経由で RDS にアクセスできるのが良いなと思いました。

:::details 参考画像
![](/images/datastream-for-bigquery/datastream-ssh-forward.png)
:::

## ストリームの作成

RDS から BigQuery にデータを転送する設定をします。
マニュアルでいうと[この部分です](https://cloud.google.com/datastream/docs/create-a-stream?hl=ja)。

### ストリーミングするデータを選べる

RDS のテーブル情報を閲覧でき、ストリーミングするテーブル・カラムを選ぶことができます。
GUI でポチポチするだけで BigQuery のスキーマを決められるのが非常に快適です。
他にも、秘匿情報を取得しない、ストリーミング量を減らせる、などのメリットがありそうです。

![](/images/datastream-for-bigquery/datastream-select-tables-columns.png)

### データの鮮度を調整できる

データの鮮度に関しても選択できます。
より鮮度が高い情報が欲しい場合はこちらをいじれば良さそうです（料金と相談ですが）。

![](/images/datastream-for-bigquery/datastream-streaming-interval.png)

## ストリームを開始

無事に接続できたので、ストリームを開始してみます！

![](/images/datastream-for-bigquery/datastream-ready-to-streaming.png)

### 初回データ反映

数分後、BigQuery のデータセット `datastream` がプロジェクト下に自動で作られ、RDS からレコードがレプリケートされていました。

![](/images/datastream-for-bigquery/bigquery-users-first.png)

### CDC (Change Data Capture)

RDS の users テーブルから、 `id = 1` を削除し、2 つレコードを追加しました。
30 分ほどかかった後、BigQuery からも `id = 1` が削除され、新しいレコードが 2 つ追加されました。

![](/images/datastream-for-bigquery/bigquery-users-second.png)

## おわりに

想像以上に簡単に RDS のデータを BigQuery にレプリケートできました。
データ分析を行う際の引き出しが増えて嬉しいです。

また気になるサービスがあれば試してみようかなと思います。
