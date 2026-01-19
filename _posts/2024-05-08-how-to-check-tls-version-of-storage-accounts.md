---
title: "ストレージアカウントの最小 TLS バージョンを確認する方法について"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
[Azure Blob Storage の TLS 1.2 に移行する](https://learn.microsoft.com/ja-jp/azure/storage/common/transport-layer-security-configure-migrate-to-tls2) という資料に以下の記述がありました。

( 引用開始 )  
<br/>
**2026 年 2 月 3 日**、_Azure Blob Storage では、トランスポート層セキュリティ (TLS) のバージョン 1.0 と 1.1 のサポートが停止されます。 TLS 1.2 が新しい最小 TLS バージョンになります。 この変更は、すべてのクラウドで、TLS 1.0 と 1.1 を使用するすべての既存および新しい BLOB ストレージ アカウントに影響します。 TLS 1.2 を既に使用しているストレージ アカウントは、この変更の影響を受けません。_  
<br/>( 引用終了 )  

上記の状況もあり、自分の手元のストレージアカウントで TLS 1.0 や 1.1 が設定されているかを確認したいのですが、どの TLS バージョンを使っているか確認する方法はありますか。

# 回答

## 特定のストレージアカウントの最小 TLS バージョンの確認方法

まず、単一のストレージアカウントで確認する場合は、Azure ポータルで該当のストレージアカウントを選択して、「概要」もしくは「構成」のブレードを開けば確認できます。それぞれのスクリーンショットを載せます。

![image-99bf3e16-8393-48b3-b318-476b31ccfe0d.png]({{site.baseurl}}/media/2024/05/image-99bf3e16-8393-48b3-b318-476b31ccfe0d.png)

「構成」ブレードでは設定変更もできます。<br/>

![image-d79d8339-6703-443d-8000-1d4a53c017d1.png]({{site.baseurl}}/media/2024/05/image-d79d8339-6703-443d-8000-1d4a53c017d1.png)

## 複数のストレージアカウントの最小 TLS バージョンの確認方法

ただ、お手元に複数のストレージアカウントがある場合には一つずつ確認するのは手間になるかと思います。この場合はAzure Resource Graph、Azure CLI、Azure PowerShell などを使うといいかと思います。ここではAzure Resource Graph と Azure CLI の例を紹介します。

### Azure Resource Graph の例

Azure ポータルで Azure Resource Graph エクスプローラーを開き、以下のようなクエリを実行します。

```
Resources  
| where type =~ 'microsoft.storage/storageaccounts'
| where subscriptionId == "XXX"
| where properties.minimumTlsVersion == "TLS1_0" or properties.minimumTlsVersion == "TLS1_1" 
| project name, properties.minimumTlsVersion
```
上記の例では特定のサブスクリプションID を指定し、かつ TLS 1.0、1.1 のみを抽出しております。  
具体的なストレージアカウント名があるのでマスクしていますが、ストレージアカウント名と最小 TLS バージョンが一覧で表示されます。

![image-c8bbf004-6925-4dbe-ae2f-edee82d88849.png]({{site.baseurl}}/media/2024/05/image-c8bbf004-6925-4dbe-ae2f-edee82d88849.png)

他のプロパティも一緒に見たい、などあるようでしたら、適宜クエリを編集ください。

### Azure CLI での例
こちらは一例として、Cloud Shell 上で実行しますが、以下のようなクエリを実行します。

```
az storage account list --query "[].{name:name,minimumTlsVersion:minimumTlsVersion}" --output table
```
結果としては以下のような表示になります。具体的なストレージアカウント名があるのでマスクしていますが、ストレージアカウント名と最小 TLS バージョンを表形式で表示しています。

![image-9ddff4a6-7e09-47d6-9da6-ebda7d7263c1.png]({{site.baseurl}}/media/2024/05/image-9ddff4a6-7e09-47d6-9da6-ebda7d7263c1.png)

# よくある質問
## (Q1)  最小 TLS バージョンを 1.2 にしているのですが、TLS 1.0 を使った接続の確立まではできてしまいます。これはなぜですか？

こちらについては、2026 年 2 月 3 日以前では想定された動作となります。ストレージアカウントについてはマルチテナントの構成となっているため、ストレージアカウント単位でポートを閉じたり、通信としての特定の TLS バージョンのみを許可する、ということはできません。そのため、TLS バージョンによるアクセス許可の制御は以下の流れになります。

1. 仮にストレージアカウントの最小 TLS バージョンが 1.2、クライアントの利用する TLS バージョンが 1.0 だったとしても、TLS の接続自体は確立する
2. クライアントがストレージアカウントに設定されている最小 TLS バージョンを満たしていない場合、HTTP の応答として、400 The TLS version of the connection is not permitted on this storage account. という応答を返す

例えば、ストレージアカウントの最小 TLS バージョンが 1.2 において、TLS 1.2 で接続すれば、例えば以下のように BLOB の内容が得られるものの、クライアントが TLS 1.0 を用いた場合は HTTP 400 応答となり、BLOB の内容は取得できません。下記の例は curl をもちいて、TLS 1.2、1.0 それぞれで同じ BLOB にアクセスした時の例となります。この例では container1 の配下に hello123.txt という BLOB があり、中身は ”hello” というテキストです。一回目の curl は TLS 1.2 になるため、BLOB の中身である "hello" が取得できていますが、二回目の curl は TLS 1.0 を利用しているため、BLOB の中身は取得できず、TlsVersionNotPermitted という文言を含んだエラーの XML 応答を受け取っています。

![image-c2994107-abf8-4c24-8959-9b9bcd698922.png]({{site.baseurl}}/media/2024/05/image-c2994107-abf8-4c24-8959-9b9bcd698922.png)

## (Q2) 特定のストレージアカウントの最小 TLS バージョンが 1.0 となっていました。このストレージアカウントを利用するクライアントで TLS 1.0 や 1.1 を使っているクライアントを確認したいのですが、どうすればいいですか。

こちらについてはストレージアカウントで診断設定を有効にしていただき、個別のアクセスを記録するようにしたうえで、TlsVersion の列に TLS 1.0 や 1.1 を利用している形跡がないかをご確認いただく必要があります。実際の方法については下記資料に詳細がございますので、ご覧ください。

[クライアント アプリケーションによって使用される TLS のバージョンを検出する](https://learn.microsoft.com/ja-jp/azure/storage/common/transport-layer-security-configure-minimum-version?tabs=portal#detect-the-tls-version-used-by-client-applications)

[最小許容バージョンとして TLS 1.2 を適用する](https://learn.microsoft.com/ja-jp/azure/storage/common/transport-layer-security-configure-migrate-to-tls2?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#enforce-tls-12-as-the-minimum-allowed-version)


# まとめ

本記事ではストレージアカウントの 最小 TLS バージョンの確認手順について説明しました。ストレージアカウントの最小 TLS バージョンの確認自体はシンプルな方法でできますが、TLS 1.0、1.1 を使っているクライアントがいた場合に、そのクライアントが TLS 1.2 を利用できるのか、ということの確認はクライアント側での確認が必要となります。

クライアントの網羅はできませんが、新しいクライアントなら基本的には TLS 1.2 の利用はできると思いますが、古いクライアントになると、TLS のバージョンを上げるために様々な対応が必要になる、というシナリオも考えられます。TLS 1.0、1.1 が非推奨であることは RFC 7525 などでも触れられておりますので、もし、TLS 1.0、1.1 を利用しているクライアントがいるようでしたら、早めの TLS 1.2 対応をご検討いただくのがよいかと思います。

## 変更履歴
(2024/10/7) 質問の節で引用しているサイトに記載のあった移行の期限が当初 2024 年 11 月 1 日だったものの、2025 年 11 月 1 日に変更されたため、その部分を修正しました。

(2025/9/24) 質問の節で引用しているサイトに記載のあった移行の期限が 2025 年 11 月 1 日だったものの、2026 年 1 月 31 日に変更されたため、その部分を修正しました。

(2026/1/19) 質問の節で引用しているサイトに記載のあった移行の期限が 2026 年 1 月 31 日だったものの、2026 年 2 月 3 日に変更されたため、その部分を修正しました。

<br>
<br>

---

<br>
<br>

2025 年 09 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>