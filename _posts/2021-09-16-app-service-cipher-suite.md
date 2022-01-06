---
title: "App Service の暗号スイート(Cipher Suite)について"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Function App
    - Web Apps
---

## 質問
- App Service で使用している暗号スイート(Cipher Suite)には何を使用しているのか。
- App Service で特定の暗号スイート(Cipher Suite)を使用しない方法はあるか。
<br>
<br>

## 回答
マルチテナントの App Servive (ASE 以外) は Load Balancer を他のお客様と共有するため、使用する暗号スイートを変更することは出来ません。

App Service (Function を含む) は内部的に以下の流れでリクエストを処理します。
```
クライアント --- [App Service: Front End --- Web Workers]
```
クライアントからのリクエストは Front End (Load Balancer) を経由し、お客様のアプリケーションが稼働している Web Worker (App Service プランに相当します) に到達します。
クライアントと App Service 間の SSL/TLS 通信は、Front End が[TLS 処理の終端](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-bindings#handle-tls-termination)となるため、Front End から Web Worker へは HTTP で通信されます。
マルチテナントの App Service では、Basic プラン以上の App Service プランを選択することで Web Worker を占有できますが、Front End を占有することはできません。

App Service のアーキテクチャ詳細は、[Inside the Azure App Service Architecture](https://docs.microsoft.com/en-us/archive/msdn-magazine/2017/february/azure-inside-the-azure-app-service-architecture) を確認ください。


### **2021 年 9 月現在の App Service の暗号スイート対応一覧**
Front End が対応している暗号スイートの要素は以下の通りです。
外部のセキュリティチェックのサイト[SSL Labs](https://www.ssllabs.com/ssltest/) で確認することができます。

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)   ECDH secp256r1 (eq. 3072 bits RSA)   FS	256
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)   ECDH secp256r1 (eq. 3072 bits RSA)   FS	128
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (0xc028)   ECDH secp256r1 (eq. 3072 bits RSA)   FS   WEAK	256
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (0xc027)   ECDH secp256r1 (eq. 3072 bits RSA)   FS   WEAK	128
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (0xc014)   ECDH secp256r1 (eq. 3072 bits RSA)   FS   WEAK	256
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (0xc013)   ECDH secp256r1 (eq. 3072 bits RSA)   FS   WEAK	128
TLS_RSA_WITH_AES_256_GCM_SHA384 (0x9d)   WEAK	256
TLS_RSA_WITH_AES_128_GCM_SHA256 (0x9c)   WEAK	128
TLS_RSA_WITH_AES_256_CBC_SHA256 (0x3d)   WEAK	256
TLS_RSA_WITH_AES_128_CBC_SHA256 (0x3c)   WEAK	128
TLS_RSA_WITH_AES_256_CBC_SHA (0x35)   WEAK	256
TLS_RSA_WITH_AES_128_CBC_SHA (0x2f)   WEAK	128
```

現時点で、マルチテナントの App Service は、下位互換性のため、暗号強度の弱い暗号スイートもサポートしています。
<br>
<br>
## 回避方法
特定の暗号スイートのみを利用するには、以下の 2 つの方法があります。

### **1) App Service の前段に Application Gatewayを設置し、Application Gateway 側で暗号スイートのカスタマイズを行う**
Application Gateway を利用することで、カスタム TLS ポリシーが設定できます。
設定方法は、下記より確認ください。

[カスタム TLS ポリシーを構成する](https://docs.microsoft.com/ja-jp/azure/application-gateway/application-gateway-configure-ssl-policy-powershell#configure-a-custom-tls-policy)

```
# set the TLS policy on the application gateway
Set-AzApplicationGatewaySslPolicy -ApplicationGateway $gw -PolicyType Custom -MinProtocolVersion TLSv1_1 -CipherSuite "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256", "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384", "TLS_RSA_WITH_AES_128_GCM_SHA256"
```

### **2) 占有環境となる App Service Envieonment (ASE) で、暗号スイートのカスタマイズを行う**
ASE は占有インスタンスを使用するため、既定の暗号スイートを変更することができます。
設定方法は、下記より確認ください。

[TLS 暗号スイートの順序変更](https://docs.microsoft.com/ja-jp/azure/app-service/environment/app-service-app-service-environment-custom-settings#change-tls-cipher-suite-order)

```
ASE では、既定の暗号スイートの変更がサポートされています。 既定の暗号セットは、マルチテナント サービスで使用されるものと同じセットです。 暗号スイートの変更は App Service デプロイ全体に影響します。これは、シングルテナント ASE でのみ可能です。 
```

ただし、ASE の利用は、マルチテナントの App Service と比べて大幅なランニングコストの上昇が見込まれるため、価格を含めてご検討ください。

[App Service の価格](https://azure.microsoft.com/ja-jp/pricing/details/app-service/windows/)

<br>
<br>

## 質問（App Service が送信側となる場合）
- 利用可能な暗号スイート(Cipher Suite)はどのようにして確認を行うのか
- 利用可能な暗号スイート(Cipher Suite)をカスタマイズする方法はあるか

## 回答（App Service が送信側となる場合）
### 利用可能な暗号スイート(Cipher Suite)はどのようにして確認を行うのか
ご利用のApp ServiceがWindowsベースかLinunxベースかによって確認いただける方法が異なります。

#### Windowsの場合
高度なツール ＞ Debug＞ CMD より以下のコマンドを実行いただくことでご確認いただけます。

##### コマンド
```
reg query HKLM\SYSTEM\CurrentControlSet\Control\Cryptography\Configuration\Local\SSL\00010002
```

##### コマンドの実行例

![1-0-kudu-reg-query]({{site.baseurl}}/media/2021/09/2021-09-16-1-0-kudu-reg-query.png)

#### Linuxの場合
高度なツール ＞ SSH より以下のコマンドを実行いただくことでご確認いただけます。

##### コマンド
```
openssl ciphers -v
```

##### コマンドの実行例

![1-3-linux-kudu-openssl-ciphers]({{site.baseurl}}/media/2021/09/2021-09-16-1-3-linux-kudu-openssl-ciphers.png)

#### （ご参考）その他の確認方法
ご参考となりますが、実際の通信パケットを確認いただくことでもApp Serviceがクライアントとなる場合に利用可能な暗号スイートの一覧をご確認いただけます。
具体的には、以下の方法で採取いただけるパケット中の、Client Helloメッセージよりご確認いただけます。
App Service上でパケットを取得いただく方法は、Windows/Linuxベースかで異なりますため、以下にそれぞれご紹介いたします。
なお、採取したパケットの解析にはWireshark などのパケットアナライザをご利用いただくことを想定しております。

##### Windowsの場合
`問題の診断と解決` 内にございますネットワークトレース機能をご利用いただくことでパケットを採取いただけます。

###### 問題の診断と解決のネットワークトレース機能
![1-1-diagnostic-tool]({{site.baseurl}}/media/2021/09/2021-09-16-1-1-diagnostic-tool.png)

###### 取得したパケットファイルに含まれている Client Hello 内の暗号スイート
![1-2-packet-clienthello]({{site.baseurl}}/media/2021/09/2021-09-16-1-2-packet-clienthello.png)

##### Linuxの場合
App Service の 高度なツール ＞ SSH より、以下のコマンドを実行していただき Client Hello、Server Hello をフィルタリングするパケットを採取いただけます。

###### tcpdump コマンド
```
tcpdump "tcp port 443 and (tcp[((tcp[12] & 0xf0) >> 2)] = 0x16)" -w client-hello.pcap
```

###### 取得したパケットファイルに含まれている Client Hello 内の暗号スイート

![1-4-linux-kudu-tcpdump]({{site.baseurl}}/media/2021/09/2021-09-16-1-4-linux-kudu-tcpdump.png)


### 利用可能な暗号スイート(Cipher Suite)をカスタマイズする方法はあるか
クライアントから App Service に対してリクエストを送信するシナリオでは、下記の方法などで暗号スイートのカスタマイズが可能でございましたが、App Serviceがクライアントとなる場合にはカスタマイズすることができません。

[TLS 暗号スイートの順序変更](https://docs.microsoft.com/ja-jp/azure/app-service/environment/app-service-app-service-environment-custom-settings#change-tls-cipher-suite-order)

App Service (Function を含む)がクライアントとなる場合、下記のようにWeb Workersから接続先へリクエストが送信されます。
```
[App Servive: Web Workers --- Front End] --- 接続先
※[] 内が App Service となります。Front End は Web Worker の通信をプロキシする役割があります。
```

クライアントから App Service に対してリクエストを行うシナリオでは、Front Endが SSL の終端となりますため、Front Endの暗号スイートとクライアントの暗号スイートから使用されるものが決定されます。
上記でご紹介したASEにおける暗号スイートのカスタマイズ方法は、このFront Endの暗号スイートをカスタマイズするものでございました。
一方で、App Serviceがクライアントとなる場合には、Front Endが SSL の終端とはならず、Web Workers と相手先がそれぞれ利用可能な暗号スイートから使用されるものが決定されます。
このため、同様の方法によるカスタマイズが叶いません。

<br>
<br>

---

<br>
<br>

2022 年 1 月 6 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
