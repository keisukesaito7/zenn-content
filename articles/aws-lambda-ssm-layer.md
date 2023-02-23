---
title: "「AWS Lambda 関数での Parameter Store パラメーターの使用」を Ruby で試してみた"
emoji: "🔑"
type: "tech"
topics: [AWS, Lambda, ssm ,parameterstore, Ruby]
published: false
---

## 概要

AWS Lambda から AWS System Manager Parameter Store にアクセスするには、通常は SDK を使用する必要があります。例えば Ruby だと [Gem aws-sdk-ssm](https://rubygems.org/gems/aws-sdk-ssm/versions/1.0.0.rc7) です。

これを Lambda の拡張機能で簡単に利用できる方法があると知ったので、まとめます。

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/ps-integration-lambda-extensions.html

## 手順

1. Lambda 関数作成
2. 拡張機能を追加
3. Lambda の 設定を変更
4. Lambda 関数を書く

## 1. Lambda 関数作成

- 関数名 : 任意
- ランタイム : Ruby2.7
- その他はデフォルトのまま

で、関数を作成しました。

## 2. 拡張機能を追加

> Lambda 関数への拡張機能の追加
> AWS Parameters and Secrets Lambda Extension を使用するには、拡張機能を Lambda 関数にレイヤーとして追加します。
> 関数に拡張機能を追加するには、以下のいずれかの方法を使用します。

今回は AWS マネジメントコンソールで、Lambda のレイヤーを追加します。

> 1. AWS Lambda コンソールを (https://console.aws.amazon.com/lambda/) 開きます。
> 2. 関数を選択します。[Layers] (レイヤー) エリアで、[Add a layer] (レイヤーの追加) を選択します。
> 3. [Choose a layer] (レイヤーを選択) エリアで、[AWS layers] オプションを選択します。
> 4. [AWS layers] で、[AWS-Parameters-and-Secrets-Lambda-Extension] を選択し、バージョンを選択してから [Add] (追加) を選択します。

手順に沿って追加しました。

![add_extension_ssm_parameter_store](/images/aws-lambda-ssm-layer/add_extension_ssm_parameter_store.png)

## 3. Lambda の設定

> 実装の詳細
> AWS パラメータとシークレット Lambda 拡張機能の設定に役立つ以下の詳細情報を参考にしてください。

を読んで、必要な設定を行います。

### Lambda の IAM ロールを更新

> 認証
> 関数の実行に使用する AWS Identity and Access Management (IAM) ロールには、Parameter Store と対話するための次の権限が必要です。
> - ssm:GetParameter — Parameter Store からパラメータの取得に必要
> - kms:Decrypt — Parameter Store から SecureString パラメータを取得するために必要

というわけで IAM ロールを更新します。

`ssm:GetParameter`と`kms:Decrypt`の許可ポリシーは、今後もこの拡張機能を使うことに備え、新しく`AllowSystemManagerGetParameterAndKmsDecrypt`というポリシーで作成しておきました。

```yml:IAM ポリシー AllowSystemManagerGetParameterAndKmsDecrypt
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "ssm:GetParameter"
      ],
      "Resource": [
        "arn:aws:kms:ap-northeast-1:{AWS_ACCOUNT_ID}:key/{aws/ssm のキー ID}",
        "arn:aws:ssm:ap-northeast-1:{AWS_ACCOUNT_ID}:parameter/*"
      ]
    }
  ]
}
```

`aws/ssm のキー ID`は、パラメータの暗号化に使用している KMS のキー ID を指定します。

![kms_key_id](/images/aws-lambda-ssm-layer/kms_key_id.png)

parameter は`*`としてすべてのパラメータを許可しておきます。
もし個別でパラメータを許可したい場合は、以下のように設定してください。

```yml:個別でパラメータを許可したい場合
"Resource": [
  "arn:aws:kms:ap-northeast-1:{AWS_ACCOUNT_ID}:key/{aws/ssm のキー ID}",
  "arn:aws:ssm:ap-northeast-1:{AWS_ACCOUNT_ID}:parameter/test_key",
  "arn:aws:ssm:ap-northeast-1:{AWS_ACCOUNT_ID}:parameter/test/params/hoge/fuga"
]
```

Lambda の IAM ロールに、作成した`AllowSystemManagerGetParameterAndKmsDecrypt`をアタッチすれば完了です。

`AWS Key Management Service`と`AWS System Manager`のポリシーが追加されました。

![lambda_iam_role_updated](/images/aws-lambda-ssm-layer/lambda_iam_role_updated.png)

## 4. Lambda 関数を書く

ここまでできたらコードを書いていきます。
「概要」の通り、SDK を使う必要はありません。

パラメータの取得手順は`HTTP GET`で行います。
今回、必要な部分を抜き出すと以下の通りです。

```
GET http://localhost:port/systemsmanager/parameters/get?name=parameter-path&withDecryption=true
```

port はデフォルトで 2773 です。

> Localhost ポート
> GET リクエストに localhost を使用します。拡張機能は、localhost ポート 2773 に送信されます。

### 設定

今回は Parameter Store の SecureString パラメータを Lambda で表示することを目指します。

- `test_key`
- `/test/params/hoge/fuga`

![parameter_store](/images/aws-lambda-ssm-layer/parameter_store.png)

### Ruby のコード

Parameter Store にリクエストし、コンソールに表示するだけのシンプルなプログラムを作成しました。

```ruby:lambda_function.rb
require_relative './request_param'

def lambda_handler(event:, context:)
  test_key = request_param(key_path: 'test_key')
  path_key = request_param(key_path: '/test/params/hoge/fuga')
  
  puts "test_key: #{test_key}"
  puts "path_key: #{path_key}"

  { statusCode: 200, body: 'OK' }
end
```

```ruby:request_param.rb
require 'net/http'
require 'uri'

def request_param(key_path:)
  query_string = URI.encode_www_form(name: key_path, withDecryption: true)
  uri = URI.parse("http://localhost:2773/systemsmanager/parameters/get?#{query_string}")
  http = Net::HTTP.new(uri.host, uri.port)
  req = Net::HTTP::Get.new(uri.request_uri)
  
  # header
  headers = {
    'X-Aws-Parameters-Secrets-Token': ENV['AWS_SESSION_TOKEN']
  }
  req.initialize_http_header(headers)
  
  # request
  response = http.request(req)

  JSON.parse(response.body, symbolize_names: true).dig(:Parameter, :Value)
end
```

:::details [補足] response.body の形
Parameter Store からのレスポンスは以下の通りです。

```ruby
"{\"Parameter\":{\"ARN\":\"arn:aws:ssm:ap-northeast-1:{AWS_ACCOUNT_ID}:parameter/test_key\",\"DataType\":\"text\",\"LastModifiedDate\":\"2023-01-05T23:52:51.09Z\",\"Name\":\"test_key\",\"Selector\":null,\"SourceResult\":null,\"Type\":\"SecureString\",\"Value\":\"this_is_secret_parameter\",\"Version\":1},\"ResultMetadata\":{}}"

# Name: test_key
# Value: this_is_secret_parameter
```
:::

実行すると以下の通りコンソールに出力されました🎉

```ruby
test_key: this_is_secret_parameter
path_key: this is path parameters
```

## おわりに

Lambda から Parameter Store に拡張機能でアクセスする方法を勉強しました。
公式のサンプルコードが Python だったので、Ruby を使いたい人の参考になれれば嬉しいです。

今回は行いませんでしたが、キャッシュの設定もできて API アクセスを節約できるようになるらしいので、実務レベルで使う場合はそこも合わせてやってみたいと思います。
