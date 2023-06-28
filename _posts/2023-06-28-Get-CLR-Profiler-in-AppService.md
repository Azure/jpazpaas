---
title: "App Service で CLR Profiler (プロファイラー) を取得する"
author_name: "yunasugimoto"
tags:
    - App Service
---
# 質問
App Service で CLR Profiler を取得する方法を教えてください。

# 回答
App Service では [問題の診断と解決](https://learn.microsoft.com/ja-jp/azure/app-service/overview-diagnostics) よりご利用いただける以下 2 つの方法でそれぞれ [.NET アプリケーションの解析に用いられる CLR Profiler](https://learn.microsoft.com/ja-jp/dotnet/framework/unmanaged-api/profiling/profiling-overview) を取得できます。ご要件に応じて CLR Profiler の取得方法をご選択ください。

1. Collect a Profiler Trace (お客様よりボタンを押下してプロファイリングを開始します)
1. Auto-Heal (条件に合致した場合にプロファイリングを開始します)

取得した CLR Profiler は [PerfView](https://jpdsi.github.io/blog/web-apps/LogCollection3/) 等のツールでお客様より解析いただくことが出来ます。CLR Profiler により取得したデータは、アプリケーションの動作が遅い・応答しない、CPUやメモリーが高いといったパフォーマンス系の問題が発生した場合に原因の追究に有効となる場合があります。


**<注意事項>**

CLR Profiler を使用した調査では、プロファイリングの開始から終了までの間に調査対象とする事象が含まれている必要があります。Option2 Auto-Heal を使用した取得方法では問題発生時に自動的にプロファイリングが行われますが、Option1 ではお客様により手動でプロファイリングを開始する必要があるため、プロファイリングの開始から終了までの間に App Service 上で調査対象とする事象が発生していることをご確認ください。

# Option1 ) Collect a Profiler Trace
1. Azure ポータルで対象の App Service を選択し、左側ブレード [問題の診断と解決] > **Collect .NET Profiler Trace** を選択します。
![image-9e1d4d02-86e4-4c8f-9ac5-b2987539d734.png]({{site.baseurl}}/media/2023/06/image-9e1d4d02-86e4-4c8f-9ac5-b2987539d734.png)
2. 対象とするインスタンスと Mode "Collect and Analyze Data"を選択します。**Collect Profiler Trace** ボタンを押下して CLR Profiler によるプロファイリングを開始します。※常時接続(Always On)を有効化してくださいといった旨の警告が表示される場合には、本記事末尾のよくあるFAQをご参照ください。
![image-d2521075-7bf6-4848-b05c-9af690aa7827.png]({{site.baseurl}}/media/2023/06/image-d2521075-7bf6-4848-b05c-9af690aa7827.png)
3. "Step2: Reproduce the issue now" に移行した後、プロファイリングが終了するまでの時間 (デフォルトでは 60 秒) 以内に調査対象とする事象を App Service 上で再現します。プロファイリングの取得時間は アプリケーション設定 IIS_PROFILING_TIMEOUT_IN_SECONDS にて変更いただけます。詳細は本記事末尾のよくある FAQ をご確認ください。
![image-0db6ad48-0749-43e5-8714-0d41399b2b30.png]({{site.baseurl}}/media/2023/06/image-0db6ad48-0749-43e5-8714-0d41399b2b30.png)
4. プロファイリングが終了した後、画面中央に表示される Reports リンクから、分析結果を確認します。複数インスタンスに対してプロファイリングを行った場合には、対象とするインスタンスもしくは任意にどれか一つのインスタンスの分析結果を選択します。
![image-14e85ccb-7688-42c3-ae38-17af5858e494.png]({{site.baseurl}}/media/2023/06/image-14e85ccb-7688-42c3-ae38-17af5858e494.png)
5. 分析結果の左上に表示されている "Trace Duration" がデフォルトの 60 秒または、アプリケーション設定 IIS_PROFILING_TIMEOUT_IN_SECONDS で設定した時間(秒) とおおよそ一致することを確認します。想定した取得時間と大幅に異なる場合には、正しくプロファイリングが行えていない可能性があるため、アプリケーション設定 IIS_PROFILING_TIMEOUT_IN_SECONDS の値を改めて確認し、再度手順 3 からプロファイリングを開始します。下図は取得時間を 200 秒にした場合の例です。
![image-6244323f-384a-4892-8735-e5c01c8e62ed.png]({{site.baseurl}}/media/2023/06/image-6244323f-384a-4892-8735-e5c01c8e62ed.png)
6. 分析結果の下部にある "Top 100 Slowest Requests in the Trace" セクションに調査対象とするリクエストが記載されていることを確認します。対象のリクエストが記載されていない場合には、その他のインスタンスの分析結果を確認します。すべてのインスタンスで対象のリクエストが確認できない場合には、取得対象とするインスタンスが正しいことや、プロファイリングの取得時間が十分であるかを確認の上、再度手順 3 からプロファイリングを開始します。
![image-6fa428b6-fc16-4436-8600-80484a66f258.png]({{site.baseurl}}/media/2023/06/image-6fa428b6-fc16-4436-8600-80484a66f258.png)
7. 手順 5. , 6. にて想定したデータであることを確認しましたら、 分析結果上部にある ”Trace File” のリンクからデータを取得します。
![image-5d936830-4a80-434c-a5d7-7c338e89381d.png]({{site.baseurl}}/media/2023/06/image-5d936830-4a80-434c-a5d7-7c338e89381d.png)


# Option2 ) Auto-Heal
1. Azure ポータルで対象の App Service を選択し、左側ブレード [問題の診断と解決] > **Auto-Heal** を選択します。
![image-1f8d9772-bb5e-49de-95a7-ea9a92156a79.png]({{site.baseurl}}/media/2023/06/image-1f8d9772-bb5e-49de-95a7-ea9a92156a79.png)
2. Define Conditions から CLR Profiler の取得条件を入力します。例えば、120 秒以上応答に時間が掛かるリクエストが 300 秒以内に 2 回発生したことを条件にする場合には、Define Conditions に [Request Duration] を選択し、下図のように構成し、[OK] ボタンを押下します。
![image-f6612ec3-92ca-4e3d-b13e-ab99a2371cd0.png]({{site.baseurl}}/media/2023/06/image-f6612ec3-92ca-4e3d-b13e-ab99a2371cd0.png)
3. Configure Actions で [Custom Action] から **CLR Profiler With Thread Stacks** を選択します。プロファイリングが終了したら、プロセスを再起動させる場合には "CollectKillAnalyze" を選択します。条件が入力できたら、[Save] ボタンを押下します。
![image-2e384a44-b772-45ee-861e-fb2212a661f9.png]({{site.baseurl}}/media/2023/06/image-2e384a44-b772-45ee-861e-fb2212a661f9.png)
4. すべて入力が完了したら、[Save] ボタンを押下します。これで条件に合致した場合に自動的にプロファイリングが行われます。なお、プロファイリングの取得時間は規定で 60 秒となっております。プロファイリングの取得時間は アプリケーション設定 IIS_PROFILING_TIMEOUT_IN_SECONDS にて変更いただけます。詳細は本記事末尾のよくある FAQ をご確認ください。
5. プロファイリングが終了した後、SCM サイト (https://<AppService名>.scm.azurewebsites.net/) より \home\data\DaaS\logs 配下にデータが保存されます。これらのデータは、ブラウザーより SCM サイトの URL (https://<AppService名>.scm.azurewebsites.net/api/zip/Data/DaaS/logs/) へアクセスいただくことでまとめてダウンロードできますので、ご状況に応じてご活用ください。

# CLR Profiler で取得したデータを解析する
CLR Profiler で取得したデータは、App Service の解析結果や PerfView を使用して解析することが出来ます。
データの解析方法について参考例をご紹介いたします。

今回ご参考例として、ASP.NET Core 6 アプリケーションに対して、以下のようにループ処理内で Thread.Sleep を実装しています。CLR Profiler のデータを解析し、以下処理にて時間が掛かっている様子を確認します。

```Csharp
    public IActionResult Whileloop()
    {
        _logger.LogInformation($"C# while loop started at: {DateTime.Now}");
        int a = 0;
        while (true)
        {

            _logger.LogInformation($"C# while loop count: {a}");
            a++;
            if (a%10000==0) Thread.Sleep(TimeSpan.FromSeconds(10));
            if (100 * 100 * 100 * 100 < a) break;
        }

        _logger.LogInformation("C# while loop completed.");
        return View();
    }
```

## Option1 ) App Service の解析結果にて確認する方法
Azure ポータルにて対象の App Service を選択後、左側ブレード [問題の診断と解決] > **Collect .NET Profiler Trace** を選択します。

![image-9e1d4d02-86e4-4c8f-9ac5-b2987539d734.png]({{site.baseurl}}/media/2023/06/image-9e1d4d02-86e4-4c8f-9ac5-b2987539d734.png)

**Collect .NET Profiler Trace** を選択した後、表示される画面下部に表示されている "Last 5 profiling sessions" から対象とするデータの Reports のリンクを選択します。直近 5 回に調査対象とするデータが含まれていない場合には、"View all sessions" リンクから対象とするデータのリンクを選択します。

![image-36aa75d5-0d7b-4ee3-a3c3-327624f8e174.png]({{site.baseurl}}/media/2023/06/image-36aa75d5-0d7b-4ee3-a3c3-327624f8e174.png)

Reports のリンクを選択すると、下図のような分析結果が表示されます。応答時間が遅い原因の調査では、"Slow Requests" タブを選択します。"Slow Requests" タブでは、処理に時間が掛かっているモジュールやリクエスト、及び対象リクエストの処理を開始したスレッドIDや Request Id(ContextId) を確認することが出来ます。
下図では、AspNetCoreModuleV2(ASP.NET Core アプリケーション) で 99.99 % ほど時間を要しており、/Home/Whileloop というパスで受け付けたリクエストで 177.66 秒ほど処理が行われたことが分かります。

![image-68a1f0b2-bda1-4e2a-8fe0-207d3919a57a.png]({{site.baseurl}}/media/2023/06/image-68a1f0b2-bda1-4e2a-8fe0-207d3919a57a.png)

次に ".NET Core Slow Requests" タブを選択し、上記で確認したパスのリクエストの "Stack Trace" ボタンを押下します。すると、対象パスで行われた処理のスタックトレースやアプリケーションで出力しているトレース内容が表示され、こちらから対象パス内でどのような処理が行われていたのか確認することが出来ます。下図では、dotnetcore6.Controllers.HomeController.Whileloop メソッドまで 1 秒未満で処理が進みましたが、そこから先の処理はメソッドの変更等は記録されていないので左記メソッド内で何らかの処理が時間を要している状況が推測されます。

![image-e3886774-b1aa-4058-bfee-03988c858e56.png]({{site.baseurl}}/media/2023/06/image-e3886774-b1aa-4058-bfee-03988c858e56.png)

処理がスタックしていると推測されるメソッドが確認されたら、"Thread Callstacks" タブから、対象のメソッドを検索します。
今回の例では、上記の通り dotnetcore6.Controllers.HomeController.Whileloop メソッドを検索すると、対象メソッド内で Thread.Sleep が行われており、これにより応答が遅い事象が発生していることが分かります。

![image-ed86afeb-233a-4ee1-8c3a-a6cac016ff9b.png]({{site.baseurl}}/media/2023/06/image-ed86afeb-233a-4ee1-8c3a-a6cac016ff9b.png)

 
## Option2 ) PerfView にて確認する方法
PerfView は GitHub 上で共同開発、公開されているパフォーマンス解析ユーティリティで、下記よりダウンロードいただくことが出来ます。

[PerfView Release](https://github.com/Microsoft/perfview/releases)
  
 
PerfView.exe (または PerfView64.exe) を実行し、左上のアドレスバーに CLR Profiler にて取得した zip ファイルを展開したフォルダパスを指定します。
対象のフォルダーから diagsession ファイルを開きますと、elt ファイルが取り出され、内部の各種メトリック状況をご確認いただくことが可能です。

調査対象のイベントの確認のため、まずは "Events" をダブルクリックで開きます。
  
![image-b614a0a3-57d9-4f27-95a8-4efe1ba23427.png]({{site.baseurl}}/media/2023/06/image-b614a0a3-57d9-4f27-95a8-4efe1ba23427.png)

アプリケーションログなどから関連と思われるアプリケーションメッセージなどが分かっている場合には、そのメッセージを Text Filter に入力します。その後、Event Types 欄から任意の Type をクリックし、Ctrl+A ですべての Event Types を選択後、Enter を押します。対象となるメッセージが CLR Profiler で取得したデータに含まれている場合には、メッセージを出力したプロセス名と TreadID が確認できます。下図では、プロセス w3wp(1956) と ThreadID(6624) が確認できます。

![image-822c5a72-ce2d-40e6-a429-b1f6b9cfdab1.png]({{site.baseurl}}/media/2023/06/image-822c5a72-ce2d-40e6-a429-b1f6b9cfdab1.png)

"Events" から調査対象とするプロセスと ThreadID を確認した後、"Thread Time (with StartStop Activities) Stacks" をダブルクリックで開きます。

![image-e0a84a60-ea59-460e-8397-ffa7786e0a6e.png]({{site.baseurl}}/media/2023/06/image-e0a84a60-ea59-460e-8397-ffa7786e0a6e.png)

先ほど確認したプロセス w3wp(1956) をクリックし、"OK" ボタンを押下します。

![image-83e84840-758f-4ee6-91fb-ac4495fc0061.png]({{site.baseurl}}/media/2023/06/image-83e84840-758f-4ee6-91fb-ac4495fc0061.png)

内容が表示されますので、[CallTree] タブへ移動して、先ほど確認した ThreadID のスレッドにチェックを入れます。
その配下の関数のスタックトレースが表示されますので、繰り返しチェックを入れていきます。
十数回ドリルダウンしていくと、対象となるコードが表示されます。下図例では、dotnetcore6.Controllers.HomeController.Whileloop メソッドにて Thread.Sleep が行われており、これにより 150 秒ほど要していることが分かります。
  
![image-280737b6-d4cd-48ca-8352-2d650060ebe0.png]({{site.baseurl}}/media/2023/06/image-280737b6-d4cd-48ca-8352-2d650060ebe0.png)
 
PerfView を使用した解析の詳細については、PerfView のドキュメント並びにチュートリアルをご確認いただけますと幸いです。

[GitHub - PerfView Project](https://github.com/Microsoft/perfview)
[Channel9 - PerfView Tutorial](https://channel9.msdn.com/Series/PerfView-Tutorial)

# よくある FAQ
## CLR Profiler の取得時間を変更する方法はありますか？
CLR Profiler の取得時間は、デフォルトで 60 秒です。アプリケーション設定 IIS_PROFILING_TIMEOUT_IN_SECONDS により最大 900 秒まで取得時間を変更することが出来ます。下図は取得時間を 200 秒に設定した例です。

![image-684e6c4f-8fd8-4d80-ba14-d3e516a9280a.png]({{site.baseurl}}/media/2023/06/image-684e6c4f-8fd8-4d80-ba14-d3e516a9280a.png)

[参考: App Service Diagnostics – Profiling an ASP.NET Web App on Azure App Service](https://azure.github.io/AppService/2018/06/06/App-Service-Diagnostics-Profiling-an-ASP.NET-Web-App-on-Azure-App-Service.html#:~:text=It%20is%20possible%20to%20increase%20the%20default%20profiling%20duration%20by,to%20the%2060%20second%20value)

[参考: Azure Web Apps の .NET Profiler の取得時間を変更する方法について](https://jpdsi.github.io/blog/web-apps/webapps-dotnet-profiler/#%E5%8F%96%E5%BE%97%E6%99%82%E9%96%93%E3%81%AE%E5%A4%89%E6%9B%B4)

## CLR Profiler で取得したデータはどこに保存されますか？どのように取得すればよいですか？
CLR Profiler で取得したデータは App Service のファイルサーバー \home\data\DaaS\logs 配下に保存されます。

CLR Profiler で取得したデータ及び、App Service による分析結果のレポートは、SCM サイトの URL (https://<AppService名>.scm.azurewebsites.net/api/zip/Data/DaaS/logs/) からまとめてダウンロードいただけます。

## CLR Profiler の取得対象とするインスタンスはどこから確認できますか？
メトリックの分割機能をご活用いただければと存じます。App Service プランの CPU Percentage や App Service の Response Time、Average memory working set といったメトリックはインスタンスごとに分割することができ、この機能を使用して対象となるインスタンスをご確認いただけます。

例えば、App Service の応答時間が遅い原因を調査する場合には、対象の App Service を選択し、[メトリック] ブレード > Response Time にて [分割を適用する] ボタンよりインスタンスごとに応答時間が表示されます。

![image-2a0617ef-81ef-4cca-98cb-1b3c4f8277c9.png]({{site.baseurl}}/media/2023/06/image-2a0617ef-81ef-4cca-98cb-1b3c4f8277c9.png)

こちらのメトリック結果をご参考に調査対象とするインスタンスをご確認ください。

![image-5a09be3f-0b69-4649-9d80-cf990d868c0f.png]({{site.baseurl}}/media/2023/06/image-5a09be3f-0b69-4649-9d80-cf990d868c0f.png)

[参考: Azure Monitor のメトリック グラフの例 - Azure Monitor](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/metric-chart-samples#website-cpu-utilization-by-server-instances)

[参考: Azure Monitor メトリックによる集計と表示の説明 - Azure Monitor](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/metrics-aggregation-explained)

## 警告が表示され **Collect Profiler Trace** ボタンが表示されない場合はどうしたらよいですか？
 [問題の診断と解決] > **Collect .NET Profiler Trace** 画面で以下のような警告が表示された場合には、Azure ポータルから対象 App Service を選択し、左側ブレード [構成] > 全般設定 タブから 常時接続(Always On) を有効化ください。
常時接続は、20 分以上リクエストを受信していない場合にも、App Service 上でお客様のアプリケーションを読み込まれたままにする機能です。これにより長期間リクエストの受付が無い場合にも、要求の待機時間を短くすることが出来ます。

左記手順の詳細については[こちら](https://learn.microsoft.com/ja-jp/azure/app-service/configure-common?tabs=portal#configure-general-settings)を
ご参照ください。 

>Error - We determined that the web app does not have Always-On enabled and due to these the diagnostic tools will not work reliably with AutoHeal. Please turn on the Always-On setting by going to the Application Settings for the web app and then run these tools.
>![image-0ea51a31-4e0f-4161-9009-1e1b5bb57125.png]({{site.baseurl}}/media/2023/06/image-0ea51a31-4e0f-4161-9009-1e1b5bb57125.png)


# 参考ドキュメント
- [App Service Diagnostics – Profiling an ASP.NET Web App on Azure App Service](https://azure.github.io/AppService/2018/06/06/App-Service-Diagnostics-Profiling-an-ASP.NET-Web-App-on-Azure-App-Service.html)
- [Azure Auto-heal – Azure Advice](https://azure-advice.com/2020/10/23/azure-auto-heal/)
- [Azure Web Apps でオンプレ IIS と同じような情報は取れるのか？](https://jpdsi.github.io/blog/web-apps/webapps-diagnostic-tools/#3-Azure-Web-Apps-%E3%81%A7-NET-%E3%81%AE-Profiler-%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9-ETW-%E3%81%A3%E3%81%A6%E5%8F%96%E3%82%8C%E3%82%8B%E3%81%AE%EF%BC%9F)

<br>
<br>
---
<br>
<br>
2023 年 06 月 28 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
<br>
<br>
