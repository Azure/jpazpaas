---
title: "Azure Functions で使用している CDN ドメインの移行について"
author_name: "Shota Nakano"
tags:
    - Function App
---

# はじめに
お世話になっております。App Service サポート担当の仲野です。いつも Azure Functions（関数アプリ）をご利用いただき誠にありがとうございます。Azure Functions では拡張バンドルのコンテンツを Contents Delivery Network（CDN） を使用して取得しています。これまで使用していた CDN プロバイダーの運用の停止に伴い、Azure Functions では新たな CDN よりコンテンツの配布を行うようになりました。本記事は CDN ドメインの移行の詳細、それに伴う影響についてを解説します。

# Azure Functions で使用している CDN 移行について
[Azure CDN from Edgio の廃止に関する FAQ](https://learn.microsoft.com/ja-jp/previous-versions/azure/cdn/edgio-retirement-faq) にて言及されているように、2025 年 1 月 15 日に Edgio 社にて提供されているサービスが終了され、それに伴い CDN サービスの移行が行われました。移行前後で使用されている CDN のドメインは下記の通りとなります。

**移行前の CDN ドメイン**
* functionscdn.azureedge.net
* functionscdnstaging.azureedge.net


**移行後の CDN ドメイン**
* cdn.functions.azure.com
* cdn-staging.functions.azure.com


# 各種ツールの CDN ドメイン対応状況
Azure Functions にて提供されている各種ツールにおいても CDN ドメインの移行対応が完了しています。例えば、Core Tools や Functions Host では、それぞれ下記のバージョン以降は CDN ドメインの移行がされています。
 
## Azure Functions Core Tools
*  Azure Functions Core Tools：[4.0.7030](https://github.com/Azure/azure-functions-core-tools/releases/tag/4.0.7030) 以降
  
   更新内容は [こちら](https://github.com/Azure/azure-functions-core-tools/pull/4265) からご確認いただけます。
 
   ローカル環境で動作している Core Tool のバージョンを確認する例として、コマンド プロンプトにて `func --version` を実行いただくことでご確認いただけます。

   ![image-3209480f-4274-4762-a67a-5dfc2b98453e.png]({{site.baseurl}}/media/2025/09/image-3209480f-4274-4762-a67a-5dfc2b98453e.png)

## Functions Host
* Functions Host：[4.1038.100](https://github.com/Azure/azure-functions-host/releases/tag/v4.1038.100) 以降
  
   更新内容は Github の Issue（[#10816](https://github.com/Azure/azure-functions-host/pull/10816)、[#10817](https://github.com/Azure/azure-functions-host/pull/10817)、[#10824](https://github.com/Azure/azure-functions-host/pull/10824)、[#10847](https://github.com/Azure/azure-functions-host/pull/10847)）よりご確認いただけます。

  
  
   動作している Functions Host のバージョンは Azure Portal にて対象の Azure Functions の [概要] ブレードよりご確認いただけます。 

   ![image-d2130638-996a-4210-90c5-abb8479f6337.png]({{site.baseurl}}/media/2025/09/image-d2130638-996a-4210-90c5-abb8479f6337.png)


# CDN 移行に伴う影響について

## 影響
CDN ドメインの移行は既に完了しており、旧 CDN プロバイダーにて使用されていた移行前のドメインについては動作保証されない状況となります。そのため、執筆時点において使用しているドメインの移行が完了していない場合、Azure Functions が正常に動作しなくなる可能性があります。

以下のようなユースケースで影響が発生しますので、お手元の環境に該当するものがないかご確認ください。

* ユーザー コードにて移行前のドメインに直接的な依存を設けている場合
* Firewall 等で対象の移行後のドメインが通信許可されていない場合
* Core Tools や Functions Host などのバージョンが最新化されていない場合


## 対策
移行後のドメインは、移行前のドメインとエンドポイント互換性があるため、移行前のドメインから使用可能なすべてのリソースは、移行後のドメインを通じて使用可能なため速やかに下記の対策をご検討ください。

* 対象のドメインへ直接的な依存関係がある場合には移行後のドメインに更新ください。
* Firewall 等で移行後のドメインが通信許可されるよう設定ください。
* Core Tools や Functions Host などのバージョンを最新化ください。


# 参考ドキュメント
[Azure CDN from Edgio の廃止に関する FAQ](https://learn.microsoft.com/ja-jp/previous-versions/azure/cdn/edgio-retirement-faq)

[Clarification on Migration of functionscdn.azureedge.net Subdomain](https://github.com/Azure/Azure-Functions/issues/2576)

[Critical Notice: Bundles are now hosted from cdn.functions.azure.com](https://github.com/Azure/azure-functions-extension-bundles/issues/474)

[Critical Notice: Core Tools is now hosted from cdn.functions.azure.com](https://github.com/Azure/azure-functions-core-tools/issues/4260)

<br>
<br>

---

<br>
<br>

2025 年 09 月 05 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>