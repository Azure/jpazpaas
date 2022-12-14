---
title: "Azure Functions の Graceful Shutdown について"
author_name: "Hayato Kuroda"
tags:
    - Azure Functions
    - Durable Functions
---

# 質問
Azure Functions の Graceful Shutdown について

# 回答
Graceful Shutdown は、より安全にインスタンスのシャットダウンを行う仕組みのことを指しています。

特に Azure Functions においては、アプリケーションが動作するインスタンス上でインスタンスの停止処理が発生する際にあるオブジェクトに対して停止処理の開始を通知し利用者側で後述するある程度の猶予のもとアプリケーションの処理を制御可能な仕組みを指しています。

本稿では、Graceful Shutdown の詳解は省略するものの、Azure Functions を利用時にどういったことを実装いただけるかという点について紹介していこうと思います。

## 停止処理の検知と実装例
C# の場合には以下のドキュメントにあるように CancellationToken クラスの IsCancellationRequested の真偽値を評価することで停止を検知することができます。

■ [Azure Functions を使用する C# クラス ライブラリ関数を開発する - キャンセル トークン](
https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-dotnet-class-library?tabs=v2%2Ccmd#cancellation-tokens)

![image-b73fa841-761d-4526-924f-bc458c83ec41.png]({{site.baseurl}}/media/2022/12/image-b73fa841-761d-4526-924f-bc458c83ec41.png)

## Azure Functions の停止処理時の動作
Azure Functions では Azure ポータルからの停止要求、スケーリングまたは Azure メンテナンスによる再起動が発生する可能性があります。HTTP トリガー等アプリケーションの処理実行中に停止されてしまう場合、どういった制御が可能であるかは関心事だと思います。

本題に入りましょう。

Azure Functions は停止処理開始時にアプリケーションの実行が行われている場合、**10 秒間** 待機する動作となります。

[WebJobsScriptHostService.cs](https://github.com/Azure/azure-functions-host/blob/ed814c569434d65abcc22a169ac2d95f1ef79c32/src/WebJobs.Script.WebHost/WebJobsScriptHostService.cs#L411)

```
01 public async Task StopAsync(CancellationToken cancellationToken)
02         {
03             _startupLoopTokenSource?.Cancel();
04 
05             State = ScriptHostState.Stopping;
06             _logger.Stopping();
07 
08             var currentHost = ActiveHost;
09             ActiveHost = null;
10             Task stopTask = Orphan(currentHost, cancellationToken);
11             Task result = await Task.WhenAny(stopTask, Task.Delay(TimeSpan.FromSeconds(10), cancellationToken));
12 
13             if (result != stopTask)
14             {
15                 _logger.DidNotShutDown();
16             }
17             else
18             {
19                 _logger.ShutDownCompleted();
20             }
21 
22             State = ScriptHostState.Stopped;
23         }
```

少し解説します。

上記の停止処理の中で、11 行目の Task result にて await Task.WhenAny でタスクの完了待ちをしています。待っているタスクは、stopTask または Task.Delay となっており、stopTaks の実体は [Orphan](https://github.com/Azure/azure-functions-host/blob/ed814c569434d65abcc22a169ac2d95f1ef79c32/src/WebJobs.Script.WebHost/WebJobsScriptHostService.cs#L669) メソッド、Task.Delay の方では遅くとも 10 待機後に完了 Task を返却します。Orphan というメソッドでは、実行中の関数を待機する処理を継続しています。つまり、関数の実行が終わるか、長くとも 10 秒待つという解釈になります。またこの 10 秒という待機時間はハードコードされているため利用者様にて変更はしていただけません。

以上から、Graceful Shutdown 時には 10 秒(ハードコード)待つけどそれ以上は強制的にシャットダウンされる動作でした。


検証してみましょう。以下の HTTP トリガーの動作を作成します。キャンセル トークンをハンドルして無限ループでログを出力し続けるプログラムを書きました。

```
#r "Newtonsoft.Json"

using System;
using System.Net;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Primitives;
using Newtonsoft.Json;
using System.Threading;

public static async Task<IActionResult> Run(HttpRequest req, ILogger log, CancellationToken cancellationToken)
{
    while(true){
        if (cancellationToken.IsCancellationRequested)
        {
            log.LogInformation("A cancellation token was received. Taking precautionary actions.");
            while(true){
                DateTime dt = DateTime.Now;
                log.LogInformation("In cancellationToken proc: " + dt.ToString("yyyy-MM-dd HH:mm:ss.fff"));
                Thread.Sleep(1000);
            }
            break;
        }
        else
        {
            DateTime dt = DateTime.Now;
            log.LogInformation(dt.ToString("yyyy-MM-dd HH:mm:ss.fff"));
            Thread.Sleep(1000);
        }
    }
    
    return new OkObjectResult(responseMessage);
}
```

実行中に Azure Functions を Azure ポータルから停止させると以下のようなログとなりました。赤枠部分の通りキャンセル トークンを受け取った後に 10 秒間は処理を継続していたことが確認できました。

![image-be68284c-551d-4b49-8080-e9bf5ba5db5c.png]({{site.baseurl}}/media/2022/12/image-be68284c-551d-4b49-8080-e9bf5ba5db5c.png)

＊ Linux OS の場合には、アーキテクチャが異なるためにキャンセル トークンをハンドルしていただけません。また、2022 年 11 月 現在では分離プロセスの C# においてもキャンセル トークンのハンドルは実施いただけませんため今後のアップデートをご期待ください。

[参考 GitHub issue](https://github.com/Azure/azure-functions-dotnet-worker/issues/654#issuecomment-970727273)

以上、シャットダウン時に猶予として与えられる 10 秒を使いより安全にアプリケーションが終了するようにできる仕組みの紹介でした。少しでも参考になればと思います。

<br>
<br>

---

<br>
<br>

2022 年 12 月 05 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
