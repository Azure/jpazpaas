---
title: "Azure Functions トリガー・バインディング シリーズ - BLOB トリガー"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本動作
[BLOB トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-blob?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp)は、指定された BLOB からファイルを取得し実行される関数です。

BLOB トリガーは、従来のクラシックな BLOB トリガーと [Event Grid を用いた BLOB トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-event-grid-blob-trigger?pivots=programming-language-csharp)の 2 種類が利用可能です。
前者のクラシックな BLOB トリガーの場合には、ストレージ アカウントの診断設定ログを用いて更新の有無を確認することで BLOB の更新または新規作成を検知し(削除は検知されません)、トリガーの実行を行います。後者の Event Grid を用いた BLOB トリガーの場合には、ストレージ アカウントの変更を Event Grid にて検知しトリガーの実行契機が Event Grid から Azure Functions へ発出されます。
どちらのトリガーを利用いただいても BLOB 以外にキューが利用されており、実際に BLOB ファイルを処理する前に一度実行契機となるキュー メッセージが作成された後に BLOB が処理される動作となります。


# 作成手順
BLOB トリガーの作成方法は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-your-first-function-visual-studio#create-a-function-app-project)で紹介されているテンプレートをご利用ください。チュートリアルでは HTTP トリガーを利用しておりますが、同画面にて BLOB トリガーを選択いただくことができます。

# 実行状況をログで確認する
## クラシックな BLOB トリガー
BLOB トリガーのポーリング動作をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。

**host.json**
```
"logging": {
  "logLevel": {
    "Azure.Core": "Trace"
  }
}
```

**Application Insights で利用するクエリ**
```
traces
| where timestamp > datetime("yyyy-mm-dd hh:mi")
| where customDimensions.Category startswith "Azure.Core" or operation_Name =~ "ClassicBLOBTrigger"
| order by timestamp asc
```

上記の設定を行った際に記録されるログの例です。
トリガーの実行にて、`Executing~` と `Executed~` に合わせて `Trigger Details:~` が記録されており、Queue トリガーのようなログが確認できます。

**トリガーの実行**

![image-fa9f4f38-d2b9-46e5-b720-c0ea086116a2.png]({{site.baseurl}}/media/2025/02/image-fa9f4f38-d2b9-46e5-b720-c0ea086116a2.png)

