---
title: AI Search のカスタムスキルで Azure Functions を呼び出す際、マネージド ID で認証する方法について
author_name: "chansiklee"
tags:
    - "AI Search"
    - "Function App"
---

# はじめに
Azure AI Search サービスのカスタムスキル機能を使い、Azure Function App の Web API スキルを呼び出すことができます。<br>
[スキルセット内のカスタム Web API スキル - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-custom-skill-web-api)<br>
そこでキーではなく、システムマネージド ID に RBAC 権限を付与して認証する方法をご案内致します。<br>

# 設定不備の際、発生するエラーの例
`Web Api response status: 'Forbidden', Web Api response details`

# 手順
既に Web API を利用するスキルセットが用意されている前提で、それぞれのリソースごとに設定方法をご案内致します。

## Function App
関数アプリを開き、「認証」ブレードから「ID プロバイダー」のリンクを押下し、下記画像の通り ID プロバーダーのリンクをクリックします。<br>
![image-e591ed72-da69-4568-91c6-0d59c3bbae36.png]({{site.baseurl}}/media/2025/02/image-e591ed72-da69-4568-91c6-0d59c3bbae36.png)<br>

「API の公開」ブレードを開きますと「アプリケーション ID URI」があり、こちらを記録して頂いて<br>
後で Azure AI Search のWebAPI スキルの「authResourceId」としてご利用ください。<br>
![image-742445c6-e67b-4c7c-851f-c8ef107ab710.png]({{site.baseurl}}/media/2025/02/image-742445c6-e67b-4c7c-851f-c8ef107ab710.png)<br>


