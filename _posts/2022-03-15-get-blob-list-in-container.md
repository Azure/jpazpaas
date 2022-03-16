---
title: "コンテナー内の Blob の一覧の取得方法"
author_name: "Eiji Mochida"
tags:
    - Storage
---

# 質問
List Blobs REST API を使って、Blob の一覧を取得する場合に取得できる件数に制限がありますか。<br/>
また、制限がある場合、その制限を超えて Blob の一覧を取得するにはどうすればいいですか。

List Blobs<br/>
[https://docs.microsoft.com/ja-jp/rest/api/storageservices/list-blobs](https://docs.microsoft.com/ja-jp/rest/api/storageservices/list-blobs)

# 回答
List Blobs REST API については、一度に取得できる Blob の最大件数は 5,000 件となります。下記資料にも 「要求で結果の最大数が指定されていない場合、または5000を超える場合、サーバーは最大5000項目を返します。」という記載がご覧いただけます。

Blob ストレージリソースの一覧表示<br/>
[https://docs.microsoft.com/ja-jp/rest/api/storageservices/enumerating-blob-resources](https://docs.microsoft.com/ja-jp/rest/api/storageservices/enumerating-blob-resources)

そのため、5,000件を超える Blob が存在する場合に一覧を取得したい場合、5,000 件ごとの取得を繰り返す必要がございます。  
List Blobs REST API を実行したときに、5,000 件を超える Blob が存在する場合、上記 List Blobs の説明にあるように XML の要素として `<NextMarker>` という要素にマーカーの要素に値が入りますので、そのマーカーの内容をクエリ文字列 marker に指定して、再び List Blobs を実行すると、1回目の List Blobs の続きから Blob の情報が取得できるという流れになります。

例えば、以下のように List Blobs のリクエストを発行したとします。

```
GET https://xxxxx.blob.core.windows.net/container1?restype=container&comp=list&prefix=&maxresults=5000
```

List Blobs REST API は XML 形式で応答を返します。  
そして、該当のコンテナーには 5,000 件以上の Blob が存在し、NextMarker として以下が応答に含まれていたとします。(実データではなくマスクしています。)

```
<NextMarker>2ZZZZZZZZ--</NextMarker>
```

この場合に続きを取得する場合、以下のようにクエリ文字列 marker に NextMarker を指定します。

```
GET https://xxxxx.blob.core.windows.net/container1?restype=container&comp=list&marker=2ZZZZZZZZ--&prefix=&maxresults=5000
```

この手順を応答の <NextMarker> が空になるまで繰り返すことで、5,000 件を超える Blob の一覧が取得できます。

なお、REST の場合は上記のとおりとなりますが、Azure PowerShell や Azure CLI、AzCopy などを使えば、上記の内容は気にする必要はなく、ライブラリ側で上記の作業を自動的に実施しますので、REST を直接利用する必要がなければ、これらのツールのご利用もご検討ください。例えば PowerShell でしたら以下の内容で 5,000 件を超える Blob の一覧が取得できます。
以下コマンドを 5,000 件を超える Blob のあるコンテナーに対して実行している時に HTTP 通信をキャプチャすると、先述のとおり、クエリ文字列 marker を使って、List Blobs を複数回呼び出していることがご覧いただけますので、興味がありましたらご覧ください。

```
$StorageAccountName = "<ストレージアカウント名>" 
$StorageAccountAccessKey = "<ストレージアカウントのアクセスキー>" 
$containerName = "<コンテナー名>" 

$context = New-AzStorageContext -StorageAccountName $StorageAccountName -StorageAccountKey $StorageAccountAccessKey
$listOfBlobs = Get-AzStorageBlob -Container $ContainerName -Context $context 

$listOfBlobs | select Name, Length
```

Blob のコンテナーに、多くの Blob が存在するときに、REST API だと 5,000 件しか取れなかった、というような場合には、この記事で記載した通り、NextMarker があるときに、次のリクエストを発行するようにしていない、ということが考えられますので、何かの参考になれば幸いです。

<br>
<br>

---

<br>
<br>

2022 年 3 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
