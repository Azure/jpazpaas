---
title: "スロットを使用した Azure Functions のインプロセスモデル から 分離ワーカープロセスモデル への移行手順について"
author_name: "Kohei Mayama"
tags:
    - Function App
---

# はじめに
お世話になっております。App Service サポート担当の間山です。

2026 年 11 月 10 日より、Azure Functions .NET アプリケーションの インプロセスモデル がサポート終了となります。インプロセスモデルにてお使いの .NET の関数コード アプリケーションを引き続き製品サポートを受ける場合には、分離ワーカープロセスモデル への変更をする必要がございます。

Azure Functions .NET アプリケーションの インプロセスモデル がサポート終了の詳細な情報につきましては、以下の弊社サポートブログにて補足をしております。

[Azure Functions インプロセス モデル のサポート終了について(追跡 ID FVN7-7PZ)](https://azure.github.io/jpazpaas/2024/04/01/azure-functions-inprocess-end-of-support-FVN7-7PZ.html)


弊社提供の公開情報では、インプロセスモデル から 分離ワーカープロセスモデル への移行手順について公開しております。商用環境へ直接アプリケーションをデプロイした際に、何かしらの問題が発生したり、再起動にリクエストが送信され正常に関数コードの処理がされない可能性があるため、ステージングスロットを利用することをご案内しております。


[Azure で関数アプリを更新する](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#update-your-function-app-in-azure)

> ダウンタイムを最小限に抑える必要がある場合は、ステージング スロットを使って、Azure で更新した構成を使って、移行したコードをテストおよび検証することを検討してください。 その後、スワップ操作を通じて、完全に移行したアプリを運用スロットにデプロイできます。

本ブログ記事では、公開情報の補足情報として スロットを使用した Azure Functions のインプロセスモデル から 分離ワーカープロセスモデル への移行手順についてご案内をしていきます。なお、今回は .NET 8 インプロセスモデルから .NET 8 分離ワーカープロセスモデルへの変更となるため、.NET ランタイムバージョンはそのままとなります。


# スロットを使用した Azure Functions のインプロセスモデル から 分離ワーカープロセスモデル への移行手順

## 【手順 0 】インプロセスモデルの関数コードアプリケーションの実装と関数アプリリソースの用意

Visual Studio にて関数コードプロジェクト作成時に .NET 8（長期的なサポート）を指定することで、.NET 8 インプロセスを選択されます。

![image-e5502e14-405d-48b0-85c2-5e9f2569c4d2.png]({{site.baseurl}}/media/2024/08/image-e5502e14-405d-48b0-85c2-5e9f2569c4d2.png)

作成されたプロジェクトは Function1.cs ファイルに HTTP トリガー関数コードの記載がされております。

![image-ebf29c8d-7ceb-4605-894e-51d073fd493a.png]({{site.baseurl}}/media/2024/08/image-ebf29c8d-7ceb-4605-894e-51d073fd493a.png)

F5 ボタンを押下して、関数コードプロジェクトをビルド、関数コードを実行します。

![image-055e5ce4-f550-4bc7-916d-965bcb774ab5.png]({{site.baseurl}}/media/2024/08/image-055e5ce4-f550-4bc7-916d-965bcb774ab5.png)

Visual Studio で動作確認した関数コードをデプロイするために、.NET 8 インプロセスモデルの関数アプリのリソースを作成します。

![image-36ecb0d1-8f21-4f79-99ed-59373c6b053a.png]({{site.baseurl}}/media/2024/08/image-36ecb0d1-8f21-4f79-99ed-59373c6b053a.png)

関数コードプロジェクトの右ブレードメニュー上部を右クリックをして、発行 ボタンを押下しデプロイ対象の関数アプリリソースを選択します。関数コードプロジェクトをデプロイする際には 公開 > 発行 ボタンを押下します。

![image-75f4faac-0705-4e5b-bbbd-91a88a76e296.png]({{site.baseurl}}/media/2024/08/image-75f4faac-0705-4e5b-bbbd-91a88a76e296.png)

デプロイされた関数コードが正常に実行されるかを簡易的に検証をします。関数アプリ左メニューブレード の 概要 > 対象関数コード名 > コードとテスト > テスト/実行 より HTTP リクエストを送信して正常に HTTP レスポンスが返却されるか確認します。実際の検証を行う場合には、Azure ポータルのテスト画面ではなく、ブラウザ等のクライアントから直接 HTTP エンドポイント（~.azurewebsites.net）に対して接続を行います。

![image-5e470560-1bb7-4da3-bc6e-0804a322cdf9.png]({{site.baseurl}}/media/2024/08/image-5e470560-1bb7-4da3-bc6e-0804a322cdf9.png)


## 【手順 1 】検証環境用のデプロイスロットを作成

検証環境用としてデプロイスロットの作成を行います。関数アプリ左メニューブレードの デプロイスロット > 追加 ボタンを押下し、デプロイスロット名を入力し作成を行います。

![image-c941da3c-b443-4d50-a719-14c53cb1d03c.png]({{site.baseurl}}/media/2024/08/image-c941da3c-b443-4d50-a719-14c53cb1d03c.png)

Visual Studio からデプロイするとデフォルトの場合には zip デプロイ と WEBSITE_RUN_FROM_PACKAGE になります。スロットを作成するとWEBSITE_RUN_FROM_PACKAGE が有効になり対象の zip ファイルがないため正常に起動できない状態となります。
そのため、再度 Visual Studio からデプロイスロット環境に対して関数コードプロジェクトをデプロイします。
その後、デプロイスロット環境に対しても正常に関数コードが実行されているか検証するためにクライアント等から HTTP リクエストを送信します。

![image-82a02e3c-b953-43cf-a2b6-aee22b4d240c.png]({{site.baseurl}}/media/2024/08/image-82a02e3c-b953-43cf-a2b6-aee22b4d240c.png)


## 【手順 2 】Visual Studio で作成した関数コードプロジェクトを .NET 8 分離ワーカープロセスモデル へ移行

公開情報に従い、Visual Studio で作成したローカル環境の関数コードプロジェクトを .NET 8 分離ワーカープロセスモデル へ移行を実施します。

[ローカル プロジェクトを移行する](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#migrate-your-local-project)

### 2-1. csproj ファイル の変更
csproj ファイルに対して、以下の部分の更新を行います。

1） `<OutputType>Exe</OutputType>` を PropertyGroup に追加します。

2） ItemGroup 内の PackageReference リストで、Microsoft.NET.Sdk.Functions のパッケージ参照を以下へ参照に置き換えます。

```
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.21.0" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.2" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="1.2.1" />
  <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="1.2.0" />
```

3） 新しい ItemGroup として以下の項目を追加します。

```
<ItemGroup>
  <Using Include="System.Threading.ExecutionContext" Alias="ExecutionContext"/>
</ItemGroup>
```

最終的に更新された csproj ファイルは以下の通りとなりました。

```
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <RootNamespace>function_migration_inprocess_isolated</RootNamespace>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
	<FrameworkReference Include="Microsoft.AspNetCore.App" />
	<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.21.0" />
	<PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.2" />
	<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="1.2.1" />
	<PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
	<PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="1.2.0" />
  </ItemGroup>
  <ItemGroup>
    <None Update="host.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Update="local.settings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <CopyToPublishDirectory>Never</CopyToPublishDirectory>
    </None>
  </ItemGroup>
  <ItemGroup>
	  <Using Include="System.Threading.ExecutionContext" Alias="ExecutionContext"/>
  </ItemGroup>
</Project>
```

また、トリガーやバインドを使用している場合には、アプリケーションが参照するパッケージを変更する必要があります。今回は HTTP トリガーのみであるため省略とします。
[パッケージ参照](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#package-references)


### 2-2. Program.cs ファイル の追加

分離ワーカー プロセスで実行できるように、Program.cs ファイルを関数コードプロジェクトへ追加し以下の記載をします。

[Program.cs ファイル](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#programcs-file)

```
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services => {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
    })
    .Build();

host.Run();
```


### 2-3. Function1.cs ファイル の修正

Function1.cs ファイルを分離ワーカープロセスモデルへ移行を実施します。

1）ILogger メソッドの追加

分離ワーカープロセスモデルでは、ILogger メソッド パラメーターに依存した関数の場合は、変更を行う必要があります。
インプロセスモデルでは　ILogger クラスが存在しないため Function1.cs ファイルで定義します。静的クラスにはコンストラクタを定義できないため、public static class <クラス名> から public class <クラス名> へ変更します。

[Logging](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#logging)

```
    public class Function1
    {
        private readonly ILogger<Function1> _logger;

        public Function1(ILogger<Function1> logger)
        {
            _logger = logger;
        }
　　（中略）・・・

　　_logger.LogInformation("C# HTTP trigger function processed a request.");
    }
```

2）Azure Functions で使用する名前空間を変更

分離ワーカープロセスモデルでは、Microsoft.Azure.WebJobs や Microsoft.Azure.WebJobs.Extensions.Http ではなく、Microsoft.Azure.Functions.Worker へ変更となります。
コードの定義を以下へ変更する必要があります。

```
using Microsoft.Azure.Functions.Worker;
（中略）

[Function("Function1")]
```

3）JSON シリアル化の変更

インプロセスモデルの関数コードでは、Newtonsoft.Json が使用されており、.NET Core 3.1 以降では System.Text.Json を使用することができます。
System.Text.Json を使用する場合には、コードの定義を以下へ変更する必要があります

[JSON シリアル化](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#json-serialization)

```
using System.Text.Json;

（中略）
            dynamic data = null;

            if (!string.IsNullOrEmpty(requestBody))
            {
                data = JsonSerializer.Deserialize<dynamic>(requestBody);
                name = name ?? data?.name;
            }
```

最終的に更新された Function1.cs は以下の通りとなります。
さらに、レスポンスを返却する際に、(.NET 8 isolated) 文字列を追加し .NET 8 の分離ワーカープロセスモデルで動作しているメッセージを出力します。

```
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using System.Text.Json;
using Microsoft.Azure.Functions.Worker;

namespace function_migration_inprocess_isolated
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;

        public Function1(ILogger<Function1> logger)
        {
            _logger = logger;
        }

        [Function("Function1")]
        public async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");

            string name = req.Query["name"];

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = null;

            if (!string.IsNullOrEmpty(requestBody))
            {
                data = JsonSerializer.Deserialize<dynamic>(requestBody);
                name = name ?? data?.name;
            }

            string responseMessage = string.IsNullOrEmpty(name)
                ? "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response. (.NET 8 isolated)"
                : $"Hello, {name}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }
    }
}
```

4）local.settings.json ファイルの変更

ローカル環境で関数コードを実行する際には 分離ワーカープロセスモデル で実行するために、local.settings.json ファイル の FUNCTIONS_WORKER_RUNTIME プロパティ値を "dotnet-isolated" へ変更します。

[local.settings.json ファイル](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#localsettingsjson-file)

```
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
```


## 【手順 3 】Visual Studio で修正した .NET 8 分離ワーカープロセスモデル の関数コードプロジェクトの実行確認

関数コードプロジェクトのビルド前に、ソリューションのクリーンを行いインプロセスモデルで使用していた名前空間の整理を行います。
その後、F5 ボタンを押下しビルドと関数コードの実行を行います。

![image-a88286bc-2e8a-4bf0-97f3-27c3ed2f843d.png]({{site.baseurl}}/media/2024/08/image-a88286bc-2e8a-4bf0-97f3-27c3ed2f843d.png)

![image-fa37fed6-ef9d-404d-b711-88de0d6ad22d.png]({{site.baseurl}}/media/2024/08/image-fa37fed6-ef9d-404d-b711-88de0d6ad22d.png)

ブラウザから関数コードを実行するとレスポンスが返却され、(.NET 8 isolated) メッセージが追加されております。

![image-15caf208-eda5-4b7c-80ec-a0adca03cd91.png]({{site.baseurl}}/media/2024/08/image-15caf208-eda5-4b7c-80ec-a0adca03cd91.png)


## 【手順 4 】.NET 8 分離ワーカープロセスモデル の関数コードプロジェクトを 検証環境用のデプロイスロットへデプロイ

関数アプリのデプロイスロット環境（Staging 環境）で、分離ワーカープロセスモデルを実行するために、設定 > 環境変数 > アプリ設定 から FUNCTIONS_WORKER_RUNTIME の値を `dotnet-isolated` へ変更します。
さらに、分離ワーカープロセスモデルを実行するために、インプロセスモデルで使用していたフラグであるアプリケーション設定 `FUNCTIONS_INPROC_NET8_ENABLED` を削除します。

![image-f33c6873-456a-4211-ac40-fec30fdbf89a.png]({{site.baseurl}}/media/2024/08/image-f33c6873-456a-4211-ac40-fec30fdbf89a.png)

Visual Studio から関数アプリのデプロイスロット環境（Staging 環境）に対して関数コードプロジェクトをデプロイします。
Visual Studio からアプリをデプロイする場合、以下のメッセージが表示されますが、ここでは はい(Y) を選択しデプロイを行います。

![image-bf442e82-c587-468b-9ba8-b56ae7ace6e6.png]({{site.baseurl}}/media/2024/08/image-bf442e82-c587-468b-9ba8-b56ae7ace6e6.png)

関数アプリのデプロイスロット環境（Staging 環境）へデプロイされた分離ワーカープロセスモデルの関数コードが正常に実行されるかを簡易的に検証をします。関数アプリ左メニューブレード の 概要 > 対象関数コード名 > コードとテスト > テスト/実行 より HTTP リクエストを送信して正常に HTTP レスポンスが返却されるか確認します。実際の検証を行う場合には、Azure ポータルのテスト画面ではなく、ブラウザ等のクライアントから直接 HTTP エンドポイント（~.azurewebsites.net）に対して接続を行います。

レスポンスが返却され、(.NET 8 isolated) メッセージが追加されているため正常に動作していることを確認できました。

![image-b6299181-a093-4325-beb0-e04527dee92d.png]({{site.baseurl}}/media/2024/08/image-b6299181-a093-4325-beb0-e04527dee92d.png)

ブラウザから関数アプリのデプロイスロット環境（Staging 環境）の関数コードを実行するとレスポンスが返却され、(.NET 8 isolated) メッセージが追加されております。

![image-95a57edb-e1e3-4a35-9203-2dc1e6973f6b.png]({{site.baseurl}}/media/2024/08/image-95a57edb-e1e3-4a35-9203-2dc1e6973f6b.png)


## 【手順 5 】stage スロットを production スロットとスワップ

関数アプリのデプロイスロット環境（Staging 環境）と production スロットをスワップします。
デプロイメント > デプロイスロット > Swap （スワップ）から実行します。
ソーススロット（staging）から ターゲットスロット（Production）に対して、分離ワーカープロセスモデルの関数コードプロジェクトと FUNCTIONS_WORKER_RUNTIME の値 `dotnet-isolated` が反映されます。

![image-b3f26bb9-313f-4349-8a28-2f1f4e62badc.png]({{site.baseurl}}/media/2024/08/image-b3f26bb9-313f-4349-8a28-2f1f4e62badc.png)

ブラウザから実行結果を確認すると、関数アプリのデプロイスロット環境（Production 環境）にて正常に分離ワーカープロセスモデルの関数コードが実行されていることが確認できました。
一方で、関数アプリのデプロイスロット環境（staging 環境）は、更新前のインプロセスモデルの関数コードが実行されていることが確認することができました。

**関数アプリのデプロイスロット環境（Production 環境）**
![image-47ff62d0-909e-4345-9497-760822513b84.png]({{site.baseurl}}/media/2024/08/image-47ff62d0-909e-4345-9497-760822513b84.png)

**関数アプリのデプロイスロット環境（staging 環境）**
![image-7d807e74-18d4-4683-b08d-ad1dc5fa9c5b.png]({{site.baseurl}}/media/2024/08/image-7d807e74-18d4-4683-b08d-ad1dc5fa9c5b.png)



# 参考公開情報

[.NET アプリをインプロセス モデルから分離ワーカー モデルに移行する](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8)


<br>
<br>

---

<br>
<br>

2024 年 08 月 30 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>