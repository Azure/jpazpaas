---
title: "内部VNETモードAPIMでセルフホステッドゲートウェイを使う"
author_name: "Taiyo Kato"
tags:
- API Management
---

Azure 上の API Management ゲートウェイの他に、自社環境にもゲートウェイをホストしたいシナリオがあります。

  

API Management（ 以下 APIM ）はデフォルトで VNET に入っておらず、特にネットワーク制限をかけてない為セルフホステッドゲートウェイ ( 以下 SHGW ) を Out of the box で使えます。しかしながら APIM を内部 VNET モードで使用している場合は各種ネットワーク制限がかかり、NSG で通信を開けたりする必要があります。

  

本記事では内部 VNET モードの APIM に SHGW を接続する際の注意点を解説していきます。

※ SHGW とは何かについては解説しません。

  

# 質問


- 内部 VNET モードの APIM に SHGW を追加したい

- 内部 VNET モードの APIM に SHGW が繋がらない

  

# 解説

まず APIM が内部 VNET モードで必要な設定を見ていきます。

  

## APIMの構成・仕様

外部・内部 VNET に関係なく APIM を VNET 統合する場合は必ず適切に穴あけを行った NSG をサブネットに関連付ける必要があります。

ドキュメントより SHGW に関連する２つの受信規則を抜粋しました。

  

| Rule | ソース / ターゲット ポート | Direction | トランスポート プロトコル | サービス タグ ソース / ターゲット |                                         目的                                         | VNet の種類 | メモ |
| ---- | :------------------------- | :-------- | :------------------------ | :-------------------------------- | :----------------------------------------------------------------------------------- | :---------- | ---- |
| 1    | * / [80],443               | 受信      | TCP                       | Internet/VirtualNetwork           | 外部からHTTP・HTTPS通信入れる。ゲートウェイでHTTPSのみにしている場合は80ポートは不要 | 外部のみ    |      |
| 2    | * / 3443                   | 受信      | TCP                       | ApiManagement/VirtualNetwork      | Azure Portal と PowerShell 用の管理エンドポイント                                    | 外部 / 内部 |      |

