---
title: "Azure Functions トリガー・バインディング シリーズ - Queue トリガー"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本動作
[Queue トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-queue?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp)は、指定されたキューからメッセージを取得し実行される関数です。キューに格納されたメッセージをロックし、処理が成功した場合にはメッセージを削除し、失敗した場合にはメッセージが Queue に戻されます。
しかし、Queue メッセージにロック機構はないため、代わりにメッセージごとに可視と不可視の状態を切り替えることで実質的なロックを実現しています。
詳細な動作イメージは[こちら](https://azure.github.io/jpazpaas/2023/05/15/queuetrigger-visibility-timeout.html)に紹介がございます。

![image-6214a137-3765-4cf9-af58-36a709aee001.png]({{site.baseurl}}/media/2025/02/image-6214a137-3765-4cf9-af58-36a709aee001.png)


# 作成手順
Queue トリガーの作成方法は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-your-first-function-visual-studio#create-a-function-app-project)で紹介されているテンプレートをご利用ください。チュートリアルでは HTTP トリガーを利用しておりますが、同画面にて Queue トリガーを選択いただくことができます。

また、処理先の Queue 名を、アプリケーション設定に外だしすることができます。C# の例ですが、QueueTrigger の引数で Queue 名を `%` 括りのアプリケーション設定名とします(上段)。ローカルの実行のため local.settings.json となりますが(下段)、queuename の中身を宣言します。Azure 上の Azure Functions リソースの場合には同項目をアプリケーション設定に設定ください。

![image-254eefd0-66fc-4d2d-96eb-19f907c98879.png]({{site.baseurl}}/media/2025/02/image-254eefd0-66fc-4d2d-96eb-19f907c98879.png)

# 実行状況をログで確認する
Queue トリガーのポーリング動作をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。

**host.json**
```
"logging": {
  "logLevel": {
    "Microsoft.Azure.WebJobs.Extensions.Storage.Common.Listeners.QueueListener": "Debug"
  }
}
```

**Application Insights で利用するクエリ**
```
traces
| where timestamp > datetime("yyyy-mm-dd hh:mi")
| where customDimensions.Category startswith "Microsoft.Azure.WebJobs.Extensions.Storage.Common.Listeners.QueueListener" or operation_Name =~ "QueueTrigger"
| order by timestamp asc
```

上記の設定を行った際に記録されるログの例です。1 行目にて Queue トリガーのリスナー(Queue のポーリング動作)が起動されていることが確認できます。`Poll for function 'QueueTrigger'~` 及び `Function 'QueueTrigger'~` のメッセージが記録され、キューの走査が行われていることがわかります。
その後、1 件メッセージが検知されますが、実行時に `Trigger Details: MessageId: df4406f3-8449-4cf5-8064-29383fef1336, DequeueCount: 1, InsertedOn: 2024-12-04T05:07:37.000+00:00` が記録されており処理するメッセージの ID とデキューカウント、メッセージの格納時刻が確認できます。また、正常終了ではあるものの、メッセージの削除ログは Azure Functions には記録されません。

**ログ出力例**

![image-397905dd-2367-49cb-978a-4fe2d404cd0b.png]({{site.baseurl}}/media/2025/02/image-397905dd-2367-49cb-978a-4fe2d404cd0b.png)


# 並列実行とチューニング

## 並列実行の考え方
Queue トリガーでは、バッチによってまとめてメッセージを取得し、取得したメッセージの中から実行可能なメッセージ数分を並行で処理を行います。パーティションなどの機構はないため、インスタンスごとにバッチの動作と実際に処理されるメッセージを考慮する必要があります。
[host.json](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-queue?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp#host-json) の構成にもある通り、**1 つの関数につき同時に処理されるメッセージの最大数は、batchSize に newBatchThreshold を加えた値**となります。

実際にローカルの環境で Queue トリガーを実行した結果が以下のログになります。`host.json` の変更は行わず、既定値を利用しています。手元環境の論理プロセッサ数は 16 であるため、batchSize:16[個]+newBatchThreshold:16*16/2[個]=144 並列で動作していることがわかります。

**requests ログ**

![image-206f8cb0-4f6b-4508-a54a-cd4133f2119d.png]({{site.baseurl}}/media/2025/02/image-206f8cb0-4f6b-4508-a54a-cd4133f2119d.png)

また、144 並列での実行中には Queue のポーリングが行われないため、`Poll for function 'QueueTrigger'~` 及び `Function 'QueueTrigger'~` のメッセージが記録されないことがわかります。

## シングルトンでの実行
Queue トリガーである瞬間で 1 つのメッセージしか処理しない構成とするには、[host.json](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-queue?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp#host-json) にて `"batchSize": 1` 及び `"newBatchThreshold": 0` とします。
ただし、インスタンス上で 1 つのメッセージしか処理しない構成となるため、スケール アウトも制限する必要があります。

# よくお問い合わせいただく動作
弊サポートにてよくお問い合わせいただく事例について紹介します。

## 同じメッセージの処理が 2 回実行された

Queue トリガーは、正常に実行完了した場合にメッセージが削除されますが、それ以外の場合(例えばプロセスのシャットダウンなど)にはメッセージがキューに戻されることがあります。[こちら](https://azure.github.io/jpazpaas/2023/05/15/queuetrigger-visibility-timeout.html#%E6%A4%9C%E8%A8%BC)に詳細な解説がございます。

実際の動作例を確認します。複数回実行された `MessageID` を確認すると、`DequeueCount` が 2 となっていることがわかり、確かに 2 回実行されていると判断できます。また、プロセス ID を確認すると、1 回目の実行と 2 回目の実行で異なっていることがわかります。つまり、1 回目の実行にて処理の完了を示す `Executed~` が記録されていないことから何かしらの理由で 1 回目の実行が正常に完了しなかったことが考えられます。

**traces ログ**

![image-441371b0-d06f-4b60-9726-1a2bf8534dd4.png]({{site.baseurl}}/media/2025/02/image-441371b0-d06f-4b60-9726-1a2bf8534dd4.png)


# まとめ
本ブログでは Queue トリガー関数の動作及び作成方法について整理しました。Queue トリガーのトラブルシューティングの参考になれば幸いです。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions における Azure Queue storage のトリガーとバインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-queue?tabs=isolated-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp)


<br>
<br>

---

<br>
<br>

2025 年 02 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>