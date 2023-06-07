---
title: "App Service で full FREB (失敗した要求トレース) を取得する方法"
author_name: "Yuna Sugimoto"
tags:
    - App Service
---


# はじめに
App Service (windows) では、Internet Information Services (IIS) と呼ばれる Web サーバーが動作しています。App Service ではオンプレミス環境で動作している IIS と同様に、要求のトレース機能を提供しており、失敗した要求トレース (FREB) を取得できます。これにより、お客様より Web サーバー内部の挙動や各コンポーネントの処理に要した時間などをご確認いただくことが出来ます。

FREB の詳細については、以下ドキュメントをご参照ください。

[Monitor Activity on a Web Server (IIS 7)](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc730608(v=ws.10)#failed-request-tracing-rules)


[インターネット Web サーバー
構築ガイドライン【ドラフト版】  第 9 章： ログやトレースを活用しよう(PDF)](https://download.microsoft.com/download/8/F/3/8F3E42CB-5E5A-4BC5-8549-5F408389469F/InternetWebServerGuideline_chapter9_draft.pdf) 

本記事では、要求のトレース機能を使用して FREB ファイルを取得する方法についてご紹介いたします。

# FREB ファイルを取得する方法
## 1) "失敗した要求トレース" の有効化
Azure ポータルより当該 App Service を選択後、[App Service ログ] ブレード > 失敗した要求のトレースを"オン"にします。


[診断ログの有効化](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#log-detailed-errors)
>Azure portal で Windows アプリのエラー ページまたは失敗した要求のトレースを保存するには、アプリに移動し、 [App Service ログ] を選択します。
>[詳細なエラー ログ記録] または [失敗した要求のトレース] で、 [オン] を選択し、 [保存]を選択します。


![image-f22b581a-1a37-449a-bd7d-1e73f9533360.png]({{site.baseurl}}/media/2022/12/image-f22b581a-1a37-449a-bd7d-1e73f9533360.png)


## 2 ) Web.config にトレース対象の要求を構成
App Service で失敗した要求トレースを有効に設定した場合、既定では HTTP ステータス コードが 400 から 600 の値となるエラー発生のみがトレース対象になります。HTTP ステータス コード 200 等も対象にトレースするためには独自の構成が必要となります。
以下のように Web.config に <tracing> を追加し、トレース対象とするリクエストを設定します。

“Web.config” 追記例 :

① エラーとなったリクエストに対して取得する例

アプリケーションのすべてのパスでステータスコード (HTTP 400-403,405-999) を返却したリクエストをトレース対象としています。
```xml
<configuration>
    <system.webServer>
        <tracing>
            <traceFailedRequests>
                <clear/>
                <add path="*">
                    <traceAreas>
                        <add provider="ASP" verbosity="Verbose" />
                        <add provider="ASPNET" areas="Infrastructure,Module,Page,AppServices" verbosity="Verbose" />
                        <add provider="ISAPI Extension" verbosity="Verbose" />
                        <add provider="WWW Server" areas="Authentication,Security,Filter,StaticFile,CGI,Compression,Cache,RequestNotifications,Module,FastCGI,WebSocket,RequestRouting,Rewrite,ANCM" verbosity="Verbose" />
                    </traceAreas>
                    <failureDefinitions statusCodes="400-403,405-999" />
                </add>
            </traceFailedRequests>
        </tracing>
    </system.webServer>
</configuration>
```

② リクエストの処理に 30 秒以上かかったリクエストに対して取得する例

アプリケーションのすべてのパスで応答までの時間 (timeTaken) が 30 秒以上要したリクエストをトレース対象としています。
```xml
<configuration>
    <system.webServer>
        <tracing>
            <traceFailedRequests>
                <clear/>
                <add path="*">
                    <traceAreas>
                        <add provider="ASP" verbosity="Verbose" />
                        <add provider="ASPNET" areas="Infrastructure,Module,Page,AppServices" verbosity="Verbose" />
                        <add provider="ISAPI Extension" verbosity="Verbose" />
                        <add provider="WWW Server" areas="Authentication,Security,Filter,StaticFile,CGI,Compression,Cache,RequestNotifications,Module,FastCGI,WebSocket,RequestRouting,Rewrite,ANCM" verbosity="Verbose" />
                    </traceAreas>
                    <failureDefinitions timeTaken="00:00:30" />
                </add>
            </traceFailedRequests>
        </tracing>
    </system.webServer>
</configuration>
```

## 3 ) トレース対象のリクエストを発生させる
App Service へアクセスを行い、当該事象が発生することを確認します。

## 4 ) 失敗した要求トレースログ (FREB) を取得する
FREB は、Kudu サイトより取得いただくことが出来ます。

具体的には、Azure Portal より当該 App Service を選択後、[高度なツール] > 移動 リンクより Kudu サイトにアクセスします。Kudu サイトへアクセス後、'Debug console' から CMD を選択 > LogFiles ディレクトリをクリック > w3svc から始まるディレクトリ内に失敗した要求のトレースログが格納されております。

ファイルをご提供いただく際には、対象ファイル/フォルダーの左側ダウンロードアイコンより対象ファイルをダウンロードできます。



![image-3127c186-0663-4685-9e1c-0c60a8a6d3b6.png]({{site.baseurl}}/media/2022/12/image-3127c186-0663-4685-9e1c-0c60a8a6d3b6.png)

![image-0a8f3d71-005e-455d-81db-bd7f35e268f4.png]({{site.baseurl}}/media/2022/12/image-0a8f3d71-005e-455d-81db-bd7f35e268f4.png)

![image-bd58ee70-301d-4ab3-81b2-55f05a2a2811.png]({{site.baseurl}}/media/2022/12/image-bd58ee70-301d-4ab3-81b2-55f05a2a2811.png)


ログファイルの格納先については、以下ドキュメントにも記載がございますため、併せてご参照ください。

[診断ログの有効化](https://learn.microsoft.com/ja-jp/azure/app-service/troubleshoot-diagnostic-logs#access-log-files)
>![image-4b58b45a-292e-45fb-b0da-99127268f055.png]({{site.baseurl}}/media/2022/12/image-4b58b45a-292e-45fb-b0da-99127268f055.png)


以上、App Service 上で FREB を取得する方法についてご紹介させていただきました。

<br>
<br>

---

<br>
<br>

2023 年 06 月 05 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>

