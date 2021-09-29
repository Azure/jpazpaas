---
title: "App Service における SNAT ポート枯渇問題とその解決方法"
author_name: "Yoichi Ozaki"
tags:
    - App Service
    - Web Apps
    - Function App
---

Azure App Service はアプリを簡単に構築・デプロイ・管理することができる PaaS（Platform as a Service）です。

App Service についてのお問い合わせとして「少数のリクエストをさばくテスト運用時には問題は発生しなかったが、大量のリクエストを受ける本番運用時にウェブアプリが不調になった」というものがあります。

そこで、本記事では原因の一つとして考えられる SNAT ポート枯渇問題について、問題の概要から解決策までご案内いたします。

# App Service における SNAT ポート枯渇問題

App Service にてデプロイされたウェブアプリケーションが、 SQL データベースや Redis キャッシュ、外部の RESTful なウェブサービスなどいくつかの外部サービスと接続する必要があるということはよくあります。しかし、 App Service では、ウェブアプリケーションが外部のサービスと直接コネクションを確立することはできません。

これは、App Service ワーカーインスタンスは必ずロードバランサー経由で外部と通信をする必要があるからです。 App Service にてデプロイされたウェブアプリケーションは一つ、もしくはいくつかの App Serivce ワーカーインスタンスによってホストされています。 ワーカーインスタンスは Scale Unit という単位でまとめられています。 ワーカーインスタンスはインターネットに公開された IP アドレスを割り振られておらず、外部の IP アドレス宛の通信を行うためには、 Scale Unit ごとに用意されたロードバランサを利用して Source Network Address Translation（SNAT） を行う必要があります。

SNAT を行うロードバランサは、ある Scale Unit 内のウェブアプリケーション、WebJobs、Functions、テレメトリサービス（Application Insights）などを含むすべての App Service サイトで共用するリソースです。よって、あるアプリケーションが Scale Unit 外部のエンドポイントと大量のコネクションを張ると SNAT ポートが不足して通信できなくなってしまいます。

SNAT ポートの枯渇を回避するためには、 1 インスタンスあたりのコネクションの数が **128 を超えないようにする**ことが必要であり、これを守る限りロードバランサーが SNAT のポート枯渇を理由にウェブアプリケーションが外部エンドポイントに接続するのをブロックすることはありません。

