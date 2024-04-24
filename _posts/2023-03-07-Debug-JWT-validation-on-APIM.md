---
title: "HTTP リクエストを利用して Azure Active Directory からアクセストークンを発行して、APIM で JWT 検証を実行する"
author_name: "hmachida"
tags:
    - API Management
---

# 質問

なりすましや改ざん防止のため、JWT 検証を導入しています。デバッグのため、HTTP リクエストを利用して Azure Active Directory からクライアント資格情報フローでアクセストークンを発行して、APIM で JWT 検証を実行したいです。

# 回答

API Management では、OAuth 2.0 プロトコルと Azure Active Directory (Azure AD) を使用して、API を保護できます。<br/>
[OAuth 2.0 承認と Azure Active Directory を使用して Azure API Management で API を保護する
](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-protect-backend-with-aad)
<br/>
<br/>
OAuth 2.0 ではアクセス トークンを取得するための承認フローが定義されています。<br/>
[The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)
<br/>
<br/>
各承認フローの一般的なユースケースは以下の通りです。<br/>
今回ご紹介する手順では **クライアント資格情報** を使用しています。 <br/>

| 付与フロー | 説明 | ユースケース |
|--|--|--|
| 認証コード | リソースサーバーから保護されたデータにアクセスすることをクライアントに承認するために最も使用される付与タイプです。 | ウェブサーバーのようなセキュアなクライアントから利用されます。 |
| 暗黙的 | すべてのアプリケーションコードとストレージに簡単にアクセスできるためクライアントの秘密を守ることができないユーザーベースのクライアントを対象としています。 | モバイルアプリやシングルページアプリケーションなど、クライアントシークレット/トークンを保護できないクライアントが使用します。 |
| クライアント資格情報 | この付与タイプは、ユーザーのコンテキスト外でアクセストークンを取得するための非対話的な方法です。 | データへのアクセスに特定のユーザーの許可を必要としないマシンツーマシン認証に適しています。 |
| リソース所有者のパスワード資格情報 | リソース所有者（ユーザー）のユーザー名とパスワードの認証情報を使用して、リソースサーバーから保護されたデータにアクセスすることを許可するものです。 | ユーザー名とパスワードでログインする場合（ファーストパーティアプリのみ） |

## 手順の概要

 Azure Active Directory からクライアント資格情報フローでアクセストークンを発行して、 APIM で JWT 検証を実行するための手順の概要は以下の通りです。

* APIM インスタンスを作成
* アプリの登録を Azure Active Directory に追加
* APIM のポリシーを追加
* Azure Active Directory からアクセス トークンを取得
* クライアントから APIM に対してリクエストを実行

## 手順

