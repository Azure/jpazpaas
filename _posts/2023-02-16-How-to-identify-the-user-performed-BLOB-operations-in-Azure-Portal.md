---
title: "Blob Storageにおいて操作ログからユーザーを特定したい"
author_name: "Rui Hiraoka"
tags:
    - Storage
---

# 質問
Azure Portal での Blob Storage のデータ操作について、誰がどのような操作を行ったかを診断設定ログから特定したいです。可能でしょうか。

# 回答
はい。診断設定ログに含まれる RequesterObjectId 又は RequesterUpn などから、
どの Microsoft Entra ユーザーなのかを特定することが可能でございます。

診断設定ログでの確認例  

![image-4cd6a29c-d7a0-4703-8b8c-5538ac57c2fc.png]({{site.baseurl}}/media/2023/02/image-4cd6a29c-d7a0-4703-8b8c-5538ac57c2fc.png)

Azure Portal ではアクセス認証の方式は、以下の 2 種類がございます。

* アクセス キー
* Microsoft Entra ユーザー認証

![image-54451daf-addb-4df9-b5ae-74bd5bea9762.png]({{site.baseurl}}/media/2023/02/image-54451daf-addb-4df9-b5ae-74bd5bea9762.png)

アクセスキーでの認証の場合は Microsoft Entra ユーザーを特定可能な情報を残すことが出来ませんので、監視対象のユーザーには、必ず Microsoft Entra ユーザー認証をさせる必要がございます。




Azure Portal のストレージ アカウントの [ ブレードメニュー ] -> [ 構成 ] から
Azure portal で Microsoft Entra 認可を既定にするを [ 有効 ] にすることで、既定の認証方法設定可能でございます。

![image-61525868-6e4f-4777-9332-ea363db40823.png]({{site.baseurl}}/media/2023/02/image-61525868-6e4f-4777-9332-ea363db40823.png)


Microsoft Entra 認証でストレージ アカウントを利用する Microsoft Entra ユーザーには、別途 BLOB データを操作するための権限が必要となります。
- ストレージ BLOB データ共同作成者など、何らかのデータ アクセス ロール
- 少なくとも Azure Resource Manager 閲覧者ロール

が必要となります。詳細につきましては
[こちら](https://azure.github.io/jpazpaas/2021/01/29/storage-permission-mismatch.html)をご参照ください。

これらの設定に加え、アクセスキーの使用についての制限を加える必要がございます。


# アクセス キー利用の制限

アクセス キーの利用について制限を設けるには以下 2 パターンの設定方法がございます。

### (A). ストレージ アカウント全体でアクセス キーによる認証を完全に無効化し、監視対象ユーザーに BLOB データを操作する権限を RBAC で制御

### (B). アクセス キーは有効化し、一部のユーザ以外はアクセス キーの表示を禁止するように権限を RBAC で制御




それぞれについて記載いたします。



## (A). アクセス キーによる認証を完全に無効化する方法



アクセス キーによる認証を完全に無効化する方法につきましては、
設定変更に関わる権限なども含め、[こちら](https://learn.microsoft.com/ja-jp/azure/storage/common/shared-key-authorization-prevent)に記載がございますので、ご参照ください。

記事内でも解説されておりますが、
Microsoft.Storage/storageAccounts/write などのリソース変更権限を監視対象となるユーザーに与えないようにご注意ください。

## (B). アクセス キーは有効化するものの、特権をもつユーザ以外はアクセス キーを用いたデータ表示を禁止する方法
対象となる Microsoft Entra ユーザーに "Microsoft.Storage/storageAccounts/listKeys/action" 権限を付与しなければ、
アクセスキーを使用してのデータアクセスはできません。

"Microsoft.Storage/storageAccounts/listKeys/action" がないロールでもポータルで見かけ上、アクセスキーに切り替えるボタンは押せますが、実際にはアクセスキーを使用する権限がないのでデータ表示まではできません。  

![image-553240a7-5c93-46e0-9b65-29b00d33d215.png]({{site.baseurl}}/media/2023/02/image-553240a7-5c93-46e0-9b65-29b00d33d215.png)

  
必要なロールとして上述した

- ストレージ BLOB データ閲覧者、ストレージ BLOB データ共同作成者など、何らかのデータ アクセス ロール
- 少なくとも Azure Resource Manager 閲覧者ロール

のロールのみであれば、listKeys / action はできません

# まとめ
Azure Portal では Microsoft Entra ユーザー認証をさせることで、監視対象ユーザーがどのような操作をしたのか、診断設定ログから判断することが可能です。その際、監視対象ユーザーがアクセスキーを利用できないように設定することも必要となります。

また、Azure Portalでのデータ操作とはスコープが異なりますが、監視対象のユーザーに、[ユーザー委任キー](https://learn.microsoft.com/ja-jp/rest/api/storageservices/get-user-delegation-key)の発行権限がある場合、ユーザー委任キーで署名して発行できる[ユーザー委任SAS](https://learn.microsoft.com/ja-jp/rest/api/storageservices/create-user-delegation-sas)を用いて、ユーザー委任キーの発行者以外であってもデータアクセスに利用することができてしまいます。

![image-6168ab72-99fe-4853-bfa0-5200bbf88f91.png]({{site.baseurl}}/media/2023/02/image-6168ab72-99fe-4853-bfa0-5200bbf88f91.png)

Azure Portal外においても厳密な統制を行う場合は、ユーザー委任キー発行に必要な権限Storage/storageAccounts/blobServices/generateUserDelegationKeyを当該ユーザーに与えないようにすることを推奨いたします。

# 変更履歴

2025年5月12日 追記： AzureAD の表示名称が Microsoft Entra に変更されたことに伴い、表示名称および設定画像を修正しました。

<br>
<br>

---

<br>
<br>

2025 年 05 月 12 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>


