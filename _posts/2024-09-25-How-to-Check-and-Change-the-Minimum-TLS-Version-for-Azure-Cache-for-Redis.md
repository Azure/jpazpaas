---
title: "Azure Cache for Redis の最小 TLS バージョンの確認、変更方法"
author_name: "yukiarai"
tags:
    - Azure Cache for Redis
---

---
# はじめに
トランスポート層セキュリティ (TLS) バージョン 1.2 以降の排他的使用に向けた業界全体の推進活動に対応するために、Azure Cache for Redis は、2025 年 3 月に TLS 1.2 の使用を求める方向に進んでいます。  
TLS バージョン 1.0 と 1.1 は、BEAST や POODLE などの攻撃を受けやすく、またその他の共通脆弱性識別子 (CVE) の弱点を持つことが知られています。  
そのため、**2025 年 3 月 1 日以降、Azure Cache for Redis の 最小 TLS バージョンの設定は自動的に 1.2となり、TLS 1.0 と 1.1 を使用した通信は接続できなくなります。**

■ ご参考：  
- [Azure Cache for Redis での使用から TLS 1.0 と 1.1 を削除する - Azure Cache for Redis](https://learn.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-remove-tls-10-11) 
- [(英語版)Remove TLS 1.0 and 1.1 from use with Azure Cache for Redis - Azure Cache for Redis | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-remove-tls-10-11)

上記ドキュメントより表を引用  
![image-49ac5f86-e95b-48d7-bb39-f3229f575be2.png]({{site.baseurl}}/media/2024/09/image-49ac5f86-e95b-48d7-bb39-f3229f575be2.png)

本記事では、Azure Cache for Redis のキャッシュ インスタンスの最小 TLS バージョンの確認方法、及び TLS 1.0 と 1.1 を設定していた場合に、TLS 1.2 への変更の手順をご案内いたします。  
<br>

---

(目次)   
[0. 注意事項](#index0)   
[1. 最小 TLS バージョン設定の確認と変更手順 (Azure ポータル編)](#index1)  
[2. 最小 TLS バージョン設定の確認と変更手順 (コマンド使用編)](#index2)   
[3. よくいただくご質問](#index3)  
<br>

---
<a id="index0"></a>
# 0. 注意事項
## ※※ ご注意ください ※※  
冒頭でご紹介のドキュメントのタイムラインに記載の通り、2024 年 3 月 1 日以降は既存のキャッシュで minimumTLSVersion を 1.0 または 1.1 に変更することができなくなっております。そのため、**現在の最小 TLS バージョンの設定が TLS 1.0、1.1 になっている状態から一度でも TLS 1.2 に変更しますと、再度 TLS 1.0、1.1 に戻すことはできません。**  
現時点でリソースの最小 TLS バージョンが TLS 1.0、1.1 に設定されている場合、接続するクライアントが TLS 1.2 未満を利用していないことをご確認いただいた後、TLS 1.2 への設定変更を行ってください。
  
クライアント側にて接続に使用している TLS バージョンの確認については、本記事内「3. よくいただくご質問」項目の「[質問4. クライアント側から通信に使用している TLS バージョンの確認方法](#faq4)」をご参照ください。

また、最小 TLS バージョンが TLS 1.2 に設定されている既存の Redis キャッシュ インスタンスに対しても、TLS 1.0、1.1 に設定変更することはできません。    
<br>

---
<a id="index1"></a>
# 1. 最小 TLS バージョン設定の確認と変更手順 (Azure ポータル編)
最小 TLS バージョンの設定は Azure ポータルで該当の Redis キャッシュ インスタンスを選択後、[詳細設定] ブレードを表示いただき、[TLS の最小バージョン] 欄より確認・設定変更可能です。

## TLS の最小バージョンの確認
[TLS の最小バージョン] 欄に表示されている値が現在の設定値です。

![image-963d71c4-6a0f-47a6-99c9-b70adc4e0a3b.png]({{site.baseurl}}/media/2024/09/image-963d71c4-6a0f-47a6-99c9-b70adc4e0a3b.png)

## TLS の最小バージョンの変更
[TLS の最小バージョン]欄にて「1.2」をご選択いただき、ブレード上部の「保存」をクリックしてください。  
※ **TLS 1.2 から TLS 1.0、1.1 への変更はできません。** 本記事内「[0. 注意事項](#index0)」をご参照ください。

![image-0d542c20-2240-43ee-9bf6-d7912532167b.png]({{site.baseurl}}/media/2024/09/image-0d542c20-2240-43ee-9bf6-d7912532167b.png)

<a id="default"></a>
## TLS 最小バージョンの表示が [既定] と表示されている場合
以下のご参考画像のように、[TLS の最小バージョン]の表示が [既定] と表示されている場合、当該リソースが作成された時期により既定の最小 TLS バージョンが異なります。
  
(ご参考画像)  
![image-f130514d-ce51-450a-9ac7-8ded82a1f599.png]({{site.baseurl}}/media/2024/09/image-f130514d-ce51-450a-9ac7-8ded82a1f599.png)

- Redis キャッシュ インスタンスを 2020 年以前に作成している場合は、最小 TLS バージョンが 1.0 として動作しております。
- Redis キャッシュ インスタンスを 2020 年以降に作成している場合は、最小 TLS バージョンが 1.2 として動作しております。
 
もし、対象の Redis キャッシュ インスタンスの作成年が不明の場合、対象のサブスクリプション ID とリソース名をいただきましたら、サポート チームより Azure 基盤側の情報から作成年を確認することが可能でございますので、ご希望がございましたら対象のサブスクリプションとリソースを選択してお問合せください。

■ ご参考：[お問い合わせ発行時と「異なる」サブスクリプションならびに AAD テナントの調査依頼に対してのセキュリティ チェックが強化されます](https://jpaztech.github.io/blog/information/Different-subscriptions-research/)  
<br> 
 
なお、[TLS の最小バージョン] の値を [既定] から 1.2 への変更は可能です。クライアント側の通信が TLS 1.2 以上で行われていることをご確認いただけましたら、Redis キャッシュ インスタンス の [TLS の最小バージョン] の値を 1.2 へご変更ください。
<br>

---
<a id="index2"></a>
# 2. 最小 TLS バージョン設定の確認と変更手順 (コマンド使用編)

## コマンドで最小 TLS バージョンを確認する
Azure CLI 及び Azure PowerShell を使用して Redis キャッシュ インスタンスの TLS 最小バージョンを確認する際は、以下のコマンド例をご利用いただき、「minimumTlsVersion」の値をご確認ください。

<span style="color: red;">※以下のコマンドで最小 TLS バージョンご確認の際、「minimumTlsVersion」の値が「null」または空白の場合は、値に「既定」が設定されています。対応など詳しくは、本記事内「[TLS 最小バージョンの表示が [既定] と表示されている場合](#default)」をご参照ください。</span>   
<br>

### ◆ Azure CLI
Azure にログインします。

```
az login
```

以下のコマンド例にて、指定したサブスクリプション内で、TLS 1.2 **以外**のバージョンを設定している Redis キャッシュ インスタンスをリストアップし、それぞれの名前と最小 TLS バージョンを表示します。  
※ <subscriptionId>(サブスクリプションID) はご利用環境に応じてご変更ください。
```
az redis list --subscription <subscriptionId> --query "[?minimumTlsVersion!='1.2'].{name:name, minimumTlsVersion:minimumTlsVersion}"
```

■ ご参考：[az redis list](https://learn.microsoft.com/ja-JP/cli/azure/redis?view=azure-cli-latest#az-redis-list)  
<br>

### ◆ Azure PowerShell
サブスクリプションへ接続します。  
※ <subscriptionId>(サブスクリプションID) はご利用環境に応じてご変更ください。

```
Connect-AzAccount -Subscription <SubscriptionId>
```
以下のコマンド例にて、現在のサブスクリプション内で TLS 1.2 **以外**のバージョンを設定している Redis キャッシュ インスタンスをリストアップし、それぞれの名前と最小 TLS バージョンを表示します。

```
Get-AzRedisCache | Where-Object { $_.MinimumTlsVersion -ne '1.2' } | Select-Object Name, MinimumTlsVersion
```

■ ご参考：[Get-AzRedisCache (Az.RedisCache)](https://learn.microsoft.com/ja-jp/powershell/module/az.rediscache/get-azrediscache?view=azps-12.2.0)  
<br>

## コマンドで最小 TLS バージョンを変更する
Azure CLI 及び Azure PowerShell を使用して Redis キャッシュ インスタンスの TLS 最小バージョンを変更するには、以下のコマンド例をご利用ください。  

※ <RedisCacheName>(Redis キャッシュ インスタンスの名前)、<ResourceGroupName>(リソースグループ名)、<subscriptionId>(サブスクリプションID) はご利用環境に応じてご変更ください。  
 
※ **TLS 1.2 から TLS 1.0、1.1 への変更はできません。** 本記事内「[0. 注意事項](#index0)」をご参照ください。

### ◆ Azure CLI
Azure にログインします。

```
az login
```
以下のコマンド例にて、Redis キャッシュ インスタンスの TLS 最小バージョンを 1.2 に変更します。
```
az redis update --name <RedisCacheName> --resource-group <ResourceGroupName> --set minimumTlsVersion=1.2 
```   
 
■ ご参考 : [az redis update](https://learn.microsoft.com/ja-jp/cli/azure/redis?view=azure-cli-latest#az-redis-update)  
<br>

### ◆ Azure PowerShell
サブスクリプションへ接続します。  

```
Connect-AzAccount -Subscription <SubscriptionId>
```
以下のコマンド例にて、Redis キャッシュ インスタンスの TLS 最小バージョンを 1.2 に変更します。
```
Set-AzRedisCache -ResourceGroupName <ResourceGroupName> -Name <RedisCacheName> -MinimumTlsVersion 1.2    
```

■ ご参考 : [Set-AzRedisCache (Az.RedisCache)](https://learn.microsoft.com/ja-jp/powershell/module/az.rediscache/set-azrediscache?view=azps-12.3.0&viewFallbackFrom=azps-11.0.0)  
<br>

---
<a id="index3"></a>
# 3. よくいただくご質問
- [質問1. 最小 TLS バージョン設定が既に 1.2 だった場合](#faq1)  
- [質問2. 最小 TLS バージョン設定で 1.2 以外のものをまとめて確認したい](#faq2)  
- [質問3. 最小 TLS バージョンを 1.2 に変更した際の影響](#faq3)  
- [質問4. クライアント側から通信に使用している TLS バージョンの確認方法](#faq4)

<a id="faq1"></a> 
## [質問1]: 使用している Redis キャッシュ インスタンスの最小 TLS バージョンの設定を確認したところ、**最小 TLS バージョンが 1.2** となっていました。クライアント側からの接続も問題なく行えています。本件について何か対応は必要でしょうか。 
  
Redis キャッシュ インスタンスの最小 TLS バージョンの設定が 1.2 以上であれば、1.2 未満での通信についてはすでに Azure Cache for Redis側にてアクセスを拒否している状態です。現在クライアント側から該当の Redis キャッシュ インスタンスへの接続が問題なくご利用いただけている場合は、本件におけるこれ以上のご対応は不要となります。  
<br>

<a id="faq2"></a>
## [質問2]: Azure Cache for Redis において複数の Redis キャッシュ インスタンスを使用しています。最小 TLS バージョンの設定を一つずつ確認するのが大変なため、TLS 1.2 以外が設定されているものがないかをもっと簡単に確認する方法はありますか。

あるサブスクリプション内にてご利用の全ての Redis キャッシュ インスタンスについて、最小 TLS バージョンの設定に 1.2 以外が設定されているものがないかのご確認は、本記事内「[2. 最小 TLS バージョン設定の確認と変更手順 (コマンド使用編)](#index2)」にてご案内しております、**Azure CLI または Azure PowerShell を使った確認方法** (サブスクリプション内で TLS 1.2 以外のバージョンを設定している Redis キャッシュ インスタンスをリストアップし、それぞれの名前と最小 TLS バージョンを表示します) をご活用ください。  
<br>

<a id="faq3"></a>
## [質問3]: 最小 TLS バージョンを 1.2 に変更した際の影響について知りたいです。
 
Redis キャッシュ インスタンスの TLS のバージョンを変更する場合、通常のメンテナンスと同様にフェールオーバーが発生します。  
フェールオーバーについては通常の Azure Cache for Redis のご利用の中でも発生する動作となりますので、普段の運用で特に問題がないようでしたら、影響はないと言えます。  

フェールオーバーの詳細は以下の資料にてご確認ください。  
■ ご参考：  
・ [Azure Cache for Redis のフェールオーバーと修正プログラムの適用](https://learn.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-failover)  
・ [Azure Cache for Redis のフェールオーバーについて - Japan PaaS Support Team Blog](https://azure.github.io/jpazpaas/2020/12/02/cache-failover.html)  
<br>

<a id="faq4"></a>
## [質問4]: 接続元のクライアント側から通信に使用している TLS バージョンの確認方法について知りたいです。

Redis キャッシュ インスタンスの接続元のクライアント側にて TLS 1.0、1.1 の通信を利用している場合は、2025 年 3 月 1 日以降、それらのクライアントからはアクセスができなくなります。

実際にクライアント側がどの TLS バージョンで接続しているか、Azure Cache for Redis の側から判断するログなどはございません。  
クライアント側が利用している TLS バージョンが不明な場合、お手元の環境で TLS 1.2 の検証用の Redis キャッシュ インスタンスをご用意いただき、実際にクライアント側からの接続に問題がないか、事前のテストをご検討ください。
 
■ご参考：[アプリケーションが既に準拠しているかどうかを確認する](https://learn.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-remove-tls-10-11#check-whether-your-application-is-already-compliant)  
上記ドキュメントより  
～～～ 引用ここから ～～～  
アプリケーションが TLS 1.2 で機能するかどうかを確認するには、テストまたはステージング キャッシュで TLS のバージョン要件を 1.2 以上に設定して、テストを実行します。 [TLS の最小バージョン] 設定は、Azure portal 内のキャッシュ インスタンスの [詳細設定] にあります。 この変更後もアプリケーションが期待どおりに機能する場合、そのアプリは TLS 1.2 以降を使っています。  
～～～ 引用ここまで ～～～  

以下の資料内では、いくつかのプログラミング言語について、TLS バージョンに関する言及の記載がございます。

■ご参考：[TLS 1.2 以降を使うようにアプリケーションを構成する](https://learn.microsoft.com/ja-jp/azure/azure-cache-for-redis/cache-remove-tls-10-11#configure-your-application-to-use-tls-12-or-later)  
<br>

---

<br>
<br>

2024 年 11 月 01 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>