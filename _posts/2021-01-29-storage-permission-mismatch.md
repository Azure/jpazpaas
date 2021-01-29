---
title: "Storage Account へのアクセスで AuthorizationPermissionMismatch エラーが発生した場合の対処方法"
author_name: "Takumi Nagaya"
tags:
    - Storage Account
---

# 質問
Azure CLI で Storage Account の BLOB 一覧を取得しようとしましたが、AuthorizationPermissionMismatch　エラーが発生し取得できません。 <br><br>
デバッグオプションを有効にした際に、以下のようなXMLが返り AuthorizationPermissionMismatch というエラーコードが確認できます。

```xml
<?xml version="1.0" encoding="utf-8"?><Error><Code>AuthorizationPermissionMismatch</Code><Message>This request is not authorized to perform this operation using this permission.
RequestId:47674462-b01e-00a1-4013-f3e7e2000000
Time:2021-01-25T12:15:21.7062505Z</Message></Error>
```

# 回答
Storage Account にリクエストを実行するユーザーに、データアクセスに対して明示的に定義されたロールが割り当てられていないことが原因です。ユーザーに Storage Account へのアクセスに必要な権限を付与する必要があります。<br>
以下のドキュメントに記載がございます。

>
>BLOB とキューの Azure ロール  
(抜粋)  
データ アクセスに対して明示的に定義されたロールによってのみ、セキュリティ プリンシパルによる BLOB データまたはキュー データへのアクセスが許可されます。 所有者、共同作成者、ストレージ アカウント共同作成者 などの組み込みロールでは、セキュリティ プリンシパルによるストレージ アカウントの管理は許可されますが、Azure AD を通じたそのアカウント内の BLOB データまたはキュー データへのアクセスは提供されません。  
https://docs.microsoft.com/ja-jp/azure/storage/common/storage-auth-aad-rbac-portal#azure-roles-for-blobs-and-queues

## エラーの再現方法
1. 特定のサブスクリプションの所有者ロールのみ付与されたユーザーで Azure Portal にログインします。
2. Cloud Shell で Bash を起動し、以下のコマンドを実行します。

    ~~~
        az storage blob list --account-name <storageAccountName> \
        --container-name <continerName> \
        --auth-mode login \
        --debug
    ~~~

3. HTTP ステータスコード 403 が返り、以下のような XML が確認できます。

    ```xml
    <?xml version="1.0" encoding="utf-8"?><Error><Code>AuthorizationPermissionMismatch</Code><Message>This request is not authorized to perform this operation using this permission.
    RequestId:47674462-b01e-00a1-4013-f3e7e2000000
    Time:2021-01-25T12:15:21.7062505Z</Message></Error>
    ```

## 対処方法

1. 実行ユーザーに「ストレージ BLOB 閲覧者」など、処理を実行するために必要な、Storage Account へのデータアクセスを許可するロールを付与します。
   今回は BLOB 一覧の取得に失敗したため、「ストレージ BLOB 閲覧者」のロールを付与します。
   以下の添付画像のように、ロールの割り当てを実行します。

    <img alt="assign-role" src="{{site.baseurl}}/media/2021/01/2021-01-29-assign-role.png" width="70%">

2. 再度 Cloud Shell で Bash を起動し、再現時と同じ以下のコマンドを実行します。
   今度は成功して BLOB の一覧が取得できます。
   
   ~~~
       az storage blob list --account-name <storageAccountName> \
       --container-name <continerName> \
       --auth-mode login \
       --debug
   ~~~

## 関連した質問

### Q. 所有者ロールが付与されているユーザーは Azure Portal で BLOB 一覧を閲覧できました。なぜ CLI では権限エラーが発生するのでしょうか。

A. ロールに Microsoft.Storage/storageAccounts/listKeys/action が含まれている場合、そのロールが割り当てられているユーザーは、アカウント アクセス キーを使って共有キー認証を使用してストレージ アカウントのデータにアクセスできます。

所有者ロールでは、Microsoft.Storage/storageAccounts/listKeys/action が含まれており、アクセスキー認証が使用されるため、Azure Portal で BLOB 一覧を閲覧することができます。  
Azure Portal では以下の画像の通り、認証方法が既定でアクセスキーとなっています。

<img alt="auth-key" src="{{site.baseurl}}/media/2021/01/2021-01-29-auth-key.jpg" width="70%">

Azure Portal で 認証方法を「Azure AD のユーザーアカウント」に切り替えると所有者ロールのみが付与されている場合はエラーが発生します。

<img alt="auth-login" src="{{site.baseurl}}/media/2021/01/2021-01-29-auth-login.jpg" width="70%">

また、エラーの再現方法でご案内したコマンド内の、`--auth-mode` オプションで認証方法を切り替えることができます。<br>
`--auth-mode` の既定値が `key` (アクセスキー認証) のため、所有者ロールであれば、`--auth-mode` を省略すると、権限エラーが発生せず BLOB 一覧を取得することができます。


---

<br>

2021 年 1 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>