- [https://docs.microsoft.com/ja-jp/azure/app-service/troubleshoot-intermittent-outbound-connection-errors#cause](https://docs.microsoft.com/ja-jp/azure/app-service/troubleshoot-intermittent-outbound-connection-errors#cause)

>  Azure アプリ サービスの各インスタンスには、当初、128 個の SNAT ポートが事前に割り当てられています。


SNAT の詳しい説明や SNAT ポートの割り当てアルゴリズムの挙動などについては、弊社エンジニアが執筆いたしました下記の記事や公式ドキュメントをご参照ください。

- [https://4lowtherabbit.github.io/blogs/2019/10/SNAT/](https://4lowtherabbit.github.io/blogs/2019/10/SNAT/)
- [https://docs.microsoft.com/ja-jp/azure/load-balancer/load-balancer-outbound-connections#default-port-allocation](https://docs.microsoft.com/ja-jp/azure/load-balancer/load-balancer-outbound-connections#default-port-allocation)

以下では、SNAT ポートが枯渇するアプリケーションを実際に用意して問題を再現することで、SNAT ポートが不足した時にウェブアプリケーションに現れる症状や問題を回避するためのアプリケーションの実装例を見ていきます。

# SNAT ポート枯渇問題の再現

SNAT ポート枯渇問題を再現するために次の 2 つのウェブアプリを用意し、各 App Service へデプロイを行います。

- `StarveSnatWebAppEastAsia`
- `DelayReplyCentralUs`

`StarveSnatWebAppEastAsia` が今回 SNAT ポートを大量に消費するアプリケーションです。`StarveSnatWebAppEastAsia` は次のように動作します。

- `GET /snat/UseClient?url=<url>`
    - `HttpClient` をコネクションごとに生成・破棄しながら `<url>` にアクセスする。アクセスに成功したら `"OK"` と返す。
- `GET /snat/ReuseClient?url=<url>`
    - `HttpClient` を複数のコネクションに対して使いまわしながら `<url>` にアクセスする。アクセスに成功したら `"OK"` と返す。

`DelayReplyCentralUs` は次のように動作します。

- `/Delay?seconds=N`：`N` 秒間の `Sleep()` 後、`"OK"` と返す。

これらを利用することで、アプリケーションが Scale Unit 外との通信を行うシナリオを再現できます。なお、`StarveSnatWebAppEastAsia` と `DelayReplyCentralUs` が異なる Scale Unit にデプロイされることを保証するために、それぞれ異なるリージョンにアプリをデプロイします。

![image.png]({{site.baseurl}}/media/2021/09/2021-09-30-image-090da467-030d-4268-a1df-88f1f647b1da.png)

`StarveSnatWebApp` へ同時に大量のアクセスを行うことで SNAT ポートを消費させ問題を再現することができます。今回は次のようなコマンドで実行しました。

```powershell
PS> # ForEach-Object -Parallel コマンドを使用（PowerShell version 7~ 提供）
PS> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      7.0.7
PSEdition                      Core
GitCommitId                    7.0.7
OS                             Microsoft Windows 10.0.19042
Platform                       Win32NT
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0

PS> # クライアントを毎回生成・200並列で合計1000アクセス
PS> 1..1000 | ForEach-Object -Parallel { curl "https://<StarveSnatWebAppEastAsia のエンドポイント>/snat/UseClient?url=https://<DelayReplyCentralUs のエンドポイント>/delay?seconds=5"; } -ThrottleLimit 200 2> $null;

PS> # クライアントを再利用・200並列で合計1000アクセス
PS> 1..1000 | ForEach-Object -Parallel { curl "https://<StarveSnatWebAppEastAsia のエンドポイント>/snat/ReuseClient?url=https://<DelayReplyCentralUs のエンドポイント>/delay?seconds=5"; } -ThrottleLimit 200 2> $null;
```

# ウェブアプリに現れる症状

SNAT ポートが枯渇すると、当該アプリケーションにて `SocketException` が発生します。当該アプリの `Application Insights > Application map` からアプリケーションがどことどういう通信をしたかが表示されます。

![image.PNG]({{site.baseurl}}/media/2021/09/2021-09-30-Slide8-61bd38a3-e64f-4a9f-aa4b-ffa5e6b3b35a.PNG)

失敗した通信があれば次の画像に示すような表示になります。赤い表示でエラーの発生が見て取れます。

![image.PNG]({{site.baseurl}}/media/2021/09/2021-09-30-Slide6-6b14bc2d-d300-4f58-b6b8-b558c2daa127.PNG)

さらにマップ上のアプリケーションに相当する部分をクリックすることでさらに詳細を確認することもできます。

![image.PNG]({{site.baseurl}}/media/2021/09/2021-09-30-Slide9-2d1158eb-92fd-4c25-a08e-429aad12c461.PNG)

また、Azure Portal から`問題の診断と解決 > 「SNAT」と検索 > 「SNAT ポートの枯渇」`で SNAT ポートに関わる問題発生の有無や割り当て済みポートと使用済みポートの時系列推移のグラフを見ることができます。

![image.PNG]({{site.baseurl}}/media/2021/09/2021-09-30-Slide10-088dd600-d3c0-4ef4-a4d7-bc4604ad4ad6.PNG)

SNAT ポートの枯渇が検出された場合には、この画面の上部に通知されます。SNAT ポートの使用状況のグラフからおおよその SNAT ポートの割り当ての挙動が見て取れます（最初に 128 ポートがアプリに割り当てられて以降、要求に応じてベストエフォート）。一方で実際の割り当ての動作と比べてプロットの粒度が荒いため、グラフを観察することで SNAT ポートの枯渇が起こったのかを判断することはできません。上部の通知を確認しましょう。

![image.PNG]({{site.baseurl}}/media/2021/09/2021-09-30-Slide3-4b4080d4-fe59-4e7a-9820-fd951d3bd022.PNG)

![image.png]({{site.baseurl}}/media/2021/09/2021-09-30-Presentation1-1f8514af-02f9-4601-a976-ec137a225a30.png)

# SNAT ポート枯渇問題を回避する実装

SNAT ポート枯渇問題を解決する方法はいくつか考えられますが、アプリケーションの実装を見直すことで解決する可能性もございます。

例えば、 C# で書かれた以下のようなコードでは、`HttpClient`がコネクションごとに生成・破棄が行われ、 SNAT ポートを多く消費してしまいます。

```csharp
public class SnatController : Controller
{
    public async Task<string> UseClient(string url)
    {
        // usingでくくることでHttpClientがリクエストごとに生成される
        using (var client = new HttpClient())
        {
            await client.GetAsync(url);
        }

    return "OK";
    }
}
```

一方で、次のように一つの `HttpClient` インスタンスを使いまわすことで SNAT ポートの消費を抑えることができます。

```csharp
public class SnatController : Controller, IDisposable
{
    private static readonly HttpClient _client;
    static SnatController()
    {
        _client = new HttpClient();
    }
    // 複数のコネクションに対して_clientを使いまわす
    public async Task<string> ReuseClient(string url)
    {
        await _client.GetAsync(url);
        return "OK";
    }
    
    public new void Dispose()
    {
        _client.Dispose();
    }
}
```

C# の REST API についてのアプリケーションの詳細な実装方法は、下記の記事にて参考いただけます。

- [https://docs.microsoft.com/ja-jp/dotnet/api/system.net.http.httpclient?view=net-5.0](https://docs.microsoft.com/ja-jp/dotnet/api/system.net.http.httpclient?view=net-5.0)
    - `HttpClient` についての弊社公式ドキュメントになります。
    - `HttpClient` の使用法についての一次情報としてご活用ください。
- [https://qiita.com/superriver/items/91781bca04a76aec7dc0](https://qiita.com/superriver/items/91781bca04a76aec7dc0)
    - 弊社エンジニアが執筆を行っております。
    - `HttpClient` を使う際の実装上の注意点について解説されています。
- [https://blog.nnasaki.com/entry/2019/10/04/143936](https://blog.nnasaki.com/entry/2019/10/04/143936)
    - Microsoft MVP を取得しているエンジニアより執筆しております。
    - .NET Core で `HttpClientFactory` を使う方法が解説されています。

# まとめ

本記事では、実際に SNAT ポートを枯渇させるアプリケーションを動かし、アプリケーションに現れる症状とその対策の一例をお見せしました。

SNAT ポートの枯渇は、アプリケーションへの大量のアクセスが起こってから顕在化することが多く、少数のアクセスを行うテスト段階では気が付きにくいです。

本記事を参考に、皆様が問題を自分で発見し、解決のきっかけになれば幸いです。

# 参考ドキュメント・記事

- SNAT そのものの挙動やポート割り当てのアルゴリズムが説明されています。
    - [https://4lowtherabbit.github.io/blogs/2019/10/SNAT/](https://4lowtherabbit.github.io/blogs/2019/10/SNAT/)
- SNAT に関わる公式ドキュメント
    - [https://docs.microsoft.com/azure/app-service/troubleshoot-intermittent-outbound-connection-errors](https://docs.microsoft.com/azure/app-service/troubleshoot-intermittent-outbound-connection-errors)
    - [https://docs.microsoft.com/azure/load-balancer/load-balancer-outbound-connections#preallocatedports](https://docs.microsoft.com/azure/load-balancer/load-balancer-outbound-connections#preallocatedports)
    - [https://docs.microsoft.com/azure/azure-functions/manage-connections?tabs=csharp](https://docs.microsoft.com/azure/azure-functions/manage-connections?tabs=csharp)
    - [https://docs.microsoft.com/azure/load-balancer/troubleshoot-outbound-connection](https://docs.microsoft.com/azure/load-balancer/troubleshoot-outbound-connection)
    - [https://docs.microsoft.com/azure/load-balancer/outbound-rules](https://docs.microsoft.com/azure/load-balancer/outbound-rules)
    - [https://docs.microsoft.com/azure/load-balancer/load-balancer-standard-diagnostics](https://docs.microsoft.com/azure/load-balancer/load-balancer-standard-diagnostics)
    - [https://azure.microsoft.com/blog/azure-load-balancer-to-become-more-efficient/](https://azure.microsoft.com/blog/azure-load-balancer-to-become-more-efficient/)
- C# での `HttpClient`/`HttpClientFactory` の使い方についての解説記事
    - [https://qiita.com/superriver/items/91781bca04a76aec7dc0](https://qiita.com/superriver/items/91781bca04a76aec7dc0)
    - [https://blog.nnasaki.com/entry/2019/10/04/143936](https://blog.nnasaki.com/entry/2019/10/04/143936)


<br>
<br>

---

<br>
<br>

2021 年 9 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
