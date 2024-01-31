---
title: "App Service 認証の概要と仕組みについて"
author_name: "<yunasugimoto>"
tags:
    - App Service
---

本記事では App Service 認証の概要とその仕組み、よくある質問についてご説明いたします。

# App Service 認証 (組み込みの認証機能) とは
App Service 認証とは、App Service 上にデプロイした Web アプリケーションでコードの実装無しで簡単に ユーザーの認証機能 (シングルサインオン(以下、SSO)) を実現する方法です。さらに、App Service 認証を有効にすると、Azure AD などの Identity Provider(以下、IdP) からアクセス元ユーザーのアクセストークン/ID トークンが取得できるので、それをもとにアプリケーション内でユーザーごとに操作を管理することも出来ます。

なお、App Service 認証で実現できる SSO で連携対象とする IdP は事前に用意されたもの中から選択する形となるため、連携していない IdP と連携を希望する場合は Web アプリケーション側で認証連携モジュールなどを使用して独自に認証を実装頂くことをご検討ください。(※認証連携に使用するプロトコルは、通常 OpenID Connect となります。SAML 認証には対応しておりませんのでご注意ください。2023-05 現在)

[App Service 認証 機能のアーキテクチャ](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#feature-architecture)
> ![image-98c5c81f-74ac-4b00-a915-a4fd24221254.png]({{site.baseurl}}/media/2023/05/image-98c5c81f-74ac-4b00-a915-a4fd24221254.png)

※ ちなみに、App Service 上の Web アプリケーションに対する認証は、App Service 認証の他にも、[クライアント証明書による認証](https://learn.microsoft.com/ja-jp/azure/app-service/app-service-web-configure-tls-mutual-auth?tabs=azurecli#enable-client-certificates)も実現できますが、この話はまた別の記事でご紹介できればと思います。


# App Service 認証を有効にする
本記事では、Azure AD を IdP として認証連携するように App Service 認証を構成します。連携可能な IdP は、Azure AD の他にも、Google、Fracebook、Twitter、さらには OpenID Connect に対応している IdP です。

App Service 認証は、Azure ポータルから App Service を選択後、[認証] ブレードから有効にできます。Azure AD を IdP として構成するには、[認証] ブレード> [ID プロバイダーの追加] をクリックし、"ID プロバイダー" ドロップダウンで [Microsoft] を選択します。

![image-36345568-8f2d-402c-aa9f-ffce7385567c.png]({{site.baseurl}}/media/2023/05/image-36345568-8f2d-402c-aa9f-ffce7385567c.png)


ID プロバイダーに [Microsoft] を選択後、基本的には規定値のままで動作できますが、構成の詳細などは、以下サイトに非常に分かりやすく記載されていますのでぜひ参照ください。

[Azure AD ログインを使用するように App Service または Azure Functions アプリを構成する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad#--option-1-create-a-new-app-registration-automatically)

### (参考) OpenID Connect のメタデータ
App Service 認証で Microsoft(Azure AD) を IdP として選択した場合には、通常 OpenID Connect が認証連携のプロトコルとして使用されます。Azure AD の OpenID Connect のメタデータは、以下 URL から取得できます。

[アプリの OpenID 構成ドキュメント URI を見つける](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/v2-protocols-oidc#find-your-apps-openid-configuration-document-uri)
> - 既知の構成ドキュメント パス: /.well-known/openid-configuration
> - 機関 URL: https://login.microsoftonline.com/{tenant}/v2.0
> GET /{tenant}/v2.0/.well-known/openid-configuration
> Host: login.microsoftonline.com

実際にリクエストしてみると issuer や authorization_endpoint などが確認できます。

```json:/.well-known/openid-configuration
{
    "token_endpoint": "https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token",
 (中略)
    "issuer": "https://login.microsoftonline.com/{tenant}/v2.0",
    "request_uri_parameter_supported": false,
    "userinfo_endpoint": "https://graph.microsoft.com/oidc/userinfo",
    "authorization_endpoint": "https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize",
    "device_authorization_endpoint": "https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode"
 (中略)
}
```

### (参考) App Service 認証の構成情報を確認する
App Service 認証の構成情報は az webapp auth show コマンドでご確認いただけます。

[az webapp auth show](https://learn.microsoft.com/ja-jp/cli/azure/webapp/auth?view=azure-cli-latest#az-webapp-auth-show) 
```powershell
az webapp auth show --name MyWebApp --resource-group MyResourceGroup
```
上記コマンドから、構成しているクライアント ID や IdP の情報などを確認することが出来ます。

実行結果例:
```
{
  "aadClaimsAuthorization": "{\"allowed_groups\":2023 年 05 月 11 日時点の内容となります。<br>,\"allowed_client_applications\":null}",
  "additionalLoginParams": null,
  "allowedAudiences": [
    "api://***"
  ],
  "allowedExternalRedirectUrls": [],
  "authFilePath": null,
  "clientId": "***",
```

### (参考) App Service 認証の構成情報を更新する
#### az コマンド
App Service 認証は、az コマンドを使用していくつかの構成情報を更新することができます。
例えば、以下コマンドでは、Azure Active Directory ID プロバイダーのオープン ID 発行者、クライアント ID、クライアント シークレットを更新します。

[az webapp auth microsoft update](https://learn.microsoft.com/ja-jp/cli/azure/webapp/auth/microsoft?view=azure-cli-latest#az-webapp-auth-microsoft-update)
```powershell
az webapp auth microsoft update  -g myResourceGroup --name MyWebApp \
  --client-id my-client-id --client-secret very_secret_password \
  --issuer https://sts.windows.net/54826b22-38d6-4fb2-bad9-b7983a3e9c5a/
```
現在ご利用いただけるコマンドの一覧は、[az webapp auth](https://learn.microsoft.com/ja-jp/cli/azure/webapp/auth?view=azure-cli-latest#commands)にリファレンスがまとめられておりますので、併せてご参照ください。

#### Azure Resource Explorer
az コマンドでは更新いただけない構成については、Azure Resource Explorer を使用して更新することが出来ます。

1. Microsoft: https://resources.azure.com で、次の手順を実行します。
1. ページの上部にある [Read/Write] を選択します。
![image-77ce2f70-720d-453b-9dba-f48b031046b5.png]({{site.baseurl}}/media/2023/05/image-77ce2f70-720d-453b-9dba-f48b031046b5.png)
1. 左側のブラウザーで、`[subscriptions]><subscription_name>>[resourceGroups]><resource_group_name>>[providers]>[Microsoft.Web]>[sites]><app_name>>[config]>[authsettingsV2]` に移動します。
1. [編集] をクリックします。
1. 編集対象とするプロパティを変更し、[Put] をクリックします。
プロパティの値は、以下リファレンスにおまとめしておりますため、ご参照ください。
[Microsoft.Web sites/config 'authsettingsV2'](https://learn.microsoft.com/ja-jp/azure/templates/microsoft.web/sites/config-authsettingsv2?pivots=deployment-language-arm-template)


例えば、App Service の認証セッションの有効期限は既定で 8 時間ですが、これは　[authsettingsV2] の login.cookieExpiration.timeToExpiration から変更することが出来ます。

```
      "cookieExpiration": {
        "convention": "FixedTime",
        "timeToExpiration": "08:00:00"
      }
```


# App Service 認証の動作を確認してみる
実際に App Service 認証の動作を確認してみます。確認対象の認証フローは、主に使用されているフローである「プロバイダーの SDK を使わない場合」（App Service により自動的に認証を行う場合）をご紹介します。
また、ここでは、IdP として Azure AD を使用します。

[Authentication flow](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#authentication-flow)
>プロバイダーの SDK を使わない場合:アプリケーションは、フェデレーション サインインを App Service に委任します。 これはブラウザー アプリで通常のケースであり、プロバイダーのログイン ページをユーザーに表示することができます。
>
>(中略)
>
>手順	プロバイダーの SDK を使わない場合
> - 1.ユーザーをサインインさせる	クライアントを /.auth/login/<provider> にリダイレクトします。
> - 2.認証をポストする	プロバイダーはクライアントを /.auth/login/<provider>/callback にリダイレクトします。
> - 3.認証済みのセッションを確立する	App Service は認証された Cookie を応答に追加します。
> - 4.認証済みのコンテンツを提供する	クライアントは以降の要求に認証クッキーを含めます (ブラウザーによって自動的に処理されます)。

Azure AD を IdP とした場合の認証フローのイメージ図は以下の通りです。

App Service と Azure AD それぞれにクライアントを操作するユーザーに対する認証セッションが生成される点に注意が必要です。Azure AD へのログインが完了したら、Azure AD にてユーザーの認証セッションが生成され、App Service にて Azure AD との認証連携が完了したらユーザーの認証セッションが App Service 内で生成されます。

※ Azure AD 内の認証セッションは既定で 90 日間ですが、セッションが切れるまでの間に一回でもサインインしていたら (Azure AD に認証が来てセッション Cookie で SSO すれば)、その時点から 90 日有効期限は延長されます。
参考: [ユーザー サインインの頻度](https://learn.microsoft.com/ja-jp/azure/active-directory/conditional-access/howto-conditional-access-session-lifetime#user-sign-in-frequency) また、Azure AD のセッション Cookie の有効期限については [ブラウザー クライアントによる AAD との SSO セッション フローと有効期限の考え方](https://jpazureid.github.io/blog/azure-active-directory/aad-token-lifetime/#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%BC-%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E3%81%AB%E3%82%88%E3%82%8B-AAD-%E3%81%A8%E3%81%AE-SSO-%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3-%E3%83%95%E3%83%AD%E3%83%BC%E3%81%A8%E6%9C%89%E5%8A%B9%E6%9C%9F%E9%99%90%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9) を参照ください。

![image-19ff11bc-a3e1-4fe2-87c0-bfe3444c570c.png]({{site.baseurl}}/media/2023/05/image-19ff11bc-a3e1-4fe2-87c0-bfe3444c570c.png)

## 1 ) ユーザーのサインイン
### 認証フロー開始
App Service 認証の設定で、以下のように指定していた場合には、認証されていない要求は自動的にプロバイダーにリダイレクトされ、認証フロー（ユーザーのサインイン）が開始されます。

- ”アクセスを制限する”: "認証が必要"
- "認証されていない要求":"HTTP 302 (検出) を返す (ID プロバイダーにリダイレクト)" 

![image-d11daee3-8bb2-4120-bb4e-bd8b7111e0df.png]({{site.baseurl}}/media/2023/05/image-d11daee3-8bb2-4120-bb4e-bd8b7111e0df.png)

”アクセスを制限する” を "認証されていない要求を許可する" にした場合には、以下ドキュメントに記載されているように Web アプリケーションにてクライアントを `/.auth/login/<provider>` にリダイレクトすることで認証フローを開始させることが出来ます。その際に post_login_redirect_uri パラメターで認証フローが完了した後、どの画面を表示させるか指定します。

[複数のサインイン プロバイダーを使用する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-customize-sign-in-out#use-multiple-sign-in-providers)
>アプリのその他の任意の場所で、有効にした各プロバイダーへのサインイン リンク (/.auth/login/<provider>) を追加します。 次に例を示します。
>
>`<a href="/.auth/login/aad">Log in with the Microsoft Identity Platform</a>`
>
>サインイン後のユーザーをカスタム URL にリダイレクトさせるには、post_login_redirect_uri クエリ文字列パラメーターを使用します。
>
>`<a href="/.auth/login/<provider>?post_login_redirect_uri=/Home/Index">Log in</a>`

### (App Service 認証の処理) 認可要求
App Service 認証は、`/.auth/login/aad` へのリクエストを受け取ったら、連携している IdP の `/authorize` エンドポイントにクライアントを自動的にリダイレクトさせます。例えば Azure AD では `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` になります。なお、この URL は、上述の "OpenID Connect のメタデータ" からも確認ができますので併せて参照してみてください。

[例: サインイン要求を送信する](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/v2-protocols-oidc#send-the-sign-in-request)
>GET https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?
client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=id_token
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&response_mode=form_post
&scope=openid
&state=12345
&nonce=678910

実際に試してみると、上記と同じ URL に、クライアント (ブラウザー) がリダイレクトされているのが分かります。

![image-7af29312-1d45-4346-867f-848a261edd18.png]({{site.baseurl}}/media/2023/05/image-7af29312-1d45-4346-867f-848a261edd18.png)

## 2 ) (App Service 認証の処理) 認可コードとアクセストークンの取得
`/authorize` エンドポイントにクライアントがリダイレクトされた後、クライアントを操作するユーザーが IdP にサインインしていなければID/パスワードを入力する画面が表示されます。IdP にサインイン済みであれば、通常画面表示はスキップされ、ユーザーは再度認証情報を入力することなく、認証が完了します。

認証が完了したら、App Service 認証の `/.auth/login/<provider>/callback` へ認可コード (authorization_code) が送られ、App Service 認証はその認可コードをもとに IdP からアクセストークンを取得します。取得したアクセストークンは App Service 認証により検証されます。具体的には、App Service 認証の設定に追加されている「発行者の URL」であるか、「許可されるトークン対象ユーザー」であるか等です。
この検証に失敗すると、認証エラーになり、App Service にアクセスできません。

## 3 ) 認証済みのセッションを確立する 
App Service でアクセストークンの検証まで完了すると、App Service 認証によりクライアントを操作するユーザーに対する認証セッションが App Service で作成され、この認証セッションによりユーザーの認証状況を保存しています。
認証セッションは、ブラウザーからアクセスした場合には、「AppServiceAuthSession」という Cookie で管理されます。ブラウザーではこの Cookie が存在していることで、認証済みユーザーとして App Service へアクセスすることが出来ます。

「AppServiceAuthSession」Cookie は、対象 App Service へアクセスした後、ブラウザーの [Application] タブより下図のようにご確認いただけます。 

![image-0f50180a-b3d4-4396-afa3-d64179c15463.png]({{site.baseurl}}/media/2023/05/image-0f50180a-b3d4-4396-afa3-d64179c15463.png)

App Service 内で作成される認証セッションは既定では 8 時間で有効期限切れになりますが、もしこれに不足がある場合には、上述の **(参考) App Service 認証の構成情報を更新する** セクションに記載の手順で時間を変更することが出来ます。

## 4 ) 認証済みのコンテンツを提供する
### Web アプリケーションからクライアントの情報にアクセスする
App Service 認証が完了したら、Web アプリケーションはリクエストヘッダーからクライアントの認証情報を取得することが出来ます。

[アプリ コードでユーザー クレームにアクセスする](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-user-identities#access-user-claims-in-app-code)
> |Header|	説明|
>| ---- | ---- |
> |X-MS-CLIENT-PRINCIPAL|	使用可能な要求の Base64 エンコード JSON 表現。 詳細については、「クライアント プリンシパル ヘッダーのデコード」を参照してください。|
> |X-MS-CLIENT-PRINCIPAL-ID|	ID プロバイダーによって設定された呼び出し元の識別子。
>|X-MS-CLIENT-PRINCIPAL-NAME|	ID プロバイダーによって設定された呼び出し元の人間が判読できる名前。
> |X-MS-CLIENT-PRINCIPAL-IDP|	App Service 認証で使用される ID プロバイダーの名前。
> |X-MS-TOKEN-AAD-ACCESS-TOKEN| アクセストークン (Azure AD を IdP として選択した場合)
> |X-MS-TOKEN-AAD-ID-TOKEN| ID トークン (Azure AD を IdP として選択した場合)


ASP.NETCore6 MVC(C#) アプリケーションでクライアントの Principal Name を取得する場合のサンプルコードはこちらです。

Controller:
```
   public IActionResult Index()
    {
        var principalName = Request.Headers["X-MS-CLIENT-PRINCIPAL-NAME"];
        return View();
    }
```

アプリコードから ユーザー クレームにアクセスする方法の詳細は、[App Service 認証機能（Easy Auth）の Tips のご紹介](https://azure.github.io/jpazpaas/2022/12/02/Tips-for-App-Service-authentication.html#3-%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B3%E3%83%BC%E3%83%89%E3%81%8B%E3%82%89%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%82%92%E5%8F%96%E5%BE%97%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95) をご確認ください。


### クライアントから自身の認証情報にアクセスする
トークンストア が有効な場合は `/.auth/me` にアクセスすることでクライアントの認証情報を確認することが出来ます。

[API を使用してユーザー クレームにアクセスする](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-user-identities#access-user-claims-using-the-api)

実際にアクセスしてみると、アクセストークンやユーザー名などの値が表示されます。

![image-b7a6ee44-12b4-47fa-b72d-e003c30c618c.png]({{site.baseurl}}/media/2023/05/image-b7a6ee44-12b4-47fa-b72d-e003c30c618c.png)

```
{
    "access_token": "***",
    "expires_on": "2023-04-27T08:52:52.5858318Z", (※アクセストークンの有効期限)
    "id_token": "***",
    "provider_name": "aad",
    "user_claims": [{
            "typ": "aud",
            "val": "***"
        }, {
            "typ": "iss",
            "val": "https:\/\/login.microsoftonline.com\/***\/v2.0"
 ...
}
```

# エラーになったときのデバッグ方法
## [問題の診断と解決] 機能を使用する
[問題の診断と解決] > 認証設定および調査検出器(イージー認証)機能 の 「簡単な認証エラー/警告」セクションにより、認証に失敗している原因についてご確認いただくことが出来ます。
もし、App Service 認証に失敗するといった状況がありましたら、まずはこちらの機能を参照いただくことをお勧めいたします。

![image-8ff0fb62-9d02-49bb-a277-061c3730ede7.png]({{site.baseurl}}/media/2023/05/image-8ff0fb62-9d02-49bb-a277-061c3730ede7.png)

### ■ よくある原因のご紹介
#### 1 ) アクセストークンの検証に失敗している
認証に失敗する原因として、App Service 認証に設定した値とトークンの値に相違がある例をご紹介します。左記の場合には、「簡単な認証エラー/警告」セクションに JWT validation failed(JWT 検証に失敗しました) が表示されます。メッセージの例は、以下の通りです。

JWT 検証に失敗しました: 発行者の検証に失敗しました - 予想: https://login.microsoftonline.com/XXX/v2.0;トークン: https://sts.windows.net/XXX/

"予想" が App Service 認証に構成されている値となり、"トークン" がアクセストークンをもとにした値となります。上記例では、発行者(Issuer) が、”予想” と "トークン" で値が異なっているため、検証に失敗しエラーとなっていることを示しております。
上記の値の相違がある場合には、App Service 認証に構成されている値を "トークン" の値と揃えていただくことで認証に成功する可能性があります。

#### 2 ) App Service と、その前段に配置している Application Gateway のドメインに相違がある
よくある例として、App Service の前段に Application Gateway を配置し、その Application Gateway 経由でクライアントからのリクエストを受け付ける構成がございます。左記構成の場合については、以下ブログにご紹介がございますため、少しでもご参考となりましたら幸いです。

[App Service の前段に ApplicationGateway を配置した際の Azure AD 認証](https://azure.github.io/jpazpaas/2022/03/09/Application-Gateway-front-of-App-Service-auth.html)

-------
<br>App Service 認証に関する概要を以上の通りおまとめいたしました。少しでもご参考となりましたら幸いです。

<br>
<br>

2023 年 02 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>