---
title: "BlobのMD5 ハッシュの整合性を確認する方法"
author_name: "Hiroaki Machida"
tags:
    - Storage
---

インターネット上で転送されたバイナリファイルについて、ファイルが破損していないかや、ファイルが改ざんされていないかを確認したい場合があります。

一般的には、バイナリファイルを取得した側が、バイナリファイルから計算したハッシュ値と配布側が公開しているハッシュ値を比較してファイルが正しいことを担保します。ハッシュ値を計算するアルゴリズムは、 MD5 (128bit) 、SHA-1 (160bit) 、SHA-256 (256bit) などが代表的です。

Azure Blob Storage では Blob の MD5 ハッシュ を利用することで、アップロードされた Blob またはダウンロードされた Blob が正しいことを検証できます。




# 質問
Azure Blob Storageで Blob をアップロードまたはダウンロードしたときに、Blob の MD5 ハッシュ検証を行うにはどうすればよいでしょうか。

# 回答
Azure Storage Blobs client library for .NET (12.11.0) を利用して Blob の MD5 ハッシュ検証を行う方法を、3 つのシナリオに分けてご説明します。


 - Blob を正しい MD5 ハッシュでアップロードする 
 - Blob を誤った MD5 ハッシュでアップロードする 
 - Blob をダウンロードして MD5 ハッシュを検証する 

## 1. Blob を正しい MD5 ハッシュでアップロードする 

![Blob Storage MD5 Check 1.png]({{site.baseurl}}/media/2022/04/2022-04-21-BlobStorageMD5Check1.png)

Blob を正しい MD5 ハッシュでアップロードします。ストレージ サービスはアップロードされた Blob から計算した MD5 ハッシュとクライアントから受信した MD5 ハッシュが同じであることを確認し、ステータスコード 202 (Created) を返します。

### ローカルにファイルを作成する
Blob Stroage にアップロードするファイルを作成します。

```csharp
string localPath = "./data/";
string fileName = Guid.NewGuid().ToString() + ".txt";
string localFilePath = Path.Combine(localPath, fileName);
string content = "Hello, World!";

Directory.CreateDirectory(localPath);
File.WriteAllText(localFilePath, content);
Console.WriteLine($"Created a local file: {localFilePath}");
```

### MD5 ハッシュを計算する
ローカルのファイルの MD5 ハッシュを計算します。

```csharp
byte[] hashBytes;
string hash;
using (var stream = File.OpenRead(localFilePath))
using (var md5 = MD5.Create())
{
    hashBytes = md5.ComputeHash(stream);
    hash = Convert.ToBase64String(hashBytes);
}
Console.WriteLine($"Calculated a local file hash: {hash}");
```


### Blob Storage のコンテナを作成する
Blob Storage のコンテナを作成します。

connectionString にはストレージ アカウントの接続文字列を指定してください。

```csharp
string connectionString = "DefaultEndpointsProtocol=https;AccountName=***;AccountKey=***;EndpointSuffix=core.windows.net";
string containerName = Guid.NewGuid().ToString();
BlobContainerClient container = new BlobContainerClient(connectionString, containerName);
container.Create();
BlobClient blobClient = container.GetBlobClient(fileName);
Console.WriteLine($"Initialized a blob client: {containerName}");
```

### Blob をアップロードする
ローカルで作成したファイルの MD5 ハッシュを blobUploadOptions で指定して、ストレージ サービスへ Blob をアップロードします。

ストレージ サービスは、クライアントが送信した MD5 ハッシュと、Blob から計算した MD5 ハッシュを照合します。このケースでは MD5 ハッシュが一致するため、アップロードが正常に完了します。

ストレージ サービスで計算された MD5 ハッシュは、レスポンスの ContentHash プロパティで確認できます。

```csharp
var blobHeaders = new BlobHttpHeaders { ContentHash = hashBytes };
var blobUploadOptions = new BlobUploadOptions { HttpHeaders = blobHeaders };

var response = blobClient.Upload(localFilePath, blobUploadOptions);
Console.WriteLine($"Uploaded a blob. Hash of the blob uploaded: {Convert.ToBase64String(response.Value.ContentHash)}");
```


## 2. Blob を誤った MD5 ハッシュでアップロードする
![Blob Storage MD5 Check 2.png]({{site.baseurl}}/media/2022/04/2022-04-21-BlobStorageMD5Check2.png)

Blob を誤った MD5 ハッシュでアップロードします。ストレージ サービスはアップロードされた Blob から計算した MD5 ハッシュとクライアントから受信した MD5 ハッシュが異なることを検知し、ステータスコード 400 (Bad Request) を返します。

