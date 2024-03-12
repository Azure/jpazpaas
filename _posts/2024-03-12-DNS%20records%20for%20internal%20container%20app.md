---
title: "内部環境で作成した Container Apps の DNS レコードの設定例"
author_name: "Takeharu Oshida"
tags:
    - Container Apps
---

# はじめに

お世話になっております。Azure Container Apps サポート担当の押田です。

この記事では VNet に閉じた内部環境で作成した Container Apps Environment にデプロイした Container Apps に対して、既定のドメインに対してVNet 内のリソースからアクセスするために必要な DNS レコードについての設定例を記載します。


# 参考資料

公式ドキュメントでは下記に記載があります。

[内部の Azure Container Apps 環境に仮想ネットワークを提供する](https://learn.microsoft.com/ja-jp/azure/container-apps/vnet-custom-internal?tabs=bash%2Cazure-cli&pivots=azure-portal#additional-resources)

> **その他のリソース**
>
> VNet スコープのイングレスを使用するには、 DNS を設定する必要があります。

[Azure Container Apps 環境でのネットワーク](https://learn.microsoft.com/ja-jp/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#dns)

> **VNet スコープのイングレス**: 内部環境で VNet スコープのイングレスを使う予定の場合は、次のいずれかの方法でドメインを構成します。
> 
> 1. **非カスタム ドメイン**: カスタム ドメインを使わない場合は、Container Apps 環境の既定のドメインを Container Apps 環境の静的 IP アドレスに解決するプライベート DNS ゾーンを作成します。 Azure プライベート DNS または独自の DNS サーバーを使用できます。 Azure プライベート DNS を使う場合は、コンテナー アプリ環境の既定のドメイン （`<UNIQUE_IDENTIFIER>.<REGION_NAME>.azurecontainerapps.io`） として名前が指定されたプライベート DNS ゾーンと、A レコードを作成します。 A レコードには、Container Apps 環境の名前 `*<DNS Suffix>` と静的 IP アドレスが含まれています。



# 設定例

## 作成リソースについて

Container Apps Environment および、Container Apps の作成手順については下記のチュートリアルに従います。

- [内部の Azure Container Apps 環境に仮想ネットワークを提供する](https://learn.microsoft.com/ja-jp/azure/container-apps/vnet-custom-internal?tabs=bash%2Cazure-cli&pivots=azure-portal#additional-resources)

本記事用に以下のリソースを作成しました。

- **Container Apps Environment**: toshida-vnet-internal-environment

  ![image-bbece5a8-19ff-42e8-be26-6d93d46da643.png]({{site.baseurl}}/media/2024/03/image-bbece5a8-19ff-42e8-be26-6d93d46da643.png)

- **Container App**: internal-app

  ![image-1c070a2b-ec49-42dd-ab4f-b962b20c7497.png]({{site.baseurl}}/media/2024/03/image-1c070a2b-ec49-42dd-ab4f-b962b20c7497.png)

Container Apps Environment は VNet `toshida-sandbox-vn` 内に作成しています。

デプロイしているイメージはクイックスタート用のコンテナ(`mcr.microsoft.com/k8se/quickstart`)となります。
Container App にはアプリケーション URLとして、`https://internal-app.bluepond-d5ce2ea5.japaneast.azurecontainerapps.io` が設定されています。
これはアプリ名と、Container Apps Environment の `<UNIQUE_IDENTIFIER>`、リージョンからなります。

![image-bd6a9d6d-8792-4c28-a094-c1fd355db2e6.png]({{site.baseurl}}/media/2024/03/image-bd6a9d6d-8792-4c28-a094-c1fd355db2e6.png)


参考:

[Azure Container Apps でアプリケーションを接続する](https://learn.microsoft.com/ja-jp/azure/container-apps/connect-apps?tabs=bash#location)

## VNet からのアクセスできるように DNS を構成する

> Azure プライベート DNS を使う場合は、コンテナー アプリ環境の既定のドメイン （`<UNIQUE_IDENTIFIER>.<REGION_NAME>.azurecontainerapps.io`） として名前が指定されたプライベート DNS ゾーンと、A レコードを作成します。 A レコードには、Container Apps 環境の名前 `*<DNS Suffix>` と静的 IP アドレスが含まれています。

上記に従って構成します。

プライベート DNS ゾーン作成時のリソース名を `<UNIQUE_IDENTIFIER>.<REGION_NAME>.azurecontainerapps.io` 、今回の例では `bluepond-d5ce2ea5.japaneast.azurecontainerapps.io` とします。

作成したプライベート DNS ゾーンリソースに対して A レコードを追加します。
今回の場合、Container App Environment には 静的 IP として `10.0.12.62` が VNet から割り当てられているため、その値を設定します。

![image-50f51ccd-28dd-4d5d-9151-d09e984549c2.png]({{site.baseurl}}/media/2024/03/image-50f51ccd-28dd-4d5d-9151-d09e984549c2.png)

併せて、この プライベート DNS ゾーン を VNet にリンクします。リンク名は任意のもので構いません。

![image-8b046c67-597a-496b-860d-2053500e1e92.png]({{site.baseurl}}/media/2024/03/image-8b046c67-597a-496b-860d-2053500e1e92.png)

参考: 

[Azure プライベート DNS ゾーンとは?](https://learn.microsoft.com/ja-jp/azure/dns/private-dns-privatednszone)

## VNet 内のリソースからアクセスしてみる

今回は同じ VNet に対して VNet 統合している App Service の Kudu コンソールをクライアントとしてみます。

下記の通り、名前解決および curl による HTTP 疎通が適切に行われていることがわかります。

![image-10dae706-4d79-4e63-9ad1-f7747c6262d2.png]({{site.baseurl}}/media/2024/03/image-10dae706-4d79-4e63-9ad1-f7747c6262d2.png)

![image-36c94e69-340e-4156-bb4a-ee2812149e39.png]({{site.baseurl}}/media/2024/03/image-36c94e69-340e-4156-bb4a-ee2812149e39.png)


2024年03月12日時点の内容となります。
本記事の内容は予告なく変更される場合がございますので予めご了承ください。