---
title: "Azure Functions（関数アプリ）の用語と関連する動作について"
author_name: "Kohei Mayama"
tags:
    - Function Apps
---

# はじめに
お世話になっております。App Service サポート担当の間山です。いつも Azure Functions（関数アプリ）をご利用いただきましてありがとうございます。関数アプリをご利用いただく中で関連する多くの単語が存在しております。類似している単語があることや、この部分で使われている単語は関数アプリではどのような部分に属しているのか混乱をしてしまう場面に遭遇することがございます。本ブログでは、関数アプリにて使用される用語の説明や関数アプリ内で使用されるコンポーネントについてご紹介して参ります。

# 関数アプリに関連する基本的な用語
## 関数アプリ、FunctionApp、App、FunctionsHost
関数アプリは関数コードを実行をホストするリソースとなります。HTTP Trigger や Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を行います。関数アプリでは、使用するランタイムスタック（言語）やバージョンを選択します。

[Function App を作成する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal#create-a-function-app)

## ワーカープロセス、worker process、言語ワーカー プロセス、LanguageWorker
関数コードを動作するためのプロセス（language worker）となります。Kudu と呼ばれるデバッグツールの Process Explorer より、w3wp.exe とdotnet.exe が分かれております。（この場合は .NET isolated のランタイムスタックであるため、FunctionHost と LanguageWorker が分離しております。）worker process は関数コードを実行するプロセスとなるため、dotnet.exe となります。FUNCTIONS_WORKER_PROCESS_COUNT のアプリケーション設定は、関数コードを実行する LanguageWorker のプロセス数を制御する設定となりますため、以下のスクリーンショットにございますdotnet.exe のプロセス数となります。

![image-5abc4e91-f130-4633-9c04-328d18b0046f.png]({{site.baseurl}}/media/2023/04/image-5abc4e91-f130-4633-9c04-328d18b0046f.png)

[FUNCTIONS_WORKER_PROCESS_COUNT](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#functions_worker_process_count)

## 関数、関数コード
実際のイベントに対する処理を実施するアプリケーションコードとなります。イベントに対する処理を実施する関数コードはトリガー（HTTP, タイマー）と呼ばれるイベントのテンプレートを設定することができ、イベントが検知されると関数コードのアプリケーションが動作します。以下は関数コードトリガーの一例として HTTP トリガーの作成方法がございます。

[HTTP トリガー関数の作成](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal#create-function)

## インスタンス、Worker インスタンス、VM インスタンス
関数アプリを動作する配下の VM インスタンス（Worker インスタンス）となります。Azure Functions より提供しているインスタンスは、従量課金プラン、App Service プラン、Elastic Premium プランとなります。

[プランの概要](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-scale#overview-of-plans)

## Plan Scale Out（プランのスケールアウト）、App Scale Out（アプリのスケールアウト）
Elastic Premium プランで制御する VM インスタンス数の定義と VM インスタンス上で動作している各関数アプリを配置する VM インスタンス数の定義を実施します。例えば以下のスクリーンショットにございます内容では、Plan Scale out の Maximum Burst は20 となっており、App Scale out の Always Ready Instances（常時使用可能なインスタンス）は 1 でございます。常時使用可能なインスタンスが 1 になりますと、常に関数アプリが 1 インスタンスで動作することを決定しており、関数コードのイベント数の負荷により、VM インスタンス は Elastic Premium Plan で設定された Maximum Burst である20 インスタンスまで増加することが出来ます。 

[プランと SKU の設定](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-premium-plan?tabs=portal#plan-and-sku-settings)
> プランを作成するときは、2 つのプラン サイズ設定があります。インスタンスの最小数 (またはプラン サイズ) と最大バースト制限です。

> 常時使用可能なインスタンス数を超えるインスタンスがアプリに必要な場合、インスタンスの数が最大バースト制限に達するまでアプリのスケール アウトを続けることができます。

![image-084e542a-4e76-438c-9c6d-0d3a937583bb.png]({{site.baseurl}}/media/2023/04/image-084e542a-4e76-438c-9c6d-0d3a937583bb.png)

Elastic Premium Plan にて割り当て済みのインスタンス数を確認する方法としては弊社サポートチームブログより公開をしておりますためご参照ください。

[Function App の Elastic Premium Plan にて割り当て済みのインスタンス数を確認する方法](https://jpazpaas.github.io/blog/2022/08/09/how-to-check-elastic-premium-plan-function-app-allocated-instance-counts-history.html)
 
---

# App Service / Azure Functions アーキテクチャについて
前述の Azure Function の基本的な用語より、簡単ではございますが、App Service / Azure Functions アーキテクチャについて説明をいたします。App Service が登場する理由といたしましては、Azure Functions は App Service プラットフォーム上にて動作が行われております。まずはじめに、App Service のアーキテクチャとなりますが、App Service は 単独の VM で構成されているのではなく、大きく分けて FrontEnd と呼ばれるクライアント側のリクエストを受け付けるコンポーネントとFrontEnd からのリクエストをWorker インスタンス（VM）と呼ばれるお客様のアプリケーションを実行するコンポーネントに分けられております。
Azure Functions は App Service の Worker インスタンスコンポーネント上にて動作をいたします。App Service のアーキテクチャーについては下記資料も併せてご参照ください。

[Inside the Azure App Service Architecture](https://learn.microsoft.com/ja-jp/archive/msdn-magazine/2017/february/azure-inside-the-azure-app-service-architecture)
 
[[日本語訳] Inside the Azure App Service Architecture](https://qiita.com/mym/items/1d364c0251daf88d417d)

Azure Functions は Function host と呼ばれる関数アプリを制御するプロセスと Language Worker と呼ばれる関数アプリにて指定されたランタイムスタックの関数コードを動作するプロセスに分かれております。Azure Functions 開発部門では複数のプロジェクトとして各 GitHub リポジトリとして公開をしており、以下に Azure Functions プロジェクトの GitHub リポジトリ一覧、Function host リポジトリと一例として .NET の Language Worker のリポジトリの内容がございますためご参考いただけますと幸いでございます。

- [GitHub repositories](https://github.com/Azure/Azure-Functions#github-repositories)
- [Azure/azure-functions-host](https://github.com/Azure/azure-functions-host)
- [Azure/azure-functions-dotnet-worker](https://github.com/Azure/azure-functions-dotnet-worker)
 
以下に App Service の Worker インスタンス内のAzure Functions の構成（Function Host, Language Worker）の概略図を記載をいたします。

![image-a1c005bf-4a04-46c1-bf60-2a43d42e9cf6.png]({{site.baseurl}}/media/2023/04/image-a1c005bf-4a04-46c1-bf60-2a43d42e9cf6.png)


## Azure Functions のスケーリングについて
Azure Functions では Worker インスタンスをスケーリング方法と 1 Worker インスタンス内に Azure Functions の Language Worker プロセスの増加方法がございます。Azure Functions でのパフォーマンスを向上するために必要な 1 つの要因となりますが、アプリケーションの処理内容にも依存するため明確な設定値をご案内することは難しく、ご自身の環境にて負荷テストを実施していただきチューニングをご検討いただけると幸いです。

### Worker インスタンスのスケーリング 
Azure Functions のスケーリングはイベントベースとなりますため、プランの CPU 使用率、メモリ使用量などのメトリクスを監視しているわけではございません。例えば、HTTP トリガーでは、 HTTP リクエストが増加すると、関数アプリへのイベントレートも増え、より多くのインスタンスが関数アプリに割り当てられる動作となっております。

[Azure Functions でのイベント ドリブン スケーリング](https://learn.microsoft.com/ja-jp/azure/azure-functions/event-driven-scaling)
> 従量課金プランと Premium プランでは、Azure Functions は、Functions ホストのインスタンスを追加することによって CPU およびメモリ リソースをスケーリングします。 インスタンスの数は、関数をトリガーするイベントの数に基づいて決定されます。

以下の図のように Worker インスタンスのスケーリング（1 台から 2 台へ増加）が行われると Worker インスタンス内にて Functions Host と Language Worker がセットで作成されます。
![function-scale-out-ade33606-2bff-4761-8c2c-1765bf0b01f9.png]({{site.baseurl}}/media/2023/04/function-scale-out-ade33606-2bff-4761-8c2c-1765bf0b01f9.png)

### Language Worker プロセスの増加
Worker インスタンスのスケーリング以外にも、1 つの Worker インスタンス内にて関数コードを実行するためのプロセスである Language Worker を増加する方法として、アプリケーション設定 `FUNCTIONS_WORKER_PROCESS_COUNT` より設定が実施できます。 1 Worker インスタンス上で動作する Language Workerの最大数を指定する値です。この値は最大で 10 まで設定でき、1 Worker インスタンスで処理されるリクエスト数が最大で `FUNCTIONS_WORKER_PROCESS_COUNT` 倍になります。なお、言語ワーカーは（イベント数に基づいた増減は行われず）`FUNCTIONS_WORKER_PROCESS_COUNT` 個になるまで 10 秒ごとに生成されます。

[FUNCTIONS_WORKER_PROCESS_COUNT](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#functions_worker_process_count)
> 言語ワーカー プロセスの最大数を指定します。既定値は 1 です。 許容される最大値は 10 です。 関数呼び出しは、言語ワーカー プロセス間で均等に分散されます。 言語ワーカー プロセスは、WORKER_PROCESS_COUNT によって設定されたカウントに達するまで、10 秒ごとに生成されます。 複数の言語ワーカー プロセスの使用は、スケーリングと同じではありません。

[複数のワーカー プロセスを使用する](https://learn.microsoft.com/ja-jp/azure/azure-functions/performance-reliability#use-multiple-worker-processes)
> 既定では、Functions のどのホスト インスタンスでも、単一のワーカー プロセスが使用されます。 特に Python などのシングルスレッド ランタイムによってパフォーマンスを向上させるには、FUNCTIONS_WORKER_PROCESS_COUNT を使用して、ホストあたりのワーカープロセス数を増やします (最大 10 まで)。 次に、Azure Functions は、これらのワーカー間で同時関数呼び出しを均等に分散しようとします。

以下の図のように Language Worker プロセスのスケーリング（1 から 2 へ増加）が行われると、1 Worker インスタンス内にて Functions Host としては 1 となりますが、Language Worker が 2 となります。
![function-process-count-36247160-fce7-4799-98b8-27bcca4d2574.png]({{site.baseurl}}/media/2023/04/function-process-count-36247160-fce7-4799-98b8-27bcca4d2574.png)


<br>
<br>

---

<br>
<br>

2023 年 04 月 25 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>