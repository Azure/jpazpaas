---
title: "Azure Communication Services メール機能でよく頂くご質問"
author_name: "Yusuke Tobo"
tags:
    - ACS
    - SMTP
    - Email
---
こんにちは。Azure Communication Services (以下 ACS) のサポートチームです。

[Exchange Online SMTP 基本認証廃止 ](https://techcommunity.microsoft.com/blog/exchange/exchange-online-to-retire-basic-auth-for-client-submission-smtp-auth/4114750) 等をきっかけに、ACS メール機能のご利用をご検討くださり、ありがとうございます。本記事では、ACS のメール機能をご利用いただく方に向けて、よく頂くご質問をおまとめしております。本情報がご参考になりましたら幸いでございます。

# SMTP の認証について
## ACS メール機能では SMTP AUTH 基本認証はサポートされていますか？
SMTP AUTH 基本認証はサポートされております。

基本認証を利用してメール送信する際のセットアップは[簡易メール転送プロトコル (SMTP) 認証の資格情報を作成する
](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email-smtp/smtp-authentication?tabs=built-in-role
) をご確認ください。

## 先端認証は利用できますか？
はい、ACS メール機能では先端認証もサポートされています。

先端認証として、XOauth2 を利用したメール送信の[チュートリアル](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email-smtp/send-email-smtp-oauth)をご覧ください。

# 送信元メールアドレスについて
## 送信元メールアドレスとして利用できるドメインはどのような種類がありますか？
ACS からメールを送信する際に利用できるドメイン種別にはマネージドドメインとカスタムドメインがあります。

マネージドドメインは、`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.azurecomm.net` 形式でお客様に払い出される弊社管理のドメインとなります。[マネージドドメイン追加のチュートリアル](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-azure-managed-domains?pivots=platform-azp) にありますように、Azure ポータルから払い出すことができます。

マネージドドメインには、`donotreply@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.azurecomm.net` から送信元メールアドレスを変更できない、１分間や１時間にメール送信できる数が極めて低く、引き上げることができないといった[制限](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-azure-managed-domains?pivots=platform-azp#azure-managed-domains-compared-to-custom-domains)があります。そのため、本番環境ではカスタムドメインが一般に利用されると考えております。

カスタムドメインは、お客様管理の任意のドメインです。カスタムドメインを ACS に追加し、同ドメインを ACS から送信するメールの送信元メールアドレスのドメインにすることができます。

カスタムドメインを ACS に追加するためには、ドメインの所有権を検証するための TXT レコード、SFP TXT レコード、DKIM CNAME レコードをそれぞれ同ドメインのレジストラーに設定していただく必要がございます。　手順は、[カスタムドメインを ACS へ追加する手順](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-custom-verified-domains?pivots=platform-azp)をご確認ください。

## onmicrosoft\.com は ACS の送信元ドメインとして使えますか？
onmicrosoft\.com は ACS としてはカスタムドメインの扱いでございます。
onmicrosoft\.com では、お客様にて DKIM CNAME レコードを追加できず、カスタムドメインとして ACS でご利用いただけません。

## 送信元メールアドレスの \@ より前のユーザー名を donotreply 以外にできますか？
マネージドドメインではできませんが、カスタムドメインでは donotreply 以外のユーザー名をご利用いただけます。

手順・[複数の送信者ユーザー名を作成する](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-multiple-senders?pivots=platform-azp#create-multiple-sender-usernames) を参考に、「Email Communication Services Domain」・「Mailfrom addresses」の「Add」ボタンから新しいユーザー名の送信元メールアドレスを作成してください。

なお、「Add」ボタンは既定の送信数の制限ではグレーアウトされて操作できないようになっております。
後述の「送信数の制限」クォータ引き上げのサポートリクエストをご発行ください。

# 送信数の制限について
## 送信数の制限はありますか？
[サービス制限](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#email) に記載しております通り、１分、１時間当たりに送信できるメール数に制限（クォータ）がございます。

## 送信数の制限は引き上げられますか？
マネージドドメインは引き上げられませんが、カスタムドメインは引き上げられます。

## 送信数の制限を引き上げるにはどうすればいいですか。
クォータ引き上げのためのサポートリクエストを ACS 宛にご発行ください。サポートリクエストには、[審査に必要な情報](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/email/email-quota-increase#5-request-an-email-quota-increase) を日本語で問題ございませんので記載ください。
審査の上、送信数の制限が引き上げられます。審査には３営業日ほどかかる場合もございますことご留意ください。

## 大量のメールを送りたいのですが、どの程度の送信数を ACS はサポートできますか？
以下[記載](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#email)ございます通り、1 時間あたり最大 100 万から 200 万メッセージのメール送信がサポートできます。

ただし、IP ウォームアップのために徐々に送信数の制限を引き上げていただく必要がございますので、ACS にてカスタムドメインからメール送信できるようになりましたら、お早めにクォータ引き上げのサポートリクエストをご発行いただくことをお勧めいたします。

```
Azure Communication Services のメール サービスは、高スループットをサポートするように設計されています。
ただし、このサービスには、お客様がオンボードを円滑に行い、新しいメール サービスに切り替えるときに発生する可能性のあるいくつかの問題を回避できるように、初期レート制限が設けられています。

メールの配信状態を注意深く監視しながら、2 週間から 4 週間かけて、Azure Communication Services Email を使用するメールの量を少しずつ増やすことをお勧めします。
このように段階的に増やすと、サード パーティのメール サービス プロバイダーはドメインのメール トラフィック用の IP の変更に適応できます。 
少しずつ変更すると、送信者評価を保護し、メール配信の信頼性を維持する時間が得られます。

Azure Communication Services のメール サービスでは、1 時間あたり最大 100 万から 200 万メッセージの大量のメッセージがサポートされます。
```

## 送信数のクォータ引き上げ審査でメール送信失敗率は加味されますか？
以下[記載](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/service-limits#failure-rate-requirements)の通り加味されます。

```
高いメール クォータを有効にするには、メールの失敗率が 1% 未満である必要があります。
失敗率が高い場合は、クォータの引き上げを要求する前に問題を解決する必要があります。
お客様は、失敗率を積極的に監視する必要があります。
クォータの引き上げ後に失敗率が上がった場合、Azure Communication Services はお客様に連絡して、直ちに対処することを求め、解決のタイムラインを確認します。
極端なケースでは、指定されたタイムライン内で失敗率が管理されない場合、Azure Communication Services は、問題が解決されるまでサービスを減らしたり中断したりする可能性があります。
```

## 失敗率の監視はどこからできますか？
[失敗率](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/analytics/insights/email-insights#failure-rate) に記載がございますように、Azure ポータルの「Communication Service」リソース、「分析情報」ブレード、「メールの概要」タブから何通のうち何件のメール配信が成功・失敗したのか確認いただけます。

お客様の監視要件に応じてテスト・カスタマイズが必要ではありますが、KQL のサンプルは以下になります。失敗率監視のご検討の参考になりましたら幸いです。

```KQL
ACSEmailStatusUpdateOperational 
| where OperationName == "DeliveryStatusUpdate" and isnotempty(DeliveryStatus) and DeliveryStatus != "OutForDelivery"
| summarize total=count(), failures=countif(DeliveryStatus != "Delivered" and DeliveryStatus != "Suppressed")
| extend rate=round(((failures * 100.0)/total),2)
| extend label="Emails failed", TotalMessage=strcat('out of ', total, ', ', rate,'% total Emails sent')
```


# ACS でのメール受信について
## ACS でメールを受け取ることはできますか？
ACS にはメール受信機能が現状ございません。バウンスメールの受信等のために、お客様でメール受信サーバーを別途ご用意ください。

<br>
<br>

---

<br>
<br>

2025 年 8 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>