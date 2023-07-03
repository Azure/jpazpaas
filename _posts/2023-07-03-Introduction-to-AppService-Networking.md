---
title: "App Service をセキュアに運用するためのネットワーク機能について"
author_name: "Kohei Hosono"
tags:
    - App Service
    - Web App
    - Function App
---

## はじめに

本稿では、日頃より App Service をご利用いただいているお客様やこれから App Service をご検討いただいているお客様向けに、 App Service をネットワークの観点からセキュアな環境で運用するための機能や仕組みについてご紹介します。

App Service プラットフォームの構成について知ったうえで読んでいただくことでより理解も深まるかと存じますので、もしよろしければ [App Service を構成する主要な要素とそれぞれの役割](https://jpazpaas.github.io/blog/2023/05/10/Appservice-components.html) もぜひご一読いただけますと幸いです。

App Service は既定でインターネットに公開され、 「クライアントから App Service (上のアプリケーション) へのアクセス」 や 「App Service (上のアプリケーション) からデータベース等のエンドポイントへの接続」 はそれぞれインターネット<sup>*1</sup> を経由して送信されます。また Visual Studio や Azure Pipelines 等からのアプリケーションのデプロイに関してもインターネットを経由して App Service (Kudu<sup>*2</sup>) に送信されます。

![image-bb9c0145-0290-4498-a3c0-851bb136cbe9.png]({{site.baseurl}}/media/2023/07/image-bb9c0145-0290-4498-a3c0-851bb136cbe9.png)

アプリケーションをインターネットに公開せず特定の IP アドレスからのみ許可したい場合や、アプリケーションから仮想ネットワーク内のエンドポイントに接続したいといった要件を実現するための機能・設定を App Service として提供しており、 Azure Portal の [ネットワーク] ブレードから構成することが可能です。

App Service では受信側 (クライアントから App Service へのトラフィック) と送信側 (App Service から外部エンドポイントへのトラフィック) は異なる経路で通信され、それぞれの通信を個別に制御することが可能です。

![image-2745c30a-d55f-4a06-8051-471c6e6c934a.png]({{site.baseurl}}/media/2023/07/image-2745c30a-d55f-4a06-8051-471c6e6c934a.png)

> - <sup>1)</sup> Azure データセンター間の通信は [マイクロソフトのグローバル ネットワーク](https://learn.microsoft.com/ja-jp/azure/networking/microsoft-global-network) 内でルーティングされるためパブリックインターネットを経由しません。
> - <sup>2)</sup> [Kudu サービスの概要](https://learn.microsoft.com/ja-jp/azure/app-service/resources-kudu)

## 受信トラフィック (クライアント → App Service)

### App Service のパブリック IP アドレスについて

App Service ではリソースごとに固有の FQDN (`<app-service-name>.azurewebsites.net`) が付与されますが、 `nslookup` 等で名前解決をしてみると一意のパブリック IP アドレスに名前解決されることが分かります。これは Azure Portal > App Service > [ネットワーク] ブレード > "受信トラフィック" 内の "受信アドレス" に表示される IP アドレスと一致します。

![image-61f6bd6b-3f84-4b43-936b-bae7e727ca95.png]({{site.baseurl}}/media/2023/07/image-61f6bd6b-3f84-4b43-936b-bae7e727ca95.png)

しかしこの IP アドレスに対してブラウザからアクセスすると HTTP 404 エラーが返却されます。これは App Service プラットフォームを構成するフロントエンドと呼ばれるロードバランサーが複数のお客様のリソースで共有しているためです。

![image-75462280-575b-498a-9504-a16c8b860140.png]({{site.baseurl}}/media/2023/07/image-75462280-575b-498a-9504-a16c8b860140.png)

クライアントから App Service への HTTP/S リクエストは、 App Service プラットフォーム内部のフロントエンドと呼ばれるロードバランサーを経由してお客様のアプリケーションが動作するインスタンスに送信されます。 App Service の受信パブリック IP アドレスはスケールユニット<sup>*3)</sup> 内で共有され、フロントエンドは HTTP/S リクエスト内の Host ヘッダーを元に送信先のアプリケーションを判断します。

![image-2ce718dc-b2ad-42a3-9d12-4bd3f91bf82b.png]({{site.baseurl}}/media/2023/07/image-2ce718dc-b2ad-42a3-9d12-4bd3f91bf82b.png)

