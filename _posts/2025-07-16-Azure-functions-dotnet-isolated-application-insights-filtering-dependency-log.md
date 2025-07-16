---
title: "Azure Functions .NET Isolated（分離ワーカー）モデルで Dependency の一部のログをフィルタリングして Application Insights に送信する方法"
author_name: "Tatsuya Yamamura"
tags:
    - Function App
---

------------------------------------------------------------------------

# はじめに
お世話になっております。App Service サポート担当の山村です。いつも Azure Functions（関数アプリ）をご利用いただき誠にありがとうございます。

Azure Functions を使用してサーバーレスアプリケーションを開発・運用する際、アプリケーションや実行状況のログ監視は非常に重要です。
Azure Functions では Application Insights と統合することでこれらの監視が可能です。

本ブログでは、Azure Functions の .NET Isolated (分離ワーカー) モデルにおいて `Language Worker → Application Insights` のログ送信パイプラインをご利用いただいている場合に、Application Insights への 依存関係（Dependency）ログの送信について、特定の `type (HTTP etc)` や `target (www.microsoft.com etc)` 毎に制御する方法を解説します。

本ブログに記載の内容は、Dependency に関するログ保存のコストを抑えたいというユースケースでご利用いただけます。
Dependency ログの `type (HTTP etc)` や `target (www.microsoft.com etc)` 属性の値をもとに Dependency ログの保存有無を制御したい場合に、本内容が有効となります。


# Azure Functions のアーキテクチャ概略

Azure Functions では、Functions Host と呼ばれるプロセスと、Language Worker と呼ばれるプロセスが動作しています。

Functions Host は、HTTP トリガーや Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を担い、Azure Functions のランタイムとして動作する Language Worker へ処理を要求します。 
Language Worker は各言語に応じた関数コードを実行するプロセスであり、お客様のアプリケーションコードが実行されます。
Azure Functions のアーキテクチャの詳細は、[こちらのブログ](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)をご参照ください。

![image-47a83b48-dab2-4d99-aaa6-82f19e20635a.png]({{site.baseurl}}/media/2025/07/image-47a83b48-dab2-4d99-aaa6-82f19e20635a.png)


# Application Insights へのログ送信フロー

Azure Functions .NET Isolated を Application Insights と統合している場合には、Language Worker が Application Insights へログを送信するパイプラインは以下の二通りがあります。

1) `Language Worker → Functions Host → Application Insights`
2) `Language Worker → Application Insights`

詳細については、[こちらのブログ](https://azure.github.io/jpazpaas/2025/05/07/Azure-functions-dotnet-isolated-application-insights-key-considerations.html)をご参照ください。

なお、2) の場合は、Language Worker が出力するログについては、Functions Host を経由しないため host.json の Application Insights の設定は効果がありません。
Language Worker が出力するログに関する Application Insights の設定は、スタートアップ構成の `Program.cs` 内で実装する必要があります。


# 本題: Language Worker → Application Insights ログ送信パイプラインの場合の Dependency ログの送信について、特定の type や target 毎に制御する方法

## 実装例
スタートアップ構成の `Program.cs` 側で ITelemetryProcessor を用いて制御します。

まずは、外部エンドポイントに HTTP リクエストを実施するアプリケーションコード例を示します。

**HTTP トリガー関数の実装例** <br>
```csharp
using Microsoft.AspNetCore.Http;  
using Microsoft.AspNetCore.Mvc;  
using Microsoft.Azure.Functions.Worker;  
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Functions.Worker.Http; //追加
namespace Company.Function;
  
public class HttpTrigger1  
{  
   private readonly ILogger<HttpTrigger1> _logger;  
   private static readonly HttpClient httpClient = new HttpClient();
   public HttpTrigger1(ILogger<HttpTrigger1> logger)  
   {  
       _logger = logger;  
   }
   [Function("HttpTrigger1")]  
   public async Task<HttpResponseData> Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req)  
   {  
       _logger.LogInformation("C# HTTP trigger function processed a request starting...");  
       var response = await httpClient.GetAsync("https://www.microsoft.com");  
       _logger.LogInformation(response.StatusCode.ToString());
       var res = req.CreateResponse(response.IsSuccessStatusCode  
           ? System.Net.HttpStatusCode.OK  
           : System.Net.HttpStatusCode.BadGateway);
       await res.WriteStringAsync($"Status: {response.StatusCode}");  
       return res;  
   }  
}  
```


次に、ビルダー構成時に `AddApplicationInsightsTelemetryProcessor` を用いて、Telemetry データのフィルタリングと加工処理を適用します。

**Program.cs** <br>
```csharp
using Microsoft.Azure.Functions.Worker;  
using Microsoft.Azure.Functions.Worker.Builder;  
using Microsoft.Extensions.DependencyInjection;  
using Microsoft.Extensions.Hosting;  
using Microsoft.Extensions.Logging;
using IsolatedApplicationInsightsCustomTelemetryProcessor;
var builder = FunctionsApplication.CreateBuilder(args);

builder.Services  
   .AddApplicationInsightsTelemetryWorkerService()  
   .ConfigureFunctionsApplicationInsights()  
   .AddApplicationInsightsTelemetryProcessor<CustomTelemetryProcessor>();
  
builder.Logging.Services.Configure<LoggerFilterOptions>(options =>  
{  
   LoggerFilterRule toRemove = options.Rules.FirstOrDefault(rule =>  
       rule.ProviderName == "Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider");
   if (toRemove is not null)  
   {  
       options.Rules.Remove(toRemove);  
   }  
});  
  
builder.Build().Run();  
```


