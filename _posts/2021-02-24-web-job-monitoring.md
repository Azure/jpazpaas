---
title: "Application Insights を用いた Web ジョブ 実行状況の死活監視"
author_name: "Kohei Mayama, Tomomi Hirotane"
tags:
    - App Service
    - Web Apps
    - WebJobs
---

Web ジョブ とは Web Apps の一部の機能として提供しており、スクリプト、プログラムファイルをアップロードを行い、継続的もしくはある時間に起動するといったタイマートリガーとしてタスクを実行することができます。しかしながら、Web ジョブ の機能のみで、タスク全体の処理が動作していないといった異常を検知することができません。

[Azure App Service で Web ジョブを使用してバックグラウンド タスクを実行する](https://docs.microsoft.com/ja-jp/azure/app-service/webjobs-create)

Web ジョブ のタスクの監視を実施するために、Kudu から提供している Web ジョブ API と Application Insights を組み合わせることにより、簡単に Web ジョブ の監視環境を構築することができます。今回の監視監視は、Azure プラットフォーム側で完結をしているため、監視に必要なアプリケーションの実装を行う必要はなく、監視アプリケーション自体の管理の手間が無くなるため、有用な一例としてご紹介いたします。

## 質問
Web Apps に設定した Web ジョブ が処理実行されない場合に、異常として検知する方法はありますか。
Web ジョブ に設定したバッチ処理の実行ログが出力されていない場合に異常を検知することで対応を検討していますが、Azure 機能で検知する方法があるか確認しています。

## 回答
Web ジョブ の実行状況は Web ジョブ 画面の「ログ」タブより確認ができるものの、Web ジョブ の実行状況に応じてアラートを発報する等、監視をする機能は提供されておりません。そのため、 Web Job の処理実行が正常にされていない場合に異常として検知するためには以下ご案内いたします手順にて監視およびアラートの設定をいただけますと幸いです。

Web ジョブ の動作状況は下記「共通」にてご案内しております Web ジョブ API を利用することにより、実行状況をご確認いただけます。併せて、以下の方法をご利用いただくことで、継続的に状態確認のためのリクエストを Web ジョブ に送信し、異常を検知いただけます。
- [方法 1：Azure Functions を使用したカスタム可能性テストを作成する方法](#method1)
- [方法 2：Application Insights の可用性テスト (Standard test) を利用する方法](#method2)

### 共通：Web ジョブ API を利用した死活監視

Web Job 画面から `ログ` へ移動すると、各 Web Job タスクの状況が表示され、該当の Web Job へ遷移するとトリガー毎の Status を確認することができます。

![1-webjob-ffd5eaee-40e6-4356-81b8-f33eb623925e.png]({{site.baseurl}}/media/2021/02/1-webjob-ffd5eaee-40e6-4356-81b8-f33eb623925e.png)

こちらの状態を API 経由にて取得することができ、以下の Web Job API に関する弊社開発部門の提供ドキュメントでは、{job name} を Web Job 内で定義しているスクリプトの名前に置き換え、該当の Web ジョブ API への送信を行います。

[Get a specific triggered job by name](https://github.com/projectkudu/kudu/wiki/WebJobs-API#get-a-specific-triggered-job-by-name)
> GET /api/triggeredwebjobs/{job name}

#### 認証情報
Web ジョブ API の実行には、認証情報が必要となり以下のように、当該の Web ジョブ が実行しております App Service よりユーザの認証情報を取得し、Web ジョブ API 実行時にその認証情報を付与します。

[User credentials (aka Deployment Credentials)](https://github.com/projectkudu/kudu/wiki/Deployment-credentials#user-credentials-aka-deployment-credentials)

認証情報を取得するには、当該 App Service の左メニューブレードの デプロイメントセンター > FTPS 資格情報 の画面にて、赤枠で囲まれているユーザ名とパスワードをコピーします。ユーザ名は、"$" 以降を含めた値を取得します。

![2-webjob-a83528f5-e5ea-4ed9-97c8-e1e91084ad92.png]({{site.baseurl}}/media/2021/02/2-webjob-a83528f5-e5ea-4ed9-97c8-e1e91084ad92.png)

#### curl コマンドを利用したテスト
以下は curl コマンドを使用した Web ジョブ API を利用して特定の Web ジョブ スクリプトの状態を取得します。`<>` で囲まれている値は使用している環境にて合わせて変更をします。

```bash
webappsname="<WebApp名>"
webjobname="<WebJob名>"
username="<ユーザ名>"
password="<パスワード>"
token=$(echo -n $username:$password | base64)

curl -H "Authorization:Basic $token" -H "accept: application/json" https://$webappsname.scm.azurewebsites.net/api/triggeredwebjobs/$webjobname/
```

### <a id="mothod1">方法 1：Azure Functions を使用したカスタム可能性テストを作成する方法</a>
Azure Funcitons を使用してカスタム可用性テストを作成いただけます。作成方法については下記資料に記載がございますのでご参照ください。

[Azure Functions を使用してカスタム可用性テストを作成して実行する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/availability-azure-functions) 


上記資料のサンプルコードの `runAvailabilityTest.csx` ファイルにて、カスタムテストのロジックをお客様ご自身でご実装いただきますが、「共通」でご案内した Web ジョブ の状態確認 API をご利用いただければと存じます。

### <a id="mothod2">方法 2：Application Insights の可用性テスト (Standard test) を利用する方法</a>
Application Insights の可用性テスト (Standard test)　を利用し、Web ジョブを監視する方法についてご紹介します。Application Insights の可用性テスト (Standard test) については下記資料に詳細が記載されておりますのでご参照ください。
[標準テストを作成する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/availability-standard-tests) 

【手順】
1. 対象の App Service のポータルにアクセスしていただき、左メニュー [Application Insights] > [Application Insights データの表示] の順に遷移してください。
2. 左メニューより [可用性] に遷移してください。
3. 上部の [+ Add Standard (preview) test] をクリックすると、テスト作成画面が表示されますので、以下項目をご設定のうえ、テストを作成してください。

```
テスト名：任意のテスト名
URL : Web Job の状態取得 API の URL（「共通」でご案内した内容をご参照ください）
HTTP要求の動詞 : GET
カスタム ヘッダーの追加 : 
　ヘッダー名 : Authorization
　ヘッダー値 : Basic <Base64エンコードのユーザー名とパスワード>（「共通」でご案内した内容をご参照ください）
HTTP応答 : 
　応答コードが次の値に等しい : 200
コンテンツの一致 : 
　見つかった場合は合格
　コンテンツに含める必要がある対象 : Runnning
```
 
4. 作成されたテストの「規則（アラート）ページを開く」を選択します。

![3-webjob-5beef572-e4d9-4bd7-a25b-d4a7e95ff60c.png]({{site.baseurl}}/media/2021/02/3-webjob-5beef572-e4d9-4bd7-a25b-d4a7e95ff60c.png)

 
5. アラート ルール の編集画面にて、通知の条件（エラー回数のしきい値などがご設定いただけます）や通知先（SMSやメールをご指定いただけます）をご設定ください。
詳しくは以下の公式情報をご参照ください。
[Azure Monitor アラートのアラート プロセス ルール](https://docs.microsoft.com/ja-jp/azure/azure-monitor/alerts/alerts-action-rules?tabs=portal) 

 
以上の手順にて、Web Job の状態が「Runnning」以外の状態を検知して、アラートを発することが可能となりますので、お試しいただければと存じます。

## アクセス制限について
上記二つの方法を設定した際のアクセス制限についても多くお問い合わせ頂いておりますので、それぞれの方法で必要なアクセス制限について下記ご案内させていただきます。

### 方法 1：Azure Functions を使用したカスタム可能性テストを作成する方法
こちらの方法では Azure Functions から Web Apps へアクセスが可能となっている必要がございます。
そのため、Web Apps のアクセス制限機能より設定をする場合は、まず Azure Functions の送信 IP を固定し、Azure Functions の送信 IP をアクセス制限で許可いただけますと幸いです。
Azure Functions の送信 IP を固定する方法につきましては下記資料に記載がございますのでご参照いただけますと幸いです。

[Azure 仮想ネットワーク NAT ゲートウェイを使用して Azure Functions の送信 IP を制御する](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-how-to-use-nat-gateway)


### 方法 2：Application Insights の可用性テスト (Standard test) を利用する方法
こちらの方法では、該当 Web Apps の SCM サイトのアクセス制限にて「ApplicationInsightsAvailability」というサービスタグを許可いただけますと幸いです。
以下に手順をご案内いたします。
 
【手順】
1. ポータルより SCM へのアクセス制限画面にアクセスしてください。
2. [+ 規制の追加] をクリックしていただき、「アクセス制限の編集」を表示いたします。
3. 以下の設定例をご参考に、サービス タグ「ApplicationInsightsAvailability」への許可を追加していただき、[ルールの更新] を押下して追加実行してください。

![access_restriction-c114958f-9a1c-4ac7-a342-f5bf7a99b22f.jpg]({{site.baseurl}}/media/2021/02/access_restriction-c114958f-9a1c-4ac7-a342-f5bf7a99b22f.jpg)

#### 補足
サービスタグ「ApplicationInsightsAvailability」は下記資料にございますように、Application Insights の可用性テストに用いられる IP アドレスとなります。

[Azure サービス タグの概要](https://docs.microsoft.com/ja-jp/azure/virtual-network/service-tags-overview#available-service-tags)
```
 ApplicationInsightsAvailability	Application Insights の可用性。
```

また、可用性テストをご利用いただきますと、世界各地の複数ポイント (上記のサービスタグに含まれる IP アドレス) から該当 Web Apps に対して Web 要求が送信されます。

[Application Insights 可用性テスト](https://docs.microsoft.com/ja-jp/azure/azure-monitor/app/availability-overview)
```
Application Insights は、世界各地の複数のポイントから定期的にアプリケーションに Web 要求を送信します。
```


<br>
<br>

---

<br>
<br>

2022 年 10 月 06 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>