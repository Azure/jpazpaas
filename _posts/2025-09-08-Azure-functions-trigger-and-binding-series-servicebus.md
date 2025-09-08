---
title: "Azure Functions トリガー・バインディング シリーズ - Service Bus トリガー"
author_name: "shotanakano"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本手順
[Service Bus トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp) は Service Bus からイベントを読み取り実行される関数です。

Service Bus トリガーは、[Service Bus のキューとトピック](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-queues-topics-subscriptions) に対し、それぞれ Service Bus キュー トリガーと Service Bus トピック トリガーの 2 種類のトリガーを設定できます。メッセージに対して、単一処理を行うか（キュー）、複数処理を行うか（トピック）により、トリガーを使い分けてご利用ください。なお、キューとトピックの処理は混在させることができないので、それぞれ異なるトリガー（Service Bus キュー トリガーと Service Bus トピック トリガー）を作る必要があります。  
Service Bus のメッセージに対して、[PeekLock](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/message-transfers-locks-settlement#peeklock)  と呼ばれるロックを取得し、ロック更新することで動作します（外部ストレージにどこまで読んだかなどのメモなどは保持しません）。PeekLock では、メッセージの Complete または Abandon を呼び出してメッセージの終わりを示します。

## ロックについて
PeekLock について、Service Bus のキューとサブスクリプションには既定の[ロック期間 1 分](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/message-transfers-locks-settlement#renew-locks) があります。このロックを更新し、ロック状態を維持することを host.json の [maxAutoLockRenewalDuration](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus?tabs=in-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp#hostjson-settings) (規定 5 分) にて定義できます。既定では 1 分ごとに 4 回のロックが更新されます。メッセージのロックが解除され、再度メッセージが
取得されると、DeliveryCount (配信回数と呼ばれメッセージの読み取り回数上限で、既定では 10 が最大) がインクリメントされ、最大配信回数に至ると[配信不能キュー(Dead Letter Queue)](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-dead-letter-queues)にメッセージが移動されます。

別のクライアントによるメッセージの処理を避けるために、Azure Functions にて実装しているアプリケーションの処理時間と同じ以上の長さ maxAutoLockRenewalDuration を設定しておく必要があります。

Service Bus キューの参考画面：

![image-a169cc6d-efc3-4434-b6a4-6985c8613529.png]({{site.baseurl}}/media/2025/09/image-a169cc6d-efc3-4434-b6a4-6985c8613529.png)

PeekLock 時の Complete または Abandon は、[オートコンプリート](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#peeklock-behavior) と呼ばれる機能で Azure Functions が自動で行うことができます。関数の実行に応じて、Complete (実行完了としてメッセージの破棄) または Abandon が自動で呼び出されます。規定ではオートコンプリートは true となっていますが、false とした場合には、アプリケーション コード内でそれぞれのメッセージに対して明示的に [Complete](https://learn.microsoft.com/ja-jp/dotnet/api/microsoft.servicebus.messaging.queueclient.complete?view=azure-dotnet#Microsoft_ServiceBus_Messaging_QueueClient_Complete_System_Guid_)  または [Abandon](https://learn.microsoft.com/ja-jp/dotnet/api/microsoft.servicebus.messaging.queueclient.abandon?view=azure-dotnet)  を呼び出す必要があります。


## セッションについて
Service Bus では、[セッション](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/message-sessions) と呼ばれる概念があります。各メッセージにセッション ID をメタデータとして保持できます。Service Bus トリガーでは、[IsSessionsEnabled 属性](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#attributes)を有効化することで、セッション機能を使用することができます。利用する Service Bus キューやサブスクリプションについても作成時にセッションを有効化しないとこの機能は使えないことをご留意ください。  

![image-7b4370e9-4a7b-44fb-81ec-ecae48191898.png]({{site.baseurl}}/media/2025/09/image-7b4370e9-4a7b-44fb-81ec-ecae48191898.png)


# 作成手順
Service Bus トリガーの作成方法は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-your-first-function-visual-studio#create-a-function-app-project)で紹介されているテンプレートをご利用ください。チュートリアルでは HTTP トリガーを利用しておりますが、同画面にて Service Bus キュー トリガーまたは Service Bus トピック トリガーを選択いただけます。選択いただいたトリガーに応じて [属性](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#attributes) として定義するべき項目が異なるためご注意ください。

![image-5759391f-c15d-43f5-af96-4339b5c6d06c.png]({{site.baseurl}}/media/2025/09/image-5759391f-c15d-43f5-af96-4339b5c6d06c.png)

また、キュー、トピック名など各属性情報は、アプリケーション設定にて管理することができます。C# の例ですが、ServiceBusTrigger の引数で 各属性情報を `%` 括りのアプリケーション設定名とします(上段黄色枠)。ローカルの実行のため local.settings.json となりますが(下段黄色枠)、設定値の中身を宣言します。

Service Bus の [接続文字列](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#connection-string) については `connection` プロパティで指定した値と一致する名前のアプリケーション設定を定義する、または、`connection` プロパティで指定した値に "AzureWebJobs" の接頭字を付与してアプリケーション設定で定義する（青色枠）ことが可能です。

Azure 上の Azure Functions リソースの場合には local.settings.json にて定義いただいた同項目をアプリケーション設定に設定ください。

Service Bus キュー トリガー例：

![image-2b6bf99e-c9d3-457b-a626-9521cd74c098.png]({{site.baseurl}}/media/2025/09/image-2b6bf99e-c9d3-457b-a626-9521cd74c098.png)

Service Bus トピック トリガー例：

![image-9118de59-a23c-4aee-97f3-22751b08ee74.png]({{site.baseurl}}/media/2025/09/image-9118de59-a23c-4aee-97f3-22751b08ee74.png)

# 実行状況をログで確認する
Service Bus トリガーの実行詳細をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。
## host.json
    "logging": {
      "logLevel": {
        "Host.General": "Debug",
        "Function": "Debug",
        "Microsoft.Azure.WebJobs.Hosting": "Debug",
        "Microsoft.Azure.WebJobs.ServiceBus": "Debug",
        "Azure.Messaging.ServiceBus": "Debug"
      }
    }


## Application Insights で利用するクエリ
    traces
    | where timestamp > datetime("yyyy-mm-dd hh:mi")
    | where customDimensions.Category startswith "Microsoft.Azure.WebJobs.ServiceBus" or customDimensions.Category startswith "Azure.Messaging.ServiceBus" or operation_Name =~ "<関数名>"
    | project timestamp, message, customDimensions
    | order by timestamp asc

上記の設定を行った際に記録されるログの例です。
### メッセージ取得
Azure Functions Host で Service Bus クライアント、Service Bus レシーバーの作成、Service Bus への認証を行い（黄色枠）、メッセージ受信を行うポーリング処理（赤色枠の `ReceiveBatchAsync` の処理）を行っていることが記録されています。

Service Bus キュー トリガー例：

![image-38465d84-1d28-4147-bf7e-af98e4a659b1.png]({{site.baseurl}}/media/2025/09/image-38465d84-1d28-4147-bf7e-af98e4a659b1.png)

Service Bus トピック トリガー例：

![image-82255ffb-feb0-48fb-847b-69e01d76bf60.png]({{site.baseurl}}/media/2025/09/image-82255ffb-feb0-48fb-847b-69e01d76bf60.png)


### 実行ログ
Service Bus よりメッセージ取得されたことで、トリガーの実行され `Executing~` と `Executed~` に合わせて `Trigger Details:~` が記録されています。`Trigger Details:~` となっているログでは、Service Bus の[メタ データ](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#message-metadata) が確認でき配信回数、エンキュー時刻やシーケンス番号などが記録されます。

Service Bus キュー トリガー例：

![image-e184c18b-7138-4f2d-ac19-baceee3a76a3.png]({{site.baseurl}}/media/2025/09/image-e184c18b-7138-4f2d-ac19-baceee3a76a3.png)

Service Bus トピック トリガー例：

![image-c08c6f4e-06b2-4aa7-b145-1949f46447f4.png]({{site.baseurl}}/media/2025/09/image-c08c6f4e-06b2-4aa7-b145-1949f46447f4.png)

### ロックの更新
60 秒以上処理時間がかかるアプリケーション（実行ログは青色ハイライト）を実装しています。メッセージを受信したことにより 1:48:49 に処理開始をしています。その後、Service Bus 側のロック期間を超過しないように 1 分以内（下記ログでは 1:49:39 に実施）にロックの更新（RenewMessageLock）を行っていることが確認できます（ロック処理の一連は黄色ハイライト）。最終的に関数実行が完了したことで、Complete （緑色ハイライト）をしています。

![image-e30fb6f5-20d1-4f61-a870-630f87d23017.png]({{site.baseurl}}/media/2025/09/image-e30fb6f5-20d1-4f61-a870-630f87d23017.png)

# 並行処理とチューニング
## 並行処理の考え方
Service Bus トリガーには「単一ディスパッチ処理」「単一ディスパッチ処理(セッションベース)」「バッチ処理」の 3 つの実行モデルがあります。選択いただいた実行モデルに基づきメッセージを何件ずつ処理するかが異なります。また、この構成は Service Bus キュー トリガーと Service Bus トピック トリガー共通となります。

上記の実行モデルは **IsBatched** と **IsSessionsEnabled** の 2 つの属性の組み合わせによって決まります。また実行数に関するプロパティとして、**maxConcurrentCalls**、**maxConcurrentSessions**、**maxMessageBatchSize(maxMessageCount)** がありますが、実行モデルによって有効なプロパティが異なるので留意する必要があります。  
実行モデルに対応する組み合わせと実行数の制御を行うプロパティについては [こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-target-based-scaling?tabs=v5%2Ccsharp#service-bus-queues-and-topics) に紹介がございます。  

![image-55fe0f8f-c0fb-466c-a499-ed1d99603f1d.png]({{site.baseurl}}/media/2025/09/image-55fe0f8f-c0fb-466c-a499-ed1d99603f1d.png)

### 単一ディスパッチ処理
関数が 1 回実行されるとメッセージは 1 件だけ処理されます。  
1 インスタンスで並行処理されるメッセージ数を **maxConcurrentCalls** プロパティで制御できます。  
下記は、.NET 分離ワーカモデルでの実装サンプルです。関数が 1 回実行されるとメッセージは 1 件だけ処理されますので、受け付ける型は `ServiceBusReceivedMessage` です。

    [Function(nameof(ServiceBusReceivedMessageFunction))]
    [ServiceBusOutput("outputQueue", Connection = "ServiceBusConnection")]
    public string ServiceBusReceivedMessageFunction(
        [ServiceBusTrigger("queue", Connection = "ServiceBusConnection")] ServiceBusReceivedMessage message)
    {
        _logger.LogInformation("Message ID: {id}", message.MessageId);
        _logger.LogInformation("Message Body: {body}", message.Body);
        _logger.LogInformation("Message Content-Type: {contentType}", message.ContentType);
    
        var outputMessage = $"Output message created at {DateTime.Now}";
        return outputMessage;
    }

Service Bus に複数メッセージがある場合でも 1 件ずつ受信され、それぞれ関数実行されることが確認できます。

![image-37acc4d2-0fac-483a-a440-b702829dfa2f.png]({{site.baseurl}}/media/2025/09/image-37acc4d2-0fac-483a-a440-b702829dfa2f.png)

### 単一ディスパッチ処理 (セッションベース)
関数が 1 回実行されるとメッセージは 1 件だけ処理されます。  
1 インスタンスで並行処理されるメッセージ数を **maxConcurrentSessions** プロパティで制御できます。  
※基本的には前項の「単一ディスパッチ処理」と違いはなく、セッションベースか否かとなります。

### バッチ処理
関数が 1 回実行されるとメッセージは N 件数分処理されます。最大で **maxMessageBatchSize** プロパティ件数分処理されます。  
1 インスタンスでメッセージを並行処理はできません。  
下記は、.NET 分離ワーカモデルでの実装サンプルです。関数が 1 回実行されるとメッセージは N 件処理されますので、受け付ける型は `ServiceBusReceivedMessage[]` です。

    [Function(nameof(ServiceBusReceivedMessageBatchFunction))]
    public void ServiceBusReceivedMessageBatchFunction(
        [ServiceBusTrigger("queue", Connection = "ServiceBusConnection", IsBatched = true)] ServiceBusReceivedMessage[] messages)
    {
        foreach (ServiceBusReceivedMessage message in messages)
        {
            _logger.LogInformation("Message ID: {id}", message.MessageId);
            _logger.LogInformation("Message Body: {body}", message.Body);
            _logger.LogInformation("Message Content-Type: {contentType}", message.ContentType);
        }
    }

ログからも一回の関数実行にて 2 つのメッセージが処理されていることを確認できます。

![image-724c46c8-6c6f-4026-a46c-414d157109c4.png]({{site.baseurl}}/media/2025/09/image-724c46c8-6c6f-4026-a46c-414d157109c4.png)

# よくお問い合わせいただく動作
弊サポートにてよくお問い合わせいただく事例について紹介します。なお、下記に記載しますログ出力例は、Function Host のランタイム バージョン が v4.1041.200.25360 時点の例となります。バージョンが異なることにより出力内容が異なる可能性がありますことをご留意ください。なお、ログは [Application Insights で利用するクエリ](#application-insights-で利用するクエリ) にて記載したクエリを実行した結果より確認しています。


## デプロイ後に Service Bus トリガーが実行できない
Azure 上に展開した後に動作しない場合の代表的な原因のひとつに、Service Bus の接続情報が正しく設定されていないことがあります。接続文字列が正しくない場合、存在しないキューやトピックを指定している場合、Service Bus 側のネットワーク制限によりアクセス拒否されている場合、Azure Fucntions にて VNet 統合を使用している際に NSG（Network Security Group） により Service Bus への通信が阻害されている場合などがあります。下記のようなログが記録されている場合は設定を見直していただくことで、事象が解消する可能性があります。
* 接続文字列が正しくなく 401 エラーが発生
  
  Azure Function にて設定している接続文字列が正しいか、有効期限内であるかをご確認ください。再設定をすることで正常に動作する可能性があります。

  **traces ログ**

  ![image-16c4b7aa-ecba-4915-afdb-d257ba4f9034.png]({{site.baseurl}}/media/2025/09/image-16c4b7aa-ecba-4915-afdb-d257ba4f9034.png)

  また Service Bus 名前空間より [概要] ブレードの ”ローカル認証” の設定にて SAS キー認証が有効となっているかも併せてご確認ください。

  ![image-c0662c24-78a8-419f-887d-65c15e13b2a8.png]({{site.baseurl}}/media/2025/09/image-c0662c24-78a8-419f-887d-65c15e13b2a8.png)

* 存在しないキューを指定したため 404 エラーが発生

  Azure Function におけるキューやトピック/サブスクリプションの設定が正しいか、Service Bus に対象が存在するかをご確認ください。また使用している接続文字列にて対象のキューやトピック/サブスクリプションが参照できる権限があるかについてもご確認ください。


  **traces ログ**
  
  ![image-2943cbff-dc56-4377-903b-c472f9818b88.png]({{site.baseurl}}/media/2025/09/image-2943cbff-dc56-4377-903b-c472f9818b88.png)

* Service Bus 側のネットワーク制限により 401 エラーが発生

  Service Bus のネットワーク設定をご確認いただき、Azure Function の送信 IP アドレスが許可されていることを確認してください。

  **traces ログ**

  ![image-1646136c-254d-401b-a847-031d6550b846.png]({{site.baseurl}}/media/2025/09/image-1646136c-254d-401b-a847-031d6550b846.png)

  
  Service Bus 名前空間の [ネットワーク] ブレードにて許可されている IP アドレスが確認できます。

  ![image-1d091cb7-5b70-4117-b987-729b9bda0f46.png]({{site.baseurl}}/media/2025/09/image-1d091cb7-5b70-4117-b987-729b9bda0f46.png)

  既定では Azure Functions の送信 IP アドレスは使用可能なアドレスの内 1つを使用するため、全てのアドレスを許可いただく必要があります。IP アドレスの確認方法は [こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/ip-addresses?tabs=portal#find-outbound-ip-addresses) を参照ください。

  ![image-ae532b9a-333e-46a5-b891-38c80e9f55de.png]({{site.baseurl}}/media/2025/09/image-ae532b9a-333e-46a5-b891-38c80e9f55de.png)

  [Azure 仮想ネットワーク NAT ゲートウェイを使用して Azure Functions の送信 IP を制御](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-how-to-use-nat-gateway) し、送信 IP アドレスを固定化をすることで特定の IP アドレスのみ許可いただくことでも実現いただけます。

* NSG により Service Bus へ通信できない状況でソケットエラーが発生

  Azure Function に統合いただいている VNet の NSG により、Outbound 通信が制限されていないかをご確認ください。

  **traces ログ**

  ![image-20c95a59-bd18-4e98-a26b-e88352ff1268.png]({{site.baseurl}}/media/2025/09/image-20c95a59-bd18-4e98-a26b-e88352ff1268.png)

  Azure Function に統合いただいている VNet の NSG のOutbound 通信の設定として、 ServiceBus のサービス タグで [Service Bus の通信に必要なポート（5671, 5672, 443）](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-faq#----------------------------) を許可する必要があります。なお、その他に　Azure Function に必要な通信設定はここでは割愛しているため、お客様の通信要件に合わせてその他設定もする必要があることをご留意ください。

  ![image-2906e519-13c9-4fa6-979d-422de7b2f89c.png]({{site.baseurl}}/media/2025/09/image-2906e519-13c9-4fa6-979d-422de7b2f89c.png)

## Message processing error という例外が発生した
ロック機構に関連してエラーが発生しています。 ServiceBus.ServiceBusException の内容を確認することで詳細なエラーが確認でき、下記の例ではアプリケーション処理時間が maxAutoLockRenewalDuration にて定義しているロック更新期間を超えた際のログとなります。Service Bus におけるロックの状態や実際のアプリケーションの処理時間を確認することで原因を切り分けいただけます。

  **traces ログ**

![image-5f8bac8d-665c-4e35-8a5b-a82cc03ca23c.png]({{site.baseurl}}/media/2025/09/image-5f8bac8d-665c-4e35-8a5b-a82cc03ca23c.png)

エラーが発生した際は Service Bus 側のメッセージ状態を確認し、状況に応じて対処ください。

* メッセージがキューやトピックに残存している場合

  自動で再試行されるため、対処不要となります。下記の例では処理失敗したメッセージの配信回数がインクリメントされた状態となります。再度メッセージ受信されることで処理されます。Service Bus のキューやトピックの [Service Bus エクスプローラー] ブレードより下記の画面をご確認いただけます。

  ![image-4a464d96-79e6-4273-8712-82a2777db081.png]({{site.baseurl}}/media/2025/09/image-4a464d96-79e6-4273-8712-82a2777db081.png)

  

* 配信不能キューにメッセージが存在する場合

  対象のメッセージのプロパティを確認することで、配信不能になった理由を確認できます。下記例では最大配信回数超過によりメッセージが移動されていますので、アプリケーションが処理できる状態であるか、処理時間に問題はないかなどをご確認いただいた上で、メッセージの再送信をすることで復旧いただける可能性があります。Service Bus のキューやトピックの [Service Bus エクスプローラー] ブレードより下記の画面をご確認いただけます。

  ![image-a0d39151-c6ee-4c6c-93a2-c5cf01d06e6f.png]({{site.baseurl}}/media/2025/09/image-a0d39151-c6ee-4c6c-93a2-c5cf01d06e6f.png)


* メッセージが配信不能キューを含め、キューやトピックに存在しない場合

  再試行処理などにより処理が完了している状態となりますので、正常終了しています。

# まとめ
本ブログでは Service Bus トリガー関数の動作及び作成方法について整理しました。Service Bus トリガーのトラブルシューティングの参考になれば幸いです。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions における Azure Service Bus のバインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-service-bus?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp)

[Azure Service Bus のメッセージング - キュー、トピック、およびサブスクリプション - Azure Service Bus](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-queues-topics-subscriptions)

[Azure Service Bus のメッセージ セッション - Azure Service Bus](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/message-sessions)

[Azure Service Bus メッセージの転送、ロック、および解決 - Azure Service Bus](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/message-transfers-locks-settlement)
<br>
<br>

---

<br>
<br>

2025 年 09 月 08 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>