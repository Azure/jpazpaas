---
title: ストレージアカウント → Event Grid → Service Bus → Function Apps の経路で可能な限りプライベートでイベントを配信する設定手順
author_name: "chansiklee"
tags:
    - Storage Account
    - Event Grid
    - Service Bus
---

# はじめに
お世話になっております。PaaS Dev サポート担当の李です。<br>
本日はストレージアカウントから Event Grid 経由で Service Bus にイベントを配信し、<br>
Service Bus に届いたイベントをトリガーに Function Apps を動作する一連の手順を、パブリックインタネットを経由せずに可能な限りプライベートアクセスで行うための設定方法をご紹介いたします。<br>
尚、Service Bus と Function Apps それぞれに Network Security Group(NSG) がついている場合どのようなポートを許可すれば良いかに関してもご紹介いたします。<br>
## 全体の構成図
![image-9c1f24ac-8cec-4caf-b3bc-ad41ca8057e1.png]({{site.baseurl}}/media/2023/04/image-9c1f24ac-8cec-4caf-b3bc-ad41ca8057e1.png)<br>
今回の構成を図で簡単に表現しますと上記の構成になります。<br>
Service Busの代わりに、お客様のご要件に合わせてEvent hubs若しくはストレージご利用頂くことも可能ですが、今回の記事ではService Busを利用した設定手順をご案内させて頂きます。<br>

## Event Grid → Function Apps に直接繋げられないか？
シンプルな構成として Storage -> Event Grid -> Functions (プライベートエンドポイント経由) とされたい場合もあるかと存じますが、<br>
大変恐れ入りますが、現時点では以下のドキュメント記載の通り、Event Grid からプライベートエンドポイント経由でのイベントの配信はサポートしておりません。<br>
<br>

