---
title: "Azure Front Door を経由してAzure AD 認証を有効にしたApp Serviceへのアクセス"
author_name: "<trangmai>"
tags:
    - Azure Front Door, App Service, Azure AD認証
---

# 質問[]()
ブラウザであるクライアントから Azure Front Door を経由して正常に App Service にアクセスするには、どのように設定すればよろしいですか。
### **構成**[]()
Azure Front Door に、App Service をバックアップエンドとして追加しています。
App Service の認証において、プロバイダーが Microsoft であり Azure AD 認証を有効にしています。
App Service の受信トラフィックにおいて、Azure Front Door のトラフィックのみ許可します。
![image-58d468c8-5399-4f36-a1ca-6f91e34fa604.png]({{site.baseurl}}/media/2023/05/image-58d468c8-5399-4f36-a1ca-6f91e34fa604.png)

# 回答[]()
認証のリダイレクトURLを Azure Front Door のフロントエンドホストに変更する設定及び App Service の proxy 設定を実施することにより、App Service アプリ側で Front Door からのアクセス許可のみという構成で、ブラウザであるクライアントから Azure FrontDoor を経由して、Azure AD 認証を有効にした App Service に正常にアクセスを行うことが可能となります。
<br>手順１：認証のリダイレクトURLを Azure Front Door のフロントエンドホストに変更する
<br>手順２：App Service の proxy 設定を変更する

各手順の詳細を以下にご説明いたします。

## 手順 1. 認証のリダイレクトURLを Azure Front Door のフロントエンドホストに設定する[]()
**設定する理由：**
<br>ブラウザから Azure Front Door にアクセスして、App Service の認証を行った後、下記のように Azure Front Door ではなく、App Service の URL にリダイレクされます。
![image-e073b36d-c2a3-46a4-a880-a2047dc788cf.png]({{site.baseurl}}/media/2023/05/image-e073b36d-c2a3-46a4-a880-a2047dc788cf.png)

上記のことから、App Service アプリ側で Front Door からのアクセス許可のみ構成する場合、App Service のアクセス制限に抵触するため、App Service 認証を行った後、403 エラーが発生してしまいます。

![image-f3bddc3e-da6a-4f10-b8b5-35fdaf8edf61.png]({{site.baseurl}}/media/2023/05/image-f3bddc3e-da6a-4f10-b8b5-35fdaf8edf61.png)

そのため、上記のエラーを解消する認証のリダイレクトを Azure Front Door のフロントエンドホストに変更する必要がございます。

**設定の詳細：**
<br>1. Azure Portal で "Front Door と CDN プロファイル"を開きます。
<br>2. [概要]メニューより確認できるフロントエンド ホスト名をコピーして、txtファイルなどに貼り付けます。
![image-047835ca-f491-48b9-a54b-922859f77ab0.png]({{site.baseurl}}/media/2023/05/image-047835ca-f491-48b9-a54b-922859f77ab0.png)

<br>3. Azure Portal で App Service を開きます。
<br>4. App Service アプリの[認証]メニューより "IDプロバイダー"項目にて表示されているリンクをクリックします。
![image-53671cae-c38c-42c4-8a73-5c4a360e182f.png]({{site.baseurl}}/media/2023/05/image-53671cae-c38c-42c4-8a73-5c4a360e182f.png)

<br>5. 表示された画面において[認証]メニューより下記のようにリダイレクトURLを修正した後、”保存”ボタンをクリックします。
<br>リダイレクトURL：
<br>https://1で確認したフロントエンドホスト名/.auth/login/aad/callback 

![image-0c41ebb4-10fc-4daa-b1d5-ad3d446f2d28.png]({{site.baseurl}}/media/2023/05/image-0c41ebb4-10fc-4daa-b1d5-ad3d446f2d28.png)


## 手順 2. App Service の proxy を設定する[]()
**設定する理由：**
<br>App Service 認証（Azure AD 認証）の仕組み上では認証時の Azure AD に対するリクエスト送信元ホスト名と、リダイレクト先のホスト名を一致させる必要がございます。
<br>認証時の Azure AD に対するリクエスト送信元ホスト名が App Services のホスト名となり、リダイレクト先は Azure Front Door のホスト名となっている場合以下のようにホスト名が一致しないエラーが発生します。

