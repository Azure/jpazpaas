---
title: "1 つのリソースに対して、2 つのプライベートエンドポイントを作成する場合のネットワーク構成案"
author_name: "Miki Sugasawa"
tags:

---

今回の記事では、1つのリソースに対して、複数のプライベートエンドポイントを作成する場合に役立つ内容を紹介しようと思います。

プライベートエンドポイントは Azure Storage や Azure Cache for Redis などのリソースに対して、アクセス元の仮想ネットワークのプライベート IP アドレスを使用して、プライベートにアクセスする仕組みです。<br>
プライベートエンドポイントは 1 つのリソースに対して複数個、作成することが可能です。

複数の仮想ネットワークから同じリソースに対して、プライベートエンドポイントを使ってアクセスする場合には、プライベート DNS ゾーンや Azure Storage アカウントの制約により、少し工夫が必要です。<br>
今回は 2 つの仮想ネットワークから Azure Storage の Blob にプライベートエンドポイントを使ってアクセスする際の、ネットワークの構成例をご紹介します。

![main.png]({{site.baseurl}}/media/2021/11/2021-11-30-main.png)

# ポイント
- 仮想ネットワーク毎にリソースグループを分けて、構成する<br>
プライベート DNS ゾーンでは、同じホスト名のレコードを持つこともできません。(aaa.privatelink.blob.core.windows.net 10.0.0.1 と aaa.privatelink.blob.core.windows.net 10.1.1.1 を別のレコードとして同じプライベート  DNS ゾーンに登録できない。)
そのため、別のプライベート DNS ゾーンにそれぞれのレコードを登録する必要がありますが、同じリソースグループ内に同じ名前のリソースは作成できないため、仮想ネットワーク毎にリソースグループを分け、それぞれにプライベート DNS ゾーンを用意することで、この制限を回避します。
(プライベート DNS ゾーンはドメイン名をリソース名にする必要があるため、"privatelink.blob.core.windows.net" というリソース名である必要があります。)
- アクセス時の FQDN は既定値を利用する<br>
例えば Azure Storage 等の一部サービスはマルチテナントで構成されているため、Azure Storage の IP アドレスにアクセスし、かつ、Host Header に FQDN が入力されている必要があります。
また、TLS 証明書を検証する際にも、FQDN は利用されるため、アクセス時の FQDN を変更することは、お勧めいたしません。
そのため、 プライベートエンドポイントを作成した際の既定値(今回の例の場合、petestv2.blob.core.windows.net)をそのまま利用することをお勧めします。

# 補足
- 1 つのリソースグループの中で同様の動作を実現する場合、DNS サーバーやプライベート DNS サーバーをご要望に合わせて設定ください。
- 上記と同様にリソースグループを分けて構成する場合には、プライベートエンドポイントを作成・削除する際に、プライベート DNS ゾーンの A レコードも付随して自動で作成・削除されますので、込み入った追加設定もなくご利用いただけます。是非、ご活用ください。


それでは、試しに図のリソースグループ rg001 を Azure ポータルから構成してみましょう。
どちらのリソースグループでも問題ないので、事前に同じサブスクリプション内に Azure Storage アカウントを作成しておきましょう。<br>
リソースグループ rg002 も以下と同様の手順で作成してください。


1. リソースグループ rg001 を作成する<br>
![create-rg.png]({{site.baseurl}}/media/2021/11/2021-11-30-create-rg.png)
2. 仮想ネットワークを作成する<br>
![create-vnet.png]({{site.baseurl}}/media/2021/11/2021-11-30-create-vnet.png)	
3. プライベートエンドポイントを作成する<br>
![create-pe.png]({{site.baseurl}}/media/2021/11/2021-11-30-create-pe.png)

4. プライベート DNS ゾーンを確認する<br>
プライベートエンドポイントを作成した際にプライベート DNS ゾーンも作成されますが、意図したリソースグループに作成されていない等があれば、プライベート DNS ゾーンを再作成またはプライベート DNS ゾーンをリソースグループ(rg001)に移動してください。<br>
また、A レコードが設定されていること、仮想ネットワーク リンクが設定されていることも合わせて確認してください。<br>
![check-arecord.png]({{site.baseurl}}/media/2021/11/2021-11-30-check-arecord.png)
![check-vnetlink.png]({{site.baseurl}}/media/2021/11/2021-11-30-check-vnetlink.png)


基本的な準備はこれで終わりです。
後はアクセス元となるリソース (Azure VM や VMSS など) を同じ仮想ネットワーク内にデプロイしてください。
Azure VM から FQDN の名前解決をした結果は以下になり、プライベートエンドポイントを指していることがわかります。<br>
![check-nslookup.png]({{site.baseurl}}/media/2021/11/2021-11-30-check-nslookup.png)

# 参考ドキュメント
- [Azure プライベート エンドポイントとは](https://docs.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview)
- [Azure Storage のプライベート エンドポイントを使用する](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints)
- [Azure プライベート DNS とは](https://docs.microsoft.com/ja-jp/azure/dns/private-dns-overview)
<br>
<br>

---

<br>
<br>

2021 年 11 月 30 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>