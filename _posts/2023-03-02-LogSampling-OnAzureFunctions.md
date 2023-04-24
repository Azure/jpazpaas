---
title: "Azure Functions のログサンプリング"
author_name: "Hayato Kuroda"
tags:
    - Function App
---

# 質問
Azure Functions のログが欠落しています。解消方法を教えてください。

# 回答
Azure Functions では[既定でアダプティブ ログサンプリングが有効化](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-monitoring#application-insights-pricing-and-limits)
されているため、ログ データが Application Insights へ送られる場合にサンプリングされて送られるためログが欠落する場合があります。これはご利用のプランなどにはよらず、host.json の設定に従います。
![image-a3fa9763-0222-4dcc-839a-6a2795a052b3.png]({{site.baseurl}}/media/2023/03/image-a3fa9763-0222-4dcc-839a-6a2795a052b3.png)

## サンプリング設定
host.json に設定されている箇所は下記の [samplingSettings](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-host-json#applicationinsightssamplingsettings) の isEnabled 項目になります。

```
{
    "version": "2.0",
    "logging": {
      "applicationInsights": {
        "samplingSettings": {
          "isEnabled": true,
          "excludedTypes": "Request"
        }
      }
    }
}
```

![image-a4dd5268-da01-4312-af72-569002a06cb4.png]({{site.baseurl}}/media/2023/03/image-a4dd5268-da01-4312-af72-569002a06cb4.png)

何をもってサンプリングが発生するかの仕組みについては[アダプティブ サンプリング](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/sampling?tabs=net-core-new#adaptive-sampling)に解説があります。送信するテレメトリデータの件数と、host.json に設定した MaxTelemetryItemsPerSecond の値によって調整される動作となります。

![image-247e3893-4783-41a5-8efb-6cf86b61be4d.png]({{site.baseurl}}/media/2023/03/image-247e3893-4783-41a5-8efb-6cf86b61be4d.png)


## サンプリング発生時のトラブルシューティング
サンプリングが起きた場合には下記のように一部のログが欠落して Application Insights で確認できます。
今回サンプルで利用したアプリケーションでは、0 から 999 までをカウントしてログに出力しています。operation_Id は 1 回の処理単位を表しますが、operation_Id `1627e2c8572fb4dc334ca8320843ea8d` の場合には `Current Number: 342` 以降のログが出力されておりません。operation_Id `5a495d1e4f1d173ff532943de6e85b7d` の場合には `Current Number: 473` 以前のログが出力されておりません。

![image-f52b2ab2-985c-4dc7-9d8c-5fba7d4a330b.png]({{site.baseurl}}/media/2023/03/image-f52b2ab2-985c-4dc7-9d8c-5fba7d4a330b.png)

また、ログ サンプリングが発生した際には Application Insights に記録されるログの itemCount 列が 1 よりも大きな値を持ちます。itemCount は、そのレコードがもつべきであったログの数を示すためです。[Log Analytics での要求](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/api-custom-events-metrics#requests-in-log-analytics)に解説があります。前述している Aplication Insights の出力結果を例にすると本来 68 件のログがあったものの、現在確認できている `Current Number: 473` のログ 1 件のみが出力されていると判断いただくことができます。この場合、Application Insights に送られなかった 67 件のログを後から確認いただくことはできません。

![image-258c0d7f-3c3c-4d2e-a505-16a9c4bd4b37.png]({{site.baseurl}}/media/2023/03/image-258c0d7f-3c3c-4d2e-a505-16a9c4bd4b37.png)

## ログ サンプリングが発生した場合の確認と対処方法
サンプリング発生有無の確認のために、次のクエリを Application Insights で実行いただくとサンプリングの発生有無を参照できます。時間の粒度を調整いただいたり、グラフを描画いただくとより細かい単位でサンプリングのレートが把握できます。[サンプリングが動作しているかどうかを把握する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/sampling?tabs=net-core-new#knowing-whether-sampling-is-in-operation)

```
//クエリ例1
union requests,dependencies,pageViews,browserTimings,exceptions,traces
| where timestamp > ago(1d)
| summarize RetainedPercentage = 100/avg(itemCount) by bin(timestamp, 1h), itemType

//クエリ例2
union requests,dependencies,pageViews,browserTimings,exceptions,traces
| where timestamp >= datetime("2023-03-01 07:09")
| summarize RetainedPercentage = 100/avg(itemCount) by bin(timestamp, 1sec), itemType
| render timechart 
```

クエリ例2 の実行結果では下記のようなグラフが描画されます。request は host.json にて excludedTypes(サンプリングをしない対象) に設定しているためサンプリングされていません。traces はピーク時は全体の約 1% のログのみが記録されていることがわかります。
![image-93cad1f0-86f0-4128-ae9d-3657c95b7faa.png]({{site.baseurl}}/media/2023/03/image-93cad1f0-86f0-4128-ae9d-3657c95b7faa.png)


サンプリング動作の解消のためには host.json にてサンプリングを無効化し、再デプロイを実施いただくもしくは App Service のアプリケーション設定から [host.json のオーバーライド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-host-json#override-hostjson-values)をする必要があります。

```
{
    "version": "2.0",
    "logging": {
      "applicationInsights": {
        "samplingSettings": {
          "isEnabled": false,
          "excludedTypes": "Request"
        }
      }
    }
}
```

![image-c989a4e3-7ef1-4084-8850-9e839ecc35f5.png]({{site.baseurl}}/media/2023/03/image-c989a4e3-7ef1-4084-8850-9e839ecc35f5.png)

また、現在のサンプリングの設定有無を確認するには、host.json の確認とともに traces テーブルにて起動時のコンフィグレーションを確認いただくこともできます。Azure Functions では、起動時に host.json の読み込みを行い、各種オプションの反映内容のログを出力します。ApplicationInsightsLoggerOptions が対象となっており、黄ハイライト部分は、サンプリングが有効化されていることを表します。SamplingSettings の要素があることが確認でき、host.json でサンプリングを有効化しているのみであれば、[既定の値](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-host-json#applicationinsightssamplingsettings)が読み込まれています。緑ハイライト部分は、サンプリングが無効化されいていることを表します。

```
traces 
| where message startswith "ApplicationInsightsLoggerOptions"
| order by timestamp asc
```

![image-e452b159-6577-4423-b684-51b5d9172753.png]({{site.baseurl}}/media/2023/03/image-e452b159-6577-4423-b684-51b5d9172753.png)

以上が Azure Functions におけるログ サンプリングの解説となります。Azure Functions を作成したけど、ログが出力されないなどトラブルシューティングにお役に立てば幸いです。

<br>
<br>

---

<br>
<br>

2023 年 04 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>