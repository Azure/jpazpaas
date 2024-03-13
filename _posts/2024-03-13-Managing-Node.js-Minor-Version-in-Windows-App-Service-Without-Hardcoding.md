---
title: "Windows App ServiceでNode.jsマイナーバージョンまでハードコーディングしないように"
author_name: "a-yhamada"
tags:
    - app-service
---
---


[[_TOC_]] 

---
#  はじめに

App Service では、 Node.js のバージョンは定期的に更新され、古いバージョンは削除されます。

Windows の App Service ではバージョンは環境変数で指定され、特定のバージョンをハードコーディングすると問題が発生する可能性があります。

本記事では App Service Windows 環境における　Node.js バージョンの指定方法についての諸注意を記載します。

# App Service の Node.js のサポートの現状
 
App Service では、Node.js のバージョンを管理し定期的に更新しています。これにより、最新の機能を利用できるようになり、セキュリティが向上します。

古いマイナーバージョンは移行のために一定期間保持しますが、予告なく削除する場合があります。 

現在、App Service でサポートされているノードの現在のバージョンは、以下の Azure CLI コマンドで確認できます。

 `az webapp list-runtimes --os windows | grep NODE`

ご参考: [Node.js アプリの構成 - Azure App Service | Microsoft Learn　Node.js のバージョンを表示する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-language-nodejs?pivots=platform-windows#show-nodejs-version)

ご参考: [Azure コマンド ライン インターフェイス (CLI) ドキュメント - az webapp list-runtimes](https://learn.microsoft.com/ja-jp/cli/azure/webapp?view=azure-cli-latest#az-webapp-list-runtimes)

また Node.js のバージョンは、サポート終了日 ( EOL )が設定されています。EOL に達すると、そのバージョンは、App Service のランタイム スタックの選択ドロップダウンから使用できなくなります。

各バージョンの EOL は Node.js リリース スケジュールで確認できます。

[app-service-linux-docs/Runtime_Support/node_support.md at master · Azure/app-service-linux-docs](https://github.com/Azure/app-service-linux-docs/blob/master/Runtime_Support/node_support.md)



# 現在指定している Node.js バージョンの確認方法

Windows App Service では、Node.js のバージョンは WEBSITE_NODE_DEFAULT_VERSION という環境変数で指定されています。この環境変数は、Azure ポータルや Azure CLI からでも確認できます。

## Azure ポータル

Azure ポータルから該当の Web アプリを開き左側の[環境変数]ブレードをクリックします。

![スクリーンショット%202024-02-15%20165024-de6d0b47-4557-4c32-af0a-1051d483dc16.png]({{site.baseurl}}/media/2024/03/スクリーンショット%202024-02-15%20165024-de6d0b47-4557-4c32-af0a-1051d483dc16.png)
## Azure CLI

PowerShell を開き Azure にログイン後、以下のコマンドを実行します。

`az webapp config appsettings list --name <アプリ名> --resource-group  <リソースグループ名> --query "[?name=='WEBSITE_NODE_DEFAULT_VERSION'].value"`

ご参考: [Node.js アプリの構成 - Azure App Service | Microsoft Learn　Node.js のバージョンを表示する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-language-nodejs?pivots=platform-linux#set-nodejs-version)

# Windows App Service で Node.js のマイナーバージョンをハードコーディングした場合の問題

App Service は Node.js の過去のマイナーバージョンを削除しています。マイナーバージョンをハードコーディングしたまま放置した場合、使用していたマイナーバージョンが削除されて、意図せずアプリケーションが起動しない状況を引き起こします。

## 問題が起きる記述例

**Azure ポータル** 

![2024-02-15%20165412-660e5d29-812f-4a64-8eb3-8787a07ceff5.png]({{site.baseurl}}/media/2024/03/2024-02-15%20165412-660e5d29-812f-4a64-8eb3-8787a07ceff5.png)


nodeProcessCommandLine で、node.exe のバージョンを指定した際、以下は非推奨の書き方になります。

**Web.config**

`<iisnode nodeProcessCommandLine="D:\Program Files (x86)\nodejs\12.22.12\node.exe"/>`


**iinode.yml**

`nodeProcessCommandLine: D:\Program Files (x86)\nodejs\12.22.12\node.exe`

App Service でマイナーバージョン（v12.22.12）が削除された場合、アプリが起動しなくなることが起こる可能性があります。

# 予防策

 ~18, ~20 のように、チルダ構文で特定のマイナーバージョンは指定せず、メジャーバージョンだけ指定します。

EOL に達したバージョンは使用せず、Node.js バージョン間の違いを確認し、次にサポートされている LTS バージョンにアプリケーションを移行することをお勧めします。

**Azure ポータル**

Azure ポータルから該当の Web アプリを開き左側の[構成]ブレードをクリックします。アプリケーション設定の[ WEBSITE_NODE_DEFAULT_VERSION ] に、メジャーバージョンだけチルダ構文で指定し適用を押します。

![image-073a8777-7223-463d-9470-8174660dbe38.png]({{site.baseurl}}/media/2024/03/image-073a8777-7223-463d-9470-8174660dbe38.png)

**Azure CLI**

Azure CLIで設定する場合は、以下のコマンドを実行します。

`az webapp config appsettings set --name <アプリ名> --resource-group <リソースグループ名> --settings WEBSITE_NODE_DEFAULT_VERSION=<新しいバージョン>`

例)
`az webapp config appsettings set --name <app-name> --resource-group <resource-group-name> --settings WEBSITE_NODE_DEFAULT_VERSION="~18"`

nodeProcessCommandLine にて、絶対パスで特定のマイナーバージョンをハードコーディングしている場合は以下のように変更します。

**Web.config** 

`<iisnode nodeProcessCommandLine="%ProgramFiles%\nodejs\%WEBSITE_NODE_DEFAULT_VERSION%\node.exe"/>`
 
**iinode.yml** 

`nodeProcessCommandLine: "%ProgramFiles%\\nodejs\\%WEBSITE_NODE_DEFAULT_VERSION%\\node.exe"`

[Avoiding hardcoding Node versions on App Service Windows - (azureossd.github.io)](https://azureossd.github.io/2022/06/24/Avoiding-hardcoding-Node-versions-on-App-Service-Windows/)


チルダ構文を利用してメジャーバージョンのみを指定することで、マイナーバージョン以降は、App Service 上で利用可能なものが動的に指定されることになります。アプリケーション設定では、~18 としたものが、アプリケーションの環境変数としては 18.12.1 として取得できることが確認できます。

Azure ポータルから [高度なツール] ブレードを選択、[移動] をクリックすると新しいページが開き、そのページ上部の Enviroment をクリックすると WEBSITE_NODE_DEFAULT_VERSION の値をご確認いただけます。

![スクリーンショット%202024-03-07%20172342-c516d837-1303-45db-a4fd-49d4d40e36e5.png]({{site.baseurl}}/media/2024/03/スクリーンショット%202024-03-07%20172342-c516d837-1303-45db-a4fd-49d4d40e36e5.png)


同じページの上部の Debug Console からも マイナーバージョンを含んだ WEBSITE_NODE_DEFAULT_VERSION の値を確認することができます。

![スクリーンショット%202024-03-07%20172425-784c886d-fbfb-4b5a-b3b0-8c37ab47c9c5.png]({{site.baseurl}}/media/2024/03/スクリーンショット%202024-03-07%20172425-784c886d-fbfb-4b5a-b3b0-8c37ab47c9c5.png)

 
注) Node.js のバージョンを変更すると、アプリケーションが再起動されます。ダウンタイムを最小限にするために、スロットのスワップやデプロイメントセンターなどの機能を利用することが推奨されます。


# まとめ
 
Windows App Service で特定の Node.js のマイナーバージョンまでハードコーディングするとある日突然アプリケーション起動できなくなるなど予期せぬトラブルが発生する恐れがあります。 Node.js のバージョンが推奨されている方法で指定されているか確認しておきましょう。
