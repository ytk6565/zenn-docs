---
title: "AWSの請求レポートを毎日Slackに通知"
emoji: "🧾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  ["aws", "lambda", "serverless", "slack"]
published: true
---

# はじめに

個人で AWS を利用していて

- 毎日どれくらいのコストがかかっているのか把握したい
- コストが多くかかっているリソースに早く気付きたい
- コストに意識を向けたい

といったモチベーションが生まれたため AWS の請求レポートを毎日 Slack に通知する Lambda 関数を開発しました。

GiHub に [リポジトリ](https://github.com/ytk6565/aws-billing-daily-report) を公開しておりますので参考にしていただけましたら幸いです。

## 前提

- Slack のワークスペース、レポート用のチャンネルは用意済みの想定で書いております。
- Serverless Framework の CLIをインストール済みの想定で書いております。
- Node.js のバージョンは `12.x` を利用しております。

# Slack Webhook URL の生成

Slack へのメッセージ送信に Incoming Webhook を利用します。
はじめに Webhook の URL を生成します。

`https://${workspaceId}.slack.com/apps/A0F7XDUAZ--incoming-webhook-` にアクセスし、「Slackに追加」をクリックする。

![WebHook-001](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/aws-billing-daily-report/webhook-001.png =500x)

「チャンネルを選択」をクリックするとプルダウンが表示されるので選択する。
「Incoming Webhook インテグレーションの追加」がアクティブになるのでクリックする。

![WebHook-002](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/aws-billing-daily-report/webhook-002.png =500x)
![WebHook-003](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/aws-billing-daily-report/webhook-003.png =500x)

Webhook URL が生成されるのでコピーしておく。

![WebHook-004](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/aws-billing-daily-report/webhook-004.png =500x)

# Lambda 関数を実装

詳細なコードに関しては GitHubの [リポジトリ](https://github.com/ytk6565/aws-billing-daily-report) を参照していただき要点のみまとめさせていただきます。

## Webhook URL を設定する。

`$ cp ./env.example.yml ./env.yml` で `.env.yml` ファイルを作成します。
先ほどコピーした Webhook URL を `.env.yml` の `SLACK_WEBHOOK_URL` に設定します。

## 必ず us-east-1 リージョンを利用する。

日本で基本的に利用するリージョンは `ap-northeast-1` ですが、
今回コストの使用状況の確認に利用する AWS Cost Explorer が `us-east-1` リージョンのみの対応のため、必ず `us-east-1` リージョンを利用してください。

## 日付は `XXXX-XX-XX` の形式にする。

`2021/01/01` や `2021-1-1` などではエラーになります。

# Slack へのメッセージの確認

`$ yarn build` で Lambda 関数をビルドします。

デプロイする前に `$ sls invoke local -f create` でローカルのコードをテストします。
Slack にメッセージが届けば、 `$ sls deploy` でデプロイを実行します。

![Slackへのメッセージサンプル](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/aws-billing-daily-report/message.png =500x)
*Slackへのメッセージサンプル*

# おわりに

メッセージをカスタマイズする場合は、 [AWS Cost Explorer のドキュメント](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CostExplorer.html) を参照してください。

最後までお読みいただきありがとうございます。
記事に対するリアクションや情報の誤り、誤字脱字などございましたらコメントや[記事を管理しているリポジトリ](https://github.com/ytk6565/zenn-docs)までお願いいたします。

# 参考にさせていただいたページ

- https://qiita.com/narikei/items/3f772c12591afdc5262d
- https://slack.com/intl/ja-jp/help/articles/115005265063-Slack-%E3%81%A7%E3%81%AE-Incoming-Webhook-%E3%81%AE%E5%88%A9%E7%94%A8
- https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CostExplorer.html
- https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CostExplorer.html#getCostAndUsage-property