### ローカルにファイルを作成する
Blob Stroage にアップロードするファイルを作成します。

```csharp
string localPath = "./data/";
string fileName = Guid.NewGuid().ToString() + ".txt";
string localFilePath = Path.Combine(localPath, fileName);
string content = "Hello, World!";

Directory.CreateDirectory(localPath);
File.WriteAllText(localFilePath, content);
Console.WriteLine($"Created a local file: {localFilePath}");
```

### MD5 ハッシュを計算する
ローカルのファイルの MD5 ハッシュを計算します。

```csharp
string hash;
using (var stream = File.OpenRead(localFilePath))
using (var md5 = MD5.Create())
{
    hash = Convert.ToBase64String(md5.ComputeHash(stream));
}
Console.WriteLine($"Calculated a local file hash: {hash}");
```

### Blob Storage のコンテナを作成する
Blob Storage のコンテナを作成します。

connectionString にはストレージ アカウントの接続文字列を指定してください。

```csharp
string connectionString = "DefaultEndpointsProtocol=https;AccountName=***;AccountKey=***;EndpointSuffix=core.windows.net";
string containerName = Guid.NewGuid().ToString();
BlobContainerClient container = new BlobContainerClient(connectionString, containerName);
container.Create();
BlobClient blobClient = container.GetBlobClient(fileName);
Console.WriteLine($"Initialized a blob client: {containerName}");
```

### Blob をアップロードする
誤った MD5 ハッシュを blobUploadOptions で指定して、ストレージ サービスへ Blob をアップロードします。

ストレージ サービスは、クライアントが送信した MD5 ハッシュと、Blob から計算した MD5 ハッシュを照合します。このケースでは MD5 ハッシュが異なるため、アップロードが失敗します。

アップロードが失敗すると、以下のようなエラーメッセージがストレージ サービスから返ります。
```
The MD5 value specified in the request did not match with the MD5 value calculated by the server.
RequestId:343e5491-901e-0084-172b-3ea0ae000000
Time:2022-03-22T20:31:56.4809549Z
Status: 400 (The MD5 value specified in the request did not match with the MD5 value calculated by the server.)
ErrorCode: Md5Mismatch
```

```csharp
try
{
    var blobHeaders = new BlobHttpHeaders { ContentHash = new byte[16] };
    var blobUploadOptions = new BlobUploadOptions { HttpHeaders = blobHeaders };

    Console.WriteLine($"Uploading a blob with a wrong hash.");
    blobClient.Upload(localFilePath, blobUploadOptions);
}
catch (RequestFailedException e)
{
    Console.WriteLine("==============================================");
    Console.WriteLine(e.Message);
    Console.WriteLine("==============================================");
}
```


## 3. Blob をダウンロードして MD5 ハッシュを検証する

![Blob Storage MD5 Check 3.png]({{site.baseurl}}/media/2022/04/2022-04-21-BlobStorageMD5Check3.png)

Blob をダウンロードして MD5 ハッシュを検証します。ダウンロードした Blob から計算した MD5 ハッシュと、ストレージ サービスから送信された MD5 ハッシュが同じであることを確認します。


### ローカルにファイルを作成する
Blob Stroage にアップロードするファイルを作成します。

```csharp
string localPath = "./data/";
string fileName = Guid.NewGuid().ToString() + ".txt";
string localFilePath = Path.Combine(localPath, fileName);
string content = "Hello, World!";

Directory.CreateDirectory(localPath);
File.WriteAllText(localFilePath, content);
Console.WriteLine($"Created a local file: {localFilePath}");
```

### Blob Storage のコンテナを作成する
Blob Storage のコンテナを作成します。

connectionString にはストレージ アカウントの接続文字列を指定してください。

```csharp
string connectionString = "DefaultEndpointsProtocol=https;AccountName=***;AccountKey=***;EndpointSuffix=core.windows.net";
string containerName = Guid.NewGuid().ToString();
BlobContainerClient container = new BlobContainerClient(connectionString, containerName);
container.Create();
BlobClient blobClient = container.GetBlobClient(fileName);
Console.WriteLine($"Initialized a blob client: {containerName}");
```

### Blob をアップロードする
ローカルで作成したファイルをストレージ サービスへアップロードします。

```csharp
var blobHeaders = new BlobHttpHeaders { ContentHash = hashBytes };
var blobUploadOptions = new BlobUploadOptions { HttpHeaders = blobHeaders };

blobClient.Upload(localFilePath, blobUploadOptions);
```

