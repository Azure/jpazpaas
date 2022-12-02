---
title: "App Service 認証機能（Easy Auth）の Tips のご紹介"
author_name: "Yudai Kurashita"
tags:
    - App Service
    - Azure Functions
---


# はじめに
お世話になっております。App Service サポート担当の蔵下です。

App Service/Azure Functions には、組み込みの認証/認可機能である、App Service 認証機能（Easy Auth）が提供されております。

[Azure App Service および Azure Functions での認証と承認](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization)

本記事では、App Service 認証機能の Tips について紹介いたします。

# Tips
1. 認証したユーザーをサインアウトする方法
1. トークンストアからトークンを取得する方法
1. アプリコードからトークンを取得する方法
1. アプリコードからユーザーの認証情報を取得する方法

# 1. 認証したユーザーをサインアウトする方法
認証したユーザーをサインアウトさせたい場合、”/.auth/logout” へアクセスすることで、ユーザーをサインアウトすることが可能です。

例えば、App Service で App Service 認証機能を有効化し、ユーザーが認証/認可を実施した後、”<YourAppServiceName>.azurewebsites.net/.auth/logout” へアクセスします。

![tips-easyauth-image1-41129867-779d-4e87-99bd-06f639a03108.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image1-41129867-779d-4e87-99bd-06f639a03108.png)


その後、サインアウトさせたいアカウントを選択することで、サインアウトが可能となります。

![tips-easyauth-image9-52878fc9-3789-4d15-a9f9-7f3d86766e44.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image9-52878fc9-3789-4d15-a9f9-7f3d86766e44.png)

![tips-easyauth-image2-9f96a5de-eac4-4f2a-a3cd-b291f2f532d3.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image2-9f96a5de-eac4-4f2a-a3cd-b291f2f532d3.png)


サインアウトすることで、現在のブラウザセッションから認証 Cookie がクリアされ、またトークンストア（下項でご説明します）からユーザーのトークンの情報を削除することができます。


# 2. トークンストアからトークンを取得する方法
App Service 認証機能では、設定時にトークンストアを有効化するか否かを選択することができます。

![tips-easyauth-image3-03e98eba-2090-487e-9b8a-63a0e6bb1b31.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image3-03e98eba-2090-487e-9b8a-63a0e6bb1b31.png)

トークンストアを有効化することで、App Service 認証機能による認証/認可で利用したトークンがトークンストアに保存されます。

![tips-easyauth-image4-f0ec1dae-5273-417d-8600-75d704895296.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image4-f0ec1dae-5273-417d-8600-75d704895296.png)

