---
title: "Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について"
author_name: "Kohei Mayama"
tags:
    - Function Apps
---

# はじめに
お世話になっております。App Service サポート担当の間山です。いつも Azure Functions（関数アプリ）をご利用いただきましてありがとうございます。関数アプリをご利用いただく中で関連する多くの単語が存在しております。類似している単語があることや、使用されている単語は関数アプリではどのような部分に属しているのか混乱をしてしまう場面に遭遇することがございます。本ブログでは、関数アプリ内部アーキテクチャの概要や関数アプリにて使用される用語の説明や関数アプリ内で使用されるコンポーネントについてご紹介して参ります。

# App Service / Azure Functions アーキテクチャについて
Azure Functions は App Service プラットフォーム上にて動作が行われております。App Service は 単独の VM（Virtual Machine）で構成されているのではなく、大きく分けて FrontEnd と呼ばれるクライアント側のリクエストを受け付けるコンポーネントと FrontEnd からのリクエストを Worker インスタンス と呼ばれるお客様のアプリケーションを実行するコンポーネントに分けられております。

Azure Functions の ワーカープロセスは App Service の Worker インスタンスコンポーネント上にて動作をいたします。App Service のアーキテクチャーについては下記資料も併せてご参照ください。

[Inside the Azure App Service Architecture](https://learn.microsoft.com/ja-jp/archive/msdn-magazine/2017/february/azure-inside-the-azure-app-service-architecture)
 
[[日本語訳] Inside the Azure App Service Architecture](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html)

[App Service を構成する主要な要素とそれぞれの役割](https://jpazpaas.github.io/blog/2023/05/10/Appservice-components.html)


Azure Functions は ホストプロセス と呼ばれる関数アプリを制御するプロセスと 言語ワーカープロセス と呼ばれる関数アプリにて指定されたランタイムスタックの関数コードを動作するプロセスに分かれております。Azure Functions 開発部門では複数のプロジェクトとして各 GitHub リポジトリとして公開をしております。以下に Azure Functions プロジェクトの GitHub リポジトリ一覧、Function host リポジトリと一例として .NET の Language Worker のリポジトリの内容がございますためご参考ください。

- [GitHub repositories](https://github.com/Azure/Azure-Functions#github-repositories)
- [Azure/azure-functions-host](https://github.com/Azure/azure-functions-host)
- [Azure/azure-functions-dotnet-worker](https://github.com/Azure/azure-functions-dotnet-worker)
 
以下に App Service の Worker インスタンス内のAzure Functions の構成（Function Host, Language Worker）の概略図を記載をいたします。（.NET in-process の場合には、FunctionsHost と Language Worker は同じプロセスにて動作します。）

![function-app-arcitecture-15a96720-2a65-44d4-86f9-cdd283d46393.png]({{site.baseurl}}/media/2023/08/function-app-arcitecture-15a96720-2a65-44d4-86f9-cdd283d46393.png)


---

# 関数アプリに関連する基本的な用語
前述の App Service / Azure Functions アーキテクチャより、以下に使用される用語の説明や関数アプリ内で使用されるコンポーネントについて解説します。

## 関数アプリ、FunctionApp
関数アプリ、FunctionApp は Azure Functions 製品そのものを指しており、関数コードを実行をホストする Azure リソースを指します。HTTP トリガー や Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を行います。関数アプリでは、[Function App を作成する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal#create-a-function-app) 際に、使用するランタイムスタック（言語）やバージョンを選択します。


## ワーカープロセス、worker process
この言葉はすべて同じものを指していて、FunctionsHost や 関数コードを動作するためのプロセスとなります。Kudu と呼ばれるデバッグツールの Process Explorer や SSH より確認することができ、以下のスクリーンショットでは .NET isolated のランタイムスタックであるため、ホストプロセス と 言語ワーカープロセス が分離しております。なお、.NET in-process の場合には、ホストプロセス と 言語ワーカープロセス は同じプロセスにて動作します。


**Windows 版 のワーカープロセス**

高度なツール > Process Explorer へ移動しプロセス情報を確認します。

![image-c1efd3de-705b-4b22-831f-ba70c1eaaaf5.png]({{site.baseurl}}/media/2023/08/image-c1efd3de-705b-4b22-831f-ba70c1eaaaf5.png)


**Linux 版 のワーカープロセス**

高度なツール > SSH へ移動し `ps -ax` コマンドにてプロセス情報を確認します。

![image-cd98e905-002a-44c0-a84e-74a021d3b3c0.png]({{site.baseurl}}/media/2023/08/image-cd98e905-002a-44c0-a84e-74a021d3b3c0.png)


## ホストプロセス、FunctionsHost
ワーカープロセス の一部である ホストプロセスは、HTTP Trigger や Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を担います。 Azure Functions のランタイムとして動作するワーカープロセスへ処理を要求します。動作している ホストプロセスは、上記の Kudu スクリーンショットでは Windows: w3wp.exe、Linux: /azure-functions-host/Microsoft.Azure.WebJobs.Script.WebHost となります。


## 言語ワーカー プロセス、LanguageWorker
ワーカープロセス の一部である [言語ワーカー プロセス（LanguageWorker）](https://learn.microsoft.com/ja-jp/azure/azure-functions/supported-languages#languages-by-runtime-version) は各言語に応じた関数コードを実行するプロセスとなります。動作している ホストプロセスは、上記の Kudu スクリーンショットでは Windows: dotnet.exe、Linux: dotnet となります。なお、2023 年 7 月 現在にて Azure Functions でサポートしている主な言語ワーカーとプロセスは以下となります。

| 言語ワーカー | プロセス名 (Windows 版) | プロセス名 (Linux 版) |
|:--:|:--:|:--:|
| .NET (isolated) | dotnet.exe | dotnet |
| Node.js | node.exe | node |
| Java | java.exe | java |
| PowerShell | dotnet.exe | dotnet |
| Python | 未対応 | python |

また、関数コードを実行する 言語ワーカー プロセス数は [FUNCTIONS_WORKER_PROCESS_COUNT](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#functions_worker_process_count) のアプリケーション設定により制御する設定となりますため、上記の Kudu スクリーンショットにございますdotnet.exe のプロセス数となります。


## 関数、関数コード
実際のイベントに対する処理を実施するアプリケーションコードとなります。イベントに対する処理を実施する関数コードはトリガー（HTTP, タイマー）と呼ばれるイベントのテンプレートを設定することができ、イベントが検知されると関数コードのアプリケーションが動作します。以下は関数コードトリガーの一例として [HTTP トリガー関数の作成](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal#create-function) 方法がございます。


## インスタンス、Worker インスタンス、VM インスタンス
関数アプリを動作するための VM インスタンス（Worker インスタンス）となります。


## Plan Scale Out（プランのスケールアウト）、App Scale Out（アプリのスケールアウト）
Plan Scale Out（プランのスケールアウト）は、Elastic Premium プランで制御するインスタンス数の定義となり Elastic Premium プランでのインスタンスの最大スケールアウト(Maximum Burst)のインスタンス数と最小のインスタンス数(Minimum Instances)を設定します。 App Scale Out（アプリのスケールアウト）は、1 つの Elastic Premium プラン上で動作する関数アプリが配置されるインスタンス数の定義を実施します。
例えば、Always Ready Instancesが 1 になりますと、最低でも関数アプリが 1 インスタンスで動作することを決定しており、関数コードのイベント数の負荷により、インスタンス は Elastic Premium Plan で設定された Maximum Burst である 20 インスタンスまで増加することが出来ます。

[プランと SKU の設定](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-premium-plan?tabs=portal#plan-and-sku-settings)
> プランを作成するときは、2 つのプラン サイズ設定があります。インスタンスの最小数 (またはプラン サイズ) と最大バースト制限です。

> 常時使用可能なインスタンス数を超えるインスタンスがアプリに必要な場合、インスタンスの数が最大バースト制限に達するまでアプリのスケール アウトを続けることができます。

Elastic Premium プランのスケーリング設定の例として、以下のスクリーンショットにございます内容では、Plan Scale out の Maximum Burst は 20 となっており、App Scale out の Always Ready Instances は 1 でございます。

![image-7504a461-0a49-496a-a5e4-feab3dfef5b5.png]({{site.baseurl}}/media/2023/08/image-7504a461-0a49-496a-a5e4-feab3dfef5b5.png)

Elastic Premium Plan にて割り当て済みのインスタンス数を確認する方法としては弊社サポートチームブログより公開をしておりますためご参照ください。

[Function App の Elastic Premium Plan にて割り当て済みのインスタンス数を確認する方法](https://jpazpaas.github.io/blog/2022/08/09/how-to-check-elastic-premium-plan-function-app-allocated-instance-counts-history.html)


---
# 付録：Azure Functions のスケーリングについて
上記の App Service/Azure Functions の簡単なアーキテクチャや各用語の紹介から付録として Azure Functions のスケーリングについてご紹介をいたします。
Azure Functions では Worker インスタンスをスケーリング方法と 1 Worker インスタンス内に Azure Functions の 言語ワーカープロセスの増加方法がございます。Azure Functions でのパフォーマンスを向上するために必要な 1 つの要因となりますが、本ブログでは用語についてのご紹介となりアプリケーションのパフォーマンスチューニングについては、負荷テストを実施していただき以下の項目箇所のチューニングをご検討いただけると幸いです。

## Worker インスタンスのスケーリング（スケールアウト）
**従量課金プランまたは Elastic Premium プラン** での Azure Functions の Worker インスタンスのスケーリングは HTTP リクエスト数や Queue メッセージ数によるイベントベースとなりますため、専用プラン（App Service プラン）の CPU 使用率、メモリ使用量などのメトリクスを監視しているわけではございません。例えば、HTTP トリガーでは、 HTTP リクエストが増加すると、関数アプリへのイベントレートも増え、より多くのインスタンスが関数アプリに割り当てられる動作となっております。

[Azure Functions でのイベント ドリブン スケーリング](https://learn.microsoft.com/ja-jp/azure/azure-functions/event-driven-scaling)
> 従量課金プランと Premium プランでは、Azure Functions は、Functions ホストのインスタンスを追加することによって CPU およびメモリ リソースをスケーリングします。 インスタンスの数は、関数をトリガーするイベントの数に基づいて決定されます。

以下の図のように Worker インスタンスのスケーリング（1 台から 2 台へ増加）が行われると Worker インスタンス内にて Functions Host と Language Worker がセットで作成されます。

![function-app-scaleout-0ae2eb6c-c46d-471f-a9dd-35cdf78f8d0e.png]({{site.baseurl}}/media/2023/08/function-app-scaleout-0ae2eb6c-c46d-471f-a9dd-35cdf78f8d0e.png)

## Language Worker プロセスの増加
Worker インスタンスのスケーリング以外にも、1 つの Worker インスタンス内にて関数コードを実行するためのプロセスである 言語ワーカー数を増加する方法として、アプリケーション設定 `FUNCTIONS_WORKER_PROCESS_COUNT` より設定が実施できます。 1 Worker インスタンス上で動作する Language Workerの最大数を指定する値です。この値は最大で 10 まで設定できます。

言語ワーカー数を増加することで、処理するプロセス数が増えるため、CPU リソースを集中的に使用する関数コードが、他の関数コードの実行をブロックする可能性が低くなります。複数の言語ワーカープロセスで動作すことから、関数アプリ上の関数コードの実行が分散されることになるため、同一言語ワーカープロセス上で実行される関数コードの数が減少することが期待されます。例えば、4 つの関数コード (A,B,C,D) 実行が 2 つの言語ワーカープロセス X (AとB), Y (CとD) へ分散されてる状況で、関数コード A の処理にてタイムアウトが発生した場合、タイムアウトが発生したプロセス X のみが再起動されるため、他の関数コード (C、D)への影響を避けることができます。


[FUNCTIONS_WORKER_PROCESS_COUNT](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-app-settings#functions_worker_process_count)
> 言語ワーカー プロセスの最大数を指定します。既定値は 1 です。 許容される最大値は 10 です。 関数呼び出しは、言語ワーカー プロセス間で均等に分散されます。 言語ワーカー プロセスは、WORKER_PROCESS_COUNT によって設定されたカウントに達するまで、10 秒ごとに生成されます。 複数の言語ワーカー プロセスの使用は、スケーリングと同じではありません。

[複数のワーカー プロセスを使用する](https://learn.microsoft.com/ja-jp/azure/azure-functions/performance-reliability#use-multiple-worker-processes)
> 既定では、Functions のどのホスト インスタンスでも、単一のワーカー プロセスが使用されます。 特に Python などのシングルスレッド ランタイムによってパフォーマンスを向上させるには、FUNCTIONS_WORKER_PROCESS_COUNT を使用して、ホストあたりのワーカープロセス数を増やします (最大 10 まで)。 次に、Azure Functions は、これらのワーカー間で同時関数呼び出しを均等に分散しようとします。

以下の図のように Language Worker プロセスのスケーリング（1 から 2 へ増加）が行われると、1 Worker インスタンス内にて Functions Host としては 1 となりますが、Language Worker が 2 となります。

![function-app-process-count-7d6c0834-69ac-4e20-8f53-e11d08ce581a.png]({{site.baseurl}}/media/2023/08/function-app-process-count-7d6c0834-69ac-4e20-8f53-e11d08ce581a.png)



<br>
<br>

---

<br>
<br>

2023 年 08 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>