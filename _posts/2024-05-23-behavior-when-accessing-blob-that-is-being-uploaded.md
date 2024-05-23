---
title: "アップロード中の BLOB にアクセスした時の挙動"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問

大きなサイズの BLOB をアップロード中に、別のクライアントからそのアップロード中の BLOB にアクセスしたら、アップロード途中の BLOB のダウンロードができますか。例えば、1GB のファイルをアップロードしていて、500MB アップロードできている状況の場合、500MB だけダウンロードできたりするのでしょうか。

# 回答
基本的にはアップロードが途中の BLOB にアクセスしても、途中までのデータが取得できることはありません。以下、2つのシナリオを元に、どのようになるかを説明します。

## シナリオ 1、標準の BLOB のエンドポイントを利用したアップロード

以下のような azcopy で <ストレージアカウント名>.blob.core.windows.net という FQDN を指定して 1GB のファイルのアップロードを行います。

```
azcopy.exe copy test1gb.dat 'https://storage1.blob.core.windows.net/con1/test1gb.dat?<SAS Token>
```

この場合、REST API としては、アップロードする内容を数 MB 程度のブロックとして PutBlock を必要な回数呼び出してファイルの内容をアップロード、最後に、PutBlockList を呼び出して、アップロード完了とする、という動作となります。
この状況の中、アップロード中に別のクライアントが同じ BLOB に対して GetBlobProperties など、BLOB の取得をしようとしても、PultBlockList が呼び出される前では BLOB が出来上がっていないため 404 Not Found の応答となります。それぞれの REST API の説明は以下となります。

[Put Block](https://learn.microsoft.com/ja-jp/rest/api/storageservices/put-block)

[Put Block List](https://learn.microsoft.com/ja-jp/rest/api/storageservices/put-block-list)



## シナリオ 2、階層型名前空間利用時に ADLS Gen2 のエンドポイントを利用したアップロード

シナリオ 1. からの変更点としては、FQDN が .blob.core.windows.net から、.dfs.core.windows.net になっている点となります。階層型名前空間を有効にしておりますので、ADLS Gen2 用の REST API がご利用いただける、ということになります。

```
azcopy.exe copy test1gb.dat 'https://storage1.dfs.core.windows.net/con1/test1gb.dat?<SAS Token>
```


この場合、REST API としては、以下のような流れとなります

1. CreatePathFile をして、ストレージ側に 0 バイトのファイルを作成
2. ファイルを数 MB 程度のブロックにしたうえで AppendFile を複数回呼び出してファイルの内容をストレージにアップロード
3. 最後にFlushFile を呼び出してアップロード完了

このパターンにおいては、CreatePathFile によって 0 バイトのファイルができたタイミングで GetBlobProperties を呼び出したら、200 応答となります。ただし、まだファイルは出来上がってないため、上記 2. の AppendFile が何度か呼び出されて、クライアントからある程度のファイルの内容が送られていた段階だとしても、Content-Length ヘッダーは 0 のまま変わりません。そして、FlushFile が終わった後に GetBlobProperties を呼び出すと、200 応答となり、かつ Content-Length ヘッダーはアップロードしたファイルのサイズとなります。それぞれの REST API の説明は以下となります。


[Path - Create](https://learn.microsoft.com/ja-jp/rest/api/storageservices/datalakestoragegen2/path/create)  
CreatePathFile はこちらの REST になります。

[Path - Update](https://learn.microsoft.com/ja-jp/rest/api/storageservices/datalakestoragegen2/path/update)  
AppendFile と FlushFile はいずれもこちらの REST API となり、"action" URI パラメーターが "appned" の場合は AppendFile、"flush" の場合は FlushFile となる、という動きになります。

### 補足

なお、階層型名前空間を利用しているストレージアカウントだとしても、シナリオ 1 のように .blob.core.windows.net に対して PutBlock & PutBlockList でアップロードすれば、シナリオ 1 に従います。


# まとめ
上記のようなシナリオとなり、アップロード中にアクセスした際のステータスコードなどは違うものの、アップロードが完了してはじめてアップロードした BLOB の内容を取得できる、ということになります。

今回は AzCopy を使いましたが、アップロードに利用するクライアントツールによって動作の違いが出る可能性はあるものの、基本的な考え方として本記事が参考になれば幸いです。

# 参考資料
[AzCopy を使ってみる](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-use-azcopy-v10)  
上記で利用した azcopy はストレージへのアクセスに利用できるコマンドラインのツールとなります。記事内では詳細に言及しておりませんので、利用方法などが気になる方はこちらをご覧ください。



<br>
<br>

---

<br>
<br>

2024 年 5 月 23 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>