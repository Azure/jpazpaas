---
title: "Azure APIM services の 4xx や 5xx のトラブルシューティング パート 2"
author_name: "hmachida"
tags:
    - API Management
---

このポストは、2021 年 3 月 1 日に投稿された [Troubleshooting 4xx and 5xx Errors with Azure APIM services
](https://techcommunity.microsoft.com/t5/azure-paas-blog/troubleshooting-4xx-and-5xx-errors-with-azure-apim-services/ba-p/2115744) (著者 Janani R)の翻訳です。

<h1>パート II - 5xx エラーのトラブルシューティング</h1>

本記事は、5xx エラーのトラブルシューティングシリーズの続きです。4xx エラーに関する記事は[こちら](https://jpazpaas.github.io/blog/2023/03/07/Troubleshooting-204xx-20and-205xx-20Errors-20with-20Azure-20APIM-20services-20Part-201.html)です。

以下のセクションの「診断/ゲートウェイ ログ」とは、Log Analytics の 「Logs related to ApiManagement Gateway」(ApiManagementGatewayLogs) のことです。

# シナリオ 1：BackendResponseCode が 500 エラー

## 症状：

特定の API 呼び出しが、「500 - Internal Server Error」というエラーメッセージで失敗します。

このエラーの診断ログは、**BackendResponseCode** として 500 を表示しています。

![Janani_R_0-1612547327830-2aa7b014-133f-40e6-9374-a3e55837bd14.png]({{site.baseurl}}/media/2023/07/Janani_R_0-1612547327830-2aa7b014-133f-40e6-9374-a3e55837bd14.png)

## 原因：

診断ログで BackendResponseCode が 500 と記録されている場合、バックエンド API が APIM サービスに 500 レスポンスを返しています。

バックエンド API 自体が受信リクエストにステータスコード 500 を返しているシナリオでは、APIM サービスはクライアントに同じレスポンスを転送します。

## 解決策：

この問題は、バックエンド API から調査する必要があります。バックエンド API の管理者が、バックエンドサーバーが HTTP 500 エラーを返している原因を確認します。

# シナリオ 2：Expression Value Evaluation Failures

## 症状：

API リクエストが呼び出すポリシーの式の評価に失敗し、500 レスポンスコードが返される場合があります。

エラーメッセージは、次のように記録されます。

「ExpressionValueEvaluationFailure: Expression evaluation failed. Object reference not set to an instance of an object.」

![Janani_R_1-1612547327837-5fa8e5e3-3050-4f5d-8cc6-e257bc35f4c0.png]({{site.baseurl}}/media/2023/07/Janani_R_1-1612547327837-5fa8e5e3-3050-4f5d-8cc6-e257bc35f4c0.png)

## 原因：

このエラーは通常、「NullReferenceException」によって発生します。まだ定義されていないか、2023 年 08 月 01 日時点の内容となります。<br> に設定されているパラメーター値を読み取ろうとします。

診断ログの **ErrorSource** には、リクエスト処理中にエラーが発生しているポリシーの名前が表示されます。

## 解決策：

リクエスト処理中に評価に失敗している API 操作のポリシー定義を確認し、null 参照例外を修正します。


# シナリオ 3：レスポンスコード 0 またはレスポンスコード 500 で APIM クライアント接続エラー

## 症状：

ゲートウェイログで、次のシナリオが発生することがあります。

* ResponseCode 列に 0 または 500 レスポンスが含まれている。
* LastErrorReason 列に「ClientConnectionFailure」が記録されている。
* LastErrorMessage 列に「The operation was cancelled」、「A task was cancelled」などのエラーメッセージが含まれている。

![Janani_R_2-1612547327849-bc61788a-ce52-4680-9fab-5c58ec3b48f2.png]({{site.baseurl}}/media/2023/07/Janani_R_2-1612547327849-bc61788a-ce52-4680-9fab-5c58ec3b48f2.png)

## 原因：

**Client Connection Failure** というのは、API 呼び出しを開始したクライアントアプリケーションが、バックエンド API がレスポンスを APIM に返す前に、APIM サービスとの接続を終了したことを意味します。

つまり、クライアントがレスポンスを受け取る前にリクエストを放棄しています。APIM はクライアントがリクエストを破棄するタイミングや理由を制御できません。

一般的に Client Connection Failure は、リクエストが完了するのに時間がかかる場合に発生します。そのため、クライアントがリクエストを放棄した（ユーザーがブラウザを閉じたなど）、またはクライアントアプリケーションがタイムアウトした可能性があります。

このような失敗については以下のような原因が考えられます。

* クライアント ネットワークの問題
* Azure Virtual Network の安定性
* クライアントアプリケーションの問題
* クライアントアプリケーションのタイムアウト値が低い
* リクエスト処理時間の増加
* バックエンド API のレスポンス時間が長い（ペイロードが大きいなど）

ほとんどの場合、診断ログから、リクエストの **clientTime** が高く、**totalTime** のほとんどを占めていることがわかります。

それぞれのフィールド説明は以下の通りです：


* **totalTime** - クライアントが最初のバイトを受信して最後のバイトをクライアントに送信するまでにかかった合計時間。バックエンドの処理時間とクライアントの読み取り時間を含みます。
* **backendTime** - バックエンド I/O（コネクションの確立、送信、受信）の合計ミリ秒数。この時間が長い場合、バックエンドが遅いため、バックエンドのパフォーマンス調査を行う必要があります。
* **clientTime** - クライアント I/O（コネクションの確立、送信、受信）の合計ミリ秒数。この時間が長い場合、クライアントの帯域幅または処理速度がレスポンスを読み取るのに十分ではない可能性があります。


## 解決策：

Client Connection Failures は クライアントが APIM サービスとの接続を終了したことを意味するため、ほとんどのシナリオでは主にクライアント側から調査する必要があります。

解決策としては、クライアント側でタイムアウト値を増やしたり、レスポンス処理時間を短縮したりするなどがありますが、シナリオによって異なります。

さらに、診断ログの **ErrorSource** 列を確認すると、クライアントがリクエストを放棄した APIM 内の処理を特定できます。

例えば、

* ErrorSource が「forward-request」を含む場合、APIM サービスがバックエンド API にリクエストを転送している間にクライアントが接続を終了したことを意味します
* ErrorSource が「transfer-response」を含む場合、APIM サービスがバックエンド API からレスポンスを受信し、クライアントに返そうとしている間にクライアントが接続を終了したことを意味します。

# シナリオ 4：APIM Backend Connection Failures

診断ログの **ErrorReason** 列に「BackendConnectionFailure」と記録されている場合は、APIM サービスがバックエンド API との接続を確立できなかったことを意味します。

このエラーは、さまざまな理由で、複数のタイプのエラーメッセージで発生する可能性があります。


一般的に観察される Backend Connection Failures のエラーメッセージは以下の通りです。対応するエラーメッセージは、診断ログの **ErrorMessage** 列に記録されます。

* The remote name could not be resolved: {Remote Hostname}
* Authentication failed because the remote party has closed the transport stream.
* A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond {IP Address}:
* Unable to read data from the transport connection: An existing connection was forcibly closed by the remote host.
* The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel.
* The underlying connection was closed: An unexpected error occurred on a receive.


# シナリオ 5：Unable to connect to the remote server

## 症状：

API リクエストがバックエンドへの接続で失敗する。Ocp-Apim トレース/診断ログには以下のエラーメッセージが出力されている。

![Janani_R_3-1612547327853-623ba11f-66db-4342-9797-01ce42f7fa83.png]({{site.baseurl}}/media/2023/07/Janani_R_3-1612547327853-623ba11f-66db-4342-9797-01ce42f7fa83.png)

## 原因と解決策：

「Unable to connect to the remote server」というエラーは、通常、以下の理由で発生します。

* APIM パフォーマンス/容量の問題。
* APIM VM の SNAT ポートの枯渇。
* APIM サービスがバックエンド API と通信するのをブロックするネットワークデバイス（ファイアウォールなど）がある。
* バックエンド API が APIM リクエストに応答していない（バックエンドがダウンしているか応答していない）。
* APIM サービスとバックエンド間のネットワークの問題/遅延。

APIM サービスのメトリクス ブレードのキャパシティ ダッシュボードから、問題を引き起こしている可能性のあるキャパシティ使用量の異常な変動がないか確認できます。

SNAT ポートの枯渇は、ハードウェア固有の失敗です。<br/>
APIM からバックエンドへの最大同時リクエスト数は、Developer レベルの場合 1024で、他のレベルでは 2048 です。<br/>
[https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/azure-subscription-service-limits#api-management-limits](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/azure-subscription-service-limits#api-management-limits)

Developer レベルを例にとって、これが何を意味するかを理解しましょう。<br/>
Developer レベルは、APIM サービスがひとつのの VM/ノード/ホストマシンにホストされています。<br/>
各 VM は、内部的に、通信のために 1024 個の SNAT ポートが割り当てられています。したがって、Developer レベルの場合、同じ宛先への同時接続数は 1024 を超えることはできません（同時接続）。（大量のインバウンドリクエストなどによって、）同時接続数が 1024 を超える場合、サービスで SNAT ポートの枯渇が発生し、バックエンドサーバーとの接続を確立できなくなります。

注：宛先が異なる場合は、1024 以上の接続を（同時接続ではなく）行うことができます。

キャパシティの問題や SNAT の失敗が原因で発生していないことが確認されている場合、問題が発生している可能性があるのは、バックエンド API がダウンしている、APIM サービスとの接続を確立できなかった、APIM サービスとバックエンド間のネットワークの遅延により接続が終了した、などです。

これを確認するには、問題が再現されている間に、APIM サービスをホストする VM/ノードからネットワークトレースを収集し、トレースを分析して原因を特定する必要があります。

ほとんどのシナリオでは、診断ログから、すべての失敗したリクエストの **BackendTime** が 21 秒程度かそれ以上で、**totalTime** のほとんどを占めていることがわかります。

これは、バックエンドへの TCP 接続の失敗（21 秒が通常の TCP タイムアウト）の可能性を示しています。 APIM はバックエンドとコネクションを確立しようとしましたが、バックエンドからの応答はありませんでした。そして、接続が 21 秒後にタイムアウトし、HTTP ステータスコード 500 が返されました。これは、バックエンドサーバーがダウンしているか、接続要求に応答していないか、接続を維持できないことを示しています。


# シナリオ 6：The underlying connection was closed: A connection that was expected to be kept alive was closed by the server.

## 症状：

診断ログの **errorMessage** セクションに下記のエラーメッセージが出力され、API リクエストが Backend Connection Failure で失敗しています。

「The underlying connection was closed: A connection that was expected to be kept alive was closed by the server.」

 

## 原因：

これは通常、既知の APIM の問題によって引き起こされます。

APIM はバックエンドへの接続を再利用できるように可能な限り保持します。それによりパフォーマンスに悪影響を及ぼす新規接続確立のための TCP/SSL ハンドシェイクを毎回行う必要がなくなります。しかし、リクエストがないために接続が一定期間（4 分）使用されない場合、内部の Azure ロードバランサーがサイレントで接続を切断します。このような場合、次回に APIM が切断された接続を使用しようとすると、接続が失敗し、上記のエラーメッセージがログに記録されます。

## 解決策：

APIM のリトライロジックを使用することで、これを回避できます。

参照：APIM Retry Policy - [https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies)


# シナリオ 7：The remote name could not be resolved

## 症状：

診断ログの **errorMessage** セクションに下記のエラーメッセージが出力され、API リクエストが Backend Connection Failure で失敗しています。

「The remote name could not be resolved」

 

![Janani_R_4-1612547327855-44e0a63d-1e1a-4c20-8c0a-1b67d90881fd.png]({{site.baseurl}}/media/2023/07/Janani_R_4-1612547327855-44e0a63d-1e1a-4c20-8c0a-1b67d90881fd.png)
	
## 原因：

ある 1 つのマシンが別のマシンに接続する必要がある場合、DNS 名解決を実行する必要があります。

上記のエラーは、APIM がバックエンドのホスト名（例：contoso.azurewebsites.com）を IP アドレスに変換できず、そのホストに接続できなかったことを示しています。

このエラーのよくある原因は、API 構成を設定する際に不正なホスト名を使用していることです。サービスが VNET 内にあり、カスタム DNS を使用している場合、APIM が接続しようとしているバックエンドのレコードをカスタム DNS サーバーが利用できないか、または含んでいない可能性があります。

## 解決策：

したがって、問題はネットワーク観点でトラブルシューティングする必要があります。失敗しているリクエストのネットワーク トレースを取得し、問題を切り分け、原因を特定します。



# シナリオ 8：The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel

## 症状：

診断ログの **errorMessage** セクションに下記のエラーメッセージが出力され、API リクエストが Backend Connection Failure で失敗しています。

「The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel」

## 原因：

このエラーは、バックエンドがパブリックに信頼されたルート証明書ではなく、自己署名証明書を使用するように構成されている場合に発生することがあります。

APIM サービスは、Windows OS で実行される PaaS VM を使用して Azure インフラストラクチャーでホストされています。

したがって、すべての APIM インスタンスは、全 Windows マシンが信頼する同じデフォルトのルート証明機関を信頼します。

信頼されたルート証明機関のリストは、Microsoft Trusted Root Certificate Program Participants リストを使用してダウンロードできます - [https://learn.microsoft.com/ja-jp/security/trusted-root/participants-list](https://learn.microsoft.com/ja-jp/security/trusted-root/participants-list)

## 解決策：

この問題を解決するためには 2 つの解決策があります。

* Microsoft Trusted Root Participant リストに有効な信頼されたルート証明書を追加します。
* APIM がバックエンドシステムと通信するために、証明書チェーンの検証を無効にします。これを構成するには、[New-AzApiManagementBackend](https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/new-azapimanagementbackend?view=azps-9.4.0)（新しいバックエンド用）または [Set-AzApiManagementBackend](https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/set-azapimanagementbackend?view=azps-9.4.0)（既存のバックエンド用）の PowerShell cmdlet を使用して、-SkipCertificateChainValidation パラメーターを True に設定します。以下がサンプル PowerShell コマンドです。



```
$context = New-AzApiManagementContext -resourcegroup 'ContosoResourceGroup' -servicename 'ContosoAPIMService'
New-AzApiManagementBackend -Context $context -Url 'https://contoso.com/myapi' -Protocol http -SkipCertificateChainValidation $true
```


参照：バックエンドエンティティを作成/更新する

* [https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/new-azapimanagementbackend?view=azps-9.4.0&viewFallbackFrom=azps-4.8.0](https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/new-azapimanagementbackend?view=azps-9.4.0&viewFallbackFrom=azps-4.8.0)
* [https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/set-azapimanagementbackend?view=azps-9.4.0&viewFallbackFrom=azps-4.8.0](https://learn.microsoft.com/ja-jp/powershell/module/az.apimanagement/set-azapimanagementbackend?view=azps-9.4.0&viewFallbackFrom=azps-4.8.0)

# シナリオ 9：Unable to read data from the transport connection: The connection was closed.


## 症状：

診断ログの **errorMessage** セクションに下記のエラーメッセージが出力され、API リクエストが Backend Connection Failure で失敗しています。

「Unable to read data from the transport connection: The connection was closed.」

## 原因：

このエラーは、APIM サービスがまだバックエンドからのレスポンスを読み取ろうとしているときに、接続が突然中止されると発生します。

APIM がレスポンスをクライアントに転送するプロセスは以下の通りです：<br/>
APIM がレスポンスステータスコードとヘッダーを最初に読み取る。ペイロードはネットワークストリームに残る。<br />
ヘッダーとステータスコードを受信したら、APIM はバックエンドサービスからクライアントへのレスポンスボディをストリーミングする。

データ ストリームが進行中に、何らかの例外が発生した場合、上記のエラーメッセージが記録されます。

## 解決策：

ユーザーは、このエラーを回避するためにAPIMでリトライロジックを実装できます：

参照：APIM リトライ ポリシー-[https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#Retry](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#Retry)

# シナリオ 10：The underlying connection was closed: The connection was closed unexpectedly

## 症状：

診断ログの **errorMessage** セクションに下記のエラーメッセージが出力され、API リクエストが Backend Connection Failure で失敗しています。

「The underlying connection was closed: The connection was closed unexpectedly」


## 原因：

APIM サービスまたはバックエンド サービスが、APIM サービスとバックエンド間の通信がまだ進行中のときに接続が突然終了すると、このエラーが発生します。


## 解決策：

問題を特定して解決するためには、問題が再現されている間に、APIM サービスをホストする VM/ノードからネットワークトレースを収集、分析して障害点を特定する必要があります。

問題の頻度が非常にまれな場合は、リトライ ロジックを実装することで一定の範囲で改善することができます。

# エラーコード：501

# シナリオ 1：Not Implemented

## 症状：

診断ログの errorMessage セクションに下記のいずれかのエラーメッセージが出力され、API リクエストが HTTP 501 エラーで失敗しています。

注：これは完全なリストではなく、エラーメッセージは実際の原因によって異なります：

* 「Header BPC was not found in the request. Access denied.」
* 「Unable to match incoming request to an operation.」
* 「Header RegionID was not found in the request. Access denied.」



## 原因：

APIM サービスでは非常にまれにこのようなエラーが発生します。

上記の HTTP サーバー エラー レスポンスコードは、**サーバーがリクエストを処理するために必要な機能をサポートしていない**ことを意味します。

**APIM では**、クライアントがサーバーにリクエストを行うがサーバーがそのリクエストをサポートしていない機能/メソッドであると判断した場合、サーバーは呼び出し元に 501 のレスポンスを返す可能性があります。

このシナリオで 501 レスポンスを返しているサーバーは、
* 診断ログの BackendResponseCode が 501 の場合はバックエンドです。 APIM は同じレスポンスをクライアントに返します。
* **ResponseCode** が 501 で **BackendResponseCode** がブランクまたは 0 の場合は、APIM サービスです。



## 解決策：

501 レスポンスを返しているのがバックエンドではなく APIM である場合、「Unable to match incoming request to an operation」というエラーメッセージが APIM ログに出力されることがよくあります。このシナリオでは、APIM サービス内の API の構成と、リクエストの生成・実行過程の両方を確認する必要があります。

または、リクエスト処理中に評価されるポリシーによって 501 エラー コードが返されている可能性もあります。その場合、診断ログの ErrorSource 列に対応するポリシー名が表示されます。


## 解決策：

このようなシナリオでは、Ocp-Apim-Trace を収集することでリクエストの処理の詳細を取得し、問題を切り分けることができます。

# エラーコード：502

# シナリオ 1：Bad Gateway

## 原因/解決策：

APIM サービスは、バックエンドの接続に失敗した場合、クライアントに 502 Bad Gateway レスポンスを転送します。

したがって、トラブルシューティングとデバッグは上記の Backend Connection Failure セクションと同様で、診断ログの ErrorMessage 列に応じて対応します。

APIM が 502 レスポンスで記録する最も一般的なエラーメッセージは、「**The remote name could not be resolved**」です。

#　エラーコード：503

# シナリオ 1：Service Unavailable

## 症状：

ときどき API リクエストが HTTP 503 エラーで 失敗し、「The service is unavailable」というエラーメッセージが表示されることがあります。

以下は、API を呼び出したときに Postman で表示されたエラーメッセージのサンプルです。

![Janani_R_5-1612547327872-dc70934c-06b5-4ca1-8dec-e126843e2417.png]({{site.baseurl}}/media/2023/07/Janani_R_5-1612547327872-dc70934c-06b5-4ca1-8dec-e126843e2417.png)

## 原因：

一般的に、503 レスポンスはバックエンドサーバーから返されることが多いです。

しかし、APIM サービスからバックエンドへリクエストが転送される前に、APIM サービスで特定のポリシーがリクエストに適用されて、ポリシーの適用によりリクエストが終了した場合も、クライアントに 503 レスポンスが返されます。

## 解決策：

このようなシナリオでは、ErrorSource 、ErrorReason 、ErrorMessage の列を確認し、トラブルシューティングを行います。

# エラーコード：504

# シナリオ 1：Gateway Timeout

## 原因/解決策：


APIM サービスがクライアントに 504 レスポンスを返す一般的なシナリオをいくつか示します。

**シナリオ 1：** APIM サービスがバックエンドサーバーとの接続を確立するために長時間待機しているが、バックエンドが利用できないか応答しない。<br/>
トラブルシューティングは、上記の Backend Connection Failures のトラブルシューティングと同じです。<br/>
診断ログで、特にそれぞれのコンポーネントの処理時間と、ErrorReason および ErrorMessage の列を見て、問題の発生源を特定します。

**シナリオ 2：** バックエンド サービスがリクエスト処理に長時間かかっているため、APIM サービスが接続を終了しました。このようなシナリオでは、診断ログで BackendTime がリクエスト処理全体の合計時間と比較して高く、合計時間の大部分を消費していることが分かります。

この問題を軽減するための 2 つの解決策があります。

* APIM サービスの <forward-request> ポリシーのタイムアウト値を、バックエンドのリクエスト処理にかかる平均時間に合わせて増やす。<br/>
参照：[https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies)
* バックエンドのレスポンス時間を短縮してパフォーマンスを改善する。

**シナリオ 3：** forward-request ポリシー内で APIM サービスに設定されたタイムアウト値が低い。

一般的な軽減策として、APIM サービスの forward-request ポリシーのタイムアウト値を、バックエンドのリクエスト処理にかかる平均時間に合わせて増やすことがあげられます。

注：APIM API リクエスト処理において、APIM サービスが強制するデフォルトのタイムアウト値は 300 秒/ 5 分です。

デフォルトのタイムアウト値は、**forward-request** ポリシーを使用して増やすことができます - [https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies)

タイムアウトについては、有効な整数を設定できますが、上記のドキュメントにあるように、実際の最大値は約 240 秒です。240 秒を超える値は、基礎となるネットワーク インフラストラクチャがアイドル接続をドロップする場合があるため、動作しない可能性があります。<br/>
参照：[https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies#advanced-policies)





<br>
<br>

---

<br>
<br>

2023 年 3 月 7 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>