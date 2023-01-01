---
title: "AWS Lambda の環境変数が GitHub Actions で見えちゃってたので KMS で暗号化しました"
emoji: "🔐"
type: "tech"
topics: ["AWS", "Lambda", "KMS", "GitHubActions", "GitHub"]
published: true
---

## 概要

AWS Lambda では、環境変数を設定できます。
私はこれまで、この環境変数を秘匿情報と思い込んでいました。

しかし、コンソール上では平文で表示されますし、API を使った際に表示されるケースもあるしで、必ずしも秘匿性があるわけではないと知りました。

![](/images/github-actions-lambda-env-kms/lambda-env-visible.png)

とりわけ、この度「え、これみんなに見えちゃってね？」という危険を味わったエピソードがありました。

そのエピソードを紹介しつつ、環境変数を暗号化する方法を共有します。

## エピソード

- GitHub Actions で Lambda のコードを自動でアップロードしよう！
- よしよし Action 動いてるな、、あれ？
- めっちゃ環境変数見えちゃってるじゃん！

![](/images/github-actions-lambda-env-kms/episode-github-actions-visible-env.png)

GitHub Actions は、リポジトリが Public だと内容も公開されてしまいます。
つまり、自分以外のアカウントで Lambda に設定した環境変数が丸見えとなってしまいます。

というわけで、これをどうにかしていきます。

## 対応方法の選定

私が考えた中で、取れる選択肢は以下の 3 つでした。

1. AWS KMS を使用して環境変数を暗号化
2. AWS SSM パラメータストアに値を登録して参照
3. GitHub のリポジトリを private にする

今回は以下の理由から **1 の KMS を使用する手段**を選択しました。

- Lambda のコンソール上で説明されている手法だったため (公式が推奨しているのかと)
- 当該 Lambda でしか使用しない環境変数であり、パラメータストアに共通化する必要がなかっため
- ソースコードを公開したかったため

## デモのゴール

GitHub Actions の job のレスポンスで、AWS Lambda の環境変数が暗号化されている状態を目指します。

:::message

記事を見やすくするため、以下の内容を省略しています

(1) GitHub OIDC を用いた AWS 認証
(2) Lambda で使用するソースコード

これらは後述の「Appendix」で補足します。
:::

使用した workflow の例は以下の通りです。
`Update Lambda`の job のレスポンスが対象です。

:::details GitHub Actions workflow
```yml
name: Update Lambda

on:
  push:
    branches:
      - main

# 認証に必要
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Git Clone the Repository
      uses: actions/checkout@v3

    - name: Set Up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: '2.7.6'
        bundler-cache: true

    # Lambda にアップロードするコードを zip 化
    - name: Compress Zip
      run: |
        zip -r package.zip lambda_function.rb vendor

    # AWS 認証
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ secrets.ROLE_NAME }}
        role-session-name: OidcLambdaUpdateRuby
        aws-region: ${{ secrets.AWS_REGION }}

    # Lambda のコードを update
    - name: Update Lambda
      run: |
        aws lambda update-function-code --function-name oidc-lambda-deploy-ruby --zip-file fileb://package.zip
```
:::


## 手順 1. KMS でキーを作成

KMS > カスタマー管理型のキー > キーの作成 に移動します。

以下の設定で作成しました。

- キーのタイプ: 対象
- キーの使用: 暗号化および復号化
- 詳細オプション: デフォルトのまま
- エイリアス: oidc-lambda-deploy-ruby (任意)
- 説明: (任意)
- キー管理者: 設定なし
- キーの使用アクセス許可: 対象の Lambda のロールを指定

![](/images/github-actions-lambda-env-kms/kms-key.png)

ロール名は Lambda のコンソールで確認できます。

![](/images/github-actions-lambda-env-kms/lambda-role-name.png)

## 手順 2. Lambda で環境変数を暗号化

Lambda > 関数 > 対象の関数 > 設定 > 環境変数 > 編集 に移動します。

この時点ではまだ平文です。

![](/images/github-actions-lambda-env-kms/lambda-env-1-plain.png)

