---
title: "[日本語訳]App Service における断続的な接続問題のトラブルシューティング"
author_name: "Tsutomu Mori"
tags:
    - App Service
---

<br>
<br>

# はじめに
お世話になっております。App Service サポート担当の森です。


本記事は [Apps on Azure Blog](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/bg-p/AppsonAzureBlog)で2023年11月に公開されている [Troubleshoot intermittent connectivity issue in azure app service](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/troubleshoot-intermittent-connectivity-issue-in-azure-app/ba-p/3969841#user-content-tcpping) の日本語抄訳としてご案内いたします。

# App Service における断続的な接続問題のトラブルシューティング

Azure App Service 上で動作しているアプリケーションでは、断続的な接続問題によりターゲットサービスへの接続時にエラーが発生することがあります。その問題を特定するためにコードを変更してデプロイするのは手間がかかります。App Service Kudu サイトを使用すれば迅速にテストを行うことができ、その場合コードを変更する必要はありません。

## Linux
Kudu サイト（例 : `https://<appname>.scm.azurewebsites.net`）にログインし、SSH メニューへ移動すると SSH Shell を確認できます。カスタムイメージを使用している場合は、このドキュメントに従ってそれを有効にする必要があります：[Configure a custom container for Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux#detect-https-session)

### nslookup

nslookup は通常、Linux Kudu イメージに既にインストールされており、FQDN が正しい IP アドレスに解決されているかを確認することができます。
以下では、`login.microsoftonline.com` に DNS 解決を行っていますが、任意のアドレスに変更することができます。

![image-bc1a959e-ab34-4089-af6c-29fdb9d749f0.png]({{site.baseurl}}/media/2024/03/image-bc1a959e-ab34-4089-af6c-29fdb9d749f0.png)

nslookup がインストールされていない場合は、以下のコマンドを実行してください。

![image-37f72a0f-4f78-4d44-93b3-1b72746b534e.png]({{site.baseurl}}/media/2024/03/image-37f72a0f-4f78-4d44-93b3-1b72746b534e.png)

以下のスクリプトをファイル（例 : `dnstest.sh`）に保存して nslookup のテストを繰り返し実行することができます。パラメータは必要に応じて調整することができます。

![image-7c48e8f0-d279-4ac5-af19-22edcd765f60.png]({{site.baseurl}}/media/2024/03/image-7c48e8f0-d279-4ac5-af19-22edcd765f60.png)

以下を実行します。

![image-adb807a0-5b02-43e9-9b3a-8229e58697df.png]({{site.baseurl}}/media/2024/03/image-adb807a0-5b02-43e9-9b3a-8229e58697df.png)

詳細をログファイルに出力します。

![image-60cb0917-5504-4fba-85ce-2f8bd446bd9b.png]({{site.baseurl}}/media/2024/03/image-60cb0917-5504-4fba-85ce-2f8bd446bd9b.png)

### dig
dig もデフォルトでインストールされています。DNS 解決の詳細な情報を確認することができます。

![image-6024d890-e908-42c2-aa12-58fa1beff845.png]({{site.baseurl}}/media/2024/03/image-6024d890-e908-42c2-aa12-58fa1beff845.png)

以下のスクリプトをファイル（例 : `digtest.sh`）に保存して、dig のテストを繰り返し実行することができます。

![image-2b055edf-9230-4b3e-a707-c1b7a20b2ea9.png]({{site.baseurl}}/media/2024/03/image-2b055edf-9230-4b3e-a707-c1b7a20b2ea9.png)
 
以下を実行します。
 
![image-99548d0e-0bb8-4886-82fe-c1b365d3db1b.png]({{site.baseurl}}/media/2024/03/image-99548d0e-0bb8-4886-82fe-c1b365d3db1b.png)
 




### NodeJS

NodeJS ランタイムを使用している場合、以下のコマンドを使用して DNS 解決に要した時間を取得できます。

![image-54eddcc3-f855-48f6-85a6-0af1f2493898.png]({{site.baseurl}}/media/2024/03/image-54eddcc3-f855-48f6-85a6-0af1f2493898.png)

### Python
以下のスクリプトを `dnstest.py` に保存します。 
 
![image-29b2d6b1-ebf9-48cb-8f88-d490850ed12a.png]({{site.baseurl}}/media/2024/03/image-29b2d6b1-ebf9-48cb-8f88-d490850ed12a.png)

以下を実行します。

![image-fac013e2-06b8-4d5a-b8a7-a1d77395c449.png]({{site.baseurl}}/media/2024/03/image-fac013e2-06b8-4d5a-b8a7-a1d77395c449.png) 





### tcpping
tcpping はインストールされていない可能性があります。その場合は以下に従ってインストールしてください。

![image-c5b8a901-ba01-458f-92a7-84241a6c9d91.png]({{site.baseurl}}/media/2024/03/image-c5b8a901-ba01-458f-92a7-84241a6c9d91.png)
 
接続をテストします。
 
![image-6247eb05-e7c0-4d53-a86f-dafe5c501497.png]({{site.baseurl}}/media/2024/03/image-6247eb05-e7c0-4d53-a86f-dafe5c501497.png)

デフォルトでは tcpping は無限に繰り返されます。-x パラメーターを使用してリピートする回数を指定してください。
 
![image-ee183163-3beb-4ed2-97d2-ec456f69bc22.png]({{site.baseurl}}/media/2024/03/image-ee183163-3beb-4ed2-97d2-ec456f69bc22.png)
 
### curl
curl コマンドでは、クライアントをシミュレートしてターゲットサービスからデータを取得することができます。 例えば、API などのターゲットアプリケーションをテストすることが可能です。postman-echo サービスを使用して curl がターゲットサービスに正しいデータを送信するかを確認し、その後サービスをターゲット API サービスに変更できます。

![image-3b7129fd-6b9c-4327-bf86-366391123bc0.png]({{site.baseurl}}/media/2024/03/image-3b7129fd-6b9c-4327-bf86-366391123bc0.png)

以下のスクリプトをファイル（例 : `curltest.sh`）に保存し、curl コマンドのテストを繰り返し実行することができます。必要に応じてパラメータを調整します。

![image-4e26b0e8-7b97-48a9-befd-f85063872305.png]({{site.baseurl}}/media/2024/03/image-4e26b0e8-7b97-48a9-befd-f85063872305.png)
 
以下を実行します。

![image-811d4c39-5755-4ea2-af25-499bf1bc1ae9.png]({{site.baseurl}}/media/2024/03/image-811d4c39-5755-4ea2-af25-499bf1bc1ae9.png)
 





### tcpdump
接続が失敗し、その原因が不明な場合、tcpdump を使用してネットワークトレースをキャプチャすることができます。

![image-d84488c0-cdb9-408a-abd6-d0bb9192f3be.png]({{site.baseurl}}/media/2024/03/image-d84488c0-cdb9-408a-abd6-d0bb9192f3be.png)
 
これで tcpdump はネットワークパケットをキャプチャするようになりました。別の Kudu サイトのインスタンスを開き、問題を再現するためのコマンドを実行してください。そして、はじめの Kudu サイトに戻り、ctrl+C を押してキャプチャを停止します。 

`https://<appname>.scm.azurewebsites.com/newui/fileManager` にアクセスして、`traffic.pcap` ファイルをダウンロードします。

![image-3eeef4b2-0366-4bf1-8ba9-80d9a3369a3e.png]({{site.baseurl}}/media/2024/03/image-3eeef4b2-0366-4bf1-8ba9-80d9a3369a3e.png)
	
ダウンロードが成功したら、[wireshark](https://www.wireshark.org/) を使用してネットワークトレースファイルを分析することができます。







##　Windows
Kuduサイト `https://<appname>.scm.azurewebsites.net/` にログインし、「Debug console」->「PowerShell」メニューに移動すると、PowerShell ウィンドウが表示されます。
### nameresolver.exe
nameresolver は nslookup と同様に DNS サーバーに対して DNS lookup を行います。

![image-34bb898e-dc01-4ea8-abb2-05bd74a30b9f.png]({{site.baseurl}}/media/2024/03/image-34bb898e-dc01-4ea8-abb2-05bd74a30b9f.png)

nameresolver のテストを繰り返し行うために、ターゲットドメインを継続的に解決するループを記述し、断続的な問題を確認することができます。 例えば、以下のファイルを `nameresolvertest.ps1` に保存します。

![image-38cef5fb-2638-43a4-904c-d2954a04a21f.png]({{site.baseurl}}/media/2024/03/image-38cef5fb-2638-43a4-904c-d2954a04a21f.png)
	 
以下を実行します。
	
![image-87c6c06c-d615-4359-8a12-a2026a112592.png]({{site.baseurl}}/media/2024/03/image-87c6c06c-d615-4359-8a12-a2026a112592.png)	
	



### tcpping.exe				
tcpping は、特定のポートでターゲットサービスへの接続が機能しているかをテストできます。	
サーバーとポートの間には ":" の記号があることに注意してください。これは Linux 環境の tcpping とは異なり、Linux では空白 " " が使われます。
		
![image-f3bb4fcf-c716-4c36-872e-08689a9f19bc.png]({{site.baseurl}}/media/2024/03/image-f3bb4fcf-c716-4c36-872e-08689a9f19bc.png)			
					
tcpping を使用して、ポートに紐づく異なるサービスをテストすることができます。
		
![image-165d8ca5-0297-4d04-bfe0-47d6cabdd436.png]({{site.baseurl}}/media/2024/03/image-165d8ca5-0297-4d04-bfe0-47d6cabdd436.png)	

Windows の tcpping では、デフォルトの ping 回数は4回ですが、-n パラメータを指定してターゲットを複数回 ping することができます。例えば、以下のスクリプトは tcpping を使ってターゲットを100回呼び出します。
			
![image-ed35a1f5-e8b3-497a-9053-a8776d514862.png]({{site.baseurl}}/media/2024/03/image-ed35a1f5-e8b3-497a-9053-a8776d514862.png)	


### curl.exe

Windows Kudu サイトでは、curl コマンドは PowerShell の Invoke-WebRequest コマンドレットを呼び出しますが、PowerShell のコマンドレットの代わりにスクリプト内で  "curl.exe" を使用することができます。
	
![image-763c8cf0-207f-4d9e-b5f3-5611f9fea6ea.png]({{site.baseurl}}/media/2024/03/image-763c8cf0-207f-4d9e-b5f3-5611f9fea6ea.png)	

curl.exe は 応答で赤文字を返すことがありますが、それ自体に問題はありません。正しい内容が返ってきていれば、それは接続が良好であることを示しています。


![image-1733edee-44e6-4c5f-b8a3-0c9ddcc1152a.png]({{site.baseurl}}/media/2024/03/image-1733edee-44e6-4c5f-b8a3-0c9ddcc1152a.png)

### Webjob
バックエンドで実行するための PowerShell スクリプトの Webjob を作成することもできます。これは、Kudu コンソールから実行するよりも安定しています。

次の例をローカルファイル （例 : `pswebjob.ps1`）に保存してください。

![image-d8035539-229d-4c4c-96dd-e4aaad798435.png]({{site.baseurl}}/media/2024/03/image-d8035539-229d-4c4c-96dd-e4aaad798435.png)

PowerShell スクリプト内で、Write-Host を Write-Output に変更する必要があることに注意してください。変更しない場合、ハンドルエラーが発生します。

継続的な Webjob を作成する方法については、以下のドキュメントを参照してください。

https://learn.microsoft.com/en-us/azure/app-service/webjobs-create#CreateContinuous

Webjob が作成されると、以下のようなログが確認できるはずです。

![image-d33b9201-5168-4481-9235-5d59e574cceb.png]({{site.baseurl}}/media/2024/03/image-d33b9201-5168-4481-9235-5d59e574cceb.png)

## 参考資料
- [Azure App Service virtual network integration troubleshooting guide - Azure | Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps#troubleshoot-outbound-connectivity-on-windows-apps)

- [Quickly test connectivity from Azure Website to SQL DB - Microsoft Community Hub](https://techcommunity.microsoft.com/t5/azure-database-support-blog/quickly-test-connectivity-from-azure-website-to-sql-db/ba-p/368868)
- [Networking Related Commands for Azure App Services - Microsoft Community Hub](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/networking-related-commands-for-azure-app-services/ba-p/392410)
- [Installing TcpPing on Azure App Service Linux - (azureossd.github.io)](https://azureossd.github.io/2021/06/17/installing-tcpping-linux/)
- [https://gist.github.com/cnDelbert/5fb06ccf10c19dbce3a7](https://gist.github.com/cnDelbert/5fb06ccf10c19dbce3a7 ) 


---

<br>
<br>

2024 年 03 月 13 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>