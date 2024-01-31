---
title: "API Management と Function Apps を構成した際の基本のパフォーマンス トラブルシューティング"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - API Management
    - Application Insights
---

# 質問
API Management の応答が遅いです。バックエンドには Functions Apps がいます。原因を教えてください。

# 回答
ログを駆使してバックエンドに設定している Function Apps の応答時間を確認します。

## 環境構成
今回は API Management をフロントとして、バックエンドの Function Apps が直列して複数呼び出される場合を考えます。`API Management -> Function Apps1(HakurodaChain001) -> Function Apps2(HakurodaChain002) -> Function Apps3(HakurodaChain003)`とします。

Function Apps には任意の HTTP トリガーをデプロイします。仕組みとして数秒~数十秒程度 Sleep するコードを組み込んでおり、要求によってランダムに応答時間を要するようにしています。
API Management は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/api-management/import-function-app-as-api)を参考に Function Apps の構成を行います。

どの Function Apps で時間を要しているのかを特定します。

## API Management 観点の確認
APIM Management では、GUI や診断設定のログよりバックエンドの応答時間が確認できます。
- 分析ブレード

  分析 ブレードや概要 ブレードの監視を確認いただくとバックエンドのデータが確認できます。今回は 3 回のリクエストがあり、平均で 14.64 秒の応答時間がかかっていることがわかります。

[分析 ブレードの表示例]

![image-8f4b121f-7067-472d-8ecb-16e28d748aca.png]({{site.baseurl}}/media/2023/03/image-8f4b121f-7067-472d-8ecb-16e28d748aca.png)


