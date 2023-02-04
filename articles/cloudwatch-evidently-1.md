---
title: "Amazon CloudWatch Evidently でフィーチャーフラグを試してみた"
emoji: "⛳"
type: "tech"
topics: ['AWS', 'CloudWatch', 'Ruby']
published: false
---

## 概要

[AWSで実現するモダンアプリケーション入門](https://www.amazon.co.jp/dp/B0BRPRQFMH/) を読み、Amazon CloudWatch Evidently という AWS のサービスでフィーチャーフラグが実現できることを知りました。

フィーチャーフラグとは、デプロイされているコードに変更を加えずに、アプリケーションの振る舞いを動的に変更できる仕組みです。
この仕組みにより、以下のようなメリットが得られます。

- feature ブランチ肥大化など、開発上の問題の解消
- 一部ユーザーのみ機能開放など、運用上の問題の解消

書籍内では、フィーチャーフラグ実現のために以下の手法が紹介されています。

- 環境変数
- SSM パラメータストア
- CloudWatch Evidently

これを見て「CloudWatch Evidently って触ったことないな」と思ったので、「とりあえず触ってみた」というのがこの記事の概要です。

## 動作環境

- macOS Monterey : 12.6.1
- Ruby : 3.2.0
- gem [aws-sdk-cloudwatchevidently](https://rubygems.org/gems/aws-sdk-cloudwatchevidently?locale=ja) : 1.10.0

## 環境構築

test_project というディレクトリ内で作業します。

```
$ mkdir test_project && cd test_project
$ bundle init
$ bundle config set --local path vendor/bundle
```

Gemfile に gem 'aws-sdk-cloudwatchevidently' を追加します。

```ruby:Gemfile
# frozen_string_literal: true

source "https://rubygems.org"

gem 'aws-sdk-cloudwatchevidently'
gem 'rexml' # xml ライブラリがないとエラーが出るので入れた
```

:::details rexml を入れた理由について

rexml なしで後述のスクリプトを実行したところ、下記のようなエラーが出たためです。

> /test_project/vendor/bundle/ruby/3.2.0/gems/aws-sdk-core-3.170.0/lib/aws-sdk-core/xml/parser.rb:74:in `set_default_engine': Unable to find a compatible xml library. Ensure that you have installed or added to your Gemfile one of ox, oga, libxml, nokogiri or rexml (RuntimeError)

:::

bundle install したら完了です。

```
$ bundle install
```

## AWS 側の準備

### 認証情報

IAM ポリシー AmazonCloudWatchEvidentlyFullAccess をアタッチしたユーザー。

:::details FullAccess にした理由

ReadonlyAccess のポリシーでスクリプトを実行したところ、以下のエラーが発生したため。

> because no identity-based policy allows the evidently:EvaluateFeature action (Aws::CloudWatchEvidently::Errors::AccessDeniedException)

:::

### CloudWatch Evidently でプロジェクト作成

1. CloudWatch のコンソールの左ペインから「Evidently」を選択
2. project を新規作成
  a. プロジェクト名 : test_project
  b. 評価イベントストレージ : CloudWatch Logs (デフォルト)
  c. クライアント側の評価を使用する : チェックなし (デフォルト)

![](/images/cloudwatch-evidently-1/create_project.png)

3. test_project を選択
4. 機能を追加
  a. 機能名 : test_feature_1
  b 機能のバリエーション : true/false を用意

![](/images/cloudwatch-evidently-1/create_feature_1.png)

## コード

今回のデモでは以下のスクリプトを実行します。
user1 と user2 がいて、それぞれの flag がどうなるかを puts しているシンプルなコードです。

```ruby:src/main.rb
require 'aws-sdk-cloudwatchevidently'

def flag(client:, entity_id:, project:, feature:)
  res = client.evaluate_feature(
    {
      entity_id:,
      project:,
      feature:
    }
  )
  res.value.bool_value
end

def puts_result(entity_id:, flag:)
  puts "#{entity_id} は #{flag} です！"
end

# ※ 環境変数 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY を設定してください
client = Aws::CloudWatchEvidently::Client.new
project = 'test_project'
feature = 'test_feature_1'

# user1
entity_id = 'user1'
puts_result(entity_id:, flag: flag(client:, entity_id:, project:, feature:,))

# user2
entity_id = 'user2'
puts_result(entity_id:, flag: flag(client:, entity_id:, project:, feature:,))
```

### デフォルト値の変更

いったん既存の設定のままで実行してみると、デフォルト値が採用され、どちらも false になります。

```
$ bundle exec ruby src/main.rb
user1 は false です！
user2 は false です！
```

試しに、デフォルトを true にしてみましょう。

1. CloudWatch > Evidently > test_project > test_feature_1 > 機能を編集
2. feature_on の方をデフォルトに変更

![](/images/cloudwatch-evidently-1/change_default_value.png)

ソースコードを変更しない状態で再度実行してみます！

```
$ bundle exec ruby src/main.rb
user1 は true です！
user2 は true です！
```

user1、user2 ともに true となりました！
「非公開状態の機能を公開する」というユースケースに使えそうですね。

### 特定のユーザーにのみ機能開放

オーバーライドを追加することで、特定のパターンでフラグの分岐をすることができそうでした。

1. CloudWatch > Evidently > test_project > test_feature_1 > 機能を編集
2. feature_off の方をデフォルトに変更
3. オーバーライドトグルを開き、user1 を feature_on に設定

![](/images/cloudwatch-evidently-1/override_user1_feature_on.png)

ここでの user1 は、evaluate_feature の引数で渡している entity_id と一致しています。

```ruby
client.evaluate_feature(
  {
    entity_id:,
    project:,
    feature:
  }
)
```

ソースコードを変更しない状態で再度実行してみます！

```
$ bundle exec ruby src/main.rb
user1 は true です！
user2 は false です！
```

user1 はオーバーライドされた値、user2 はデフォルト値となりました。
「特定のユーザーにのみ機能開放」というユースケースで使えそうですね。

ただ、オーバーライドの個数は最大で 20 個までのようでした。
なのでユーザーIDを直接指定するのは、場合によっては要件を満たせないのかな、、と思いました。

### 機能 ON のタイミングをスケジューリング

起動設定なる項目がありました。
詳細を見るに、「指定の時間にどういう状態にするか」を設定できそうでした。

せっかくなのでこれも試してみます。

1. CloudWatch > Evidently > test_project > 起動を作成
2. 任意の時間で feature_on が 100 % になるようにスケジュール

![](/images/cloudwatch-evidently-1/all_user_feature_on.png)

ソースコードを変更しない状態で再度実行してみます！

```
# 17:10 前に実行しておく
$ bundle exec ruby src/main.rb
user1 は true です！
user2 は false です！
$ date
Sat Feb  4 17:09:58 JST 2023

# 17:10 以降に実行してみる
$ date
Sat Feb  4 17:10:02 JST 2023
$ bundle exec ruby src/main.rb
user1 は true です！
user2 は true です！
```

17:10 より前は user2 のみ false でしたが、17:10 以降は user2 も true になりました！すごい🎉
「この時間に指定の割合で機能を開放したい」というユースケースで使えそうですね。

ちなみに、現状の値の設定がどうなっているか、というと、、

![](/images/cloudwatch-evidently-1/after_schedule_all_user_feature_on.png)

- デフォルトが変わるわけではない
- feature_on が削除不可能になっている

という状態でした。

また、「起動をキャンセル」という項目があったので実行してみたところ、スケジュール前の状態に戻りました。

```
$ bundle exec ruby src/main.rb
user1 は true です！
user2 は false です！
```

### 実験

調べたところ、いわゆる A/B テストとして使用できる機能のようでした。
ソースコードの変更が必要そうなので、今回はスキップします。

## まとめ

Amazon CloudWatch Evidently でフィーチャーフラグを試してみました。

機能開放をスケジュールできる部分がとても良いと思いました。
1 つの PJ で複数の機能を追加できるので、いろんな要望に対応できそうです。実務で使う機会があれば試してみようと思います。