「転送時の暗号化に使用するヘルパーの有効化」にチェックを入れると、環境変数部分に「暗号化」ボタンが表示されるので、それを押します。

「転送時の暗号化を行うための AWS KMS キー」に作成した鍵が表示されるのでそちらを選択します。

![](/images/github-actions-lambda-env-kms/lambda-env-2-set-key.png)

なお、暗号化した場合は、「シークレットスニペットの復号」に沿ってソースコードを変更する必要があるのでご注意ください。

:::details シークレットスニペットの復号
```ruby
require 'aws-sdk-kms'
require 'base64'

ENCRYPTED = ENV['SLACK_WEBHOOK_URL']
# Decrypt code should run once and variables stored outside of the function
# handler so that these are decrypted once per container
DECRYPTED = Aws::KMS::Client.new
    .decrypt({
        ciphertext_blob: Base64.decode64(ENCRYPTED),
        encryption_context: {'LambdaFunctionName' => ENV['AWS_LAMBDA_FUNCTION_NAME']},
    })
    .plaintext

def lambda_handler(event:, context:)
    # TODO handle the event here
end
```
:::


「暗号化」を押すと、下記画像の通り暗号化されます。

![](/images/github-actions-lambda-env-kms/lambda-env-3-encrypted.png)

## 結果

ここまでできたら、再度 GitHub Actions で Re-run してみます。
レスポンスを見ると、環境変数が暗号化されていました！🎉

![](/images/github-actions-lambda-env-kms/result-encrypted-env.png)

## おわりに

無事に暗号化できてよかったです。
これからもこういった落とし穴に注意してインフラ構築していきたいです！

## Appendix

### 1. 費用

AWS KMS の使用量がかかります。
おおむね 100 円ちょっとくらいです。

- 鍵作成
  - $1 / month
- リクエスト
  - 無料枠: 20,000 requests/month
  - 課金: $0.03 per 10,000 requests

https://aws.amazon.com/jp/kms/pricing/

### 2. GitHub Actions から AWS への認証について

GitHub OIDC を用いて AWS に認証しています。
手順は下記を参照ください。

https://zenn.dev/k_saito7/articles/github-actions-oidc-aws

### 3. ソースコードの変更について

環境変数が暗号化されたため、ソースコード上ではデコードした上で値を参照する必要があります。
今回は以下のように実装しました。

[挙動]
任意の文字列を Slack 通知する

```ruby:lambda_function.rb
require 'slack-notifier'
require 'aws-sdk-kms'
require 'base64'

ENCRYPTED_SLACK_WEBHOOK_URL = ENV['SLACK_WEBHOOK_URL']
DECRYPTED_SLACK_WEBHOOK_URL = Aws::KMS::Client.new
  .decrypt({
    ciphertext_blob: Base64.decode64(ENCRYPTED_SLACK_WEBHOOK_URL),
    encryption_context: {'LambdaFunctionName' => ENV['AWS_LAMBDA_FUNCTION_NAME']},
    })
  .plaintext

def lambda_handler(*)
  message = 'Hi, Jun !! You are No.1 !!'
  post_message(message)
end

def post_message(message)
  notifier = Slack::Notifier.new DECRYPTED_SLACK_WEBHOOK_URL
  notifier.ping message
end
```

[補足]
Lambda で実行する場合は、`Aws::KMS::Client.new` に必要なクレデンシャルと、`ENV['AWS_LAMBDA_FUNCTION_NAME']` の設定は不要です。

## 参考

1. [AWS 公式 | Using AWS Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html?icmpid=docs_lambda_help#configuration-envvars-encryption)
2. [AWS 公式 | AWS SDK for Ruby | Decrypting a Data Blob in AWS KMS (ローカルで試すときの参考)](https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/kms-example-decrypt-blob.html)
3. [Github Actionsを利用してAWS Lambdaに自動デプロイをしてみた](https://dev.classmethod.jp/articles/lambda-github-actions/)
4. [AWS Lambdaで秘密情報をセキュアに扱う - アンチパターンとTerraformも用いた推奨例の解説](https://blog.flatt.tech/entry/lambda_secret_security)