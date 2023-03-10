---
title: Queueストレージのメッセージの33件目以降を確認する方法
author_name: "chansiklee"
tags:
    - Storage Account
---

# はじめに
こんにちは。Azure PaaS サポート担当の李です。

Azure ポータルやストレージエクスプローラ、PowerShell などのツールを用いて Azure キューストレージのメッセージ数を確認すると、32 件を超える場合、最初の 32 件のみが表示され、33 件目以降が確認できないことがございます。<br>
![image-83343c6e-78b8-4d05-b0e7-0de5e1214fcc.png]({{site.baseurl}}/media/2023/03/image-83343c6e-78b8-4d05-b0e7-0de5e1214fcc.png)<br>
図1 Azure ポータルで確認した結果<br>

![image-67e8ab24-5152-4fe1-b6f5-be588371bb84.png]({{site.baseurl}}/media/2023/03/image-67e8ab24-5152-4fe1-b6f5-be588371bb84.png)<br>
図2 Azureストレージエクスプローラで確認した結果<br>

これは下記の記事の内容通り、キューサービスの REST API でメッセージを取得する際、取得するメッセージ数のパラメーター「numofmessages」の上限が 32 件までであり、クライアント側で、繰り返しの取得を想定していないことが考えられます。<br>
・[キューサービスの REST API ‐ メッセージの取得 - URI パラメーター](https://learn.microsoft.com/ja-jp/rest/api/storageservices/get-messages#uri-parameters)<br>
![image-f977e59d-cce4-4732-80cb-ff6d7fc2603a.png]({{site.baseurl}}/media/2023/03/image-f977e59d-cce4-4732-80cb-ff6d7fc2603a.png)<br>

そのため、 33 件以上のメッセージを一挙に取得・表示したい場合には、まず 32 件のメッセージを取得した後、取得した 32 件のメッセージを削除し、そのあと次の 32 件を取得するという手順をふむ必要があります。<br>
・[StackOverflow - Azure Queue Peek All Messages](https://stackoverflow.com/questions/26485608/azure-queue-peek-all-messages)<br>

ただし 32 件を超えるすべての格納されているメッセージを確認したい場合、上記の方法ですと必ずメッセージを削除する必要があり、メッセージを確認した後もメッセージを利用する必要がある場合は、利用できないことが想定されます。

そのため、別の方法として、メッセージを検索する際に パラメーター「visibilitytimeout」を十分長く設定し、その指定した期間の間に繰り返しキューからのメッセージの取得を繰り返すことで全てのメッセージを取得することができます。<br>
![image-8ae47764-6938-4d63-9b45-d202b0f937b1.png]({{site.baseurl}}/media/2023/03/image-8ae47764-6938-4d63-9b45-d202b0f937b1.png)<br>
visibilitytimeout を指定してGet Messages のリクエストを送信すると、対象のメッセージ (表示された、最大32件のキュー) は指定した期間内では取得対象外となるため、
再度 Get Messages のリクエストを送信すると次の 32 件が表示されます。<br>
※ 但し、**別のクライアントからも参照している状況** である場合、visibilitytimeout を指定すると **その別のクライアントからも指定した期間の間キューのメッセージが参照できなくなる** ため、下記のコマンドを実行する前に、必ず visibilitytimeout のパラメータを指定することにより影響が発生しないかどうかご確認ください。

また、 PowerShell で Azure キューストレージを使用する方法や、必要なモジュールなどに関しては以下の記事をご参照ください。<br>
・[PowerShell から Azure Queue Storage を使用する方法 - Azure Storage | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/storage/queues/storage-powershell-how-to-use-queues)<br>

PowerShell スクリプトで実現する場合、以下がサンプルスクリプトとなります。
ご活用いただく際には Windows PowerShell 若しくは Cloud Shell にスクリプトをコピーし、必要なパラメータを入力の上、実行してください。<br>
また、下記スクリプトをご利用いただく際には、お客様環境やシステム構成などを考慮いただき、また十分な検証を実施したうえでご活用ください。

```
$accountName="<Account名>"
$accountKey="<Key>"
$queueName="<QueueName>"
$invisibleTimeout = [System.TimeSpan]::FromSeconds(40)     # visibilitytimeout 時間を40秒に指定します。

$ctx = New-AzStorageContext -StorageAccountName $accountName -StorageAccountKey $accountKey
$queue = Get-AzStorageQueue –Name $queueName –Context $ctx

$cnt = 0;
$outputStr
while ($true) # visibilitytimeoutを指定して32件のメッセージを取得する作業を繰り返し行います。
{
    $queueMessages = $queue.CloudQueue.GetMessages(32, $invisibleTimeout,$2023 年 03 月 10 日時点の内容となります。<br>,$null)

    foreach ($message in $queueMessages)
    {
        $cnt++;
        $outputStr = $cnt.ToString() + " Message目 : " + $message.AsString
        Write-Output $outputStr
    }

   if ($queueMessages.Count -eq 0)   # 全てのキューメッセージの取得が終わったら終了します。
    {
        break;
    }
}

```
![image-1e53bdee-3d9b-4fd5-a911-f143d4bf2df1.png]({{site.baseurl}}/media/2023/03/image-1e53bdee-3d9b-4fd5-a911-f143d4bf2df1.png)<br>
図3 上記のスクリプトを実行した結果 

図3 をご確認いただくと、メッセージが 32 件を超えて全て表示されたことを確認いただけるかと思います。ぜひお手元の環境でもお試しいただければと思います。<br>

以上、PowerShell スクリプトを用いて、Queue ストレージのメッセージの 33 件目以降を確認する方法を紹介いたしました。

<br>
<br>

---

<br>
<br>

2023 年 02 月 17 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>