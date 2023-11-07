---
title: "Function App の Elastic Premium Plan にて割り当て済みのインスタンス数を確認する方法"
author_name: "Kohei Mayama"
tags:
    - Function App
---

## はじめに
みなさん、こんにちは。App Service サポート担当の間山です。
関数アプリを実行するにあたり、ホストされるプランとして Elastic Premium Plan が提供されております。Elastic Premium Plan を使用して関数アプリの関数コードアプリケーションを実行することで、イベント量に応じて動的にインスタンスがスケールされます。今回の内容は、以下の弊社グローバルチームが執筆しております Tech Community 内のブログ記事ございます
[How to check Elastic Premium Plan Function App allocated instance counts history](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/how-to-check-elastic-premium-plan-function-app-allocated/ba-p/2697852)
の内容を日本ユーザ向けに解説をいたします。

## Elastic Premium Plan とは
関数アプリをホストする上で、3 種類のホスティングプラン（従量課金 Plan、App Service Plan、Elastic Premium Plan）の中から選択することが出来ます。App Service Plan を選択すると専用ホスティングプランとして通常の App Service（Web Apps）と同様に、指定されたインスタンス数に基づいて関数アプリの関数コードアプリケーションが実行されます。また、Web Apps と 関数アプリを 1 つの App Service Plan 内にて実行することが出来ます。スケーリングする場合には、手動でインスタンス追加のオペレーションや CPU 使用率や時刻による自動スケーリングを設定することが出来ます。
一方で、従量課金 Plan と Elastic Premium Plan は、関数コードアプリケーションのイベント数に応じてインスタンス数がスケーリングいたします。違いといたしましては、従量課金 Plan は他のユーザが実行している関数アプリとインスタンスを共有が行われ、Elastic Premium Plan は占有インスタンスとして、自身の関数アプリのみ実行されます。

Elastic Premium Plan を使用することで、App Service Plan のように関数アプリを占有のインスタンス上にて実行が行われ、従量課金プランのようにイベントに応じてスケールが行われる機能を利用することができ、以下の利点がございます。そのため、関数アプリをホスティングする際には、Elastic Premium Plan をご検討いただけると幸いです。

```
Elastic Premium Plan のホスティングでは、関数に次の利点があります。

- インスタンスを常にウォーム状態に維持することでコールド スタートを回避。
- 仮想ネットワーク接続。
- 無制限の実行期間 (60 分保証)。
- Premium インスタンス サイズ (1 コア、2 コア、4 コアのインスタンス)。
- 従量課金プランと比較して、予測可能な料金。
- 複数の Function App を含むプランでの高密度アプリ割り当て。
```

- [Azure Functions の専用ホスティング プラン](https://docs.microsoft.com/ja-jp/azure/azure-functions/dedicated-plan)
- [Azure Functions の従量課金プラン ホスティング](https://docs.microsoft.com/ja-jp/azure/azure-functions/consumption-plan)
- [Azure Functions の Premium プラン](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-premium-plan?tabs=portal)


## 割り当て済みのインスタンス数を確認する方法
### 1. 問題の診断と解決
関数アプリの Azure Portal 左メニューブレードにて `問題の診断と解決` がございます。検索ボックスから、HTTP と入力を行い候補として「HTTP 関数のスケーリング」にて表示されます。画面内では以下のスクリーンショットのように `関数アプリに割り当てられたワーカーの数` で表示されます。関数アプリに割り当てられたインスタンス数が 0 > 2 > 3 台へ遷移していることを確認することが出来ます。

**問題の診断と解決- HTTP 関数のスケーリングの確認例**

![image-0dddfad3-35ed-4f12-aa3a-5fb6f4fb2e20.png]({{site.baseurl}}/media/2022/08/image-0dddfad3-35ed-4f12-aa3a-5fb6f4fb2e20.png)


### 2. ライブメトリック
関数アプリにて Application Insights を有効後に、接続された Application Insights の Azure ポータル左メニューブレードにございます `ライブメトリック` へ移動することで、現在関数アプリに割り当て済みのインスタンス数を確認するすることが出来ます。以下のスクリーンショットの右上には、`2 servers online` と出力しており、関数アプリ配下では 2 台のインスタンスが動作していることを確認することが出来ます。さらには、スクリーンショット下部の `サーバー(10s の平均)` では割り当てられているインスタンス ID の一覧が表示されております。

**ライブメトリックの出力例**

![image-1d0c0b2d-ed00-4c17-8c77-0a201bd7b7b6.png]({{site.baseurl}}/media/2022/08/image-1d0c0b2d-ed00-4c17-8c77-0a201bd7b7b6.png)

### 3. スケールコントローラーのログ
スケールコントローラーと呼ばれるコンポーネントを使用して、イベントレートを監視し、スケールアウトとスケールインが決定されます。スケールコントローラーのログ出力となりますが、以下の弊社提供の公開情報にて詳細のご案内がございますため、ご参考になれば幸いです。

[スケール コントローラーのログを構成する](https://docs.microsoft.com/ja-jp/azure/azure-functions/configure-monitoring?tabs=v2#configure-scale-controller-logs)

**Application Insights のログ内でのクエリ例**

```
traces
| extend CustomDimensions = todynamic(tostring(customDimensions))
| where CustomDimensions.Category == "ScaleControllerLogs"
| where message == "Instance count changed"
| extend Reason = CustomDimensions.Reason
| extend PreviousInstanceCount = CustomDimensions.PreviousInstanceCount
| extend NewInstanceCount = CustomDimensions.CurrentInstanceCount
```

**スケールコントローラーのログ出力例**

![image-d01e8a43-615a-4d57-824e-2579c50f7770.png]({{site.baseurl}}/media/2022/08/image-d01e8a43-615a-4d57-824e-2579c50f7770.png)


また、スケールコントローラーのログを有効にしていない場合には以下のクエリを用いて Application Insights の trace テーブルから cloud_RoleInstance プロパティより summarize 構文を使用してインスタンス数も確認をすることが出来ます。

**Application Insights のログ内での summarize 構文クエリ例**

```
traces
| summarize count() by cloud_RoleInstance
```

**Application Insights のログ内での summarize 構文出力例**

![image-97ea69c5-a891-4574-8c79-997902ab4ca6.png]({{site.baseurl}}/media/2022/08/image-97ea69c5-a891-4574-8c79-997902ab4ca6.png)


## まとめ
Elastic Premium Plan は、自身の関数アプリを占有インスタンスにて実行することができ、関数コードアプリケーションのイベントに応じてインスタンスのスケーリングが行われます。インスタンス数が変動するため、課金に不安があると想定しておりますが、今回ご紹介いたしました 3 つの方法を利用して、例えば関数コードアプリケーションの負荷試験や突然のイベントが発生した際に、Elastic Premium Plan にて実行されているインスタンス数はどれくらい実行されているか確認することが出来ますため、ご参考いただけますと幸いでございます。


<br>
<br>

---

<br>
<br>

2023 年 11 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>