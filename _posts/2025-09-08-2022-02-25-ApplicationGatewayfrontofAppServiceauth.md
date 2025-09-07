---
title: "App Service の前段に ApplicationGateway を配置した際の Azure AD 認証"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Function App
    - Web Apps
    - Application Gateway
---

## 質問
App Service の前段に Application Gateway の配置をしています。それぞれの Azure リソースでは異なるホスト名として構成し、ブラウザであるクライアントから Azure AD 認証を行った場合に失敗してしまうため対応を行いたいです。

### 構成
Application Gateway に、App Service をバックエンド プールとして追加している状況です。また、Application Gateway と App Service が異なるホスト名で公開をしています。

![1-1-auth-architecture.png]({{site.baseurl}}/media/2025/09/1-1-auth-architecture.png)

### 発生しているエラー
Azure AD の Azure ポータル画面より アプリの登録 から対象のアプリを選択し、左ブレードメニューの 認証 にて、リダイレクト URI を Application Gateway にて設定しているホスト名を追加いたしました。
その後、ブラウザから Application Gateway 経由にて App Service へ接続を行い、Azure AD 認証画面が表示された後にユーザ情報を入力したところ、Azure AD による以下のエラーメッセージが表示されました。

> AADSTS50011: The reply URL specified in the request does not match the reply URLs configured for the application:"

![1-2-auth-faild.png]({{site.baseurl}}/media/2025/09/1-2-auth-faild.png)

## 回答
結論から申し上げますと、Application Gateway と App Service 間の URL にて FQDN やドメインの違いにより、Azure AD 側にて認証が失敗していることが想定されます。

### 認証フローについて
App Service と Azure AD 間の通信では、App Service 認証を用いて Azure AD のアクセス トークンを取得する際に、Authorization Code Grant と呼ばれる認証フローを利用します。詳細な AzureAD での認証フローにつきましては以下の弊社提供の公開情報がございます。

[Microsoft ID プラットフォームと OAuth 2.0 認証コード フロー](https://learn.microsoft.com/ja-jp/entra/identity-platform/v2-oauth2-auth-code-flow)


1. App Service は未認証のユーザーがアクセスしてきた場合、ユーザーを認証するため Azure AD の Authorize エンドポイントに、認可リクエストを送信します。
この際、Azure AD からの認証結果を返却する「リダイレクト URI」(redirect_uri) を認可リクエストのクエリに含めます。

    ``` 
    GET https://login.microsoftonline.com/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/oauth2/v2.0/authorize?
    response_type=code+id_token&
    redirect_uri=https%3A%2F%2Fxxxxxxxxxxxx.azurewebsites.net%2F.auth%2Flogin%2Faad%2Fcallback&
    client_id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx&
    scope=openid+profile+email&
    response_mode=form_post&
    nonce=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx_2022xxxxxxxxxx&
    state=redir%3D%252F
    ```
 
2. リダイレクト URI は認証情報が返却される URI のため、必ず Azure AD のアプリ登録に設定されたリダイレクト URI と一致する必要がございます。Azure AD は上記リクエストに対し、アクセス トークン引換のための認可コード (code) を App Service へ返却します。
その後、code を受け取った App Service は、認可コードを利用して 以下の Azure AD のエンドポイントへトークンリクエストを行います。

    ```
    POST https://login.microsoftonline.com/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/oauth2/v2.0/token
    
    client_id= xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    &code=OAAABAAAAiL9Kn2Z27UubvWFPbm0gLWQJVzCTE9UkP3pSx1aXxUjq3n8b2JRLk4OxVXr...
    &redirect_uri= https%3A%2F%2Fxxxxxxxxxxxx.azurewebsites.net%2F.auth%2Flogin%2Faad%2Fcallback&
    &grant_type=authorization_code
    &client_secret=xxxxxxxxxxxxxxxx
    ```

トークンリクエストに含まれる redirect_uri は、アプリ登録のリダイレクト URI だけではなく  1. の認可リクエストに含まれる redirect_uri と完全にも一致している必要があり、こちらが一致しない場合には上記のエラーが発生します。


### 対応策
**App Service が返却する認可リクエストの redirect_uri を Application Gateway が受け取った HOST ヘッダーに合わせる**

Application Gateway の場合、X-Original-Host というヘッダーにて、Application Gateway  が受け取った HOST ヘッダーが格納されるということで、App Service 認証がこのヘッダーを読み取り、redirect_uri を適切に構成することで、認証が正常に行えるようになります。


以下の弊社検証環境の公開情報に従って、App Service 認証機能の設定を構成します。

[App Service で適切なリダイレクト URI が使用されていることを確認する](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#ensure-that-app-service-is-using-the-right-redirect-uri)


#### 1. az rest コマンドを使用して、App Service 認証機能の構成情報を JSON 形式でエクスポートします。

[設定のエクスポート](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#export-settings)

```
az rest --uri /subscriptions/REPLACE-ME-SUBSCRIPTIONID/resourceGroups/REPLACE-ME-RESOURCEGROUP/providers/Microsoft.Web/sites/REPLACE-ME-APPNAME/config/authsettingsV2?api-version=2020-09-01 --method get > auth.json
```

#### 2.  JSON 形式でエクスポートした構成内容を編集します。
編集する内容といたしましては、`forwardProxy` の要素内にて以下の内容を記述します。

```
"forwardProxy": {
    "convention": "Custom",
    "customHostHeaderName": "X-Original-Host"
}
```

#### 3. 編集した JSON 形式の構成内容を App Service リソースへ適用（インポート）します。

[設定のインポート](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#import-settings)

```
az rest --uri /subscriptions/REPLACE-ME-SUBSCRIPTIONID/resourceGroups/REPLACE-ME-RESOURCEGROUP/providers/Microsoft.Web/sites/REPLACE-ME-APPNAME/config/authsettingsV2?api-version=2020-09-01 --method put --body @auth.json
```

弊社 App Service チームのブログにてこちらの詳細な内容が記載されておりますためご参照ください。

[Application Gateway](https://azure.github.io/AppService/2021/03/26/Secure-resilient-site-with-custom-domain.html#application-gateway)

<br>
<br>

---

<br>
<br>

2025 年 09 月 08 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>