>現時点では、プライベート エンドポイントを使用してイベントを配信することはできません。 つまり、配信されたイベント トラフィックがプライベート IP 空間から外に出てはならないという、ネットワークの分離の厳格な要件がある場合はサポートされません。<br>
[マネージド ID を使用したイベント配信](https://learn.microsoft.com/ja-jp/azure/event-grid/managed-service-identity#private-endpoints)<br>

したがって、Event Gridと Functions 間に Service Bus/Event Hubs/Storage を構成し、Functions がイベントをプルするという構成をする必要があるため、今回は一例として Service Busを間に挟んだ場合にどのように構成するか記載いたします。<br>

# ストレージアカウント → Event Grid
ストレージアカウントから Service Bus へイベントを配信する場合は直接配信されるのではなく、Event Grid サービスを経由して Service Bus に配信する形となります。こちらの通信は常に Microsoft バックボーンネットワークでのアクセスになっているため、既にパブリックインターネット経由しない構成となっておりお客様側で何らかの設定を加える必要はございません。<br>


# Event Grid → Service Bus（若しくはEvent Hubs、Storage）
![image-2b041c50-d659-4672-8bd2-0e99fa2674cc.png]({{site.baseurl}}/media/2023/04/image-2b041c50-d659-4672-8bd2-0e99fa2674cc.png)<br>
この手順に関しては上記の図の通り、Managed Id を利用して安全にイベントを送信することができます。今回は、Service Bus のキューに対してイベントを配信してみます。<br>

![image-c12773d3-c103-42ad-9b2e-e523ef2f27c3.png]({{site.baseurl}}/media/2023/04/image-c12773d3-c103-42ad-9b2e-e523ef2f27c3.png)<br>
一般的には上記の画像の通りイベントブレードから「＋イベントサブスクリプション」ボタンを押下し、イベントサブスクリプションを作成することでシステムトピックも追加することができますが、Managed Id をご利用頂くためにはシステムトピックを先に作成して頂く必要がございます。<br>

![image-eb83c770-642b-48b4-97e9-ff83ee494e11.png]({{site.baseurl}}/media/2023/04/image-eb83c770-642b-48b4-97e9-ff83ee494e11.png)<br>
「＋リソースの作成」からEvent Grid システムトピックを作成します。「システム割り当てIDを有効化する」は「有効」にします。<br>

![image-1c624ba5-9f88-4e20-ac56-aa6d7e288354.png]({{site.baseurl}}/media/2023/04/image-1c624ba5-9f88-4e20-ac56-aa6d7e288354.png)<br>
Identity ブレードで状態が「オン」になっているか確認し、Azure ロールの割り当てボタンを押下します。<br>

![image-154b8dec-ee0e-4256-b4d6-d64ffca94daa.png]({{site.baseurl}}/media/2023/04/image-154b8dec-ee0e-4256-b4d6-d64ffca94daa.png)<br>
ロールの割り当ての追加ボタンを押下しスコープをリソースグループ若しくはサブスクリプションにして頂き、役割には「Azure Service Bus のデータ送信者」を指定します。<br>
※ 配信不能(Dead-Lettering)を有効にした場合はストレージコンテナーへの権限も必要となるため、「ストレージ BLOB データ共同作成者」も追加してください。<br>

![image-06be48cd-5238-4681-ab05-5dbbcfe75271.png]({{site.baseurl}}/media/2023/04/image-06be48cd-5238-4681-ab05-5dbbcfe75271.png)<br>
なお、Service Bus のネットワークブレードでパブリックネットワークアクセスを全て無効にし、<br>
例外-信頼された Microsoft サービスがこのファイアウォールをバイパスすることを許可しますか？項目に関して「はい」を選択します。<br>

その後ストレージのイベントブレードから「＋イベントサブスクリプション」ボタンを押下し、イベントサブスクリプション作成画面に遷移します。<br>
![image-be9e485b-40e4-4610-a2c9-c1f665784782.png]({{site.baseurl}}/media/2023/04/image-be9e485b-40e4-4610-a2c9-c1f665784782.png)<br>
上記の画像の通りそれぞれの入力値を入れ、最後に配信用の Managed Id の種類を「システム割り当て」に指定します。<br>

![image-a1a97c80-9de5-42e1-85f0-54ecda52ca3c.png]({{site.baseurl}}/media/2023/04/image-a1a97c80-9de5-42e1-85f0-54ecda52ca3c.png)<br>
配信不能(Dead-Lettering)も有効にする場合は、同じく Managed Id の種類を「システム割り当て」に指定します。<br>
これで Managed Id を設定したシステムトピック・イベントサブスクリプションの新規作成手順は完了です。<br>

既存のイベントサブスクリプションをご利用頂く場合は以下の追加機能から設定を変更することができます。<br>
![image-15755bdf-f06f-4b35-8b0d-fef2ca9599da.png]({{site.baseurl}}/media/2023/04/image-15755bdf-f06f-4b35-8b0d-fef2ca9599da.png)<br>

上記の手順通り、システムトピックを作成・設定してイベントサブスクリプションを作成するのではなく、イベントブレードからイベントサブスクリプションとシステムトピックを一緒に作成する場合は、システムトピックのアクセス許可設定がされていないため以下の様な認証エラーが発生します。
![image-0c81601e-573e-4c60-a221-f2b4c1c13b32.png]({{site.baseurl}}/media/2023/04/image-0c81601e-573e-4c60-a221-f2b4c1c13b32.png)<br>
```
"code":"Managed Identity Authorization Error"..."subscription does not have permission to perform Microsoft.ServiceBus/namespaces/messages/send/..."<br>
```
この場合はシステムトピックの作成はできていますが、イベントサブスクリプションの作成で失敗になった状態で<br>
リソースグループに戻り、ここで作成されたシステムトピックリソースに移動して上記の手順でシステムトピックにアクセス許可を付与する段階から進めてください。<br>


詳細の内容は以下の記事をご参照ください。<br>
[マネージド ID に Event Grid の配信先へのアクセスを許可する (Managed Id 権限設定方法)](https://learn.microsoft.com/ja-jp/azure/event-grid/add-identity-roles)<br>
[マネージド ID を使用したイベント配信 (Event Grid の設定方法)](https://learn.microsoft.com/ja-jp/azure/event-grid/managed-service-identity)<br>
[プライベート リンク サービスを使用してイベントを配信する](https://learn.microsoft.com/ja-jp/azure/event-grid/consume-private-endpoints)<br>
[Service Bus - 信頼できる Microsoft サービス](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-ip-filtering#trusted-microsoft-services)<br>

# Service Bus → Function Apps
![image-6c1e53e6-2f3e-4ce6-929f-b105c36aa459.png]({{site.baseurl}}/media/2023/04/image-6c1e53e6-2f3e-4ce6-929f-b105c36aa459.png)<br>
この手順に関しては同じく上記の図の通り、Private Endpoint を利用してプライベートアクセスを実現することができます。<br>
Private Endpoint は、Service Bus の Premium SKU のみでサポートしておりますのでご了承ください。<br>
こちらの設定に関しては、Service Bus と Function Apps それぞれ既に NSG がついている状況でご案内いたします。

先ず通信経路に関して、正確には Service Bus → Functions ではなく Function Apps から VNET を経由し Service Bus の Private Endpoint を通って Service Bus にイベントを取りに行くイメージとなります。<br>
![image-bf4ef96f-7710-4b7d-9ce5-2e20bb8a562b.png]({{site.baseurl}}/media/2023/04/image-bf4ef96f-7710-4b7d-9ce5-2e20bb8a562b.png)<br>
つまり、図で表現すると上記の様な構成になります。<br>

先ず仮想ネットワークを追加します。サブネットは Service Bus 用と Function Apps 用、２個が必要となります。<br>
以下がFunction Apps 用サブネット作成の画面になります。
![image-74162b27-d7c9-4035-ab70-dd440fba84d8.png]({{site.baseurl}}/media/2023/04/image-74162b27-d7c9-4035-ab70-dd440fba84d8.png)<br>
<br>
以下が Service Bus 用サブネット作成の画面になります。
※ Service Bus の場合、画像の通りプライベートエンドポイントネットワークポリシーの設定を NSG にする必要がございます。<br>
![image-a98a7e0d-f4b3-47f0-9691-91b54930af97.png]({{site.baseurl}}/media/2023/04/image-a98a7e0d-f4b3-47f0-9691-91b54930af97.png)<br>
<br>


![image-36885c61-bdb2-4e18-8b95-01075efe65ab.png]({{site.baseurl}}/media/2023/04/image-36885c61-bdb2-4e18-8b95-01075efe65ab.png)<br>
上記のサブネット追加が終わった後、Service Bus のネットワークブレードからプライベートエンドポイントを作成します。<br>
プライベートエンドポイント作成の詳細手順は以下の記事をご参照ください。<br>
[既存の名前空間のプライベート アクセスを構成する](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/private-link-service#configure-private-access-for-an-existing-namespace)<br>

![image-5d37a408-1eb2-4a9d-93ff-3a6b18d11f19.png]({{site.baseurl}}/media/2023/04/image-5d37a408-1eb2-4a9d-93ff-3a6b18d11f19.png)<br>
上記のプライベートエンドポイント作成が終わった後、下記手順に従い Function Apps の VNET 統合を行います。<br>
![image-90267737-2a94-45b2-aaf7-9a0e3f996019.png]({{site.baseurl}}/media/2023/04/image-90267737-2a94-45b2-aaf7-9a0e3f996019.png)<br>

![image-468a8bb9-de50-46ea-a562-601e4f998ce2.png]({{site.baseurl}}/media/2023/04/image-468a8bb9-de50-46ea-a562-601e4f998ce2.png)<br>
その後上記の NSG 設定で Functions → Service Bus のアウトバウンド通信に対して関連ポートを許可する必要がございます。<br>
![image-30842ad0-bbee-48d6-a425-07ae3db407a9.png]({{site.baseurl}}/media/2023/04/image-30842ad0-bbee-48d6-a425-07ae3db407a9.png)<br>
上記の記載の通り Function Apps 側に設定されているNSGのアウトバウンド通信(送信セキュリティ規則)に AMQP の5671と5672、HTTPS の443の許可が必要です。同じく、Service Bus 側の NSG にも以下の内容通り設定してください。<br>

※ Function Apps のVNET統合されたサブネットに設定するネットワークセキュリティグループのルール
||受信規則（設定必要なし）|送信規則|
|---|---|---|
|ソース|x|Any|
|ソースポート範囲|x|＊|
|宛先|x|Service Bus プライベートエンドポイントが設定されたサブネットの CIDR|
|サービス|x|Custom|
|宛先ポート範囲|x|443,5671,5672|
|プロトコル|x|TCP|
|アクション|x|許可|
|優先度|x|適宜設定|

※ Service Bus のサブネットに設定するネットワークセキュリティグループのルール
||受信規則|送信規則（設定必要なし）|
|---|---|---|
|ソース|IP Addresses|x|
|└ ソース IP アドレス / CIDR 範囲|Functions で VNET 統合設定をしたサブネット ソース IP アドレス範囲<br>（例：10.3.2.0/24）|x|
|ソースポート範囲|*|x|
|宛先|Any|x|
|サービス|Custom|x|
|宛先ポート範囲|443,5671,5672|x|
|プロトコル|TCP|x|
|アクション|許可|x|
|優先度|適宜設定|x|

許可するポートの情報の詳細は以下の記事をご参照ください。<br>
[ファイアウォールで開く必要があるのはどのポートですか。](https://learn.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-faq#----------------------------)<br>

補足として、プライベート エンドポイントを使用して Azure Functions を Azure 仮想ネットワークに統合する手順に関しては以下のチュートリアル記事をご参照ください。<br>
[プライベート エンドポイントを使用して Azure Functions を Azure 仮想ネットワークに統合する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-vnet)

これで全ての設定手順が完了しました。<br>

![image-ca8757ae-44a7-4b9b-9fea-c21bab80338f.png]({{site.baseurl}}/media/2023/04/image-ca8757ae-44a7-4b9b-9fea-c21bab80338f.png)<br>
全てのリソースに対してパブリックアクセスを無効にしても、ストレージに BLOB が追加されると Function Apps で Service Bus キュートリガーが実行されることが確認できました。

以上、ストレージアカウント → Event Grid → Service Bus → Function Apps をプライベートでアクセスする設定手順(NSG を含め)を紹介いたしました。

<br>
<br>

---

<br>
<br>

2023 年 04 月 21 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>