## AI Search
スキルセットにまだ WebAPI スキルが実装されていなければ、以下のページから抜粋して頂きます。<br>
[Bing Entity Search API を使用したカスタム スキルの例 - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-create-custom-skill-example#connect-to-your-pipeline)
スキルを以下の様に修正します。<br>
① uri の「?code」以降を消します。<br>

② uri の下に authResourceId を追加して、appId には上の Function App の手順で取得した ID を入れます。<br>
`"authResourceId" :"api://<appId>"`<br>
[スキルセット内のカスタム Web API スキル - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-custom-skill-web-api#skill-parameters)<br>
  
③ inputs の内容などは Function App に実装して頂いたプログラムに合わせて適宜変更してください。<br>
```
{
       "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
       "description": "Our new Bing entity search custom skill",
       "uri": "https://[your-entity-search-app-name].azurewebsites.net/api/EntitySearch",
       "authResourceId" :"api://<appId>"
         "context": "/document",
         "inputs": [
           {
             "name": "name",
             "source": "/document/content"
           }
         ],
         "outputs": [
           {
             "name": "description",
             "targetName": "description"
           }
         ]
     }
```
上述の手順で、Azure AI Search からマネージド ID を用いて Function App の Web API スキルを利用することができます。<br>


## エンタープライズアプリケーション（オプション）
以下も必要に応じてご参考になる点があれば幸いです。<br>
まず以下の手順に従い、Azure AI Search 側の ID を表現する「エンタープライズ アプリケーション」の「アプリケーション ID」 をメモしておきます。<br>
Azure AI Search の ID → システム割り当て済み → オブジェクト（プリンシパル） ID をコピーします。<br>
![image-c460e1e3-f9bb-48a4-9cd2-d897a98c523c.png]({{site.baseurl}}/media/2025/02/image-c460e1e3-f9bb-48a4-9cd2-d897a98c523c.png)<br>

コピーした ID をAzure ポータルの検索ボックスに入力し、「Microsoft Entra ID で検索を続行してください」をクリックします。<br>
![image-4ec96ea7-5411-4711-a4d5-6ee804f5e8aa.png]({{site.baseurl}}/media/2025/02/image-4ec96ea7-5411-4711-a4d5-6ee804f5e8aa.png)<br>

概要のページに以下の様にテナント ID での検索が行われますので、<br>
「エンタープライズ アプリケーション」で Azure AI Search のリソース名で表示されたアプリをクリックします。<br>
![image-dc4c337f-7496-439a-a75c-a50f20013ede.png]({{site.baseurl}}/media/2025/02/image-dc4c337f-7496-439a-a75c-a50f20013ede.png)<br>

「アプリケーション ID」を取得します。<br>
![image-743dce19-d2fd-4c94-8f50-2c020807a79b.png]({{site.baseurl}}/media/2025/02/image-743dce19-d2fd-4c94-8f50-2c020807a79b.png)<br>

関数アプリを開き、「認証」ブレードから、「ID プロバイダー」の「編集」リンクを押下します。<br>
![image-cdc6c273-e654-4845-b588-afbabb02746d.png]({{site.baseurl}}/media/2025/02/image-cdc6c273-e654-4845-b588-afbabb02746d.png)<br>

「許可されたクライアント アプリケーション」において、鉛筆マークを押下して、上でメモしておいた、<br>
Azure AI Search 側の アプリケーション ID をここに追加し、保存をします。<br>
![image-4c020258-bc87-43dd-a511-557e6cdce6d1.png]({{site.baseurl}}/media/2025/02/image-4c020258-bc87-43dd-a511-557e6cdce6d1.png)<br>


## 解決できなかった場合（オプション）
詳細な原因調査のトラブルシューティングは弊社サポート窓口へご連絡いただけますと幸いですが、多くの場合、<br>
Function App で Entra ID 認証（EasyAuth）を行いつつ、かつ、対象の Function App 内の関数でも関数レベルの認証を行う形で二重の認証が用意され、<br>上述の手順で EasyAuth 側の認証は通っていたものの、関数レベルの認証に失敗している可能性がございます。<br>
関連する内容は以下のページをご参照ください。<br><br>
■ 関数のアクセスキー<br>
[Azure Functions のセキュリティ保護 | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/security-concepts#function-access-keys)<br><br>
■ 承認レベル<br>
[Azure Functions の HTTP トリガー | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cfunctionsv2&pivots=programming-language-javascript#http-auth)<br>
そのため、以下のいずれかの対処策を実施頂き、事象が改善できるかご確認をお願い致します。<br>


### A 案） Function App の各関数で関数レベルの認証を無効化する
既に Function App で Entra ID 認証を実施しているため、二重での認証となる関数レベルの認証は無効化することで回避する方法となります。<br>
（今回の投稿でご案内させて頂いたのは A 案を前提にご案内させて頂きました。）<br>
具体的には、開発するに VS Code などで Python v1 モデル・v2 モデルを使用する場合と、Azure ポータルで開発する場合それぞれ以下の手順をご参照ください。<br>
Function App を VS Code 等で開発されている場合は、Function App の Python プロジェクトで<br>
Python v1 モデルの場合は各関数の function.json で authLevel を functions -> anonymous にして頂き、<br>
![image-a6e9d4b5-2795-448c-9e49-4ef6423828b6.png]({{site.baseurl}}/media/2025/02/image-a6e9d4b5-2795-448c-9e49-4ef6423828b6.png)<br>
[Azure Functions の Python 開発者向けリファレンス | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-python?tabs=get-started%2Casgi%2Capplication-level&pivots=python-mode-configuration#programming-model)<br>

Python v2 モデルの場合は各関数のコード上で http_auth_level を func.AuthLevel.FUNCTION から func.AuthLevel.ANONYMOUS に変更をお願いします。<br>
![image-9ada2b24-6a1e-45e8-810d-2bd4cb95cdea.png]({{site.baseurl}}/media/2025/02/image-9ada2b24-6a1e-45e8-810d-2bd4cb95cdea.png)<br>
[Azure Functions の Python 開発者向けリファレンス | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-python?tabs=get-started%2Casgi%2Capplication-level&pivots=python-mode-decorators#http-streams-examples)<br>

ポータルで開発されている場合は、Function App リソースの「概要」ページにて、各関数へのリンクをクリックし、<br>
![image-59cd1e28-01d7-4134-a178-5bb951774854.png]({{site.baseurl}}/media/2025/02/image-59cd1e28-01d7-4134-a178-5bb951774854.png)<br>

各関数の functions.json を開き、authLevel をfunction -> anonymous に変更をお願い致します。<br>
![image-496ab4c2-0258-45d4-bd69-33dd2a23ce44.png]({{site.baseurl}}/media/2025/02/image-496ab4c2-0258-45d4-bd69-33dd2a23ce44.png)<br>


### B 案） 関数レベルの認証も通るようにAI Search 呼び出し時のURL を更新する
Function App で Entra ID 認証を実施しているため不要ではあると思いますが、もし Function App の関数側でも認証を残されたい場合は、<br>
Function App を呼び出す際に関数の認証キーも付与するように URL を変更いただければと思います。<br>
具体的には、関数の認証キーも付与した (関数を呼び出すための) URL は下記のドキュメントに記載の手順で手に入れることができるため、<br>
該当のアクセス キーを URL のクエリ文字列として付与していただく形になります。<br>
例) 下記赤枠の uri を「https://document-search-test-customskill.azurewebsites.net/api/file2txt?code=<関数キー>」といた形式に変更いただければと存じます。（当初消して頂くように案内した部分です。）<br>

![image-468b98c6-f0bf-45d7-806f-7443184d00fa.png]({{site.baseurl}}/media/2025/02/image-468b98c6-f0bf-45d7-806f-7443184d00fa.png)<br>

関連する内容は以下のドキュメントをご参照ください。<br><br>
■ 関数のアクセスキーを取得する<br>
[Azure Functions でアクセス キーを操作する | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/function-keys-how-to?tabs=azure-portal#get-your-function-access-keys)<br><br>
■ アクセスキーの承認<br>
[Azure Functions の HTTP トリガー | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cfunctionsv2&pivots=programming-language-javascript#api-key-authorization)<br>

アクセス キーを付与した URL は Function App リソースの関数のリンクから <br>
「関数URL の取得」を選択しても得られますので、よろしければそちらもご活用ください。<br>
![image-086fccbc-7cb2-4dfb-af91-f8f9cbc8ff8a.png]({{site.baseurl}}/media/2025/02/image-086fccbc-7cb2-4dfb-af91-f8f9cbc8ff8a.png)<br>

上述の対応で正常に認証が行えない場合は、より具体的な調査が必要になるため弊社サポート窓口へご連絡ください。<br>

以上、Azure AI Search サービスのカスタムスキル機能を使い、Azure Function App の Web API スキルを呼び出す際、マネージド ID で認証する手順を紹介いたしました。

<br>
<br>

---

<br>
<br>

XXXX 年 XX 月 XX 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>