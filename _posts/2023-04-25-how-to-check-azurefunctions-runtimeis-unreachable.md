---
title: "Azure Functions での「Azure Functions Runtime に到達できません」のよくある原因と対応策"
author_name: "Hayato Kuroda"
tags:
    - Function App
---

# 質問
Azure Functions を利用しており、「Azure Functions Runtime に到達できません」とエラーが表示されます。対応を教えてください。

![image-5e22ac13-1e1b-4f56-9355-8ff737735036.png]({{site.baseurl}}/media/2023/04/image-5e22ac13-1e1b-4f56-9355-8ff737735036.png)

# 回答
Azure Functions が何かしらの理由で正常に起動ができていないことを表します。
多くの原因はストレージ アカウントとの接続ができないことに起因し、原因は Application Insights ログに記録されていない場合が多いです。よくあるシナリオに分けて解説します。

Azure Functions Runtime に到達できませんについては、[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-recover-storage-account)にご案内もありますが、本稿ではより具体的なトラブルシューティング方法と追加のシナリオをご案内します。

## Azure Functions が規定で利用するストレージ アカウント
Azure Functions ではその動作のためにストレージ アカウントを利用しますが、それぞれの用途に応じて別々のアプリケーション設定に定義されています。代表的な 2 つを説明します。

- AzureWebJobsStorage

Azure Functions ランタイムにて利用される汎用ストレージ アカウントを指定する接続文字列を保持するアプリケーション設定になります。従量課金、Premium または専用プランに依らずすべての Azure Functions が各トリガーの管理や関数実行のログ管理などの操作にストレージ アカウントを使用します。Azure Functions 作成時にストレージ アカウントが同時に作成されるのはこれらに利用されるためです。

トリガーを作成していない初期状態では、azure-webjobs-hosts と azure-webjobs-secrets の BLOB が作成されます。
![image-1eca6a87-5197-47a3-a16e-2f13baeda221.png]({{site.baseurl}}/media/2023/04/image-1eca6a87-5197-47a3-a16e-2f13baeda221.png)

