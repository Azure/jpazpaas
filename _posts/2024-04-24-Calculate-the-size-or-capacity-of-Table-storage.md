---
title: "Table ストレージのエンティティ数と容量の確認方法"
author_name: "a-yukiarai"
tags:
    - storage
---

# Table ストレージのエンティティ数と容量の確認方法

この Blog 記事では Table ストレージ について、 Azure Storage Explorer と PowerShell を使用してTable 内のエンティティの数を確認する方法とTable 内の各エンティティの容量の計算方法をご紹介いたします。

(目次)<br>
[1. Azure Storage Explorer を使用して、Table 内のエンティティ数を確認する](#storageexplorer)<br>
[2. PowerShell を使用して、Table 内のエンティティ数を確認する](#powershell)<br>
[3. Table 内の各エンティティの容量の計算方法](#size)

---

※ **Blobコンテナーのコンテナーごとの容量の確認方法**については下記のブログ記事をご覧ください。
- [Blobコンテナーのコンテナーごとの容量確認方法](https://azure.github.io/jpazpaas/2021/10/06/blob-container-usage-per-container.html)

---
<a id="storageexplorer"></a>
## 1. Azure Storage Explorer を使用して、Table 内のエンティティ数を確認する

(1) Azure Storage Explorer を起動します。

(2) 左側のパネルで対象のストレージ アカウント内の該当のテーブルを選択します。

![image-39ae7131-c7e0-4184-b9b6-63987eeae607.png]({{site.baseurl}}/media/2024/04/image-39ae7131-c7e0-4184-b9b6-63987eeae607.png)

(3) ウィンドウ右側のテーブル内容のメニューから「テーブルの統計」をクリックします。

![image-5db4b6e7-b1b1-4794-a37c-557ef8e8ecf3.png]({{site.baseurl}}/media/2024/04/image-5db4b6e7-b1b1-4794-a37c-557ef8e8ecf3.png)

(4) ウィンドウ下部の「アクティビティ」欄に、このテーブル内のエンティティ数が表示されます。

![image-75543cc1-c9ba-484e-afd6-4f168a97ac42.png]({{site.baseurl}}/media/2024/04/image-75543cc1-c9ba-484e-afd6-4f168a97ac42.png)

---
<a id="powershell"></a>
## 2. PowerShell を使用して、Table 内のエンティティ数を確認する

**※ご注意ください※**<br>
PowerShell からこの Azure 機能を使用いただくには、`Az` モジュールがインストールされている必要があります。 `AzTable` の現在のバージョンは、以前の AzureRM モジュールと互換性がありません。 必要に応じて、[Az モジュールの最新のインストール手順](https://learn.microsoft.com/ja-jp/powershell/azure/install-azure-powershell?view=azps-11.5.0)に則って作業くださいますようお願いいたします。

ご参考：[PowerShell を使用した Azure Table Storage 操作の実行](https://learn.microsoft.com/ja-jp/azure/storage/tables/table-storage-how-to-use-powershell?toc=https%3A%2F%2Flearn.microsoft.com%2Fja-jp%2Fazure%2Fstorage%2Ftables%2Ftoc.json&bc=https%3A%2F%2Flearn.microsoft.com%2Fja-jp%2Fazure%2Fbread%2Ftoc.json)


(1) サブスクリプションへ接続します。

```

Connect-AzAccount -Subscription <Subscription Id>

```

(2) Table ストレージへの接続に必要な変数を設定します。

```

$storageAccountName = "<storage-account>"
$resourceGroup = "<resource-group>"

```

(3) Azure Storage に対する操作を行う際に認証情報や接続情報を保持するためのオブジェクトである Azure Storage コンテキストを作成します。

```

$storageAccount = Get-AzStorageAccount -ResourceGroupName $resourceGroup -Name $storageAccountName
$ctx = $storageAccount.Context

```

- (ご参考) ストレージ アカウント内の Table の一覧を表示します。

```

Get-AzStorageTable -Context $ctx | select Name


(上記のコマンドの実行結果の表示例)

Name
----
sampletable001
sampletable002
```

(4) 確認したい Table 内のエンティティ数を表示します。

```

$tableName = "<table_name>"
$storageTable = Get-AzStorageTable -Name $tableName -Context $ctx
$cloudTable = $storageTable.CloudTable

$totalEntities=(Get-AzTableRow -table $cloudTable | measure).Count
Echo $totalEntities


(上記コマンドの実行結果の表示例)
4

```

- (ご参考)  Table 内のエンティティの一覧を表示します。

```

Get-AzTableRow -table $cloudTable | ft


(上記のコマンドの実行結果の表示例)

LastName FirstName PartitionKey    RowKey    TableTimestamp             Etag
-------- --------- ------------    ------    --------------             ----
Aaa      Bbb       mypartarionkey  mtrowkey1 2024/04/04 14:48:24 +09:00 W/"datetime'2024-04-04T05%3A48%3A24.7867297Z'"
Ccc      Ddd       mypartarionkey  mtrowkey2 2024/04/04 16:40:19 +09:00 W/"datetime'2024-04-04T07%3A40%3A19.2571354Z'"
Aab      Bba       mypartarionkey2 mtrowkey1 2024/04/04 16:40:49 +09:00 W/"datetime'2024-04-04T07%3A40%3A49.1851516Z'"
Ccd      Ddc       mypartarionkey2 mtrowkey2 2024/04/04 16:41:18 +09:00 W/"datetime'2024-04-04T07%3A41%3A18.1617072Z'"

```
---
<a id="size"></a>
## 3. Table 内の各エンティティの容量の計算方法

Table 内のエンティティごとに消費されるストレージの容量を見積もる計算式を下記にご案内いたします。

ご参考1：[Calculate the size/capacity of storage account and it services (Blob/Table)  \#8. Calculate the size of each entity in azure storage table](https://techcommunity.microsoft.com/t5/azure-paas-blog/calculate-the-size-capacity-of-storage-account-and-it-services/ba-p/1064046)<br>
ご参考2： [How the size of an entity is caclulated in Windows Azure table storage? | Microsoft Learn](https://learn.microsoft.com/en-US/archive/blogs/avkashchauhan/how-the-size-of-an-entity-is-caclulated-in-windows-azure-table-storage)


### 各エンティティの容量の計算式:
4 バイト<br> 
   \+ (PartitionKey の文字数 + RowKey の文字数) * 2 バイト<br>
   \+ 各プロパティの容量 (8 バイト + プロパティ名の文字数 * 2 バイト + プロパティの内容)<br>
 

#### ※上記計算式の内訳：
- 各エンティティの 4 バイトのオーバーヘッド
- Unicode として格納される PartitionKey 値と RowKey 値の文字数にそれぞれ 2 バイトを掛けたもの
- 各プロパティ容量については、8 バイトのオーバーヘッド、プロパティ名 * 2 バイト、およびプロパティのデータ内容量 (プロパティのデータ型によって異なります。下記をご参照ください。)の合計

#### ※プロパティのデータ型による容量:
- Binary ... バイナリの容量のバイト数 + バイナリ配列の長さの格納用 4 バイト
- Boolean ... 1 バイト
- DateTime ... 8 バイト
- Double ... 8 バイト
- GUID ... 16 バイト
- Int32 ... 4 バイト
- Int64 ... 8 バイト
- String ... 文字数 * 2 バイト + 文字数の格納用 4 バイト

ご参考： [プロパティの型](https://learn.microsoft.com/ja-jp/rest/api/storageservices/Understanding-the-Table-Service-Data-Model#property-types)


### エンティティの容量の計算方法例:
下図のようなエンティティ内容の場合の計算例をご紹介いたします。<br>
![image-684fedc0-fe40-4e2d-8e85-2ccf0a50842c.png]({{site.baseurl}}/media/2024/04/image-684fedc0-fe40-4e2d-8e85-2ccf0a50842c.png)

- エンティティのオーバーヘッド ... **4** バイト
- PartitionKey 値の文字数に 2 バイトを掛けたもの ... 14 文字 *2=**28** バイト
- RowKey 値の文字数に 2 バイトを掛けたもの ... 8 文字 *2=**16** バイト
- プロパティ 1 容量 (Timestamp、Datetime型) ... オーバーヘッド (8 バイト) 、プロパティ名 (9 文字 *2=18 バイト)、プロパティのデータ内容量 (8 バイト) → 小計 **34** バイト
- プロパティ 2 容量 (Name、Stringe型) ... オーバーヘッド (8 バイト) 、プロパティ名 (4 文字 *2=8 バイト)、プロパティのデータ内容量 (14 文字 *2+4=32 バイト) → 小計 **48** バイト
- プロパティ 3 容量 (Deleteflag、Boolean型) ... オーバーヘッド (8 バイト) 、プロパティ名 (10 文字 *2=20 バイト)、プロパティのデータ内容量 (1 バイト) → 小計 **29** バイト

★上記の総合計... 4+28+16+34+48+29= **159バイト** となります。


### ご注意ください：
個々のエンティティの容量を計算する機能は、標準では組み込まれておりませんので、容量計算を行う際にはスクリプトをお客様にてご作成いただく等のご対応が必要となります。なお、Table に対するあらゆる種類の操作 (読み取り、書き込み、削除を含む) はトランザクションとしてカウントされ、課金の対象となります。

ご参考：[Azure Tables Storage の料金 | Microsoft Azure](https://azure.microsoft.com/ja-jp/pricing/details/storage/tables/)


<br>
<br>

---

<br>
<br>

2024 年 04 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>