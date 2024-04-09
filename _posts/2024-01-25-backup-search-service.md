---
title: "AI Search の設定やインデックスデータをバックアップしたい"
author_name: "Takumi Nagaya"
tags:
    - "AI Search"
---

# 質問
AI Search では作成後にレベルの変更 (Basic から Standard 等) ができない認識です。  
レベルの変更等に柔軟に対応できるようにするため、AI Search の設定やインデックスデータをバックアップ/リストアする方法はありますでしょうか。

# 回答
AI Search の設定とインデックスデータそれぞれのバックアップ/リストア方法をご案内いたします。

## AI Search の設定のバックアップ/リストア方法
AI Search の設定については、以下の 2 つでそれぞれ方法が異なります。

1. AI Search サービス
1. AI Search サービス内のインデックス、スキルセット、データソース、インデクサー等の設定

1 と 2 の違いとしては、1 の AI Search サービスは設定を Azure Resource Manager (ARM) という管理レイヤーを利用するのに対して、  
2 のインデックス等は、ARM を介さず AI Search サービスに対して管理リクエストを実行するという点が異なります。
<br/><br/>
バックアップ/リストア方法について、以下に記載します。

### 1. AI Search サービス自体のバックアップ/リストア方法
AI Search サービス自体は ARM で設定を管理できるため、ARM テンプレートとパラメータを保持しておけば、ARM テンプレートとパラメータを利用して同様のリソースを作成できます。<br/>

既存の AI Search サービスリソースにて、ARM テンプレートをエクスポートすることは可能ですが、以下のドキュメントに記載の通り、  
最初から ARM テンプレート (もしくは Bicep ファイル、terraform) を利用してリソースをデプロイいただけますと幸いです。  
>エクスポートが成功するという保証はありません。 <br/>
>エクスポートは、既存のリソースを運用環境で使用できるテンプレートに変換するための信頼性の高い方法ではありません。<br/>手書きの Bicep ファイル、ARM テンプレート、または terraform を使用して、リソースを最初から作成することをお勧めします。