Azure Functions が利用する[ストレージ アカウントの要件](https://learn.microsoft.com/ja-jp/azure/azure-functions/storage-considerations?tabs=azure-cli#storage-account-requirements
)と本アプリケーション設定の説明([AzureWebJobsStorage](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#azurewebjobsstorage))があります。

- WEBSITE_CONTENTAZUREFILECONNECTIONSTRING と WEBSITE_CONTENTSHARE

Azure Functions の Premium プランと Windows の従量課金プランで、ファイル システムとして Azure Files が既定で設定されています。 この Azure Files のストレージ アカウントの接続文字列と Azure Files 名を指定するためのアプリケーション設定になり、セットで利用されます。下記例の設定では、ストレージ アカウント/blogresourcesa87a のファイル共有/hakurodacontosofunc8aff を使うような設定となります。
![image-e60b96db-7a37-4a8d-9872-f8481423dcd5.png]({{site.baseurl}}/media/2023/04/image-e60b96db-7a37-4a8d-9872-f8481423dcd5.png)

こちらに、本アプリケーション設定の説明([WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring)、[WEBSITE_CONTENTSHARE](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#website_contentshare))があります。


## Azure Functions Runtime に到達できませんの発生シナリオ

### 各シナリオのチートシート

![image-cbcd2ef1-0eb9-448e-9ab7-b90a7c5df073.png]({{site.baseurl}}/media/2023/04/image-cbcd2ef1-0eb9-448e-9ab7-b90a7c5df073.png)


### シナリオ１：ストレージ アカウント用のアプリケーション設定が無い場合
「Azure Functions が規定で利用するストレージ アカウント」で説明したいずれかのアプリケーション設定が無い場合には Azure Functions が正常に起動しません。
***Azure ポータル > 構成*** から、AzureWebJobsStorage、(ご利用のプランに応じて)WEBSITE_CONTENTAZUREFILECONNECTIONSTRING 及び WEBSITE_CONTENTSHAREアプリケーション設定が存在することを確認ください。

[AzureWebJobsStorage が無い場合の例]

![image-d15e533e-1f8a-4f94-a5d9-43d6eb2698e1.png]({{site.baseurl}}/media/2023/04/image-d15e533e-1f8a-4f94-a5d9-43d6eb2698e1.png)


### シナリオ２：ストレージ アカウントの接続文字列が不正の場合
「Azure Functions が規定で利用するストレージ アカウント」で説明したアプリケーション設定に指定した接続文字列が正しい形式ではない場合には Azure Functions が正常に起動しません。

接続想定のストレージ アカウントの接続文字列との一致を確認します。以下の例は、ストレージ アカウントの接続文字列を確認する画面となります。
![image-8b4728be-1d6f-4aad-9be8-22d3eefb5934.png]({{site.baseurl}}/media/2023/04/image-8b4728be-1d6f-4aad-9be8-22d3eefb5934.png)


### シナリオ３：ストレージ アカウントが削除されている場合
「Azure Functions が規定で利用するストレージ アカウント」で説明したアプリケーション設定に指定したストレージ アカウントが無い場合には Azure Functions が正常に起動しません。
***Azure ポータル > 構成*** から、AzureWebJobsStorage、(ご利用のプランに応じて)WEBSITE_CONTENTAZUREFILECONNECTIONSTRING に指定したストレージ アカウントが存在することを確認ください。


### シナリオ４：ストレージ アカウントにアクセスできない場合
ストレージ アカウントにアクセス制限や Azure Functions の通信経路上で通信が拒否されており、Azure Functions からストレージ アカウントへ通信ができない場合には Azure Functions が正常に起動しません。

もし Azure Functions が利用するストレージ アカウントでアクセス制限を設けたい場合には、[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/configure-networking-how-to?tabs=portal#restrict-your-storage-account-to-a-virtual-network)の手順に従って構成します。

### シナリオ４ー１：ストレージ アカウントにアクセス制限を設けている場合
ストレージ アカウントのアクセス制限の設定状況を確認します。

![image-24067dee-d4b0-4872-bfff-7099faa79b69.png]({{site.baseurl}}/media/2023/04/image-24067dee-d4b0-4872-bfff-7099faa79b69.png)

ご確認いただいても事象が解消しない場合にはストレージ アカウントの診断設定を有効化します。少なくとも BLOB の診断設定を有効化し、ストレージ アカウントへのアクセス ログを確認します。
![image-bf34df67-8c43-48a1-af7c-7a958455a742.png]({{site.baseurl}}/media/2023/04/image-bf34df67-8c43-48a1-af7c-7a958455a742.png)

また Azure Functions とストレージ アカウントが同一リージョンにある場合には IP アドレスによる[アクセス制限](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-network-security?tabs=azure-portal#grant-access-from-an-internet-ip-range)は構成いただけませんためご注意ください。

![image-34cf08a0-a87f-4990-af5a-e9dd063f3fe8.png]({{site.baseurl}}/media/2023/04/image-34cf08a0-a87f-4990-af5a-e9dd063f3fe8.png)


### シナリオ４ー２：Azure Functions が VNet 統合している場合
ストレージ アカウントのアクセス制限設定が問題ない場合には、Azure Functions からストレージ アカウントへの通信経路を確認します。比較的多く利用いただいているシナリオでは `Azure Functions -[VNet 統合]-> 仮想ネットワーク|サブネット -[NSG]-> プライベート エンドポイント(BLOB など) -> ストレージ アカウント` や `Azure Functions -[VNet 統合]-> 仮想ネットワーク|サブネット -[NSG]-[ルートテーブル]-> ファイア ウォール -> プライベート エンドポイント(BLOB など) -> ストレージ アカウント` が考えられます。VNet 統合しているサブネット以降の通信経路にて（NSG やファイア ウォールなど）ストレージ アカウントへのアクセスを遮断していないかを確認します。



### シナリオ５：Azure Functions の再起動が繰り返されている場合
Azure Functions の再起動が繰り返されている場合、一時的に「Azure Functions Runtime に到達できません」が表示されたり解消される動作となります。これは Azure Functions はアプリケーションの起動を試みる一方で、何かしらの理由によってシャットダウンすることで非常に不安定な動作となるためです。


#### シナリオ５ー１：アプリケーションの起動に時間がかかっている場合
Azure Functions の Linux OS をご利用の場合、アプリケーションの起動時間の上限が [230 秒(標準値)](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/faqs-app-service-linux?toc=%2Fazure%2Fapp-service%2Ftoc.json#-----------------------------------------------------------)となっています。アプリケーションの起動に 230 秒以上要する場合には、Azure Functions の再起動が繰り返されてしまいます。


Linux OS をご利用の場合、/home/LogFiles に docker ログ(YYYY_MM_DD_<インスタンスID>_docker.log 形式)が出力されます。docker ログには、アプリケーションの起動時の記録が残るため、そちらを確認します。230 秒以上要しており起動に失敗した場合には下記のようなログが出力されます。

![image-9cbcfe36-142d-458a-a00b-66b422acfad0.png]({{site.baseurl}}/media/2023/04/image-9cbcfe36-142d-458a-a00b-66b422acfad0.png)

アプリケーションの起動に 230 秒以上要しないことが望ましいものの、どうしても時間を要する場合にはアプリケーション設定 [WEBSITES_CONTAINER_START_TIME_LIMIT](https://learn.microsoft.com/ja-jp/azure/app-service/reference-app-settings?tabs=kudu%2Cdotnet#custom-containers) を追加して 230 秒以上要しても起動を続けるように設定します。
また、[パッケージからの実行](https://learn.microsoft.com/ja-jp/azure/azure-functions/run-functions-from-deployment-package#enable-functions-to-run-from-a-package)を行うように設定しアプリケーションがなるべく早く起動するようにします。

#### シナリオ５ー２：アプリケーションが正しく起動できていない場合
例えば C# をご利用の場合には、[Startup.cs](https://learn.microsoft.com/ja-jp/aspnet/core/fundamentals/startup?view=aspnetcore-6.0) を定義することで起動時の初期処理など(アプリケーション スタートアップ コード)を実装いただけます。
アプリケーションの起動時(実行時ではない)に例外が発出されると、Azure Functions は起動に失敗し永遠と再度起動を試みます。結果として再起動が繰り返されるため「Azure Functions Runtime に到達できません」が表示されます。

この場合には、アプリケーション ログが記録される場合があります。具体例で、下記のようなログが確認できる場合があります。今回は Startup.cs に例外を発生させるコードを仕込んでます。

![image-41ea3ec8-be78-4c96-aa86-8b7fc91d485d.png]({{site.baseurl}}/media/2023/04/image-41ea3ec8-be78-4c96-aa86-8b7fc91d485d.png)


### シナリオ６：Azure Functions が BYOS を利用している場合
ストレージ アカウントの Azure Files を[任意のディレクトリにマウント](https://learn.microsoft.com/ja-jp/azure/azure-functions/storage-considerations?tabs=azure-cli#mount-file-shares)して利用いただくことができます。この時の設定誤りなどの理由により正しくマウントできない場合には「Azure Functions Runtime に到達できません」が発生します。

以下はマウントした Azure Files が存在しない場合の例です。/mounted を Azure Files にマウントしていましたが、マウント先の Azure Files の削除しています。この状態になっている場合には Kudu や SSH が起動しません。

![image-24c9e0df-ddca-4909-a347-07097e7452a2.png]({{site.baseurl}}/media/2023/04/image-24c9e0df-ddca-4909-a347-07097e7452a2.png)

Azure ポータルの問題の診断と解決から、”コンテナの問題”を確認します。一例ですが、マウントが原因の旨確認できます。

![image-a74bcd49-3a56-4fa3-9fdf-2a6058a95025.png]({{site.baseurl}}/media/2023/04/image-a74bcd49-3a56-4fa3-9fdf-2a6058a95025.png)

![image-901d5b7d-3b54-4415-80dc-12b09e2f9b93.png]({{site.baseurl}}/media/2023/04/image-901d5b7d-3b54-4415-80dc-12b09e2f9b93.png)


### シナリオ７：デイリーのクォータを超過している場合
Azure Functions の従量課金プランでは、1 日当たりに利用できるクォータの上限を設定可能です。設定状況は、***Azure ポータル > 構成 - 関数のランタイム設定*** から確認可能です。
![image-e626b835-dd43-4bab-8df2-2a9d168eeb7d.png]({{site.baseurl}}/media/2023/04/image-e626b835-dd43-4bab-8df2-2a9d168eeb7d.png)

従量課金プランをご利用の場合には、こちらを確認いただき必要に応じてクォータの上限の変更と Azure Functions の再起動ください。


### シナリオ８：App_Offline.htm が配置されている場合
Azure Functions へのアプリケーションのデプロイ中など一時的に Azure Functions を停止させるために /home/site/wwwroot に App_Offline.htm が作成されます。App_Offline.htm が削除されれば「Azure Functions Runtime に到達できません」は解消されます。

![image-66af2f95-b0df-433c-a8f5-ace367e2abde.png]({{site.baseurl}}/media/2023/04/image-66af2f95-b0df-433c-a8f5-ace367e2abde.png)

App_Offline.htm は、[ASP.NET Core 用](https://learn.microsoft.com/ja-jp/aspnet/core/host-and-deploy/app-offline?view=aspnetcore-7.0)のファイルとなり、利用者での手動作成は推奨されません。


今回は以上のとなります。他にも発生するシナリオがあれば随時追記していきますため今後のアップデートをご期待ください。お読みいただき、ありがとうございました。

<br>
<br>

---

<br>
<br>

2023 年 04 月 25 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>