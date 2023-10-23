---
title: "App Service認証(easy auth)が有効な App Service に色々なクライアントからアクセスする方法"
author_name: "<yunasugimoto><tangmai>"
tags:
    - App Service
    - Function App
---

# App Service 認証が有効な App Service のサイトへアクセスする
App Service では組み込みの認証機能を提供しており、App Service Authentication または Easy Auth と呼ばれています。この組み込みの認証機能により、App Service でホストしているアプリケーションでは認証/認可用のコードを実装せずに、簡単に認証と認可の機能をアプリケーションで利用することが出来ます。

[参考: 認証と承認 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization)

App Service 認証が有効な App Service(Azure Functions 含む) でホスティングされているお客様の Web アプリケーションや API (以下、まとめてサイトと表記します) へアクセスする方法はクライアントの種類により異なります。本記事では、代表的なクライアントからアクセスする例を 5 つご紹介します。

イメージ:<br>
Client --> App Service 認証が有効な App Service/Azure Functions のサイト

1 ) Client: ブラウザー(ログインユーザーあり)
<br>
2 ) Client: デーモン(ログインユーザーなし)
<br>
3 ) Client: マネージドID を使用したサービス(Logic Apps)(ログインユーザーなし)
<br>
4 ) Client: AppService (ログインユーザーあり、AppService 認証でアクセストークンを取得する)
<br>
5 ) Client: SPA アプリケーション(ログインユーザーあり、MSAL でアクセストークンを取得する)　※ App Service のセッショントークンを使用するシナリオ
<br>

前提として本記事では、特に断りがない場合、ID プロバイダーとして Microsoft Entra ID を使用し、以下のように App Service 認証の設定を構成した場合を想定しています。(※シナリオ 3 を除く)

