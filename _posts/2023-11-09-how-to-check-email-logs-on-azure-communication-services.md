---
title: "Azure Communication Services のメール機能でメール送信ログを確認する方法"
author_name: "yukurash"
tags:
    - Azure Communication Services
---

# 質問
Azure Communication Services のメール機能でメールを送信しましたが、メールが正常に送信できたか等のログはどのように確認することができますか？

# 回答
診断設定を用いることでメール送信のログを確認することができます。具体的には Azure ポータルから Azure Communication Services を開き、[ログ] ブレードから各テーブルのログを確認する方法と、[分析情報（プレビュー）] ブレードからグラフ等で視覚的に確認する方法の 2 種類があります。

## 前提条件
どちらの方法でメール送信のログを確認する場合にも、Azure Communication Services の診断設定機能を有効化する必要があります。
![acslog1-dbc82f53-55f9-4f90-8ed9-659133d1252a.png]({{site.baseurl}}/media/2023/11/acslog1-dbc82f53-55f9-4f90-8ed9-659133d1252a.png)

診断設定より下記の 3 つのカテゴリ、”Log Analytics ワークスペースへの送信” を選択し、保存することで、Azure Communication Services のメールログを確認するための診断設定が有効化されます。

| カテゴリ | 概要 | 詳細 URL |
|--|--|--|
| Email Service Send Mail Logs | メール送信のリクエストのログ | [ACSEmailSendMailOperational](https://learn.microsoft.com/ja-jp/azure/azure-monitor/reference/tables/acsemailsendmailoperational) |
| Email Service Delivery Status Update Logs | メール送信のリクエストのステータス等の詳細なログ | [ACSEmailStatusUpdateOperational](https://learn.microsoft.com/ja-jp/azure/azure-monitor/reference/tables/acsemailstatusupdateoperational) |
| Email Service User Engagement Logs | ユーザーエンゲージメントに関するログ | [ACSEmailUserEngagementOperational](https://learn.microsoft.com/ja-jp/azure/azure-monitor/reference/tables/acsemailuserengagementoperational) |

![acslog2-41be438b-029c-4f5b-a915-8e7748031c6a.png]({{site.baseurl}}/media/2023/11/acslog2-41be438b-029c-4f5b-a915-8e7748031c6a.png)

Log Analytics をご利用される場合、追加で費用が掛かる可能性もございますのでご注意ください。

[Azure Monitor の価格](https://azure.microsoft.com/ja-jp/pricing/details/monitor/)

## [ログ] ブレードから確認する方法
Azure ポータルの [ログ] ブレードより、診断設定で有効化した各テーブルの詳細をご確認いただけます。

例えば、Azure Communication Services から送信したメールの情報を確認したい場合、”ACSEmailSendMailOperational” テーブルを確認することで、送信したメールのサイズや宛先の人数、添付の数等を確認することが可能です。
![acslog9-a5138136-8e08-42e5-abbe-b1cd0c538e67.png]({{site.baseurl}}/media/2023/11/acslog9-a5138136-8e08-42e5-abbe-b1cd0c538e67.png)
クエリ例
```
ACSEmailSendMailOperational
| where TimeGenerated > ago(1d)
| project TimeGenerated, Size, ToRecipientsCount, CcRecipientsCount, BccRecipientsCount, UniqueRecipientsCount, AttachmentsCount, CorrelationId
| take 10
```

また、Azure Communication Services からのメール送信が成功したか失敗したか、失敗した場合なぜ失敗したかについては、"ACSEmailStatusUpdateOperational" テーブルを確認することで、送信したメールのステータス（成功や失敗等）やエラーメッセージ等を確認することが可能です。
![acslog10-9b7b7444-9674-40e3-9888-24c18899036e.png]({{site.baseurl}}/media/2023/11/acslog10-9b7b7444-9674-40e3-9888-24c18899036e.png)
クエリ例
```
ACSEmailStatusUpdateOperational
| where TimeGenerated > ago(1d)
| where OperationName =~ "DeliveryStatusUpdate"
| where RecipientId != ""
| project TimeGenerated, RecipientId, DeliveryStatus, SmtpStatusCode, EnhancedSmtpStatusCode, FailureReason, FailureMessage, CorrelationId
| take 10
```

補足ではございますが、”DeliveryStatus" カラムに記載される Status につきましては、下記のドキュメントに記載されておりますのでご参照ください。

![acslog7-2892e925-19f3-4c31-a7bd-d86f1655216b.png]({{site.baseurl}}/media/2023/11/acslog7-2892e925-19f3-4c31-a7bd-d86f1655216b.png)

[Azure Communication Services - Email イベント](https://learn.microsoft.com/ja-jp/azure/event-grid/communication-services-email-events#microsoftcommunicationemaildeliveryreportreceived-event)
## [分析情報（プレビュー）] ブレードから確認する方法
Azure ポータルの [ログ] ブレードより、表やグラフで Azure Communication Services から送信したメールの状態等がご確認いただけます。
例えば、下記のようにメール送信の結果やメールサイズ、メール受信者数等を確認することが可能です。
![acslog6-1b4b746b-dd2f-4216-9ea6-3f036abf8fa6.png]({{site.baseurl}}/media/2023/11/acslog6-1b4b746b-dd2f-4216-9ea6-3f036abf8fa6.png)

なお、こちらはプレビュー機能であるため、予めご留意ください。

# 参考ドキュメント
- [電子メール ログのAzure Communication Services](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/analytics/logs/email-logs)
- [Email Insights](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/analytics/insights/email-insights)

<br>
<br>

---

<br>
<br>

2023 年 11 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>