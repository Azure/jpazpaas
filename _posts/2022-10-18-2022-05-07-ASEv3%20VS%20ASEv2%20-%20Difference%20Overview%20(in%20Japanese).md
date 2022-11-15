---
title: "ASEv3 VS ASEv2 - Difference Overview (日本語訳版)"
author_name: "Atsushi Omachi"
tags:
    - App Service Environment
---
App Service Environment (以下、 ASE) の導入をご検討いただいているお客様より、 ASEv2 と ASEv3 の主な違いについてご質問をいただくことがございます。  
以下の弊社エンジニアが執筆した記事に情報がよくまとまっておりますため、ご案内の際に参考情報としてご紹介差し上げることがございますが、 2022 年 5 月現在では英語表記のみとなっております。

App Service environment difference, ASEv3 VS ASEv2  
https://techcommunity.microsoft.com/t5/apps-on-azure-blog/asev3-vs-asev2-difference-overview/ba-p/3276225

そのため、上記記事の日本語訳版 (一部意訳を含む) を以下に掲載いたします。少しでもご参考になれば幸いです。  
なお、以下の内容は 2022 年 8 月 9 日時点の元記事の内容を、 2022 年 9 月 10 日時点に翻訳した内容となりますので、ご注意ください。

---

App Service Environment v3 (ASEv3) は 2021 年 7 月から 一般提供が開始されています。そして [ASEv1 と ASEv2 は 2024 年 8 月 31 日 をもって廃止となります](https://azure.microsoft.com/ja-jp/updates/app-service-environment-v1-and-v2-retirement-announcement/)。   

ASEv3 は ASEv1 や ASEv2 と比べ、[様々な利点や異なる機能](https://docs.microsoft.com/ja-jp/azure/app-service/environment/overview#feature-differences)を有しています。
予期せぬ問題を防ぐため、 ASEv3 への移行を行う前にそれらの機能をご確認いただくようお勧めしております。  

多くのお客様から ASEv3 の変更点についてお問い合わせやフィードバックを頂いておりますので、この記事では ASE のバージョンごとの違いについてご紹介いたします。少しでもお客様のご参考になれば幸いです。
 
## 移行方法
ASEv2 → ASEv3: 自動マイグレーションが利用可能です (https://docs.microsoft.com/ja-jp/azure/app-service/environment/migrate)  
ASEv1 → ASEv3: 手動での移行が必要となります (https://docs.microsoft.com/ja-jp/azure/app-service/environment/migration-alternatives)

## 公開ドキュメント
ASEv1: https://docs.microsoft.com/ja-jp/azure/app-service/environment/app-service-app-service-environment-intro  
ASEv2: https://docs.microsoft.com/ja-jp/azure/app-service/environment/intro  
ASEv3: https://docs.microsoft.com/ja-jp/azure/app-service/environment/overview

## バージョンごとの違いの概要

|機能| ASEv1 | ASEv2 | ASEv3 |
|--|--|--|--|
| ASE 内の各リソースのマニュアル管理 (フロントエンド、ワーカー、 IP ベースの TLS/SSL バインディング用の IP アドレス) | 可(必要) | 不可(不要) | 不可(不要) |
| ワーカープール (ASP のスケール前のワーカープーリング) | 可 | 不可 | 不可 |
| **NSG での依存ネットワークの許可** | **必要** | **必要** | **不要** |
| フロントエンドのマニュアルスケール | 可 | 可 | 不可 (自動でスケールされるため不要) |
| スケール実行中に他のスケール操作がブロックされるか | される | される | されない |
| プライベートエンドポイントの利用 | 不可 | 不可 | 可 |
| ゾーン冗長 | 不可 | [ゾーンの固定](https://docs.microsoft.com/ja-jp/azure/app-service/environment/zone-redundancy)が可能 | [ゾーン冗長](https://docs.microsoft.com/ja-jp/azure/app-service/environment/overview-zone-redundancy)が可能 (リージョン内の 3 つのゾーンにインスタンスが分散される。 ASE 作成時に設定が必要。一部リージョンでのみ利用可能) |
| Dedicated ホストグループへのデプロイ | 不可 | 不可 | 可 (ただし、ゾーン冗長とは併用不可) |
| グローバル VNet ピアリング経由で ILB ASEv3 内のアプリにアクセスできるか | 不可 | 不可 | 可 |
| スケーリング速度 | 低 | 中 | 高 |


| 制限 (開発中も含む) | ASEv1 | ASEv2 | ASEv3 |
|--|--|--|--|
| FTP(S)の利用<br>([有効化](https://docs.microsoft.com/ja-jp/azure/app-service/environment/configure-network-settings#ftp-access)することで利用可能) | 可 | 可 | 可 (2022 年 4 月以降) |
| リモートデバッグの利用<br>([有効化](https://docs.microsoft.com/ja-jp/azure/app-service/environment/configure-network-settings#remote-debugging-access)することで利用可能) | 可 | 可 | 可 (2022 年 4 月以降) |
| SMTP 通信 | 可 | 可 | 可 |
| ネットワークウォッチャーもしくは NSG フロー ログによる通信の監視 | 可 | 可 | 不可 (開発中) |
| IP ベースの TLS/SSL バインディング | 可 | 可 | 不可 |
| カスタムドメイン サフィックスの設定 | 可 | 不可 | 可 |
| ファイアウォールが有効化されたストレージアカウントを利用したバックアップ/リストア | 可 | 可 | 不可 |


| 価格関連情報 | ASEv1 | ASEv2 | ASEv3 |
|--|--|--|--|
| 最大インスタンス数 | 55 ホスト (フロントエンド + ワーカー) | ASP 毎に 100 台 (合計最大 200 台) | ASP 毎に 100 台 (合計最大 200 台) |
| 課金対象 | vCPU 毎への課金 (ワークロードをホストしていないワーカーを含みます) | スタンプ料金 + ASP の料金 | ASP の料金 (ASP がなく ASE が空の場合は 1 インスタンス分の料金が発生します) |
| プラン形式 | vCPU 毎への課金 (ワークロードをホストしていないワーカーを含みます) | Isolated プラン | Isolated v2 プラン |
| スタンプ料金 | あり | あり | なし |
| リザーブド インスタンス(1年/3年) の適用 | 不可 | 不可 | 可 |

## ASEv3 で変更されていない部分(一部) (マルチテナントの App Service との比較を含みます):
1. アプリケーションのネットワーク トラフィック(受信, 送信)を細かく制御できます
2. 大規模な環境・分離されセキュアなネットワーク・潤沢なメモリ・高い RPS (requests per seconds) 処理能力が利用可能です
3. 商用環境の構築の際には、十分なアドレス量を確保できるよう、 256 アドレスが利用可能な **/24** のサブネットへの ASE 作成を推奨しております。なお、 /27 (32 アドレス) が最小サイズとなっております。
4. [App Service マネージド証明書](https://docs.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal#create-a-free-managed-certificate)は ASE では利用できません
5. 外部 ASE の DNS は自動的にパブリック DNS となります。内部 ASE の DNS は設定が必要です。

## Q&A
### 1. ASEv3 における「VNet へのネットワーク依存がない」とはどういう意味ですか？

ASEv3 では ASEv2 以前とは異なり、ネットワーク依存関係のために NSG の受信/送信ルールを設定する必要がありません。  
ASE 基盤内の構成変更により、 ASE が依存するネットワーク関係についてはお客様の VNet ではなく Azure バックボーン ネットワークを経由するようになったためです。


https://docs.microsoft.com/ja-jp/azure/app-service/environment/overview#feature-differences


最小で必要となる設定は**アプリケーションが行う通信**のみです。
- 受信ルール
  - VNet から ASE のサブネットへの 80/443 ポートへの通信の許可 (アプリケーションへの通信の許可)
  - Azure Load Balancer から ASE のサブネットへの 80 ポートへの通信の許可 (内部で行われる正常性チェックの許可)
- 送信ルール
  - お客様のアプリケーションが行う外部通信に応じて設定してください

![XinyuShan_0-1649147379514-e295dd0a-861b-4cdf-ab20-7e459a4754f0.png]({{site.baseurl}}/media/2022/10/XinyuShan_0-1649147379514-e295dd0a-861b-4cdf-ab20-7e459a4754f0.png)
![XinyuShan_1-1649147379518-70fc6ec6-f8a8-4d9d-ac00-e00fe1e0b8f4.png]({{site.baseurl}}/media/2022/10/XinyuShan_1-1649147379518-70fc6ec6-f8a8-4d9d-ac00-e00fe1e0b8f4.png)

#### 参考
- ASEv2 での NSG ルールについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/network-info#inbound-and-outbound-dependencies
- ASEv3 での NSG ルールについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/networking#ports-and-network-restrictions


### 2. なぜ ASEv3 ではファイアウォールが有効化されたストレージアカウントでバックアップ/リストアが実行できないのですか？

こちらは、 ASE 基盤内の構成変更のためです。 ASE が依存する通信をお客様の VNet を経由しないよう分離したことで、通信が Azure バックボーンネットワークを経由するようになりました。  


バックアップは内部で動作しているコントローラー ロールと呼ばれるマシンから実行される管理操作となります。  
コントローラー ロールは **Azure 基盤内部で利用される VNet から割り当てられた IP アドレスを利用してストレージ アカウントへアクセス**するため、ストレージ アカウントのファイアウォール設定で ASE のサブネットを許可しても、コントローラー ロールからはストレージ アカウント へアクセスできるようにはなりません。

### 3. 「ASEv3 では IP ベースの TLS/SSL バインディングがサポートされない」とはどういう意味ですか？[証明書](https://docs.microsoft.com/ja-jp/azure/app-service/environment/overview-certificates)はまだサポートされているように見えます。 IP ベースのバインディングを使用できないと、一般的な利用方法ができないのでしょうか？

SNI ベースのバインディングは引き続きサポートされます。 IP ベースのバインディングは ASEv1 と ASEv2 ではサポートされますが、 ASEv3 では現時点でサポートされていません。  

マルチテナントの環境では、**他のお客様と共有されない固有の受信 IP アドレス**を保持するため [IP ベースの SSL バインディング](https://azure.microsoft.com/ja-jp/pricing/details/app-service/windows/)が利用できます。  
しかし ASE はシングルテナントのため、既に共有されない受信 IP アドレスを持っています。  

ASEv1 では、 ASE で使われるすべてのリソースをお客様が管理する必要があり、IP SSL アドレスを取得することが可能でした。  
また、 IP SSL を利用できる ASE 内で IP アドレス数を変更することはスケール操作の一部でした。

![XinyuShan_0-1649149510892-20250540-1091-4ec1-b1a5-7a1a69e8bcf3.png]({{site.baseurl}}/media/2022/10/XinyuShan_0-1649149510892-20250540-1091-4ec1-b1a5-7a1a69e8bcf3.png)

ASEv2 でも ASEv1 と同様に IP アドレスの割り当てがサポートされていますが、[いくつかの要件](https://docs.microsoft.com/ja-jp/azure/app-service/environment/network-info#app-assigned-ip-addresses)がございます。

しかし、 ASEv3 ではすべてのアプリの受信 IP アドレスはただ 1 つのみとなりました。現時点では ASEv3 において IP アドレスの割り当て機能のサポート予定時期は未定です。

- ASEv1 の複数 IP アドレスについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/app-service-web-configure-an-app-service-environment#settings
- ASEv2 の IP アドレス割り当てについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/network-info#app-assigned-ip-addresses

### 4. App Service にはカスタムドメイン機能がありますが、 ASE の「カスタムドメイン サフィックス」機能とは何ですか？

カスタムドメイン サフィックス機能は ASE 固有のものであり、 App Service に対するカスタムドメインの設定とは異なります。  
カスタムドメイン サフィックスは ASEv1 および [ASEv3 (2022年8月以降)](https://azure.microsoft.com/ja-jp/updates/general-availability-azure-app-service-environment-v3-support-for-custom-domain-suffix/?utm_source=dlvr.it&utm_medium=twitter) でサポートされていますが、 ASEv2 では削除された機能です。

カスタムドメイン サフィックスは ASE で使用されるルートドメインを定義します。マルチテナントの App Service では、デフォルトのルートドメインは `azurewebsites.net` です。 ILB ASE の場合、デフォルトのルートドメインは `appserviceenvironment.net` です。しかし、ILB ASE はお客様の仮想ネットワーク内部にデプロイされるため、デフォルトのルートドメインに加えて、お客様の仮想ネットワークに合わせたルートドメインを利用できます。例えば、 Contoso Corporation という架空の企業では、仮想ネットワーク内でのみ解決およびアクセス可能になるよう設計されたアプリに対して、 internal-contoso.com という既定のルート ドメインが使用されます。その場合は、この仮想ネットワーク内のアプリケーションに対しては `APP-NAME.internal-contoso.com` からアクセスすることができます。

カスタムドメイン名はアプリケーションへのアクセスには機能しますが、 scm サイトでは使用できません。 scm サイトへは `APP-NAME.scm.ASE-NAME.appserviceenvironment.net` という形式でのみアクセスが可能です。

カスタムドメイン サフィックス機能は ASE 固有のものであり、 App Service に対するカスタムドメインの設定とは異なります。カスタムドメインのバインディングの詳細については[こちら](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-custom-domain?tabs=a%2Cazurecli)をご参照ください。


- ASEv1 におけるカスタム DNS サフィックスについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/app-service-app-service-environment-create-ilb-ase-resourcemanager#creating-the-base-ilb-ase
- ASEv2 におけるデフォルト ドメイン サフィックスについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/using#app-access
- ASEv3 におけるカスタム ドメイン サフィックスについて: https://docs.microsoft.com/ja-jp/azure/app-service/environment/how-to-custom-domain-suffix?pivots=experience-azp

<br>
<br>

---

<br>
<br>

2022 年 11 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
