---
title: "Storage サービス SAS の作成手順 (Azure Portal, Azure Storage Explorer, PowerShell, Azure CLI)"
author_name: "a-yukiarai"
tags:
    - "Storage"
---

# はじめに
お客様がShared Access Signatures ( SAS ) をご利用になる際には SASトークンを取得する必要があります。

なお前提として、権限は最小特権の原則に従うべきものとなります。
- ご参考： [SAS を使用する際のベスト プラクティス](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#best-practices-when-using-sas)

## Shared Access Signatures ( SAS ) とは
Shared Access Signature ( SAS ) とは、ストレージ アカウント内のリソースへのセキュリティで保護された、署名された URL を介した制限付きの委任アクセス機能です。
SAS を使用すると、クライアントがデータにアクセスする方法をきめ細かく制御できます。
なお、SAS トークンは以下の 3 つの種類がございます。
- ユーザー委任 SAS
- サービス SAS
	- ご参考：本記事
- アカウント SAS
	- ご参考：[Storage アカウント SAS の作成手順 (Azure Portal, Azure Storage Explorer, PowerShell, Azure CLI)](https://azure.github.io/jpazpaas/2024/02/19/Procedure-for-creating-a-Storage-Account-SAS-token.html#gsc.tab=0)

ご参考：[Shared Access Signatures (SAS) でデータの制限付きアクセスを付与する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview)

この Blog 記事では**サービス SAS** について、Azure portal、Azure Storage Explorer、PowerShell、およびAzure CLIを使用した 4 通りの作成方法を紹介します。

## サービス SAS とは
サービス SAS は、アカウント アクセス キーを使用して署名され、次のうち **1 つだけ**の Azure Storage サービスのリソースへのアクセスを委任します。
- Blob コンテナー・Blob
- ファイル共有
- キュー
- テーブル

またサービス SAS は、**アクセス ポリシー**を使用して SAS のアクセス許可と期間を定義することができることが他の SAS との大きな違いとなります。

ご参考：[サービス SAS の作成](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-service-sas)


---

# SAS 利用上のご注意

- SAS トークンの保存について
SAS トークンは、Azure Storage によって追跡されません。お客様側で計算して生成される署名文字列です。そのため、**ウィンドウが閉じられると SAS トークンの値は取得できなくなるため、お客様にて生成された値をコピーして安全な場所に保存しておく必要があります。** 
	- ご参考：[SAS トークンの保存](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#sas-token)

- SAS URL について
SAS URL を作成するには、SAS トークン ( URI ) をストレージ サービスのリソース URL に追加します。作成時に各プラットホームから返される SAS トークンには、URL クエリ文字列に必要な区切り文字 ( "?" ) が含まれていません。**SAS URL を作成する際は、必ずリソース URL と SAS トークンの間に区切り文字を追加してください。**
	- ご参考：[SAS URL について](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json#sas-token)

 - SAS の失効について
有効な SAS を所有するすべてのクライアントは、その SAS で許可されているストレージ アカウントのデータにアクセスできます。 SAS を悪意のある、または意図しない用途から保護することが重要です。 **SAS の配布は慎重に行い、侵害された SAS を失効させるための計画を用意しておいてください。**
	- ご参考 1 ： [Shared Access Signature (SAS) トークンの失効方法](https://azure.github.io/jpazpaas/2023/06/15/How-To-Expire-SAS-Token.html#gsc.tab=0)
	- ご参考 2 ： [SAS を取り消す](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-service-sas#revoke-a-sas)


---

# サービス SAS の作成を行う際に設定する主なパラメータについて

サービス SAS の作成に際しては、主に下記のパラメータを使用します。

- **署名方法 ( Signing method ):**
サービス SAS を作成する際は、「アカウント キー(Account key)」を選択します。
	
- **保存されているアクセス ポリシー( Stored access policy ):**
	- 保存されているアクセス ポリシーを使用して、SAS のアクセス許可と期間を定義することができます。 既存の保存されているアクセス ポリシーの名前が指定されている場合、そのポリシーは、SAS に関連付けられます。 
		- ご参考： [格納されているアクセス ポリシーを定義する](https://learn.microsoft.com/ja-JP/rest/api/storageservices/define-stored-access-policy)
	- SAS が侵害された場合は、できるだけ早くその SAS を失効させることをお勧めします。 保存されているアクセス ポリシーに関連付けられているサービス SAS を失効させるには、保存されているアクセス ポリシーを削除するか、ポリシーの名前を変更するか、または有効期限を過去の時間に変更します。
	- 保存されているアクセス ポリシーを指定しない場合は、SAS のアクセス許可と期間を設定してご使用ください。保存されているアクセス ポリシーに関連付けられていないサービス SAS を失効させることはできません。 このため、SAS の有効期限が **1 時間以下**になるように制限することをお勧めします。
		- ご参考： [ID 管理とアクセス管理](https://learn.microsoft.com/ja-jp/azure/storage/blobs/security-recommendations#identity-and-access-management)

- **開始日時と有効期限の日時 ( Start and expiry date-time ):**
SAS トークンの有効期間を定義します。開始日時と終了日時を設定できます。指定を省略した場合は、それぞれの作成方法でデフォルトの有効期限が設定されます。
	- SAS を発行する場合は、可能な限り短い有効期限を使用するようにします。
		-  ご参考 1 ：[BLOB ストレージのセキュリティに関する推奨事項](https://learn.microsoft.com/ja-jp/azure/storage/blobs/security-recommendations)
		- ご参考 2 ：[SAS を使用する際のベスト プラクティス](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview#best-practices-when-using-sas)
	
- **アクセス許可 ( Permissions ):**
	- SAS が許可する操作を指定します。読み取り ( r ) 、書き込み ( w ) 、削除 ( d ) 、リスト表示 ( l ) 、追加 ( a ) 、作成 ( c ) 、更新 ( u ) 、処理 ( p ) などの操作を選択できます。
	- 選択できるアクセス許可はサービス SAS の対象となるリソースによって異なります。
		- ご参考：[アクセス許可を指定する](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-service-sas#specify-permissions)
	- サービス SAS では、特定の操作へのアクセスを許可できません。
		- コンテナー、キュー、およびテーブルを作成、削除、または一覧表示することはできません。
		- コンテナーのメタデータとプロパティを読み取ったり書き込んだりすることはできません。
		- キューをクリアすることはできません。また、そのメタデータを書き込むことができません。
		- コンテナーをリースすることはできません。

     これらの操作へのアクセスを許可する SAS を構築するには、アカウント SAS をご使用ください。
			- ご参考：[アカウント SAS を作成する](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-account-sas)
	
- **使用できる IP アドレス ( Allowed IP addresses ):**
SAS を使用できる IP アドレスを指定します。ただし、パブリック IP アドレスのみ設定可能、Azure リソースからのアクセスの場合は異なるリージョンのパブリック IP アドレスであることにご留意ください。
	- ご参考：[IP アドレスまたは IP 範囲を指定する](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-account-sas#specify-an-ip-address-or-ip-range)

- **許可されるプロトコル ( Allowed protocols ):**
SAS を使用できるプロトコルを指定します。「 HTTPS のみ( HttpsOnly ) 」または「 HTTPS と HTTP ( HttpsOrHttp ) 」を選択できます。

- **署名キー ( Signing key ):**
SAS トークンの生成と検証に使用されます。key1 または key2 のどちらかを指定します。

- **使用できるリソースの種類 ( SignedResource )：**
サービスSAS がアクセスを許可するリソースの種類を指定します。
	- Blob Storage のみ
     BLOB ( b )	、BLOB バージョン ( Bv )、BLOB スナップショット ( bs )、コンテナー ( c )、ディレクトリ ( d ) のいずれか、または複数を選択できます。
		- ご参考：[署名されたリソースを指定する (Blob Storage のみ)](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-service-sas#specify-the-signed-resource-blob-storage-only)

	- Azure Files
     共有リソースがファイルかどうかを指定 ( f )、 共有リソースが共有であるかどうかを指定 ( s ) のいずれか、または複数を選択できます。
		- ご参考：[署名されたリソースを指定する (Azure Files)](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-service-sas#specify-the-signed-resource-azure-files)


---

# Blob のサービス SAS の作成手順

下記では以下の4つの条件を許可する設定の例をご案内します。

- **あるコンテナー内の特定の <span style="color: red; ">Blob</span> に対して、**
- **<span style="color: red; ">読み取り</span>のみ、**
- **<span style="color: red; ">HTTPS 接続</span>のみ、**
- **有効期限が「<span style="color: red; ">現在～ 30 分後</span>」** 

## Azure Portal を使用してサービス SAS を作成する

(1) Azure Portalにログインし、ポータル上部の検索欄に該当のストレージ アカウント名を入力・検索し、選択します。

(2) サービス SAS を作成したい Blob の名称上で右クリックし、「SAS の作成」を選択します。

![image-3fc8d291-d87b-4b31-a831-d5e9c1ca3805.png]({{site.baseurl}}/media/2024/02/image-3fc8d291-d87b-4b31-a831-d5e9c1ca3805.png)

(3) 必要なオプションを指定します。

![image-310dff2c-d9c5-4b2e-9890-57380767a0d9.png]({{site.baseurl}}/media/2024/02/image-310dff2c-d9c5-4b2e-9890-57380767a0d9.png)

(4) 必要事項の設定後、「SAS トークンおよび URL を生成」を選択します。

![image-c30ee357-6939-42ae-8099-ae306a685dfd.png]({{site.baseurl}}/media/2024/02/image-c30ee357-6939-42ae-8099-ae306a685dfd.png)

(5) SASトークンが生成されます。

![image-0b8aa289-6633-43a2-b841-31353f443bf2.png]({{site.baseurl}}/media/2024/02/image-0b8aa289-6633-43a2-b841-31353f443bf2.png)


## Azure Storage Explorer を使用してサービス SAS を作成する

(1) Azure Storage Explorer を起動します。

(2) 左側のパネルで対象のストレージ アカウント内の 該当の Blob の名称上で右クリックし、「Shared Access Signatureの取得…」 を選択します。

![image-8e80e05f-3b34-4b57-998d-4188e26e9765.png]({{site.baseurl}}/media/2024/02/image-8e80e05f-3b34-4b57-998d-4188e26e9765.png)

(3) Shared Access Signature ダイアログ ボックスが表示されます。
必要なオプションを指定します。

![image-941d1688-b426-4e9d-a457-5fb3bfc21b62.png]({{site.baseurl}}/media/2024/02/image-941d1688-b426-4e9d-a457-5fb3bfc21b62.png)

(4) 必要事項の設定後、「作成」を押下します。

![image-74f0dcb7-1853-4ba6-bf59-4e0721414d3f.png]({{site.baseurl}}/media/2024/02/image-74f0dcb7-1853-4ba6-bf59-4e0721414d3f.png)

(5) SAS トークンが生成されます。

![image-8445032b-b0d8-45b3-9c92-05753a6d99ea.png]({{site.baseurl}}/media/2024/02/image-8445032b-b0d8-45b3-9c92-05753a6d99ea.png)


## PowerShellを使用してサービス SAS を作成する

(1) サブスクリプションへ接続します。

```

Connect-AzAccount -Subscription <Subscription Id>

```

(2) SASトークンの作成に必要な各種パラメータの変数を設定していきます。

```

$accountName = "<storage-account>"
$containerName = "<container-name>"
$blobName = "<blob-name>"
$permissions = "r"
$protocol = "HttpsOnly"
$startTime = Get-Date
$expiryTime = $startTime.AddMinutes(30)

```

- 上記では省略されているパラメータもご利用可能です。
	- ご参考：[New-AzStorageBlobSASToken (Az.Storage)](https://learn.microsoft.com/ja-jp/powershell/module/az.storage/new-azstorageblobsastoken?view=azps-11.2.0)

(3) Azure Storage に対する操作を行う際に認証情報や接続情報を保持するためのオブジェクトである Azure Storage コンテキストを作成します。

```

$ctx = New-AzStorageContext -StorageAccountName $accountName

```

(4) SAS トークンを作成します。

```

New-AzStorageBlobSASToken `
	-Container $containerName `
	-Blob $blobName `
	-Permission $permissions `
	-Protocol $protocol `
	-StartTime $startTime `
	-ExpiryTime $expiryTime `
	-Context $ctx

```

(5) 生成された SAS トークンが表示されます。

![image-7184614a-51d7-46b7-96b1-ad400a649d72.png]({{site.baseurl}}/media/2024/02/image-7184614a-51d7-46b7-96b1-ad400a649d72.png)


## Azure CLI を使用してサービス SAS を作成する

(1) Azure にログインします。

```

az login

```

(2) サブスクリプションへ接続します。

```

az account set --subscription <Subscription Id>

```

(3) SASトークンの作成に必要な各種パラメータの変数を設定していきます。

```

storageAccount="<storage-account>"
containerName="<container-name>"
resourceGroup="<resource-group>"
blobName="<blob-name>"
expiry=$(date -u -d "30 minutes" '+%Y-%m-%dT%H:%MZ')
permissions="r" 

```

- 上記では省略されているパラメータもご利用可能です。
	- ご参考： [az storage blob generate-sas](https://learn.microsoft.com/ja-jp/cli/azure/storage/blob?view=azure-cli-latest#az-storage-blob-generate-sas)

(4) ストレージ アカウントの接続文字列を取得します。

```

connectionString=$(az storage account show-connection-string \
	--name $storageAccount \
	--resource-group $resourceGroup \
	--output tsv)
	
```

(5) SAS トークンを作成します。

```

az storage blob generate-sas \
	--account-name $storageAccount \
	--connection-string $connectionString \
	--container-name $containerName \
	--name $blobName \
	--permissions $permissions \
	--expiry $expiry \
	--https-only

```

(6) 生成された SAS トークンが表示されます。

![image-5554483e-cbef-4d95-ac9a-7080fb0c6916.png]({{site.baseurl}}/media/2024/02/image-5554483e-cbef-4d95-ac9a-7080fb0c6916.png)


---

# 保存されているアクセス ポリシーを使用したサービス SAS の作成例 ( Azure Portal )

以下に、保存されているアクセス ポリシーを使用した、 Blob へのサービス SAS の作成で最も簡単なAzure Portal での作成方法をご案内します。
- ご参考： [格納されているアクセス ポリシーを定義する](https://learn.microsoft.com/ja-JP/rest/api/storageservices/define-stored-access-policy)

(1) アクセス ポリシーを作成し、保存します。 (保存済みの場合は手順 (5) にお進みください。)
Blob のアクセス ポリシーはその Blob を内包するコンテナーに対して設定します。
該当のコンテナーの名称上で右クリックし、「アクセス ポリシー」を選択します。

![image-0787f3e6-08ca-412a-bc0d-3c8d8a8394f6.png]({{site.baseurl}}/media/2024/02/image-0787f3e6-08ca-412a-bc0d-3c8d8a8394f6.png)

(2) アクセス ポリシー ブレード内の「＋ ポリシーの追加」を選択します。

![image-ffd296a9-aefe-456b-864f-2916bdae039b.png]({{site.baseurl}}/media/2024/02/image-ffd296a9-aefe-456b-864f-2916bdae039b.png)

(3) ID 欄にアクセス ポリシー名を入力し、アクセス許可・開始時刻・有効期限を設定後「 OK 」を押下します。

![image-37d23e35-b4c5-4d18-bf65-ffdabb5f0a5f.png]({{site.baseurl}}/media/2024/02/image-37d23e35-b4c5-4d18-bf65-ffdabb5f0a5f.png)

(4) リスト内に作成したアクセス ポリシーが表示されたことを確認したら、「保存」を選択します。

![image-bac60b36-9b52-4212-89dc-04f0cabf4592.png]({{site.baseurl}}/media/2024/02/image-bac60b36-9b52-4212-89dc-04f0cabf4592.png)

以上でアクセス ポリシーの作成及び保存ができました。

(5) 保存されているアクセス ポリシーを使用した SAS トークンを作成します。
サービス SAS を作成したい Blob の名称上で右クリックし、「 SAS の作成」を選択します。

![image-288b86dd-e4d5-41e9-8663-32dc57607903.png]({{site.baseurl}}/media/2024/02/image-288b86dd-e4d5-41e9-8663-32dc57607903.png)

(6) 「保存されているアクセス ポリシー」欄で、使用したいアクセス ポリシー名を選択し、その他必要事項の設定後、「SAS トークンおよび URL を生成」を選択します。
「保存されているアクセス ポリシー」が選択されると、アクセス許可及び開始日時と有効期限の日時欄がグレーアウトして入力できなくなることをご確認ください。

![image-36851203-f533-46dc-bf26-135231c2e106.png]({{site.baseurl}}/media/2024/02/image-36851203-f533-46dc-bf26-135231c2e106.png)

(7) SASトークンが生成されます。

![image-422cabc2-f92f-48f6-b467-34dc540479a3.png]({{site.baseurl}}/media/2024/02/image-422cabc2-f92f-48f6-b467-34dc540479a3.png)


---

# SAS を使用する

上記の手順で作成した SAS トークンの簡単な使い方をご案内いたします。

(1) 該当のBLOBのURLを取得します。

![image-f91add2e-b850-4982-91f6-dea37d69dc22.png]({{site.baseurl}}/media/2024/02/image-f91add2e-b850-4982-91f6-dea37d69dc22.png)

(2) ブラウザのアドレス欄に、コピーしたURLに区切り文字「 ? 」および SAS トークンを加えた文字列を入力し、BLOBにアクセスします。

ブラウザのアドレス欄には以下のような内容を貼り付けています。

![image-86ba456f-3430-479e-b02f-b57f71037121.png]({{site.baseurl}}/media/2024/02/image-86ba456f-3430-479e-b02f-b57f71037121.png)

![image-de574f43-8cf1-4214-ac54-c6f18aa978f4.png]({{site.baseurl}}/media/2024/02/image-de574f43-8cf1-4214-ac54-c6f18aa978f4.png)


<br>
<br>

---

<br>
<br>

2024 年 02 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>