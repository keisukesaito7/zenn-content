---
title: "[GCP Datastream] AWS RDS から BigQuery に簡単にレプリケーションできるか試してみた"
emoji: "✨"
type: "tech"
topics: ["GCP", "BigQuery", "datastream", "AWS", "RDS"]
published: false
---

## 概要

PostgreSQL や MySQL などの運用データベースから、GCP のデータウェアハウスである BigQuery に、直接かんたんにデータをレプリケートできる [Datastream for BigQuery](https://cloud.google.com/datastream-for-bigquery) のプレビュー版が提供されました。
BigQuery のスキーマ定義、BigQuery に適した形へのデータ変換、データを BigQuery に送信する日バッチ処理などが不要になることが期待されたので、試してみました。

## 構成

本記事では以下の構成で GCP Datastream for BigQuery を試しました。

- EC2        : RDS の踏み台サーバー
- RDS        : レプリケート元データベース
- BigQuery   : レプリケート先データウェアハウス
- Datastream : RDS のデータを BigQuery にレプリケート

## 手順の概要

以下の内容に沿って進める。

https://cloud.google.com/datastream/docs/how-to?hl=ja

- ソース側の構成について: https://cloud.google.com/datastream/docs/configure-your-source-mysql-database?hl=ja

:::details バイナリログについて

バイナリログとは？

- データベースに行われた変更が記録されている
- レプリケーション、データリカバリで使用される
- レプリケーションは、ソース側のバイナリログがレプリカ側に送信され、ソース側と同じデータ変更を行うことで実現する

このバイナリログには、以下 3 つの記録形式があります。

- SQL ステートメントベース : STATEMENT
- 個々のテーブルの行ベース : ROW
- 上記 2 つの混合 : MIXED

各形式にトレードオフがありますが、GCP Datastream では、ソースの MySQL のバイナリログ形式が `ROW` であることが要求されています。
そのため、RDS のデフォルトパラメータグループを以下のように変更しました。

![](/images/datastream-for-bigquery/rds-parameter-group.png)

バイナリログについての詳細は以下のリンクから
https://dev.mysql.com/doc/refman/8.0/ja/binary-log.html
:::
## 接続プロファイルを作成

https://cloud.google.com/datastream/docs/create-connection-profiles?hl=ja

ここで Datastream と RDS の接続を設定します。
SSH トンネルもサポートされており、踏み台サーバー経由で RDS にアクセスできるのが良いなと思いました。

![](/images/datastream-for-bigquery/datastream-ssh-forward.png)

## ストリームの作成

https://cloud.google.com/datastream/docs/create-a-stream?hl=ja

記事作成段階では、まだプレビュー版ですね。
![](/images/datastream-for-bigquery/datastream-bigquery.png)

ストリーミングするテーブルと、カラムも選ぶことができます。
秘匿情報を取得しない、ストリーミング料を減らすことができますね。

![](/images/datastream-for-bigquery/datastream-select-tables-columns.png)

データの鮮度に関しても選択できました。BigQuery を使用する用途によってはここで費用を抑えられそうですね。

![](/images/datastream-for-bigquery/datastream-streaming-interval.png)

接続検証が OK だったのでつないでみます！

![](/images/datastream-for-bigquery/datastream-ready-to-streaming.png)

## レプリケーションの確認

### バックフィル

数分後、初回データ反映。
BigQuery のデータセット `datastream` がプロジェクト下に自動で作られ、RDS からレコードがレプリケートされていました。

![](/images/datastream-for-bigquery/bigquery-users-first.png)

### CDC

上記の users テーブルから、 id = 1 を削除し、2 つレコードを追加しました。
こちらに関しては反映まで 30 分ほどかかりましたが、BigQuery と RDS が同じレコードとなりました。

![](/images/datastream-for-bigquery/bigquery-users-second.png)

## おわりに