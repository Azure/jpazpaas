---
title: "Queue トリガーの visibilityTimeout"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - WebJobs
---

# 質問
Azure Functions の Queue トリガーを利用しています。host.json に visibilityTimeout を設定しているのですが、Queue トリガーの再試行時にvisibilityTimeout の値通りの時間で動作しません。

# 回答
Queue トリガーでは、処理対象のメッセージを不可視状態にしてメッセージ ロックに近い機構を実現しています。場合によってメッセージが再度可視状態になるタイミングが異なります。そのため、Queue トリガーの再試行時の動作に寄って再試行時の実行間隔が異なります。

下記 Azure Functions が利用する内部の SDK のソースコードを基に解説します。

## 不可視状態への変更

Queue トリガーでは、処理対象のメッセージを不可視状態にすることで他のクライアントから同一のメッセージが取得できないように制御しています。これは元来 Queue メッセージに対する操作として存在しており、Azure Functions 独自のものではありません。

Queue トリガーが新規のメッセージを処理する際に [GetMessagesAsync メソッド](https://github.com/Azure/azure-webjobs-sdk/blob/863f835059ce0f27b5e7e662c00f9887d640e5bc/src/Microsoft.Azure.WebJobs.Extensions.Storage/Queues/Listeners/QueueListener.cs#L204)が呼び出されます。基底のクラスの [GetMessagesAsync](https://learn.microsoft.com/ja-jp/dotnet/api/microsoft.azure.storage.queue.cloudqueue.getmessagesasync?view=azure-dotnet) が呼び出されます。この時に引数に渡されている visibilityTimeout が不可視状態の期間を示す値となります。
基底のクラスの GetMessagesAsync に定義がある通り、visibilityTimeout は可視性のタイムアウト間隔を指定する項目となっています。

```
batch = await TimeoutHandler.ExecuteWithTimeout(nameof(CloudQueue.GetMessageAsync), context.ClientRequestID,
                        _exceptionHandler, _logger, cancellationToken, () =>
                        {
                            return _queue.GetMessagesAsync(_queueProcessor.BatchSize,
                                _visibilityTimeout,
                                options: DefaultQueueRequestOptions,
                                operationContext: context,
                                cancellationToken: cancellationToken);
                        });
```

この GetMessageAsync メソッドを呼び出す元は、Queue リスナーとなりますが、visibilityTimeout は固定で [10 分](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs.Extensions.Storage/Queues/Listeners/QueueListener.cs#L89
)が設定されています。つまり、Queue トリガーの動作にて 10 分ごとに不可視状態を更新し続けることで同一の Queue メッセージが Azure Functions の 1 つのインスタンスのみから処理されることを保証します。
```
// if the function runs longer than this, the invisibility will be updated
// on a timer periodically for the duration of the function execution
_visibilityTimeout = TimeSpan.FromMinutes(10);
```

## Queue トリガー処理終了時の動作

実行結果の可否によらず、Queue トリガーの処理終了時にはメッセ－ジが更新されます。アプリケーションが正常終了した場合にはメッセ－ジを削除しますが、異常終了した場合には Queue メッセージを可視状態に戻します。

Queue トリガーの処理終了時には [CompleteProcessingMessageAsync](https://github.com/Azure/azure-webjobs-sdk/blob/3f4ec78be9f43bb041937425ced00b341883aa42/src/Microsoft.Azure.WebJobs.Extensions.Storage/Queues/QueueProcessor.cs#L99-L134) が実行されます。正常終了であれば、DeleteMessageAsync が呼び出されメッセ－ジの削除が行われます。正常終了ではない場合には、デキュー カウント(そのメッセージを取り出した回数)を比較しつつ、ReleaseMessageAsync が呼び出されます。ReleaseMessageAsync の処理の中で [UpdateMessageAsync](https://learn.microsoft.com/ja-jp/dotnet/api/microsoft.azure.storage.queue.cloudqueue.updatemessageasync?view=azure-dotnet) にて VisibilityTimeout が反映されます。この分岐に入った場合には、host.json に設定された [visibilityTimeout](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-storage-queue?tabs=in-process%2Cextensionv5%2Cextensionv3&pivots=programming-language-csharp#host-json) が Queue メッセージに反映されます。
```
/// <summary>
/// This method completes processing of the specified message, after the job function has been invoked.
/// </summary>
/// <remarks>
/// If the message was processed successfully, the message should be deleted. If message processing failed, the
/// message should be release back to the queue, or if the maximum dequeue count has been exceeded, the message
/// should be moved to the poison queue (if poison queue handling is configured for the queue).
/// </remarks>
/// <param name="message">The message to complete processing for.</param>
/// <param name="result">The <see cref="FunctionResult"/> from the job invocation.</param>
/// <param name="cancellationToken">The <see cref="CancellationToken"/> to use</param>
/// <returns></returns>
public virtual async Task CompleteProcessingMessageAsync(CloudQueueMessage message, FunctionResult result, CancellationToken cancellationToken)
{
    if (result.Succeeded)
    {
        await DeleteMessageAsync(message, cancellationToken);
    }
    else if (_poisonQueue != null)
    {
        if (message.DequeueCount >= MaxDequeueCount)
        {
            await HandlePoisonMessageAsync(message, cancellationToken);
        }
        else
        {
            await ReleaseMessageAsync(message, result, VisibilityTimeout, cancellationToken);
        }
    }
    else
    {
        // For queues without a corresponding poison queue, leave the message invisible when processing
        // fails to prevent a fast infinite loop.
        // Specifically, don't call ReleaseMessage(message)
    }
}
```

一方で、いずれの分岐でもキャッチされない例外が発生する場合を考えます。この場合、不可視状態への変更にてお伝えした通り、Queue メッセージの処理を開始する際に設定された 10 分間が適用された状態となります。そのため、Queue メッセージが再度実行開始されるまで最大で 10 分間の遅延が発生します。


## 検証
Queue トリガーの動作時に Azure Functions のメンテナンスなどの動作にて処理中にプロセスが強制的に再起動される場合とアプリケーション処理で例外が発生する場合を検証します。すると、Application Insights にて下記のログが確認できます。

- 処理中にプロセスが強制的に再起動される場合

2023/4/24 12:41:09.583 に実行開始した Queue トリガーにて、メッセージ ID/77ad22a3-23c6-4811-839a-57f9437a5a4e が処理されています。直後にアプリケーション プロセスを手動で kill しており、実行の完了や失敗などを示すログがないものの同一のメッセージに対して 2023/4/24 12:51:13.042 に再度 Queue トリガーが開始されています。この場合には、いずれの分岐でもキャッチされないため 10 分後に再度実行が開始されています。

![image-d70ba119-bd70-4db3-9db5-d5d06fa43493.png]({{site.baseurl}}/media/2023/05/image-d70ba119-bd70-4db3-9db5-d5d06fa43493.png)


- アプリケーション処理で例外が発生する場合

2023/4/25 1:24:15.292 に実行開始した Queue トリガーにて、メッセージ ID/c9c992b1-702e-4f2b-8470-bf1489df67e7 が処理されています。その後、例外の発生により処理が失敗し、ログには、`Executed 'Functions.QueueTrigger1' (Failed, Id=c8418101-5a74-4d53-8b93-2869e6dd415d, Duration=48ms)` が出力されています。その後、同じ Queue メッセージに対してトリガーの実行が 2023/4/25 1:24:15.637 に開始されています。host.json に設定した visibilityTimeout(標準 00:00:00) が反映されていることがわかります。

![image-8ad1b2dc-006f-4bd7-9002-eeae2b400eb5.png]({{site.baseurl}}/media/2023/05/image-8ad1b2dc-006f-4bd7-9002-eeae2b400eb5.png)



以上、Queue トリガーの動作について解説しました。visibilityTimeout については、どんな制御ができるかなどわかりにくい部分もあるかと思います。少しでもご参考になれば幸いです。

<br>
<br>

---

<br>
<br>

2023 年 05 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>