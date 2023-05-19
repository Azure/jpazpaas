---
title: "2023 年 5 月頃に送付された Azure Storage のセキュリティ設定の変更に関する通知(_LVK-RT8)について"
author_name: "hishiono"
tags:
    - Storage
---

# 質問
"Important notice: Default security settings for new Azure Storage accounts will be updated" (_LVK-RT8)<br>
というタイトルの通知が来ました。
- この通知の内容はなんですか？
- 既存のストレージアカウントも影響を受けますか？
- 何か対応をする必要がありますか？

# 回答
最もお問い合わせを頂いている内容から記載させて頂きます。<br>
## <概要>
- 2023 年 8 月以降、**新規に作成するストレージ**に関して、<br>
**「（アカウントレベルの）パブリックアクセスを許可する」** と 
**「クロステナントレプリケーションを許可する」** という設定の初期値が変更されるという内容です。 <br>

- **既存のストレージアカウントの設定は変更されません。** <br>
また、**変更ができなくなるということではございません。**<br>
これまで初期値が "有効" として作成されていた項目が、新規に作られるストレージアカウントでは "無効" で作成されるようになる、というだけであり、<br>
**必要に応じて変更頂くことが可能です。** 

- 上述の通り、既存のストレージアカウントには影響がないことから、ほとんどのユーザーにおいて **特に何か対応する必要がない** ものと判断しています。<br>
2023 年 8 月以降、新規にストレージアカウントを作成された際は、<br>
「アカウントレベルの匿名パブリックアクセス」と 「クロステナントレプリケーションを許可する」を有効として利用されたい場合、<br>
設定を変更する必要がある点だけご注意下さい。

## <詳細>
今回の通知は Azure Storage をご利用しているユーザーにお送りしています。 <br>

### Blobパブリックアクセスについて
まず、ストレージアカウントには "Blob パブリックアクセスを許可する" という設定がございます。 <br>
これはストレージ内の Blob ファイルの、匿名アクセス（認証を利用せずアクセスすること）に関係する設定です。 <br>
より詳細は下記の資料をご確認下さい。 
 
コンテナーと BLOB の匿名パブリック読み取りアクセスを構成する<br>
https://learn.microsoft.com/ja-jp/azure/storage/blobs/anonymous-read-access-configure?tabs=portal
 
これまでは、この設定について、初期値が "有効" となっておりましたが、 <br>
2023 年 8 月より、新規に作成されるストレージアカウントに対して、この設定の初期値を "無効" に変更します。 <br>
既存のストレージアカウントの設定は変更されません。<br>
<br>
現状の設定は Azure Portal から簡単にご確認頂けます。 <br>
ストレージアカウントの管理画面から "構成" を開いて、"BLOB パブリックアクセスを許可する" を確認してください。 

![image-df1dff38-ce80-4081-ae2c-e846e9f17c51.png]({{site.baseurl}}/media/2023/05/image-df1dff38-ce80-4081-ae2c-e846e9f17c51.png)

コマンドから確認する場合は、 Get-AzStorageAccount 等を利用して、 AllowBlobPublicAccess を確認します。 <br>
詳細は下記の資料をご確認下さい。 <br>
<br>
Get-AzStorageAccount<br>
https://learn.microsoft.com/ja-jp/powershell/module/az.storage/Get-azStorageAccount?view=azps-8.3.0

### クロステナントレプリケーションについて
また、クロステナントレプリケーションという設定がございます。 <br>
オブジェクトレプリケーションという機能があり、これを別の Azure AD テナント上のストレージと実施するための設定です。 <br>
より詳細は下記の資料をご確認下さい。 <br>

Azure Active Directory テナント間でのオブジェクト レプリケーションを禁止する<br>
https://learn.microsoft.com/ja-JP/azure/storage/blobs/object-replication-prevent-cross-tenant-policies?tabs=portal

 
これまでは、この設定について、初期値が "有効" となっておりましたが、 <br>
2023 年 8 月より、新規に作成されるストレージアカウントに対して、この設定の初期値を "無効" に変更します。 <br>
既存のストレージアカウントの設定は変更されません。<br>
現状の設定は Azure Portal から簡単にご確認頂けます。 

ストレージアカウントの管理画面より、"オブジェクトレプリケーション" を開いて、上部の "詳細設定" を確認してください。 

![image-cd02a7a1-0bd1-4891-852b-1eb71d0863ac.png]({{site.baseurl}}/media/2023/05/image-cd02a7a1-0bd1-4891-852b-1eb71d0863ac.png)

コマンドから確認する場合は、Get-AzStorageAccount 等を利用して、AllowCrossTenantReplication を確認します。 <br>
詳細は下記の資料をご確認下さい。 <br>
<br>
Get-AzStorageAccount<br>
https://learn.microsoft.com/ja-jp/powershell/module/az.storage/Get-azStorageAccount?view=azps-8.3.0

## <補足>
通知の中に "EnableAnonymousAccessForNewStorageAccounts" という機能について紹介がございます。<br>
サブスクリプション レベルで「EnableAnonymousAccessForNewStorageAccounts」を登録すれば、<br>
現状と同じく初期値が "有効" の状態で作成されます。<br>
<br>
Azure Portal よりサブスクリプションの管理画面を開いて、<br>
プレビュー機能 ⇒ EnableAnonymousAccessForNewStorageAccounts を検索 <br>
と実施頂くことで登録可能です。 <br>

![image-79e30737-9910-4f2d-8dca-a3eef2037ab7.png]({{site.baseurl}}/media/2023/05/image-79e30737-9910-4f2d-8dca-a3eef2037ab7.png)


# 参考ドキュメント
- コンテナーと BLOB の匿名パブリック読み取りアクセスを構成する<br>
https://learn.microsoft.com/ja-jp/azure/storage/blobs/anonymous-read-access-configure?tabs=portal 
- Azure Active Directory テナント間でのオブジェクト レプリケーションを禁止する<br>
https://learn.microsoft.com/ja-JP/azure/storage/blobs/object-replication-prevent-cross-tenant-policies?tabs=portal
- Get-AzStorageAccount<br>
https://learn.microsoft.com/ja-jp/powershell/module/az.storage/Get-azStorageAccount?view=azps-8.3.0
- Azure サブスクリプションでプレビュー機能を設定する<br>
https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/preview-features?tabs=azure-portal

<br>
<br>

---

<br>
<br>

2023 年 5 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>