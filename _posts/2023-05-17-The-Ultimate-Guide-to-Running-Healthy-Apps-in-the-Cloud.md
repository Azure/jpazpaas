---
title: "[日本語訳] The Ultimate Guide to Running Healthy Apps in the Cloud"
author_name: "Masafumi Kokui"
tags:
    - App Service
    - Function App
    - Web Apps
    - App Service Environment
---

# はじめに
***
本記事は 2020 年 5 月に投稿されました
[The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html) の日本語抄訳です。2022 年 5 月 12 日時点の内容に合わせて一部加筆修正を行っています。

[[_TOC_]]

現代のデータセンターは非常に複雑で、多くの可動部分があります。VM の再起動や移動、システムのアップグレード、ファイルサーバーのスケールアップやスケールダウン。これらの事象はクラウド環境では予想されることです。しかしながら、ベストプラクティスに従うことで、クラウドアプリケーションをこれらの事象に対応することができます。本稿では、アプリケーションをクラウドに対応させるために取るべき 13 の重要なステップを紹介します。これらのステップを踏むことで、データセンターで発生した事象がアプリケーションに与える影響を少なくし、アプリケーションがより弾力的で将来性のあるものにすることができます。

前述のように、VM インスタンスは再起動することが予想されますし、実際に行われます。また、VM インスタンスはアップグレードされますし、しばしばファイルサーバーの移動に悩まされることもあります。しかしながら、アプリケーションをこのような事態に備えることは可能です。アプリケーションの可用性を最大限にするために、多くのプラクティスを取り入れることをお勧めします。