ITelemetryProcessor で具体的なフィルタリング処理を追加します。<br>
※ CustomTelemetryProcessor.cs を新しいファイルとして作成し、Program.cs にインポートするような実装としています。

**CustomTelemetryProcessor.cs** <br>
```Csharp
namespace IsolatedApplicationInsightsCustomTelemetryProcessor  
{  
   using Microsoft.ApplicationInsights.Channel;  
   using Microsoft.ApplicationInsights.DataContracts;  
   using Microsoft.ApplicationInsights.Extensibility;
   public class CustomTelemetryProcessor : ITelemetryProcessor  
   {  
       private ITelemetryProcessor Next { get; set; }
       public CustomTelemetryProcessor(ITelemetryProcessor next)  
       {  
           this.Next = next;  
       }
       public void Process(ITelemetry telemetry)  
       {  
           // if (telemetry is DependencyTelemetry dependency && dependency.Target == "www.microsoft.com") // target URL or service name etc.  
           // if (telemetry is DependencyTelemetry dependency && dependency.Type == "Http") // "Http", "InProc" etc.  
           if (telemetry is DependencyTelemetry dependency) // Don't send all dependencies telemetry  
           {  
               Console.WriteLine($"target: {dependency.Target}");  
               Console.WriteLine($"type: {dependency.Type}");  
               return;  
           }  
           this.Next.Process(telemetry);  
       }  
   }  
}
```

## 具体的なフィルタリング例
上記までのアプリケーションコードを Azure Functions にデプロイします。<br>
また、実行した際の Application Insights のログに対して、下記 KQL クエリで Dependency のログが記録されているかを確認します。
```KQL
union requests, traces, dependencies, exceptions
| where timestamp > ago(30d)
| where operation_Id == '<対象の実行の operation id を入力>' or customDimensions['InvocationId'] == '<対象の実行の invocation id を入力>'
| order by timestamp asc
| project-reorder timestamp, itemType, target, type, sdkVersion, message
```


### 1. すべての Dependency ログを送信する場合
Program.cs で `AddApplicationInsightsTelemetryProcessor` を利用しなければ、すべての Dependency のログが送信されます。

![image-29f168f8-d4b6-44f8-8c44-e4b5994afac5.png]({{site.baseurl}}/media/2025/07/image-29f168f8-d4b6-44f8-8c44-e4b5994afac5.png)

### 2. すべての Dependency ログを送信しない場合
ITelemetryProcessor で `if (telemetry is DependencyTelemetry dependency)` とします。<br>
下記のように Dependency のログは記録されません。

![image-36d61019-319b-405b-9e48-8da291d6a5a9.png]({{site.baseurl}}/media/2025/07/image-36d61019-319b-405b-9e48-8da291d6a5a9.png)


### 3. 特定の target のみを送信しない場合
ITelemetryProcessor で `if (telemetry is DependencyTelemetry dependency && dependency.Target == "www.microsoft.com")` とします。<br>
下記のように Dependency のログのうち "www.microsoft.com" を target とするログは記録されません。

![image-de3c40b8-6688-4733-a720-a0bfd07ed1a6.png]({{site.baseurl}}/media/2025/07/image-de3c40b8-6688-4733-a720-a0bfd07ed1a6.png)


ITelemetryProcessor で `if (telemetry is DependencyTelemetry dependency && dependency.Target == "")` とします。<br>
※ Application Insights 側に送信されると target が Invoke となっているように見えますが、ローカルでは null なので null でフィルタリングします。<br>
下記のように Dependency ログのうち "Invoke" を target とするログは記録されません。

![image-4fa9d2fc-ebd6-485a-a5a7-695e1206dfa0.png]({{site.baseurl}}/media/2025/07/image-4fa9d2fc-ebd6-485a-a5a7-695e1206dfa0.png)

### 4. 特定の type のみを送信しない場合
ITelemetryProcessor で `if (telemetry is DependencyTelemetry dependency && dependency.Type == "Http")` とします。<br>
下記のように Dependency ログのうち "Http" を type とするログは記録されません。

![image-cf83a03d-2f19-4554-8a16-07b46ea00205.png]({{site.baseurl}}/media/2025/07/image-cf83a03d-2f19-4554-8a16-07b46ea00205.png)

ITelemetryProcessor で `if (telemetry is DependencyTelemetry dependency && dependency.Type == "InProc")` とします。<br>
下記のように Dependency ログのうち "InProc" を type とするログは記録されません。

![image-97cd9549-160a-4740-88ce-94550c3a9c8e.png]({{site.baseurl}}/media/2025/07/image-97cd9549-160a-4740-88ce-94550c3a9c8e.png)


# 最後に
上記の通り、ITelemetryProcessor を用いてフィルタリングを追加することで、特定の target や type のみを Application Insights に送信しないことを実現可能ですが、本番や運用環境でお試しいただく前に、検証環境でお客様想定通りとなるかを確認してご利用いただくようにお願いいたします。
また、Application Insights にログを送信しないと、問題発生時のトラブルシューティングが難しくなる場合もございますので、どのログをフィルタリングするかは要検討いただく必要がございます。

## 参考リンク

* [Application Insights SDK におけるフィルター処理および前処理 - Azure Monitor | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/api-filtering-sampling?tabs=dotnetcore%2Cjavascriptwebsdkloaderscript)


<br>
<br>

2025 年 07 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>