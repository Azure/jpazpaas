---
title: "Storage アカウント SAS の作成手順 (Azure Portal, Azure Storage Explorer, PowerShell, Azure CLI)"
author_name: "a-mnanami"
tags:
    - "Storage"
---

[[_TOC_]]

---

# はじめに
お客様が Shared Access Signatures ( SAS ) をご利用になる際には SAS トークンを取得する必要があります。

なお前提として、権限は最小特権の原則に従うべきものとなります。
ご参考： [SAS を使用する際のベスト プラクティス](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#best-practices-when-using-sas)

## Shared Access Signatures ( SAS ) とは
Shared Access Signature ( SAS ) とは、ストレージ アカウント内のリソースへのセキュリティで保護された、署名された URL を介した制限付きの委任アクセス機能です。
SAS を使用すると、クライアントがデータにアクセスする方法をきめ細かく制御できます。
なお、SAS トークンは以下の 3 つの種類がございます。
- ユーザー委任 SAS
- サービス SAS
  ご参考：[Procedure-for-creating-a-Storage-Service-SAS-token](https://dev.azure.com/jpazpaas/blog/_wiki/wikis/blogwiki/7769/Procedure-for-creating-a-Storage-Service-SAS-token)
- アカウント SAS
  ご参考：本記事

ご参考：[Shared Access Signatures (SAS) でデータの制限付きアクセスを付与する - Azure Storage | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview)

この Blog 記事は **アカウント SAS** について、Azure Portal、Azure Storage Explorer、PowerShell、Azure CLI を使用した 4 通りの作成方法を紹介します。

## アカウント SAS とは
アカウント SAS は、アカウント アクセス キーを使用して署名されます。
 **ストレージ アカウント全体へのアクセスを委任** します。

他の SAS との違いかつアカウント SAS の大きな特徴には以下の 2 点が挙げられます。
- 複数のサービスにまたがった SAS を発行できる 
- サービス SAS では利用できるアクセス ポリシーが利用できない

ご参考： [アカウント SAS の作成 ( REST API )](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-account-sas)

---
## SAS 利用上のご注意

- SAS トークンの保存について
SAS トークンは、Azure Storage によって追跡されません。お客様側で計算して生成される署名文字列です。そのため、**ウィンドウが閉じられると SAS トークンの値は取得できなくなるため、お客様にて生成された値をコピーして安全な場所に保存しておく必要があります。** 
ご参考：[SAS トークンの保存](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#sas-token)

- SAS URL について
SAS URL を作成するには、SAS トークン ( URI ) をストレージ サービスのリソース URL に追加します。作成時に各プラットホームから返される SAS トークンには、URL クエリ文字列に必要な区切り文字 ( "?" ) が含まれていません。**SAS URL を作成する際は、必ずリソース URL と SAS トークンの間に区切り文字を追加してください。**
ご参考：[SAS URL について](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#sas-token)

 - SAS の失効について
有効な SAS を所有するすべてのクライアントは、その SAS で許可されているストレージ アカウントのデータにアクセスできます。 SAS を悪意のある、または意図しない用途から保護することが重要です。 **SAS の配布は慎重に行い、侵害された SAS を失効させるための計画を用意しておいてください。**
ご参考： [Shared Access Signature (SAS) トークンの失効方法](https://azure.github.io/jpazpaas/2023/06/15/How-To-Expire-SAS-Token.html#gsc.tab=0)

---

# アカウント SAS の作成を行う際に設定する主なパラメータについて
アカウント SAS の作成に際しては、主に下記のパラメータを使用します。

- **使用できるサービス ( Available Services )：** 
SAS がアクセスを許可するストレージ サービスを指定します。Blob ( b ) 、File ( f ) 、Queue ( q ) 、Table ( t )  のいずれか、または複数を選択できます。
Azure Portal、Azure Storage Explorer、PowerShell、Azure CLI で設定が必須です。

- **使用できるリソースの種類 ( Resource Types )：**
SAS がアクセスを許可するリソースの種類を指定します。Service ( s ) 、Container ( c ) 、Object ( o ) のいずれか、または複数を選択できます。
Azure Portal、Azure Storage Explorer、PowerShell、Azure CLI で設定が必須です。
	
- **アクセス許可 ( Permissions ) ：**
SAS が許可する操作を指定します。読み取り ( r ) 、書き込み ( w ) 、削除 ( d ) 、リスト表示 ( l ) 、追加 ( a ) 、作成 ( c ) 、更新 ( u ) 、処理 ( p ) などの操作を選択できます。
	- 選択できるアクセス許可はサービス、リソースの種類の組み合わせによって異なります。
ご参考：[操作別のアカウント SAS アクセス許可](https://learn.microsoft.com/ja-JP/rest/api/storageservices/create-account-sas?redirectedfrom=MSDN#account-sas-permissions-by-operation)
	
- **BLOB バージョン管理のアクセス許可 ( Blob Versioning Permissions )：**
	SAS が許可する Blob バージョンの操作を指定します。Blob バージョニングは、オブジェクトの以前のバージョンを自動的に保持するために Blob ストレージのバージョニングを有効にすることができます。Blob バージョニングが有効になっている場合、Blob が変更または削除された場合にデータを復元するために以前のバージョンの Blob にアクセスできます。

- **許可された BLOB インデックスのアクセス許可 ( Allowed Blob Indexing Permissions ) ：**
SAS が許可する Blob インデックスの操作を指定します。特定の Blob インデックスに対する読み取り、書き込み、削除などの操作を制御します。

- **開始日時と有効期限の日時 ( Start and Expiry Date-Time ) ：**
SAS トークンの有効期間を定義します。開始日時と終了日時を設定できます。指定を省略した場合は、それぞれの作成方法でデフォルトの有効期限が設定されます。Azure CLI のみ終了日時の設定が必須です。
	- SAS を発行する場合は、可能な限り短い有効期限を使用するようにします。
	  - ご参考1：[BLOB ストレージのセキュリティに関する推奨事項](https://learn.microsoft.com/ja-jp/azure/storage/blobs/security-recommendations)
	  - ご参考2：[SAS を使用する際のベスト プラクティス](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview#best-practices-when-using-sas)
	
- **使用できる IP アドレス ( Allowed IP Addresses ) ：**
	SAS を使用できる IP アドレスを指定します。
	ただし、パブリック IP アドレスのみ設定可能、Azure リソースからのアクセスの場合は異なるリージョンのパブリック IP アドレスであることにご留意ください。
	ご参考：[IP アドレスまたは IP 範囲を指定する](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-account-sas#specify-an-ip-address-or-ip-range)

- **許可されるプロトコル ( Allowed Protocols ) ：**
SAS トークンを HTTPS プロトコルのみで使用できるようにする場合は、このフラグを指定します。
SAS を使用できるプロトコルを指定します。「 HTTPS のみ」または「 HTTPS と HTTP 」を選択できます。
	
- **署名キー ( Signing Key ) ：**
SAS トークンの生成と検証に使用されます。key1 または key2 のどちらかを指定します。
	
- **暗号化スコープ ( Encryption-scope ) ：**
サービス上のデータを暗号化するために使用される定義済みの暗号化スコープです。PowerShell、Azure CLI のみ使用可能です。
ご参考：[BLOB ストレージの暗号化スコープ](https://learn.microsoft.com/ja-jp/azure/storage/blobs/encryption-scope-overview)


# アカウント SAS の作成手順
下記では以下の4つの条件を許可する設定の例をご案内します。
|- ストレージ アカウント内の <span style="color: red; ">Blob</span> に対して - 使用できるリソースの種類は <span style="color: red; ">オブジェクト</span> のみ- <span style="color: red; ">読み取り</span>のみ、- 有効期限が「<span style="color: red; ">現在～ 30 分後</span>」 |
|--|

## Azure Portal を使用してアカウント SAS を作成する
![image-a3f1a099-afd7-4db2-86c4-b933429802af.png]({{site.baseurl}}/media/2024/02/image-a3f1a099-afd7-4db2-86c4-b933429802af.png)
1. Azure Portalにログインし、ポータル上部の検索欄に該当のストレージ アカウント名を入力・検索し、選択します。
1. 左側のメニューから「 Shared Access Signature 」を選択します。
![image-3ce44da6-8318-4030-a5eb-984cc4b2bdab.png]({{site.baseurl}}/media/2024/02/image-3ce44da6-8318-4030-a5eb-984cc4b2bdab.png)
1. 「 Shared Access Signature 」ブレードが開きます。
特定のストレージ サービスやリソースへのアクセス許可を制御する設定をします。
開始日時と有効期限の日時はデフォルトで現在時刻から 8 時間後までとなります。必要に応じてご変更ください。
![image-caecc0e7-6ba6-4842-bd60-64a7cce60c71.png]({{site.baseurl}}/media/2024/02/image-caecc0e7-6ba6-4842-bd60-64a7cce60c71.png)
1. 「 SAS と接続文字列を生成する」を押下します。
![image-e184f7a2-cc95-420b-abbf-f59bd8d397cd.png]({{site.baseurl}}/media/2024/02/image-e184f7a2-cc95-420b-abbf-f59bd8d397cd.png)
1. SAS トークンが生成されます。
![image-d71fe66b-68f1-44a8-a6cd-1b3365cdac06.png]({{site.baseurl}}/media/2024/02/image-d71fe66b-68f1-44a8-a6cd-1b3365cdac06.png)

## Azure Storage Explorer を使用してアカウント SAS を作成する
![image-30b1ce82-31b3-41d0-b11a-b5a2ae22876c.png]({{site.baseurl}}/media/2024/02/image-30b1ce82-31b3-41d0-b11a-b5a2ae22876c.png)
1. Azure Storage Explorer を起動します。
1. 左側のパネルで対象のストレージ アカウントを右クリックし、「 Shared Access Signatureの取得… 」を選択します。
![image-b7730612-7c7e-4c61-b825-7b7f5e4a0181.png]({{site.baseurl}}/media/2024/02/image-b7730612-7c7e-4c61-b825-7b7f5e4a0181.png)
1. Shared Access Signature ダイアログ ボックスが表示されます。
特定のストレージ サービスやリソースへのアクセス許可を制御する設定をします。
開始日時と有効期限の日時はデフォルトで現在時刻から 24 時間後までとなります。
![image-e5990e88-aaee-4729-8581-64c6c49b1de9.png]({{site.baseurl}}/media/2024/02/image-e5990e88-aaee-4729-8581-64c6c49b1de9.png)
1. 「作成」を押下します。
![image-d63e4e01-b857-4b9f-928a-53e166055ba8.png]({{site.baseurl}}/media/2024/02/image-d63e4e01-b857-4b9f-928a-53e166055ba8.png)
1. SAS トークンが生成されます。
![image-a7c415c3-df88-4122-a25c-fc776f8b577f.png]({{site.baseurl}}/media/2024/02/image-a7c415c3-df88-4122-a25c-fc776f8b577f.png)

## PowerShell を使用してアカウント SAS を作成する
![image-f56975b7-977c-4e48-b062-5e4d9185f91a.png]({{site.baseurl}}/media/2024/02/image-f56975b7-977c-4e48-b062-5e4d9185f91a.png)

1. サブスクリプションへ接続します。
```
Connect-AzAccount -Subscription <Subscription Id>
```

2. SAS トークンの作成に必要な各種パラメータの変数を設定していきます。

```
$storageAccount = Get-AzStorageAccount `
    -ResourceGroupName "<resource-group>" `
    -Name "<storage-account>"
$services = "Blob"
$resourceTypes = "Object"
$permissions ="r"
$startTime = Get-Date
$expiryTime = $startTime.AddMinutes(30)
```
- 上記では省略されているパラメータもご利用可能です。
ご参考：[New-AzStorageAccountSASToken (Az.Storage) | Microsoft Learn](https://learn.microsoft.com/ja-jp/powershell/module/az.storage/new-azstorageaccountsastoken?view=azps-11.2.0)

3. SAS トークンを作成します。
```
New-AzStorageAccountSASToken `
   -Service $services `
   -ResourceType $resourceTypes `
   -Permission $permissions `
   -StartTime $startTime `
   -ExpiryTime $expiryTime `
   -Context $storageAccount.Context
```

4. 上記を実行すると SAS トークンが表示されます。
![image-c57ac90e-8b67-4a8f-816a-5fd433cf7bfc.png]({{site.baseurl}}/media/2024/02/image-c57ac90e-8b67-4a8f-816a-5fd433cf7bfc.png)

また、本コマンドを変数に格納することで後から容易に SAS トークンを確認することが可能となります。
```
$sas = New-AzStorageAccountSASToken `
    -Service $services `
    -ResourceType $resourceTypes `
    -Permission $permissions `
    -StartTime $startTime `
    -ExpiryTime $expiryTime `
    -Context $storageAccount.Context
```

SAS トークンの表示は「 $sas 」を入力します。

## Azure CLI を使用してアカウント SAS を作成する
![image-2a27c78b-240b-4b94-8a3c-7b834e1766d6.png]({{site.baseurl}}/media/2024/02/image-2a27c78b-240b-4b94-8a3c-7b834e1766d6.png)
1. Azure にログインします。
```
az login
```

2. サブスクリプションへ接続します。
```
az account set --subscription <Subscription Id>
```

3. SAS トークンの作成に必要な各種パラメータの変数を設定していきます。
```
resourceGroup="<resource-group>"
storageAccount="<storage-account>"
services="b"
resourceTypes="o"
permissions="r"
expiry=$(date -u -d "30 minutes" '+%Y-%m-%dT%H:%MZ')
```
- 上記では省略されているパラメータもご利用可能です。
ご参考：[az storage account | Microsoft Learn](https://learn.microsoft.com/ja-jp/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-generate-sas)

4. ストレージ アカウントの接続文字列を取得します。
```
connectionString=$(az storage account show-connection-string --name $storageAccount --resource-group $resourceGroup --output tsv)
```


5. SAS トークンを作成します。
```
az storage account generate-sas \
    --connection-string $connectionString \
    --services $services \
    --resource-types $resourceTypes \
    --permissions $permissions \
    --expiry $expiry \
    --https-only
```
6. 生成された SAS トークンが表示されます。
![image-29c34067-597f-46b1-a540-747bd226eb5e.png]({{site.baseurl}}/media/2024/02/image-29c34067-597f-46b1-a540-747bd226eb5e.png)

---

# SAS を使用する
上記の手順で作成した SAS トークンの簡単な使い方をご案内いたします。

1. 該当のBLOBのURLを取得します。
![image-af21a938-8e19-4728-a273-ba0d6572c274.png]({{site.baseurl}}/media/2024/02/image-af21a938-8e19-4728-a273-ba0d6572c274.png)
1. ブラウザのアドレス欄に、コピーしたURLに区切り文字「 ? 」および SAS トークンを加えた文字列を入力し、BLOBにアクセスします。

ブラウザのアドレス欄には以下のような内容を貼り付けています。
![image-86ba456f-3430-479e-b02f-b57f71037121.png]({{site.baseurl}}/media/2024/02/image-86ba456f-3430-479e-b02f-b57f71037121.png)

![image-de574f43-8cf1-4214-ac54-c6f18aa978f4.png]({{site.baseurl}}/media/2024/02/image-de574f43-8cf1-4214-ac54-c6f18aa978f4.png)

---

<br>
<br>

2024 年 02 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>