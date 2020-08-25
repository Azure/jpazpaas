---
title: "Notification Hubs の IP アドレスについて"
author_name: "Yo Motoya"
tags:
    - Notification Hubs
---

# 質問

アクセスを制限している環境にて、Notification Hubs へのアクセスを許可したい。<br>
許可が必要な IP アドレスを教えてほしい。

# 回答

Notification Hubs の IP アドレスは、[Service Bus 同様](https://docs.microsoft.com/ja-jp/azure/service-bus-messaging/service-bus-faq#what-ip-addresses-do-i-need-to-add-to-allow-list)、nslookup コマンドでご確認ください。以下のように `nslookup <YourNamespaceName>.servicebus.windows.net` を実行し、「権限のない回答」で返却された IP アドレスをご利用いただけます。

![nhub-ip-nslookup.png]({{site.baseurl}}/media/2020/08/2020-08-26-nhub-ip-nslookup.png)

上記手順で取得した IP アドレスは静的に構成されています。そのため、基本的には IP アドレスは変更は行われません。しかしながら、プラットフォームで重大な障害が発生し、対象のネームスペースを他のクラスタに復元する必要が発生した場合などには、変更されます。

なお、Azure では、仮想ネットワークでご利用いただける [サービスタグ](https://docs.microsoft.com/ja-jp/azure/virtual-network/service-tags-overview) を提供しております。サービスタグをご利用いただきますと、ネットワークセキュリティグループや Azure Firewall でネットワークのアクセス制御を定義することが可能です。しかしながら、Notification Hubs のサービスタグは提供されていません。そのため、アクセス制御をご検討いただく際には、前述の方法で取得した IP アドレスを利用した方法についてご検討ください。

<br>
<br>

---

<br>
<br>

2020 年 8 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>