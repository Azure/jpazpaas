---
title: "AI Search で OCR 等の組み込みスキルを Azure AI Service でパブリックアクセスを無効化した状態で実行したい"
author_name: "Takumi Nagaya"
tags:
    - AI Search
---

# 質問
AI Search のスキルセットで、AI マルチサービス リソースの OCR スキルを設定しています。  
しかし、AI マルチサービス リソースのパブリックネットワークアクセスを無効にするとエラーが発生しました。回避する方法はありますか?

# 回答
恐れ入りますが、現時点で AI マルチサービス リソースの OCR 等の組み込みスキルは AI マルチサービス リソースのパブリックネットワークアクセスを無効にして利用することはかないません。<br/>
以下のドキュメントに記載の通り、AI マルチサービス リソースのパブリックネットワークを無効にする場合は、Azure Function で AI マルチサービス リソースの API を実行する処理をお客様にて実装いただき、プライベート環境を構築いただく必要がございます。

>現時点では、 組み込みのスキル の課金には、Azure AI Search から別の Azure AI サービスへのパブリック接続が必要です。 公衆ネットワークへのアクセスを無効にすると、課金ができなくなります。 パブリック ネットワークを無効にする必要がある場合は、 プライベート エンドポイント をサポートする Azure Functionを実装したカスタム Web API スキルを構成し、 同じ VNETに Azure AI サービス リソースを追加します。 この方法では、プライベート エンドポイントを使用して、カスタム スキルから直接 Azure AI サービス リソースを呼び出すことができます。