[制限事項](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/export-template-portal#limitations)

つまり、ARM テンプレートの構成ファイルが、設定のバックアップとなり、同様のリソースを作成 (リストア) する際に利用できるとご認識いただけますと幸いです。

以下のドキュメントにチュートリアルがございますので、参考になれば幸いです。

- [クイック スタート: Azure Resource Manager テンプレートを使用して Azure AI Search をデプロイする](https://learn.microsoft.com/ja-jp/azure/search/search-get-started-arm)  
- [ARM テンプレートと Azure PowerShell を使用したリソースのデプロイ](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deploy-powershell)

以下では実際に AI Search サービスを ARM テンプレートを利用してデプロイします。

#### 1. ARM テンプレートを作成します。
[テンプレートを確認する](https://learn.microsoft.com/ja-jp/azure/search/search-get-started-arm#review-the-template) に記載のサンプル JSON もしくは、[AI Search ARM テンプレート定義](https://learn.microsoft.com/ja-jp/azure/templates/microsoft.search/searchservices?pivots=deployment-language-arm-template) を元にファイルを編集いただき `azuredeploy.json` としてローカルに保存します。<br/>
今回は [テンプレートを確認する](https://learn.microsoft.com/ja-jp/azure/search/search-get-started-arm#review-the-template) に記載のサンプル JSON をそのまま使用します。

#### 2. 1 で作成した ARM テンプレートで使用するパラメータファイルを作成します。`parameters.json` として保存します。
[パラメーター ファイル](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/parameter-files#parameter-file) 参考に、以下のようにします。

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "name": {
        "value": "blog-search-sample"
      },
      "sku": {
        "value": "basic"
      },
      "replicaCount": {
        "value": 1
      },
      "partitionCount": {
        "value": 1
      },
      "hostingMode": {
        "value": "default"
      },
      "location": {
        "value": "japaneast"
      }
    }
  }
```

#### 3. PowerShell でリソースグループを作成し、テンプレートをデプロイします。
- [ARM テンプレートと Azure PowerShell を使用したリソースのデプロイ](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deploy-powershell) の手順が参考になれば幸いです。

- リソースグループを作成します。

```powershell
$rgName = "Blogsample"
$location = "Japan East"
New-AzResourceGroup -Name $rgName -Location $location
```

- デプロイ名を設定します。

```powershell
$today=Get-Date -Format "MM-dd-yyyy"
$deploymentName="ExampleDeployment"+"$today"
```

- テンプレートを指定してデプロイします。

```powershell
New-AzResourceGroupDeployment `
  -Name $deploymentName `
  -ResourceGroupName $rgName `
  -TemplateFile <path\to\azuredeploy.json> `
  -TemplateParameterFile <path\to\parameters.json>
```

以上の手順で AI Search サービスを ARM テンプレートから作成することができました。

### 2. AI Search サービス内のインデックス等設定のバックアップ/リストア方法
インデックス等も、AI Search サービスを ARM テンプレートで管理するのと同様に、REST API の要求本文を保持しておくか、SDK でリソース作成のコードを記述し、リストア用に保持しておく形になります。

例えばインデックスの場合、以下のクイックスタートを参考に、REST API の実行環境を整え、インデックスを作成します。  
クイックスタートの手順 [インデックスの定義](https://learn.microsoft.com/ja-jp/azure/search/search-get-started-rest#index-definition) で作成した要求本文があれば、今後インデックスを再作成やリストアする場合にそのままご利用可能となります。

- [クイック スタート: REST API を使用して Azure AI Search インデックスを作成する](https://learn.microsoft.com/ja-jp/azure/search/search-get-started-rest)

他にもインデクサー、データソース、スキルセットなどの作成/更新も REST API で提供されておりますので、  
同様にご利用いただけますと幸いです。

- [インデックスの作成 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-index)
- [インデクサーの作成 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-indexer)
- [データ ソースの作成 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-data-source)
- [スキルセットの作成 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-skillset)

お客様によっては、プレビューの REST API をご利用の場合もあると存じます。その場合は [Search Service REST API のプレビュー](https://learn.microsoft.com/ja-jp/rest/api/searchservice/index-preview) より、プレビューの API 仕様をご確認いただけますと幸いです。  
また、[検索 REST API の API バージョン](https://learn.microsoft.com/ja-jp/rest/api/searchservice/search-service-api-versions) もご確認いただき、どの API バージョンを利用するか検討いただけますと幸いです。<br/><br/>

また、Azure Portal のインポートウィザードから自動的に各リソースを作成いただく場合もあると存じます。  
その場合は、以下の手順で REST API 経由でリソースを管理するようにしていただければ、今後同様のリソースをすぐに作成することができます。  
※以下はインデックスでの例ですが、インデクサー、データソース等も同様に実施いただけますと幸いです。  
※ただし、データソースの場合、接続文字列等の Blob Storage や SQL Database 等にアクセスするシークレット情報は、Azure Portal から取得すると削除されている場合がございます。接続文字列等が正しいか念のためご確認いただけますと幸いです。

#### 1. Azure Portal で既存のインデックス定義の JSON を取得します。

![image-a7b64ac5-a0a1-4971-b83a-7da52970b429.png]({{site.baseurl}}/media/2024/01/image-a7b64ac5-a0a1-4971-b83a-7da52970b429.png)

#### 2. JSON 内の以下の不要なプロパティを削除します。

```
  "@odata.context": "https://<search service name>.search.windows.net/$metadata#indexes/$entity",
  "@odata.etag": "\"0x8DC10B88B43FA8D\"",
```

#### 3. [インデックスの作成 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-index) または [インデックスの更新 (Azure AI Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/update-index)を元に Postman でリクエストを設定します。要求本文に 2 で作成した JSON を設定します。

![image-516b7473-4fbf-4dad-9a90-765105bfc8b1.png]({{site.baseurl}}/media/2024/01/image-516b7473-4fbf-4dad-9a90-765105bfc8b1.png)

#### 4. API を実行し、エラーがなければ正常に作成、更新できた形になります。これで仮にインデックスが削除されても再度この要求を実施すれば同じインデックスが作成されます。

## インデックスデータのバックアップ/リストア方法
恐れ入りますが、AI Search の機能としてインデックスデータのバックアップ/リストアする方法はございません。  
以下のドキュメントにも記載がございます。

>インデックス管理のネイティブ サポートはありません。

[インデックスの移動、バックアップ、復元はできますか?](https://learn.microsoft.com/ja-jp/azure/search/search-faq-frequently-asked-questions#--------------------------)

AI Search の機能としてインデックスデータのバックアップ/リストアする方法がないのは、そもそも Blob Storage や SQL Database 等に保持されたデータを元に AI Search のインデックスデータが作成されるため、仮にインデックスデータを復元する必要がある場合は、AI Search のインデックス、インデクサー、データソース、スキルセットを再構成すれば Blob Storage や SQL Database 等からインデックスデータを再構成できるためです。<br/>
<br/>
したがって、基本的には Blob Storage や SQL Database 等に保持されたデータがあれば、AI Search のインデックス、インデクサー、データソース、スキルセットを再構成することで、インデックスデータが復元できるため、インデックスデータを個別にバックアップいただく必要はないと認識いただけますと幸いです。<br/>
<br/>
もちろん、お客様の状況によってはインデックスデータをバックアップする必要があると存じます。  
その場合は、以下の FAQ に記載のサンプルコードを試していただければと存じます。  
**※あくまでサンプルコードでございますので、お客様にてコードの内容を確認し動作検証いただいた上でご利用いただけますと幸いです。**  

>ただし、検索サービス間でインデックスを移動する場合は、この [Azure AI Search .NET サンプル リポジトリ](https://github.com/Azure-Samples/azure-search-dotnet-utilities)にある index-backup-restore サンプル コードを試すことができます。  

[インデックスの移動、バックアップ、復元はできますか?](https://learn.microsoft.com/ja-jp/azure/search/search-faq-frequently-asked-questions#--------------------------)

### サンプルコードの実行手順
[Run the sample](https://github.com/Azure-Samples/azure-search-dotnet-utilities/tree/main/index-backup-restore#run-the-sample) に詳細の記載がありますが、簡単に手順を記載いたします。  
また、**サンプルコードには、10万以上のレコードはバックアップできないなど、重要な記載がございますので、必ず [IMPORTANT - PLEASE READ](https://github.com/Azure-Samples/azure-search-dotnet-utilities/blob/main/index-backup-restore/README.md#important---please-read) を事前にご確認いただけますと幸いです。**


#### 1. GitHub からサンプルコードを開いて解凍し、`index-backup-restore/v11` にあるプロジェクトを Visual Studio で開きます。
#### 2. プロジェクト内のファイル `appsettings.json` を編集します。

```json
{
  "SourceSearchServiceName": "<バックアップする Search サービス名>",
  "SourceAdminKey": "<バックアップする Search サービスの API キー>",
  "SourceIndexName": "<バックアップするインデックス名>",
  "TargetSearchServiceName": "<リストアする Search サービス名>",
  "TargetAdminKey": "<リストアする Search サービスの API キー>",
  "TargetIndexName": "<リストアするインデックス名>",
  "BackupDirectory": "<バックアップ処理で生成されるファイルを保存するディレクトリ>"
}
```

#### 3. プロジェクトをビルドして、実行します。

実行時に以下のようなエラーが発生する場合があります。  

~~~
Search request failed: {"error":{"code":"FeatureNotSupportedInApiVersion","message":"This index was created with a newer version of the Azure Search API and uses features exclusive to that version. Please use the latest API version (2023-10-01-Preview) to manage this index."
~~~

これは [AzureSearchBackupRestoreIndex/AzureSearchHelper.cs](https://github.com/Azure-Samples/azure-search-dotnet-utilities/blob/main/index-backup-restore/v11/AzureSearchBackupRestoreIndex/AzureSearchHelper.cs#L25) で設定されている、AI Search サービスの REST API バージョンの値が古いことによるもので、  
以下のようにエラーメッセージに含まれる REST API のバージョン (例: 2023-10-01-Preview) に修正することで解消する可能性があります。
```cs
        public const string ApiVersionString = "api-version=2023-10-01-Preview";
```

#### 4. 完了するとバックアップ元とリストア先のインデックスのインデックス内のドキュメント数が以下のように表示されます。バックアップ元のインデックスのドキュメント数 36 と同数のドキュメントがリストアされたことが確認できます。

~~~
SAFEGUARD CHECK: Source and target index counts should match
 Source index contains 36 docs
 Target index contains 36 docs
~~~

---

<br>
<br>

2024 年 04 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>