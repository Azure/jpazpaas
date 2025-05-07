---
title: "Azure Functions .NET Isolated（分離ワーカー）モデルで Application Insights を利用する際の注意点"
author_name: "Tatsuya Yamamura"
tags:
    - Function App
---

# はじめに
お世話になっております。App Service サポート担当の山村です。いつも Azure Functions（関数アプリ）をご利用いただきましてありがとうございます。

Azure Functions を使用してサーバーレスアプリケーションを開発・運用する際、アプリケーションや実行状況のログの監視は非常に重要です。
Azure Functions では Application Insights と統合することで、これらの監視が可能です。

本ブログでは、.NET Isolated（分離ワーカー）モデルを例に、`Language Worker → Functions Host → Application Insights` と `Language Worker → Application Insights` の 2 種類のログ送信パイプラインの設定方法と注意点、ログ出力結果の違いについて解説します。


# Azure Functions のアーキテクチャ概略

Azure Functions では、Functions Host と呼ばれるプロセスと、Language Worker と呼ばれるプロセスが動作しています。

Functions Host は、HTTP トリガーや Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を担い、Azure Functions のランタイムとして動作する Language Worker へ処理を要求します。 
Language Worker は各言語に応じた関数コードを実行するプロセスであり、お客様のアプリケーションコードが実行されます。
Azure Functions のアーキテクチャの詳細は、[こちらのブログ](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)をご参照ください。

![image-47a83b48-dab2-4d99-aaa6-82f19e20635a.png]({{site.baseurl}}/media/2025/05/image-47a83b48-dab2-4d99-aaa6-82f19e20635a.png)


# Application Insights へのログ送信フロー

Azure Functions を Application Insights と統合している場合、通常は Language Worker 上で動作するお客様のアプリケーションで書き込まれたログは Functions Host を経由して Application Insights に送信されます。

ログ送信パイプラインは以下の通りです。

```
Language Worker → Functions Host → Application Insights
```

一方、.NET Isolated（分離ワーカー）モデルおよび Java の場合は、Language Worker 上で動作するアプリケーションから直接 Application Insights へログを送信することが可能です。
```
Language Worker → Application Insights
````

# Language Worker → Functions Host → Application Insights ログ送信パイプラインの構成方法

現時点では、Visual Studio や VS Code で Azure Functions の .NET Isolated モデルプロジェクトを作成すると、`Language Worker → Functions Host → Application Insights` ログ送信パイプラインとなるようなひな型が作成されます。

この構成では、以下のようなファイルが生成されます。

**host.json** <br>
```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      },
      "enableLiveMetricsFilters": true
    }
  }
}
````

**Program.cs** <br>
```csharp
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.Hosting;

var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication();

// Application Insights isn't enabled by default. See [https://aka.ms/AAt8mw4](https://aka.ms/AAt8mw4).
// builder.Services
//     .AddApplicationInsightsTelemetryWorkerService()
//     .ConfigureFunctionsApplicationInsights();

builder.Build().Run();
```

**.csproj（抜粋）** <br>
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>_02_logcheck</RootNamespace>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <!-- Application Insights isn't enabled by default. See [https://aka.ms/AAt8mw4](https://aka.ms/AAt8mw4). -->
    <!-- <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" /> -->
    <!-- <PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="2.0.0" /> -->
    <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="2.0.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="3.2.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="2.0.1" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.0.0" />
  </ItemGroup>
</Project>
```

**HTTP トリガー関数の例** <br>
```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace Company.Function
{
    public class HttpTrigger1
    {
        private readonly ILogger<HttpTrigger1> _logger;

        public HttpTrigger1(ILogger<HttpTrigger1> logger)
        {
            _logger = logger;
        }

        [Function("HttpTrigger1")]
        public IActionResult Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequest req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request. (not modify Program.cs)");
            return new OkObjectResult("Welcome to Azure Functions!");
        }
    }
}
```

# Language Worker → Application Insights ログ送信パイプラインの構成方法

この構成を実現するには、上記ひな形で作成されたプロジェクトに以下の変更が必要です。

**.csproj の変更** <br>
必要なパッケージをインストールするために、*.csproj ファイル内で、ひな形作成時に以下の部分がコメントアウトされているので、コメントアウトを外します。
```xml
<PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="2.0.0" />
```

**Program.cs の変更** <br>
Program.cs ファイル内で、ひな形作成時に以下の部分がコメントアウトされているので、コメントアウトを外します。
```csharp
builder.Services
    .AddApplicationInsightsTelemetryWorkerService()
    .ConfigureFunctionsApplicationInsights();
```

このようにすることで、`Language Worker -> Application Insights` ログ送信パイプラインの構成となります。

ただし、既定で Application Insights SDK は、Warning とそれより深刻なログのみをキャプチャするようにロガーに指示するログ フィルターが追加されていますので、ILogger の Information レベルも記録されるように、このログフィルターを無効にするように設定します。
無効にするためには、サービス構成の一部としてフィルター規則を削除します。
Program.cs に下記のように追加すれば実現できます。
```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = FunctionsApplication.CreateBuilder(args);

builder.Services
    .AddApplicationInsightsTelemetryWorkerService()
    .ConfigureFunctionsApplicationInsights();

// ここから
builder.Logging.Services.Configure<LoggerFilterOptions>(options =>
    {
        LoggerFilterRule defaultRule = options.Rules.FirstOrDefault(rule => rule.ProviderName
            == "Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider");
        if (defaultRule is not null)
        {
            options.Rules.Remove(defaultRule);
        }
    });
