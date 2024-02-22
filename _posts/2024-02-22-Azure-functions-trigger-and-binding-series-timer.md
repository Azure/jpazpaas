---
title: "Azure Functions トリガー・バインディング シリーズ - Timer トリガー"
author_name: "Hayato Kuroda"
tags:
    - Function App
    - トリガー・バインディング シリーズ
---

# 概要と基本動作
[Timer トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cin-process&pivots=programming-language-csharp)は、指定されたスケジュールに沿って実行される関数です。スケジュールは CRON 式で指定します。

また、Timer トリガーは BLOB コンテナに配置したトリガーごとのファイル ロックを取得したインスタンスでのみ動作し、[シングルトン](https://github.com/Azure/azure-webjobs-sdk-extensions/wiki/TimerTrigger#singleton-locks)(ある時点で 1 インスタンスでのみ処理が動く)が実装されています。詳細動作はシングルトンのリンクにもございますが、いずれかのインスタンスで Timer トリガーが動作するようにすべてのインスタンスが当該ファイルのロックの取得を試みます。結果的にロックを取得できたインスタンスで Timer トリガーが動作し、ロックが取得できなかったインスタンスでは Timer トリガーは動作しません。
また、CRON 式にて指定したスケジュールはプロセス内でのみ保持されており、プロセス内でのみ次回の実行予定を把握しています。

本動作に利用されるファイルは BLOB コンテナに配置されたファイルとなります。Azure Functions リソースのアプリケーション設定 `AzureWebJobsStorage` に指定されたストレージ アカウントの BLOB コンテナー `azure-webjobs-hosts` が利用されます。`azure-webjobs-hosts/locks/<Azure Functions ホスト ID>/<Azure Functions 名>.<Timer トリガー名>.Run.Listener` に作成されたファイルが利用されます。

**Timer トリガー用のロック ファイル**

![image-a2ba9829-3fc2-4e59-b441-dba474edfe19.png]({{site.baseurl}}/media/2024/02/image-a2ba9829-3fc2-4e59-b441-dba474edfe19.png)

# 作成手順
Timer トリガーの作成方法は[チュートリアル](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-your-first-function-visual-studio#create-a-function-app-project)で紹介されているテンプレートをご利用ください。チュートリアルでは HTTP トリガーを利用しておりますが、同画面にて Timer トリガーを選択いただくことができます。

また、スケジュールを CRON 式で指定しますが、これはアプリケーション設定に外だしすることができます。C# の例ですが、TimerTrigger の引数で CRON 式の宣言部分を `%` 括りのアプリケーション設定名とします(上段)。ローカルの実行のため local.settings.json となりますが(下段)、CRON 式のスケジュールを値として設定します。Azure 上の Azure Functions リソースの場合には同項目をアプリケーション設定に設定ください。

![image-57cde1c6-2418-4a47-8eaf-2a1cfe957cd0.png]({{site.baseurl}}/media/2024/02/image-57cde1c6-2418-4a47-8eaf-2a1cfe957cd0.png)

# 実行状況をログで確認する
Timer トリガーの実行詳細をログから確認します。host.json の logging で下記のカテゴリのログを出力するように設定します。デバッグ用となるため、運用状況によって有効/無効化を検討ください。

**host.json**
```
"logging": {
  "logLevel": {
    "Host.Singleton": "Debug",
    "Host.Triggers.Timer": "Debug"
  }
}
```

**Application Insights で利用するクエリ**
```
traces
| where timestamp > datetime("yyyy-mm-dd hh:mi")
| where customDimensions.Category == "Host.Singleton" or customDimensions.Category == "Host.Triggers.Timer" or operation_Name =~ "TimerTrigger"
| order by timestamp asc
```

上記の設定を行った際に記録されるログの例です。1 行目にて前述の Timer トリガーのロック ファイルのロックが取得されていることが確認できます。この時の `cloud_RoleInstance` は `30c87c2bd10c437e5ec7dc4611fa40547c164d3231c557ebd2e973011322516a` となっています。一方で、`Unable to~` となっているログでは Timer トリガーのロック ファイルのロックが取得できなかったことが確認できますが、この時の `cloud_RoleInstance` は `9cc419d397c0e2a23ed385ba1416260bafe15894444cf3f90cc31127a0c25b4f` となっています。
また、4 行目にてプロセス内で認識されている実行予定を確認することができます。

**ログ出力例**

![image-19b65725-2e8f-4ef1-b16a-8703408795b5.png]({{site.baseurl}}/media/2024/02/image-19b65725-2e8f-4ef1-b16a-8703408795b5.png)

# 並列実行とチューニング
Timer トリガーはシングルトンで動作するため並列実行はございません。そのため、チューニング項目として挙げられる項目はございません。


# よくお問い合わせいただく動作
弊サポートにてよくお問い合わせいただく事例について紹介します。

## 意図しない時刻に実行された
Timer トリガーには [`UseMonitor` プロパティ](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cisolated-process%2Cnodejs-v4&pivots=programming-language-csharp#attributes)が存在しており、この値が `true` となっている際に意図しない時刻に Timer トリガーが実行されることがあります。

`UseMonitor` プロパティが `true` の場合には、BLOB コンテナに配置されたファイルに動作記録となるテキスト ファイルを保持し、（逐次ではなく）起動時に本ファイルを参照し Timer トリガーを実行するか否かを判断します。Azure Functionsが一定期間停止していたなどの理由でテキスト ファイルの `Next`（後述の UseMonitor プロパティ用のファイル参照）を超えた時刻で Azure Functions が起動した場合には、一度 Timer トリガーが実行されたのちにスケジュールに従います。

本動作に利用されるテキスト ファイルは BLOB コンテナに配置されたファイルとなります。Azure Functions リソースのアプリケーション設定 `AzureWebJobsStorage` に指定されたストレージ アカウントの BLOB コンテナー `azure-webjobs-hosts` が利用されます。`azure-webjobs-hosts/timers/<Azure Functions ホスト ID>/<Azure Functions 名>.<Timer トリガー名>.Run/status` に作成されたファイルが利用されます。記載のフォーマットは画像の通りであり、その内容はログからも確認いただくことができます。

**UseMonitor プロパティ用のファイル**

![image-32b0c857-7221-4ed0-89eb-dbe8b3287c5f.png]({{site.baseurl}}/media/2024/02/image-32b0c857-7221-4ed0-89eb-dbe8b3287c5f.png)

ログの記録例です。3-4 行目にて、テキスト ファイルの内容及びプロセス内で認識されている実行予定が確認できます。テキスト ファイルでは、次回の時刻が `2024-01-29T00:58:00+00:00` となっておりますが、スケジュール上の直後の実行は `01/29/2024 01:01:00Z` となっています。つまり、テキスト ファイル上は `2024-01-29T00:58:00+00:00` の実行がされていないことがわかります。
この状態になっているかつ `UseMonitor` プロパティが `true` の場合に、一度 Timer トリガーが実行されます。この予定外の実行の場合にはログに `Function 'TimerTrigger' is past due on startup. Executing now.` 及び `Trigger Details: UnscheduledInvocationReason: IsPastDue, OriginalSchedule: 2024-01-29T00:58:00.0000000+00:00` が記録されます。また、この予定外の実行後には当初のスケジュールに従いテキスト ファイルの更新及び実行がされていることが確認できます。

**ログ出力例**

![image-c777e277-0a42-404b-965d-f45862e9459a.png]({{site.baseurl}}/media/2024/02/image-c777e277-0a42-404b-965d-f45862e9459a.png)

## 意図した時刻に実行されない
1. Linux OS の従量課金プランで[タイムゾーン](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cisolated-process%2Cnodejs-v4&pivots=programming-language-csharp#ncrontab-time-zones)(アプリケーション設定 `WEBSITE_TIME_ZONE`)を設定している場合

Timer トリガーでは、CRON 式に設定したスケジュールに従うタイムゾーンをアプリケーション設定 `WEBSITE_TIME_ZONE` で指定することができます。しかし、このアプリケーション設定は Linux OS の従量課金プランでは利用できません。[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cisolated-process%2Cnodejs-v4&pivots=programming-language-csharp#ncrontab-time-zones)にご案内もございますが、ログからも確認ができます。

**ログ出力例**

![image-6622eb99-109c-4701-a017-a043715eafe5.png]({{site.baseurl}}/media/2024/02/image-6622eb99-109c-4701-a017-a043715eafe5.png)

2. Windows OS と Linux OS で誤ったタイムゾーンをしている場合

Timer トリガーに指定するタイムゾーンは、Windows OS と Linux OS で形式が異なります。Windows OS に Linux OS での形式で指定しているや、Linux OS に Windows OS での形式で指定している場合には正しくタイムゾーンが認識されません。

![image-982ca4f8-19d8-4c9a-aff6-746d42e50161.png]({{site.baseurl}}/media/2024/02/image-982ca4f8-19d8-4c9a-aff6-746d42e50161.png)

## Timer トリガーが並列して実行された
シングルトンの動作を実現しているのはファイルのロック機構によるものです。何らかの理由で Timer トリガーの実行中にファイルのロックが失われてしまった場合には、別のインスタンスでファイルのロックが行われ、Timer トリガーが並列して動作する可能性がございます。

一概にシナリオをご案内できかねるものの、高 CPU やメモリなどのコンピューティング リソースの不足によってファイルのロックが継続できないことが疑われます。コンピューティング リソースの利用状況は、Azure ポータルの問題の診断と解決より確認いただくことができます。

![image-8a867ad5-e5da-4a38-8b0e-2cb2c3928857.png]({{site.baseurl}}/media/2024/02/image-8a867ad5-e5da-4a38-8b0e-2cb2c3928857.png)

# まとめ
本ブログでは Timer トリガー関数の動作及び作成方法について整理しました。`UseMonitor` はデフォルトで `true` となっており、特に気づきにくくお問い合わせをいただくことがございます。Timer トリガーのトラブルシューティングの参考になれば幸いです。

# 参考ドキュメント
[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

[Azure Functions のタイマー トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cisolated-process%2Cnodejs-v4&pivots=programming-language-csharp)


<br>
<br>

---

<br>
<br>

2024 年 02 月 22 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>