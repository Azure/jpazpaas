---
title: "App Service の組み込みの診断機能により出力されるログのタイムゾーンを現地時間で出力したい"
author_name: "Yuna Sugimoto"
tags:
    - App Service
---

# 質問
App Service の組み込みの診断機能により出力される以下ログのタイムゾーンを UTC から JST (現地時刻) で出力したい
- アプリケーションのログ
- Web サーバーのログ
- 詳細なエラー メッセージ
- 失敗した要求トレース
- デプロイ ログ

[<ご参考: 診断ログの有効化 - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#overview)


# 回答
現時点では App Service の組み込みの診断機能の診断ログ (Web サーバー ログ など) を現地時間で出力するといった機能はご提供がございません。
ワークアラウンドとして以下 2 点をご提案いたします。
- a ) Log Analytics を使用して診断ログを表示する際のタイムゾーンを現地時間に変更する
- b ) 診断ログの代わりに、お客様のアプリケーション内で独自にログを出力する

以下、それぞれ順にご紹介いたします。ご要件に合わせてご検討いただければ幸いです。

## a ) Log Analytics を使用して診断ログの表示時刻を変更する
Web サーバー ログなどの一部の診断ログは、[Diagnostic settings (診断設定)] より Azure Monitor の Log Analytics に転送することが出来ます。Log Analytics に転送後、タイムゾーンを UTC から 現地時間 に変更してログを表示することが出来ます。
診断設定による Azure Monitor への転送がサポートされるログの種類は以下ドキュメントをご参照ください。