[Azure AI マルチサービス リソースを Azure AI Search のスキルセットにアタッチする](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-attach-cognitive-services?tabs=portal%2Cportal-remove#how-the-key-is-used)

構成としては以下になります。<br/>
![image-e33fa8c0-4664-444e-843d-723147966ca3.png]({{site.baseurl}}/media/2024/08/image-e33fa8c0-4664-444e-843d-723147966ca3.png)

また、AI Search の共有プライベートリンクおよびスキルセットを利用して、AI マルチサービス リソースにプライベートにアクセスするため、共有プライベートリンクの前提条件から、Standard 2 以上のレベルの AI Search が必要となりますのでご注意ください。
> AI エンリッチメントとスキルセットを使用している場合、レベルは Standard 2 (S2) 以上である必要があります。

[前提条件](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#prerequisites)

以下に設定の手順を記載いたします。

## 手順
### AI Service の組み込みスキルに対応する API の特定
AI Search の AI Service 組み込みスキルのドキュメントには、Azure AI Service のどの API を使用しているかの記載があります。<br/>
お客様にて Azure Function のコードを実装する上で、お客様ご利用の AI Service 組み込みスキルに対応した AI Service の API をあらかじめ特定いただく必要があります。<br/>
例えば OCR スキルの場合、以下になります。

>OCR スキルでは、Azure AI サービスの Azure AI Vision API v3.2 によって提供される機械学習モデルが使用されます。 OCR スキルは、次の機能にマップします。
>- 「Azure AI Vision の言語サポート」に記載されている言語については、Read API が使用されます。
>- ギリシャ語とセルビア語のキリル語では、バージョン 3.2 API のレガシ OCR が使用されます。

[OCR 認知スキル](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-skill-ocr)

AI Service のどの API が使用されいているか特定できたら、AI Service のドキュメントから開発するにあたり必要な情報を確認します。<br/>
OCR スキルの利用する Azure AI Vision API v3.2 の Read API であれば、以下のクイックスタートが参考になります。

[クイック スタート: Azure AI Vision v3.2 GA Read](https://learn.microsoft.com/ja-jp/azure/ai-services/computer-vision/quickstarts-sdk/client-library?tabs=linux%2Cvisual-studio&pivots=programming-language-csharp)

### カスタムスキルの開発
呼び出す AI Service の API の情報がそろったら、カスタムスキルの開発を進めます。<br/><br/>
カスタムスキルの概要については、ドキュメント [Azure AI Search エンリッチメント パイプラインにカスタム スキルを追加する](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-custom-skill-interface) をご確認ください。<br/>
このブログでは、以下を参考に OCR スキルに対応する Azure AI Vision API v3.2 の Read API を呼び出すカスタムスキルを実際に作成していきます。あくまでサンプルでございますので、お客様の開発環境、要件に応じて適宜読み替えていただけますと幸いです。
- [例: Bing Entity Search API を使用してカスタム スキルを作成する](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example)
- [クイック スタート: Azure AI Vision v3.2 GA Read](https://learn.microsoft.com/ja-jp/azure/ai-services/computer-vision/quickstarts-sdk/client-library?tabs=linux%2Cvisual-studio&pivots=programming-language-csharp)
- [Azure AI Search Power Skills](https://github.com/Azure-Samples/azure-search-power-skills)
  - [TextAnalyticsForHealth](https://github.com/Azure-Samples/azure-search-power-skills/blob/main/Text/TextAnalyticsForHealth/README.md)

#### Visual Studio でのプロジェクトの作成と実装
- プロジェクトを[ドキュメント](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#create-a-project) 参考に作成します。
- サンプルの `WebApiSkillHelpers` クラスを利用するため、[Azure AI Search Power Skills](https://github.com/Azure-Samples/azure-search-power-skills) の Common プロジェクトを参照に追加します。
- AI Vision の[クイックスタートの手順](https://learn.microsoft.com/ja-jp/azure/ai-services/computer-vision/quickstarts-sdk/client-library?tabs=linux%2Cvisual-studio&pivots=programming-language-csharp#read-printed-and-handwritten-text) を参考に NuGet パッケージ `Microsoft.Azure.CognitiveServices.Vision.ComputerVision` を追加します。
- あくまでサンプルになりますが、本ブログのために用意したコードを記載します。ポイントは、以下になります。
  - AI Search のスキルセットで指定する AI Service のマルチサービスリソースのキーとエンドポイントを環境変数 `VISION_KEY` と `VISION_ENDPOINT` として設定します。
  - AI Search の画像抽出で取得された画像が Base64 エンコードされた状態で `image` プロパティに格納されることを想定しています。
    - Blob Storage 等にアップロードされ、URL を指定してデータを取得することはできないため、AI Vision のクイックスタートとは異なり [ReadInStreamAsync](https://learn.microsoft.com/en-us/dotnet/api/microsoft.azure.cognitiveservices.vision.computervision.computervisionclientextensions.readinstreamasync?view=azure-dotnet) を使用しています。

```cs
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using AzureCognitiveSearch.PowerSkills.Common;
using System.Collections.Generic;
using System.Threading;
using Microsoft.Azure.CognitiveServices.Vision.ComputerVision;
using Microsoft.Azure.CognitiveServices.Vision.ComputerVision.Models;
using System.Collections.Concurrent;

namespace customskillblogsample
{
    public static class Function1
    {
        // Add your Computer Vision key and endpoint
        static string key = Environment.GetEnvironmentVariable("VISION_KEY");
        static string endpoint = Environment.GetEnvironmentVariable("VISION_ENDPOINT");

        private static readonly int defaultTimeout = 230;
        private static readonly int maxTimeout = 230;
        private static readonly int timeoutBuffer = 5;

        [FunctionName("Function1")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");
            IEnumerable<WebApiRequestRecord> requestRecords = WebApiSkillHelpers.GetRequestRecords(req);

            if (requestRecords == null)
            {
                return new BadRequestObjectResult("Invalid request record array.");
            }

            // Get a custom timeout from the header, if it exists. If not use the default timeout.
            int timeout;
            if (!int.TryParse(req.Headers["timeout"].ToString(), out timeout))
            {
                timeout = defaultTimeout;
            }
            timeout = Math.Clamp(timeout - timeoutBuffer, 1, maxTimeout - timeoutBuffer);
            var timeoutMiliseconds = timeout * 1000;
            var timeoutTask = Task.Delay(timeoutMiliseconds);

            using (ComputerVisionClient client = Authenticate(endpoint, key))
            {
                // Create the response record
                WebApiSkillResponse response = await WebApiSkillHelpers.ProcessRequestRecordsAsync("skillName", requestRecords,
                async (inRecord, outRecord) =>
                {

                    if (timeoutTask.IsCompleted)
                    {
                        // The time limit for all the skills has been met
                        outRecord.Errors.Add(new WebApiErrorWarningContract
                        {
                            Message = "Error: The OCR Operation took too long to complete."
                        });
                        return outRecord;
                    }

                    await ReadFileUrl(inRecord, client, outRecord, log);
                    
                    return outRecord;
                });
                return new OkObjectResult(response);
            }
        }

        public static ComputerVisionClient Authenticate(string endpoint, string key)
        {
            ComputerVisionClient client =
              new ComputerVisionClient(new ApiKeyServiceClientCredentials(key))
              { Endpoint = endpoint };
            return client;
        }

        public static async Task ReadFileUrl(WebApiRequestRecord record, ComputerVisionClient client, WebApiResponseRecord outRecord, ILogger log)
        {
            WebApiResponseRecord waRecord = new WebApiResponseRecord();

            record.Data.TryGetValue("image", out object image);
            byte[] imagebytes = Convert.FromBase64String(image.ToString());

            // Read text from stream
            MemoryStream stream = new MemoryStream(imagebytes);
            var textHeaders = await client.ReadInStreamAsync(stream);
            
            // Close MemoryStream
            stream.Close();

            // After the request, get the operation location (operation ID)
            string operationLocation = textHeaders.OperationLocation;
            Thread.Sleep(2000);

            // Retrieve the URI where the extracted text will be stored from the Operation-Location header.
            // We only need the ID and not the full URL
            const int numberOfCharsInOperationId = 36;
            string operationId = operationLocation.Substring(operationLocation.Length - numberOfCharsInOperationId);

            // Extract the text
            ReadOperationResult results;
            log.LogInformation($"RecordId: {record.RecordId}, operationId: ${operationId}");
            do
            {
                results = await client.GetReadResultAsync(Guid.Parse(operationId));
            }
            while ((results.Status == OperationStatusCodes.Running ||
                results.Status == OperationStatusCodes.NotStarted));

            // Display the found text.
            var textFileResults = results.AnalyzeResult.ReadResults;
            List<string> lines = new List<String>();
            foreach (ReadResult page in textFileResults)
            {
                foreach (Line line in page.Lines)
                {
                    lines.Add(line.Text);
                }
            }

            outRecord.RecordId = record.RecordId;
            outRecord.Data.Add("text", string.Join(" ", lines));
        }
    }
}
```

#### ローカル環境でのデバッグ
[Visual Studio から Functions をテストする](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#test-the-function-from-visual-studio) を参考に、<br/>
OCR の対象の画像をローカル端末に用意し、以下のサンプル PowerShell スクリプトを実行します。

```powershell
$image1Path = "C:\path\to\image1.png"
$image2Path = "C:\path\to\image2.png"
$image1Base64 = [Convert]::ToBase64String((Get-Content -Path $image1Path -AsByteStream -Raw))
$image2Base64 = [Convert]::ToBase64String((get-content -Path $image2Path -AsByteStream -Raw))

$headers = @{
    'Content-Type' = 'application/json' 
}

$body = @"
{
    "values": [
        {
            "recordId": "e1",
            "data":
            {
                "image": "${image1Base64}"
            }
        },
        {
            "recordId": "e2",
            "data":
            {
                "image": "${image2Base64}"
            }
        }
    ]
}
"@

$url = "http://localhost:7230/api/Function1" # ローカルで Function を起動した際のエンドポイントを指定
Invoke-RestMethod -Uri $url -Headers $headers -Method Post -Body $body | ConvertTo-Json
```

以下のように `text` に元の画像に含まれる文字列が抽出されていれば成功です。
```
{
    "values": [
        {
            "recordId": "e2",
            "data": {
                "text": "Visual Studio から Functions をテストする F5 キーを押して、プログラムを実行 し、関数の動作をテストします。"
            },
            "errors": [],
            "warnings": []
        },
        {
            "recordId": "e1",
            "data": {
                "text": "blogocr ... Container º Search 0 Overview Diagnose and solve problems PQ"
            },
            "errors": [],
            "warnings": []
        }
    ]
}
```
### Azure Function リソースの準備
[Azure に関数を発行する](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#publish-the-function-to-azure) を参考に Azure Function リソースを作成し、コードをデプロイします。<br/>
<br/>
ただし、この後プライベートアクセスとするため、以下の前提と制約を考慮すると、<br/>
**作成する Azure Function では「Premiumプラン」のホスティングモデルを選択いただく形になります。**

- Azure Function 観点では以下の前提が必要となります。参考: [Azure Functions のネットワーク オプション](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-networking-options?tabs=azure-portal)
  - プライベートエンドポイント経由で AI Search からアクセスされること、つまり「受信プライベートエンドポイント」がサポートされていること。
  - Azure Function から仮想ネットワーク経由で、AI Service のプライベートエンドポイントにアクセスできること。
- AI Search 観点では以下の前提があります。
  - 共有プライベートリンクで Azure Function に接続する場合、現時点では、App Service Environment (ASE)、Azure Kubernetes Service (AKS)、Azure API Management はサポートされていません。

#### [ドキュメント](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#test-the-function-in-azure) 参考に Azure Function で応答をテストします。
ローカル環境でデバッグした際と同様の結果が得られれば問題ありません。

### AI Search スキルセットへの組み込み
[パイプラインに接続する手順](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#connect-to-your-pipeline) と [カスタム Web API スキル](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-custom-skill-web-api) の定義を参考に設定します。<br/>

[normalized_images の data プロパティ](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-concept-image-scenarios#about-normalized-images) に正規化画像を Base64 でエンコードした文字列が入るため、スキルの入力で `/document/normalized_images/*/data` をソースとして指定します。<br/>
例えば以下のようにスキルを設定します。
```json
    {
      "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
      "name": "#2",
      "description": "Our new Bing entity search custom skill",
      "context": "/document/normalized_images/*",
      "uri": "https://{functionName}.azurewebsites.net/api/Function1?code={hostKey}",
      "httpMethod": "POST",
      "timeout": "PT30S",
      "batchSize": 4,
      "degreeOfParallelism": 5,
      "authResourceId": null,
      "inputs": [
        {
          "name": "image",
          "source": "/document/normalized_images/*/data"
        }
      ],
      "outputs": [
        {
          "name": "text",
          "targetName": "text"
        }
      ],
      "httpHeaders": {},
      "authIdentity": null
    }
```

可能であれば、この後のプライベート環境の構築の手順での問題の切り分け (少なくともパブリックネットワークアクセスが有効であれば動作する) につながるため、予め検証環境で Azure Function、AI Search、AI Service のマルチサービスリソースを全てパブリックネットワークアクセスを有効化した状態で、AI Search のインデクサーを実行いただき、エラー無くスキルが動作し、インデックスに OCR の結果が格納されることを確認しておくと良いでしょう。
### プライベート環境の構築
構成としては、本ブログの最初に記載したものと同じになりますが、以下になります。<br/>
![image-e7cd3cbf-99dc-4aa5-903b-97d5641a6afa.png]({{site.baseurl}}/media/2024/08/image-e7cd3cbf-99dc-4aa5-903b-97d5641a6afa.png)

#### AI Service のマルチサービスリソースのプライベート環境の構成
- Azure Portal の AI Service のマルチサービスリソース 「ネットワーク」ブレードより、プライベートエンドポイントを追加します。
- サブリソースは `account` とします。
- プライベートエンドポイントを作成したら、パブリックネットワークアクセスを無効とします。

#### Azure Function の仮想ネットワーク統合の設定
- Azure Portal の Azure Function 「ネットワーク」ブレードより、「仮想ネットワーク統合」を設定し、接続先の 仮想ネットワーク/サブネットを設定します。
- 念のため Azure Function での AI Service マルチサービスリソース の FQDN の名前解決でプライベート IP アドレスが返るか確認します。
  - Azure Portal の Azure Function 「高度なツール」を起動し、[nameresolver](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-networking-options?tabs=azure-portal#tools) をのコマンドを以下のように実行します。

```powershell
nameresolver <AI Service 名>.cognitiveservices.azure.com
```
以下のように、`10.0.1.4` といったプライベート IP アドレスが返れば一旦は問題なさそうです。

~~~
nameresolver <AI Service 名>.cognitiveservices.azure.com
Server: 168.63.129.16

Non-authoritative answer:
Name: <AI Service 名>.privatelink.cognitiveservices.azure.com
Addresses: 
	10.0.1.4
Aliases: 
	<AI Service 名>.privatelink.cognitiveservices.azure.com
~~~

#### AI Search 共有プライベートリンク設定
[プライベート リンクを経由した送信接続の作成](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create) を参考に AI Search にて Azure Function リソースへの共有プライベートリンクを作成します。<br/>

ポイントは以下になります。
- リソースの種類は `Microsoft.Web/sites`、サブリソースは `sites` として、対象の Azure Function リソースを指定します。以下の画像が参考になれば幸いです。

![image-026a7b3d-3b4d-443a-9f8d-100a046eb3a2.png]({{site.baseurl}}/media/2024/08/image-026a7b3d-3b4d-443a-9f8d-100a046eb3a2.png)

- [ドキュメント](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#2---approve-the-private-endpoint-connection) にも記載がございますが、共有プライベートリンク作成後、Azure Function 側でプライベートエンドポイントを承認する手順をお忘れなく実施いただけますと幸いです。
- [ドキュメント](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#4---configure-the-indexer-to-run-in-the-private-environment) にも記載がございますが、共有プライベートリンク作成後、インデクサーでプライベート接続を利用する設定を追加いただく必要がありますので、お忘れなく実施いただけますと幸いです。

### インデクサーを実行し、エラー無くドキュメントが取り込まれることを確認します。

# 補足
なお、現時点でプレビューの機能とはなりますが、[Azure OpenAI Embedding スキル](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-skill-azure-openai-embedding) については、共有プライベートリンクを利用することで、Azure OpenAI Service のパブリックネットワークアクセスを無効としてもご利用が可能です。以下のブログに詳細がございますので、参考になれば幸いです。<br/>
[Azure AI Search から Azure OpenAI Service へ可能な限りセキュアに接続したい](https://azure.github.io/jpazpaas/2024/01/25/search-private-access-to-openai.html)

# 免責事項
サンプル コードは、ご要件を満たす最低限の処理を実装したコードであり、弊社にてその動作を保証するものではございません。ご使用の際には、貴社環境に合わせて変更およびエラー処理を追加していただき、検証、動作確認をご実施くださいますようお願いいたします。サンプル内で使用しております API などの詳細な情報に関しては、弊社公開情報等をご参照くださいますようお願い申し上げます。

<br>
<br>

---

<br>
<br>

2024 年 08 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>