> - <sup>3)</sup> [App Service を構成する主要な要素とそれぞれの役割](https://jpazpaas.github.io/blog/2023/05/10/Appservice-components.html)

### カスタムドメインの追加について

App Service ではお客様が所有するカスタムドメインを使用してアプリケーションにアクセスすることができますが、 DNS サーバーへのレコード追加だけではカスタムドメインを使用してアプリケーションにアクセスすることができません (HTTP 404 エラー応答が返却されます)。

これは前述しましたように、 App Service プラットフォーム内部のフロントエンドは HTTP/S リクエスト内の Host ヘッダーを元に送信先のアプリケーションを判断しているためで、フロントエンドに対してドメインとアプリケーションの関係性を登録する必要があります。

具体的には、 Azure Portal の [カスタムドメイン] ブレード > [+ カスタムドメインの追加] からカスタムドメインを登録することで、フロントエンドが当該ドメインへのリクエストをアプリケーションに送信することができるようになります (App Service ドメイン以外のドメインレジストラをご利用の場合、実際にカスタムドメインを名前解決するための DNS レコードの追加は別途必要となりますのでご留意ください)。

![image-0a337572-8ec2-4d6f-9bf0-7bbb09e6f260.png]({{site.baseurl}}/media/2023/07/image-0a337572-8ec2-4d6f-9bf0-7bbb09e6f260.png)

なお App Service にカスタムドメインを追加する際にはドメイン所有権の検証が必要となり、具体的には `asuid.<custom-domain>` に対する TXT レコードをパブリック DNS サーバーに登録する必要があります。

これは App Service をインターネットに公開せずに (プライベートエンドポイントを構成して) 使用する場合でも同様となり、パブリック DNS サーバーへの TXT レコードの追加が必要となりますのでご留意ください (内部 ASE では `.local` などのドメインを所有権の検証をスキップして登録することが可能です)。

![image-b924617e-ae47-484c-a5ee-e766dd51ed30.png]({{site.baseurl}}/media/2023/07/image-b924617e-ae47-484c-a5ee-e766dd51ed30.png)

> - [既存のカスタム DNS 名を Azure App Service にマップする](https://learn.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-custom-domain)

### プライベートエンドポイントについて

App Service ではプライベートエンドポイントを構成することで、仮想ネットワーク内のプライベート IP アドレスでアプリケーションを公開することが可能です。

これによりフロントエンドとプライベートエンドポイントがプライベートリンクで接続されます。言い換えますと、プライベートエンドポイントを構成した場合でもフロントエンドを経由しますので、プライベート IP アドレスを直接指定してアクセスすることはできず、 Host ヘッダーは必要となります。

![image-337a6a62-d385-4d25-9b43-a156b5352629.png]({{site.baseurl}}/media/2023/07/image-337a6a62-d385-4d25-9b43-a156b5352629.png)

また App Service では FTP/S を使用して App Service 上のファイルサーバーに接続することができますが、 FTP/S はプライベートエンドポイント経由で接続することは叶わず、インターネット経由で接続する必要がございますのでご留意ください。

> - [App Service アプリにプライベート エンドポイントを使用する](https://learn.microsoft.com/ja-jp/azure/app-service/networking/private-endpoint)

### アクセス制限について

App Service のアクセス制限では、指定した IP アドレス (レンジ) からのみアプリケーションへのアクセスを許可することや拒否することができます。規則には 「IPv4/IPv6 アドレス」 に加えて 「サービスエンドポイント」 や 「サービスタグ」 を使用することも可能です。なお規則に FQDN を指定することは叶いません。

またアクセス制限の規則は "メインサイト (アプリケーション)" と "高度なツールサイト (Kudu サイト)" をそれぞれ独立して管理することが可能です。例えば 「アプリケーションはインターネットに公開せずプライベートエンドポイント経由でのみ公開するが、アプリケーションのデプロイはインターネット上の特定の IP アドレスから実行したい」 というような場合にも、メインサイトはすべてのアクセスを拒否するように規則を構成し、高度なツールサイトで対象の IP アドレスからのアクセスを許可することで実現が可能です。

![image-2ea531fc-b964-406b-906d-16ea92187bde.png]({{site.baseurl}}/media/2023/07/image-2ea531fc-b964-406b-906d-16ea92187bde.png)

App Service のアクセス制限機能では、アクセス元の判断はフロントエンドで行われますので、拒否されるリクエストはお客様のアプリケーションが動作するインスタンスまで送信されません。 App Service 上に `web.config` ファイルを配置 (詳細はここでは割愛) し `ipSecurity` などを使用する場合に比べてセキュリティ面でもメリットと捉えることができます。

![image-28c91a53-3e87-42f1-b20b-dd047b92b9a7.png]({{site.baseurl}}/media/2023/07/image-28c91a53-3e87-42f1-b20b-dd047b92b9a7.png)

> - [Azure App Service のアクセス制限](https://learn.microsoft.com/ja-jp/azure/app-service/overview-access-restrictions)
> - [App Service アプリを構成する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-common)

## 送信トラフィック (App Service → エンドポイント)

### App Service の送信 IP アドレスについて

App Service 上のアプリケーションが外部のエンドポイント (Web API や Database 等) に接続するシナリオが考えられます。その際に使用される IP アドレスは前述した受信側の IP アドレスとは異なるものとなり、スケールユニットごとに割り当てられた IP アドレス (群) のいずれかが使用されます。

具体的には、 Azure Portal の [プロパティ] ブレードを開き、 "送信 IP アドレス" のうちのいずれかが使用されますので、アプリケーションから接続先となるサーバー (Database 等) でファイアウォールを構成している場合には、 "送信 IP アドレス" に記載の IP アドレス群をファイアウォールで許可する必要があります。

また App Service Plan をスケールアップ/ダウンさせると、この "送信 IP アドレス" が変更される可能性がございます<sup>*4)</sup> 。 "追加の送信 IP アドレス" はスケールアップ/ダウン時に使用される可能性のある IP アドレスを含んでおりますので、アプリケーションの運用過程でスケールアップ/ダウンの可能性がございます場合には、送信先サーバーのファイアウォールでは "追加の送信 IP アドレス" を許可していただければと存じます。

![image-82c73c32-9194-4bb4-9ca5-8f2f90fa8aea.png]({{site.baseurl}}/media/2023/07/image-82c73c32-9194-4bb4-9ca5-8f2f90fa8aea.png)

なお、これら "送信 IP アドレス" や "追加の送信 IP アドレス" はお客様のアプリケーションで専有される IP アドレスではなく、スケールユニット内の複数のお客様のアプリケーションで共有されます。専有のパブリック IP アドレスを使用して外部のエンドポイントに接続したいといった場合には、 NAT ゲートウェイとの統合<sup>*5)</sup> をご検討ください。

> - <sup>4)</sup> [送信 IP はいつ変更されるか - Azure App Service における受信 IP アドレスと送信 IP アドレス](https://learn.microsoft.com/ja-jp/azure/app-service/overview-inbound-outbound-ips#when-outbound-ips-change)
> - <sup>5)</sup> [Azure NAT Gateway の統合](https://learn.microsoft.com/ja-jp/azure/app-service/networking/nat-gateway-integration)

### App Service からインターネットへの送信 (SNAT) について

App Service からインターネット上のエンドポイントに接続する場合、スケールユニット内のロードバランサーで SNAT が行われます。

この際に使用できる SNAT ポート数はインスタンスごとに制限があり、 SNAT ポートを使い果たすとアプリケーションエラーが発生することがございます。詳細については [App Service における SNAT ポート枯渇問題とその解決方法](https://jpazpaas.github.io/blog/2021/09/29/app-service-snat.html) をご参照ください。

![image-49930902-1933-4483-a679-41f1bbfc00fa.png]({{site.baseurl}}/media/2023/07/image-49930902-1933-4483-a679-41f1bbfc00fa.png)

> - [Azure App Service での断続的な送信接続エラーのトラブルシューティング](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-intermittent-outbound-connection-errors)

### VNet 統合について

App Service 上のアプリケーションから仮想ネットワーク内のエンドポイントに接続したいといった場合、 VNet 統合を構成することでアプリケーションからのトラフィックが VNet を経由でき、 VNet 内のプライベート IP アドレスに対して接続が可能です。

VNet 統合ではサブネットを委任する必要があるため、空の専用のサブネットが必要となり、プライベートエンドポイントや他のリソースとサブネット内で共存させることは叶いません。また App Service はインスタンスごとに VNet 統合先サブネット内の IP アドレスを使用し、特にスケールアップ/ダウン、スケールアウト/インを実施する際には一時的にインスタンス数の 2 倍の IP アドレスを使用します。サブネットのサイズは作成後に変更することが叶いませんので、十分に余裕のあるアドレスレンジを持つサブネットを使用することをご検討ください (`/26` 以上のサイズが推奨されます<sup>*6)</sup> )。

![image-d1e7ed9f-f622-4498-ab43-4e411cd9fa60.png]({{site.baseurl}}/media/2023/07/image-d1e7ed9f-f622-4498-ab43-4e411cd9fa60.png)

> - <sup>6)</sup> [アプリを Azure 仮想ネットワークと統合する](https://learn.microsoft.com/ja-jp/azure/app-service/overview-vnet-integration)

<br>
<br>

---

<br>
<br>

2023 年 07 月 03 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>