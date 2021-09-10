---
title: "API Managementにおけるゾーン冗長有効化ができない"
author_name: "Koichiro Higashi"
tags:
    - API Management
---

# 質問
仮想ネットワーク上に配置したAPI Management ( APIM ) にてゾーン冗長有効化しようとしたところ以下のエラーが出力されて有効化できませんでした。

エラー ①
> API Management service deployment into a Virtual Network using Public IP Address, requires a new subnet, as we setup a parallel compute and swap it after successful setup, to avoid downtime. For more details refer to https://aka.ms/apimvnet.
Subnet \<既にAPIMが属しているSubnetリソース情報\> referenced by \<ロードバランサーリソース情報\> cannot be used because it contains external resources. The external resources in this subnet are (Microsoft.ApiManagement/service, \<APIMリソース情報\>). You need to delete these external resources before deploying into this subnet.'

また新規サブネットを作成して仮想ネットワーク構成を変更したところ以下のエラーが出力されました。

エラー ②
> API Management service deployment into a Virtual Network using Public IP Address, requires that the PublicIPAddress resource is not already assigned to another resource, as we setup a parallel compute and swap it after successful setup, to avoid downtime. For more details refer to https://aka.ms/apimvnet. Resource \<ロードバランサーリソース情報\> is referencing public IP address \<パブリックIPアドレスリソース情報\> that is already allocated to resource \<ロードバランサーリソース情報\>.

上記２点のエラー原因ならびに対処方法をご教示願います。

# 回答
## 原因
### エラー ①
仮想ネットワーク上にAPIM インスタンスが API バージョン 2020-12-01 以前を使用して生成された場合、当該 APIM が配置されるサブネットは APIM インスタンスのみが含まれる専用サブネットとなります。そのため、以下の操作を契機に APIM 内部の構成を最新の構成にしようとします。具体的にはロードバランサーを当該サブネット内に作成しようとして本エラーが発生いたします。

* ゾーン冗長の有効化
<br>https://docs.microsoft.com/ja-jp/azure/api-management/zone-redundancy#enable-zone-redundancy---portal
* 専用サブネットを変更せずにパブリック IP アドレスを指定して仮想ネットワーク構成を更新
<br>https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet#enable-vnet-connectivity-using-the-azure-portal


### エラー ②
エラー ① 発生時に指定したパブリック IP アドレスは、エラー ① 発生後最大 6 時間は解放されません。そのため新規サブネットを作成し、仮想ネットワーク構成時に当該サブネットとパブリック IP アドレスを指定すると当該パブリック IP アドレスは既に別のリソースに割り当てられていると判断してしまうために本エラーが発生いたします。


## 対処方法
1. 仮想ネットワーク内に新しいサブネットを作成します。その際に新しいサブネットのサービスエンドポイントや NSG 、 UDR を APIM が展開されている既存のサブネットと同一に設定してください。(新しいサブネットのサブネットアドレス範囲は異なるものとなりますので、前述の設定において念のためご留意ください。) また、この新しいサブネットに対して、いかなる委任も有効にしないでください（「サブネットのサービスへの委任」オプションは「なし」に設定してください ）。
サブネットの委任に関しましては以下をご参照いただけますと幸いです。<br>[サブネットの委任とは](https://docs.microsoft.com/ja-jp/azure/virtual-network/subnet-delegation-overview)
 
2. パブリック IP アドレスのリソースを新規に作成します。パブリック IP リソースを作成する際には、必ず「 DNS 名ラベル」を割り当ててください。どの DNS 名ラベルを使用するかは問題にはなりませんが、このパブリック IP アドレスリソースを API Management に割り当てる際には DNS 名ラベル を指定する必要があります。
パブリック IP アドレスに関しましては以下をご参照いただけますと幸いです。<br>[パブリック IP アドレス](https://docs.microsoft.com/ja-jp/azure/virtual-network/public-ip-addresses)
 
3. PATCH API を送信して、新しいサブネットとパブリック IP アドレスを使用するように API Management を更新します。この操作は、完了するまでに最大 1 時間かかることがあります。以下は、PATCHリクエストのサンプルとなります。
   <br><br>PATCH Request sample
   > https://management.azure.com/subscriptions/subid/resourceGroups/rg1/providers/Microsoft.ApiManagement/service/apimService1?api-version=2021-01-01-preview

   ```json
{
"properties": {
    "publisherEmail": admin@live.com,
    "publisherName": "Contoso",
    "virtualNetworkConfiguration": {
        "subnetResourceId": "/subscriptions/<サブスクリプション ID>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/apimService1v2/subnets/default" },
        "publicIpAddressId": "/subscriptions/<サブスクリプション ID>/resourceGroups/rg1/providers/Microsoft.Network/publicIPAddresses/apimService1ip",
        "virtualNetworkType": "Internal"
    }
}
   ```

   上述の API の詳細に関しましては以下 API リファレンスをご参照いただけますと幸いでございます。<br>[Api Management Service - Update](https://docs.microsoft.com/en-us/rest/api/apimanagement/2021-01-01-preview/api-management-service/update)

   また REST API ではなく Azure ポータルから APIM 内の [ 仮想ネットワーク ] ブレードから [ サブネット ] 、[ 管理パブリック IP アドレス ] にそれぞれ新規サブネットならびにパブリック IP アドレスを指定することで同様の更新を実施することもできます。

   ![2021-09-10-set-public-ip.png]({{site.baseurl}}/media/2021/09/2021-09-10-set-public-ip.png)

   Azure ポータルによる手順に関しましては以下も併せてご参考にしていただけますと幸いでございます。<br>[Azure ポータルを使用して VNET 接続を有効にする](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet#enable-vnet-connectivity-using-the-azure-portal)

   なお既存のサブネットを変更したくない場合は、仮想ネットワーク構成を一旦 [ なし ] に変更していただき、6 時間後に既存のサブネットを用いて所望の仮想ネットワーク構成 ( 外部 or 内部 ) を設定いただくようお願いいたします。

   ![2021-09-10-set-vnet.png]({{site.baseurl}}/media/2021/09/2021-09-10-set-vnet.png)

4. Azure ポータルにてゾーン冗長を有効にします。
なお本手順を実施する際には 手順 3 にて設定したパブリック IP アドレスとは別のパブリック IP アドレスを指定する必要がありますので、再度 手順 2 にてパブリック IP アドレスを作成していただき、
ゾーン冗長を有効にする際に再度 手順 2 にて作成したパブリック IP アドレスを指定いただくようお願いいたします。
<br>ゾーン冗長の有効化の手順の詳細は以下をご参照いただけますと幸いでございます。<br>[ゾーン冗長を有効にする - ポータル](https://docs.microsoft.com/ja-jp/azure/api-management/zone-redundancy#enable-zone-redundancy---portal)


# 参考ドキュメント

* [Azure API Management の可用性ゾーンのサポート](https://docs.microsoft.com/ja-jp/azure/api-management/zone-redundancy)
* [VNET接続の有効化](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet#enable-vnet-connection)

<br>
<br>

---

<br>
<br>

2021 年 9 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>