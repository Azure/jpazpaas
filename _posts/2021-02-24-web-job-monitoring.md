---
title: "Application Insights を用いた Web ジョブ 実行状況の死活監視"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Web Apps
    - WebJobs
---

Web ジョブ とは Web Apps の一部の機能として提供しており、スクリプト、プログラムファイルをアップロードを行い、継続的もしくはある時間に起動するといったタイマートリガーとしてタスクを実行することができます。しかしながら、Web ジョブ の機能のみで、タスク全体の処理が動作していないといった異常を検知することができません。

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-0-webjob.png" width="30%">

[Azure App Service で Web ジョブを使用してバックグラウンド タスクを実行する](https://docs.microsoft.com/ja-jp/azure/app-service/webjobs-create)

Web ジョブ のタスクの監視を実施するために、Kudu から提供している Web ジョブ API と Application Insighst を組み合わせることにより、簡単に Web ジョブ の監視環境を構築することができます。今回の監視監視は、Azure プラットフォーム側で完結をしているため、監視に必要なアプリケーションの実装を行う必要はなく、監視アプリケーション自体の管理の手間が無くなるため、有用な一例としてご紹介いたします。

## 質問
Web Appsに設定した Web ジョブが処理実行されない場合に、異常として検知する方法はありますか。
Web ジョブ に設定したバッチ処理の実行ログが出力されていない場合に異常を検知することで対応を検討していますが、Azure で検知する方法があるか確認しています。

## 回答
Web ジョブ 自体に監視機能は含まれておりませんが、Web ジョブ の実行を管理するための WebJobs API と Application Insights を組み合わせることで Web ジョブ のタスク実行状況を監視することができます。

## WebJobs API より状態を取得

Web ジョブ 画面から `ログ` へ移動すると、各 Web ジョブ タスクの状況が表示され、該当の Web ジョブ へ遷移するとトリガー毎の Status を確認することができます。
<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-1-webjob.png" width="70%">

こちらの状態を API 経由にて取得することができ、以下の WebJobs API に関する弊社開発部門の提供ドキュメントでは、{job name} を Web ジョブ内で定義しているスクリプトの名前に置き換え、該当の WebJobs API への送信を行います。

[Get a specific triggered job by name](https://github.com/projectkudu/kudu/wiki/WebJobs-API#get-a-specific-triggered-job-by-name)
> GET /api/triggeredwebjobs/{job name}

### 認証情報
WebJobs API の実行には認証情報が必要となるため、Web ジョブ を実行する App Service のユーザの認証情報を取得します。

[User credentials (aka Deployment Credentials)](https://github.com/projectkudu/kudu/wiki/Deployment-credentials#user-credentials-aka-deployment-credentials)

認証情報を取得するには、当該 App Service の左メニューブレードの デプロイメントセンター > FTPS 資格情報 の画面にて、赤枠で囲まれているユーザ名とパスワードをコピーします。ユーザ名は、"$" 以降を含めた値を取得します。

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-2-webjob.png" width="70%">

### curl コマンドを利用したテスト
WebJob API を利用して特定の WebJob スクリプトの状態を取得します。"<>" で囲まれている値は使用している環境にて合わせて変更をします。

```bash
webappsname="<WebApp名>"
webjobname="<WebJob名>"
username="<ユーザ名>"
password="<パスワード>"
token=$(echo -n $username:$password | base64)

curl -H "Authorization:Basic $token" -H "accept: application/json" https://$webappsname.scm.azurewebsites.net/api/triggeredwebjobs/$webjobname/
```

## Application Insights の設定

### Visual Studio から Web テストプロジェクトの構築
Visual Studio Enterprise より、Application Insihgts 内で用いるテストファイルの作成を行います。Visual Studio は Professional の場合だと、Application Insighsts のテストファイルを作成することができず、Web パフォーマンスとロードテストツールは、Visual Studio Enterprise や Ultimate が必要となります。

テストファイルを作成するために、Visual Studio にてコンポーネントのインストールとロードテストプロジェクトを作成します。詳しい情報につきましては、弊社提供のドキュメントを参照ください。

[クイック スタート: ロード テスト プロジェクトを作成する](https://docs.microsoft.com/ja-jp/visualstudio/test/quickstart-create-a-load-test-project?view=vs-2019)


### Web テストのファイルを作成
プロジェクト作成後に、`webtest` ファイルが作成していますので、以下の要素を追加します。
- Web ジョブ API に対する URL 
- 認証情報である Authorization ヘッダ（curl コマンドで利用した Authorization ヘッダの値を利用します）
- 検証規則（レスポンスにて探す文字列）

**webtest ファイルの構成**

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-3-webjob.png" width="70%">

**検証テキスト**

Web ジョブ API のレスポンスから、Success という文字列を検索するために、検索テキストに `Success` を入力します。

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-4-webjob.png" width="70%">


最終的な webtest ファイルは以下のような構成となります。こちらのように xml ファイルから Web Test の作成を行っても問題はございませんが、動作に保証はなくかつサポートをいたしかねる状況でございますので、Visual Studio Enterprise より作成することを推奨します。

```xml
<?xml version="1.0" encoding="utf-8"?>
<WebTest Name="WebTest1" Id="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" Owner="" Priority="xxxxxxxxxx" Enabled="True" CssProjectStructure="" CssIteration="" Timeout="0" WorkItemIds="" xmlns="http://microsoft.com/schemas/VisualStudio/TeamTest/2010" Description="" CredentialUserName="" CredentialPassword="" PreAuthenticate="True" Proxy="default" StopOnError="False" RecordedResultFile="" ResultsLocale="">
  <Items>
    <Request Method="GET" Guid="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" Version="1.1" Url="https://xxxxxxxxxxxxxx.scm.azurewebsites.net/api/triggeredwebjobs/xxxxxxxxxxxxxxxxx" ThinkTime="0" Timeout="300" ParseDependentRequests="True" FollowRedirects="True" RecordResult="True" Cache="False" ResponseTimeGoal="0" Encoding="utf-8" ExpectedHttpStatusCode="200" ExpectedResponseUrl="" ReportingName="" IgnoreHttpStatusCode="True">
      <Headers>
        <Header Name="Authorization" Value="Basic xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" />
      </Headers>
    </Request>
  </Items>
  <ValidationRules>
    <ValidationRule Classname="Microsoft.VisualStudio.TestTools.WebTesting.Rules.ValidationRuleFindText, Microsoft.VisualStudio.QualityTools.WebTestFramework, Version=10.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" DisplayName="検索テキスト" Description="指定されたテキストが応答に存在するかを確認します。" Level="High" ExectuionOrder="BeforeDependents">
      <RuleParameters>
        <RuleParameter Name="FindText" Value="Success" />
        <RuleParameter Name="IgnoreCase" Value="False" />
        <RuleParameter Name="UseRegularExpression" Value="False" />
        <RuleParameter Name="PassIfTextFound" Value="True" />
      </RuleParameters>
    </ValidationRule>
  </ValidationRules>
</WebTest>
```

### Webtest ファイルを Application Insights へアップロード
Azure Portal の Application Insights 左ブレードメニューの `可用性` へ移動し、上部にある `テストの追加` を選択します。
テストの種類を `複数ステップの Web テスト` を選択し、作成した webtest ファイルをアップロードします。

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-5-webjob.png" width="70%">

### 監視状況の確認
テストの追加を行い、可用性のグラフを散布図へ変更すると監視状況を可視化することができます。

<img alt="assign-role" src="{{site.baseurl}}/media/2021/02/2021-02-24-6-webjob.png" width="70%">


<br>
<br>

---

<br>
<br>

2020 年 2 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>