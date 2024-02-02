---
title: "Azure Resource Graph を用いた App Service 証明書の管理"
author_name: "Takeharu Oshida"
tags:
    - App Service 証明書
    - Azure Resource Graph
    - Function App
---

# 2023/11/29 追記

2023年11月時点でプレビューではありますが、Azure Monitor から Azure Resource Graph に対してクエリを実行、アラートルールすることが可能となっています。
下記記事に本記事の続編を記載しています。

[Azure Resource Graph を用いた App Service 証明書の管理 その2](https://azure.github.io/jpazpaas/2023/11/29/check-app-service-certificate-expiration-with-azure-resource-graph-part2.html)

---

# はじめに


お世話になっております。App Service サポート担当の押田です。

App Service 証明書の有効期間は既定では 1 年です。 有効期限が近づいたら、自動または手動により App Service 証明書を 1 年単位で更新できます。
手動更新に設定している場合や、自動更新に設定している場合においても、ドメイン検証のために App Service 証明書の有効期間を把握しておくことが重要となります。
当ブログや、弊社エンジニアによる Qiita 記事でもご紹介させていただいております。

- [App Service 証明書の自動更新に伴うドメイン検証 \(有効期間 395 日\) 作業の必要について](https://azure.github.io/jpazpaas/2022/08/26/2022-08-25-App-Service-Certificate-Renewal.html)
- [App Service 証明書の有効期限を Azure Functions を使用して監視する\(2022年12月版\)](https://qiita.com/shogo-ohe/items/fd1ae6f66644f246edfd)
- [App Service 証明書の有効期限を Azure Functions を使用して監視する](https://qiita.com/shogo-ohe/items/b25e3a322b6f32ce1530)

本記事では、Azure Resource Graph を用いて、サブスクリプションに存在する購入済み App Service 証明書の状態を確認する方法について紹介いたします。

Azure Resource Graph については、

- [公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/governance/resource-graph/overview) 
- 弊社エンジニアによる Qiita 記事 [Azure Resource Graph \(ARG\) に馴染んでみる](https://qiita.com/task_woof/items/f3b513e745a052bccce0)

をご参考くださいませ。

## Azure Resource Graph を用いて App Service 証明書リソースを取得する

Azure Resource Graph のクエリを簡単に実行するには、ポータルから [Azure Resource Graph エクスプローラー](https://ms.portal.azure.com/#view/HubsExtension/ArgQueryBlade) にアクセスします。

下記記事もご参照くださいませ。

 - [クイック スタート:初めてのポータル クエリ \- Azure Resource Graph \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/resource-graph/first-query-portal)

App Service 証明書は `resource` テーブルにおいて、`type == "microsoft.certificateregistration/certificateorders"` で絞り込むことができ、詳細な情報は `properties` に json フォーマットで含まれております。
下記例では`subscriptionId` と `type` で絞り込み、見やすさのために `properties` の一部項目と、`propeties` 全体を出力したものとなります。

```kql
resources
| where type == "microsoft.certificateregistration/certificateorders"
| where subscriptionId == "<検索対象のサブスクリプションID>"
| project id, resourceGroup, name, properties["distinguishedName"] ,properties["provisioningState"], properties["expirationTime"], properties["autoRenew"], properties
```

![image-8a6e2fc5-7a52-4190-b04b-9ceea0aef08e.png]({{site.baseurl}}/media/2023/01/image-8a6e2fc5-7a52-4190-b04b-9ceea0aef08e.png)

`properties["expirationTime"]`から、有効期限を取得することができます。

例えば以下のように、現在有効で365日以内に期限が訪れる証明書に絞り込むことが可能です。

```kql
resources
| where type == "microsoft.certificateregistration/certificateorders"
| where subscriptionId == "<検索対象のサブスクリプションID>"
| where properties["expirationTime"] >= now() and properties["expirationTime"] < datetime_add("day", 365, now())
| project id, resourceGroup, name, expirationTime=tostring(properties["expirationTime"]), distinguishedName=properties["distinguishedName"], provisioningState=properties["provisioningState"], autoRenew=properties["autoRenew"]
| order by expirationTime asc
```

# App Service 証明書の有効期限を監視する

残念ながら Azure Resource Graph そのものには定期実行や、結果からのログアラートのような機能が提供されていないため、[Azure Workbooks](https://learn.microsoft.com/ja-jp/azure/azure-monitor/visualize/workbooks-data-sources) のデータソースとして Azure Resource Graph を利用してダッシュボードに表示するか、各言語の SDK を用いてご自身で用意したクライアントから実行する必要があります。


## 定期実行する方法

定期実行には例えば Azure Functions のタイマートリガー、App Service の WebJob、Logic Apps、任意のサーバーの crond などが選択肢として考えられます。後述の実装例では、Azure Functions を使った例を紹介いたします。

## 検索結果をもとにアラートを上げる(外部へ通知する)方法

Azure Functions からメール送信については SendGrid を利用した方法が [App Service 証明書の有効期限を Azure Functions を使用して監視する\(2022年12月版\)](https://qiita.com/shogo-ohe/items/fd1ae6f66644f246edfd) で紹介されております。後述の実装例では、Azure Monitor のログアラートを応用する方法をご紹介します。

## 実装例

大筋としては以下のような構成となります。

1. Azure Functions は TimerTrigger によって定期実行される
2. Azure Functions 内では、Managed ID による認証情報を用いて Azure Resource Graph から App Service 証明書を検索する
3. 指定した期日に有効期限が切れる証明書が見つかった場合に、ログ出力を行う
4. Azure Monitor ログアラート機能を用いて、特定のログが出力された場合にメールを送信する

この記事で紹介する内容のサンプルコードは以下で公開しています。

- [georgeOsdDev/appservice\-certrificate\-expiration\-checker](https://github.com/georgeOsdDev/appservice-certrificate-expiration-checker)


### Azure Functions を構成する

Azure Functions にはマネージド ID を用いて Azure Resource Graph クエリを実行するための権限を Azure Functions に付与します。 

#### マネージド ID のセットアップ

Azure Functions にシステムマネージド ID を設定し、該当の ID に対してサブスクリプションの[Reader](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#reader) 権限を付与します。

Azure Functions にマネージド ID を設定する方法についての詳細は下記記事をご参照ください。

- [マネージド ID \- Azure App Service \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/app-service/overview-managed-identity?toc=%2Fazure%2Fazure-functions%2Ftoc.json&tabs=portal%2Chttp)

Azure Resource Graph でのアクセス許可についての詳細は下記記事をご参照ください。

- [Azure Resource Graph の概要 \- Azure Resource Graph \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/governance/resource-graph/overview#permissions-in-azure-resource-graph)

#### タイマートリガー関数を作成する

マネージド ID を用いた認証を利用して、Azure Resource Graph にアクセスするため、2 つの npm モジュールをインストールします。

```
npm install --save @azure/arm-resourcegraph @azure/identity
```

以下のような関数を作成します。
```javascript
const { DefaultAzureCredential } = require("@azure/identity");
const { ResourceGraphClient } = require("@azure/arm-resourcegraph");

const credentials = new DefaultAzureCredential();
const client = new ResourceGraphClient(credentials);

const expireThreshold = process.env["expireThreshold"] || "90"; // days
const query = `Resources
| where subscriptionId == "${process.env["subscriptionId"]}"
| where type == "microsoft.certificateregistration/certificateorders"
| where properties["expirationTime"] >= now() and properties["expirationTime"] < datetime_add("day", ${expireThreshold}, now())
| project id, resourceGroup, name, expirationTime=tostring(properties["expirationTime"]), distinguishedName=properties["distinguishedName"], provisioningState=properties["provisioningState"], autoRenew=properties["autoRenew"]
| order by expirationTime asc`;

module.exports = async function (context, myTimer) {
    context.log("Start to check App Service Certificates Expirination");
    try {
        const result = await client.resources({query},{ resultFormat: "table" });
        if (result.totalRecords > 0) {
            context.log.warn(`Found App Service Certificates Expiring in the next ${expireThreshold} days, Count: ${result.totalRecords}`);
        } else {
            context.log(`No App Service Certificate Expiring in the next ${expireThreshold} days`);
        }
    } catch (error) {
        context.log.error("Failed to execute query", error);
    }
    context.log("Finish to check App Service Certificates Expirination");
};
```

[DefaultAzureCredential ](https://github.com/Azure/azure-sdk-for-js/blob/main/sdk/identity/identity/README.md#defaultazurecredential) を利用することで、マネージド ID が有効な環境ではマネージド ID が利用され、ローカル環境などでは Azure CLI によるログイン情報が利用されることになります。

上記のコード例では、`expireThreshold` 日以内に有効期限を迎える証明書のレコードが 1 件でも見つかった場合に、
`Found App Service Certificates Expiring in the next ${expireThreshold} days, Count: ${result.totalRecords}`　というログが　`traces` テーブルに出力されます。

上記の関数をTimerTrigger で 日次実行(`0 0 0 * * *`)されるようにします。 

Azure Functions タイマートリガーについての詳細は下記記事をご参照ください。

- [Azure Functions のタイマー トリガー \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=in-process&pivots=programming-language-javascript)

### ログアラートを構成する

ログアラートの設定としては、以下のように構成します。 `traces` テーブルに対するのクエリ結果が1件以上存在する場合にアラートを行うように構成します。

``` kql
traces
| where customDimensions.Category == "Function.DailyChecker.User"
| where customDimensions.LogLevel == "Warning"
| where message startswith "Found App Service Certificates Expiring in the next"
```

![image-ede24934-cfb2-4c9f-8708-a5528e527134.png]({{site.baseurl}}/media/2023/01/image-ede24934-cfb2-4c9f-8708-a5528e527134.png)

Azure Functions の監視の方法についての詳細は下記記事をご参照ください。

 - [Azure Functions の監視を構成する方法](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-monitoring?tabs=v2#configure-categories)

Azure Monitor ログ アラートについての詳細は下記記事をご参照ください。

 - [チュートリアル:Azure リソースのログ クエリ アラートを作成する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/alerts/tutorial-log-alert)


#### アラート例
上記のログアラートがトリガーされた場合、アラートに指定されたアクションが実行されます。例えばメール送信をアクションに指定していた場合、以下のようなメールが送信されることになります。

![image-cebd6d90-c5cc-440f-b3f6-8db231d2bb5e.png]({{site.baseurl}}/media/2023/01/image-cebd6d90-c5cc-440f-b3f6-8db231d2bb5e.png)

アクション グループについての詳細は下記記事をご参照ください。

- [Azure Portal でアクション グループを作成および管理する \- Azure Monitor \| Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-monitor/alerts/action-groups)

以上、Azure Resource Graph を用いて、サブスクリプションに存在する購入済み App Service 証明書の状態を確認する方法についてご紹介させていただきました。

2023 年 11 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>