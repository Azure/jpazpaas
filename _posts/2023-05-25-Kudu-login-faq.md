---
title: Kudu の認証についてよくあるご質問
author_name: "Takeharu Oshida"
tags:
    - App Service
    - Function App
    - Kudu
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

本記事では App Service および Azure Functions で SCM サイトとして提供されている Kudu の認証に関する FAQ について記載します。

Kudu の機能そのものについては下記記事もご参考ください。

- [Kudu サービスの概要 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/resources-kudu)
- [Kudu サイトの使い方 (Tips 4 選) - Japan PaaS Support Team Blog](https://jpazpaas.github.io/blog/2022/11/28/How-to-use-Kudu-site.html)

# Kudu へ認証について

## Q: Kudu の認証はいつ行われますか？
**A** : Kudu の認証は、ブラウザを利用してポータルの高度なツールより、https://<AppName>.scm.azurewebsites.net にアクセスして、Kudu UI を利用する場合や、curl 等の HTTP クライアントで
[Kudu が提供する API](https://github.com/projectkudu/kudu/wiki/REST-API) を利用する際に必要となります。
代表的な API には デプロイAPI があります。`az cli` や VS Code 、GitHub Actions からのデプロイ時にもこれらの API が利用されることになります。

参考:

- [App Service にファイルをデプロイする - Azure App Service]https://learn.microsoft.com/ja-jp/azure/app-service/deploy-zip?tabs=api#deploy-a-zip-package

## Q: Kudu の認証はどのように行われますか？
**A**: Kudu の認証は AAD(Azure Active Directory) を用いたシングルサインオン、と [Deployment－credentials（発行プロファイル）](https://github.com/projectkudu/kudu/wiki/Deployment-credentials) を用いた Basic 認証の2つの方法があります。

ブラウザから Kudu UI にアクセス時には標準では AAD によるシングルサインオンが行われ、AAD のログイン画面に遷移します。
ブラウザからのアクセス時に Basic 認証を利用したい場合は、https://<AppName>.scm.azurewebsites.net/basicauth にアクセスすることで Basic 認証を用いることが可能となっています。

AZ CLI については `2.48.1` からは Kudu アクセス時に AAD を優先して利用するようになっております。

GitHub Actions を用いたデプロイにおいても、発行プロファイルを用いた Basic 認証と、Service principal を用いた AAD 認証のいずれかを選択することができます。

参考:

- [Accessing the kudu service · projectkudu/kudu Wiki](https://github.com/projectkudu/kudu/wiki/Accessing-the-kudu-service#authentication)
- [Release notes & updates – Azure CLI](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli#april-25-2023)
- [Azure/webapps-deploy: Enable GitHub developers to deploy to Azure WebApps using GitHub Actions](
https://github.com/Azure/webapps-deploy)

# Kudu へ AAD を利用したログインについて
## Q: AAD を利用する場合に必要となる権限を教えてください。

**A** : 組み込みの Azure ロールを用いる場合は「所有者」あるいは、「Contributor」ロールが必要となります。「[Website Contributor](Website Contributor)」も下記の権限を含むためアクセス可能となります。
カスタムロールを用いる場合、リソースプロバイダー操作としては、`microsoft.web/sites/publish/action`、`microsoft.web/sites/slots/publish/action` の 2 つが必要となります。

参考:

- [Azure リソース プロバイダーの操作](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/resource-provider-operations#microsoftweb)


## Q: Kudu への AAD ログイン時にリダイレクトされる URL を教えてください。

**A** : 現時点で AAD でのログイン時には、下記ドキュメントの 「ID 56」、「ID 59」、「ID 125」に該当するエンドポイントが利用されます。

[Office 365 URL および IP アドレス範囲 - Microsoft 365 Enterprise](https://learn.microsoft.com/ja-jp/microsoft-365/enterprise/urls-and-ip-address-ranges?view=o365-worldwide#microsoft-365-common-and-office-online)


上記に加えて以下のエンドポイントが利用されます。

- `*.sso.azurewebsites.windows.net`
- `*.scm.azurewebsites.net`

Kudu にアクセスするクライアント環境において、アウトバウンド通信を制御している場合もあると存じます。
AAD でのログイン認証時には、下記のエンドポイントに対してクライアントが疎通できる必要がございます。
プロキシー等をご利用の場合は、エンドポイントへの疎通を許可してください。

# Kudu へ Basic 認証ログインについて
## Q: Basic 認証に用いる User/Password はどこで確認できますか
**A**: Basic 認証には、「ユーザー レベルの資格情報」、「アプリ レベルの資格情報」 2 種類の資格情報を利用することができます。

「ユーザー レベルの資格情報」はその名の通り、Azure にログインしているユーザー毎の設定となり、サブスクリプション内のすべての関数アプリと Web アプリは、同じデプロイ資格情報を共有されます。[デプロイセンター]や、 [`az webapp deployment user set`](https://learn.microsoft.com/ja-jp/cli/azure/webapp/deployment/user?view=azure-cli-latest
) コマンドにてリセットすることが可能となっております。 

「アプリ レベルの資格情報」については、[デプロイセンター]や、[`az webapp deployment list-publishing-credentials`](https://learn.microsoft.com/ja-jp/cli/azure/webapp/deployment?view=azure-cli-latest#az-webapp-deployment-list-publishing-credentials)コマンドにて確認することができます。
GitHub Actions 等で利用可能な発行プロファイルと同様の値となります。

![image-c83f3c4e-adb9-4c7a-ad6a-17845bbc3a14.png]({{site.baseurl}}/media/2023/05/image-c83f3c4e-adb9-4c7a-ad6a-17845bbc3a14.png)

ポータル上では FTPS ユーザー名として `<アプリ名>\$<アプリ名>` と記載されていますが、curl コマンド等の基本認証のユーザ名として渡す値は、$<アプリ名> となります。

例えばアプリ名が jpazpaassample、パスワードが myPassword の場合は、以下のようなコマンドとなります。

curl コマンド実行例:

`curl -u ' $jpazpaassample:myPassword' https://jpazpaassample.scm.azurewebsite,net`


参考:

- [デプロイ資格情報を構成する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/deploy-configure-credentials?tabs=cli)
- [Deployment credentials · projectkudu/kudu Wiki](https://github.com/projectkudu/kudu/wiki/Deployment-credentials)