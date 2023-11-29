---
title: "Azure Resource Graph を用いた App Service 証明書の管理 その2"
author_name: "Takeharu Oshida"
tags:
    - App Service 証明書
    - Azure Resource Graph
---

# はじめに
お世話になっております。App Service サポート担当の押田です。

この記事は [Azure Resource Graph を用いた App Service 証明書の管理](https://jpazpaas.github.io/blog/2023/01/30/check-app-service-certificate-expiration-with-azure-resource-graph.html)の続編となります。
前回の記事投稿時点では、Azure Resource Graph (以降 ARG 表記)から直接アラートを実行することができなかったため、Azure Functions のタイマートリガーを利用することで App Service 証明書の有効期限とステータスを定期管理する方法を紹介しました。

2023年11月時点でプレビューではありますが、[Azure Monitor から Azure Data Explorer と Azure Resource Graph のデータに対するクエリ](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/azure-monitor-data-explorer-proxy#query-data-in-azure-resource-graph-preview)を実行することが可能となっており、Japan Azure Monitoring Support Blog の下記の記事でも紹介されています。

[リソース変更を検知するアラートを作成する](https://jpazmon-integ.github.io/blog/ame/HowToResourceChangeAlert/)

この記事では、上記手法に習い、Azure Monitor から ARG に対してクエリを実行、アラートルールを用いることで App Service 証明書の有効期限を監視、期限切れの防止を防ぐ方法を紹介します。

# Azure Monitor の Log Analytics ワークスペースから ARG に対してクエリを実行する

Log Analytics ワークスペースから ARG の各テーブルにクエリを実行する場合 `arg("").<ARGテーブル名>` とする必要があります。
それ以外は、基本的には前回の記事にて Azure Functions 関数内から実行していたクエリとほぼ同じです。

```kql
let varSubscriptionId = "<対象のサブスクリプションID>";
let expireThreshold = 60d; // days
arg("").Resources
| where type == "microsoft.certificateregistration/certificateorders"
| where subscriptionId == varSubscriptionId
| where properties["status"] == "Issued" and properties["expirationTime"] < now() + expireThreshold
| project 
    resourceGroup, 
    name, 
    distinguishedName = properties["distinguishedName"],
    expirationTime = properties["expirationTime"],
    lastCertificateIssuanceTime = properties["lastCertificateIssuanceTime"], 
    autoRenew = properties["autoRenew"],
    appServiceCertificateNotRenewableReasons = properties["appServiceCertificateNotRenewableReasons"]
```

残念ながら、ポータル上ではテーブル定義の先読みができないため警告で真っ赤になってしまいますが、クエリの実行は可能です。
また、下記画像では閾値を `90d` としていますが、証明書の更新は、自動更新の場合は 31 日前から、手動で更新する場合には、有効期限の 60 日前から更新が可能となります。詳細は[こちら](https://jpazpaas.github.io/blog/2022/08/26/2022-08-25-App-Service-Certificate-Renewal.html)。
必要に応じて適宜変更してください。


![image-28ce98f8-a35c-4b25-9244-42e74d4c5f1a.png]({{site.baseurl}}/media/2023/11/image-28ce98f8-a35c-4b25-9244-42e74d4c5f1a.png)


# アラートルールを作成する

上記のクエリに対して、通常の[ログアラート](https://learn.microsoft.com/ja-jp/azure/azure-monitor/alerts/alerts-types#log-alerts)の作成手順に従って、アラートルールを設定します。

例えば、下記のように、集計の粒度と評価の頻度を1日とすることで日時でのチェックが可能となります。

![image-a744a7a5-dd64-4897-b0c3-002df4ec9dda.png]({{site.baseurl}}/media/2023/11/image-a744a7a5-dd64-4897-b0c3-002df4ec9dda.png)

以上、
Azure Monitor から ARG に対してクエリを実行しアラートルールを用いて App Service 証明書の有効期限を監視、期限切れの防止を防ぐ方法をご紹介させていただきました。

関連記事

- [Azure Resource Graph を用いた App Service 証明書の管理](https://jpazpaas.github.io/blog/2023/01/30/check-app-service-certificate-expiration-with-azure-resource-graph.html)
- [App Service 証明書の自動更新に伴うドメイン検証 \(有効期間 395 日\) 作業の必要について](https://jpazpaas.github.io/blog/2022/08/26/2022-08-25-App-Service-Certificate-Renewal.html)
- [App Service 証明書の有効期限を Azure Functions を使用して監視する\(2022年12月版\)](https://qiita.com/shogo-ohe/items/fd1ae6f66644f246edfd)
- [App Service 証明書の有効期限を Azure Functions を使用して監視する](https://qiita.com/shogo-ohe/items/b25e3a322b6f32ce1530)
