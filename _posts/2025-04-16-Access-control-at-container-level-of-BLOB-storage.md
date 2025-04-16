---
title: "BLOB のコンテナーレベルでのアクセス制御"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
    
特定の人に対して、BLOB ストレージアカウントの特定のコンテナーにだけ読み書きできるようにしたいのですが、どうすればいいですか。できれば、アクセス許可を与えたコンテナー以外にどのようなコンテナーがあるか、ということも見えないようにしたい。

# 解説
## 前提
例としてはシンプルにストレージアカウント内に二つの BLOB コンテナー (con1、con2) があり、con1 配下にある BLOB に対してだけ読み書きできる、という前提にします。

![image-5eb1a9f5-1ee3-4c1d-b382-9fa4b42e5dbb.png]({{site.baseurl}}/media/2025/04/image-5eb1a9f5-1ee3-4c1d-b382-9fa4b42e5dbb.png)

## 回答

こちらは以下の資料が参考になります。

[BLOB データにアクセスするための Azure ロールを割り当てる](https://learn.microsoft.com/ja-jp/azure/storage/blobs/assign-azure-role-data-access?tabs=portal)  

この資料にあるように結論としては、以下を割り当てます。
* ストレージアカウントレベルでの **閲覧者**ロール
* コンテナーレベルでの**ストレージ BLOB データ共同作成者**などのデータ アクセス ロール

それぞれ、ロールの割り当てで探すと以下のように出てきます。

![image-37e4d55b-2146-4435-ba9c-171b2f6a776b.png]({{site.baseurl}}/media/2025/04/image-37e4d55b-2146-4435-ba9c-171b2f6a776b.png)

![image-eb72b557-8d55-4d47-8535-d1335ffa0205.png]({{site.baseurl}}/media/2025/04/image-eb72b557-8d55-4d47-8535-d1335ffa0205.png)

上記の二つのロールを割り当てたユーザーから、Azure ポータルで該当のストレージアカウントを参照した例を以下に示します。 
まず、コンテナー自体は両方とも見えます。

![image-5eb1a9f5-1ee3-4c1d-b382-9fa4b42e5dbb.png]({{site.baseurl}}/media/2025/04/image-5eb1a9f5-1ee3-4c1d-b382-9fa4b42e5dbb.png)


そして、con1 についてはストレージ BLOB データ共同作成者のロールがあるため、コンテナー内の BLOB が見えます。また、アップロードや削除等もできます。

![image-cd1d29be-14b9-4ecb-8c57-2c20357b5e62.png]({{site.baseurl}}/media/2025/04/image-cd1d29be-14b9-4ecb-8c57-2c20357b5e62.png)

一方、con2 については con1 と異なり、コンテナーレベルでBLOB データ共同作成者のロールを割り当てていませんので、コンテナー内の BLOB は見えません。以下のようにエラーが表示されます。

![image-5f7b1c72-9ac4-4379-8e9a-96b3a608c8cc.png]({{site.baseurl}}/media/2025/04/image-5f7b1c72-9ac4-4379-8e9a-96b3a608c8cc.png)

ここまでは、正直なところ先述の資料に書いてある通りなのですが、こちらの方法ですとコンテナーの一覧まではストレージアカウントレベルの閲覧者のロールがあることによって見ることはできます。ロールを割り当てていないコンテナーの中身は見えない状況ですが、コンテナーの一覧も該当のユーザーには見せたくない、という場合にできる方法を紹介します。

## 他のコンテナーも見せたくない場合に実施できる方法

Azure ポータルを利用すると、どうしてもストレージアカウントのリソースを表示してからコンテナーをたどる、という手順になり、リソースを表示するために必要な閲覧者ロールの割り当てを消すことはできません。そのため、今回は Storage Explorer を利用します。
Azure Storage Explorer は GUI のツールとなり、BLOB の操作などができます。後述の参考ドキュメントのリンクよりダウンロードすることができます。

そして、実際にやることですが、ストレージアカウントレベルの閲覧者の権限は削除をし、以下の権限だけを該当のユーザーに割り当てます。

* コンテナーレベルでの**ストレージ BLOB データ共同作成者**などのデータ アクセス ロール

そして、該当のユーザーを利用して、Storage Explorer を用いて接続する手順を以下に示します。

(1) Storage Explorer から接続するときに、BLOB コンテナーまたはディレクトリ、を選択します。

![image-d694ca3a-534b-47bc-9842-3d616d2cea21.png]({{site.baseurl}}/media/2025/04/image-d694ca3a-534b-47bc-9842-3d616d2cea21.png)

      
(2) OAuth  を使用してサインインする、を選択し、利用するユーザーとサインインするテナントを指定  

![image-86b1596f-fc7b-430e-81af-82cfbcdc4250.png]({{site.baseurl}}/media/2025/04/image-86b1596f-fc7b-430e-81af-82cfbcdc4250.png)


(3) BLOB  コンテナーまたはディレクトリの URL の欄に、”https://mystorage0123.blob.core.windows.net/con1” というような URL を指定します。  
mystorage0123 の部分はストレージアカウント名、con1 の部分はコンテナー名で書き換えてください。

![image-246ee1ab-f538-4310-802f-d7528b6e092f.png]({{site.baseurl}}/media/2025/04/image-246ee1ab-f538-4310-802f-d7528b6e092f.png)

上記の手順を実施しますと、以下のように対象のコンテナー(今回は con1) の中身だけを表示し、かつ、別のコンテナーの名前が表示される、ということもございません。

![image-2f07977d-5e97-4b50-bf84-4a6597925467.png]({{site.baseurl}}/media/2025/04/image-2f07977d-5e97-4b50-bf84-4a6597925467.png)

閲覧者の権限がないため、接続するストレージアカウント名とコンテナー名は別途該当のユーザーに連携する、という必要がございますが、コンテナーの一覧も見せたくない、というような要件がある場合には、こちらの方法のご利用もご検討いただける内容かと思います。



# 参考ドキュメント
上記で利用している Azure Storage Explorer は以下からダウンロードができます。

[Azure Storage Explorer](https://azure.microsoft.com/ja-jp/products/storage/storage-explorer/?msockid=3e7805faccf567970fad1128cdfa66e6)
    
<br>
<br>
   
---
    
<br>
<br>
    
2025 年 4 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
    
<br>
<br>