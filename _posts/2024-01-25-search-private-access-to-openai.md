---
title: "Azure AI Search から Azure OpenAI Service へ可能な限りセキュアに接続したい"
author_name: "Takumi Nagaya"
tags:
    - "AI Search"
---

# 質問
Azure AI Search でベクトル検索を利用するために、インデクサー実行時に Azure OpenAI Embedding スキルでコンテンツをベクトル化したいです。<br/>
しかし、Azure OpenAI Service のネットワーク設定でパブリックアクセスを無効とすると、Azure OpenAI Embedding スキルが失敗します。<br/>
可能な限り Azure AI Search から Azure OpenAI Service への接続をセキュアにする方法はありますでしょうか。

インデクサーを実行すると以下のようなエラーが発生します。

~~~
メッセージ
Could not execute skill because the Web Api request failed.
詳細
Web Api response status: 'Forbidden', Web Api response details: '{"error":{"code":"403","message": "Public access is disabled. Please configure private endpoint."}}'
~~~
![image-afddb292-ce11-42dc-a5d6-c25aeb92e136.png]({{site.baseurl}}/media/2024/01/image-afddb292-ce11-42dc-a5d6-c25aeb92e136.png)

また、Azure OpenAI Service のパブリックアクセスを無効化せず、「選択したネットワークとプライベート エンドポイント」からのアクセスに制限している場合は以下のエラーが発生します。 
~~~
メッセージ
Could not execute skill because the Web Api request failed.
詳細
Web Api response status: 'Forbidden', Web Api response details: '{"error":{"code":"403","message": "Access denied due to Virtual Network/Firewall rules."}}'
~~~

# 回答
現在は以下の 2 点の選択肢がございます。<br/>
それぞれ長所と短所を記載いたしましたので、ご検討いただけますと幸いです。<br/>
恐れ入りますが、2 つ目のマネージド ID を利用する方法は、Azure OpenAI Service がパブリックアクセスを無効としているとご利用することがかないませんのでご注意ください。

1. **Azure AI Search の Azure OpenAI Service への共有プライベートリンク経由でアクセスするように設定します。**
   - 長所
     - Azure AI Search から Azure OpenAI Service へプライベート接続ができます。
     - 手順として共有プライベートリンクの作成とインデクサーの設定を変更すればよく、管理が容易です。
   - 短所
     - スキルセットを利用する場合、Azure AI Search のプランは Standard 2 以上が必要となり、Standard 以下のプランと比較して、ご利用料金が高くなります。
2. **Azure AI Search から Azure OpenAI Service に対してマネージド ID 認証を利用して接続します。**  
   - 長所
     - マネージド ID は Azure AI Search の Basic プランから利用可能であるため、共有プライベートリンクを使用する場合と比較してご利用料金は低くなります。 
   - 短所
     - Azure AI Search から Azure OpenAI Service へのアクセスは Microsoft バックボーンネットワークを経由するため、パブリックインターネットを経由することはございませんが、プライベート接続とはなりません。
     - Azure OpenAI Service 側でパブリックアクセスを無効とした場合はご利用することがかないません。

1 の選択肢が可能となった 2024 年 1 月以前は、以下に記載の Azure Functions をご利用する方法もございました。<br/>
ただし、Azure OpenAI Service への共有プライベートリンクがサポートされたことで、Azure OpenAI Service への接続で、わざわざ Azure Functions への共有プライベートリンクを利用する意義は薄くなりました。

