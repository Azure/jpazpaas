---
title: "Azure API Management のコンピューティング プラットフォーム バージョン"
author_name: "Hiroaki Machida"
tags:
    - API Management
---

このポストは、2021 年 10 月 12 日に投稿された [Compute Platform Versions for Azure API management service](https://techcommunity.microsoft.com/t5/azure-paas-blog/compute-platform-versions-for-azure-api-management-service/ba-p/2836971) の翻訳です。

<hr />

# 背景

APIM のユーザーのみなさまはお気づきかもしれませんが、サービス機能を強化するために API Management のコンピューティング プラットフォーム（サービスをホストする Azure コンピュート リソース）のバージョンが、例えば、いくつかの価格レベルでアップグレードされています。

この記事では、プラットフォーム バージョン stv1 から stv2 へのアップグレードの内容と、これらのコンピューティング プラットフォームの主な違いについて説明します。内容は以下の通りです。

- 利用可能なコンピューティング プラットフォーム バージョン
- 既存の APIM についてコンピューティング プラットフォーム バージョンを確認する方法
- APIM を新規作成する場合にコンピューティング プラットフォーム バージョン (stv1 または stv2) を指定する方法
- stv1 と stv2 の違い
<br />

# 利用可能なコンピューティング プラットフォーム バージョン

現在は 3 つのプラットフォーム バージョンがあります： stv1、stv2、mtv1

従量課金 レベルは [App Service](https://docs.microsoft.com/ja-jp/azure/app-service/overview) を利用するアーキテクチャであるため、プラットフォーム バージョンは mtv1 のみです。<br />
その他の価格レベル (開発者、 Basic 、 Standard 、 Premium) についは、 stv1 と stv2 のプラットフォーム バージョンがあります。 stv2 は[仮想マシン スケール セット](https://docs.microsoft.com/ja-jp/azure/virtual-machine-scale-sets/overview)が利用されており、 stv1 は [Azure Cloud Services (クラシック)](https://docs.microsoft.com/ja-jp/azure/cloud-services/cloud-services-choose-me) が利用されています。


| バージョン | 説明 | アーキテクチャ | API Management 価格レベル |
|--|--|--|--|
| stv2 | Single-tenant v2 | [仮想マシン スケール セット](https://docs.microsoft.com/ja-jp/azure/virtual-machine-scale-sets/overview) | 開発者、 Basic 、 Standard 、 Premium |
| stv1 | Single-tenant v1 | [Azure Cloud Services (クラシック)](https://docs.microsoft.com/ja-jp/azure/cloud-services/cloud-services-choose-me) | 開発者、 Basic 、 Standard 、 Premium |
| mtv1 | Multi-tenant v1 | [App Service](https://docs.microsoft.com/ja-jp/azure/app-service/overview) | 従量課金 |

参考 コンピューティング プラットフォームについて：https://docs.microsoft.com/ja-jp/azure/api-management/compute-infrastructure
<br />

# 既存の APIM についてコンピューティング プラットフォーム バージョンを確認する方法
API バージョン 2021-04-01-preview 以降の API Management インスタンスは、プラットフォーム情報を表示する読み取り専用の platformVersion プロパティがあります。

API の例：https://docs.microsoft.com/ja-jp/rest/api/apimanagement/current-ga/api-management-service/get#apimanagementservicegetservice

**PlatformVersion**

サービスを実行しているコンピューティング プラットフォームのバージョン。

| Name | Type | Description |
|--|--|--|
| mtv1 | string | マルチテナント V1 プラットフォームでサービスを実行しているプラットフォーム。 |
| stv1 | string | シングル テナント V1 プラットフォームでサービスを実行しているプラットフォーム。 |
| stv2 | string | シングル テナント V2 プラットフォームでサービスを実行しているプラットフォーム。 |
| undetermined | string | コンピューティング プラットフォームがデプロイされていないので、プラットフォームのバージョンを確認できません。 |
<br />

# APIM を新規作成する場合にコンピューティング プラットフォーム バージョン (stv1 または stv2) を指定する方法

以下のフローチャートにすべてのシナリオが記載されています。

![2022-05-16-hailey_ding_1-1634027559696-5276ee50-7924-46f6-9c9a-2991346a18c8.png]({{site.baseurl}}/media/2022/05/2022-05-16-hailey_ding_1-1634027559696-5276ee50-7924-46f6-9c9a-2991346a18c8.png)

上記のチャートに基づき：
1. 仮想ネットワークなしで APIM を新規作成した場合、 APIM は VMSS ベースの stv2 PlatformVersion になります。
2. 仮想ネットワークありで APIM を新規作成した場合、 以下の条件をすべて満たすと VMSS ベースの APIM (stv2) になります。
   - [パブリック IPv4 アドレス](https://docs.microsoft.com/ja-jp/azure/virtual-network/ip-services/public-ip-addresses#standard)または[可用性ゾーン](https://docs.microsoft.com/ja-jp/azure/api-management/zone-redundancy)を APIM に設定する。
   - APIM を Azure ポータルから新規作成する、または API バージョン 2021-04-01-preview (Azure REST API、 ARM テンプレートなどで) で新規作成する。
3. 上記の条件に当てはまらなければ、 APIM は Azure Cloud Services (クラシック) ベースの stv1 になります。
<br />

# stv1 と stv2 の違い

仮想ネットワーク上に APIM を配置する場合、 stv1 と stv2 ではいくつかの重要な違いがあります。 


| [stv2 バージョン](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#prerequisites) | [stv1 バージョン](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?tabs=stv1#prerequisites) |
|--|--|
| API Management インスタンスと同じリージョンおよびサブスクリプション内に存在する仮想ネットワークとサブネット。<br />このサブネットには他の Azure リソースが含まれている可能性がある。 | API Management インスタンスと同じリージョンおよびサブスクリプション内に存在する仮想ネットワークとサブネット。<br />サブネットは、API Management インスタンス専用である必要がある。 |
| Standard SKU [パブリック IPv4 アドレス](https://docs.microsoft.com/ja-jp/azure/virtual-network/ip-services/public-ip-addresses#standard)<br />パブリック IP アドレス リソースは、外部または内部アクセスのいずれかに対して仮想ネットワークを設定するときに必要。  | パブリック IP アドレスは不要。 |
| 以下の[サービス エンドポイント](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#troubleshooting)が必要：<br />1. Azure SQL<br />2. Azure Storage<br />3. Azure Event Hub<br />4. Azure Key Vault | 以下の[サービス エンドポイント](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv1#troubleshooting)が必要：<br />1. Azure SQL<br />2. Azure Storage<br />3. Azure Event Hub<br /> |
| Azure Key Vault にアクセスするための 送信[ポート 443](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#required-ports) が必要。  | Azure Key Vault へのアクセスは不要。 |


参考：<br />
仮想ネットワーク内の stv1 APIM：https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv1#prerequisites <br />仮想ネットワーク内の stv2 APIM：https://docs.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2#prerequisites


<br>
<br>

---

<br>
<br>

2022 年 5 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