- 診断設定ログ

  API Management の[診断設定を有効化](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-use-azure-monitor#resource-logs)すると、API Management のログが確認できます。
今回は 1 つの API のみとなりますが、TotalTime と BackendTime を並べてみてみるとほぼ同じであることがわかります。API のバックエンドの Function Apps が怪しいですね。

![image-4b36a42e-d419-4352-bcc3-fc038f4e7d44.png]({{site.baseurl}}/media/2023/03/image-4b36a42e-d419-4352-bcc3-fc038f4e7d44.png)

  ```
  //クエリ例
  ApiManagementGatewayLogs
  | project TimeGenerated, TotalTime, ResponseCode, BackendTime,ApiId
  | summarize avg(TotalTime),avg(BackendTime) by bin(TimeGenerated,1s)
  | render timechart
  ```


## Function Apps 観点の確認
### バックエンドその1：Function Apps1(HakurodaChain001)
API Management から呼び出された時間帯で Function Apps1 のログを確認しましょう。

Application Insights を開き、下記のクエリを実行します。requests テーブルから [operation_Id](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/correlation#data-model-for-telemetry-correlation)(一連の実行を表す一意の ID でテーブル間で共有されます) を取得し、他のテーブルも operation_Id で検索しています。

```
//クエリ例
let operationIds = requests 
| where timestamp between(datetime("YYYY-MM-DD HH:MI")..10m)
| where operation_Name =~ "<APIM から呼び出されるトリガー名>"
| project operation_Id;
union traces,requests,dependencies
| where operation_Id in (operationIds)
//見やすくするための project
| project timestamp, message,itemType, operation_Id, id, name, url, resultCode, duration, target, type
| order by timestamp asc 
```

複数のログが確認できるため、1 つの operation_Id `99824b7c8c56e29ef39e980703af9421` を選択しフィルタします。それぞのログ元のテーブルは itemType から確認ができ、requests(赤ハイライト)、traces(黄ハイライト)、dependencies(青ハイライト)としています。

[Function Apps1(HakurodaChain001) ログの確認結果]

![image-4f568876-4e1f-433b-a59f-602629d10c3b.png]({{site.baseurl}}/media/2023/03/image-4f568876-4e1f-433b-a59f-602629d10c3b.png)

- requests テーブルの duration 列からは、この Function Apps/HakurodaChain001 に到達した HTTP トリガー/ChainHTTP001 の実行に 11138.1142 ミリ秒要していることがわかります。前述で確認した API Management のログでも同様の時間が確認できていました。
- traces テーブルからは、HTTP トリガーの実行時間がわかります。Excuted から始まるログの Duration を確認すると 11132 ミリ秒要していることがわかります。おおむね requests テーブルの値と同一で、通信そのもの(HTTP 通信の確立や HTTP 通信の確立後にアプリケーションの開始が遅れたなど)に時間がかかったわけではなく、アプリケーションの実行に時間がかかったことがわかります。
- dependencies テーブルからは、外部依存関係と要した時間がわかります。target 列に記録されている Function Apps/HakurodaChain002 の "/api/ChainHTTP2" にリクエストしていることがわかります。この duration が 11131.6227 ミリ秒要していることがわかります。

以上から API Management からバックエンドの API として HakurodaChain001 を呼び出していますが、その応答時間のほとんどは dependencies テーブルの通り HakurodaChain002 が使っているとわかります。

**Azure Functions では既定でログ サンプリングが有効化されています。場合によってはすべてのログが記録されないことがございますため、こちらの[ブログ](https://azure.github.io/jpazpaas/2023/03/02/LogSampling-OnAzureFunctions.html)を参考にログ サンプリングを無効化することをご検討ください。**

### バックエンドその2：Function Apps2(HakurodaChain002)
同様の手順で Function Apps/HakurodaChain002 を確認します。先の operation_Id `99824b7c8c56e29ef39e980703af9421` を利用します。

[Function Apps2(HakurodaChain002) ログの確認結果]

![image-c5972a66-8e4e-427e-b1f0-735ed020fe3a.png]({{site.baseurl}}/media/2023/03/image-c5972a66-8e4e-427e-b1f0-735ed020fe3a.png)

- requests テーブルの duration 列からは、この Function Apps/HakurodaChain002 に到達した HTTP トリガー/ChainHTTP002 の実行に 11077.9077 ミリ秒要していることがわかります。前述で確認した API Management のログでも同様の時間が確認できていました。
- traces テーブルからは、HTTP トリガーの実行時間がわかります。Excuted から始まるログの Duration を確認すると 11073 ミリ秒要していることがわかります。また、traces テーブルの `Start Sleep.` から `End Sleep.` までの間で約 1 秒要していることがわかります。
- dependencies テーブルからは、target 列に記録されている Function Apps/HakurodaChain003 の "/api/ChainHTTP3" にリクエストしていることがわかります。この duration が 10059.2205 ミリ秒要していることがわかります。

以上から Function Apps/HakurodaChain001 から呼び出された HTTP トリガー/HakurodaChain002 は、約 11 秒要していますが、内訳は  HTTP トリガー/HakurodaChain002 の内部処理で 1 秒要しており、dependencies テーブルより HakurodaChain003 が約 10 秒使っているとわかります。

### バックエンドその3：Function Apps3(HakurodaChain003)
最後に同様の手順で Function Apps/HakurodaChain003 を確認します。 operation_Id `99824b7c8c56e29ef39e980703af9421` を利用します。

[Function Apps3(HakurodaChain003) ログの確認結果]

![image-272b1048-64d8-4575-8172-6dec932624f5.png]({{site.baseurl}}/media/2023/03/image-272b1048-64d8-4575-8172-6dec932624f5.png)

- requests テーブルの duration 列からは、この Function Apps/HakurodaChain003 に到達した HTTP トリガー/ChainHTTP003 の実行に 10008.9659 ミリ秒要していることがわかります。
- traces テーブルからは、HTTP トリガーの実行時間がわかります。Excuted から始まるログの Duration を確認すると 10002 ミリ秒要していることがわかります。また、traces テーブルの `Start Sleep.` から `End Sleep.` までの間で約 10 秒要していることがわかります。

以上から Function Apps/HakurodaChain002 から呼び出された HTTP トリガー/ChainHTTP003 は、約 10 秒要しています。dependencies テーブルにはログが確認できないので、この Function Apps/HakurodaChain003 が最下層の実行となっており、この応答時間が結果的に API Management で確認できていたものと考えられます。

### Function Apps 観点の確認のまとめ
最後に上記でトレースした全体を並べて確認してみましょう。Application Insights を共有しておらず、それぞれの Application Insights にて同一の Log Analytics ワークスペースを利用している場合には、以下のように確認できます。
各 Function Apps のトリガーが入れ子になって順々に呼び出されていることが確認できました。

```
//LogAnalytics のクエリ例
union AppTraces, AppRequests, AppDependencies
| where OperationId =~ "<OperationId>"
| project TimeGenerated, Message, OperationName, Type, Id, Name, Url, ResultCode, DurationMs, Target, DependencyType
| order by TimeGenerated asc 
```

![image-f3cbefc0-80f3-4c76-b633-b58c4b88ddae.png]({{site.baseurl}}/media/2023/03/image-f3cbefc0-80f3-4c76-b633-b58c4b88ddae.png)

## API Management と Function Apps を並べてみる
Function Apps 単体での確認してきましたが、API Management のリクエストから紐づけて Functions Apps で利用されている operation_Id を特定する方法を解説します。

API Management は Application Insights と[こちら](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-howto-app-insights)の方法で統合できます。Application Insights との統合後に requests テーブルに API Management のログが確認できます。requests テーブルの id 列を確認します。今回 id は `|aa296dd5-4c581d75e1ad5e50.` です。

```
//クエリ例
requests 
| where cloud_RoleName startswith "<API Management 名>"
```

![image-c925075b-1641-49fe-9bf6-6a3adc665a7c.png]({{site.baseurl}}/media/2023/03/image-c925075b-1641-49fe-9bf6-6a3adc665a7c.png)

確認した id をもとに Function Apps の operation_Id  を特定します。API Management と直接通信する Functions Apps(今回は HakurodaChain001) の Application Insights を確認します。requests テーブルの operation_ParentId に先ほど確認した id を検索条件にすると、operation_Id `99824b7c8c56e29ef39e980703af9421` を確認することができます。

```
//クエリ例
requests 
| where operation_ParentId startswith "<id>"
```

![image-f24b4169-3a77-4bc0-8b6a-082e5d2e45cc.png]({{site.baseurl}}/media/2023/03/image-f24b4169-3a77-4bc0-8b6a-082e5d2e45cc.png)

## おまけ：データモデル

これまで利用してきた Application Insights のテーブルの[データモデル](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/data-model)について解説します。今回利用したログは requests(赤枠)、traces(黄枠)、dependencies(青枠)でした。他にもいくつかログがありますが、今回確認しなかった主なログとして例外が発生した場合には exceptions に記録されます。

![image-6ed2ff02-2654-41a6-b83c-d6848d2576f2.png]({{site.baseurl}}/media/2023/03/image-6ed2ff02-2654-41a6-b83c-d6848d2576f2.png)

各テーブルのデータ モデルは各種テーブルごとのドキュメント（[データ モデル](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/data-model) からそれぞれ Request、依存関係、例外、Trace、Event、メトリック、PageView、Contect のドキュメント）を参照ください。

もし、時系列で並べてみてもアプリケーションの実行は時間を要しておらずそれぞれの通信の間が遅いなどございましたら、今回紹介させていただいた情報をおまとめいただき Azure サポート窓口へお問い合わせください。
性能のトラブルシューティングに参考になれば幸いです。

<br>
<br>

---

<br>
<br>

2023 年 04 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>