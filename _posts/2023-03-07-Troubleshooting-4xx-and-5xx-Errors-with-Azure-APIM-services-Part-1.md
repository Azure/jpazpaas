---
title: "Azure APIM services の 4xx や 5xx のトラブルシューティング パート 1"
author_name: "hmachida"
tags:
    - API Management
---

このポストは、2021 年 3 月 1 日に投稿された [Troubleshooting 4xx and 5xx Errors with Azure APIM services
](https://techcommunity.microsoft.com/t5/azure-paas-blog/troubleshooting-4xx-and-5xx-errors-with-azure-apim-services/ba-p/2115752) の翻訳です。

<h1>パート I - 4xx エラーのトラブルシューティング</h1>

# デバッグとトラブルシューティングの概要

 

API Management は、クライアントサイドからバックエンドの API サービスにリクエストを転送するためのプロキシに他なりません。APIM は、クライアント側からの入力に基づいて、リクエストがバックエンドに到達する前にリクエストの変更や特定の処理ができます。理想的なシナリオでは、APIM 内の API は、バックエンド API からの正しいレスポンス本文と成功のレスポンス コード（通常は 200 OK）を返すことが期待されます。

リクエストが失敗した場合は、API 呼び出し中に何が失敗していたかというエラーメッセージと失敗のレスポンスコードが出力されることがあります。


しかし、API リクエストが汎用的な 4xx や 5xx エラーで失敗し、詳細なエラーメッセージがなく、エラーの原因の切り分けや特定が難しい場合もあります。

このような場合、トラブルシューティングの最初のポイントは、エラーコードが APIM によって投げられているのか、バックエンド API によって投げられているのかを切り分けることです。ほとんどの場合、エラーコードはバックエンドによって生成され、プロキシである APIM はそのレスポンス（エラーコード）をユーザーに転送するだけです。そのため、切り分けが重要です。ただ、APIM がエラーコードを転送しているため、エラーが APIM から発生したと誤解することもあります。

# Azure APIM リクエストの失敗に関するトラブルシューティング

 

APIM サービスに対して API リクエストを送信し、リクエストが最終的に「HTTP 500 - Internal Server Error」メッセージで失敗したとします。

そのような汎用的なエラー メッセージでは、API 呼び出しプロセスで関係する内部および外部のコンポーネントが複数あるため、失敗した API リクエストの原因や発生元を切り分けることが困難になります。


* responseCode が backendResponseCode と一致する場合、バックエンドに問題があります。APIM に設定されているバックエンド API をトラブルシューティングする必要があります。
* responseCode が backendResponseCode と一致せず errorReason が空の場合、インスペクタートレースを使って、そのポリシーロジックがエラーを返しているかどうかを確認する必要があります。
* errorReason が空でない場合は、APIM の問題であり、エラーコードのトラブルシューティングが問題解決に役立ちます。

**インスペクタートレース**

問題が再現可能である場合、APIM の API リクエストのトレースを有効にすることでトラブルシューティングができます。Azure APIM サービスには API リクエストの "Ocp-Apim-Trace" を有効にするオプションがあります。このオプションでは、詳細な情報を含むトレースを生成し、リクエスト処理をステップ・バイ・ステップで検証できるため、エラーの原因調査を効率的に進めることができます。


参考：https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-api-inspector

 

## Azure Monitor Log Analytics の診断ログ

APIM サービスの診断ログを有効にすることもできます。診断ログはストレージアカウントへのアーカイブ、Event Hubs リソースへのストリーミング、または Azure Monitor Log Analytics ログへの送信が可能で、シナリオや要件に応じてさらにクエリでログを抽出することができます。

これらのログは、トラブルシューティングだけでなく、監査にも重要な操作やエラーに関する情報を提供します。診断ログのもっとも良い点は、トラブルシューティングに必要な API リクエストごとの詳細なログが提供され、トラブルシューティングの助けになることです。


参考：https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-use-azure-monitor#resource-logs

 

APIM 診断設定を有効にするとき、ストレージアカウントと Event Hubs は、診断ログ収集/ストリーミングの単一の宛先として機能します。一方、宛先を Log Analytics Workspace にすると、リソースログ収集に関する 2 つのモードが提供されます。

* Azure diagnostics - データは、複数の異なるリソースタイプのリソースから診断情報を収集する **AzureDiagnostics** テーブルに書き込まれます。
* リソース固有 - データは、リソースのカテゴリごとに個別のテーブルに書き込まれます。APIM の場合、ログは **ApiManagementGatewayLogs** テーブルに保存されます。
 

参考記事：https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/resource-logs#send-to-log-analytics-workspace

 

リソースのログを **ApiManagementGatewayLogs** テーブルに保存したい場合は、以下のサンプルのスクリーンショットにあるように、**[リソース固有]** オプションを選択します。

![SherrySahni_0-1612545513929-1325052d-878d-4d44-ad31-8138351c97a4.jpeg]({{site.baseurl}}/media/2023/03/SherrySahni_0-1612545513929-1325052d-878d-4d44-ad31-8138351c97a4.jpeg)

以下は、Log Analytics Workspace で生成される診断ログのサンプルです。これらのログは、タイムスタンプ、リクエストステータス、api/operation id、経過時間、呼び出し元/クライアント IP、メソッド、呼び出した URL、呼び出したバックエンド URL、レスポンスコード、バックエンド レスポンス コード、リクエストサイズ、レスポンスサイズ、エラーソース、エラー理由、エラーメッセージ、など API リクエストの詳細なログを提供します。

![SherrySahni_1-1612545513952-fd97418f-518c-44e4-a095-585aa4f34a16.jpeg]({{site.baseurl}}/media/2023/03/SherrySahni_1-1612545513952-fd97418f-518c-44e4-a095-585aa4f34a16.jpeg)

**注**：初期設定後、リソースプロバイダーから診断ログが宛先にストリーミングされるまでに数時間かかる場合があります。


以下は、ログ収集のモードに応じて API リクエストの診断ログを照会するためのサンプルクエリです。API ID やレスポンスコードなどクエリを調整することで、特定のログをフィルタリングすることも可能です。



APIM Management サービス > [診断設定] > 指定の Log Analytics ワークスペース > [ログ] ブレードを順に選択して、クエリを実行します。


```
AzureDiagnostics
| where TimeGenerated > ago(24h)
| where_ResourceId == “apim-service-name”
| limit 100
```

```
ApiManagementGatewayLogs
| where TimeGenerated > ago(24h)
| limit 100
```

## Application Insights へのログ

 

もう一つの選択肢は、診断ログデータを生成するために、APIM サービスを Application Insights と統合することです。

Azure API Management と Azure Application Insights を統合する方法 - https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-app-insights

 

以下は、Azure APIM API のリクエストに関連する診断データを取得できる requests テーブルへのサンプル クエリです。

統合した Application Insights > [ログ] ブレードを順に選択して、クエリを実行します。


```
requests
| where timestamp > ago(24h)
| limit 100
```

あるいは、エラー処理ポリシーを使って、APIM のエラー処理を行うこともできます。
https://learn.microsoft.com/ja-jp/azure/api-management/api-management-error-handling-policies

<br/>
 

ここで、失敗した API リクエストのエラータイプとエラー メッセージの詳細を取得するために、診断ログを有効にしました。次に、APIM サービスでよく見られる 4xx と 5xx エラーについて見ていきます。

 

このトラブルシューティングシリーズでは、以下のシナリオを取り上げます。

* Azure APIM サービスを使用してAPI リクエストを行う際に見られる、一般的な 4xx および 5xx エラーを捕捉する。
* これらのエラーで失敗する API リクエストをデバッグまたはトラブルシューティングする方法についてガイダンスを提供する。
* 一般的に見られる 4xx および 5xx エラーを修正するための解決策を提供する。





# APIM サービスでの 4xx および 5xx エラーのトラブルシューティング

 

失敗した API リクエストのトラブルシューティングの最初の重要なステップは、返されたレスポンスコードの発生元を調査することです。

APIM サービスの診断ログを有効にしている場合、ResponseCode と BackendResponseCode のカラムからレスポンスコードの発生元を調査できます。

クライアントに返される 4xx または 5xx レスポンスが主にバックエンド API から返されている場合（BackendResponseCode 列を確認）、APIM サービスはバックエンドで発生したエラーをクライアントに転送しているだけで問題の発生元ではないため、バックエンド側から調査を行うべきです。

## 4xx エラー

### エラーコード: 400

#### シナリオ 1

##### 症状

API Management は、実装中は問題なく動作していました。しかし、Azure ポータルの API 管理で **[テスト]** オプションを使用して呼び出すと、400 Bad Request が返るようになりました。クライアントアプリやアプリケーションを使用してアクセスした場合は、期待通りの結果が得られます。


##### トラブルシューティング

上記のシナリオから、Azure ポータルの API 管理から呼び出した場合のみ、API が 400 Bad Request が返されることがわかります。 

![SherrySahni_2-1612545513958-67d2f811-5a58-4f9e-88a3-3880f52aebd4.png]({{site.baseurl}}/media/2023/03/SherrySahni_2-1612545513958-67d2f811-5a58-4f9e-88a3-3880f52aebd4.png)


しかし、他の方法で呼び出すと正常な結果が返ります。エラーメッセージには、APIM のエンドポイント（`contosoapisk.azure-api.net`）を解決できなかったと明記されています。もしエンドポイントの問題であれば、API の呼び出しメソッドすべてで問題が発生するはずです。今回のケースではすべての呼び出しで問題が発生しているわけではないので、エンドポイントを検証します。同じマシンからコマンドプロンプトを使用してエンドポイントを解決するか、Ping テストを試してください。



##### 解決


この種のシナリオでは、APIM が仮想ネットワーク内に存在するかどうか、また、APIM が内部モードで構成されているかどうかを確認します
。[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-using-with-internal-vnet?tabs=stv2#enable-a-virtual-network-connection-using-the-azure-portal)によると、「ゲートウェイ URL はパブリック DNS には登録されないため、Azure portal で利用可能なテストコンソールは、**内部** VNet にデプロイされたサービスでは機能しません。 **開発者ポータル** で提供されるテストコンソールを使用してください。」とあります。


#### シナリオ 2

##### 症状
API Management の API を呼び出すと、「Error: The remote server returned an error: (400) Invalid client certificate」というエラーが発生する。


##### トラブルシューティング
シナリオを分析します。

この問題は、お客様が相互クライアント証明書認証を実装している場合に発生します。この場合、クライアントはポリシーの条件に従って有効な証明書を渡す必要があります。


```
<policies>
<inbound>
<base />
<choose>
<when condition="@(context.Request.Certificate == 2023 年 03 月 07 日時点の内容となります。<br> || !context.Request.Certificate.Verify() || context.Request.Certificate.Issuer.Contains("*.azure-api.net") || !context.Request.Certificate.SubjectName.Name.Contains("*.azure-api.net") || context.Request.Certificate.Thumbprint != "4BB206E17EE41820B36112FD76CAE3E0F7104F36") ">
<return-response>
<set-status code="403" reason="Invalid client certificate" />
</return-response>
</when>
</choose>
</inbound><backend><base />
</backend><outbound><base /></outbound><on-error>
<base /></on-error>
</policies>
```

証明書が通ったかどうかを確認するために、ocp-apim-trace を有効にします。以下のトレースは、API Management サービスがクライアント証明書を受信していないことを示しています。

![SherrySahni_3-1612545513972-7ee4e2e7-879e-41fa-a9f4-72227ef204e4.jpeg]({{site.baseurl}}/media/2023/03/SherrySahni_3-1612545513972-7ee4e2e7-879e-41fa-a9f4-72227ef204e4.jpeg)

##### 解決

有効なクライアント証明書を追加し、問題が解決しました。

![SherrySahni_4-1612545513976-32273dcf-0da9-4b74-a942-288098be32c2.jpeg]({{site.baseurl}}/media/2023/03/SherrySahni_4-1612545513976-32273dcf-0da9-4b74-a942-288098be32c2.jpeg)



### 類似のシナリオ

#### シナリオ 3
Error Reason: OperationNotFound<br/>
Error message: Unable to match incoming request to an operation.<br/>
Error Section: Backend


##### 解決
API に対して呼び出されるオペレーションが、API Management に存在しているか確認してください。存在しない場合は、操作を追加するか、リクエストを適宜修正してください。


#### シナリオ 4
Error Reason: ExpressionValueEvaluationFailure<br/>
Error message: Expression evaluation failed. EXPECTED400: URL cannot contain query parameters. Provide root site url of your project site (Example: https://sampletenant.sharepoint.com/teams/sampleteam )<br/>
Error Section: inbound

##### 解決
API Management の設定に従って、URL に API で定義されたクエリパラメーターのみが含まれていることを確認してください。クエリパラメーターが一致しない場合、このようなエラーメッセージが表示されることがあります。例えば、入力値として期待される値が整数で、文字列を入力した場合、このようなエラーになる可能性があります。


### 401 - Unauthorized issues

#### シナリオ 1

##### 症状

突然、Echo API が HTTP 401 - Unauthorized エラーを出力するようになった。

**Message**

HTTP/1.1 **401 Unauthorized**<br/>

{    "statusCode": 401,    "message": "Access denied due to missing subscription key. Make sure to include subscription key when making requests to an API."}


{
"statusCode": 401,
"message": "Access denied due to invalid subscription key. Make sure to provide a valid key for an active subscription."
}

##### トラブルシューティング

* API にアクセスするためには、開発者はまずプロダクトのサブスクリプションに参加する必要があります。サブスクリプションに参加すると、そのプロダクト内のすべての API に対して有効でリクエスト ヘッダーの一部として送信するサブスクリプションキーを取得できます。**Ocp-Apim-Subscription-Key** は、API に関連付けられたプロダクトのサブスクリプションキーを送信するためのリクエストヘッダーです。キーは自動的に入力されます。
* **Access denied due to invalid subscription key** については、**アクティブなサブスクリプションの有効なキーを提供していることを確認してください。** **Create resource** と **Retrieve resource** の操作の際に、**Ocp-Apim-Subscription-Key** リクエストヘッダーで誤った値を送信していることが分かります。
* APIM Developer ポータルから、サインイン後に **Profile** ページに移動して、特定のプロダクトの Subscription Key を確認できます（下図参照）。
* **Show** ボタンを選択すると、サブスクリプションに参加している各プロダクトのサブスクリプションキーを確認できます。<br/>
![SherrySahni_5-1612545513989-48c1fca3-89f0-4e33-a031-f4fbfee5a708.png]({{site.baseurl}}/media/2023/03/SherrySahni_5-1612545513989-48c1fca3-89f0-4e33-a031-f4fbfee5a708.png)<br/>
* Test タブから送信されるヘッダーを確認すると、Ocp-Apim-Subscription-Key リクエストヘッダーの値が間違っていることに気づきます。なぜそのようなことが起きるのか不思議に思うかもしれませんが、それはAPIM は自動的にリクエストヘッダーを正しいサブスクリプションキーで埋めるためです。
* **Design** タブの **Create resource** と **Retrieve resource** 操作の Frontend の定義を確認してみます。注意深く見ると、Headers タブの下で、それらの操作が誤った値がハードコードされた Ocp-Apim-Subscription-Key リクエストヘッダーを持っています。
* これを削除すれば、無効なサブスクリプションキーの問題は解決するはずですが、それでもサブスクリプションキーが見つからないというエラーが発生します。

このようなエラーメッセージが表示されました。

**Message**


HTTP/1.1 **401 Unauthorized**<br/>
<br/>
Content-Length: 152<br/>
Content-Type: application/json<br/>
Date: Sun, 29 Jul 2018 14:29:50 GMT<br/>
Vary: Origin<br/>
WWW-Authenticate: AzureApiManagementKey realm="https://pratyay.azure-api.net/echo",name="Ocp-Apim-Subscription-Key",type="header" {<br/>
  "statusCode": 401,<br/>
  "message": "Access denied due to missing subscription key. Make sure to include subscription key when making requests to an API."<br/>
}

* Echo API の設定に移動し、利用可能な製品のいずれかと関連付けられているかどうかを確認します。もしそうでなければ、この API を製品に関連付けて、サブスクリプションキーを取得する必要があります。

##### 解決

* 開発者は、API にアクセスするために、まず製品のサブスクリプションに参加する必要があります。サブスクリプションに参加すると、その製品内のすべての API で有効なサブスクリプションキーが取得できます。APIM インスタンスを作成した場合、あなたはすでに管理者なので、既定ですべての製品のサブスクリプションに参加しています。



### エラー コード: 401 Unauthorized issues

#### シナリオ

##### 症状

Developer Console で Echo API の OAuth 2.0 によるユーザ認証を有効にしました。API を呼び出す前に、Developer Console はリクエストの Authorization ヘッダーからユーザーに代わってアクセストークンを取得します。

**Message**

![SherrySahni_6-1612545513997-41d08d93-9930-4331-bcd6-6487164d3732.png]({{site.baseurl}}/media/2023/03/SherrySahni_6-1612545513997-41d08d93-9930-4331-bcd6-6487164d3732.png)





##### トラブルシューティング

* このシナリオのトラブルシューティングために、[APIM インスペクタートレース](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-api-inspector)の確認から始めます。レスポンスから Ocp-Apim-Trace へのリンクを見つけることができます。
* "JWT Validation Failed : Claim Mismatched" というメッセージがトレースにあり、送信されたヘッダーのトークンをデコードできないことが分かります。

![SherrySahni_7-1612545514011-9d1e8bed-f1e7-4a71-af61-6749554ef655.png]({{site.baseurl}}/media/2023/03/SherrySahni_7-1612545514011-9d1e8bed-f1e7-4a71-af61-6749554ef655.png)

* JWT Validation ポリシーのスコープを確認するには、**Calculate effective policy** ボタンを選択します。どのスコープにもアクセス制限ポリシーが実装されていない場合、次は製品レベルでポリシーを確認します。関連する製品に移動して、ポリシーオプションを選択します。


 
```
<inbound>
 <base />
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
          <openid-config url="https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration" />
           <required-claims>
        <claim name="aud">
             <value>bf795850-70c6-4f22- </value>
        </claim></required-claims>
        </validate-jwt>
 </inbound>
```

##### 解決

クレームに記載されているクレーム名が、AAD に登録されているアプリ名と一致しません。

クライアントアプリが登録されている Application ID を Claims セクションに入力することで認証エラーが解消します。
有効な Application ID を提供すると、HTTP レスポンスに HTTP/1.1 **200 OK** と表示されます。

### エラーコード: 403 - Forbidden issues

#### シナリオ

##### 症状

**GetSpeakers** API オペレーションは、パラメーターに指定された値に基づいてスピーカーの詳細を取得します。数日間使用した後、このオペレーションから HTTP 403- Forbidden エラーが発生するようになりました。他のオペレーションは期待どおりに正常に動作しています。


**Message**

HTTP/1.1 **403 Forbidden**<br/>
{  <br/>
    "statusCode": 403,<br/>
    "message": "Forbidden"<br/>
}

##### トラブルシューティング

* このシナリオのトラブルシューティングために、[APIM インスペクタートレース](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-api-inspector)の確認から始めます。レスポンスから Ocp-Apim-Trace へのリンクを見つけることができます。

![SherrySahni_8-1612545514029-6116e6ce-7896-4609-9fff-36f8e71f0bc9.png]({{site.baseurl}}/media/2023/03/SherrySahni_8-1612545514029-6116e6ce-7896-4609-9fff-36f8e71f0bc9.png)

特定の IP アドレス範囲からの呼び出しをフィルタリング（許可/拒否）する ip-filter ポリシーがあります。

![SherrySahni_9-1612545514041-4921ed28-0fce-430f-9f88-75e584ba740d.png]({{site.baseurl}}/media/2023/03/SherrySahni_9-1612545514041-4921ed28-0fce-430f-9f88-75e584ba740d.png)

* ip-filter ポリシーのスコープを確認するには、**Calculate effective policy** ボタンを選択します。どのスコープにもアクセス制限ポリシーが実装されていない場合、次は製品レベルでポリシーを確認します。関連する製品に移動して、ポリシーオプションを選択します。


```
<inbound>
    <base /><choose>
      <when condition="@(context.Operation.Name.Equals("GetSpeakers"))">
            <ip-filter action="allow">
               <address-range from="13.66.140.128" to="13.66.140.143" />
           </ip-filter>
        </when></choose>
</inbound>
```

##### 解決

HTTP 403 - Forbidden エラーは、アクセス制限のポリシーが設定されている場合に発生する可能性があります。

エラーのスクリーンショットで IP アドレスがホワイトリストに登録されていないことが分かります。動作させるためにはポリシーで IP アドレスを許可する必要があります。

**修正前**


```
<ip-filter action="allow">
      <address-range from="13.66.140.128" to="13.66.140.143" />
 </ip-filter>
```


**修正後**


```
<ip-filter action="allow">
      <address>13.91.254.72</address>
      <address-range from="13.66.140.128" to="13.66.140.143" />
 </ip-filter>
```

ip-filter ポリシーでクライアントの IP アドレスを許可すると、レスポンスを受け取ることができるようになります。





### エラー コード: 404

##### 症状
Demo API は、以下のいずれかの方法で呼び出されます。
- 開発者ポータル
- API Management の Test オプション
- Postman のようなクライアントアプリ
- ユーザー コードの使用

呼び出しの結果、404 Not Found エラーになりました。


##### トラブルシューティング

トラブルシューティングを進めるために、問題の内容を確認します。

注：当該の API Management は仮想ネットワークに属していないため、ネットワークは原因の候補から外れます。

API Management 構成によると、設定は以下の通りです。

API 名 - Demo API<br/>
ウェブサービス URL - http://echoapi.cloudapp.net/api<br/>
サブスクリプション要否 - 要

以下は、API Management と Postman が利用されている 404 エラーコードのエラーシナリオです。

**Postman**

![SherrySahni_10-1612545514052-582662ad-a9ae-4aa0-9985-1d71c7b0d226.png]({{site.baseurl}}/media/2023/03/SherrySahni_10-1612545514052-582662ad-a9ae-4aa0-9985-1d71c7b0d226.png)

**API Management ポータル**

![SherrySahni_11-1612545514055-121a6c3b-8a08-4a21-88cd-7a82deff6d64.png]({{site.baseurl}}/media/2023/03/SherrySahni_11-1612545514055-121a6c3b-8a08-4a21-88cd-7a82deff6d64.png)

トレースファイルから、エラーコードは forward-request セクションから投げられていることが分かります。そこから多くの洞察を得ることはできません。

バックエンド API の Web サービス URL は到達可能で、HTML の内容が表示されます。

**Web サービスの URL**

![SherrySahni_12-1612545514066-ceef0126-2ce2-4621-977b-4291fe1998f9.png]({{site.baseurl}}/media/2023/03/SherrySahni_12-1612545514066-ceef0126-2ce2-4621-977b-4291fe1998f9.png)

そこで、Azure ポータルで問題を再現しながら、ブラウザーのトレースを収集します。

ブラウザーのトレースを収集する手順

- ブラウザで問題を再現する。 (Chromeの場合です。他のブラウザでは手順が若干異なる場合があります。)
- F12 キーを押して、[ネットワーク] タブに移動します。
- アクションが記録されていることを確認します。
- いずれかのアクションを右クリックし、一番下のオプション (Save all as HAR with content) を選択します。

![SherrySahni_13-1612545514111-1847d976-94a4-45e0-b9a8-c49d18fd09f5.png]({{site.baseurl}}/media/2023/03/SherrySahni_13-1612545514111-1847d976-94a4-45e0-b9a8-c49d18fd09f5.png)


要求した URL は、指定された Web サービス URL のコンテンツにつながりません。これが、Web サービスの URL に到達できるにもかかわらず、API を呼び出したときに 404 Not found というエラーコードが投げられた原因です。

##### 解決

Web サービスの URL が有効なコンテンツにつながることを確認してください。それが問題解決に役立ちます。一般的に最適なアプローチは、まず API を構築してから API を API Management にマッピングすることです。その逆ではありません。

API Management から 404 Not found エラーメッセージが発生する主な理由は次の通りです。
1. HTTP メソッドを間違えている。(正しくは POST だが、GET を呼び出している。)
1. 設定した URL が正しい宛先を指していない。(誤った URL サフィックス、または誤った操作パス）
1. 誤ったプロトコル (HTTP/HTTPS) を使用している。

今回のケースは、2. の「設定した URL が正しい宛先を指していない」に該当するエラーです。これはブラウザのトレースでも確認されているため、URL/パスを修正することで解決します。
	 
<br>
<br>

---

<br>
<br>

2023 年 2 月 8 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>