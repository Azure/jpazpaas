---
title: "App Service Planのファイル システム ストレージ 使用量を監視する方法について"
author_name: "Yutaro Inoue"
tags:
    - App Service
    - Web Apps
---

# 質問
App Service Plan のファイル システム ストレージ(*)の使用量を監視（使用率上昇の検知及びアラートを含む）する方法について知りたい。

(*)ファイル システム ストレージ\
・App Service Plan の「ファイル システム ストレージ」ブレードよりご確認いただける情報を指します。
![image.png]({{site.baseurl}}/media/2021/09/2021-09-01-filesystemstorage-blade.png)


# 回答
現時点で、ASE (App Service Environment) を除き、File System Usage メトリックを監視いただく方法は提供されておりません。ASE 以外をご利用の場合には REST API などでの代用をご検討ください。

（ご参考）[File System Usageについて](https://docs.microsoft.com/ja-jp/azure/app-service/web-sites-monitor#understand-metrics)
~~~
ファイル システムの使用量 は、グローバルにロールアウトされている新しいメトリックです。アプリが App Service Environment でホストされていない場合、データは表示されません。
ファイル システムの使用量：ストレージ共有別の使用量 (バイト単位)。
~~~

# 詳細
ファイル システム ストレージを監視する方法は以下の通りです。

## 1) ASE をご利用の場合
メトリック ”File System 
Usage” にて使用バイト量の情報が取得可能ですので、当該メトリックに対してアラートを構成することができます。アラートの構成方法は、以下資料をご参照ください\
（ご参考）[メトリックアラート機能の概要](https://docs.microsoft.com/ja-jp/azure/azure-monitor/alerts/alerts-metric-overview)\
（ご参考）[メトリックをもとにしたアラートの構成方法](https://docs.microsoft.com/ja-jp/azure/azure-monitor/essentials/metrics-charts#alert-rules)

## 2) ASE 以外をご利用の場合
メトリック情報を取得する API を実行することで、当該情報の取得が可能です。API を利用した監視方法については、2 通りの方法があります。

（ご参考）[App Service Plans - List Usages (Azure App Service)](https://docs.microsoft.com/en-us/rest/api/appservice/appserviceplans/listusages)\
・API の戻り値にファイル システム ストレージの合計使用量の情報が含まれており、“FileSystemStorage” という項目で返却されます。

＜APIより取得できる情報のサンプル＞
~~~
{
  "value": [
   ....
    {
      "unit": "Bytes",
      "nextResetTime": "9999-12-31T23:59:59.9999999Z",
      "currentValue": 640584704,
      "limit": 327491256320,
      "name": {
        "value": "FileSystemStorage",
        "localizedValue": "File System Storage"
      }
    },
   ....
  ],
  "nextLink": null,
  "id": null
}
~~~

### ■案１：Azure Monitor のカスタムメトリックを作成し、メトリックアラートを構成する
上記 API で、定期的にファイル使用量を取得し、Azure Monitor の “カスタム メトリック” という機能を利用することで、取得したファイル使用量を元に独自のメトリックを作成することができます。
カスタムメトリックは通常のメトリックと同じように利用できますので、Azure Monitor からアラートを設定できます。ただし、カスタムメトリックの実装には以下の一連の処理を定期的に実行する仕組みを独自に用意する必要があります。

①データ取得 (ファイルシステムストレージの使用量)\
②Azure Monitor が求めるデータフォーマットへの加工 (JSON)\
③Azure Monitor へカスタムメトリックデータの送信

以下はカスタムメトリックの参考資料です。 \
（ご参考）[Azure Monitor のカスタム メトリック](https://docs.microsoft.com/ja-jp/azure/azure-monitor/platform/metrics-custom-overview)\
（ご参考）[REST API を使用して Azure リソースのカスタム メトリックを Azure Monitor メトリック ストアに送信する](https://docs.microsoft.com/ja-jp/azure/azure-monitor/platform/metrics-store-custom-rest-api)

 
### ■案２：Azure Functions の Timer トリガー関数を利用して定期的にメトリック取得 API を実行し、閾値によりアラート(メール等)を発報するロジックを構築する
Azure Functions の Timer トリガー関数を利用することで、定期的に処理を呼び出すことができます。Timer トリガー関数の処理内に "メトリック取得 API の実行"、及び "FileSystemStorage 値によってアラート(メール等)を発報する" ロジックを実装します。メール発報には、SendGrid をご利用いただけます。\
（ご参考）[Azure Functions における SendGrid のバインディング](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-sendgrid?tabs=csharp)

また、"メトリック取得 API を定期的に実行する仕組み"、及び "閾値に応じてアラートを発報する仕組み" をすべて自前で構築いただくことも可能です。最も負担の高い方法とはなりますが、カスタマイズの幅が広くなるため、監視要件によっては適した方法となります。

以上が、App Service Plan のファイル システム ストレージ使用量を監視する方法のご紹介です。

最後になりますが、ファイル システム ストレージの使用量を ASE に限らず、メトリックで利用できる機能改善を多くいただいております。開発部門にて鋭意検討が続けられておりますので、今後のアップデートにご期待ください。
 
（ご参考）[Metric and alert for Web App file system used quota](https://feedback.azure.com/forums/169385-web-apps/suggestions/19958668-metric-and-alert-for-web-app-file-system-used-quot)


---

<br>
<br>

2021 年 9 月 1 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>
