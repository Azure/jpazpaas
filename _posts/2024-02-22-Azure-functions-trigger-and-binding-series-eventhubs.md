---
title: "Azure Functions トリガー・バインディング シリーズ - Event Hubs トリガー"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本動作
[Event Hubs トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs?tabs=isolated-process%2Cextensionv5&pivots=programming-language-csharp)は、指定された Event Hubs からイベントを取得し実行される関数です。

Event Hubs トリガーは、BLOB コンテナに配置された Event Hubs のパーティションごとに作成されるファイルのロックを取得したインスタンスでそのパーティションのイベントの処理が行われます。また、Event Hubs のどこまで処理したかの情報も BLOB コンテナ上に保持してできるだけ処理漏れが発生しないように動作します。動作イメージは[こちら](https://learn.microsoft.com/ja-jp/azure/architecture/serverless/event-hubs-functions/event-hubs-functions#consuming-events-with-azure-functions)にも紹介がございます。

![image-0627fdfb-00d4-470e-963c-fc45e11ecc43.png]({{site.baseurl}}/media/2024/02/image-0627fdfb-00d4-470e-963c-fc45e11ecc43.png)


本動作に利用されるファイルは BLOB コンテナに配置されたファイルとなります。Azure Functions リソースのアプリケーション設定 `AzureWebJobsStorage` に指定されたストレージ アカウントの BLOB コンテナー `azure-webjobs-eventhub` が利用されます。`azure-webjobs-eventhub/<Event Hubs 名前空間名>.servicebus.windows.net/<Event Hubs 名>/<コンシューマー グループ名>` に `checkpoint` と `ownership` が作成されます。それぞれ Event Hubs のパーティションの数分のファイルが作成されます。
`checkpoint` は、各ファイルのメタデータに各パーティションごとに処理済みのイベントの位置を保持しており、`ownership` は、各ファイルのメタデータに各 Event Hubs のクライアントを表す識別子を保持し、処理するインスタンスを識別します。`ownership` に示す識別子を持った Event Hubs クライアントが `checkpoint` に沿って Event Hubs のイベントを処理します。

**Event Hubs トリガー動作用のファイル群**

![image-69f0da17-902d-4935-b4d0-a10d1deec26d.png]({{site.baseurl}}/media/2024/02/image-69f0da17-902d-4935-b4d0-a10d1deec26d.png)

**checkpoint 例**

![image-109ed989-693d-40cb-8ef1-782c08af1602.png]({{site.baseurl}}/media/2024/02/image-109ed989-693d-40cb-8ef1-782c08af1602.png)

**ownership 例**

![image-88a9837a-c1c8-49dd-8fc9-b398fd6b5983.png]({{site.baseurl}}/media/2024/02/image-88a9837a-c1c8-49dd-8fc9-b398fd6b5983.png)


この動作の仕組みは Azure Functions に限ったことではありません。Event Hubs SDK を利用いただき構成する際には同様の仕組みで Event Hubs の処理が制御されます。Event Hubs のドキュメントですが、[こちら](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-processor-balance-partition-load)の「パーティションの所有権」及び「Checkpoint」の章をご確認ください。


# 作成手順
Event Hubs トリガーの作成方法は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-your-first-function-visual-studio#create-a-function-app-project)で紹介されているテンプレートをご利用ください。チュートリアルでは HTTP トリガーを利用しておりますが、同画面にて Event Hubs トリガーを選択いただくことができます。

また、Event Hubs 名を、アプリケーション設定に外だしすることができます。C# の例ですが、EventHubTrigger の引数で Event Hubs 名を `%` 括りのアプリケーション設定名とします(上段)。ローカルの実行のため local.settings.json となりますが(下段)、eventhubsname の中身を宣言します。Azure 上の Azure Functions リソースの場合には同項目をアプリケーション設定に設定ください。

![image-38b55271-62d7-4a3b-9309-48e7552d39fb.png]({{site.baseurl}}/media/2024/02/image-38b55271-62d7-4a3b-9309-48e7552d39fb.png)

# 実行状況をログで確認する
Event Hubs トリガーの実行詳細をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。

**host.json**
```
"logging": {
  "logLevel": {
    "Azure.Messaging.EventHubs": "Debug"
  }
}
```

**Application Insights で利用するクエリ**
```
traces
| where timestamp > datetime("yyyy-mm-dd hh:mi")
| where customDimensions.Category startswith "Azure.Messaging.EventHubs" or customDimensions.Category startswith "Function.EventHubsTrigger" or operation_Name =~ "EventHubsTrigger"
| order by timestamp asc
```

上記の設定を行った際に記録されるログの例です。1-2 行目にて Event Hubs トリガーの ownership ファイルが識別子 `c8167cb8-9275-4548-af65-28b6448ed05f`(Event Hubs へ接続する[クライアントの識別子](https://learn.microsoft.com/ja-jp/azure/event-hubs/event-processor-balance-partition-load#partition-ownership)であり、ownership のメタデータに記載されています) で更新されていることが確認できます。
また、パーティション 3 について 10 個のイベントを取得し Event Hubs トリガーが実行されたことが確認できます。`Trigger Details:~` となっているログでは、Event Hubs の[メタ データ](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cfunctionsv2%2Cextensionv5&pivots=programming-language-javascript#event-metadata) が確認できパーティション、エンキュー時刻やイベント数などが記録されます。

**ログ出力例**

![image-e1afe51c-2fae-41d9-9e77-2b946c929613.png]({{site.baseurl}}/media/2024/02/image-e1afe51c-2fae-41d9-9e77-2b946c929613.png)

# 並列実行とチューニング

## 並列実行の考え方
Event Hubs トリガーでは、パーティションごとの分割による並列実行と 1 回のトリガー当たりに処理するイベント数の 2 点を考慮します。

パーティションごとの分割による並列実行では、Azure Functions のインスタンス数が Event Hubs のパーティション数と一致する場合に最大のリソース利用効率が期待できます。結果として性能改善につながる可能性があります。この Azure Functions のインスタンスと Event Hubs のパーティションを考慮したアプローチは、Event Hubs トリガーが動作するインスタンス固有の情報を共有していたり、CPU やメモリなどのリソースを多く処理する場合に有効です。

1 回のトリガー当たりに処理するイベント数は、1 つか 2 つ以上かを制御することができます。`host.json` の `maxEventBatchSize` と `functions.json` の `cardinality` (`C#` 以外の言語) で指定します。また、ひとつのインスタンスが処理する特定のパーティションは、そのパーティションから取得したイベントの処理がすべて終わるまでは次のイベントが取得されません。

**動作イメージ**

![image-1654f6de-59e8-4723-9254-b3b90134ae44.png]({{site.baseurl}}/media/2024/02/image-1654f6de-59e8-4723-9254-b3b90134ae44.png)

## 並列実行時のログとチューニングの例

通常は処理に 1 秒要するものの、同じインスタンス内で並列して Event Hubs トリガーが実行された場合にペナルティとして処理に 5 秒要するプログラムを考えます。

実際に 1 インスタンスで同プログラムを実行した結果が以下のログになります。`host.json` の [`maxEventBatchSize`](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs?tabs=isolated-process%2Cextensionv5&pivots=programming-language-python#hostjson-settings) を 1 と設定し、1 回のトリガーで 1 つのイベントを処理するように制御しており、100 回の実行(100 個のイベントの処理)が行われています。同じインスタンスが動作しており、`duration` からわかるようにたびたび 5 秒のペナルティが発生していることがわかります。

**requests ログ**

![image-f73a4f5e-01ee-4154-8900-9231c3b9aa3f.png]({{site.baseurl}}/media/2024/02/image-f73a4f5e-01ee-4154-8900-9231c3b9aa3f.png)

また、`Trigger Details:` からは始まるログを確認すると、処理するイベントが属していたパーティションと取得されたイベントの数が確認できます。5 秒のペナルティを受けたときには、直前のトリガーでパーティション 1 の処理をしており、その完了前にパーティション 5 の処理が並列で実行されていることが確認できます。

**traces ログ**

![image-5d115b4a-15d0-494d-9e2a-a46fb003e719.png]({{site.baseurl}}/media/2024/02/image-5d115b4a-15d0-494d-9e2a-a46fb003e719.png)

次に、パーティションの数に合致するようにインスタンスを 8 にスケールアウトし、実行した結果が以下のログになります。多くは別々のインスタンスで処理されており、同じインスタンスで処理されていても 1 秒で完了していることがわかります。

**requests ログ**

![image-685abfe6-f866-455b-8adf-8456c91fc6fe.png]({{site.baseurl}}/media/2024/02/image-685abfe6-f866-455b-8adf-8456c91fc6fe.png)

また、8 インスタンスで処理した際に、同一のインスタンスで処理された場合には、約 1 秒ごとに実行が行われています。これは前述の「ひとつのインスタンスが処理する特定のパーティションは、そのパーティションから取得したイベントの処理がすべて終わるまでは次のイベントが取得されません。」となるためです。
以下はパーティション 5 に対する処理の `traces` ログになります。パーティション 5 のイベントを 1 件処理したら次のイベントを取得し処理していることがわかります。

**traces ログ**

![image-38d40fcc-fd4d-4eed-91ee-2ab8b530fbe4.png]({{site.baseurl}}/media/2024/02/image-38d40fcc-fd4d-4eed-91ee-2ab8b530fbe4.png)


# よくお問い合わせいただく動作
弊サポートにてよくお問い合わせいただく事例について紹介します。

## Event Hubs のイベント送信から Event Hubs トリガーの実行が遅延する

Event Hubs トリガーは、ひとつのインスタンスが処理する特定のパーティションは、そのパーティションから取得したイベントの処理がすべて終わるまでは次のイベントが取得されない動作のため、次の実行が遅延することがあります。

実際の動作例を確認します。1 パーティションのみの Event Hubs で 1 イベント当たりの処理に 10 秒要するかつ 1 回のトリガーで 10 個のイベントを処理する場合を考えます。`traces` ログより、パーティション 0 のイベントの処理が開始しますが、そのトリガーの処理全体が完了した後に次の処理が実行される動作が確認できます。

**traces ログ**

![image-53c1198b-501c-415f-985b-565e1b90d2ad.png]({{site.baseurl}}/media/2024/02/image-53c1198b-501c-415f-985b-565e1b90d2ad.png)



## Event Hubs と Event Hubs トリガーの実行回数が不一致

**並列実行とチューニング**に示した通り 1 回のトリガー当たりに処理するイベント数を 2 つ以上とすることができます。`host.json` の [`maxEventBatchSize`](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs?tabs=isolated-process%2Cextensionv5&pivots=programming-language-python#hostjson-settings) で設定ができます。
本設定値のデフォルト値は 100 となるため、1 回のトリガー当たり最大で 100 個のイベントが処理されます。そのため、トリガーの実行が 1 回であっても 100 個のイベントが処理されている場合があります。

以下は、8 回のトリガーで 100 個のイベントを処理した例となります。

**traces ログ**

![image-a5346a7a-ccf4-4e0a-8d33-a7b0b3e35a67.png]({{site.baseurl}}/media/2024/02/image-a5346a7a-ccf4-4e0a-8d33-a7b0b3e35a67.png)



# まとめ
本ブログでは Event Hubs トリガー関数の動作及び作成方法について整理しました。Event Hubs トリガーのトラブルシューティングの参考になれば幸いです。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions における Azure Event Hubs のトリガーとバインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs?tabs=isolated-process%2Cextensionv5&pivots=programming-language-csharp)


<br>
<br>

---

<br>
<br>

2024 年 02 月 22 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>