[<ご参考: サポートされるログの種類 - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#supported-log-types)


### 1 ) App Service のログを Log Analytics へ送信
Azure ポータルより対象の App Service を選択後、[Diagnostic settings (診断設定)] ブレードより、ご希望の App Service ログを Log Analytics へ送信します。Log Analytics への送信手順と、送信いただけるログの種類の詳細は、以下ドキュメントに記載がございますため、ご参照ください。

[<ご参考: 診断ログの有効化 - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#send-logs-to-azure-monitor)

![image-0e06e1b7-7e69-4a83-bd83-f1a3cdd17949.png]({{site.baseurl}}/media/2022/12/image-0e06e1b7-7e69-4a83-bd83-f1a3cdd17949.png)

### 2 ) ログの表示時刻を変更する
Log Analytics へログを送信後に、下図のように Azure ポータルより [ログ] ブレードにて対象のログのクエリを実行し、[時刻の表示] を現地時刻へ変更いただくことで、JST にてログをご確認いただけます。

![image-6dd510da-5f03-444f-a037-06570b379429.png]({{site.baseurl}}/media/2022/12/image-6dd510da-5f03-444f-a037-06570b379429.png)

## b ) 開発者様が独自のログを作成する方法
App Service では、オンプレミス環境のアプリケーションと同様にお客様のアプリケーションコードからログ等をファイル出力いただくことが出来ます。
お客様のアプリケーションのコード上でログを出力するよう構成いただくことで、ローカル時刻も含めて記録する独自のログを作成いただけます。

ご参考までに、C# ですとお客様のアプリケーションにて以下のようなコードを実装いただくことで JST で日時を取得しログファイルへメッセージを出力できます。

```csharp
    // JST で日時を取得します。
    var day = DateTime.Now;
    TimeZoneInfo tst = TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time");
    DateTime tokyoTime = TimeZoneInfo.ConvertTime(day, TimeZoneInfo.Local, tst);

    // ログの出力先となるパスを取得します。今回は %HOME% フォルダー直下としています。
    var homePath = Environment.GetEnvironmentVariable("HOME");

    // ログを出力します
    File.AppendAllText($@"{homePath}\test.log", "Tokyo Time:"+tokyoTime + Environment.NewLine);
```

実際に上記コードを実行すると、App Service のファイルシステム上の %HOME% フォルダー直下に test.log ファイルが出力されます。

![image-d78ec243-e790-49ff-9190-85defaf0965c.png]({{site.baseurl}}/media/2022/12/image-d78ec243-e790-49ff-9190-85defaf0965c.png)

### 注意事項） ログの出力先について
App Service でパッケージからの実行が有効である (WEBSITE_RUN_FROM_PACKAGE が true または 1) とき、お客様のアプリケーションが配置されている wwwroot ディレクトリ配下が読み取り専用となります。そのため、当該機能が有効である場合に wwwroot ディレクトリ配下をログの出力先として指定すると、ログの書き込みエラーが発生いたします。

[<ご参考: ZIP パッケージからアプリを実行する - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/deploy-run-package#troubleshooting)

> パッケージから直接実行すると、wwwroot は読み取り専用になります。 アプリがこのディレクトリにファイルを書き込もうとすると、エラーが発生します。

コンテンツ共有 (%HOME%) ディレクトリはお客様のアプリケーションから書き込みが可能となりますため、ログの書き込みエラーが発生された場合には、コンテンツ共有 (%HOME%) ディレクトリをログの出力先としてをご検討ください。コンテンツ共有 (%HOME%) ディレクトリの詳細は、以下ドキュメントをご参照ください。

[<ご参考: オペレーティング システムの機能 - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/operating-system-functionality#file-access-across-multiple-instances)
>コンテンツ共有 (%HOME%) ディレクトリにはアプリのコンテンツが含まれ、アプリケーション コードからの書き込みが可能です。 
>(中略)
>アップロードされたファイルがアプリによって %HOME% ディレクトリに保存されると、それらのファイルはすぐにすべてのインスタンスから利用できるようになります。




# よくある質問
## App Service の組み込みの診断機能の診断ログ (Web サーバー ログ など) のタイムスタンプのタイムゾーンを "WEBSITE_TIME_ZONE" により変更できますか？
WEBSITE_TIME_ZONE 設定は、App Service が動作するインスタンス (仮想マシン) のタイムゾーンを変更できますが、これにより診断ログのタイムスタンプは変更されません。

[<ご参考: Web Appsの構成と管理に関する FAQ - Azure | Microsoft Learn>](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/web-apps-configuration-and-management-faqs#how-do-i-set-the-server-time-zone-for-my-web-app)
> Web アプリのサーバー タイム ゾーンを設定するには、次の手順に従います。
> [アプリ設定] で次の設定を追加します。
> 
> Key = WEBSITE_TIME_ZONE
>
> Value = 目的のタイム ゾーン

WEBSITE_TIME_ZONE 設定は、下記ドキュメントに記載のように Web ジョブの CRON 式などで基準とするタイムゾーンを変更する際に利用されます。しかしながら、この設定により App Service 基盤にてご用意しているログの出力日時を変更いただけませんため、もし現地時刻によるログの出力をご希望でしたら上述のワークアラウンド (Log Analytics を使用して診断ログの表示時刻を変更する方法または、独自ログを出力する方法) のご利用をご検討ください。

[<ご参考: Web ジョブでバックグラウンド タスクを実行する - Azure App Service | Microsoft Learn>](https://learn.microsoft.com/ja-jp/azure/app-service/webjobs-create#ncrontab-expressions)

> CRON 式の実行に使用される既定のタイム ゾーンは、協定世界時 (UTC) です。 別のタイム ゾーンに基づいて CRON 式を実行するには、ご使用の関数アプリ用に WEBSITE_TIME_ZONE という名前のアプリ設定を作成します。


# 参考ドキュメント
- [Azure App Service でのアプリの診断ログの有効化](https://docs.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs)
- [Azure Monitor の Log Analytics の概要](https://docs.microsoft.com/ja-jp/azure/azure-monitor/logs/log-analytics-overview)
- [Log Analytics のチュートリアル](https://docs.microsoft.com/ja-jp/azure/azure-monitor/logs/log-analytics-tutorial)



<br>
<br>

---

<br>
<br>

2022 年 12 月 20 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>