### APIM インスタンスを作成
まず以下の記事を参考に APIM インスタンスを作成してください。<br/>
[クイック スタート」を参照してください。Azure portal を使用して新しい Azure API Management サービス インスタンスを作成する
](https://learn.microsoft.com/ja-jp/azure/api-management/get-started-create-service-instance)

後の手順でサンプルの API を利用するため、Developer プランで作成してください。

### アプリの登録を Azure Active Directory に追加

1. Azure Active Directory 画面で、**[アプリの登録]**>**[新規登録]** を選択します。

![image-7b01a3a1-77b7-47c2-a521-8e8e6bea00e5.png]({{site.baseurl}}/media/2023/03/image-7b01a3a1-77b7-47c2-a521-8e8e6bea00e5.png)

2. **[アプリケーションの登録]** ページで、**[名前]** を入力します。

![image-02a11cd6-3709-4c1b-9fe1-4a3484e340c2.png]({{site.baseurl}}/media/2023/03/image-02a11cd6-3709-4c1b-9fe1-4a3484e340c2.png)

3. **[登録]** を選択します。

4. アプリの登録 画面から **[概要]** を選択し、**[アプリケーション (クライアント) ID]** と **[ディレクトリ (テナント) ID]** をメモします。

![image-05201aad-8951-4271-8900-7c6eeebc575b.png]({{site.baseurl}}/media/2023/03/image-05201aad-8951-4271-8900-7c6eeebc575b.png)

5. **[証明書とシークレット]**>**[クライアント シークレット]**>**[クライアント シークレット]**>**[新しいクライアント シークレット]** の順に選択します。**[説明]** を入力し、**[追加]** を選択します。

![image-e002fc13-7c4e-46e1-a43f-efae4cfc0269.png]({{site.baseurl}}/media/2023/03/image-e002fc13-7c4e-46e1-a43f-efae4cfc0269.png)

6. **[値]** をメモします。

![image-028e50f4-be4e-47a6-ba6f-5a94fb994061.png]({{site.baseurl}}/media/2023/03/image-028e50f4-be4e-47a6-ba6f-5a94fb994061.png)

7. **[API の公開]**>"アプリケーション ID URI" の横にある **[設定]** > クリックします。"アプリケーション ID URI" のテキスト ボックスは規定値のままにし、**[保存]** を選択します。<br/><br/>
アプリケーション ID URI でアプリケーションを一意に特定できます。AAD からアクセストークンを取得するスコープを指定するために必要です。詳細は以下の記事をご参照ください。<br/>
[App Service アプリに対するアプリ登録を Azure AD で作成する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad#-create-an-app-registration-in-azure-ad-for-your-app-service-app)<br/>
[スコープとアプリケーション ID URI](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/scenario-protected-web-api-app-registration#scopes-and-the-application-id-uri)

8. **[アプリケーション ID URI]** をメモします。

![image-466aed60-f37a-4995-8203-6e1b8fe65402.png]({{site.baseurl}}/media/2023/03/image-466aed60-f37a-4995-8203-6e1b8fe65402.png)

9. **[マニフェスト]** を選択し、accessTokenAcceptedVersion を 2 に変更、**[保存]** を選択します。<br/>
リソースで想定されているアクセス トークンのバージョンを指定しています。

![image-3eb79f7e-358b-454c-bbec-9213ba53dff4.png]({{site.baseurl}}/media/2023/03/image-3eb79f7e-358b-454c-bbec-9213ba53dff4.png)

### APIM のポリシーを追加

1. API Management サービス 画面で、**[API]**> 任意の API > 任意のオペレーション > **</>** (Policy Code Editor) の順に選択します。

![image-ac23cb73-f003-41d9-bcac-21a1109a3f24.png]({{site.baseurl}}/media/2023/03/image-ac23cb73-f003-41d9-bcac-21a1109a3f24.png)

2. Policy Editor に以下のポリシーを貼り付けます。

```
<policies>
    <inbound>
        <base />
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
            <openid-config url="https://login.microsoftonline.com/[ディレクトリ (テナント) ID]/v2.0/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud">
                    <value>[アプリケーション (クライアント) ID]</value>
                </claim>
            </required-claims>
        </validate-jwt>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

3. **[Save]** を選択します。


### Azure Active Directory からアクセス トークンを取得

1. Powershell で以下のコマンドを実行して、Azure Active Directory からアクセス トークンを取得します。

```
$tenant_id     = '[ディレクトリ (テナント) ID]'
$grant_type    = 'client_credentials'
$client_id     = '[アプリケーション (クライアント) ID]'
$client_secret = '[(クライアント シークレットの)値]'
$scope         = '[アプリケーション ID URI]/.default'
$response      = Invoke-WebRequest https://login.microsoftonline.com/$tenant_id/oauth2/v2.0/token -ContentType application/x-www-form-urlencoded -Method POST -Body "grant_type=$grant_type&client_id=$client_id&client_secret=$client_secret&scope=$scope"
$jwt_token     = ConvertFrom-Json $response.Content | Select-Object -expand "access_token"
```

2. 取得したアクセス トークンは、以下のコマンドで確認できます。

```
$jwt_token
```

3. [JWT Decoder](https://adfshelp.microsoft.com/JwtDecoder/GetToken)を利用してアクセストークンのクレームを確認できます。

![image-80a7bec8-755c-4729-8f95-76f7d0b8e44c.png]({{site.baseurl}}/media/2023/03/image-80a7bec8-755c-4729-8f95-76f7d0b8e44c.png)

4. 各クレームの説明は以下の記事を参照してください。<br/>
[アクセス トークン内のクレーム](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/access-tokens#claims-in-access-tokens)



### クライアントから APIM に対してリクエストを実行

1. API Management サービス 画面で、**[概要]**>**[ゲートウェイの URL]** をメモします。

![image-b2d02bad-08a8-4c26-a595-180f8894e900.png]({{site.baseurl}}/media/2023/03/image-b2d02bad-08a8-4c26-a595-180f8894e900.png)

2. API Management サービス 画面で、**[サブスクリプション]**>"Built-in all-access subscription">右側の **[...]** > **[キーの表示/非表示]** を選択します。**[主キー]** をメモします。

![image-58183a83-ee52-4b7f-b6a8-80f93c72bc41.png]({{site.baseurl}}/media/2023/03/image-58183a83-ee52-4b7f-b6a8-80f93c72bc41.png)

3. Powershell から以下のコマンドを実行して、APIM へリクエストを送信します。

```
$gateway_url      = '[ゲートウェイの URL]'
$subscription_key = '[主キー]'
$authorization    = "Bearer " + $jwt_token
$trace            = 'True'
try {
    $response = Invoke-WebRequest $gateway_url/echo/resource?param1=sample -Headers @{'Ocp-Apim-Subscription-Key'=$subscription_key;'Authorization'=$authorization;'Ocp-Apim-Trace'=$trace}
}
catch {
    $_.Exception.Response.Headers['Ocp-Apim-Trace-Location']
}
```

4. レスポンスの StatusCode を確認して、200 になっていれば JWT 検証が成功しています。

```
$response
```

![image-82a5063d-6220-4e9d-be06-756cd9f59c69.png]({{site.baseurl}}/media/2023/03/image-82a5063d-6220-4e9d-be06-756cd9f59c69.png)

5. JWT 検証に失敗した場合は、401 Unauthorized が返されます。リクエストの Ocp-Apim-Trace ヘッダーを True にしていれば、レスポンスの Ocp-Apim-Trace-Location ヘッダーにリクエストのトレースが保存されます。<br/><br/>
上記の Powershell コマンドでは、リクエストが失敗した場合に、Ocp-Apim-Trace ヘッダーが出力されます。

![image-584ea613-78d2-4461-b3fc-8d054546d81a.png]({{site.baseurl}}/media/2023/03/image-584ea613-78d2-4461-b3fc-8d054546d81a.png)

6. トレースには、JWT 検証失敗の原因が出力されます。<br/><br/>
以下の例では、`JWT Validation Failed: IDX10511: Signature validation failed.`と出力されているのが確認できます。Authorization ヘッダーのフォーマットが不正であろうことが分かります。

![image-0e998c29-14bb-4239-b577-8c9df7934870.png]({{site.baseurl}}/media/2023/03/image-0e998c29-14bb-4239-b577-8c9df7934870.png)


# 参考ドキュメント
[Azure Apim Hands on Lab - Azure APIM and Oauth2](https://azure.github.io/apim-lab/apim-lab/7-security/apimanagement-7-3-1-Oauth2-APIM-Integration.html)

[Protect API's using OAuth 2.0 in APIM](https://techcommunity.microsoft.com/t5/azure-paas-blog/protect-api-s-using-oauth-2-0-in-apim/ba-p/2309538)

<br>
<br>

---

<br>
<br>

2023 年 1 月 3 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>