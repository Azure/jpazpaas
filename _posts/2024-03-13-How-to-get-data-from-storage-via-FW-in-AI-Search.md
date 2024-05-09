---
title: ファイアウォールを有効にしたストレージアカウントに対して、 Azure AI Search のインデクサーを作成したいがエラーが発生する
author_name: "chansiklee"
tags:
    - AI Search
    - Storage
---

# はじめに
Azure AI Search サービスを作成し、選択されたネットワークからのアクセスを許可しているストレージアカウントからデータを取得しようとする際にエラーが発生する場合がございます。<br>
<br>
Azure Portal では「概要」ページより、「データのインポート」ウィザードでデータに接続する段階で発生する場合があります。
```
データ ソースからのインデックス スキーマの検出でエラーが発生しました: "This request is not authorized to perform this operation."
```

また、インデクサー作成・変更時に以下のエラーが発生する場合があります。
```
インデクサー "xxxxxx-indexer" を更新できませんでした。エラー: "Error with data source: Credentials provided in the connection string are invalid or have expired.
For more information on troubleshooting connection issues to Azure Storage accounts, please see https://go.microsoft.com/fwlink/?linkid=2049388  Please adjust your data source definition in order to proceed."
```

# 原因
Azure AI Search サービスとストレージアカウントが同じリージョンにあり、ストレージアカウントが選択されたネットワークからのアクセスを許可しているためです。

# 回避策
BLOB、Table ストレージそれぞれ設定方法が多少異なりますが、いずれもマネージド ID での認証が必要になります。<br/>
ただし、マネージド ID での認証は Basic レベル以上でのみ利用できるためご注意ください。

>前提条件
>- Basic レベル以上の検索サービス。

