---
title: "Azure ポータルで BLOB のリストやダウンロードはできるけれど削除ができないときに考えられること"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
Azure 仮想マシン上でブラウザを起動して Azure ポータルを表示しています。また、該当の Azure 仮想マシンが属している仮想ネットワークのサブネットに、今回操作対象のストレージアカウントの blob のプライベート エンドポイントはすでに作成済みです。

この環境で、Azure ポータルで対象のストレージアカウント内のコンテナー配下の BLOB の一覧を出したり、個別の BLOB のアップロードやダウンロード、個別の BLOB の画面まで行って削除はできるのですが、BLOB の一覧でチェックを入れて削除、などをすると以下のように失敗します。考えられることを教えてください。

例えば以下のようにコンテナー con1 の BLOB の一覧は見えています。あわせて、ダウンロードやアップロードもできました。

![image-64e7a0ef-ca22-4471-b3d8-b98e1b98263b.png]({{site.baseurl}}/media/2026/01/image-64e7a0ef-ca22-4471-b3d8-b98e1b98263b.png) 

ただ、一覧からチェックを入れて削除しようとすると 403 エラー (This request is not authorized to perform this operation) となって削除に失敗しました。

![image-3b997c6f-19ba-422d-8348-83d5f89a7bc7.png]({{site.baseurl}}/media/2026/01/image-3b997c6f-19ba-422d-8348-83d5f89a7bc7.png)


# 回答

## 概要と回避方法  

可能性として考えられることは、該当のストレージアカウントにおいて、

- ストレージアカウントのファイアウォールの設定を入れて経路を絞っている
- プライベート エンドポイント経由のアクセスのみが許可されている
- 階層型名前空間が有効になっている
- プライベート エンドポイントについて、blob 向けのプライベート エンドポイントは作成したものの dfs 向けのプライベート エンドポイントは作成していない

といった状況になっていることが疑われます。この場合の問題の解決方法としては、blob 向けのプライベート エンドポイントと同様に dfs 向けのプライベート エンドポイントも作ってください、ということになります。

## 詳細

Azure Blob Storage において Azure ポータル上で BLOB の一覧を出したり、アップロードやダウンロードをするといった操作はブラウザを起動しているクライアントからストレージに対して REST API を呼び出してその操作の結果をブラウザーに反映しているという動作になります。

例えば、Azure ポータルで コンテナー内の BLOB の一覧を取得する場合、以下の ListBlobs という REST API を発行することになります。ListBlobs の場合、REST API を発行する FQDN は **(storagename).blob.core.windows.net** となります。

[BLOB の一覧表示 REST API - Azure Storage](https://learn.microsoft.com/ja-jp/rest/api/storageservices/list-blobs)

そして、ストレージアカウントでファイアウォールの設定をしていて、プライベート エンドポイント経由からのアクセスの許可のみをしている場合、blob 向けのプライベート エンドポイントが存在し、そのプライベート エンドポイントがポータルを実行している Azure 仮想マシンからアクセス可能な場合、上記の ListBlobs は成功することになります。

その一方で、Azure ポータルで BLOB の一覧でチェックを入れて複数の BLOB をまとめて削除、という操作をすると、階層型名前空間が無効なストレージアカウントでは以下の Delete BLOB という REST APIを実行します。

[BLOB の削除 REST API - Azure Storage](https://learn.microsoft.com/ja-jp/rest/api/storageservices/delete-blob)

上記の DeleteBlob という REST API も REST API を発行する FQDN は ListBlobs と同様、**(storagename).blob.core.windows.net** となります。この場合、ListBlobs の時と同様に、プライベート エンドポイントへのアクセスになるので、経路としては許可された経路になります。

ただ、階層型名前空間が有効な環境でAzure ポータルで BLOB の一覧でチェックを入れて複数の BLOB をまとめて削除、という操作をすると、REST API は上記の DeleteBlob とは異なり、Path - Delete の REST API を利用します。

[Path - Delete](https://learn.microsoft.com/ja-jp/rest/api/storageservices/datalakestoragegen2/path/delete)

この Path - Delete という REST API の場合、REST API を発行する FQDN は **(storagename).dfs.core.windows.net** となります。ここでもし dfs 向けのプライベート エンドポイントを作成していないと、ポータルからは (storagename).dfs.core.windows.net のエンドポイントへのアクセスはパブリックの IP になり、許可していない経路となるため、403 応答を受け取ることになります。

以下、Azure ポータルで、一覧から BLOB を選択して削除しに行った時のブラウザートレースを取得した画面となりますが、(storagename).dfs.core.windows.net のエンドポイントに HTTP の DELETE を発行して 403 になったことが確認できます。

![image-35cea72b-9bc6-4340-a4f2-ccba35f179a9.png]({{site.baseurl}}/media/2026/01/image-35cea72b-9bc6-4340-a4f2-ccba35f179a9.png)

このような挙動になるため、階層型名前空間が有効な場合、blob 向けのプライベート エンドポイントだけでなく、dfs 向けのプライベート エンドポイントも作成する必要がある、ということなります。この点は以下の資料にも記載がございます。

[プライベート エンドポイントを使用する - Azure Storage](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints)

以下、上記資料よりの引用となります。

`
Data Lake Storage ストレージ リソース用にプライベート エンドポイントを作成する場合は、Blob Storage リソース用にも作成する必要があります。 これは、Data Lake Storage エンドポイントをターゲットとする操作が、BLOB エンドポイントにリダイレクトされる可能性があるためです。 同様に、Blob Storage 用のみにプライベート エンドポイントを追加し、Data Lake Storage 用に追加しない場合、API には DFS プライベート エンドポイントが必要であるため、一部の操作 (ACL の管理、ディレクトリの作成、ディレクトリの削除など) が失敗します。 両方のリソースにプライベート エンドポイントを作成して、すべての操作が正常に完了できるようにします。
`

なお、今回は一覧でチェックを入れての削除、というパターンを例に取りましたが、ACL の設定など階層型名前空間を利用しているとき特有の操作が失敗する、というような見え方もするシナリオとなります。

# まとめ
コンテナー内の BLOB の一覧は出せるけれど削除ができない、というような見え方になるため、経路の問題というより、アクセスしているユーザーの権限、と考えがちな内容になります。ただ、階層型名前空間を利用しているストレージアカウントの場合においては、以下のことを気にしてみるとご自身でもトラブルシューティングができるかと思います。

- ストレージ アカウントのファイアウォールの設定でアクセスの経路を絞っているかどうか
- Azure ポータルを使用している環境からストレージ アカウントへの経路は、プライベート エンドポイントを利用する環境になるか
- プライベート エンドポイントを利用する環境の場合、blob、dfs 両方のプライベート エンドポイントを作成しているか
- プライベート エンドポイントを利用している場合、ポータルを利用しているクライアントの環境から、blob、dfs 両方に対してプライベート エンドポイントの IP に名前解決できているか

一部の操作が失敗、という挙動から権限周りの確認はしていただくものの、ストレージのファイアウォールまわりやプライベート エンドポイントの構成などを確認する、というところに視点が向かないということはそれなりに起こりうるシナリオかと思います。もちろんこのシナリオも一例に過ぎないので、似たような状況だけれどこのシナリオには完全に合致しない、ということもあるかとは思いますが、こういうシナリオもある、ということで参考になれば幸いです。

# 参考資料
[Azure Storage のファイアウォール規則 | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security)  

[プライベート エンドポイントを使用する - Azure Storage | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints)
<br>
<br>

---

<br>
<br>

2026 年 01 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>