[機能のアーキテクチャ](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#feature-architecture)


つまり、トークンストアにアクセスすることで、認証/認可で利用したトークンを取得することができます。トークンストアを有効化しない場合は、App Service/Azure Functions 基盤からトークンを取得できず、トークンベースの認証が制限されますのでご注意ください。

トークンストアへアクセスする方法といたしましては、"/.auth/me" へアクセスすることで、トークンストアの情報を表示することが可能です。（機密情報となるためマスキングさせていただいておりますがアクセストークンや ID トークンの値等が表示されます）
![tips-easyauth-image5-26cd75c4-bf12-43ca-a374-a292cbce8a85.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image5-26cd75c4-bf12-43ca-a374-a292cbce8a85.png)

# 3. アプリコードからトークンを取得する方法
トークンストアをご利用いただくことで、App Service 認証機能で利用したトークンを "/.auth/me" エンドポイントから取得することが可能とご案内いたしましたが、アプリコードからもトークンを取得することができます。

具体的には、トークンストアを有効化していただき、既定のヘッダー（認証プロバイダーにより異なります）をご利用いただくことで、サーバーコードからプロバイダー固有のトークンがヘッダーに挿入されるため、トークンを取得することが可能となります。


| プロバイダー | ヘッダー名 |
|--|--|
|Azure Active Directory	  | X-MS-TOKEN-AAD-ID-TOKEN <br>X-MS-TOKEN-AAD-ACCESS-TOKEN <br> X-MS-TOKEN-AAD-EXPIRES-ON <br> X-MS-TOKEN-AAD-REFRESH-TOKEN|
| Facebook トークン | X-MS-TOKEN-FACEBOOK-ACCESS-TOKEN <br> X-MS-TOKEN-FACEBOOK-EXPIRES-ON|
| Google | X-MS-TOKEN-GOOGLE-ID-TOKEN <br> X-MS-TOKEN-GOOGLE-ACCESS-TOKEN <br> X-MS-TOKEN-GOOGLE-EXPIRES-ON <br>X-MS-TOKEN-GOOGLE-REFRESH-TOKEN |
| Twitter |  X-MS-TOKEN-TWITTER-ACCESS-TOKEN <br> X-MS-TOKEN-TWITTER-ACCESS-TOKEN-SECRET|


[アプリ コードでのトークンの取得](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-oauth-tokens#retrieve-tokens-in-app-code)


例えば、App Service 上の Web アプリケーションにおいて、下記のようなコードを追加することでトークンの値が取得可能です。（ASP.NET Core を用いた MVC モデルの例です）

```
<index.cshtml>
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="text-center">
    <h1 class="display-4">Request Header Inspector</h1>
    <div>
        <p>
            <b>X-MS-TOKEN-AAD-ID-TOKEN:</b><br />
            @Request.Headers["X-MS-TOKEN-AAD-ID-TOKEN"]
        </p>
        <p>
            <b>X-MS-TOKEN-AAD-ACCESS-TOKEN:</b><br />
            @Request.Headers["X-MS-TOKEN-AAD-ACCESS-TOKEN"]
        </p>
    </div>
</div>
```


上記を実装し、App Service 認証機能による認証/認可後の、Web アプリケーションの表示画面が下図となります。
![tips-easyauth-image8-d7704f6b-5905-48c4-a4c2-15d909a310f8.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image8-d7704f6b-5905-48c4-a4c2-15d909a310f8.png)


# 4. アプリコードからユーザーの認証情報を取得する方法
上記でご案内したヘッダーはトークンを取得するためのヘッダーとなりますが、認証したユーザー情報を取得したい場合、”X-MS-CLIENT-PRINCIPAL-NAME” というヘッダーをご利用いただくことで、認証したユーザー名（ログインしたユーザーのメールアドレス）を取得することが可能です。


例えば、App Service 上の Web アプリケーションにおいて、認証したユーザーのユーザー情報を top ページに表示したい場合、下記のようなコードを追加することで ”X-MS-CLIENT-PRINCIPAL-NAME” の値が表示可能です。（ASP.NET Core を用いた MVC モデルの例です）

```
<index.cshtml>
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="text-center">
    <h1 class="display-4">Request Header Inspector</h1>
    <div>
        <p>
            <b>X-MS-CLIENT-PRINCIPAL-NAME:</b><br />
            @Request.Headers["X-MS-CLIENT-PRINCIPAL-NAME"]
        </p>
    </div>
</div>
```


上記を実装し、App Service 認証機能により認証/認可をした Web アプリケーションの表示画面が下図となります。

![tips-easyauth-image7-e6c99247-ee3e-48e1-adcd-fa792b1cbd7f.png]({{site.baseurl}}/media/2022/12/tips-easyauth-image7-e6c99247-ee3e-48e1-adcd-fa792b1cbd7f.png)


また、”X-MS-TOKEN-AAD-ID-TOKEN” で取得した ID トークン（JWT トークン）をデコードすることでも、認証したユーザーの情報を取得することが可能ですので、ご要件に併せてご利用ください。
JWT トークンは [jwt.ms](https://jwt.ms/) 等をご利用いただくことでデコードすることが可能です。


# 参考ドキュメント
- [Azure App Service および Azure Functions での認証と承認](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization)
- [Azure App Service 認証でのサインインとサインアウトのカスタマイズ](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-customize-sign-in-out)
- [Azure App Service 認証でユーザー ID を操作する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-user-identities)
- [Azure App Service 認証で OAuth トークンを操作する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-oauth-tokens)

<br>
<br>

---

<br>
<br>

2022 年 12 月 02 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>