[NSG規則の構成 - Azure API Management を使用して仮想ネットワークに接続する | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#configure-nsg-rules)

### Rule 1 - 80 / 443 ポート

シンプルにゲートウェイコンポーネントへアクセスを許可する為に外部 VNET モードにのみ必要です。

内部 VNET モードでは APIM 自体が 80 / 443 を外部向けに通信しない為 NSG で許可してもアクセスできません。

### Rule 2 - 3443 ポート

APIM の各種設定の操作を行うには管理エンドポイントに対してリクエストを送信します。

管理エンドポイントは 443、3443 ポートでアクセス可能ですが、VNET 統合している場合は APIM 自体が 443 ポートを非公開にしている為アクセスできません。3443 ポートは常時アクセス可能です。


なおドキュメントではInbound 3443 ApiManagement / VirtualNetwork と書かれていますが、これは Portal・Powershell・REST API で操作できるようにする為ソースが ApiManagement となってます。
（この ApiManagement サービスタグは APIM リソースの IP レンジのタグではなく、APIM リソースプロバイダーの送信用サービスタグです）

ユーザーが送信したリクエストは Azure Resource Manager ( ARM ) を経由して APIM リソースプロバイダーへ転送されます。そして APIM リソースプロバイダーが更に管理エンドポイントへ転送する流れとなっています。
Portal 上で 3443 ポートを許可していない場合に、管理エンドポイントに接続できない旨のエラーが出る理由がこれです。

![]({{site.baseurl}}/media/2022/07/2022-07-01-missing-3443port.png)


リクエストの流れを図にすると以下のようになります。

![]({{site.baseurl}}/media/2022/07/2022-07-01-managementendpoint-flow.png)

内部 VNET モードで管理エンドポイントへ接続にはこの仕組みをうまく利用することでアクセスできるようになります。  

## SHGW の準備

  

SHGW を作成・デプロイするときにまず Portal にて SHGW として受け入れるゲートウェイを作成します。

 

![]({{site.baseurl}}/media/2022/07/2022-07-01-portal-gateway-1.png)
![]({{site.baseurl}}/media/2022/07/2022-07-01-portal-gateway-2.png)



SHGW にはそれぞれ v0 & v1 と v2 の２つのバージョングループがあります。

Portal 上でバージョンのタブからそれぞれのグループで使える管理エンドポイントURLが表示されます。


| バージョングループ |                                           接続エンドポイントURL                                            |        名称        |
| ------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------ |
| SHGW v0 & v1       | config.service.endpoint=https://{servicename}.management.azure-api.net/{resource id}?api-version={version} | 管理エンドポイント |
| SHGW v2            | config.service.endpoint={servicename}.configuration.azure-api.net                                          | 構成エンドポイント |



それぞれのURLを見て分かるように、ポート指定が無いのでセルフホステッドゲートウェイ立ち上げ時に HTTPS の 443 へアクセスを試みます。

ここで上述の 443 ポート非公開問題が発生します。

![]({{site.baseurl}}/media/2022/07/2022-07-01-log-connrefused-443.png)

解決方法は単純で、上記 Portal が 3443 ポートで管理エンドポイントにアクセスできるようになったのと同じく、443 ポートの代わりに FQDN の後ろに3443 ポートを指定することでアクセスできるようになります。
この方法の場合は上記 NSG の 3443 ポートに関する受信規則を修正若しくは別途 SHGW からのアクセスを許可する必要があります。
以下のいずれを実施してください。

### NSG の変更 ( 赤字部分 )

 | ソース / ターゲット ポート | Direction | トランスポート プロトコル | サービス タグ ソース / ターゲット |                              目的                               | VNet の種類 |
 | ------------------------- | -------- | ------------------------ | -------------------------------- | -------------------------------------------------------------- | ---------- |
 | * / 3443                   | 受信      | TCP                       | <span style="color: red; ">**Any**</span>/VirtualNetwork                | Azure Portal と PowerShell とSHGWアクセス用の管理エンドポイント | 外部 / 内部 |

### NSG の追加

 | ソース / ターゲット ポート | Direction | トランスポート プロトコル | サービス タグ ソース / ターゲット |                      目的                      | VNet の種類 |
 | ------------------------- | -------- | ------------------------ | -------------------------------- | --------------------------------------------- | ---------- |
 | * / 3443                   | 受信      | TCP                       | Internet/VirtualNetwork           | SHGW 用の管理エンドポイント (外部ホストの場合) | 外部 / 内部 |

![]({{site.baseurl}}/media/2022/07/2022-07-01-log-connsuccess3443.png)


注意点として、SHGW v2 の構成エンドポイントは役割として管理エンドポイントと同じですが、3443 ポートを公開していません。加えて内部 VNET モードですと上記で述べた通り 443　ポートは非公開となります。 
しかしながら SHGW v2 は構成エンドポイントのフォールバックとして管理エンドポイントにも接続できます。

## 結論

結果として、内部 VNET モードでのアクセス可否は以下の図の通りになります。

|             接続可否             | SHGW v0 & v1 | SHGW v2 |
| -------------------------------- | ------- | ------- |
| management.azure-api.net:443     | ×       | ×       |
| management.azure-api.net:3443    | 〇      | 〇      |
| configuration.azure-api.net:443  | ×       | ×       |
| configuration.azure-api.net:3443 | ×       | ×       |



以上お試しください。


# 参考ドキュメント

  

[Azure API Management を使用して内部モデルで仮想ネットワークに接続する](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-internal-vnet?tabs=stv2)

  
  

<br>

<br>

  

---

<br>

<br>

2022 年 7 月 1 日時点の内容となります。<br>

本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>

<br>