[Azure AI Search で マネージド ID 利用時の前提条件](https://learn.microsoft.com/ja-jp/azure/search/search-howto-managed-identities-data-sources?tabs=portal-sys%2Cportal-user#prerequisites)

## 1. Blob ストレージの場合
最初の手順は、以下の 1-1 と 1-2 いずれかの方法をご活用ください。

### 手順1-1. 信頼されたサービスのアクセス許可
下記の内容の通り BLOB ストレージの場合信頼されたサービスのアクセス許可機能を利用することができます。<br>

>注意
>
>Azure AI Search では、信頼できるサービス接続は、Azure Storage 上の BLOB と ADLS Gen2 に制限されます。 Azure Table Storage と Azure File Storage へのインデクサー接続ではサポートされていません。
>
>信頼されたサービス接続では、システム マネージド ID を使用する必要があります。 このシナリオでは、ユーザー割り当てマネージド ID は現在サポートされていません。

[Azure Storage へのインデクサー接続を信頼できるサービスとして作成する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-trusted-service-exception#prerequisites)<br>

下記の画像の通りストレージアカウントのネットワーク設定から、例外 → 「信頼されたサービスの一覧にある Azure サービスがこのストレージ アカウントにアクセスすることを許可します。」を有効にしてください。<br>
この場合、同じサブスクリプション内にある、特定のタイプのすべてのリソースに対してアクセス許可を行います。（例えば、全ての Azure AI Search サービスに対してBLOBストレージのアクセスが可能になります。）<br/>
![image-8f05e8a5-7cc2-4046-a2cc-c842e9e87b48.png]({{site.baseurl}}/media/2024/03/image-8f05e8a5-7cc2-4046-a2cc-c842e9e87b48.png)<br>

### 手順1-2. リソース インスタンスのアクセス許可
同じくストレージアカウントのネットワーク設定でリソース インスタンスにリソースの種類は「Microsoft.Search/searchServices」を、インスタンス名は接続対象の Azure AI Search サービスを指定することで、該当の Azure AI Search サービスのみアクセスを許可することもできます。
![image-7ee01f2c-f65b-4973-8df8-d04696c79b49.png]({{site.baseurl}}/media/2024/03/image-7ee01f2c-f65b-4973-8df8-d04696c79b49.png)<br>

### 手順2. Azure AI Search サービスのマネージド ID を有効にし、RBAC 権限を設定
Azure AI Search サービスのマネージド ID を有効にし、ストレージアカウントに対して適宜権限を付与して頂く必要がございます。<br>
下記の画像の通り Azure AI Search サービスの ID → システム割り当て済みで「状態」を「オン」にする → アクセス許可の Azure ロールの割り当てをクリックしてください。<br>
![image-6af3f275-5940-48c1-af50-78c6cf50e11c.png]({{site.baseurl}}/media/2024/03/image-6af3f275-5940-48c1-af50-78c6cf50e11c.png)<br>

ロールの割り当ての追加（プレビュー）で当該ストレージアカウントに対して「ストレージ BLOB データ共同作成者」役割を追加してください。<br>
RBAC 設定が反映されるまで数分かかる場合がございますので暫くお待ちください。<br>
[アクセス許可を確認する - Storage Blob データ共同作成者](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-trusted-service-exception#check-permissions)<br>
![image-95280a38-c96f-414e-afdd-2e9c40f951af.png]({{site.baseurl}}/media/2024/03/image-95280a38-c96f-414e-afdd-2e9c40f951af.png)<br>


マネージド ID に適切な権限を付与せず、手順3の通り マネージド ID での認証を行う場合、以下のエラーが発生します。
```
"Unable to retrieve account keys for account '{StorageAccountName}' using your managed identity. Ensure the resource ID is correct and that the managed identity for your search service has been granted permission (e.g., Reader and Data Access) to read account keys for this account. See https://go.microsoft.com/fwlink/?linkid=2193521 for detailed documentation of required permissions for your scenario."
```

### 手順3. システム割り当て マネージド ID で認証を行う
データの接続手順で「マネージド ID の認証」項目を「システム割り当て」にすると正常に接続できます。<br>
![image-8fddc3fb-0581-4b73-b8c0-a88878ee214e.png]({{site.baseurl}}/media/2024/03/image-8fddc3fb-0581-4b73-b8c0-a88878ee214e.png)<br>
<br>


## 2. Table ストレージの場合
### 手順1. ストレージのアカウントキーへのアクセスを許可する
現時点では、Table ストレージへアクセスするためには必ずアクセスキーでの認証を利用する必要がございます。<br>
そのため下記の画像の通りストレージアカウントの 構成 → 「ストレージ アカウント キーへのアクセスを許可する」項目を有効にしてください。 <br>
![image-1578852f-28af-42ba-8603-9b5fb98ffffd.png]({{site.baseurl}}/media/2024/03/image-1578852f-28af-42ba-8603-9b5fb98ffffd.png)<br>

上記の項目が無効の場合、データインポート時に以下のエラーが発生します。
```
データ ソースからのインデックス スキーマの検出でエラーが発生しました: "Key based authentication is not permitted on this storage account."
```

### 手順2. Azure AI Search サービスのマネージド ID を有効にし、RBAC 権限を設定
Azure AI Search サービスのマネージド ID を有効にし、ストレージアカウントに対して適宜権限を付与して頂く必要がございます。<br>
下記の画像の通り Azure AI Search サービスの ID → システム割り当て済みで「状態」を「オン」にする → アクセス許可の Azure ロールの割り当てをクリックしてください。<br>
![image-50fe6e67-1d4d-4f36-b55d-158ce6ac0daa.png]({{site.baseurl}}/media/2024/03/image-50fe6e67-1d4d-4f36-b55d-158ce6ac0daa.png)<br>

ロールの割り当ての追加（プレビュー）で当該ストレージアカウントに対して「閲覧者とデータ アクセス」役割を追加してください。<br>
[前提条件 - データおよび閲覧者](https://learn.microsoft.com/ja-jp/azure/search/search-howto-indexing-azure-tables#prerequisites)<br>
RBAC 設定が反映されるまで数分かかる場合がございますので暫くお待ちください。<br>
![image-75088068-e778-40ac-9d0f-6596b9b58298.png]({{site.baseurl}}/media/2024/03/image-75088068-e778-40ac-9d0f-6596b9b58298.png)<br>
マネージド ID に適切な権限を付与せず、手順4の通り マネージド ID での認証を行う場合、BLOB ストレージの手順2と同様なエラーが発生します。

### 手順3. 共有プライベートリンクの設定
現時点でテーブルストレージの場合、BLOB ストレージの手順1の様に信頼サービスやリソースインスタンス許可でのアクセスをサポートしておらず、共有プライベートリンクを経由してアクセスを行う必要がございます。<br>
[共有プライベート リンクを経由した送信接続の作成](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create)<br>

Azure AI Search サービスのネットワーク → 共有プライベート アクセスタブで共有プライベート アクセスの追加を押下し、データ接続先のストレージアカウントを指定して、対象のサブリソースを「table」に指定します。<br>
![image-6fffaf86-f3bd-45d7-abea-54413b5f2fd8.png]({{site.baseurl}}/media/2024/03/image-6fffaf86-f3bd-45d7-abea-54413b5f2fd8.png)<br>

プロビジョニングの状態が「成功」になるまで暫くお待ちください。<br>
![image-14acc54a-9a96-44d4-8bc2-15a44e679b30.png]({{site.baseurl}}/media/2024/03/image-14acc54a-9a96-44d4-8bc2-15a44e679b30.png)<br>

成功になったらストレージアカウントに移動し、ネットワーク → プライベートエンドポイント接続で共有プライベートリンクの接続リクエストを承認します。<br>
![image-7b760f32-e508-4fd3-bac6-0336b0908c3b.png]({{site.baseurl}}/media/2024/03/image-7b760f32-e508-4fd3-bac6-0336b0908c3b.png)<br>

共有プライベートリンクを設定しないでTable ストレージに対してデータのインポート行う場合、以下のエラーが発生します。<br>
```
データ ソースからのインデックス スキーマの検出でエラーが発生しました: "Forbidden"
```

### 手順4. システム割り当て マネージド ID で認証を行う
データの接続手順で「マネージド ID の認証」項目を「システム割り当て」にすると正常に接続できます。<br>
![image-40679222-898c-4620-bc44-c282e525d792.png]({{site.baseurl}}/media/2024/03/image-40679222-898c-4620-bc44-c282e525d792.png)<br>
<br>


以上、ストレージアカウントにファイアウォールがある場合、Azure AI Search サービスからデータを取得する方法について紹介いたしました。

<br>
<br>

---

<br>
<br>

2024 年 03 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>