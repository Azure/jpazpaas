---
title: "Azure Functions のトリガーの同期とは"
author_name: "Hayato Kuroda"
tags:
    - Function App
---

# 質問
Azure Functions に[トリガーの同期](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-deployment-technologies?tabs=windows#trigger-syncing)がありますが何の機能ですか。

> トリガーの同期
トリガーのいずれかを変更する場合、Functions インフラストラクチャにその変更を認識させる必要があります。 同期は、多くのデプロイ テクノロジでは自動的に実行されます。 ただし、場合によっては、トリガーを手動で同期する必要があります。 外部パッケージ URL、ローカル Git、クラウドの同期、または FTP を参照することで更新をデプロイする場合は、トリガーを手動で同期する必要があります。 次の 3 つの方法のいずれかでトリガーを同期できます。
Azure portal で関数アプリを再起動する。
https://{functionappname}.azurewebsites.net/admin/host/synctriggers?code=<API_KEY>を使用して HTTP POST 要求を https://{functionappname}.azurewebsites.net/admin/host/synctriggers?code=<API_KEY> に送信する。
HTTP POST 要求を https://management.azure.com/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME>/providers/Microsoft.Web/sites/<FUNCTION_APP_NAME>/syncfunctiontriggers?api-version=2016-08-01 に送信する。 プレースホルダーは、ご使用のサブスクリプション ID、リソース グループ名、および関数アプリの名前に置き換えてください。
外部パッケージ URL を使用してデプロイし、パッケージの内容が変更されていても URL 自体が変更されない場合は、関数アプリを手動で再起動して更新プログラムを完全に同期する必要があります。

# 回答
Azure Functions にデプロイされたアプリケーションのトリガーの情報を Azure Functions のバックエンドに同期するための機能です。

## なぜトリガーの同期が必要か
Azure Functions は、アプリケーション ルート(/home/site/wwwroot/)を基幹として複数のトリガーごとにフォルダを分割した構造となっています。下記は C#(インプロセス) の場合の[フォルダ階層](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-dotnet-class-library?tabs=v4%2Ccmd#functions-class-library-project)の例です。

![image-7da7eea6-9e7d-4aec-852d-34ac06f2cff2.png]({{site.baseurl}}/media/2023/09/image-7da7eea6-9e7d-4aec-852d-34ac06f2cff2.png)


host.json と呼ばれる Azure Functions 全体の設定ファイルに加えて、ご利用の各トリガーごとに function.json と呼ばれる定義ファイルを保持しております。どういったバインディングを利用しているかの一覧などの情報を知るためにはファイル名などの表面的な情報では不足しており、すべての設定ファイルを読み込み・解釈する必要があります。

例えば、Azure ポータルの関数一覧に実装したトリガーを表示することを考えます。下記のような複数のトリガーを実装した場合には、この Azure Functions リソースにデプロイされているアプリケーションには、BLOBTrigger~TimerTrigger まで 15 個のトリガーが実装されていますが、この一覧という情報はどの設定ファイルにも無く、それぞれのトリガーごとのフォルダに含まれている functions.json を確認する必要があります。

<関数一覧>

![image-46fd2f8a-e707-4450-8525-fe6f71bcebdc.png]({{site.baseurl}}/media/2023/09/image-46fd2f8a-e707-4450-8525-fe6f71bcebdc.png)

<BLOBTrigger の function.json の例>

![image-b6c0a2a3-1896-4566-9365-d3d18a9650d6.png]({{site.baseurl}}/media/2023/09/image-b6c0a2a3-1896-4566-9365-d3d18a9650d6.png)

こういった情報(いわば Azure Functions のメタデータ)を更新するための契機としてトリガーの同期が存在しています。


## 動作シーケンス
前述の解説と[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-deployment-technologies?tabs=windows#trigger-syncing)にある通り、トリガーの同期はデプロイ時に実施する必要があります。Azure Functions での推奨としている ZIP デプロイの場合の動作を確認します。

1. [ZIP デプロイ](https://github.com/projectkudu/kudu/wiki/Deploying-from-a-zip-file-or-url)とは、Kudu(SCM サイト) の `/api/zipdeploy` エンドポイントにアプリケーション コンテンツと一緒に POST 通信することです。このデプロイ時に、Kudu の PostDeploymentHelper クラスの中で Azure Functions の場合には[トリガーの同期](https://github.com/projectkudu/kudu/blob/98d12cf400675bf0c5fefffd92c4dc6f36d6e33b/Kudu.Core/Helpers/PostDeploymentHelper.cs#L299-L300) (`/admin/host/synctriggers` へのリクエスト)が呼び出されています。


2. このトリガーの同期のエンドポイント `/admin/host/synctriggers` は Azure Functions がリスンしているエンドポイントです。[FunctionsSyncManager クラス](https://github.com/Azure/azure-functions-host/blob/ed7e802e2e48d31754bf2ce9ee7033a4f0ab9c00/src/WebJobs.Script.WebHost/Management/FunctionsSyncManager.cs#L725-L726)で、`/admin/host/synctriggers` を受けたら `/operation/settriggers` を呼び出しています。

以上の2段の HTTP リクエストが発出されることでトリガーの同期が行われます。`/operation/settriggers` を受け付けた以降の処理は Azure Functions 内部動作となっており現時点で公開情報はございません。 

VNet 統合している場合の動作シーケンスのイメージは下記となります。
![image-63418e81-91c7-4edb-8fd0-148e98fa756d.png]({{site.baseurl}}/media/2023/09/image-63418e81-91c7-4edb-8fd0-148e98fa756d.png)

## トリガーの同期失敗による影響と対策
トリガーの同期が失敗するとどういった影響があるか解説します。
構成例として、ある Azure Functions が VNet 統合を設定しており、サブネットに適用された NSG で外部接続（Outbound）がすべて拒否されている場合を考えます。

### 1. Azure ポータルにトリガーが表示されない
Kudu から確認するとアプリケーション ルート(/home/site/wwwroot/)にアプリケーション コードは配置されていますが、Azure ポータルの関数には一切の関数が表示されず、あたかも何もデプロイされていないように見えてしまいます。この状態ではデプロイされているアプリケーション コードは起動するため、処理は行われます。画面表示との乖離が問題となります。

![image-31938ab1-6302-4bfb-88ff-d50e2443ba5d.png]({{site.baseurl}}/media/2023/09/image-31938ab1-6302-4bfb-88ff-d50e2443ba5d.png)

この時 Azure Functions のアクティビティ ログを確認すると、直前のトリガーの同期が失敗していることがわかります。

![image-6bb326fd-1d34-4c04-b4b5-7fae36e5233d.png]({{site.baseurl}}/media/2023/09/image-6bb326fd-1d34-4c04-b4b5-7fae36e5233d.png)

さらに、[こちら](https://learn.microsoft.com/ja-jp/rest/api/appservice/web-apps/sync-function-triggers)から手動でトリガーの同期を実行することができます。実際にトリガーの同期を行ってみると 400 BadRequest が応答されます。

![image-6e6ef434-b0e5-460b-a33f-2e309c0cc4dd.png]({{site.baseurl}}/media/2023/09/image-6e6ef434-b0e5-460b-a33f-2e309c0cc4dd.png)


### 2. 従量課金プランまたは EP プランのスケーリングが動作しない

従量課金プラン及び EP プランでは、[イベントドリブン スケーリング](https://learn.microsoft.com/ja-jp/azure/azure-functions/event-driven-scaling?tabs=azure-cli#runtime-scaling)と呼ばれる動作で Azure Functions プラットフォームにてスケール イン/アウトが行われます。この際に Azure Functions のスケール コントローラーと呼ばれるコンポーネントが動作しスケール イン/アウトの決定をしてますが、このスケール コントローラーも Azure Functions のメタデータを利用します。

キュー トリガーを例とすると、対象のキューの長さを基にスケール コントローラーがスケール イン/アウトの決定をします。トリガーの同期が失敗している場合には、そもそもスケール コントローラーが対象のキューの長さを参照するように動作せず、スケール イン/アウトの決定を行いません。結果として、多量のキュー メッセージがあるにもかかわらず、スケール アウトされない(性能上のボトルネック)や、従量課金プランの場合には 0 インスタンスとなる場合があるため、いつまで経ってもキュー メッセージが処理されないことが発生し得ます。


### 対策
動作シーケンスの通りトリガーの同期の通信が通るように通信許可設定を追加します。
前述の NSG によって通信が拒否されている場合には、AppService のサービスタグを通信許可ルールとして追加します。AzureFirewall などを利用の場合には、`*.azurewebsites.net` の FQDN で 80 番及び 443 番ポートの外部通信を許可します。

<NSG の送信ルール例>

![image-c22289ef-2b81-459f-a953-57b9e050a74b.png]({{site.baseurl}}/media/2023/09/image-c22289ef-2b81-459f-a953-57b9e050a74b.png)


以上が Azure Functions におけるトリガーの同期の解説となります。Azure Functions にアプリケーションをデプロイしたけど、ポータルで参照できないなどトラブルシューティングにお役に立てば幸いです。


<br>
<br>

---

<br>
<br>

2023 年 09 月 15 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>