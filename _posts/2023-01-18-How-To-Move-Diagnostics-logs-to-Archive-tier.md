---
title: "診断設定のログ (追加 BLOB) をアーカイブ層に移動する方法"
author_name: "hamatsum"
tags:
    - Storage Account
    - Storage
---

# 質問
- 保管コスト削減のため、診断設定のログをアーカイブ層に移動したいのですが、移動できません。
<br>
<br>

# 回答
## 前提
Azure Monitor の機能である診断設定によって収集されるログは、出力の宛先の一つとしてストレージ アカウントに出力いただくことが可能です。その際、出力されるログの BLOB は「追加 BLOB」となります。

Azure Monitor リソース ログの形式変更のための準備 - Azure Monitor | Microsoft Learn<br>
[https://learn.microsoft.com/ja-JP/azure/azure-monitor/essentials/resource-logs-blob-format](https://learn.microsoft.com/ja-JP/azure/azure-monitor/essentials/resource-logs-blob-format)<br>
> この新しい形式では、Azure Monitor で**追加 BLOB** を使用してログ ファイルをプッシュすることができ、継続的に新しいイベント データを追加する場合により効率的です。

しかしながら、現在、BLOB ストレージにて層の変更はブロック BLOB にのみ対応しており、追加 BLOB を直接アーカイブ層に移動いただくことができません。また、BLOB の種類は作成時にのみ指定が可能であり、後から変更もできません。

BLOB データのホット、クールおよびアーカイブ アクセス層 - Azure Storage | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-overview](https://learn.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-overview)<br>
> **アクセス層の設定はブロック BLOB でのみ許可されています。** これらは、追加 BLOB とページ BLOB ではサポートされていません。

そこで、コスト削減を目的とした代替案について、以下にご紹介させていただきます。

## 代替案
考え得る代替案について、主に以下の 2 通りの方法が考えられます。

1. ストレージ アカウントの既定のアクセス層を「クール」に変更する。[↓](#default-tier)<br>
2. 何らかの手段によってストレージ アカウント内の別コンテナーに「ブロック BLOB」に変換してコピーし、アーカイブ層に移動する。[↓](#copy-as-blockblob)<br>

<a id="default-tier"></a>
### 1. ストレージ アカウントの既定のアクセス層を「クール」に変更する。
追加 BLOB の層は、以下にて設定されているストレージ アカウントの既定のアクセス層に従い決定されます。そのため、ストレージ アカウントの既定のアクセス層を「クール」に設定いただくことで、容量当たりのコストがより安価な層をご利用いただくことが可能です。ただし、既定のアクセス層はオンライン層のみが指定可能なため、オフライン層であるアーカイブ層はご利用いただけない点ご留意ください。

BLOB データのホット、クールおよびアーカイブ アクセス層 - Azure Storage # アカウントのデフォルト アクセス層の設定 |  Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-overview#default-account-access-tier-setting](https://learn.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-overview#default-account-access-tier-setting)<br>
![image-ef5a4e57-5bf4-486d-a874-3d739a0ff39f.png]({{site.baseurl}}/media/2023/01/image-ef5a4e57-5bf4-486d-a874-3d739a0ff39f.png)<br>

> **※注意**<br>
上記デフォルト アクセス層の設定のドキュメント内にて言及があるように、ストレージ アカウントのデフォルト アクセス層の設定を変更すると、ストレージ アカウント内のアクセス層を明示的に設定していないすべての BLOB に層の変更が適用されます。そのため、汎用 v2 アカウントで既定のアクセス層の設定をホットからクールに切り替えると、そのアクセス層が推定されるすべての BLOB に対する書き込み操作 (10,000 件ごと) に料金が発生する点、ご留意ください。<br>たとえば、明示的に層を設定していない BLOB が 10,000 件存在するストレージ アカウントにて、ホットからクールへ既定のアクセス層を変更した場合、10,000 件の SetBlobTier リクエストが発生いたします。<br>

> **※注意 2**<br>
既定のアクセス層の設定はストレージ アカウント単位となっており、コンテナごとには設定できません。そのため、ストレージ アカウント内の別コンテナーの利用で、ホット層へのアップロードと混在したい際には、アップロード時にアクセス層を明示的にご指定いただくなどの運用でカバーいただく必要がございます。<br>
<br> 
BLOB のアクセス層を設定する - Azure Storage # アップロード時に BLOB の層を設定する |  Microsoft Docs<br>
[https://docs.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-online-manage?tabs=azure-portal#set-a-blobs-tier-on-upload](https://docs.microsoft.com/ja-jp/azure/storage/blobs/access-tiers-online-manage?tabs=azure-portal#set-a-blobs-tier-on-upload)<br>
<br> 
明示的なアクセス層の指定が煩わしい場合や、ホット層へアップロードする対象が追加 BLOB の場合には、診断設定のログ保管用のストレージ アカウントを別途用意し、運用したい層毎にストレージアカウント単位で分けてご利用いただくことをご検討ください。

<br>

<a id="copy-as-blockblob"></a>
### 2. 何らかの手段によってストレージ アカウント内の別コンテナーに「ブロック BLOB」に変換してコピーし、アーカイブ層に移動する。
前提にてお話しの通り、BLOB の種類はあとから変更ができないため、診断設定により出力された「追加 BLOB」そのものを「ブロック BLOB」に変更してアーカイブ層に移動することは叶いません。一方で、BLOB をコピーする際は新規に BLOB を作成することと同義ですので、何らかの手段で

・ブロック BLOB として別コンテナ―にコピーを作成<br>
・作成したコピーをアーカイブ層に保存<br>
・保存完了後、元の追加 BLOB を削除<br>

という処理工程を実装いただくことで、アーカイブ層にブロック BLOB のログを保管いただくという運用が実現可能
です。<br>

「何らかの手段」の部分は SDK / REST API 等を利用し、プログラムを実装いただくことももちろん可能ですが、変換のためのパラメーターなどが準備されており、比較的簡単に実現いただける AzCopy / Azure PowerShell / Azure CLI のコマンド、ならびに、ノン コーディングで実現可能なサービスをそれぞれ紹介させていただきます。

#### ・AzCopy のコマンド

以下コマンドにて、コピー元の追加 BLOB を、ブロック BLOB としてコピーし、さらに、コピー先の BLOB の層としてアーカイブ層を指定可能です。このようなコマンドをもとに、スクリプトを実装し、スケジュール実行が可能なバッチ処理などで日次などで実行いただくことが考えられます。

```
azcopy copy "https://<storage-account-name>.blob.core.windows.net/<source-container-name>/<blob-path><SAS-token>" "https://<storage-account-name>.blob.core.windows.net/<destination-container-name><SAS-token>" --blob-type=BlockBlob --block-blob-tier=Archive --recursive
```


本例では、`https://<storage-account-name>.blob.core.windows.net/<source-container-name>/<blob-path><SAS-token>` がコピー元のファイルパス（もともとの追加 BLOB のパス）、`https://<storage-account-name>.blob.core.windows.net/<destination-container-name><SAS-token>` がコピー先のファイルパス（アーカイブ層として保存するブロック BLOB のパス）となります。`<blob-path>` 部分には対象のディレクトリなどを記載いただければと存じます。

また、BLOB の種類とアクセス層を変更するために使用したオプションは次の通りです。

azcopy copy | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/storage/common/storage-ref-azcopy-copy](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-ref-azcopy-copy)<br>
>・--blob-type=[BlockBlob|PageBlob|AppendBlob]<br>
ブロック、ページ、または追加 BLOB として BLOB をコピーします。<br>
・--block-blob-tier=[None|Hot|Cool|Archive]<br>
特定のアクセス層 (アーカイブ層など) にコピーします。<br>
・--recursive<br>
対象のディレクトリ配下を再帰的にコピーします。<br>


AzCopy v10 の基本的なご利用方法等に関しましては、下記ドキュメントなどがございますので必要に応じてご活用くださいませ。

- AzCopy v10 を使用して Azure Storage にデータをコピーまたは移動する | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/storage/common/storage-use-azcopy-v10](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-use-azcopy-v10)<br>
- AzCopy v10 を使用して Azure ストレージ アカウント間で BLOB をコピーする | Microsoft Docs<br>
[https://docs.microsoft.com/ja-jp/azure/storage/common/storage-use-azcopy-blobs-copy?toc=/azure/storage/blobs/toc.json](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-use-azcopy-blobs-copy?toc=/azure/storage/blobs/toc.json)<br>
<br>

#### ・Azure PowerShell のコマンド

以下コマンドにて、コピー元の追加 BLOB を、ブロック BLOB としてコピーし、さらに、コピー先の BLOB の層としてアーカイブ層を指定可能です。このようなコマンドをもとに、ループ処理などで対象の BLOB 群のコピーを実現するスクリプトを実装し、スケジュール実行が可能なバッチ処理などで日次などで実行いただくことが考えられます。

```
$context = New-AzStorageContext -StorageAccountName $storageAccountName -UseConnectedAccount
Copy-AzStorageBlob -SrcContainer $sourceContainerName -SrcBlob $sourceBlobName -SourceContext $context -DestContainer $destContainer -DestBlob $destBlobName -DestBlobType "Block" -DestContext $context -StandardBlobTier "Archive"
```

BLOB の種類とアクセス層を変更するために使用したオプションは次の通りです。

Copy-AzStorageBlob (Az.Storage) | Microsoft Learn<br>
[https://learn.microsoft.com/en-us/powershell/module/az.storage/copy-azstorageblob?view=azps-9.3.0](https://learn.microsoft.com/en-us/powershell/module/az.storage/copy-azstorageblob?view=azps-9.3.0)<br>
>・-DestBlobType [Block|Page|Append]<br>
ブロック、ページ、または追加 BLOB として BLOB をコピーします。<br>
・-StandardBlobTier [Hot|Cool|Archive]<br>
特定のアクセス層 (アーカイブ層など) にコピーします。<br>


> **※注意**<br>
BLOB の種類を変換する `-DestBlobType` オプションは、Az モジュールの 9.1.0 にて新たに追加されたオプションのため、古いバージョンをご利用のお客様はバージョン 9.1.0 以降にアップデートの上、ご利用ください。<br>
<br>
Azure PowerShell release notes |  Microsoft Learn<br>
[https://learn.microsoft.com/en-us/powershell/azure/release-notes-azureps?view=azps-9.3.0#azstorage-2](https://learn.microsoft.com/en-us/powershell/azure/release-notes-azureps?view=azps-9.3.0#azstorage-2)
<br>

#### ・Azure CLI のコマンド

以下コマンドにて、コピー元の追加 BLOB を、ブロック BLOB としてコピーし、さらに、コピー先の BLOB の層としてアーカイブ層を指定可能です。このようなコマンドをもとに、ループ処理などで対象の BLOB 群のコピーを実現するスクリプトを実装し、スケジュール実行が可能なバッチ処理などで日次などで実行いただくことが考えられます。

```
az storage blob copy start --account-name $accountName --destination-blob $destBlobName --destination-container $destcontainerName --destination-blob-type BlockBlob --source-blob $srcblobName --source-container $containerName --tier Archive
```

BLOB の種類とアクセス層を変更するために使用したオプションは次の通りです。

az storage blob copy # az storage blob copy start |  Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/cli/azure/storage/blob/copy?view=azure-cli-latest#az-storage-blob-copy-start](https://learn.microsoft.com/ja-jp/cli/azure/storage/blob/copy?view=azure-cli-latest#az-storage-blob-copy-start)<br>
>・--destination-blob-type [Detect|BlockBlob|PageBlob|AppendBlob]<br>
ブロック、ページ、または追加 BLOB として BLOB をコピーします。<br>
・--tier [Hot|Cool|Archive]<br>
特定のアクセス層 (アーカイブ層など) にコピーします。<br>


> **※注意**<br>
上記オプションは、バージョン 2.44.0 にて用意されたオプションのため、古いバージョンをご利用のお客様はバージョン 2.44.0 以降にアップデートの上、ご利用ください。<br>
<br>
Release notes & updates – Azure CLI |  Microsoft Learn<br>
[https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli?toc=%2Fcli%2Fazure%2Ftoc.json&bc=%2Fcli%2Fazure%2Fbreadcrumb%2Ftoc.json&view=azure-cli-latest#storage](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli?toc=%2Fcli%2Fazure%2Ftoc.json&bc=%2Fcli%2Fazure%2Fbreadcrumb%2Ftoc.json&view=azure-cli-latest#storage)
<br>

#### ・ノン コーディングで実現可能なサービス

Azure Data Factory というサービスでは、ノン コーディングで BLOB のコピー処理が実現できます。<br>
本サービスで利用可能な、BLOB ストレージのコピー アクティビティは、コピー元の BLOB の種類に依らず、すべてブロック BLOB へのコピーとなります。
そのため、このコピー アクティビティを利用して、コンテナー間で追加 BLOB からブロック BLOB へ変換してBLOB のコピーが可能です。また、スケジュール実行のトリガーを利用することで、定期的な自動実行へも対応できます。

ただ、本サービス単体では、層の移動には対応していないため、別途変換先のブロック BLOB のコンテナーに対して、ストレージの機能であるライフサイクル管理を利用し、変換後のブロック BLOB のアーカイブ層への移動を構成いただくことをお勧めいたします。

Azure Data Factory の基本的な内容ならびにコピー アクティビティの利用について、下記にドキュメントをご紹介いたしますので、あわせてご参考となれば幸いです。

- Azure Data Factory の概要 - Azure Data Factory | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/data-factory/introduction](https://learn.microsoft.com/ja-jp/azure/data-factory/introduction)
- 最初のデータ ファクトリ パイプラインの開始と試用 - Azure Data Factory | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/data-factory/quickstart-get-started](https://learn.microsoft.com/ja-jp/azure/data-factory/quickstart-get-started)

- データのコピー ツールを使用してデータをコピーする - Azure Data Factory | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/data-factory/quickstart-hello-world-copy-data-tool](https://learn.microsoft.com/ja-jp/azure/data-factory/quickstart-hello-world-copy-data-tool)
- Azure Blob Storage のデータをコピーし変換する - Azure Data Factory & Azure Synapse | Microsoft Learn<br>
[https://learn.microsoft.com/ja-jp/azure/data-factory/connector-azure-blob-storage?tabs=data-factory](https://learn.microsoft.com/ja-jp/azure/data-factory/connector-azure-blob-storage?tabs=data-factory)


<br>

## 付録
各シナリオ毎の料金の計算結果をご紹介いたします。なお、<strong><u>下記条件はあくまで計算を簡易に行うためのサンプルであり、実際のログ出力の状況と合致しない場合がございます。</u></strong>また、書き込み回数や容量の条件によって、どのシナリオが優位となるかについては変動する場合がございます。<br>
あくまで計算のサンプルとご認識いただきつつ、実際にご利用の環境で発生しているログ容量および書き込み回数を参考に、お客様自身にて試算をいただきますようお願い申し上げます。

> ・ストレージ アカウントのリージョン : 東日本<br>
・ファイル構造 : フラットな名前空間<br>
・ストレージ アカウントの冗長性 : LRS <br><br>
・ 1 件のログ ファイル容量 : 10MB<br>
・ 1 件のログ ファイルに発生する書き込み回数 : 100 回<br>
・ 1 日に作成されるログ ファイルの件数 : 100 件<br>
→ <br>
・1 日毎に増えるログ容量: 10MB * 100 件 = 1GB<br>
・1 日に発生するログへの書き込み回数 : 100 回 * 100 件 = 10,000 回

### ・シナリオ 1 : ホット層で 30 日保管した場合
・ログの書き込み操作で発生する料金 : &yen;6.7113 / 10,000 単位 * 10,000 回 * 30 日 =  &yen;201.339 

・ひと月 (30 日) あたりの平均利用量 : (1 GB * 1 日 + 2 GB * 1 日 + ... + 29 GB * 1 日 + 30 GB * 1 日) / 30 日 = 15.5 GB

・容量にかかる料金 : &yen;2.6846 * 15.5 GB = &yen;41.6113

→ 計 : &yen;242.9503

### ・シナリオ 2 : 既定値をクールに変更した場合
・ログの書き込み操作で発生する料金 : &yen;13.4226  / 10,000 単位 * 10,000 回 * 30 日 =  &yen;402.678

・ひと月 (30 日) あたりの平均利用量 : (1 GB * 1 日 + 2 GB * 1 日 + ... + 29 GB * 1 日 + 30 GB * 1 日) / 30 日 = 15.5 GB

・容量にかかる料金 : &yen;1.47648 * 15.5 GB = &yen;22.88544

→ 計 : &yen;425.56344

### ・シナリオ 3 : 既定値をホットでホット層に書き込みしつつ、3 日後にアーカイブ層にブロック BLOB としてコピーし、元の BLOB を削除する場合
※純粋に Blob ストレージの観点で発生する料金のみの試算であり、コピーを実施するにあたりスクリプトや Azure Data Factory を実行するための料金は含まれません。

・ログの書き込み操作で発生する料金 : &yen;6.7113  / 10,000 単位 * 10,000 回 * 30 日 =  &yen;201.339 

・元のログの削除操作で発生する料金 : 無料

・ログのコピー (書き込み) 操作にかかる料金 : &yen;15.3285 / 10,000 単位 * (100 件 * 2 操作(PutBlock & PutBlockList)) 回 * 27 日 = &yen;8.27739

・ログのコピー (読み取り) 操作にかかる料金 : &yen;0.5370 / 10,000 単位 * (100 件 * 1 操作(GetBlob)) 回 * 27 日 = &yen;0.14499
<br><br>

・ひと月 (30 日) あたりのホット層平均利用量 : (1 GB * 1 日 + 2 GB * 2 日 + 3 GB * 28 日) / 30 日 = 2.9 GB

・ひと月 (30 日) あたりのアーカイブ層平均利用量 : (0 GB + 3 日 + 1 GB * 1 日 + 2 GB * 1 日 + ... + 26 GB * 1 日 + 27 GB * 1 日) / 30 日 = 12.6 GB

・容量にかかる料金 : &yen;2.6846 * 2.9 GB + &yen;0.26846 * 12.6 GB = &yen;11.167936

→ 計 : &yen;220.929316

<br>

**＜参照情報＞**<br>
- ストレージ アカウントの課金について | Japan Azure 課金 サブスクリプション サポート ブログ<br>
[https://jpazasms.github.io/blog/AzureSubscriptionManagement/20190226c/](https://jpazasms.github.io/blog/AzureSubscriptionManagement/20190226c/)

- Azure Storage Blob の価格 | Microsoft Azure<br>
[https://azure.microsoft.com/ja-jp/pricing/details/storage/blobs/](https://azure.microsoft.com/ja-jp/pricing/details/storage/blobs/)
※ 2023/01/16 時点の価格情報
![2023-01-16_18h09_03-9c08687e-6bc1-4308-bbcc-ec9ab4898e14.jpg]({{site.baseurl}}/media/2023/01/2023-01-16_18h09_03-9c08687e-6bc1-4308-bbcc-ec9ab4898e14.jpg)
![2023-01-16_18h09_18-b7261a1c-95db-4079-9a32-cdc93fa451f6.jpg]({{site.baseurl}}/media/2023/01/2023-01-16_18h09_18-b7261a1c-95db-4079-9a32-cdc93fa451f6.jpg)
<br>
<br>

---

<br>

2023 年 01 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>