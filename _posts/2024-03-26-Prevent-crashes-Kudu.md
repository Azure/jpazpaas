---
title: "「'system.webServer/runtime' already defined 」エラーによる Kudu クラッシュの回避方法について"
author_name: "syukumoto"
tags:
    - App Service
---




<br>
<br>

# はじめに

お世話になっております。App Service サポート担当の行本です。

本記事は 2024 年 3 月 20 日に公開されました [Prevent crashes due to ‘system.webServer/runtime’ already defined](https://azure.github.io/AppService/2024/03/20/Azure-WebApp-crashing-due-to-duplicate-runtime-section.html) の日本語訳です。  
翻訳内容には細心の注意を払っておりますが、もし原文内容と齟齬がある場合は、原文の内容を優先ください。  
  
本記事では、App Service / Azure Functions (以下、アプリと呼称します) において、お客様がカスタム XDT 変換、または古いバージョンの Site Extension (拡張機能) を使用していた場合に発生する可能性のある問題について説明します。症状は以下のとおりです。  
1. Kudu (SCM) サイトがクラッシュし、復旧しません。(Kudu サイトにアクセスした際、"HTTP Error 503. The service is unavailable." によりアクセスに失敗します)  
2. EventLog.xml ファイル (%HOME%\LogFilesフォルダにあります) を開くと以下の様に、「'**system.webServer/runtime**' already defined」というエラーが記録されています。  

```
<Data>~1YourSiteName</Data>
<Data>Config section 'system.webServer/runtime' already defined. Sections 
must only appear once per config file. See the help topic <location> for exceptions.
</Data>
<Data>\\?\D:\DWASFiles\Sites\#1YourSiteName\Config\applicationhost.config</Data>
<Data>1150</Data>
<Binary>B7000000</Binary>
```

この問題は、アプリの Site Extension (または[カスタム XDT 変換](https://github.com/projectkudu/kudu/wiki/Xdt-transform-samples)) で、applicationHost.config ファイルのコレクション内にカスタム環境変数を追加するために (**InsertIfMissing** ではなく) **Insert** を使用して正しくない XDT 変換が含まれている場合に発生します。  
これにより、'**system.webServer/runtime**' タグが重複し、エラーが発生します。  
以下の Site Extension には、この問題に対処するための更新プログラムがあります。  

1. NewRelic  
2. Composer  
  
上記 Site Extension は一例ですが、カスタム XDT 変換や、**InsertIfMissing** の代わりに **Insert** を使用して XDT 変換を行っているその他の Site Extension を使用しているときに、エラーが発生する可能性があることに注意してください。

# 解決方法

この問題を解決するには、次の手順を実行します。  
  
1. アプリの FTP/S エンドポイントを取得します (詳細については、[「FTP/S エンドポイントを取得する」](https://learn.microsoft.com/ja-JP/azure/app-service/deploy-ftp?tabs=portal#get-ftps-endpoint)を参照してください)。  
2. (FileZilla や FFFTP など任意のクライアントを使用して、FTP/S エンドポイントに接続し) **/SiteExtensions** フォルダに移動します。  
3. 問題となっている Site Extension (Composer、NewRelicなど) のフォルダを削除します。  
4. Site Extension の削除を反映するため、Azure Portal からアプリを再起動します。  
5. アプリの [高度なツール] より Kudu コンソール (.azurewebsites.net) に入り、**[Site Extensions]** タブに移動して、ギャラリーから最新バージョンの Site Extension をインストールします。  

>注意: 上記の症状で説明されている問題が発生していることを確認した場合にのみ、上記の手順を実行してください。

アプリで[カスタム XDT 変換](https://github.com/projectkudu/kudu/wiki/Xdt-transform-samples)を利用している場合は、**Insert** ではなく **InsertIfMissing** を使用してください。  

# 追加情報
この問題は、App Service による Diagnostic as a Service (DaaS) Site Extension の更新に伴って発生するようになりました。  
2024 年 2 月に展開が開始された DaaS Site Extension の更新においては DaaS が診断目的で使用する新しい環境変数が導入されています。DaaS Site Extension は適切に InsertIfMissing 変換を使用していましたが、system.webServer/runtime セクションを挿入したことで、他の Insert 変換を使用する XDT 変換が存在する場合は競合し、エラーが発生するようになりました。

<br>
<br>

2024 年 03 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>