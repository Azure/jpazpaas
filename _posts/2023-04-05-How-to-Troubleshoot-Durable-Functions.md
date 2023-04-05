---
title: "Durable Functions のトラブルシューティングガイド"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - Durable Functions
---

# 質問
Durable Functions のトラブルシューティングについて確認方法を教えてください。

# 回答
Duralbe Functions の動作を解説し、トラブルシューティング方法について説明します。
##Durable Functions の動作
簡単に Durable Functions の動作について解説します。

Durable Functions で実装される関数は、大きくオーケストレーター関数とアクティビティ関数の２種類に分けることができます。
- [オーケストレーター関数](
https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp)
は、アクティビティ関数のような他の関数の処理順番を制御する関数となります。構成のパターンについては[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-overview?tabs=csharp-inproc#application-patterns)を参照ください。標準の実装では 1 回の Durable Functions の実行ごとに、一意となるオーケストレーション ID が発行されてそれぞれで一連の Durable Functions が動作いたします。

- [アクティビティ関数](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-types-features-overview#activity-functions)は、業務処理の実体となります。オーケストレーター関数から呼び出されるため、引数でのデータの入出力はオーケストレーター関数を介して実施いただくことができます。

次のコードを考えます。Durable Functions の SDK バージョンは、2.9.3 を利用しています。

・TestFunction_Orchestrator…オーケストレーター関数

・TestFunction_Activity…アクティビティ関数

・TestFunction_OrchestratorStart…Durable Functions の開始を行うための HTTP トリガー

となっており、HTTPリクエストによって TestFunction_OrchestratorStart が呼び出されることで Durable Functions が起動します(33 行目)。オーケストレーター関数 TestFunction_Orchestrator からは、アクティビティ関数 TestFunction_Activity がそれぞれ引数が異なる形で順番に 1 つづつ呼び出しています(11~13 行目)。これは[関数チェーン パターン](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-overview?tabs=csharp-inproc#chaining)となります。

```
01 public static class Function1
02     {
03         [FunctionName("TestFunction_Orchestrator")]
04         public static async Task<List<string>> RunOrchestrator(
05             [OrchestrationTrigger] IDurableOrchestrationContext context)
06         {
07             var outputs = new List<string>();
08 
09             // Replace "hello" with the name of your Durable Activity Function.
10             outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo"));
11             outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"));
12             outputs.Add(await context.CallActivityAsync<string>(nameof(SayHello), "London"));
13 
14             // returns ["Hello Tokyo!", "Hello Seattle!", "Hello London!"]
15             return outputs;
16         }
17 
18         [FunctionName("TestFunction_Activity")]
19         public static string SayHello([ActivityTrigger] string name, ILogger log)
20         {
21             log.LogInformation($"Saying hello to {name}.");
22             return $"Hello {name}!";
23         }
24 
25         [FunctionName("TestFunction_OrchestratorStart")]
26         public static async Task<HttpResponseMessage> HttpStart(
27             [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestMessage req,
28             [DurableClient] IDurableOrchestrationClient starter,
29             ILogger log)
30         {
31             // Function input comes from the request content.
32             string instanceId = await starter.StartNewAsync("TestFunction_Orchestrator", null);
33 
34             log.LogInformation($"Started orchestration with ID = '{instanceId}'.");
35 
36             return starter.CreateCheckStatusResponse(req, instanceId);
37         }
38     }
```
実際に上記のコードをデプロイすると 3 つの関数が作成されることが確認できます。

![image-0bd91a10-f596-4837-a470-e3e41c54769f.png]({{site.baseurl}}/media/2023/04/image-0bd91a10-f596-4837-a470-e3e41c54769f.png)

他のパターンの実装例は、[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-sequence?tabs=csharp)や[クイック スタート](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-create-first-csharp?pivots=code-editor-vscode)を参照ください。


次に、オーケストレーター関数とアクティビティ関数のやり取りについて少し解説します。

- オーケストレーター関数は、アクティビティ関数の処理順番を制御すると説明しました。実際に処理を制御する方法として、**ストレージ アカウントのテーブルとキュー**を利用しています。

- 各オーケストレーション ID ごとに、どこまで処理を行ったかの情報をテーブルに記録しておきます。アクティビティ関数を呼び出す際には、オーケストレーター関数がキューにメッセージを追加することでキューをポーリングしているアクティビティ関数がメッセージを検知して実行が開始されます。
この動作の際にオーケストレーター関数は実行開始、実行完了、アクティビティ関数の実行完了待といった状態が遷移するためログを確認いただいた際には複数回オーケストレーター関数が実行されているような状況が確認できます。

- テーブルには 2 種類あり、標準では TestHubNameInstances（オーケストレーション ID ごとの実行開始終了時刻やステータス情報など） と TestHubNameHistory（オーケストレーション ID のオーケストレーター関数及びアクティビティ関数ごとの実行開始終了や実行結果など） があります。出力例は下記となります。

![image-3d1878db-2b2f-4669-99ff-bf4e949cb2c7.png]({{site.baseurl}}/media/2023/04/image-3d1878db-2b2f-4669-99ff-bf4e949cb2c7.png)

- オーケストレーター関数からアクティビティ関数の実行はアクティビティ作業項目キューと呼ばれるキューが利用されます。キューの名前は、<task hub 名>-workitems( [標準](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp#hostjson-functions-20-1) では testhubname-workitems) にメッセ－ジが作成されます。キュー メッセージが消えない限りはアクティビティ関数が完了せず実行され続けます。 [キュー メッセ－ジの有効期限が 7 日間](https://learn.microsoft.com/ja-jp/azure/storage/queues/storage-queues-introduction)となるため、何かしらの要因によってアクティビティ関数の実行が（実行結果の成否問わず）完了しない場合には最大で 7 日間実行され続けます。

以上の動作でオーケストレーター関数の開始及びオーケストレーション ID の生成から、テーブルやキューといった揮発しにくいストレージに情報を保持しておくことで持続的にアプリケーションの実行が行われます。

下記のドキュメントにも動作解説があります。一連の動作にて利用されるストレージ アカウントのテーブルやキューの集まりを**タスク ハブ**と呼称しており、host.json の設定によって任意のタスク ハブ名（標準では TestHubName が利用され各種ストレージの接頭語に利用されています。）を指定することができます。

[Durable Functions におけるタスク ハブ (Azure Functions)](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp#work-items)

![image-82c929e6-ec4a-4126-b9b3-60b7fdb76ab5.png]({{site.baseurl}}/media/2023/04/image-82c929e6-ec4a-4126-b9b3-60b7fdb76ab5.png)

ローカル環境での実行をお勧めしますが、もしもっと具体的にログから動作を確認したい場合には、host.json(Azure Functions の設定ファイル) の[ログ レベル](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-monitoring?tabs=v2#configure-log-levels)を Trace に変更いただくと Durable Functions の詳細な動作をご確認いただくことができます。
以降の Application Insights の確認のシナリオでもこのログ レベルを基準に説明しております。
```
{
    "version": "2.0",
    "logging": {
      "logLevel": {
      "default": "Trace"
    },
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": false,
        "excludedTypes": "Request"
      }
    }
  }
}
```

## トラブルシューティング
問題の診断と解決や Application Insights などを使ってどういった観点でトラブルシューティングをしていけばよいかを紹介します。<br>

### 1. 問題の診断と解決

Azure ポータルで確認いただける問題の診断と解決に、耐久性のある関数(Durable Functions) と呼ばれる画面があります。ないかしら問題の有無について簡易的な診断を行うことが可能です。

**表示画面例**

![image-e7eea426-a533-4161-abf4-ceb5bd583608.png]({{site.baseurl}}/media/2023/04/image-e7eea426-a533-4161-abf4-ceb5bd583608.png)

### 2. Application Insights などログを使ったトラブルシューティング

ある Durable Functions の実行が遅いや期待した時間内で完了しないシナリオを考えます。多くの場合にはいずれかのアクティビティ関数が実行し続けており完了していないことが考えられます。また、先に解説したタスク ハブ上の情報連携のどこかで処理が失敗している可能性があります。

#### 2-1. 実行完了していないオーケストレーション ID を特定します。

次の API([すべてのインスタンス ステータスを取得する](https://learn.microsoft.com/ja-jp/azure/azure-functions/durable/durable-functions-http-api#get-all-instances-status)) を実行して実行完了していないオーケストレーション ID を特定します。インスタンスごとに実行状態が確認できるため、Completed 以外の状態となっているオーケストレーション ID を確認します。

<応答例>

![image-7acb09bb-c5ea-4ec3-af96-939330258cf1.png]({{site.baseurl}}/media/2023/04/image-7acb09bb-c5ea-4ec3-af96-939330258cf1.png)

また、前述したテーブルの TestHubNameInstances を確認すると、CompletedTime が空となっている行が確認できます。このときの PartitionKey がオーケストレーション ID（今回は f71ac761454f41468d9923ac39e9bdbc） になります。

![image-af057bee-f728-4770-b43d-efd8fc62d777.png]({{site.baseurl}}/media/2023/04/image-af057bee-f728-4770-b43d-efd8fc62d777.png)

*Durable Functions の API を利用する際にはパラメータに code を追加する必要があります。Azure Functions がリスンしているエンドポイントは [API キー](https://learn.microsoft.com/ja-jp/azure/azure-functions/security-concepts?tabs=v4#system-key)による保護が標準で有効化されているため、Durable Functions のインスタンス ステータスを取得する際には **API キーを与える必要があります（与えないと 401 となります）。** キーの参照方法については、[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=in-process%2Cfunctionsv2&pivots=programming-language-csharp#obtaining-keys)を参照ください。

#### 2-2. Application Insights を確認します。
以下のクエリを実行すると、次のような結果が得られます。するとオーケストレーター関数からアクティビティ関数が順番に呼び出されている様子が確認できます。

Executing から始まるログと Executed から始まるログまでの間（黄色ハイライト部分）でひとつの関数の実行となります。また、オーケストレーター関数が実行されたのちに、アクティビティ関数が実行されている状況が確認できます。これは、先の説明の通りオーケストレーター関数がアクティビティ関数の実行契機を与えるためです。

さらに、オーケストレーション ID(緑ハイライト部分) は Durable Functions の動作の一連が確認できます。一番最初に TestFunction_OrchestratorStart の実行の中でオーケストレーター関数 TestFunction_Orchestrator の起動をしていること(```Function 'TestFunction_Orchestrator (Orchestrator)' scheduled. Reason: NewInstance.~```)がわかります。その後タスク ハブの各種情報更新を行い、実際に TestFunction_Orchestrator の処理に入った際には、アクティビティ関数 TestFunction_Activity の実行計画(```Function 'TestFunction_Activity (Activity)' scheduled. Reason: TestFunction_Orchestrator.~```)を行った後に一回の実行は終わるものの、await の状態に戻る(```Function 'TestFunction_Orchestrator (Orchestrator)' awaited.~```)ことがわかります。その実行計画に従い TestFunction_Activity が動作していることがわかります。また、ログの中に散見される \#0 や \#1 は、TaskScheduleId と呼ばれており、そのオーケストレーション ID 内でのタスクの実行単位を表します。今回の例では 1 回目のアクティビティ関数の実行がタスク \#0  となっており、後続でインクリメントされていきます。この値は、TestHubNameHistory の TaskScheduleId と一致します。
```
traces
| where timestamp >= datetime("2023-04-05 03:30")
| where message startswith "f71ac761454f41468d9923ac39e9bdbc" or operation_Name != "" 
| order by timestamp asc 
```

![image-8e52489a-e70f-42ab-94cb-0dc8d3439341.png]({{site.baseurl}}/media/2023/04/image-8e52489a-e70f-42ab-94cb-0dc8d3439341.png)

さらにログを確認するとログが途中で切れているものがあることがわかります。オーケストレーター関数が動作し、アクティビティ関数(TaskScheduleId \#2 の実行)の実行計画を行い、それを受けてアクティビティ関数が起動しています。本来であれば Executing から始まるログがあり、アプリケーションで出力している "Saying hello to London." の次に Executed から始まるログがあるはずですが、出力されていません。
項番 1 で確認したこのインスタンスの状態から考えても、このアクティビティ関数 TestFunction_Activity が完了していないことが問題とわかります。

![image-5b75a437-dd4c-49a0-8a40-1c837ec31ec4.png]({{site.baseurl}}/media/2023/04/image-5b75a437-dd4c-49a0-8a40-1c837ec31ec4.png)

さらに追加のログがあればアプリケーションの具体的にどこまで処理が進んだかなどご確認いただけます。実は今回はデバッグ実行で途中で処理を止めており、意図的に処理途中で Durable Functions を終了させていました。このような状態になってもタスク ハブ(アクティビティ関数の実行であれば作業項目キュー)にデータが残っているために次回の起動時に未完了の箇所から処理が再開されますのでご安心ください。

<作業項目キューに残存している TaskScheduleId \#2 の実行計画>

![image-377b5135-4712-4f7f-a5d6-707396b4c836.png]({{site.baseurl}}/media/2023/04/image-377b5135-4712-4f7f-a5d6-707396b4c836.png)
<作業項目キューのメッセージの内容例>
```
{
   "$type":"DurableTask.AzureStorage.MessageData",
   "ActivityId":"d96bfaf1-e29d-4f65-8e64-502d6112f967",
   "TaskMessage":{
       "$type":"DurableTask.Core.TaskMessage",
       "Event":{
           "$type":"DurableTask.Core.History.TaskScheduledEvent",
           "EventType":4,
           "Name":"TestFunction_Activity",
           "Version":"",
           "Input":"[\"London\"]",
           "EventId":2,
           "IsPlayed":false,
           "Timestamp":"2023-04-05T03:31:23.6268922Z"
       },
       "SequenceNumber":0,
       "OrchestrationInstance":{
           "$type":"DurableTask.Core.OrchestrationInstance",
           "InstanceId":"f71ac761454f41468d9923ac39e9bdbc",
           "ExecutionId":"f6ac7180e1ac41498e8afe327ee18518"
       },
       "OrchestrationExecutionContext":{
           "$type":"DurableTask.Core.OrchestrationExecutionContext",
           "OrchestrationTags":{
               "$type":"System.Collections.Generic.Dictionary`2[[System.String, System.Private.CoreLib],[System.String, System.Private.CoreLib]]"
           }
       }
   },
   "CompressedBlobName":null,
   "SequenceNumber":6,
   "Episode":3,
   "Sender":{
       "InstanceId":"f71ac761454f41468d9923ac39e9bdbc",
       "ExecutionId":"f6ac7180e1ac41498e8afe327ee18518"
   },
   "SerializableTraceContext":null
}
```

同様の方法で Durable Functions 及びアプリケーションの動作がどこまで進んでいるのかを確認いただくことができます。

今回の内容は以上となります。ご覧いただきありがとうございました。

<br>
<br>

---

<br>
<br>

2023 年 04 月 05 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>