![image-8b9cb8df-1f5a-4b57-a93a-f24e3b7d3f24.png]({{site.baseurl}}/media/2023/05/image-8b9cb8df-1f5a-4b57-a93a-f24e3b7d3f24.png)

App Service の proxy を設定することにより、上記の事象を解消することが可能となります。
App Service や Azure Front Door は製品ごとに異なるヘッダーが使用され、異なる forwardProxy 設定となっているため、proxy 設定し直す必要がございます。

<br>詳細は下記のドキュメンにて記載がございます。
<br>[Azure Front Door を使用する場合の考慮事項](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#considerations-when-using-azure-front-door)

**設定の詳細：**
<br>1. Azure Resource Explorer (https://resources.azure.com)　へ接続をしていただき、サブスクリプション> リソースグループ> providers > Microsoft.Web > sites > 対象のApp Service > config > authsettingsV2 へ移動をします。

<br>2.上部メニューにて、Edit ボタンを押下してauthsettingsV2 の編集を行います。
![image-8f3d7177-c12d-4ee2-bda0-33bb70a3a7e6.png]({{site.baseurl}}/media/2023/05/image-8f3d7177-c12d-4ee2-bda0-33bb70a3a7e6.png)

<br>3. 編集する内容といたしましては、forwardProxy の要素内にて以下の内容を記述します。
```
 "forwardProxy": {
    "convention": "Standard"
 }
```
![image-4062b8c1-e0b8-48ad-b1c5-e9d40d1fe8db.png]({{site.baseurl}}/media/2023/05/image-4062b8c1-e0b8-48ad-b1c5-e9d40d1fe8db.png)

4.上部メニューにて、PUT ボタンを押下することで設定が反映されます。

![image-9c72ca0f-7476-4080-a3df-bfe2907f6ecc.png]({{site.baseurl}}/media/2023/05/image-9c72ca0f-7476-4080-a3df-bfe2907f6ecc.png)

<br>上述の手順で実施した結果として以下のように Azure Front Door からアクセスして、正常に認証が完了し、App Service に正常にアクセスすることを確認することが出来ます。

![image-b121cff0-5eaa-400c-aa94-54f392e0c508.png]({{site.baseurl}}/media/2023/05/image-b121cff0-5eaa-400c-aa94-54f392e0c508.png)

![image-296a3bdf-5520-44d2-b299-3f99c60e1eaa.png]({{site.baseurl}}/media/2023/05/image-296a3bdf-5520-44d2-b299-3f99c60e1eaa.png)

**補足**
<br>
App Service にアクセス制限を構成していないシナリオでも、上述リダイレクト先のホスト名が一致しないエラーが発生する場合、手順 2 の通りApp Service の proxy を設定する必要がございます。

# 参考ドキュメント
[アプリケーション用のフロント ドアを作成する](https://learn.microsoft.com/ja-jp/azure/frontdoor/quickstart-create-front-door#create-a-front-door-for-your-application)

[Azure App Service のアクセス制限を設定する](https://learn.microsoft.com/ja-jp/azure/app-service/app-service-ip-restrictions)

[ID プロバイダー](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization#identity-providers)

[Microsoft ID プラットフォームと OAuth 2.0 認証コード フロー](https://learn.microsoft.com/ja-jp/azure/active-directory/develop/v2-oauth2-auth-code-flow)

[エラー AADSTS50011: 要求で指定されたリダイレクト URI が一致しません](https://learn.microsoft.com/ja-jp/troubleshoot/azure/active-directory/error-code-aadsts50011-redirect-uri-mismatch)

本ブログは App Service の前段に FrontDoor を構成する場合のシナリオについて記載されましたが、App Service の前段に Application Gateway を構成する場合の Azure AD 認識について以下のブログをご参考までにご参照ください。

[App Service の前段に ApplicationGateway を配置した際の Azure AD 認証](https://azure.github.io/jpazpaas/2022/03/09/Application-Gateway-front-of-App-Service-auth.html)

------------------
<br>
<br>
2023 年 02 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>