**Learn More**
- [App Service を構成する主要な要素とそれぞれの役割](https://jpazpaas.github.io/blog/2023/05/10/Appservice-components.html)
- [[日本語訳] Inside the Azure App Service Architecture](https://jpazpaas.github.io/blog/2023/05/10/Inside-the-Azure-App-Service-Architecture.html)
- [Azure App Service の計画メンテナンスの事前通知](https://jpazpaas.github.io/blog/2023/02/16/Routine-Planned-Maintenance-Notifications-for-Azure-App-Service.html)

<br>

# 複数のインスタンスを使用する
***
アプリケーションを 1 つのインスタンスで動作させると、単一障害点となります。アプリケーションに複数のインスタンスを割り当てることで、特定の インスタンスに何か問題が発生した場合であっても、アプリケーションは他のインスタンスからリクエストに応答することができるようになります。**アプリケーションコードは、データソースからの読み取りやデータソースへの書き込みの際に、同期の問題なしに複数のインスタンスで動作させる必要があることに留意ください。** Azure ポータルの設定セクションの「スケール アウト（App Service のプラン）」より、アプリケーションに複数のインスタンスを割り当てることができます。

![image-8115a83b-6e10-4351-9622-a24144e3f210.png]({{site.baseurl}}/media/2023/05/image-8115a83b-6e10-4351-9622-a24144e3f210.png)

単一障害点を避けるために、**少なくとも2-3個のインスタンス**でアプリケーションを実行することをご検討ください。これは、アプリケーションの起動に時間がかかる場合（コールドスタートと呼ばれます）には特に重要です。複数のインスタンスを実行することで、App Service の基盤となります インスタンスが移動、またはアップグレードした場合でもアプリケーションが利用できるようになります。また、以下のような事前定義されたルールに基づいて、自動的にスケールアウトするルールを構成することもできます。

- 時間帯（アプリケーションのトラフィックが多い時間帯など）
- リソースの使用状況（メモリ、CPUなど）
- 両方の組み合わせ

**Learn More**
- [Azure での自動スケールの使用](https://docs.microsoft.com/azure/azure-monitor/platform/autoscale-get-started?toc=/azure/app-service/toc.json)
- [App Service Warm-Up Demystified](https://michaelcandido.com/app-service-warm-up-demystified/)

<br>

# 初期設定の更新
***
App Service には、開発者が Web アプリケーションをユースケースに合わせて構成するための多くの設定があります。**常時接続（Always-On）** は、過去 20 分間にリクエストが受信されなかった場合でも、インスタンスを存続させます。常時接続を有効にすることで、アプリケーションのコールドスタートを緩和することができます。

**ARR アフィニティ** は、スティッキーセッションを作成し、クライアントがその後のリクエストで同じアプリケーションインスタンスに接続するようにします。ただし、ARR Affinity を使用すると、インスタンス間で要求が不均等に分散され、インスタンスに過負荷がかかる可能性があります。ARR アフィニティを無効にすることは、アプリケーションがステートレスであるか、セッションの状態がキャッシュやデータベースなどのリモートサービスに保存されていることを想定しています。

最後に、**プラットフォーム**を 32 ビットから 64 ビットに設定してください。32ビットプロセス（64ビットオペレーティングシステム上でも）で利用可能なメモリ量は最大 2GB です。デフォルトでは、App Service のワーカープロセスは 32 ビットに設定されています（レガシーなWebアプリケーションとの互換性のためです）。利用できる追加のメモリを活用できるように、64 ビットプロセスへの切り替えを検討してください。

堅牢性を重視する本番アプリケーションでは、**［常時接続］を［オン］、［ARR アフィニティ］を［オフ］、［プラットフォーム］を［64 Bit］** に設定することをお勧めします。

これらの設定は、Azure ポータルの設定セクションの「構成」->「全般設定」タブで変更することができます。

![image-e325a3f5-5ae2-45fc-a8f9-7d00206de5a4.png]({{site.baseurl}}/media/2023/05/image-e325a3f5-5ae2-45fc-a8f9-7d00206de5a4.png)

**Learn More**
- [App Service アプリを構成する](https://docs.microsoft.com/azure/app-service/configure-common#configure-general-settings)
- [Disable Session affinity cookie (ARR cookie) for Azure web apps](https://azure.github.io/AppService/2016/05/16/Disable-Session-affinity-cookie-(ARR-cookie)-for-Azure-web-apps.html)

<br>

# Production Hardware を使用する
***
App Service は、お客様のさまざまなニーズに合わせて、さまざまなハードウェア階層（SKU とも呼ばれます）を提供しています。新しい App Service プランを作成する際に SKU を選択するオプションがあります。

![image-933c43be-a30e-46db-b6c6-8523adfa1800.png]({{site.baseurl}}/media/2023/05/image-933c43be-a30e-46db-b6c6-8523adfa1800.png)

App Service プランを運用環境で使用する場合、App Service プランが推奨する「Production」価格帯のいずれかを指定ください。さらに、お客様のアプリケーションが大量にリソースを消費する場合には、お客様のアプリケーションの必要性に応じて、推奨される価格帯の中から適切な価格帯を選択するようにしてください。例えば、アプリケーションが多くの CPU サイクルを消費する場合、S1 の価格帯で実行すると、CPU が高くなり、アプリケーションのダウンタイムや遅延を引き起こす可能性があります。

**Learn More**
- [Azure App Service でアプリをスケールアップする](https://docs.microsoft.com/azure/app-service/manage-scale-up)

<br>

# デプロイメントスロットを活用する
***
新しいプログラムコードを運用環境にデプロイする前に、App Services のデプロイメントスロットを利用してテストすることができます。デプロイメントスロットは、独自のホスト名を持つアプリケーションです。アプリケーションのコンテンツと構成要素は、運用スロットを含む2つのデプロイメントスロット間でスワップすることができます。

アプリケーションを非運用スロットにデプロイすると、次のような利点があります。
- 運用スロットにスワップする前に、ステージング環境でアプリケーションの変更を検証することができます。
- アプリケーションを最初にスロットにデプロイして運用環境にスワップすると、運用環境にスワップする前にステージングスロットのすべてのインスタンスがウォームアップされます。これにより、アプリケーションをデプロイする際のダウンタイムをなくすことができます。トラフィックのリダイレクトはシームレスで、スワップ操作のためにリクエストがドロップされることもありません。自動スワップを設定することで、このワークフロー全体を自動化することができます。
- スワップ後のステージングスロットには以前の運用スロットのアプリケーションが動作しています。運用スロットにスワップされた変更が期待通りでない場合は、すぐに同じスワップを実行して「最後に確認された正常なサイト」に戻すことができます。

![image-65507357-2d5a-4980-8313-233dfd023e64.png]({{site.baseurl}}/media/2023/05/image-65507357-2d5a-4980-8313-233dfd023e64.png)

_デプロイメントスロットは、Standard、Premium、または Isolated App Service のプランでのみ利用可能です。_

**プレビューでのスワップ (複数フェーズのスワップ)** の使用を強くお勧めします。プレビューでのスワップを使用すると、ステージングスロットのアプリケーションを運用スロットの設定と比較してテストし、またアプリケーションをウォームアップすることができます。テストを行い、必要なパスをすべてウォームアップした後、スワップを完了することで**アプリケーションは再起動することなく**運用環境のトラフィックを受信するようになります。これは、アプリケーションの可用性とパフォーマンスに大きな影響を与えます。

**Learn More**
- [Azure App Service でステージング環境を設定する](https://docs.microsoft.com/azure/app-service/deploy-staging-slots)
- [プレビューでのスワップ (複数フェーズのスワップ)](https://learn.microsoft.com/ja-jp/azure/app-service/deploy-staging-slots#swap-with-preview-multi-phase-swap)
- [デプロイ スロットの使用](https://docs.microsoft.com/azure/app-service/deploy-best-practices#use-deployment-slots)

<br>

# 正常性チェックのパスを設定する
***
App Service では、アプリケーションのヘルスチェックパスを指定することができます。プラットフォームはこのパスに ping を送信して、アプリケーションが正常にリクエストに応答しているかどうかを判断します。サイトが複数のインスタンスにスケールアウトされている場合には App Service は異常なインスタンスを切り離し、全体の可用性を向上させます。アプリケーションのヘルスチェックパスは、データベース、キャッシュ、メッセージングサービスなど、アプリケーションの重要なコンポーネントをポーリングする必要があります。これにより、ヘルスチェックパスから返されるステータスが、アプリケーションの全体的な正常性を正確に表していることが保証されます。

1. Azure ポータルの監視セクションの「正常性チェック」をクリックします。

![image-84c07222-1a13-493c-9181-bdf7ca098cd6.png]({{site.baseurl}}/media/2023/05/image-84c07222-1a13-493c-9181-bdf7ca098cd6.png)

2. 当機能が ping を送信するパスを設定します。
3. 「保存」をクリックして設定を保存します。

_**正常性チェックにより異常なインスタンスが切り離されるのは 2 つ以上のインスタンスがある場合に限られます。** シングルインスタンスの場合、異常な ping が 1 時間継続するとインスタンスの置換が行われます。_

**Learn More**
- [正常性チェックを使用して App Service インスタンスを監視する](https://learn.microsoft.com/azure/app-service/monitor-instances-health-check?tabs=dotnet)
- [アプリが 1 つのインスタンスで実行されている場合は、どうなるでしょうか?](https://learn.microsoft.com/azure/app-service/monitor-instances-health-check?tabs=dotnet#what-happens-if-my-app-is-running-on-a-single-instance)

<br>

# Application Initialization を利用する
***
Application Initialization は、アプリケーションのインスタンスがリクエストを受け付ける前にアプリケーションが完全に開始されることを保証します。Application Initialization は、サイトの再起動、自動スケーリング、および手動スケーリング時に使用されます。これは、サイトのルートパス('/') がリクエストを受け付けるだけではアプリケーションが起動しない場合に重要な機能です。このために、認証を必要としないウォームアップパスをアプリケーション上に作成し、Application Initialization がこの URL パスを使用するように構成する必要があります。

ウォームアップ用 URL で実装されたメソッドが、すべての重要な機能に触れるようにし、ウォームアップが完了したときだけレスポンスを返すようにします。レスポンス（成功または失敗）を返したときにのみ、Application Initialization は「アプリケーションは問題なく起動した」と判断します。Application Initialization は、web.config ファイル内で設定することができます。

**Learn More**
- [カスタム ウォームアップを指定する](https://learn.microsoft.com/azure/app-service/deploy-staging-slots#specify-custom-warm-up)

<br>

# ローカルキャッシュの有効化
***
この機能を有効にすることで、サイトは（サイトコンテンツが保存されている）Azure ストレージから取得するのではなく、ローカルの仮想マシンインスタンスからファイルを読み書きするようになります。これにより、アプリケーションの リサイクルを減らすことができます。これは、Azure ポータルの設定セクションの「構成」-> 「アプリケーション設定」から有効にすることができます。「アプリケーション設定」で、キーに「WEBSITE_LOCAL_CACHE_OPTION」を、値を「Always」として追加します。また、「WEBSITE_LOCAL_CACHE_SIZEINMB」を追加し、最大 2000MB のローカルキャッシュサイズを指定します（指定しない場合のデフォルトは 300MB になります）。サイトのコンテンツが 300MB を超える場合にはキャッシュサイズを指定する必要があります。この機能を使用するには、サイトのコンテンツが 2000MB 未満であることを確認してください。また、スワップ時に削除されないように、スロットにも設定ください。**ここで最も重要なことは、アプリケーションがデータやトランザクションの状態を維持するためにローカルディスクへの書き込みを行うべきでないということです。** ストレージはストレージコンテナや DB、Cosmos DB のような外部ストレージを使用する必要があります。

_ローカルキャッシュの動作は、使用している言語と CMS に依存することに注意してください。最良の結果を得るためには、アプリケーションがローカルディスクに書き込みを行っていない限り、.NET および .NET Core アプリケーションで使用することをお勧めします。_

![image-d56cc4c9-88bf-4c15-b77e-703499cb83c6.png]({{site.baseurl}}/media/2023/05/image-d56cc4c9-88bf-4c15-b77e-703499cb83c6.png)

**Learn More**
- [Azure App Service のローカル キャッシュの概要](https://docs.microsoft.com/azure/app-service/overview-local-cache)

<br>

# Auto-Heal
***
アプリケーションに予期せぬ動作が発生し、単純な再起動で解決することがあります。Auto-Heal 機能は、まさにそれを可能にします。Auto-Heal のトリガーとなる「条件」と、条件が満たされたときに Auto-Heal が開始する「アクション」を定義することができます。

Auto-Heal のルールは、Azure ポータルの「問題の診断と解決」-> 「Diagnostic Tools」タイルを開き、「Proactive tools」セクションの「Auto-Heal」より定義することができます。

![image-68834c23-54da-4fcb-a13d-0d144ccabb69.png]({{site.baseurl}}/media/2023/05/image-68834c23-54da-4fcb-a13d-0d144ccabb69.png)

下記はフィルターの設定例ですが、他のエラーコードや頻度が適している場合は、適宜修正してください。

| Condition | Value |
| ---- | ---- |
| Request Count | 70 |
| Status Code | 500 |
| Sub-status code | 0 |
| Win32-status code | 0 |
| Frequency in seconds | 60 |

上記の条件を満たした上で、アクションを設定することをお勧めします。

- Recycle Process

を追加し、「Override when Action executes (Optional)」を追加します。

- Auto-Heal が実行されるまでのプロセスのスタートアップ時間： 3600秒(1時間)

**Learn More**
- [自動復旧](https://learn.microsoft.com/ja-jp/azure/app-service/overview-diagnostics#auto-healing)
- [Azure App Service Auto-Heal](https://stack247.wordpress.com/2019/05/20/azure-app-service-auto-healing/)
- [Announcing the New Auto-Heal Experience in App Service Diagnostics](https://stack247.wordpress.com/2019/05/20/azure-app-service-auto-healing/)

<br>

# App Service プランの密度を最小にする
***
App Service プランで多くのアプリケーションを実行すると、パフォーマンスに影響が出ることがあります。App Service プランで実行されているすべてのアプリは、Azure ポータルの App Service プランの「設定」セクションの「アプリ」で確認できます。

App Service プランの密度は、App Service Plan Density Check で確認することができます。詳しくは [Risk Assesments の活用](#Risk-Assesments-の活用) をご確認ください。Risk Assesments の「可用性とパフォーマンスのベスト プラクティス」もしくは「Get Resiliency Score report」よりご確認いただくことができます。

- [App Service Plan Density Check](https://azure.github.io/AppService/2019/05/21/App-Service-Plan-Density-Check.html)


<br>

# ディスク容量の監視
***
デプロイするアプリケーションコンテンツが 1GB 未満であることを確認します。これは、アプリケーションの再起動時のダウンタイムを削減し、アプリケーションのパフォーマンスを向上させるための必要です。ファイルシステムの使用量は、Azure ポータルの「App Service プラン」の「設定」セクションの「ファイル システム ストレージ」より確認することができます。

![image-3b45777c-da36-49e3-a259-840921559809.png]({{site.baseurl}}/media/2023/05/image-3b45777c-da36-49e3-a259-840921559809.png)

<br>

# Application Insights の有効化
***
Application Insights は、アプリケーションで発生したインシデントのトラブルシューティングを支援する機能群を提供します。コードエラーのデバッグ、依存関係に起因するパフォーマンス低下の診断などに使用できます。

Application Insights の強力な機能の 1 つに、Application Insights Profiler があります。Application Insights Profiler を有効にすると、Azure で本番稼働しているアプリケーションのパフォーマンストレースを確認できます。Profiler は、ユーザーに悪影響を与えることなく、スケールで自動的にデータを取得します。Profiler を使用すると、リクエストを処理する際に最も時間がかかる「ホット」なコードパスを特定することができます。Profiler は、.NET アプリケーションで動作します。これを有効にするには、Azure ポータルの「Application Insights」にアクセスします。「調査」 セクションの「パフォーマンス」をクリックします。

1. 「パフォーマンス」ペインで「プロファイラー」をクリックします。

 ![image-6411c722-d39b-4e60-bad3-41b691b6e5dc.png]({{site.baseurl}}/media/2023/05/image-6411c722-d39b-4e60-bad3-41b691b6e5dc.png)

2. 「プロファイラ」ペインで、「今すぐプロファイル」をクリックすると、Profiler が開始されます。
 
 ![image-fd9d3056-1a0c-4b81-8f75-05210bd78b42.png]({{site.baseurl}}/media/2023/05/image-fd9d3056-1a0c-4b81-8f75-05210bd78b42.png)

3. Profiler が実行されている場合、1 時間に 1 回程度、2 分間、ランダムにプロファイルが作成されます。アプリケーションが安定してリクエストを処理している場合、Profiler は1時間ごとにトレースをアップロードします。トレースを表示するには、「パフォーマンス」ペインで「詳細の表示...」->「Profiler Traces」ボタンをクリックします。
 
 ![image-3e9c52bf-48a8-415a-82e7-53bfe1fd1e51.png]({{site.baseurl}}/media/2023/05/image-3e9c52bf-48a8-415a-82e7-53bfe1fd1e51.png)

4. Application Insights では、アプリケーションの依存関係を追跡することも可能です。この機能を活用して、遅いリクエストのトラブルシューティングを行うことができます。.NET コンソールアプリから依存関係を自動的に追跡するには、Nuget パッケージ Microsoft.ApplicationInsights.DependencyCollector をインストールし、次のように DependencyTrackingTelemetryModule を初期化します。

```
 DependencyTrackingTelemetryModule depModule = new DependencyTrackingTelemetryModule();
 depModule.Initialize(TelemetryConfiguration.Active);
```

5. 各リクエストイベントは、アプリケーションがリクエストを処理している間に追跡される依存関係の呼び出し、例外、およびその他のイベントと関連付けられています。そのため、一部のリクエストの調子が悪い場合、依存関係からのレスポンスが遅いことが原因かどうかを突き止めることができます。「パフォーマンス」ペインでも、「依存関係」タブからリクエストのウォーターフォールビューを見ることができます。
 
 ![image-59336e52-9bd3-4842-80a9-a9e4855e9860.png]({{site.baseurl}}/media/2023/05/image-59336e52-9bd3-4842-80a9-a9e4855e9860.png)

また、 [Application Insights integration with App Service Diagnostics](https://azure.github.io/AppService/2020/04/21/Announcing-Application-Insights-Integration-with-App-Service-Diagnostics.html) を活用することもできます。

**Learn More**
- [Application Insights Profiler を使用した Azure のプロファイル運用アプリケーション](https://docs.microsoft.com/azure/azure-monitor/app/profiler-overview)
- [Application Insights を使用した Web アプリの例外の診断](https://docs.microsoft.com/azure/azure-monitor/app/asp-net-exceptions)
- [リクエストから依存関係までのトレース](https://docs.microsoft.com/azure/azure-monitor/app/asp-net-dependencies#diagnosis)

<br>

# 複数のデータセンターに展開する
***
Azure Front Door や Azure Traffic Manager を導入することで、トラフィックがサイトに到達する前に中継させることができます。これらは、インスタンス/リージョン間のトラフィックのルーティングと分散を支援します。Azure リージョンの 1 つで壊滅的な災害が発生した場合でも、別のリージョンに投資することで、アプリケーションの実行とリクエストへの対応を保証することができます。Azure Front Door や Azure Traffic Manager を使用することで、クライアントの地域に基づいて受信リクエストをルーティングし、クライアントに最短の応答時間を提供したり、インスタンス間で負荷を分散して 1 つのインスタンスにリクエストが集中しないようにするなどの利点もあります。<br>
また、2021 年に App Service は可用性ゾーンに対応しました。これにより App Service プランを指定したリージョン内の 3 つのゾーン（Azure データセンター）に分散配置することができるようなりました。いずれかの方法で App Service プランを複数のデータセンターに配置されることをお勧めします。

**Learn More**
- [Azure Traffic Manager による Azure App Service トラフィックの制御](https://docs.microsoft.com/azure/app-service/web-sites-traffic-manager)
- [クイック スタート:グローバル Web アプリケーションの高可用性を実現するフロント ドアを作成する](https://docs.microsoft.com/azure/frontdoor/quickstart-create-front-door)
- [可用性の高いゾーン冗長 Web アプリケーション](https://learn.microsoft.com/ja-jp/azure/architecture/reference-architectures/app-service-web-app/zone-redundant)
- [App Service を可用性ゾーンのサポートありに移行する](https://learn.microsoft.com/azure/reliability/migrate-app-service)
- [Azure リージョンと可用性ゾーンとは](https://learn.microsoft.com/ja-jp/azure/reliability/availability-zones-overview)


<br>

# Risk Assesments の活用
***
最後に、Azure ポータルの「問題の診断と解決」の「Risk Assesments」を活用することで、アプリケーションの耐障害性向上の対応状況を確認することができます。

 ![image-7d34c073-b964-4935-a14c-ff896e5e0894.png]({{site.baseurl}}/media/2023/05/image-7d34c073-b964-4935-a14c-ff896e5e0894.png)

2つのオプションが表示されます。

- 可用性とパフォーマンスのベスト プラクティス
- 最適な構成のためのベスト プラクティス

これらの検出器に記載されているすべてのベスト プラクティスを確認することをお勧めします。<br>
また、「Get Resiliency Score report」より 2 つのオプションを含めた耐障害性向上レポートを取得することもできます。

![image-0f36a465-2328-4751-b425-4c2c4918a5c2.png]({{site.baseurl}}/media/2023/05/image-0f36a465-2328-4751-b425-4c2c4918a5c2.png)

最後に、アプリケーションの開始時間を最小化し、より弾力性を高めるために [クラウド設計パターン](https://docs.microsoft.com/azure/architecture/patterns/) を参照することもお勧めします。

<br>
<br>

---

<br>
<br>

2023 年 05 月 17 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>