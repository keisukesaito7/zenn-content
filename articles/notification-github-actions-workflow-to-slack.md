---
title: "GitHub Actions の実行結果を Slack に通知する"
emoji: "🔔"
type: "tech"
topics: ["GitHubActions", "GitHub", "Slack"]
published: false
---

## 概要

GitHub Actions のワークフローの結果を、以下のアクションで Slack に通知している方は多いと思います。

https://github.com/slackapi/slack-github-action#technique-3-slack-incoming-webhook

1. Slack App を作成して、Incoming Webhook URL を発行
2. GitHub Actions に Slack 通知用の job を追加
3. 1 で発行した URL をリポジトリの secrets に登録

こちらが、GitHub の Slack App を用いて実現できることを知ったので、備忘録として記録します。
https://github.com/integrations/slack#actions-workflow-notifications

## この記事を読むメリット

GitHub Actions から Slack 通知をするのにかかる手間を減らせる。

## 手順

1. Slack に GitHub App を追加
2. subscribe の設定を行う

これだけでした。
1 については、Slack で GitHub の App を選択していけばよしなにできるので、省略します。

## ワークフロー

今回は、GitHub が用意している「Simple Workflow」を使います。

![](/images/notification-github-actions-workflow-to-slack/template-simple-workflow.png)

中身はこんな感じです。
main ブランチに変更がある、もしくは main ブランチに対して PR が出たら実行されるワークフローです。

:::details .github/workflows/hello_world.yml
```yml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v3

      # Runs a single command using the runners shell
      - name: Run a one-line script
        run: echo Hello, world!

      # Runs a set of commands using the runners shell
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```
:::

## Slack 側の設定

### 通知設定　

通知したいチャンネル内で、以下のコマンドを実行することで、ワークフローを subscribe できます。

```
/github subscribe ユーザー名/リポジトリ名
workflows:{event:"pull_request","push" branch:"main"}
```
例に上げた書き方だと、以下の条件が通知の対象となります。

- main ブランチに向けた PR で実行されるワークフロー
- main ブランチにコミットが push されたときに実行されるワークフロー

### 通知を試す

main ブランチに向けた PR を作成し、ワークフローを実行します。
すると、以下の通りチャンネルに通知が来ました！🎉

![](/images/notification-github-actions-workflow-to-slack/notification-test-pr.png)

このブランチを main にマージします。
こちらも同じく通知が来ました！

![](/images/notification-github-actions-workflow-to-slack/notification-test-merge-to-main.png)

## フィルタリング

https://github.com/integrations/slack#workflow-notification-filters

こちらのリンクの通り、通知するワークフローをフィルタリングできます。

- name : ワークフロー名
- event : ワークフローのトリガーとなるイベント
- actor : ワークフローを起動した人、または実行責任者
- branch : ワークフローが実行されているブランチ。pull_request イベントが含まれている場合のみ、このブランチはプルリクエストが作成されたターゲットブランチを指す

### ブランチのフィルタリング

今回は main ブランチに向けた PR を subscribe してるので、例えば develop ブランチに向けた PR で実行されたワークフローは通知対象外です。

それを確かめてみます。
まずワークフローが、develop に向けた PR でも実行されるように変更します。

```diff yml:.github/workflows/hello_world.yml
@@ -8,7 +8,7 @@
pull_request:
-  branches: [ "main" ]
+  branches: [ "main", "develop" ] # develop に向けた PR でも実行されるように
```

commit して push して、develop に向けて PR を作成します。
設定通りワークフローが実行されました。

![](/images/notification-github-actions-workflow-to-slack/create-pr-to-develop.png)

一方、Slack への通知は行われませんでした。

![](/images/notification-github-actions-workflow-to-slack/notification-test-pr-to-develop.png)

もしこの PR のワークフローも通知したい場合は、以下のように Slack でコマンドを打ちます。

```
/github subscribe ユーザー名/リポジトリ名
workflows:{event:"pull_request","push" branch:"main","develop"}
```

これでワークフローを Rerun してみると、Slack に通知が来るようになりました！

![](/images/notification-github-actions-workflow-to-slack/notification-test-pr-to-develop-2.png)

## おわりに

手間も情報漏えいリスクも軽減されたので、これからはこちらのやり方で Slack 連携していきたいと思いました。
