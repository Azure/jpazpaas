---
title: "通信サービスにドメインを接続する方法"
author_name: "Shinji Kano"
tags:
    - Azure Communication Services
---

# 質問
以下のドキュメントに従って通信サービス（Azure Communication Services）に、ドメインの接続しようとしたところ、確認済み（検証済み）のドメイン（Verified Domain）が表示されません。

[クイック スタート: 検証済みメール ドメインを Azure Communication Service リソースに接続する方法](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/connect-email-communication-resource?pivots=azure-portal)

![image-114c4a23-6a9e-44dd-897f-2dd896c89f29.png]({{site.baseurl}}/media/2023/06/image-114c4a23-6a9e-44dd-897f-2dd896c89f29.png)


# 回答
通信サービス（Communication Services）と、メール通信サービス（Email Communication Services）のデータの保存場所が一致しているかを確認ください。<br>

## データの場所について
「通信サービス」のデータの場所は、チャット メッセージやリソース データが保存される場所です。<br>
[データの保存場所](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/privacy#data-residency)

「メール通信サービス」のデータの場所は、Emailメッセージ コンテンツやドメイン送信者のユーザー名が保存される場所です。<br>
[Email](https://learn.microsoft.com/ja-jp/azure/communication-services/concepts/privacy#email)


## 確認箇所
通信サービスにドメインの接続をするためには、以下のドキュメントに記載させていただいているように、通信サービスおよびメール通信サービスのデータの場所が一致している必要があります。

[クイック スタート: 検証済みメール ドメインを Azure Communication Service リソースに接続する方法](https://learn.microsoft.com/ja-jp/azure/communication-services/quickstarts/email/connect-email-communication-resource?pivots=azure-portal)<br>
> 注意<br>
> 地理的な場所が同じドメインの接続のみが許可されます。 リソースの作成時に選択された通信リソースとメール通信リソースのデータの場所が同じであることを確認してください。

- 「通信サービス」のデータの場所<br>
![image-12428ef0-4086-4241-b149-879fa51e6e09.png]({{site.baseurl}}/media/2023/06/image-12428ef0-4086-4241-b149-879fa51e6e09.png)

- 「メール通信サービス」のデータの場所<br>
![image-2c60350e-e662-4a5b-82b0-d28cb75c79e2.png]({{site.baseurl}}/media/2023/06/image-2c60350e-e662-4a5b-82b0-d28cb75c79e2.png)

## 留意点
「通信サービス」および「メール通信サービス」のデータの場所については、各リソースの作成時に設定いただく項目となっており、リソース作成後に変更いただくことはできません。そのため、もし「通信サービス」と「メール通信サービス」のデータの場所が一致していない場合には、データの場所を設定しなおすために新規でリソースを作成いただく必要があります。<br>

また、「通信サービス」では対応しているデータの場所であっても、「メール通信サービス」では対応していない場合があります。本記事を執筆している 2023/6/16 時点では、「通信サービス」は Japan に対応していますが、「メール通信サービス」は Japan には対応していません。そのため、メール通信サービスをご利用いただく場合には、通信サービスのデータの場所とあわせてご検討ください。