[認証と承認 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#authorization-behavior)
>![image-75ca95a4-1a06-41be-9e89-ec944ccee5a4.png]({{site.baseurl}}/media/2023/10/image-75ca95a4-1a06-41be-9e89-ec944ccee5a4.png)


## 1 ) ブラウザークライアントから App Service 認証が有効な App Service/Azure Functions のサイトへアクセスする
クライアントがブラウザーの場合、App Service/Azure Functions の URL(https://<リソース名>.azurewebsites.net) へアクセスすると、デフォルトでは自動的に App Service 認証で指定した ID プロバイダーと認証連携 (シングルサインオン) が自動的に行われ、認証に成功すると、App Service/Azure Functions でホストしているサイトへアクセスすることが出来ます。

実際に試してみると以下のような流れになります。
<br>
手順１：App Service/Azure Functions のサイト URL を確認する
<br>
手順２：サイト URL へアクセスする


### 手順１：App Service/Azure Functions のサイト URL を確認する
Azure ポータルから対象リソースを選択後、[概要] > 既定のドメイン(URL) から App Service/Azure Functions のサイト URL を確認します。

![image-581c1900-3e06-480a-ba62-63425284dd6d.png]({{site.baseurl}}/media/2023/10/image-581c1900-3e06-480a-ba62-63425284dd6d.png)

Azure Functions の関数へアクセスする場合は、[概要] から対象関数を選択後、[概要] > 関数の URL の取得から実行対象の関数の URL を確認します。

![image-56ef764c-1f74-45ed-b3d3-bda9323cb584.png]({{site.baseurl}}/media/2023/10/image-56ef764c-1f74-45ed-b3d3-bda9323cb584.png)


### 手順２：サイト URL へアクセスする
ブラウザーのアドレスバーに手順１で確認した URL を入力すると、認証が行われていないクライアントの場合、デフォルトの設定では自動的に App Service 認証で連携を行っている ID プロバイダー(Microsoft Entra ID など) へリダイレクトされます。
リダイレクトされない場合、認証の設定で "アクセスを制限する"が"認証が必要"、"認証されていない要求"が"HTTP 302 (検出) を返す (ID プロバイダーにリダイレクト)" 以外が構成されている可能性がございます。詳細については、[認証と承認 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#authorization-behavior) をご参照ください。

クライアントが ID プロバイダー(Microsoft Entra ID など) へリダイレクトされた後、連携対象の ID プロバイダーにユーザーが既にログイン済みであった場合、ID/PW の入力画面が表示されることなく、そのまま認証が完了します。連携対象の ID プロバイダーにユーザーがログイン済みではない場合、ID/PW などで ID プロバイダーにユーザーがログインを行い、認証を成功させます。
認証が完了した後、初回リクエストでは以下のようにアクセス許可が求められ、[Accept] ボタンよりアクセス許可を行います。[ご参考:「管理者の承認が必要」のメッセージが表示された場合の対処法](https://jpazureid.github.io/blog/azure-active-directory/azure-ad-consent-framework/)

※ 初回リクエストのみ

![image-96147ca8-edd5-415b-87ac-25d6cad3244b.png]({{site.baseurl}}/media/2023/10/image-96147ca8-edd5-415b-87ac-25d6cad3244b.png)

認証及びアクセス許可が完了すると、App Service/Azure Functions のサイトが表示されます。

![image-6b8cd62a-ba53-4816-8a38-03e5b4abd738.png]({{site.baseurl}}/media/2023/10/image-6b8cd62a-ba53-4816-8a38-03e5b4abd738.png)

※ Azure Functions の HTTP トリガー関数の場合、以下のように実行結果が表示されます。

![image-f527a4ea-b5c6-4c7e-81c0-40e507ee4572.png]({{site.baseurl}}/media/2023/10/image-f527a4ea-b5c6-4c7e-81c0-40e507ee4572.png)

(ご参考) ブラウザークライアントからアクセスする場合の認証フローは、[認証と承認 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#authentication-flow)で "プロバイダーの SDK を使わない場合" のシナリオとして記載しております。また、仕組みの詳細については
[App Service 認証の概要と仕組みについて - Japan PaaS Support Team Blog](https://jpazpaas.github.io/blog/2023/05/11/what-is-appserviceauthentication.html)にもご紹介しておりますので必要に応じてご参照ください。

## 2 ) デーモンクライアント(ログインユーザーなし)から App Service 認証が有効な App Service/Azure Functions のサイトへアクセスする

デーモンアプリケーションなど、ユーザーがいないクライアントを使用するシナリオで App Service/Azure Functions でホストしているサイトへアクセスする場合には、アプリケーション内で ID プロバイダーからアクセストークンを取得し、そのトークンを Authorization リクエストヘッダーに付与することで、App Service/Azure Functions でホストしているサイトへアクセスすることが出来ます。

[ご参考: デーモン クライアント アプリケーション (サービス間の呼び出し)](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad#daemon-client-application-service-to-service-calls)
>アプリケーションは、(ユーザーの代わりではなく) それ自体の代わりに App Service または関数アプリでホストされる Web API を呼び出すトークンを取得できます。 このシナリオは、ログイン ユーザーなしでタスクを実行する非対話型デーモン アプリケーションに役立ちます。 これには標準の OAuth 2.0 クライアント資格情報の付与が使用されます。

ID プロバイダーとして Microsoft Entra ID を選択した場合のイメージは以下の通りです。
<br>
![image-1449da9e-87c7-4bf2-9183-4abcd4a3a4c9.png]({{site.baseurl}}/media/2023/10/image-1449da9e-87c7-4bf2-9183-4abcd4a3a4c9.png)
<br>
具体的な手順は以下のような流れになります。
<br>
手順１：デーモンアプリケーションを Microsoft Entra ID に登録する (上図手順①)
<br>
手順２：Microsoft Entra ID からアクセストークンをするために必要な情報を確認する
<br>
手順３：Microsoft Entra ID からアクセストークンを取得する (上図手順②)
<br>
手順４：App Service 認証を有効にした App Service/Azure Functions へリクエストを送信する (上図手順③)

### 手順１：デーモンアプリケーションを Microsoft Entra ID に登録する
クライアントとする デーモンアプリケーションを Microsoft Entra ID に登録します。手順は以下ドキュメントに記載がございますのでご参照ください。App Service/Azure Functions は、App Service 認証を有効にした際に Microsoft Entra ID にアプリケーション登録が行われますので追加の構成は不要となります。

[Azure AD 認証を構成する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad?tabs=workforce-tenant#daemon-client-application-service-to-service-calls)
>![image-a240d535-0899-4e7d-b29a-8780fc57ab24.png]({{site.baseurl}}/media/2023/10/image-a240d535-0899-4e7d-b29a-8780fc57ab24.png)

[ご参考: クイック スタート: Microsoft ID プラットフォームでアプリを登録する - Microsoft Entra](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/quickstart-register-app#register-an-application)

### 手順２：Microsoft Entra ID からアクセストークンをするために必要な情報を確認する
手順1を参考にデーモンアプリケーションのクライアント ID とクライアントシークレットを確認します。
次に、Azure ポータルから対象の App Service/Azure Functions を選択後、[認証] ブレードより ID プロバイダー(Microsoft) のアプリ (クライアント) ID を確認します。

![image-e14594fe-a98f-4148-af1e-4928316886ea.png]({{site.baseurl}}/media/2023/10/image-e14594fe-a98f-4148-af1e-4928316886ea.png)

対象 App Service/Azure Functions とデーモンアプリケーションを登録したテナントのテナント ID を確認します。テナント ID は、Azure ポータルから App Service/Azure Functions を選択後、[認証] ブレード > ID プロバイダーより対象プロバイダーのリンクをクリック > [概要] の ディレクトリ(テナント) ID からご確認いただけます。

![image-ecb402b6-2470-4f8c-b902-29078c725a2a.png]({{site.baseurl}}/media/2023/10/image-ecb402b6-2470-4f8c-b902-29078c725a2a.png)

![image-7f10ed4b-940f-4ad0-8675-bcf9235fb572.png]({{site.baseurl}}/media/2023/10/image-7f10ed4b-940f-4ad0-8675-bcf9235fb572.png)


### 手順３：Microsoft Entra ID からアクセストークンを取得する
Microsoft Entra ID のアクセストークンは、https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token に POST 要求を送信することで取得できます。または、[MSAL などの SDK を使用して取得します](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/msal-acquire-cache-tokens)。


[ご参考: Microsoft ID プラットフォームの OAuth 2.0 クライアント資格情報フロー](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow#first-case-access-token-request-with-a-shared-secret)
>最初のケース:共有シークレットを使ったアクセス トークン要求
>```
> POST /{tenant}/oauth2/v2.0/token HTTP/1.1
> Host: login.microsoftonline.com
>Content-Type: application/x-www-form-urlencoded
>
>client_id=535fb089-9ff3-47b6-9bfb-4f1264799865
>&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default
>&client_secret=sampleCredentia1s
>&grant_type=client_credentials
>```

上記リクエスト例の値をそれぞれ 手順２で確認した値に置き換えます。
{tenant} はテナント ID、client_id は デーモンアプリケーションのクライアント ID、client_secret はデーモンアプリケーションのクライアントシークレット、scope は "{App Service/Azure Functions のクライアント ID}/.default" として指定します。
例: ID プロバイダー(Microsoft) のアプリ (クライアント) ID が 190XXX-XXX-XX だった場合、scope の値は 190XXX-XXX-XX/.default （または、api://190XXX-XXX-XX/.default）となります。

PowerShell スクリプトでアクセストークンを取得する例は以下の通りです。

```powershell
$tenantId = 'アプリケーションを登録したテナントのテナント ID'
$authBody=@{
    client_id = 'デーモンアプリケーションのクライアント ID'
    client_secret = 'デーモンアプリケーションのクライアントシークレット'
    scope = '{App Service/Azure Functions のクライアント ID}/.default'
    grant_type = 'client_credentials'
}
$tokenUri = "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token"

# アクセストークンを取得
$accessResponse = Invoke-WebRequest -Uri $tokenUri -ContentType 'application/x-www-form-urlencoded' -Body $authBody -Method Post -ErrorAction Stop -Verbose

# 取得したアクセストークンを確認
$accessToken = $accessResponse.content | ConvertFrom-Json
$accessToken.access_token
```

$accessToken.access_token でアクセストークンが確認できます。 なお、jwt 形式のアクセストークンの場合、以下サイトからデコードができます。
https://jwt.ms/


### 手順４：App Service 認証を有効にした App Service/Azure Functions へリクエストを送信する
手順3で取得したアクセストークンを Authorization ヘッダーに付与し、App Service 認証を有効にした App Service/Azure Functions へリクエストを送信します。

```powershell
$authHeader = @{
    'Content-Type' = 'application/json'
    'Authorization' = 'Bearer ' + $accessToken.access_token
    }
 
$siteURI = 'https://{App Service リソース名}.azurewebsites.net/'
$getResponse = Invoke-WebRequest -Method Get -UseBasicParsing -Uri $siteURI  -Headers $authHeader -SessionVariable session -Verbose
```

HTTP 401 や 403 などの応答が返却される場合、アクセストークンを取得する際の scope に正しいクライアント ID が指定されていない、App Service 認証の構成が正しくない、または [組み込み承認ポリシー](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad?tabs=workforce-tenant#use-a-built-in-authorization-policy)が有効になっていることがよくある原因として考えられます。
組み込みの承認ポリシーが有効である場合には allowedApplications にデーモンアプリケーションのクライアント ID を構成いただくことでエラーが解消する可能性がございます。対応方法は、[ご参考: エラーになったときのデバッグ方法](https://jpazpaas.github.io/blog/2023/05/11/what-is-appserviceauthentication.html#%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%AB%E3%81%AA%E3%81%A3%E3%81%9F%E3%81%A8%E3%81%8D%E3%81%AE%E3%83%87%E3%83%90%E3%83%83%E3%82%B0%E6%96%B9%E6%B3%95) と本記事シナリオ 3 の手順 4 をご参照ください。


## 3 ) マネージドID を使用したサービス(Logic Apps)(ログインユーザーなし)から App Service 認証が有効な App Service/Azure Functions のサイトへアクセスする
Logic Apps など、一部 Azure サービスではマネージドIDを使用して他サービスを呼び出す機能が組み込まれています。本シナリオでは、Logic Apps で公開されている以下ドキュメントに従って、Logic Apps から マネージド ID の認証を使用して App Service 認証が有効な Azure Functions の HTTP トリガー関数を呼び出す手順をご紹介します。

- [関数呼び出しの認証を有効にする (従量課金ワークフローのみ) - Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-azure-functions?tabs=consumption#enable-authentication-for-function-calls-consumption-workflows-only)
- [例:マネージド ID を使用して組み込みのトリガーまたはアクションを認証する](https://learn.microsoft.com/ja-jp/azure/logic-apps/create-managed-service-identity?tabs=consumption#example-authenticate-built-in-trigger-or-action-with-a-managed-identity)

本記事では上記ドキュメントの手順に記載していない情報もいくつか載せていますので参考までにご確認いただけますと幸いです。
具体的な手順は以下のような流れになります。
<br>
手順１：Logic Apps(従量課金) でシステム割り当てマネージド ID を有効にする
<br>
手順２：Azure Functions で HTTP トリガー関数を作成する
<br>
手順３：Azure Functions で App Service 認証を有効にし、CORS 設定を変更する
<br>
手順４：Logic Apps から関数の実行ができるように Azure Functions の App Servce 認証の設定を変更する
<br>
手順５：Logic Apps から HTTP トリガー関数を呼び出す


### 手順１：Logic Apps(従量課金) でシステム割り当てマネージド ID を有効にする
従量課金プランの Logic Apps を作成し、Azure ポータルの [ID] ブレードからシステム割り当てマネージド ID を有効にします。マネージドIDを有効にした後、表示されるオブジェクト ID をコピーします。

[マネージド ID を使用して接続を認証する - Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/create-managed-service-identity?tabs=consumption#enable-system-assigned-identity-in-azure-portal)
>Azure portal で、ロジック アプリ リソースに移動します。ロジック アプリのメニューの [設定] で、 [ID] を選択します。
[ID] ペインの [システム割り当て] で、[オン][保存] を選びます。 確認を求めるメッセージが表示されたら、 [はい] を選択します。
>
>![image-4e24d6aa-1eb1-4855-964f-f7c72c505ebb.png]({{site.baseurl}}/media/2023/10/image-4e24d6aa-1eb1-4855-964f-f7c72c505ebb.png)

### 手順２：Azure Functions で HTTP トリガー関数を作成する
Azure ポータルにて Azure Functions を選択後、[概要] ブレード > 関数 > "+作成" から HTTP トリガーの関数を作成します。
今回、[承認レベル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=python-v2%2Cin-process%2Cnodejs-v4%2Cfunctionsv2&pivots=programming-language-csharp#http-auth) は Function とします。

![image-10c97881-8b07-46b5-bdd3-c478a28a55c3.png]({{site.baseurl}}/media/2023/10/image-10c97881-8b07-46b5-bdd3-c478a28a55c3.png)

関数を作成したら、下図赤枠のように [概要] ブレード > 関数 から作成した関数を選択し、[概要] > "関数の URL の取得" から default(function key) の URL をコピーします。

![image-838fe7fa-e628-4bda-ad36-ec99f6f3dc9a.png]({{site.baseurl}}/media/2023/10/image-838fe7fa-e628-4bda-ad36-ec99f6f3dc9a.png)


### 手順３：Azure Functions で App Service 認証を有効にし、CORS 設定を変更する
App Service 認証の構成を行うために、テナント ID を確認します。Azure ポータル上部の検索ウィンドウから "Entra ID" と入力し、Microsoft Entra ID リソースを開きます。[概要] ブレード > 概要タブの基本情報にテナント ID が記載されているのでコピーします。

![image-83a357ff-4789-4fa3-b998-605a6f5d187a.png]({{site.baseurl}}/media/2023/10/image-83a357ff-4789-4fa3-b998-605a6f5d187a.png)

[参考: Azure AD のテナント ID を特定する](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-azure-functions?tabs=consumption#find-the-tenant-id-for-your-azure-ad)

Azure ポータルから Azure Functions を選択後、[認証] ブレード > [ID プロバイダーの追加] > [Microsoft] を選択します。[アプリの登録] の [アプリの登録の種類] で、[既存アプリの登録の詳細を提供します] を選択し、前の手順でコピーした値を入力します。[参考: 関数アプリのアプリ登録を作成する (従量課金ワークフローのみ)  - Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-azure-functions?tabs=consumption#create-app-registration-for-your-function-app-consumption-workflows-only)

| 項目 | 値|
|:-----------|:------------|
| アプリケーション ID       | 手順１でコピーした Logic Apps のマネージド ID のオブジェクト ID       | 
| クライアントシークレット       | ※入力しません (マネージドIDを使用して Azure Functions を呼び出すため)    | 
| 発行者の URL      | `https://sts.windows.net/<テナント ID>`    | 
| 許可されるトークン対象ユーザー    | `https://management.azure.com`    | 

![image-a8541865-fa2c-4676-92af-a255eb955592.png]({{site.baseurl}}/media/2023/10/image-a8541865-fa2c-4676-92af-a255eb955592.png)


App Service 認証の設定が完了したら、以下ドキュメントに従って Azure Functions の [CORS] ブレードから 許可対象のオリジンを * にします。

[ワークフローから Azure Functions を呼び出す - Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-azure-functions?tabs=consumption#find-functions-that-have-openapi-descriptions)
>![image-809493a5-7f9c-4666-a718-11be2d133552.png]({{site.baseurl}}/media/2023/10/image-809493a5-7f9c-4666-a718-11be2d133552.png)

### 手順４：Logic Apps から関数の実行ができるように Azure Functions の App Servce 認証の設定を変更する
Azure Functions で [組み込み承認ポリシーを有効にしている](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad?tabs=workforce-tenant#use-a-built-in-authorization-policy)場合は、Logic Apps からの実行を許可するように構成します。Logic Apps から関数を実行した際に HTTP 403 エラーが発生する場合には、組み込みの承認ポリシーでアクセスが拒否されている可能性があります。

組み込み承認ポリシーを設定するため、手順１でコピーしたオブジェクト ID を Microsoft Entra ID で検索し、エンタープライズアプリケーション に表示されるアプリケーションをクリック > アプリケーション ID から Logic Apps のマネージド ID のアプリケーション ID を確認します。

![image-a556ca9e-6079-4195-b6ec-3ff0736136da.png]({{site.baseurl}}/media/2023/10/image-a556ca9e-6079-4195-b6ec-3ff0736136da.png)

![image-78da0882-94f5-4241-a288-e27915bcb784.png]({{site.baseurl}}/media/2023/10/image-78da0882-94f5-4241-a288-e27915bcb784.png)

Azure Resource Explorer https://resources.azure.com/ にアクセスし、`Azure Functions が所属するサブスクリプション > リソースグループ > providers > Microsoft.Web > sites > "Azure Functions のリソース名" > config > authsettingsV2` から、`identityProviders.azureActiveDirectory.validation.defaultAuthorizationPolicy` の `allowedApplications` に上記で確認した Logic Apps のマネージド ID のアプリケーション ID を入力します。左記設定により、組み込みの承認ポリシーで Logic Apps からマネージドIDを使用して関数の実行が許可されます。

`allowedApplications` のみ構成されている場合、アプリケーションは Logic Apps からのみ Azure Functions を実行できますが、同一テナントのユーザーは関数を実行できます。同一テナントのユーザーからも関数を実行できないようにする場合には、`allowedPrincipals.identities` にも Logic Apps のマネージド ID のオブジェクト ID を構成します。この設定により「対象オブジェクト ID を持つ Logic Apps」 以外のすべてのユーザー、アプリケーションからは関数を実行することが出来なくなります。

[参考: Azure AD 認証を構成する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad?tabs=workforce-tenant#use-a-built-in-authorization-policy)

![image-fc9737ce-6187-4a0c-b696-6709fe3161e4.png]({{site.baseurl}}/media/2023/10/image-fc9737ce-6187-4a0c-b696-6709fe3161e4.png)


### 手順５：Logic Apps から HTTP トリガー関数を呼び出す
[マネージド ID を使用して接続を認証する - Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/create-managed-service-identity?tabs=consumption#authenticate-access-with-managed-identity) を参考に、Azure Functions を呼び出す HTTP リクエストのアクションを構成します。

| 項目 | 値|
|:-----------|:------------|
| 方法      | GET       | 
| URI      | 手順２でコピーした URL(?code=~ の部分は クエリ セクションに入力します)    | 
| クエリ     | 手順２でコピーした URL の code とキーの値    | 
| 認証の種類    | マネージドID    | 
| マネージドID    | システム割り当てマネージドID    | 
| 対象ユーザー    | `https://management.azure.com`    | 

![image-d6a27057-152c-4a19-8208-4b4c466900ad.png]({{site.baseurl}}/media/2023/10/image-d6a27057-152c-4a19-8208-4b4c466900ad.png)

アクションを保存して関数を実行すると、以下のように実行結果が返却され、関数を実行できていることが確認できます。

![image-ddf5b9a3-cb71-4b54-83a9-811c502f5756.png]({{site.baseurl}}/media/2023/10/image-ddf5b9a3-cb71-4b54-83a9-811c502f5756.png)


## ４ ) App Service (ログインユーザーあり、AppService 認証でアクセストークンを取得する) から App Service 認証が有効な App Service/Azure Functions のサイトへアクセスする
App Service 認証が有効な App Service からさらに別の App Service 認証が有効な App Service/Azure Funcitons を呼び出すシナリオについてご紹介します。

Client(ユーザー) -->  App Service 認証が有効な App Service -->  App Service 認証が有効な App Service/Azure Funcitons

上記方法では、Client(ユーザー) が前段の App Service へアクセスする際に App Service 認証で取得したアクセストークンを使用して後続の App Service 認証が有効な App Service/Azure Funcitons へアクセスします。本記事で紹介したシナリオ 1 と 2 の組み合わせのようなイメージです。

- 例1: Client(ユーザー) --ブラウザーで認証-->  App Service 認証が有効な App Service --Bearer トークンで認証-->  App Service 認証が有効な App Service/Azure Funcitons

上記シナリオの詳細については、以下のドキュメントにおまとめがございますため本記事では省略いたします。

[参考: チュートリアル:ユーザーを E2E で認証する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/tutorial-auth-aad?pivots=platform-windows)



## ５ ）SPA アプリケーション(ログインユーザーあり、MSAL でアクセストークンを取得する) から App Service 認証が有効な App Service/Azure Functions のサイトへアクセスする ※App Service のセッショントークンを使用するシナリオ
シナリオ 4 では、App Service/Azure Functions 上でホストされているアプリケーションについての方法をご紹介しました。シナリオ 5 では App Serivce/Azure Functions でホストされていない シングルページ アプリケーション (SPA) にて MSAL ライブラリによりユーザーをログインさせ、取得したアクセストークンを X-ZUMO-AUTH ヘッダーへ付与し App Service 認証が有効な App Service へアクセスするシナリオをご紹介いたします。

<br>
手順１：Microsoft Entra ID（Azure AD）に SPA アプリケーションを登録する
<br>
手順２：JavaScript SPA アプリケーションのプロジェクトをダウンロードする
<br>
手順３：JavaScript SPA アプリケーションの構成ファイルを編集する
<br>
手順４：Microsoft Entra ID に登録された SPA アプリケーションに対して、App Service のアクセス許可を追加する
<br>
手順５：App Service 認証の ID プロバイダーを構成する
<br>
手順６：プロジェクトを実行してアクセストークンを取得する
<br>
手順７：取得したアクセストークンを App Service 認証に POST して App Service 独自の認証トークン(セッショントークン)を取得する
<br>
手順８：取得した App Service 独自の認証トークン(セッショントークン)を利用して、App Service にアクセスする

### 手順１：Microsoft Entra ID（Azure AD）に SPA アプリケーションを登録する

以下の手順により、Microsoft Entra ID にアプリケーションを登録します。
<br>

[手順 1:アプリケーションの登録](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/single-page-app-quickstart?pivots=devlang-javascript#step-1-register-your-application)
>1.Azure portal にサインインします。
<br>2.複数のテナントにアクセスできる場合は、トップ メニューの [ディレクトリとサブスクリプション] フィルター  を使用して、アプリケーションを登録するテナントに切り替えます。
<br>3.Azure Active Directory を検索して選択します。
<br>4.[管理] で [アプリの登録]>[新規登録] の順に選択します。
<br>5.アプリケーションの [名前] を入力します。 この名前は、アプリのユーザーに表示される場合があります。また、後で変更することができます。
<br>6.[サポートされているアカウントの種類] で、 [Accounts in any organizational directory and personal Microsoft accounts](任意の組織のディレクトリ内のアカウントと個人用の Microsoft アカウント) を選択します。
<br>7.[登録] を選択します。 後で使用するために、アプリの [概要] ページで、 [アプリケーション (クライアント) ID] の値を書き留めます。
<br>8.[管理] で、 [認証] を選択します。
<br>9.[プラットフォーム構成] で [プラットフォームを追加] を選択します。 表示されたウィンドウで [シングルページ アプリケーション] を選択します。
<br>10.[リダイレクト URI] の値を http://localhost:3000/ に設定します。
<br>11.[構成] をクリックします。

Microsoft Entra ID に SPA アプリケーションを登録できた場合、以下のスクリーンショットのように Microsoft Entra ID のアプリの登録よりアプリケーション(クライアント) ID やテナント ID を確認できます。

 ![image-dd10d57f-43e2-4ea1-b7ab-590ed749b138.png]({{site.baseurl}}/media/2023/10/image-dd10d57f-43e2-4ea1-b7ab-590ed749b138.png)


### 手順２：JavaScript SPA アプリケーションのプロジェクトをダウンロードする
下記のリンクより SPA アプリケーションのプロジェクトをダウンロードします。

[手順 2:プロジェクトのダウンロード](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/single-page-app-quickstart?pivots=devlang-javascript#step-2-download-the-project)
>Node.js を使用して Web サーバーでプロジェクトを実行するために、コア プロジェクト ファイルをダウンロードします。

### 手順３：JavaScript SPA アプリケーションの構成ファイルを編集する

app フォルダに格納された authConfig.js ファイルを開き、下記の項目を修正します。
<br>・clientId　 ： Microsoft Entra ID に登録した SPA アプリケーションのアプリケーション (クライアント) ID
<br>・authority　： https://login.microsoftonline.com/<Microsoft Entra ID に登録した SPA アプリケーションのテナント ID>
<br>・redirectUri： http://localhost:3000/

![image-cdace94c-f74c-4e73-8c7d-cb4efcd565fe.png]({{site.baseurl}}/media/2023/10/image-cdace94c-f74c-4e73-8c7d-cb4efcd565fe.png)

<br>・loginRequest：
<br>App Service 向けに SPA アプリケーションからアクセストークンを取得するため、loginRequest パラメータscopes を以下のように変更します。
```
const loginRequest = {
scopes: ["api://{App Service 認証のアプリケーション(クライアント) ID}/user_impersonation"]
};
```
![image-80275503-3ac9-4fd4-9284-e5926ca3ff70.png]({{site.baseurl}}/media/2023/10/image-80275503-3ac9-4fd4-9284-e5926ca3ff70.png)

※ {App Service 認証のアプリケーション(クライアント) ID} は以下のスクリーンショットのように App Service の認証メニューより確認することが出来ます。

![image-a594edbc-74c8-417d-997f-ff1434715d41.png]({{site.baseurl}}/media/2023/10/image-a594edbc-74c8-417d-997f-ff1434715d41.png)

上記の構成ファイルの修正は以下の公式ドキュメントにも記載がございます。
<br><参考: JavaScript アプリの構成>
<br>https://learn.microsoft.com/ja-jp/azure/active-directory/develop/single-page-app-quickstart?pivots=devlang-javascript#step-3-configure-your-javascript-app

### 手順４：Microsoft Entra ID に登録された SPA アプリケーションに対して、App Service のアクセス許可を追加する

Microsoft Entra ID に登録された SPA アプリケーションの「API のアクセス許可」メニューより、「アクセス許可の追加」をクリックします。
「API アクセス許可の要求」画面にて、「自分のAPI」タブにてApp Service 認証で登録したアプリケーションを選択します。

<br>![image-55fc36bf-adc8-45a1-89d5-6e777b6b25af.png]({{site.baseurl}}/media/2023/10/image-55fc36bf-adc8-45a1-89d5-6e777b6b25af.png)

アクセス許可で、「user_impersonation」を選択した上で、「アクセス許可の追加」をクリックします。

<br>![image-fb0ab1f4-136e-4010-8b7d-fb316476ef8f.png]({{site.baseurl}}/media/2023/10/image-fb0ab1f4-136e-4010-8b7d-fb316476ef8f.png)

アクセス許可の追加が完了できましたら、以下のスクリーンショットのように　API/アクセス許可一覧にて App Service が追加されます。

![image-b91570b8-f6c5-499e-b7da-928328b152c8.png]({{site.baseurl}}/media/2023/10/image-b91570b8-f6c5-499e-b7da-928328b152c8.png)

### 手順５：App Service 認証の ID プロバイダーを構成する
App Service の [認証] ブレードより、Microsoft ID プロバイダー項目にて、編集アイコンをクリックして、以下の項目の値を編集します。

・発行者のURL：手順３で指定された authority の値に編集します。
```
https://login.microsoftonline.com/<Microsoft Entra ID に登録した SPA アプリケーションのテナント ID>
```

・許可されるトークン対象ユーザー：
```
api://<App Service 認証のアプリケーション(クライアント) ID>
```

![image-dd0f3398-2d55-4b3f-82e0-d1c7312f0b22.png]({{site.baseurl}}/media/2023/10/image-dd0f3398-2d55-4b3f-82e0-d1c7312f0b22.png)

### 手順６：プロジェクトを実行してアクセストークンを取得する
下記の手順によりプロジェクトを実行します。
<br>[Node.js を使用して Web サーバーでプロジェクトを実行します。](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/single-page-app-quickstart?pivots=devlang-javascript#step-4-run-the-project)
>1.サーバーを起動するために、プロジェクト ディレクトリ内から次のコマンドを実行します。
<br>npm install
<br>npm start
<br>2.「 http://localhost:3000/ 」を参照してください。
<br>3.[サインイン] を選択してサインイン プロセスを開始してから、Microsoft Graph API を呼び出します。
<br>初回サインイン時に、自分のプロファイルにアプリケーションがアクセスして自分をサインインさせることについて同意を求められます。 
<br>正常にサインインすると、ユーザー プロファイル情報がページに表示されます。

正常にサインインさせると、以下のスクリーンショットのようにアクセストークンを取得できます。

![image-8399449f-655d-4515-b4ea-ab45db1e1f5d.png]({{site.baseurl}}/media/2023/10/image-8399449f-655d-4515-b4ea-ab45db1e1f5d.png)

サインインさせた際に、「管理者の承認が必要」または、「要求されているアクセス許可」というメッセージが表示された場合には、アプリケーション管理者による承認が必要です。
<br>手順の詳細については、以下ドキュメントをご参照ください。
<br><参考:「管理者の承認が必要」のメッセージが表示された場合の対処法>
<br>https://jpazureid.github.io/blog/azure-active-directory/azure-ad-consent-framework/

### 手順７：取得したアクセストークンを App Service 認証に POST して App Service 独自の認証トークン(セッショントークン)を取得する
下記の形式でアクセストークンを Body に指定して、`https://App Service名.azurewebsites.net/.auth/login/aad` にPOSTして、App Service 独自の認証トークン(セッショントークン) authenticationToken の値を取得します。

[参考: クライアント主導型サインイン](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-customize-sign-in-out#client-directed-sign-in)
```
{"access_token":"<手順6で取得したaccess_token>"}
```
下記が Postman ツールで試した結果の例です。

![image-6641cca9-558e-49c6-917b-724d5ce2fca1.png]({{site.baseurl}}/media/2023/10/image-6641cca9-558e-49c6-917b-724d5ce2fca1.png)

### 手順８：取得した App Service 独自の認証トークン(セッショントークン)を利用して、App Service にアクセスする

手順7で取得した authenticationToken の値をヘッダー "X-ZUMO-AUTH" で指定した上で、App Service のURL（`https://App Service 名.azurewebsites.net`）に対してGETを実行します。
<br>Postman ツールで試してみると、以下の様に App Service でホストしているサイトから HTTP 200 応答が返却され、正常にアクセスできていることが確認できます。

![image-0e2df657-cdb1-4533-8bbf-81bba2dbe043.png]({{site.baseurl}}/media/2023/10/image-0e2df657-cdb1-4533-8bbf-81bba2dbe043.png)
---

上記の認証トークンについて以下のドキュメントにて記載がございますので併せてご参照ください。

<参考: Authentication flow>
<br>https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#authentication-flow


<br>
<br>

---

<br>
<br>

2023 年 10 月 23 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>