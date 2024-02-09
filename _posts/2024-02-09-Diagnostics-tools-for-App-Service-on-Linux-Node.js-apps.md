---
title: "App Service on Linux Node.js アプリの診断ツールの一般提供について"
author_name: "a-tmori"
tags:
    - app service
---

※本記事は Azure App Service ブログ [General availability of Diagnostics tools for App Service on Linux Node.js apps](https://azure.github.io/AppService/2024/01/05/Diagnose-Tools-for-NodeJs-Linux-apps.html)の日本語抄訳版です。

# App Service on Linux Node.js アプリの診断ツールの一般提供について

Node.js アプリケーション向けの Linux 用 Azure App Service の診断ツールが公開されました。この機能によりアプリケーションコードの問題をデバッグするのに役立つ、詳細な診断アーティファクトを収集するためのビルトインサポートが提供されます。

これには、メモリダンプやプロファイラトレースが含まれ、開発者が Linux の Node.js コードについて様々なシナリオを診断するのに役立ちます。

具体的なシナリオとしては、以下のようなものがあります：

- メモリ使用率が高い
- CPU 使用率が高い

この機能は [V8 sample-based profiler](https://v8.dev/docs/profile)を使用し、診断トレースやスナップショットを収集することで、アプリケーションコードが問題の原因となっているかを特定します。

# 「問題の診断と解決」を利用した取得方法
**問題の診断と解決** ブレードから **Diagnostic Tools** に移動します。
![image-8c4bb580-aab2-45c4-8a5b-dda097222995.png]({{site.baseurl}}/media/2024/02/image-8c4bb580-aab2-45c4-8a5b-dda097222995.png)

**Collect Node.js Heap Dump** もしくは **Collect Node.js CPU Profiler** を選択します。

こちらの機能は AlwaysOn 設定が TRUE に設定されている、かつ　SKU が Standard、Premium、もしくは Isolated の場合でのみご利用いただけます。

![image-d358c0b3-9b45-4b79-964a-34962703b04a.png]({{site.baseurl}}/media/2024/02/image-d358c0b3-9b45-4b79-964a-34962703b04a.png)

# Kudu を用いた取得方法
---
[Linux app service 用の Kudu コンソール](https://learn.microsoft.com/ja-jp/azure/app-service/resources-kudu) がアップデートされ、Process Explorer ページにメモリダンプとプロファイル用の新しいログ収集オプションが追加されました。

この新しいKuduの機能を使用するには、以下にアクセスしてください。

```https://<app 名>.scm.azurewebsites.net/newui ```

Process Explorer を選択すると、デバッグしたいプロセスを特定できます。

![image-41091d96-5b5c-4652-b211-ab106dc73097.png]({{site.baseurl}}/media/2024/02/image-41091d96-5b5c-4652-b211-ab106dc73097.png)

ドロップダウンメニューからメモリダンプの種類を選択し、**Collect Dump** をクリックします。または、ドロップダウンからプロファイルの期間を選択し、**Start Profilling** をクリックします。

![image-13a6d320-d46d-44a4-b02f-a6019deb8d54.png]({{site.baseurl}}/media/2024/02/image-13a6d320-d46d-44a4-b02f-a6019deb8d54.png)

# ヒープスナップショットを分析してメモリの問題を調査する
---
ヒープダンプは「*.heapsnapshot」という拡張子で作成されます。
ダンプが作成されると、ローカルマシンにダウンロードするためのリンクが提供されます。ダンプは、任意の Chromium ブラウザーを使用して分析することができます。

Chrome と Edge は同じ Javascript ランタイム (V8 エンジン) を使用しているため、ヒープスナップショットは Chrome または Edge DevTools for Node を使用して読み取ることができます。

**Chrome を使用する場合**：ブラウザで `chrome://inspect/` と入力し、**Open dedicated DevTools for Node** をクリックします。

![image-449d0f92-656f-4a2c-a656-9271eecac714.png]({{site.baseurl}}/media/2024/02/image-449d0f92-656f-4a2c-a656-9271eecac714.png)

**Edge を使用する場合**：ブラウザで `edge://inspect/` と入力し、同様に **Open dedicated DevTools for Node** をクリックします。
![image-a67e3eae-4296-4460-a04a-b23f3b3aa136.png]({{site.baseurl}}/media/2024/02/image-a67e3eae-4296-4460-a04a-b23f3b3aa136.png)

「Memory」タブを選択し、ここでヒープスナップショットを読み込んで分析します。

重要なのは「Shallow Size」と「Retained Size」の二つの列です。

>**Shallow Size** : オブジェクト自体が保持しているメモリのサイズです。通常、配列や文字列だけが顕著な Shallow Size を持ちます。
>
>**Retained Size** : オブジェクトが GC（ガベージコレクション）のルートから到達不能になり削除されることで解放されるメモリのサイズです。オブジェクトによって暗黙的に保持されます。

Retained Size の最も高い割合を探し、Shallow Size とも比較します。これにより、メモリリークや無駄なメモリ使用を特定できます。

<IMG  src="https://azure.github.io/AppService/media/2024/01/heapsnapshot.jpg"  alt="Heap snapshot"/>

Chromium ブラウザーで heapsnapshot ダンプを分析する方法の詳細については、[Chrome - Devtools - Heapsnapshot - Reference](https://developer.chrome.com/docs/devtools/memory-problems/heap-snapshots/) をご参照ください。

# CPU プロファイルを分析して CPU 高使用率を調査する
---

CPUプロファイラートレースは「*.cpuprofile」という拡張子で作成されます。
トレースが作成されると、ローカルマシンにダウンロードするためのリンクが提供されます。

トレースは、任意の Chromium ブラウザーを使用して分析できます。

**Chrome を使用する場合**： Chrome ブラウザーで `chrome://inspect` と入力し、**Open dedicated DevTools for Node** をクリックします。

**Edge を使用する場合**：Edgeブラウザーで `edge://inspect` と入力し、**Open dedicated DevTools for Node** をクリックします。

DevTools 内で「Performance」タブを選択します。
ここにトレースファイルをドラッグアンドドロップします。

<IMG  src="https://azure.github.io/AppService/media/2024/01/node-cpu-profile.jpg"  alt="CPU Profile"/>

<br><br>
「Call Tree」や「Bottom-Up」などのビューも使用することができます。
これらのビューを使って、プロファイラートレース内のフレームをより詳細に調べることができます。フレームを拡大して、さらに詳細な情報を得ることも可能です。

# おわりに
Azure App Serviceでは、診断機能の充実と向上に努めており、本番環境のアプリケーションの健全性を詳細に分析し、トラブルシューティングするための幅広いツールを提供しています。


<br>
<br>

---

<br>
<br>

2024 年 02 月 09 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>