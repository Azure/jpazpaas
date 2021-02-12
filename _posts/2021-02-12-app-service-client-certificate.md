---
title: "App Service にてクライアント証明書モードを有効にし大量のデータを送信したが通信ができない"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Function App
    - Web Apps
---

App Service の `構成` 内には、Web Apps や Functions Apps がクライアント証明書を受け入れるための設定がございます。デフォルトでは、`無視` となっており送信元となるアプリケーションがクライアント証明書を設定する必要はございませんが、`必須` にするとクライアント証明書を付与する必要がございます。

 <img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-12-clientcertificate.png" width="70%">

クライアント証明書モードを 必須 に設定を行い、大量のデータ（約 100 KB 以上）を POST するとユーザ側では長時間の通信が発生してしまい、HTTP Status Code 403 が返却されます。しかし、少量のデータ（約 100 KB 以下）を POST すると通信に成功します。この動作の原因としては、App Service の内部に存在する内部ロードバランサーの仕様になります。詳しい情報は以下の内容をご覧ください。

## 質問
App Service (Web Apps, Function App) の構成に存在する着信クライアント証明書を必須に設定し、大量のデータを POST 送信したところ通信ができなくなります。原因と改善策を教えてください。

## 回答
App Service (Web Apps, Function App) にて、クライアント証明書機能を必須にした状態にて、大量データを送信しました通信状況を確認しますと、App Service にございます 内部ロードバランサーにリクエスト到達後、アプリケーションがコードが実行されております Web Worker にリクエストは到達しておりませんでした。内部ロードバランサーから変更された HTTP Status Code を確認しましたところ、403.72.995 (Status, SubStatus, Win32 Status) が記録されておりました。
App Service の内部ロードバランサーは、IIS サーバを使用しており、クライアント証明書機能を必須にし、大量のデータを POST や PUT にて送信する場合に通信が行われなくなります。詳細につきましては、以下の弊社提供のブログがございますのでご参考ください。

[Posting a large file can fail if you enable Client Certificates](https://docs.microsoft.com/ja-jp/archive/blogs/waws/posting-a-large-file-can-fail-if-you-enable-client-certificates)

### 改善策
これらの問題を回避するには、HTTP のヘッダ情報に、`Expect: 100-continue` を付与することで、内部ロードバランサーから 100-continue 応答が送信するまで待機を行い、POST するデータの送信はその応答後に行われます。

### クライアントとサーバ間の通信
Azure の App Service にございます web.config より、Expect: 100-continue ヘッダの設定を行うことができますが、こちらの設定はクライアントのアプリケーションから、Expect: 100-continue ヘッダ を受け付ける必要があるかという設定になり、既定値では true となります。そのため、実際に、Expect: 100-continue ヘッダ を送信する場所はクライアントのアプリケーションでなければなりません。

[servicePointManager 要素 (ネットワーク設定)](https://docs.microsoft.com/ja-jp/dotnet/framework/configure-apps/file-schema/network/servicepointmanager-element-network-settings)

> expect100Continue	POST メソッドがサーバーからの応答を受信する必要があるかどうかを指定し 100-continue ます。 既定値は true です。

```web.config
  <system.net>
     <settings>
        <servicePointManager expect100Continue="true"/>  
     </settings>
  </system.net>
```

大量のデータを送信する前には、Expect: 100-Continue ヘッダ を App Service へ送信をし、クライアントアプリケーションは、 HTTP Status Code 100 を受け取ります。その後、クライアントのアプリケーションは、大量のデータを送信することができます。インフラ部分である App Service のみの問題ではなく、 クライアントアプリケーション と App Service 間の通信が必要となります。

[クライアント アプリケーションで不要な 100-Continue 状態メッセージを送信しない](https://docs.microsoft.com/ja-jp/azure/architecture/best-practices/api-implementation#avoid-sending-unnecessary-100-continue-status-messages-in-client-applications)

> クライアント アプリケーションは、データを送信する前に、Expect: 100-Continue ヘッダー、データのサイズを示す Content-Length ヘッダー、空のメッセージ本文を含む HTTP 要求を送信できます。 サーバーが要求を処理する用意がある場合、HTTP 状態コード 100 (Continue) を指定したメッセージで応答する必要があります。 クライアント アプリケーションは要求を続行し、メッセージ本文にデータが含まれた完全な要求を送信できます。

### 参考情報（クライアントツールの通信状況）
代表的な API のテストツールとして以下の 2 つが使用されますが、クライアントツールの通信の挙動が異なりますためご注意ください。

#### curl
curl-7.13 以降のバージョンになりますと POST データを送信する場合に、自動的に Expect: 100-continue ヘッダ情報が埋め込まれ送信されるため、大量のデータの送信を意識せず行うことができます。

#### POSTMAN
Expect: 100-continue ヘッダを付与して送信を行うと、POSTMAN 側にてそのヘッダを無視して送信してしますため、大量のデータを送信した場合でも通信に失敗してしまいます。

<br>
<br>

---

<br>
<br>

2021 年 2 月 12 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>

---