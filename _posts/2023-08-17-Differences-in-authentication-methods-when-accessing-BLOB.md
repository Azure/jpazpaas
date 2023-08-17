---
title: "Azure ポータルで BLOB にアクセスするときの認証方法の違いについて"
author_name: "eijimoch"
tags:
    - BLOB Storage
---

# 質問

サブスクリプションレベルでの所有者の権限を持っているユーザーで Azure ポータルから、BLOB のコンテナー一覧からコンテナーを選択したところ、BLOB の一覧が出ず **Azure AD のユーザー アカウントを使用してデータを一覧表示するためのアクセス許可がありません。クリックして Azure AD を使用した認証の詳細をご確認ください** というエラーになり、BLOB にアクセスできません。所有者の権限を持っているのに、なぜ BLOB が見られないのでしょうか。


![image-13eb6b05-5e46-441b-8301-260f0bd3f496.png]({{site.baseurl}}/media/2023/08/image-13eb6b05-5e46-441b-8301-260f0bd3f496.png)

# 回答

## 結論
Azure ポータルで **認証方法** が **Azure AD のユーザー アカウント** になっている場合、所有者の権限のみでは BLOB の一覧やダウンロードをすることはできません。これは、所有者のロールでは BLOB のデータにアクセスする権限がないためです。

一方、 **認証方法** を **アクセス キー**  にすると、こちらは所有者の権限のみで BLOB の一覧を表示できます。所有者の権限があるとストレージ アカウントのアクセス キーを取得する権限があり、アクセス キーが取得できれば BLOB 等のデータにアクセス可能な SAS (Shared Access Signature) の生成が可能になるためです。

次の節で、認証法の違いによるアクセス方法の違いを説明します。

## Azure AD のユーザーアカウント と アクセス キーでのアクセス方法の違いについて

認証方法が異なると、それぞれ以下のような流れで BLOB の一覧を取得する操作を行います。はじめの前提に従い、所有者権限でアクセスを行い、認証方法が **Azure AD のユーザー アカウント** では失敗、**アクセス キー** では成功という例で説明します。


### パターン 1 - Azure AD のユーザーアカウント
ポータルにサインインしているユーザーを用いて、認証情報を HTTP の Authorization ヘッダーに付加してストレージに対して要求を送ります。そして、Storage で該当のユーザーが BLOB データにアクセスできるかどうかを判断して、応答を返します。今回の場合、該当のユーザーに BLOB データにアクセスできる権限がないため、拒否を示す 403 (This request is not authorized to perform this operation using this permission.) 応答が返り、結果として以下のスクリーンショットの左側にあるように BLOB の一覧が見えないことになります。  


![image-ed0b2ff7-4a91-40c6-b77d-a4bf06385852.png]({{site.baseurl}}/media/2023/08/image-ed0b2ff7-4a91-40c6-b77d-a4bf06385852.png)

なお、要求先は以下のように BLOB のエンドポイントになり、List Blobs という REST API を呼び出している状況となります。  
``
https://aaaaa.blob.core.windows.net/container01?restype=container&comp=list&prefix=&delimiter=%2F&marker=&maxresults=30&include=metadata
``

また、403 の応答の HTTP の Body については以下のような XML となります。  
```
<?xml version="1.0" encoding="utf-8"?><Error><Code>AuthorizationPermissionMismatch</Code><Message>This request is not authorized to perform this operation using this permission.
RequestId:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxx000000
Time:2023-08-15T07:05:38.1247485Z</Message></Error>
```


### パターン 2 - アクセス キー
ポータルにサインインしているユーザーの権限でストレージ アカウントのアクセス キーを取得します。そして、その取得したアクセス キーを元に BLOB にアクセスするための SAS を生成し、その SAS を用いて ストレージ アカウントに対して BLOB の一覧を取得する要求を送ります。アクセス キーがあれば、BLOB へのアクセス可能な SAS を生成できるため、アクセスが拒否されることなく、ストレージから BLOB の一覧が応答され、結果として以下のスクリーンショットの左部分のように Azure ポータルで BLOB の一覧が表示されます。


![image-5bc3ad28-d6ec-4763-ab8b-8d5e92c51fdb.png]({{site.baseurl}}/media/2023/08/image-5bc3ad28-d6ec-4763-ab8b-8d5e92c51fdb.png)

要求 URL の部分を見ると、"sv=2022-11-02&ss=bqtf&srt=sco(以下略)" というように、**Azure AD のユーザー アカウント** の場合と比較して、 クエリ文字列に SAS が付加されたうえで、List Blobs の REST API が呼び出されていることが確認できます。以下、一部マスクしましたが、実際の URL となります。

``
https://aaaaa.blob.core.windows.net/container01?restype=container&comp=list&prefix=&delimiter=%2F&marker=&maxresults=30&include=metadata&sv=2022-11-02&ss=bqtf&srt=sco&sp=rwdlacuptfxiy&se=2023-08-15T14:39:13Z&sig=xxx
``

アクセス方法の違いは以上となります。

## 該当のユーザーで認証方法が Azure AD のユーザーアカウント でも BLOB の一覧を表示するためにはどうするか

所有者権限を持つユーザーでも認証方法が Azure AD のユーザーアカウントでも BLOB の一覧表示などの操作ができるようにするには、該当のユーザーに BLOB へアクセスする権限を持ったロールを割り当てる必要があります。BLOB に対して読み書きができる組み込みロールとして、**ストレージ BLOB データ共同作成者** がありますので、こちらの権限をコンテナーレベルもしくはストレージ アカウントレベルで割り当てます。以下、ロールの割り当てを行う画面となります。

![image-d98cbe5f-0bee-4d2f-a442-5c60f4beb946.png]({{site.baseurl}}/media/2023/08/image-d98cbe5f-0bee-4d2f-a442-5c60f4beb946.png)


# まとめ
本ブログでは、Azure ポータルでの BLOB へのアクセス時の認証情報の違いによって、アクセス方法にどのような違いが生じるかを説明いたしました。特に、所有者の権限があると BLOB データへのアクセスでも何でもできる権限がある、と誤解されがちなので、そうではない、というところのご理解のお役に少しでも立ちましたら幸いです。

# 参考情報

以下に参考資料を記載いたします。SAS の詳細な説明や List Blobs の説明などは本文では記載しなかったので、こちらも参考にしていただけますと幸いです。

[Azure portal で BLOB データへのアクセスの承認方法を選択する](https://learn.microsoft.com/ja-jp/azure/storage/blobs/authorize-data-operations-portal)

[Shared Access Signatures (SAS) を使用して Azure Storage リソースへの制限付きアクセスを許可する](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview)

[List Blobs](https://learn.microsoft.com/ja-jp/rest/api/storageservices/list-blobs?tabs=azure-ad)


<br>
<br>

---

<br>
<br>

2023 年 08 月 17 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>