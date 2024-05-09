---
title: "ストレージへのアクセスを特定の Http Header の有無で制限する(Azure Front Door 連携)"
author_name: "chansiklee"
tags:
    - Storage
---

# はじめに
お世話になっております。PaaS Dev サポート担当の李です。

Azure ストレージアカウントではネットワーク設定機能を用いてファイアウォールで特定のIPアドレスやVNETでアクセスを制限することができ、若しくはパブリックアクセスを全て無効にしてプライベート エンドポイントでのアクセス制限を構成することも可能です。<br>
・[Azure Storage ファイアウォールおよび仮想ネットワークを構成する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal)
<br>
・[Azure Storage のプライベート エンドポイントを使用する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints)
<br>

本記事では、これらのアクセス制御方法以外に、特定の Http Header を含めた(若しくは含めてない)リクエストを制限することでアクセス制御を行う方法について紹介いたします。但し現時点で Azure Storage 単体では 特定の Http Header をチェックする機能がないため、Azure Front Door を組み合わせる方法を紹介いたします。

Azure Front Door については、<br>
・[Azure Front Door とは](https://learn.microsoft.com/ja-jp/azure/frontdoor/front-door-overview)<br>
をご参照ください。

![image-80abe6ba-0847-45d5-bd9b-948062db5512.png]({{site.baseurl}}/media/2023/02/image-80abe6ba-0847-45d5-bd9b-948062db5512.png)<br>
先ず Azure Front Door 側で、ストレージに送信されたリクエストの Http Header を確認し、アクセスを制限する任務を遂行するためにはクライアントからストレージアカウントに直接アクセスできず、必ず Azure Front Door を通すように構成する必要がございます。そのための構成を設定する手順を今からご案内いたします。<br>
必要な構成は Azure Front Door のリソース作成とアクセス設定、ストレージアカウントのアクセス構成と Azure Front Door との連携となります。

# ストレージアカウントへのアクセス方法を選択 - パブリック・プライベート
Azure Front Door リソースを作成する前にストレージアカウントへのアクセス方法を決める必要がございます。アクセス方法には大きく二つがあり、パブリックネットワーク経由とプライベートリンクでのアクセスがございます。これらのアクセス方法は、Azure Front Door の SKU プランを決める基準の一つになります。<br>
Azure Front Door の SKU プランには二つがあり、スタンダードとプレミアムプランがございます。
それぞれサポートする機能の違いは以下のリンクをご覧ください<br>
![image-c1ab0c07-2a14-4f81-a854-613318b7a87b.png]({{site.baseurl}}/media/2023/02/image-c1ab0c07-2a14-4f81-a854-613318b7a87b.png)<br>
・[Azure Front Door レベル間の機能の比較](https://learn.microsoft.com/ja-jp/azure/frontdoor/standard-premium/tier-comparison#feature-comparison-between-tiers)

各 SKU プランのコストに関しては、Azure Front Door では４つの課金要素があり<br>
・基本料金<br>
・Azure Front Door → Client の通信料<br>
・Azure Front Door → Backend の通信料<br>
・Request 数<br>
により金額が計算されて課金されます。それぞれの詳細金額に関しては以下の価格ページをご参照ください。<br>
・[Azure Front Door の価格](https://azure.microsoft.com/ja-jp/pricing/details/frontdoor)


**パブリックアクセス**をご利用頂く場合は Azure Front Door のスタンダード以上のどのプランを選んで頂いてもかまいませんが、**プライベートアクセス**のためのプライベートリンクの場合は現在プレミアムプランでのみサポートしており、必ずプレミアムプランで Azure Front Door リソースを作成する必要がございます。作成後でもスタンダードからプレミアムプランにスケールアップすることはできますが、プレミアムからスタンダードプランにスケールダウンする機能は提供しておりませんのでご注意ください。<br>
それぞれのアクセス方法の設定手順と特徴は以下の内容をご確認ください。

# Azure Front Door
## リソースの作成（スタンダード・プレミアム共通）
以下の クイックスタートの手順をご参照頂き、Azure Front Door プロファイルを作成してください。<br>
・[クイックスタート: Azure Front Door プロファイルを作成する - Azure portal](https://learn.microsoft.com/ja-jp/azure/frontdoor/create-front-door-portal)<br>
レベル(SKU)項目に関しては上記案内した通り、アクセス方法に応じてお選びください。<br>
![image-be728ede-bdb6-45ee-9efb-f5ef70ff77b6.png]({{site.baseurl}}/media/2023/02/image-be728ede-bdb6-45ee-9efb-f5ef70ff77b6.png)<br>
エンドポイントの設定にある配信元の種類では、ストレージ（Azure BLOB）をお選び頂き、配信元のホスト名では対象のストレージアカウントをお選びください。プライベート エンドポイント設定はリソースの作成後でも可能です。<br>
WAF ポリシーは今回 Http Header を制限するルールを設定する項目となり、新規作成で任意の名前を入力してください。<br>
![image-f2aca085-93a8-4b14-a60d-d0ce2f07cd1e.png]({{site.baseurl}}/media/2023/02/image-f2aca085-93a8-4b14-a60d-d0ce2f07cd1e.png)<br>
上記のように入力が終わったら、確認および作成ボタンを押下し、Azure Front Door リソースを作成してください。その他設定方法などのご不明点は、クイックスタートの手順をご参照ください。

## 詳細設定（一部プレミアムの追加手順あり）
![image-74cce91e-819d-4ae0-8aec-1ae397766656.png]({{site.baseurl}}/media/2023/02/image-74cce91e-819d-4ae0-8aec-1ae397766656.png)<br>
Azure Front Door プロファイルリソースの作成に成功した後、フロントドア マネージャーブレードを押下すると Azure Front Door の FQDN などの情報確認や、右側のセキュリティポリシーから WAP ページに移動することができます。

![image-c9164552-16c6-4408-8cfd-8d63c19f6d67.png]({{site.baseurl}}/media/2023/02/image-c9164552-16c6-4408-8cfd-8d63c19f6d67.png)<br>
初めてリソースを構築した時点では上記の通りポリシーモードが検出モードになっているため、アクセスを制限するために「防止モードに切り替える」必要がございます。

![image-e7dc316f-4695-43d1-82a1-cf771c5d3311.png]({{site.baseurl}}/media/2023/02/image-e7dc316f-4695-43d1-82a1-cf771c5d3311.png)<br>
その後、画像の通りカスタム ルールブレードから新規のカスタム ルールを追加し、適宜条件を付けて追加ボタンを押下してください。設定値と一致・不一致する場合や、演算子などを用いてトラフィックを許可・拒否することができますので適宜ご活用ください。<br>
例えば上記画像通りの設定の場合は、リクエストに<br>
①testheaderというヘッダーがそもそも含まれていない、若しくは<br>
②testheaderというヘッダーは含まれているが、値がtestではない場合、リクエストが拒否されます。

![image-39562473-cb3d-4e35-b63f-972df2371cea.png]({{site.baseurl}}/media/2023/02/image-39562473-cb3d-4e35-b63f-972df2371cea.png)<br>
カスタム ルールを追加した後、必ず保存ボタンを押下してください。

※ 以下の手順はプレミアムプランでプライベート エンドポイント設定をする場合のみ必要となる手順となり、スタンダードプランでパブリックアクセスをする場合は以下の手順を省略してください。
![image-bbd71b29-bd0f-4577-97a8-346bcb30371d.png]({{site.baseurl}}/media/2023/02/image-bbd71b29-bd0f-4577-97a8-346bcb30371d.png)<br>
リソース作成の段階で PEP の設定をしてない場合、上記の画像の通り Azure Front Door プロファイルの配信元グループブレード→配信元グループを選択して頂き、ホスト名を選択すると<br>
![image-70f79fa5-a750-4fa4-8230-532c17fc31d9.png]({{site.baseurl}}/media/2023/02/image-70f79fa5-a750-4fa4-8230-532c17fc31d9.png)<br>
プライベート エンドポイント設定を有効にすることができます。
これで Azure Front Door 側の設定は完了となります。

# Azure ストレージアカウント
## プライベートアクセス(プレミアムプラン必須。お勧め)
Azure Front Door からストレージのプライベート エンドポイント経由で通信する構成となります。<br>
![image-cf554150-824e-4e68-8567-89488d66fee1.png]({{site.baseurl}}/media/2023/02/image-cf554150-824e-4e68-8567-89488d66fee1.png)<br>
ストレージ – ネットワーク – ファイアウォールと仮想ネットワークタブで**パブリックアクセスを無効**にします。<br>
![image-fa73d706-f987-4f56-92e5-d1e0f2e649e7.png]({{site.baseurl}}/media/2023/02/image-fa73d706-f987-4f56-92e5-d1e0f2e649e7.png)<br>
前章の Azure Front Door 側の設定が完了できるまで暫くお待ち頂きますと、ストレージアカウントのプライベート エンドポイント接続タブに新しいプライベート エンドポイントが追加され、チェックボックスにチェックを入れて承認すると全ての手順は完了となります。<br>
設定が反映されるまで数分以上時間が掛かる場合があり、しばらくお待ちください。詳細の手順は以下の記事をご参照ください。<br>
・
[Azure Front Door Premium で Private Link を使用して配信元を保護する | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/frontdoor/private-link)<br>
これでプライベートアクセスの設定は完了です。プライベートリンクでのアクセスする場合セキュアなプライベートアクセスが可能ですが、必ず Azure Front Door のプレミアム SKU をご利用頂く必要があるため、コストの面をご検討頂く必要がございます。

## パブリックアクセス(スタンダードプラン以上)
ストレージアカウントのファイアウォール (FW) で Azure Front Door からのアクセスのみを許可する形で構成することで、ストレージで Azure Front Door からの通信のみを受け入れるように構成することができます。但し、いくつかの注意事項があり、必ず全ての内容をご確認頂き、使用をご検討ください。<br>
① ストレージアカウントの FW で Azure Front Door を許可するためには、以下のサービスタグ リストにある Azure Front Door の全ての IP を FW に登録する必要があり、
それに加えて毎週の更新により、非定期的に新しい IP アドレスが追加された場合その IP を追加する必要が生じます。<br>
・[Download Azure IP Ranges and Service Tags – Public Cloud from Official Microsoft Download Center](https://www.microsoft.com/en-us/download/details.aspx?id=56519)<br>
※ REST API では次の API を実施することで同等の情報を取得することが可能です。<br>
・[Service Tags - List - REST API (Azure Virtual Networks) | Microsoft Learn](https://learn.microsoft.com/ja-jp/rest/api/virtualnetwork/service-tags/list?tabs=HTTP)<br>
![image-9e8683a8-fd1f-47a4-b970-d2c21f2b87c6.png]({{site.baseurl}}/media/2023/02/image-9e8683a8-fd1f-47a4-b970-d2c21f2b87c6.png)<br>
![image-a7d71ad4-1ef0-442d-bcf5-51fad041d0e1.png]({{site.baseurl}}/media/2023/02/image-a7d71ad4-1ef0-442d-bcf5-51fad041d0e1.png)<br>
そのため、外部サイトとなりますがアプローチとして以下のページの様に
Service Tag の REST API を用いて Bash や Azure Functions など定期実行可能なプラットフォームにて IP アドレスの変更を確認し、
ストレージアカウントの FW の構成更新を行うよう、実装を行う必要がございます。<br>
・[Using the Azure IP Service Tag API - gordon.byers.me](https://gordon.byers.me/azure/using-the-azure-ip-service-tag-api)<br>
※ 新しい IP アドレスはまずこの JSON 上に表示されます。この表示は、実際に Azure プラットフォームで利用する少なくとも１週間以上前には追加されますので、毎週一度のチェックが必要となります。<br>
------------------------ ダウンロードページの Details から引用 --------------------------------<br>
This file is updated weekly. New ranges appearing in the file will not be used in Azure for at least one week. Please download the new json file every week and perform the necessary changes at your site to correctly identify services running in Azure. These service tags can also be used to simplify the Network Security Group rules for your Azure deployments though some service tags might not be available in all clouds and regions.<br>
------------------------ 引用 ここまで  --------------------------------<br>

② パブリックアクセスは Azure Front Door の SKU がスタンダードプランの場合でもご利用できると申し上げましたが、
スタンダードプランの場合**マルチテナント構成**になっているため、他のユーザの Front Door 経由でもアクセスができてしまう可能性が存在します。
もちろん本記事のシナリオの場合は Front Door からバックエンドにリクエストを送信する際、皆様の Front Door にしか付与されないヘッダがあるので、
それを使えば他のユーザの Front Door からのアクセスはほぼブロックできるかと存じますが、皆様の固有のヘッダの中身を知られてしまうと、
第三者によるアクセスができてしまうリスクは存在します。そのため、ストレージアカウント側の認証機能である、SASやAD認証などの仕組みを
併用いただき、よりセキュアな構成を維持いただくことをお勧めいたします。

上記のような制約に基づき、セキュリティの面でも設定の手間の面でもプライベートリンクでのプライベートアクセスを推奨しておりますが、コストの面など考慮するポイントもあり、皆様のご要件に応じて適切なアクセス方法をお選び頂けますと幸いでございます。

上記のいずれの手順が全て完了した後、Azure Front Door の FQDN / {BlobPath}でアクセスして確認すると、それぞれ Header の有無によってBLOBコンテンツが表示・またはブロックされることが確認できました。<br>
※ Azure Front Door の FQDN はフロントドア マネージャーブレードからご確認できます。）<br>
![image-fa02bfb4-3f67-4e37-b05c-9b3d08891132.png]({{site.baseurl}}/media/2023/02/image-fa02bfb4-3f67-4e37-b05c-9b3d08891132.png)<br>
↑Header ありの場合、BLOB が表示

![image-9b80ea50-5d8b-4d60-935d-db4c3af11191.png]({{site.baseurl}}/media/2023/02/image-9b80ea50-5d8b-4d60-935d-db4c3af11191.png)<br>
↑Header 無しの場合、リクエストがブロック

# 既存のスタンダードプランの Azure Front Door をプレミアムにスケールアップ<br>
![image-e6f4e8e2-2961-4ca2-abf7-3a56ed949ff1.png]({{site.baseurl}}/media/2023/02/image-e6f4e8e2-2961-4ca2-abf7-3a56ed949ff1.png)<br>
Front Door プロファイルの構成ブレードで既存の Azure Front Door プロファイルのSKUをスタンダードからプレミアムプランにスケールアップすることができます。現時点でプレミアムからスタンダードプランにスケールダウンする機能は提供しておりませんのでご注意ください。

以上、Azure Front Door を用いて特定の Http Header を含めた(若しくは含めてない)リクエストを制限することでアクセス制御を行う方法についてパブリック・プライベートアクセス両方の観点で紹介いたしました。

<br>
<br>

---

<br>
<br>

2023 年 02 月 17 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