### Blob をダウンロードする
Blob をダウンロードします。

ダウンロードしたコンテンツはローカルのファイルに保存します。


```csharp
var response = blobClient.DownloadContent();

string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
File.WriteAllBytes(downloadFilePath, response.Value.Content.ToArray());
```

### MD5 ハッシュを取得する
ストレージ サービスから返された Blob Download Result に含まれる ContentHash プロパティから MD5 ハッシュを取得します。

次に、ローカルのファイルの MD5 ハッシュを計算します。

```csharp
string responseHash = Convert.ToBase64String(response.Value.Details.ContentHash);
Console.WriteLine($"Downloaded the blob. Hash from blob download result: {responseHash}");

string downloadedFileHash;
using (var stream = File.OpenRead(downloadFilePath))
using (var md5 = MD5.Create())
{
    downloadedFileHash = Convert.ToBase64String(md5.ComputeHash(stream));
}
```

### MD5 ハッシュを検証する
ダウンロードしたファイルの MD5 ハッシュと、ストレージ サービスから返された MD5 ハッシュが同じであることを検証します。

```csharp
if (downloadedFileHash.Equals(responseHash))
{
    Console.WriteLine($"Hash from downloaded file({downloadedFileHash}) and hash from response ({responseHash}) matched!");
}
```

# 参考ドキュメント
## サンプルコード
[BlobMD5Check](https://github.com/jpazpaas/samples/tree/main/DeveloperPOD/Storage/BlobMD5Check)

## Azure Storage Blobs client library for .NET
[Azure Storage Blobs client library for .NET](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/storage/Azure.Storage.Blobs)

## クイックスタート Azure Blob Storage クライアント ライブラリ v12 
[クイックスタート: .NET 用 Azure Blob Storage クライアント ライブラリ v12](https://docs.microsoft.com/ja-jp/azure/storage/blobs/storage-quickstart-blobs-dotnet?tabs=environment-variable-windows)

## Pub Blob の MD5 ハッシュ照合
[Put Blob 要求ヘッダー (すべての BLOB の種類)](https://docs.microsoft.com/ja-jp/rest/api/storageservices/put-blob#request-headers-all-blob-types)

| 要求ヘッダー | 説明 |
|-------------|-----------|
| Content-MD5	  | 任意。 BLOB のコンテンツの MD5 ハッシュ。 このハッシュは転送時の BLOB の整合性を確認するために使用します。 **このヘッダーを指定すると、ストレージ サービスによって、到達したハッシュと送信されたハッシュが照合されます。** 2 つのハッシュが一致しない場合、操作はステータス コード 400 (Bad Request) で失敗します。<br /><br />2012-02-12 以降のバージョンで省略した場合、BLOB Service では MD5 ハッシュが生成されます。<br /><br />BLOB の [取得]、 [BLOB プロパティの取得] 、および BLOB の一覧表示 [] の結果には 、MD5 ハッシュが含まれます。|

## Get Blob のレスポンスに含まれる MD5 ハッシュ
[Get Blob レスポンス ヘッダー](https://docs.microsoft.com/ja-jp/rest/api/storageservices/get-blob#response-headers)


| 要求ヘッダー | 説明 |
|-------------|-----------|
| Content-MD5	  | **BLOB に MD5 ハッシュがあり、この Get Blob 操作で BLOB 全体を読み取る場合は、クライアントがメッセージ コンテンツの整合性をチェックできるようにこの応答ヘッダーが返されます。**<br /><br />バージョン 2012-02-12 以降では、Put Blob は、Put Blob 要求に MD5 ヘッダーが含まれない場合でも、ブロック BLOB の MD5 ハッシュ値を設定します。<br /><br />要求が指定された範囲を読み取ることで、がに設定されている場合、 x-ms-range-get-content-md5 true 範囲サイズが 4 MiB 以下である限り、要求は範囲の MD5 ハッシュを返します。<br /><br />これらの条件セットのいずれも真でない場合は、Content-MD5 ヘッダーの値は返されません。<br /><br />x-ms-range-get-content-md5 が Range ヘッダーなしで指定されている場合、サービスはステータス コード 400 (Bad Request) を返します。<br /><br />x-ms-range-get-content-md5範囲が 4 MiB を超える場合、がに設定されている true と、サービスはステータスコード 400 (Bad Request) を返します。|

<br>
<br>

---

<br>
<br>
2022 年 4 月 21 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>