---
title: "[GitHub OIDC] OIDC を使って GitHub Actions から AWS にアクセス"
emoji: "🔒"
type: "tech"
topics: ["GitHubActions", "GitHub", "AWS", "OIDC"]
published: false
---

## 概要

GitHub Actions のワークフローで AWS リソースにアクセスする際、よくある流れは以下の通りです。

1. IAM ユーザーを発行
2. 1 のアクセスキー id とシークレットアクセスキーを GitHub にシークレットとして登録

みなさんも、リポジトリにこんな設定をした記憶があるかと思います。

![](/images/github-actions-oidc-aws/github-secrets-example-1.png)

当然これでも OK なのですが、OIDC を用いることで、クレデンシャルの発行・GitHub への登録を行わずに認証を行うことができると知りました。

こちらの方がセキュアに AWS にアクセスできますので、今後の備忘録として共有していきます。

## 対象読者

本記事では、扱う各技術・サービスについて詳細に説明をしません。
以下について、「なんとなく知っている」「触ったことがある」方を前提としております。

- AWS
  - IAM
  - S3
- Open ID Connect (OIDC)
- GitHub Actions

とはいえ、よくわからなくても見れば使えるように書いていきますので、駆け出しエンジニアの方などもぜひご参考ください！

## デモ内容

本記事では、GitHub Actions を用いて、
`リポジトリにある s3/index.html ファイルを、AWS S3 バケットにアップロードする`
というデモを行います。

## 手順

GitHub の公式ドキュメントに沿って進めました。
https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services

本記事の操作手順は以下の通りです。

1. [AWS] ID プロバイダ追加
2. [AWS] IAM ロール作成
3. [AWS] S3 バケット作成
4. [AWS] IAM ポリシー作成 → IAM ロールにアタッチ
5. [GitHub Actions] ワークフロー作成

また、記事内の[こちらのページ](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)で、AWS への認証フローや、セキュアである理由など、勉強になることが書かれていますので、合わせて見てみると良さそうでした。

## 1. [AWS] ID プロバイダ追加

まずは、GitHub OIDC プロバイダを IAM に追加します。
ドキュメントでいうと[この部分です](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#adding-the-identity-provider-to-aws)。

- プロバイダのタイプ: OpenID Connect
- プロバイダの URL: `https://token.actions.githubusercontent.com`
- 対象者: `sts.amazonaws.com` (※ GitHub Actions 公式アクションを使うため)

プロバイダ URL を入力した後に、「サムプリントを取得」を押します。
（下記画像は「サムプリントを取得」押下後です）

![](/images/github-actions-oidc-aws/aws-iam-id-provider.png)

## 2. [AWS] IAM ロール作成

続いて、AWS で IAM ロールを作成します。
ドキュメントでいうと[この部分です](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#configuring-the-role-and-trust-policy)。

今回は、AWS コンソールでポチポチ作っていきました。
信頼ポリシーの書き方がドキュメントに記載されていましたので、そちらを参考にしています。

条件は以下の通りです。

- ロール名: GitHubOIDCRoleForExample
- 対象: [このリポジトリ](https://github.com/keisukesaito7/github-actions-oidc-test)の main ブランチ

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::{自分の AWS アカウント id}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEqual": {
          // 例) repo:keisukesaito7/github-actions-oidc-test
          // 上記リポジトリの main ブランチが対象となる
          "token.actions.githubusercontent.com:sub": "repo:{GitHub ユーザー名}/{GitHub リポジトリ名}:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

ここでロールを引き受けられる条件を定義できているので、セキュアとなります。

:::details 条件を満たさない場合の workflow のテスト

異なるリポジトリ上で AWS 認証を実行したところ、以下の画像の通りエラーとなりました。
リポジトリ名: `keisukesaito7/github-actions-oidc-test-dummy`

![](/images/github-actions-oidc-aws/github-actions-workflow-on-dummy-repo.png)
:::

## 3. [AWS] S3 バケット作成

### S3 バケット作成

アップロード先の S3 バケットを作成します。

- バケット名: `my-oidc-example`（※ バケット名はグローバルに一意）
- 設定: デフォルト

## 4. [AWS] IAM ポリシー作成 → IAM ロールにアタッチ

### IAM ポリシー作成

2 で作成した IAM ロールにアタッチする IAM ポリシーを作成します。

- ポリシー名: `S3PutObjectForOIDCGitHubActionsExamplePolicy`
- 権限: S3 バケット `my-oidc-example` に `index.html` をアップロードできる

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": [
        "arn:aws:s3:::my-oidc-example/index.html",
        "arn:aws:s3:::my-oidc-example"
      ]
    }
  ]
}
```

### IAM ロールに IAM ポリシーをアタッチ

作成したポリシーを、2 で作成したロールにアタッチします。
これで、Assume Role したときに、対象の S3 バケットにアップロードできるようになりました。

![](/images/github-actions-oidc-aws/iam-policy-attached.png)

これで、AWS 側の準備は完了です！

## 5. [GitHub Actions] workflow 作成

本題です。
GitHub Actions を使って、S3 に `index.html` をアップロードします。

ドキュメントでいうと[この部分です](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#updating-your-github-actions-workflow)。

### workflow の設定

リポジトリ上の `s3/index.html` を S3 バケット `my-oidc-example` にアップロードします。
（※ 一部、公式と異なっています）

```yaml:./github/workflows/deploy.yml
name: AWS example workflow

on:
  push

env:
  BUCKET_NAME : "my-oidc-example"
  AWS_REGION : "ap-northeast-1"

# ワークフローレベルで OIDC トークンを取得
permissions:
  id-token: write
  contents: read

jobs:
  S3PackageUpload:
    runs-on: ubuntu-latest
    steps:
      - name: Git clone the repository
        uses: actions/checkout@v3
      # AWS 認証. 事前に secrets を登録しておくこと
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ secrets.ROLE_NAME }}
          aws-region: ${{ env.AWS_REGION }}
      # S3 にアップロード
      - name:  Copy index.html to s3
        run: |
          aws s3 cp ./s3/index.html s3://${{ env.BUCKET_NAME }}/
```

### 結果

workflow が無事に実行されました！🎉

![](/images/github-actions-oidc-aws/github-actions-success.png)

S3 にも無事にアップロードされていました！🪣

![](/images/github-actions-oidc-aws/aws-s3-index-html-uploaded.png)

## おわりに

OIDC を意識して使ったのは初めてでしたが、ドキュメントが充実しておりかなりスムーズに構築できました。
特に、「これってみんなこのロールが使えちゃう心配はないの？」という初心者ライクな疑問なども、ドキュメント内で解消されている点が、とても素晴らしいと思いました。

今後も使っていきたいです！
（リソースを Terraform 等 IaC で構築できるように勉強していきたい！）