>パブリック ネットワークを無効にする必要がある場合は、 プライベート エンドポイント をサポートする Azure Functionを実装したカスタム Web API スキルを構成し、 同じ VNETに Azure AI サービス リソースを追加します。 この方法では、プライベート エンドポイントを使用して、カスタム スキルから直接 Azure AI サービス リソースを呼び出すことができます。<br/>
>[キーの使用方法](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-attach-cognitive-services?tabs=portal%2Cportal-remove#how-the-key-is-used)

以下に 2 つの選択肢の詳細を記載いたします。

## 1. Azure AI Search の Azure OpenAI Service への共有プライベートリンク経由でアクセスするように設定します。
[共有プライベートリンク](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create) は、Azure AI Search から別の Azure サービスへのプライベート接続が必要な場合に利用します。<br/>
共有プライベートリンクを作成し、インデクサーでプライベート接続を利用する設定を追加するだけでご利用可能になります。また、共有プライベートリンクによって作成されるプライベートエンドポイントリソースは Microsoft で管理されるため、お客様にてプライベートエンドポイントやプライベート DNS ゾーン等を管理する必要はございません。<br/>

共有プライベートリンクの利用によって以下の構成となります。<br/>
![search-private-access-9a3051ea-003c-4767-9d32-88cc714bb4ca.jpg]({{site.baseurl}}/media/2024/01/search-private-access-9a3051ea-003c-4767-9d32-88cc714bb4ca.jpg)
<br/>
<br/>
ただし、共有プライベートリンクの[前提条件](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#prerequisites) に以下のように記載がございます。スキルセットをご利用の場合、Azure AI Search サービスのレベルが Standard 2 以上である必要があり、Azure AI Search のご利用料金が高くなる点はご注意ください。<br/>
また、既に Azure AI Search を Standard 1 以下のプランで作成いただいている場合は、既存のリソースのプラン変更はできないため、新しく Standard 2 以上の Azure AI Search リソースを作成し、移行いただけますと幸いです。<br/>

> AI エンリッチメントとスキルセットを使用している場合、レベルは Standard 2 (S2) 以上である必要があります。 

あくまで参考情報となりますが、以前は Azure OpenAI Service への共有プライベートリンク経由の接続はサポートされておりませんでしたが、2024 年 1 月 16 日 に[ドキュメントが更新](https://github.com/MicrosoftDocs/azure-docs/commit/b60428329ec81d25e770d5696a15bc6fb6c16b02)され、Azure OpenAI Service への共有プライベートリンクがプレビューでご利用可能となりました。現在は一般利用可能になっております。<br/>
[サポートされているリソースの種類](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#supported-resource-types) の項目に以下のように記載されております。<br/>

|リソースの種類 | サブリソース (またはグループ ID) |
|---|---|
|Microsoft.CognitiveServices/accounts | openai_account|

### 設定手順
以下に設定手順について参考となるドキュメントと、手順実施の際の参考情報を記載いたします。

#### 1. [共有プライベート リンクを作成する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#1---create-a-shared-private-link) を参考に設定します。
Azure OpenAI Service へ接続するため、以下の画像のように「リソースの種類」 は `Microsoft.CognitiveServices/accounts`、「対象サブリソース」は `openai_account` とします。<br/>
![image-98f65f48-6c67-4845-86d2-f8d48411b208.png]({{site.baseurl}}/media/2024/01/image-98f65f48-6c67-4845-86d2-f8d48411b208.png)
#### 2. [プライベート エンドポイント接続を承認する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#1---create-a-shared-private-link) を参考に設定します。
#### 3. [共有プライベート リンクの状態を確認する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#3---check-shared-private-link-status) を参考に設定します。
#### 4. [プライベート環境で実行されるようにインデクサーを構成する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#4---configure-the-indexer-to-run-in-the-private-environment) を参考に設定します。
既にインデクサーが存在する場合は、Azure Portal より以下のように `"executionEnvironment": "private"` を追記して保存いただけますと幸いです。<br/>
![image-582712e4-1123-4e50-89a3-999ae9bc1223.png]({{site.baseurl}}/media/2024/01/image-582712e4-1123-4e50-89a3-999ae9bc1223.png)

## 2. Azure AI Search から Azure OpenAI Service に対してマネージドID認証を利用して接続します。
共有プライベートリンクとスキルセットをご利用の場合、Azure AI Search サービスの Standard 2 以上のプランが必要となり、料金が高くなります。<br/>

一方で、プライベート接続はできないものの、Azure OpenAI Service のアクセス制限を構成しつつ、Azure AI Search からアクセスする方法として、マネージド ID 認証を利用する方法がございます。マネージド ID 認証は Azure AI Search の Basic プラン以上であれば利用できるため、Azure AI Search の料金は共有プライベートリンクを利用する場合と比較して安くなります。<br/>
Azure OpenAI Service のドキュメントに以下の記載がございますので、参考になれば幸いです。

>他のアプリのネットワーク規則を維持しながら、信頼された Azure サービスのサブセットに Azure OpenAI へのアクセス権を付与することができます。 その後、これらの信頼されたこれらのサービスでは、マネージド ID を使用して、Azure OpenAI サービスの認証が行われます。<br/>
>次の表は、これらのサービスのマネージド ID に適切なロールが割り当てられている場合に Azure OpenAI にアクセスできるサービスを示しています。<br/>
>(中略)<br/>
>Azure AI Search<br/>

[Azure OpenAI の信頼された Azure サービスへのアクセス権を付与する](https://learn.microsoft.com/ja-jp/azure/ai-services/cognitive-services-virtual-networks?tabs=portal#grant-access-to-trusted-azure-services-for-azure-openai)

以下のような構成になります。<br/>
![search-private-access-Page-2-a143ba1e-96bc-474d-88eb-0e80128750eb.jpg]({{site.baseurl}}/media/2024/01/search-private-access-Page-2-a143ba1e-96bc-474d-88eb-0e80128750eb.jpg)

もちろん、プライベート接続はできないものの、[マイクロソフトのグローバル ネットワーク](https://learn.microsoft.com/ja-jp/azure/networking/microsoft-global-network#get-the-premium-cloud-network) のドキュメントに記載がある通り、Microsoft のサービス間の通信はパブリックインターネットを経由することはございませんので、その点はご安心ください。<br/>

共有プライベートリンクのご利用がコストの都合上難しい場合の代替案として、ご検討いただけますと幸いです。

また、質問に記載の 2 番目のエラーメッセージ `Access denied due to Virtual Network/Firewall rules.` を解消する方法にもなっています。

### 設定手順
以下に設定手順を記載いたします。

#### 1. Azure OpenAI Service 側でネットワーク規則の例外を許可します。
[Azure OpenAI の信頼された Azure サービスへのアクセス権を付与する - Azure Portal の使用](https://learn.microsoft.com/ja-jp/azure/ai-services/cognitive-services-virtual-networks?tabs=portal#using-the-azure-portal) のドキュメントにしたがい、対象の Azure OpenAI Service に対して REST API を使用してネットワーク規則の例外を作成します。<br/>

具体的には、Azure OpenAI の「ネットワーク」ブレードから、「選択したネットワークとプライベートエンドポイント」を選択し、
「Allow Azure services on the trusted services list to access this cognitive services account.」をチェックします。<br/>
以下のスクリーンショットが参考になれば幸いです。<br/>
![image-207eddb1-ca9f-4ab0-bf38-69ff93f63c0b.png]({{site.baseurl}}/media/2024/01/image-207eddb1-ca9f-4ab0-bf38-69ff93f63c0b.png)


#### 2. Azure AI Search 側で [システム マネージド ID を作成する](https://learn.microsoft.com/ja-jp/azure/search/search-howto-managed-identities-data-sources?tabs=portal-sys%2Cportal-user#create-a-system-managed-identity) を参考に、システム割り当てマネージドIDを有効化します。
#### 3. Azure OpenAI Service リソースのスコープで、手順 2 で作成したシステム割り当てマネージド ID に対して「Cognitive Services OpenAI ユーザー」ロールを割り当てます。[ロールの割り当て](https://learn.microsoft.com/ja-jp/azure/search/search-howto-managed-identities-data-sources?tabs=portal-sys%2Cportal-user#assign-a-role) の手順をご確認ください。

必要なロールについては以下のドキュメントに記載がございます。

>Azure OpenAI にテキストを送信するには、マネージド ID に Cognitive Services OpenAI ユーザー アクセス許可が必要です。<br/>

[Azure OpenAI Embedding スキル](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-skill-azure-openai-embedding#skill-parameters)
 
#### 4. スキルセットで `apiKey` と `authIdentity` を `null` にします。もし既存の定義に記載がない場合はそのままで問題ございません。

```json
    {
      "@odata.type": "#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill",
      "name": "#1",
      "description": null,
      "context": "/document/pages/*",
      "resourceUri": https://airesource.openai.azure.com,
      "apiKey": null,
      "deploymentId": "****",
```
以下のドキュメントにマネージド ID を利用する方法について記載がございますので、参考になれば幸いです。

>Azure OpenAI に接続するために検索サービスによって使用されるユーザー マネージド ID。 システムまたはユーザーのマネージド ID を使用できます。 システム マネージド ID を使用するには、apiKey と authIdentity を空白のままにします。 システム マネージド ID が自動的に使用されます。 <br/>

[Azure OpenAI Embedding スキル](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-skill-azure-openai-embedding#skill-parameters)

#### 5. インデックスの設定で Azure OpenAI Service への接続でマネージド ID を利用するようにします。

ベクトル検索をする場合、検索文字列に対しても Azure OpenAI Service の API を利用してベクトル化する必要があります。  
マネージド ID を利用する設定がインデックスにないと、検索時に以下のようなエラーが発生します。

~~~
Could not complete vectorization action. Could not reach the vectorization endpoint.
~~~
![image-631d1172-f493-4659-8ac5-82d475769dcc.png]({{site.baseurl}}/media/2024/01/image-631d1172-f493-4659-8ac5-82d475769dcc.png)

インデクサーの設定と同様に、インデックス定義の以下の部分で `apiKey` と `authIdentity` を `null` にして保存します。
```json
    "vectorizers": [
      {
        "name": "vector-1706146801319-vectorizer",
        "kind": "azureOpenAI",
        "azureOpenAIParameters": {
          "resourceUri": "https://********.openai.azure.com",
          "deploymentId": "tsample",
          "apiKey": null,
          "authIdentity": null
        },
        "customWebApiParameters": null
      }
    ]
```
Azure Portal では以下のように変更し、保存します。<br/>
![image-997b6bee-19e4-4c0b-bbcf-67fd668f114f.png]({{site.baseurl}}/media/2024/01/image-997b6bee-19e4-4c0b-bbcf-67fd668f114f.png)

#### 6. インデクサーを実行しエラーなく処理が完了することと、インデックスの検索でエラーが発生しないことを確認します。

<br>
<br>

---

<br>
<br>

2024 年 10 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>