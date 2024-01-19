---
title: "メール通信サービスを使用するための Azure ロールについて"
author_name: "Kohei Hosono"
tags:
    - Azure Communication Services
---
# はじめに

メール通信サービスを使用するにあたり、必要最低限の Azure ロールを知りたいといったご要望がある場合、 2024 年 1 月 17 日時点では、メール通信サービスならびに Communication Services 向けに作成された組み込みロールは提供されておらず、 [共同作成者](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#contributor) ロールなどでの代替を検討いただく必要がございます。

しかしながら [カスタムロール](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/custom-roles-portal) を使用して、より厳密に RBAC ロール (Azure リソースプロバイダー操作) を制御したい場合もあるかと存じます。本記事では、メール通信サービスを使用するための [Azure リソースプロバイダー操作](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/resource-provider-operations) について紹介いたします。

なおメール通信サービスに関する Azure ロールにつきましては、 「メール通信サービスリソースの操作」 と 「メール通信サービス (メール送信)」 の観点で検討いただければと存じます。これは、メール通信サービスリソースの操作は Azure リソースプロバイダー操作によって実行されますが、メール通信サービス (メール送信) は "アクセスキー" と "Microsoft Entra ID 認証" の 2 種類の方法で認証できるためです。

# メール通信サービス<u>リソース</u>を操作するための Azure リソースプロバイダー操作

Microsoft Learn ではメール通信サービスを使用してメールを送信するための [クイックスタート](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/create-email-communication-resource) を提供しております。本セクションではクイックスタートを実施するために必要なリソースプロバイダー操作を検証し、紹介いたします。

※ 2024 年 1 月 17 日時点での弊社環境での検証結果をもとにした内容となり、動作については変更される可能性がございますのでご留意ください。<br/>
※ Azure Portal を使用するシナリオについて紹介いたします。 Azure CLI や Azure REST API を使用する場合には必要な操作が異なる場合がございます。

## 1) Communication Services リソースならびにメール通信サービスを作成する

メール通信サービスを使用してメールを送信するには、 Communication Services リソースとメール通信サービスリソースを作成する必要があります。その場合、以下のリソースプロバイダー操作を許可するカスタムロールを割り当てることで Communication Services リソースとメール通信サービスリソースを作成できます。

```txt
Microsoft.Resources/subscriptions/resourceGroups/read
Microsoft.Resources/deployments/validate/action
Microsoft.Resources/deployments/write
Microsoft.Resources/deployments/read
Microsoft.Communication/CommunicationServices/write
Microsoft.Communication/CommunicationServices/read
Microsoft.Communication/emailServices/write
Microsoft.Communication/emailServices/read
```

参考 : [クイック スタート: Azure Communication Service でメール通信サービスのリソースを作成して管理する](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/create-email-communication-resource)

## 2) メール通信サービスにカスタムドメインを追加し、 Communication Services に接続する

メール通信サービスを使用してメールを送信するには、ドメインをプロビジョニングする必要があります。 Azure マネージドドメインを使用することもできますが、ここではカスタムドメインを使用することを想定します。

以下のリソースプロバイダー操作を許可するカスタムロールを割り当てることでメール通信サービスにカスタムドメインをプロビジョニングし、カスタムドメインを Communication Services に接続できます。

```txt
Microsoft.Resources/subscriptions/resourceGroups/read
Microsoft.Communication/CommunicationServices/write
Microsoft.Communication/CommunicationServices/read
Microsoft.Communication/emailServices/read
Microsoft.Communication/emailServices/domains/write
Microsoft.Communication/emailServices/domains/read
Microsoft.Communication/emailServices/domains/initiateVerification/action
```

参考 : [クイック スタート: メール通信サービスにカスタムの検証済みドメインを追加する方法](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/add-custom-verified-domains)<br/>
参考 : [クイック スタート: 検証済みメール ドメインを Azure Communication Service リソースに接続する方法](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/connect-email-communication-resource)

# メール通信サービスでメールを送信するための認証

メール通信サービスを使用してメールを送信するには、 ["アクセスキー" と "Microsoft Entra ID 認証" の 2 種類の方法で認証できます](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/authentication)。

## 方法 1) アクセスキーを使用してメールを送信する

例えば [Azure CLI を使用してメールを送信する](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email?tabs=windows%2Cconnection-string&pivots=platform-azcli#send-an-email-message) 場合、以下のコマンドでメールを送信できますが、この際にはアクセスキーを使用してメールを送信するため、 Azure CLI で Azure にログインしたユーザーに対して Azure ロールを割り当てる必要はありません。

```bash
az communication email send
	--connection-string "yourConnectionString"
	--sender "<donotreply@xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.azurecomm.net>"
	--to "<emailalias@emaildomain.com>"
	--subject "Welcome to Azure Communication Services Email" --text "This email message is sent from Azure Communication Services Email using Azure CLI."
```

各言語で用意された [Email SDK](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email?tabs=windows%2Cconnection-string&pivots=programming-language-csharp#send-an-email-message) についても、接続文字列を使用する場合には Azure ロールを意識いただく必要はありません。

### Azure Portal の [メールを試す] からメールを送信する

Azure Portal で Communication Services リソースを開き、 [[メールを試す] からメールを送信する](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/send-email) 場合、以下のリソースプロバイダー操作を許可するカスタムロールを割り当てることでメールを送信できます。

![image-202c4f8f-c365-4fae-b347-dfc5c58eea2e.png]({{site.baseurl}}/media/2024/01/image-202c4f8f-c365-4fae-b347-dfc5c58eea2e.png)

```txt
Microsoft.Resources/subscriptions/resourceGroups/read
Microsoft.Communication/CommunicationServices/read
Microsoft.Communication/CommunicationServices/listKeys/action
Microsoft.Communication/emailServices/domains/read
Microsoft.Communication/emailServices/domains/senderUsernames/read
```

Azure Portal の [メールを試す] からメールを送信する際にはアクセスキーを使用しており、 `Microsoft.Communication/CommunicationServices/listKeys/action` のリソースプロバイダー操作で Communication Services のアクセスキーを取得し、メールを送信しています。

## 方法 2) Email SDK でマネージド ID を使用してメールを送信する

各言語に用意された Email SDK では、マネージド ID を使用して Microsoft Entra ID 認証でメールを送信することも可能です。

しかしながら現時点では Email SDK を使用するマネージド ID に適した Azure ロールはなく、 [共同作成者](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#contributor) ロールのみがサポートされている状況となります。

なおマネージド ID に適したカスタムロールについては [GitHub Issue](https://github.com/MicrosoftDocs/azure-docs/issues/109461) でも挙げられておりますのでご参考いただけますと幸いです。

# まとめ

- メール通信サービスリソースならびに Communication Services リソースの操作については Azure リソースプロバイダー操作を許可する Azure ロールが必要です。
- メール通信サービスでメールを送信する際には、 "アクセスキー" と "Microsoft Entra ID 認証" のどちらを使用するか検討ください。
    - アクセスキーを使用する場合、メールの送信に Azure ロール (リソースプロバイダー操作) の割り当ては必要ありません。
    - Microsoft Entra ID 認証を使用する場合、現時点では共同作成者ロールのみがサポートされております。

# 参考

メール通信サービスならびに Communication Services に関する Azure リソースプロバイダー操作は、例えば Azure Portal の [サブスクリプション] > [アクセス制御 (IAM)] > [追加] > [カスタムロールの追加] > [アクセス許可] > [アクセス許可の追加] から "Azure Communication Services" に移動することでも確認できます。

![image-39b8dceb-47a1-42e5-8551-b600ee9fba64.png]({{site.baseurl}}/media/2024/01/image-39b8dceb-47a1-42e5-8551-b600ee9fba64.png)

<br>
<br>
---
<br>
<br>
2024 年 01 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>