// ここまで

builder.Build().Run();
```


# Application Insights へのログ送信のサンプリングの設定等の制御方法の違い
上記のように 2 種類のログ送信パイプラインがありますが、どちらを使用するかによって Application Insights へのログ送信のサンプリングの設定等の制御方法が異なります。
`Language Worker → Functions Host → Application Insights` の場合は、host.json に設定し、他方で、`Language Worker → Application Insights` の場合は、Program.cs に設定します。
※ Functions Host 側のログは引き続き host.json で制御されます。

# Application Insights に記録されたログの確認
`Language Worker → Functions Host → Application Insights` と `Language Worker → Application Insights` ログ送信パイプライン設定で作成したアプリケーションをそれぞれ Azure Functions にデプロイし、HTTP トリガーを実行し、Application Insights に記録されたログを確認します。

HTTP トリガー実行に関するログだけを抽出しました。

赤枠が `Language Worker → Functions Host → Application Insights` パイプラインで、HTTP トリガーアプリケーション内で `_logger.LogInformation("C# HTTP trigger function processed a request. (not modify Program.cs)");` として出力したアプリケーションログの sdkVersion を確認すると、4.1038.400.2 となっており、Functions Host のバージョンとなっていることがわかります。
つまり、Functions Host の SDK を使用して送信されていることを意味しています。

一方で、青枠が `Language Worker → Application Insights` パイプラインで、HTTP トリガーアプリケーション内で `_logger.LogInformation("C# HTTP trigger function processed a request. (modify Program.cs)");` として出力したアプリケーションログの sdkVersion を確認すると、azurefunctions-netiso: 2.0.0 となっており、csproj で指定した Microsoft.Azure.Functions.Worker.ApplicationInsights のバージョンとなっていることがわかります。
つまり、Language Worker の Application Insights から送信されていることを意味しています。

![image-a74ffaa0-b4ee-404e-9137-265962d04856.png]({{site.baseurl}}/media/2025/05/image-a74ffaa0-b4ee-404e-9137-265962d04856.png)

また、ログ送信元が異なるが故に、ログフォーマットがやや異なる点も留意する必要があります。

`Language Worker → Functions Host → Application Insights`

![image-74e00ed3-545f-48d3-a794-32bb8ceb8755.png]({{site.baseurl}}/media/2025/05/image-74e00ed3-545f-48d3-a794-32bb8ceb8755.png)

`Language Worker → Application Insights`

![image-72c790ce-3337-485d-90af-95e48c8d7df9.png]({{site.baseurl}}/media/2025/05/image-72c790ce-3337-485d-90af-95e48c8d7df9.png)

---

---

# よくある質問

## Language Worker → Application Insights ログ送信パイプライン利用時

Q. `host.json` でサンプリング設定を無効化したのに、アプリケーション内で出力したログがサンプリングされています。

A.
アプリケーション内で出力したログを Application Insights に送信する際にどの程度サンプリングするかについて、`Language Worker → Application Insights` ログ送信パイプラインの場合は、host.json ではなく Program.cs で設定します。
また、Application Insights ではアダプティブサンプリングが既定で有効となっていますので、Program.cs 内で明示的に無効化しないと、サンプリングされることがあります。
下記を Program.cs に記載することで、アプリケーション内で出力したログがサンプリングされないようにできます。
```csharp
services.AddApplicationInsightsTelemetryWorkerService(options =>
        {
            options.EnableAdaptiveSampling = false; //アダプティブサンプリングを無効化
        });
services.ConfigureFunctionsApplicationInsights();
```

参考：
[アダプティブ サンプリングを無効にする](https://learn.microsoft.com/ja-jp/previous-versions/azure/azure-monitor/app/sampling-classic-api#turning-off-adaptive-sampling)



## Language Worker → Functions Host → Application Insights ログ送信パイプライン利用時
Q. host.json で出力する logLevel を Trace 以上としているのに、アプリケーション内で出力した Debug や Trace レベルのログが Application Insights に記録されません。
<br>
<br>
A. Program.cs で builder を作成すると、logLevel の規定値が Information となっているため、Debug や Trace のログが Language Worker で出力されません。
それ故、Functions Host にも到達していないので、Application Insights にアプリケーション内で出力した Debug や Trace レベルのログが出力されません。
下記を Program.cs に追記することで、アプリケーション内で出力した Debug や Trace レベルのログを Application Insights に記録できます。
```
builder.Logging.SetMinimumLevel(LogLevel.Trace);
```

なお、Program.cs に上記の設定は入れているが、host.json で logLevel を Trace 以上と設定していない場合 (host.json の logLevel の既定値は Information です) は、Language Worker で出力した Debug や Trace レベルのログが、Functions Host のログ設定によりフィルタリングされ、Application Insights に記録されませんのでこの点もご注意ください。
下記は、host.json で logLevel を Trace 以上と設定している例です。
```
{
    "version": "2.0",
    "logging": {
        "logLevel": {
            "default": "Trace"
        }
    }
}
```


## 参考リンク

* [ILogger ログの記録（Worker Service）](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/worker-service#ilogger-logs)
* [.NET Isolated プロセスガイド](https://learn.microsoft.com/ja-jp/azure/azure-functions/dotnet-isolated-process-guide?tabs=hostbuilder%2Cwindows#application-insights)
* [Azure Functions 監視の構成](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-monitoring?tabs=v2)


<br>
<br>

2025 年 05 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>