BLOB トリガーを実行するにあたり実行されている API を見ていくと、まずは [BLOB の一覧](https://learn.microsoft.com/ja-jp/rest/api/storageservices/list-blobs?tabs=microsoft-entra-id)を取得しています。次に、検知された [BLOB ファイルのメタデータ](https://learn.microsoft.com/ja-jp/rest/api/storageservices/get-blob-metadata?tabs=microsoft-entra-id)を取得しています。再度 BLOB ファイルのメタデータを取得と [BLOB ファイルのリース取得](https://learn.microsoft.com/ja-jp/rest/api/storageservices/lease-container?tabs=microsoft-entra-id)をしていますが、対象が `blobreceipts` 配下となっております。`blobreceipts` 配下には、ETag ごとに BLOB ファイル名が格納されています(黄色が BLOB に対する操作)。

その後、キューに対して[メッセージの作成](https://learn.microsoft.com/ja-jp/rest/api/storageservices/put-message)を行い、[キューの取得](https://learn.microsoft.com/ja-jp/rest/api/storageservices/get-messages)を行っています(緑色がキューに対する操作)。

**ポーリングの動作** 

![image-80290d51-78b7-49d9-b08a-275582a75cea.png]({{site.baseurl}}/media/2025/02/image-80290d51-78b7-49d9-b08a-275582a75cea.png)


Queue に格納されたメッセージの中身をみると、以下の JSON が確認できます。
```
{
  "Type": "BlobTrigger",
  "FunctionId": "Host.Functions.ClassicBLOBTrigger",
  "BlobType": "BlockBlob",
  "ContainerName": "samples-workitems",
  "BlobName": "testfile.txt",
  "ETag": "\"0x204612BCB029720\""
}
```

## Event Grid を用いた BLOB トリガー

前述のクラシックな BLOB トリガーの課題を解決するために考案されたのが Event Grid を用いた BLOB トリガーとなります。動作のアイディアは[こちら](https://github.com/Azure/azure-sdk-for-net/pull/17137)に記載がございます。また、構成方法は[Azure Functions で Event Grid トリガーとバインドを使用する方法](https://learn.microsoft.com/ja-jp/azure/azure-functions/event-grid-how-tos?tabs=v2%2Cportal)を参照ください。

Event Grid を用いた BLOB トリガーの動作をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。

**host.json**
```
"logging": {
  "logLevel": {
    "Azure.Core": "Trace"
  }
}
```

**Application Insights で利用するクエリ**
```
traces
| where timestamp > datetime("yyyy-mm-dd hh:mi")
| where customDimensions.Category startswith "Azure.Core" or operation_Name =~ "EventGridBLOBTrigger"
| order by timestamp asc
```


上記の設定を行った際に記録されるログの例です。
トリガーの実行にて、`Executing~` と `Executed~` に合わせて `Trigger Details:~` が記録されており、Queue トリガーのようなログが確認できます。また、実行の直前に BLOB に対するアクセスが確認できます。

**トリガーの実行**

![image-32bc4445-33b1-4940-bcf8-237fada9b0ec.png]({{site.baseurl}}/media/2025/02/image-32bc4445-33b1-4940-bcf8-237fada9b0ec.png)

次に BLOB のアップロードからトリガーの実行までの流れを確認していきます。詳細に確認するために今回は `ngrok` というツールを使い、ローカル環境で実行しています。

**実行までの流れ**

1. BLOB にファイルが作成されると、Event Grid から Azure Functions のエンドポイント `/runtime/webhooks/blobs` にリクエストが発出されます。
また、`ngrok` のログを確認すると Event Grid から送られてきたリクエストのペイロードにストレージ アカウントの BLOB ファイルの情報が含まれていることがわかります。

![image-eff995ff-ce04-4848-9b95-bcd05f30bbd3.png]({{site.baseurl}}/media/2025/02/image-eff995ff-ce04-4848-9b95-bcd05f30bbd3.png)

2. リクエストの受け付け後にキューに対して[メッセージの作成](https://learn.microsoft.com/ja-jp/rest/api/storageservices/put-message)が行われていることがわかります。

![image-244315d4-5803-4b33-975a-7ed883728faa.png]({{site.baseurl}}/media/2025/02/image-244315d4-5803-4b33-975a-7ed883728faa.png)


**作成されたキュー メッセージの内容**

![image-743aa4d4-7a95-4f7c-a312-2fb9ad3e2c5a.png]({{site.baseurl}}/media/2025/02/image-743aa4d4-7a95-4f7c-a312-2fb9ad3e2c5a.png)

3. 直後に対象の BLOB が `HEAD` メソッド参照されていることがわかります。その後にトリガーが実行されていることがわかります。この API は前述でも利用していた [BLOB ファイルのメタデータ](https://learn.microsoft.com/ja-jp/rest/api/storageservices/get-blob-metadata?tabs=microsoft-entra-id) となります。

![image-921d10c7-ab92-42b5-9f0f-51a12463d3a5.png]({{site.baseurl}}/media/2025/02/image-921d10c7-ab92-42b5-9f0f-51a12463d3a5.png)


# 並列実行とチューニング

## 並列実行の考え方

公式ドキュメントに記載のある[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-python#memory-usage-and-concurrency)の方法をご参照ください。`host.json` の `maxDegreeOfParallelism` によって構成します。
実際に試してみると確かに BLOB トリガーの実行中に要求のあった BLOB トリガーは先のトリガーの完了後に実行されていることがわかります。さらにキュー メッセージの作成時刻(`InsertedOn`)より、キュー メッセージは BLOB トリガーの要求時に作成されていることがわかります。つまり並列実行の制御はキューに対するポーリングを止めることで実現していることがわかります。
また、この構成はクラシックな BLOB トリガーと Event Grid を用いた BLOB トリガーで共通となります。 

**クラシックな BLOB トリガーの場合**

![image-35d09b53-b22f-4295-8756-df255c9a0ee4.png]({{site.baseurl}}/media/2025/02/image-35d09b53-b22f-4295-8756-df255c9a0ee4.png)

**Event Grid を用いた BLOB トリガーの場合**

![image-a560525d-c1c0-44ed-8f64-ab722c7bc1da.png]({{site.baseurl}}/media/2025/02/image-a560525d-c1c0-44ed-8f64-ab722c7bc1da.png)


# よくお問い合わせいただく動作
弊サポートにてよくお問い合わせいただく事例について紹介します。

## BLOB トリガーが遅れて実行された
- コールドスタートによる遅延

従量課金プランを利用の場合には、0 インスタンスの状態となることがあります。その状態からの起動は[コールドスタート](https://learn.microsoft.com/ja-jp/azure/azure-functions/event-driven-scaling?tabs=azure-cli#cold-start)と呼ばれ、何かしらのイベント(BLOB の変更や Event Grid からの HTTP 通信など)を基盤側で検知後に新たにインスタンスを割り当て、アプリケーションの起動を行います。この一連の動作に数分程度時間がかかる場合がありトリガー対象の BLOB のイベント後、数分程度遅延してトリガーが実行されることがございます。
もし、こういった実行の開始遅延を少しでも抑制したい場合には、Premium プランや App Service プランの利用を検討ください。

- 診断設定ログの出力遅れによる遅延

クラシックな BLOB トリガーの場合には、ストレージ アカウントの診断設定ログが処理対象の BLOB か否かの判断に利用されます。この診断設定ログの出力が遅れると処理対象の BLOB の判断に時間を要し、トリガー対象の BLOB のイベント後、数分程度遅延してトリガーが実行されることがございます。[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#polling-and-latency) にも注意書きがございます。もし、こういった実行の開始遅延を少しでも抑制したい場合には、Event Grid を用いた BLOB トリガーの利用を検討ください。

![image-3c8ccaea-d824-42c7-ae4d-db41c273c768.png]({{site.baseurl}}/media/2025/02/image-3c8ccaea-d824-42c7-ae4d-db41c273c768.png)

## BLOB トリガーが実行されなかった

前項にもある通りクラシックな BLOB トリガーの場合には、ストレージ アカウントの診断設定ログが処理対象の BLOB か否かの判断に利用されます。しかしながら、この診断設定ログはベストエフォートで作成されており、何かしらの理由で欠落した場合には正しく処理対象の BLOB が認識されないことが有ります。[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cextensionv5&pivots=programming-language-csharp#polling-and-latency) にも注意書きがございます。もし実行されなかった場合には、再度処理対象の BLOB を変更いただき、改めて処理対象とする必要があります。
![image-dff68c03-84f7-4eca-9461-9ff2a53ca113.png]({{site.baseurl}}/media/2025/02/image-dff68c03-84f7-4eca-9461-9ff2a53ca113.png)


# まとめ
本ブログでは BLOB トリガー関数の動作及び作成方法について整理しました。BLOB トリガーのトラブルシューティングの参考になれば幸いです。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions における Azure Blob Storage のトリガーとバインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-blob?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp)


<br>
<br>

---

<br>
<br>

2025 年 02 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>