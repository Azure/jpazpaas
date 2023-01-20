---
title: Static Web Apps におけるネットワークアクセス制限について
author_name: "Takeharu Oshida"
tags:
    - Static Web Apps
---


# はじめに
お世話になっております。App Service サポート担当の押田です。

本記事では [Azure Static Web Apps](https://learn.microsoft.com/ja-jp/azure/static-web-apps/) におけるネットワーク構成によるアクセス制限についてご紹介致します。

なお、本記事ではネットワーク構成によるアクセス制限について記載し、ユーザによる [認証と認可](https://learn.microsoft.com/ja-jp/azure/static-web-apps/authentication-authorization) についてのアクセス制限については触れません。

# Static Web Apps におけるネットワークアクセス制限方法

Static Web Apps においてアクセス制限を行う方法として、以下 2 つの方法があり、組み合わせて利用することも可能となっています。

 1. [プライベート エンドポイント](https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview)を利用する方法
 2. [staticwebapp\.config\.json](https://learn.microsoft.com/ja-jp/azure/static-web-apps/configuration)の `networking.allowedIpRanges`　を利用する方法

## 1. プライベートエンドポイントを利用する方法

Static Web Apps にプライベート エンドポイントを構成すると、VNet のプライベート IP アドレスを使用できるようになります。 このリンクが作成されると、Static Web App の 静的 Web アプリが VNet に統合されます。 その結果、静的 Web アプリはパブリック インターネットから利用できなくなり、Azure VNet 内のマシンからのアクセスのみが可能になります。

構成の詳細な手順については、下記ドキュメントをご参照ください。

[Azure Static Web Apps でのプライベート エンドポイントの構成](https://learn.microsoft.com/ja-jp/azure/static-web-apps/private-endpoint)

プライベートエンドポイントが有効な Static Web Apps に VNet 外のクライアントからアクセスすると HTTP Status Code 403 エラーとなります。
```
$ curl https://icy-bay-02e7d2a00.2.azurestaticapps.net -vvv
*   Trying 168.63.140.18:443...
* TCP_NODELAY set
* Connected to icy-bay-02e7d2a00.2.azurestaticapps.net (168.63.140.18) port 443 (#0)
～省略～
* Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
< HTTP/2 403
< content-type: text/html
< date: Fri, 20 Jan 2023 03:33:56 GMT
< cache-control: no-store
<
```

一方でプライベートエンドポイントにアクセス可能な VNet 内のクライアントにおいては、HTTP Status Code 200 OK となります。
```
root@3353a9c8bf66:/home# curl https://icy-bay-02e7d2a00.2.azurestaticapps.net -vvv
*   Trying 10.0.0.10:443...
* Connected to icy-bay-02e7d2a00.2.azurestaticapps.net (10.0.0.10) port 443 (#0)
～省略～
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
< HTTP/2 200 
< content-type: text/html
```

## 2. staticwebapp.config.json を利用する方法

`staticwebapp.config.json` は、Static Web Apps の各種設定を行うファイルとなり、通常 ワークフロー ファイルで `app_location` として設定されたフォルダー内に保存され、通常リポジトリのコードとともに管理されていることが多いものと存じます。

`staticwebapp.config.json` が適切に配置され、[`networking.allowedIpRanges`](https://learn.microsoft.com/ja-jp/azure/static-web-apps/configuration#networking) が設定されている場合、設定内で許可された範囲以外のクライアントからのリクエストに対しては 403 エラーが返却されることになります。

なお、ネットワーク構成は、Azure Static Web Apps Standard プランでのみ使用できます。

---

## プライベートエンドポイントと、`networking.allowedIpRanges` を組み合わせて利用した際の挙動について

以下では、[Application Gateway](https://learn.microsoft.com/ja-jp/azure/application-gateway/) のバックエンドとして、Static Web Apps のプライベートエンドポイントを利用した場合を例としてご紹介します。
詳細な構成内容、手順については省略させていただきますが、Application Gateway にはフロントエンドとしてパブリック IP アドレス、バックエンド プールとして Static Web Apps の プライベートエンドポイントを構成しています。

Static Web Apps は `staticwebapp.config.json` の `networking.allowedIpRanges`にて、`10.0.1.0/24` と `10.0.2.0/24` を許可している状態となります。
このときに各クライアントからのリクエストがどのような経路で到達し、許可あるいは拒否されるかを説明します

![image-c1ae88bc-597f-4955-b3cf-d64982b19aa8.png]({{site.baseurl}}/media/2023/01/image-c1ae88bc-597f-4955-b3cf-d64982b19aa8.png)

 
### 1 Application Gateway の バックエンド正常性プローブの経路（青矢印）

Application Gateway からの正常性プローブは、プライベートエンドポイントを経由して Static Web Apps に到達します。送信元 IPは Application Gateway のインスタンスのプライベート IPとなり、サブネットの範囲(`10.0.1.0/24`) のいずれかになります。networking.allowedIpRanges に `10.0.1.0/24` が含まれていない場合は　403 、`10.0.1.0/24` が含まれている場合はSWAが正常に動作していれば通常 200 応答となります。

上記例では`10.0.1.0/24` が含まれているため、正常性プローブの疎通は可能となります。

※Static Web Apps の異常によって50Xエラーの発生可能性はございますが、いったんここでは省かせていただきます。
 
参考
[Azure Application Gateway の正常性監視の概要](https://learn.microsoft.com/ja-jp/azure/application-gateway/application-gateway-probe-overview)

> * バックエンド プール内のサーバー アドレスがプライベート エンドポイントの場合、ソース IP アドレスは Application Gateway サブネットのプライベート IP アドレス空間からのものです。
 
### 2 Application Gateway を経由したリクエストの経路(オレンジ色矢印)
Application Gateway のフロントエンドを経由するリクエストは、エンドクライアントが VNet 内部かパブリックインターネット経由によらず、送信元 IPは Application Gateway のインスタンスのプライベート IPとなり、サブネットの範囲(`10.0.1.0/24`)のいずれかになります。 

Static Web Apps の観点では、`networking.allowedIpRanges` に10.0.1.0/24が含まれているかどうかが判定されます。
上記例では`10.0.1.0/24` が含まれているため、疎通は可能となります。
 
参考
[アプリケーション ゲートウェイの動作](https://learn.microsoft.com/ja-jp/azure/application-gateway/how-application-gateway-works#modifications-to-the-request)

> 内部的に解決できる FQDN またはプライベート IP アドレスが含まれている場合、アプリケーション ゲートウェイでは、インスタンスのプライベート IP アドレスを使用して、バックエンド サーバーに要求がルーティングされます。

### 3,4 Application Gateway を介さない、プライベートエンドポイント経由のリクエストの経路
 
Static Web Apps のデフォルトドメインがプライベートエンドポイントとして解決できるVNet 内のクライアントからアクセスした場合、送信元IP アドレスは、各クライアントに割り当てられるプライベートIPとなります。

ClientA(緑矢印)は送信元のIPアドレスが `networking.allowedIpRanges` に含まれているため 200 OK、
ClientB(赤破線矢印)は 403 エラーとなります。

 
### 5 デフォルトドメインでのパブリックアクセス（えんじ色破線矢印）
Static Web Apps に対してプライベートエンドポイントを有効にした場合、パブリックインターネット経由でのアクセスは無効になります。

この動作は、networking.allowedIpRanges より優先されます。

以上、 Azure Static Web Apps におけるネットワーク構成によるアクセス制限についてご紹介させていただきました。


2023 年 01 月 20 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
