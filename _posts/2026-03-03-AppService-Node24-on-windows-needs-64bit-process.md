---
title: "App Service / Azure Functions の Windows 環境において `Node.js 24` を利用する場合は 64 bit process の有効化を行ってください。"
author_name: "Takeharu Oshida"
tags:
    - App Service
    - Web Apps
    - Function App
date: 2026-03-03 00:00:00
---

# はじめに
お世話になっております。Azure App Service サポート担当の押田です。
本記事では、App Service / Azure Functions の Windows 環境において、 `Node.js 24` を利用する際の注意点を記載します。
結論から先にお伝えしますと、**プラットフォームの設定** にて `64bit` プロセスの有効化をしていただく必要がございます。

# Windows 環境で `Node.js 24` を利用する方法

App Service / Azure Functions の Windows 環境においては、Azure ポータルから `Node.js 24` が選択可能な状態となっています。

![image-d6ac67a5-c68b-4528-afb7-bc663ce85014.png]({{site.baseurl}}/media/2026/03/image-d6ac67a5-c68b-4528-afb7-bc663ce85014.png)

ここで選択した Node.js バージョンは、App Settings `WEBSITE_NODE_DEFAULT_VERSION` に `~24` を指定する動作となります。
App Service では `WEBSITE_NODE_DEFAULT_VERSION` に指定された値に応じて、プラットフォーム側があらかじめ配置したバイナリに対してパスが通るような仕組みとなっています。
ただし、32 bit プロセス用(`C:\Program Files (x86)\nodejs` 配下) には 24 系のバイナリファイルが用意されていません。24 系のバイナリは 64 bit プロセス用(`C:\Program Files\nodejs` 配下) のみに提供されているものとなります。

## 64 bit プロセスを有効にする

### ポータルから操作する

App Service / Azure Functions いずれもポータルでは [構成] メニューから変更可能です。

![image-113ac236-99be-41fa-94ae-9be9478b0e67.png]({{site.baseurl}}/media/2026/03/image-113ac236-99be-41fa-94ae-9be9478b0e67.png)
 
### Az CLI から操作する

[`az webapp config set`](https://learn.microsoft.com/ja-jp/cli/azure/webapp/config?view=azure-cli-latest#az-webapp-config-set)

```bash
az webapp config set \
  --resource-group <RG名> \
  --name <WebApp名> \
  --use-32bit-worker-process false
```


あるいは [`az functionapp config set`](https://learn.microsoft.com/ja-jp/cli/azure/functionapp/config?view=azure-cli-latest#az-functionapp-config-set) を利用します。

```bash
az functionapp config set \
  --resource-group <RG名> \
  --name <WebApp名> \
  --use-32bit-worker-process false
```

## Node 24 が利用されていることを確認する

Kudu Process Explorer から node プロセスの情報を確認することができます。

![image-ad6881ba-231f-4969-ac15-9dea34bb221a.png]({{site.baseurl}}/media/2026/03/image-ad6881ba-231f-4969-ac15-9dea34bb221a.png)

## 32 bit プロセスを指定した場合のエラー例

### App Service の場合

IIS Node で実行する Node プロセスが見つからないため、500 エラーとなります。
下記ドキュメント [node アプリケーションが起動しない] シナリオに該当します。

- [Node.js のベスト プラクティスとトラブルシューティング ガイド - Azure](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/app-service-web-nodejs-best-practices-troubleshoot-guide#iisnode-http-status-and-substatus)
- [Avoiding hardcoding Node versions on App Service Windows](https://azureossd.github.io/2022/06/24/Avoiding-hardcoding-Node-versions-on-App-Service-Windows/)

### Azure Functions の場合

Node Language Worker プロセスを起動できないため、Functions Host が正常起動しません。

[概要] に下記のように表示されます。

![image-ac42b9df-47ed-44b6-b146-8dd4c3acf5aa.png]({{site.baseurl}}/media/2026/03/image-ac42b9df-47ed-44b6-b146-8dd4c3acf5aa.png)

また、`exceptions` テーブルには、`RecordAndThrowExternalStartupException` として、`node exited with code 1 (0x1)` といった情報が確認できます。

![image-23cd3aef-a60e-48fa-aef4-5594aa08bf47.png]({{site.baseurl}}/media/2026/03/image-23cd3aef-a60e-48fa-aef4-5594aa08bf47.png)

# 参考ドキュメント

- [言語ランタイム サポート ポリシー - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/language-support-policy?tabs=windows)
- [Node.js Apps の構成 - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/configure-language-nodejs?pivots=platform-windows#set-the-nodejs-version)
- [App Service アプリを構成する - Azure App Service](https://learn.microsoft.com/ja-jp/azure/app-service/configure-common?tabs=portal#configure-general-settings)
- [Azure Functions 用 Node.js 開発者向けリファレンス](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-node?tabs=javascript%2Cwindows%2Cazure-cli&pivots=nodejs-model-v4#supported-versions)
- [app-service-linux-docs/Runtime_Support/node_support.md at master · Azure/app-service-linux-docs](https://github.com/Azure/app-service-linux-docs/blob/master/Runtime_Support/node_support.md)
- [Node.js — Node.js リリース](https://nodejs.org/ja/about/previous-releases)
- [Node.js — Node.js v22 to v24](https://nodejs.org/en/blog/migrations/v22-to-v24)

<br>
<br>
2026 年 03 月 03 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>