---
title: "Network Troubleshooter ツール を使用したネットワーク接続のトラブルシュートについて"
author_name: "Tsutomu Mori"
tags:
    - App Service
---




<br>
<br>

#はじめに

お世話になっております。App Service サポート担当の森です。

今回は、Azure Portal で使用できる診断ツール "Network Troubleshooter" についてご紹介します。

本ツールでは外部エンドポイントへの接続、DNS 解決、ネットワークセキュリティグループの構成などの問題について、問題の特定と推奨の解決策を見つけることができます。

App Service における外部接続のトラブルシューティングについては、当ブログや公式ドキュメントで詳しく説明しています。 Kudu を利用した tcpping や nameresolver の実行方法、API を利用した疎通確認方法などについてはこちらをご覧ください。

[仮想ネットワークとAzure App Serviceの統合のトラブルシューティング](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps)

[Kudu サイトの使い方 (Tips 4 選)](https://azure.github.io/jpazpaas/2022/11/28/How-to-use-Kudu-site.html#gsc.tab=0)

[API を用いた App Service のネットワークトラブルシューティング
](https://azure.github.io/jpazpaas/2024/01/26/Howto-troubleshoot-network-connectivity-with-api.html#gsc.tab=0)

#Network Troubleshooter の利用方法

本ツールを利用するには、App Service の [問題の診断と解決] ブレードから検索バーを使用して検索するか、[Troubleshooting categories] セクションから [Diagnostic Tools] ページへ移動し、Network/Connectivity Troubleshooter へのクイックリンクを選択します。


![image-0ac27df0-3ffb-4287-ad8e-8870d9c565f6.png]({{site.baseurl}}/media/2024/02/image-0ac27df0-3ffb-4287-ad8e-8870d9c565f6.png)

Network Troubleshooter は入力に基づいた解決策を提供するため、問題を段階的に理解することができます。

まずドロップダウンメニューから診断したい問題について選択します。

![image-ea8b26f9-a209-40b7-9816-3eceb8639a9d.png]({{site.baseurl}}/media/2024/02/image-ea8b26f9-a209-40b7-9816-3eceb8639a9d.png)

#トラブルシュート

## 1. Connection issues

---


このメニューでは以下の問題について確認することができます。

- App Service worker と仮想ネットワークの接続の状態

- DNS 構成の健康状態

- 特定のエンドポイントへの接続テスト

問題が見つかった場合は、見つかった問題と推奨される手順が表示されます。すべてが正常に見える場合、または警告インサイトのみが検出された場合、フローは引き続き接続をテストするエンドポイントを要求します。ホスト名とポート、または IPアドレスとポートの組み合わせを使用して接続をテストできます。




https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps#network-troubleshooter

![image-1c0271d4-c713-4da1-bad8-0e3469c785fc.png]({{site.baseurl}}/media/2024/02/image-1c0271d4-c713-4da1-bad8-0e3469c785fc.png)

**注意**

本ツールの Connection issues (接続の問題) については、Linux または コンテナーベースの App Service にはまだ対応しておりません。

![image-5a72ea75-0ef0-4856-be6e-fad04ed324bb.png]({{site.baseurl}}/media/2024/02/image-5a72ea75-0ef0-4856-be6e-fad04ed324bb.png)

[仮想ネットワークと Azure App Service の統合のトラブルシューティング](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps#network-troubleshooter)

### 特定のエンドポイントへの接続テスト
---
App Service から特定のエンドポイントへの接続を確認するには、以下のいずれかの形式でエンドポイントを指定します。

- <接続先 URL>:<ポート番号> 
- <接続先 IP アドレス>:<ポート番号>

URL とポート番号で確認を行った場合、 Ip analysis の欄で、DNS 解決後の IP アドレスを確認することができます。
また、プライベート接続、VNET 経由のルーティングの有無も併せて表示されます。

下記画像は VNET 内にプライベートエンドポイントを持つ App Service のプライベート IP に対して接続確認をしている例です。

TCP ping に成功すると以下のように出力されます。ただし、これは TCP レベルでの接続確認です。

エンドポイントサーバーエラーなどにより 4xx エラーが出るような状況でもこのテストに成功することがある点にご注意ください。

![image-3e54f408-d00a-442c-bbb9-d5758db092ed.png]({{site.baseurl}}/media/2024/02/image-3e54f408-d00a-442c-bbb9-d5758db092ed.png)

### 接続に失敗する場合
---
接続テストに失敗した場合、結果とともに推奨の手順が表示されます。



**接続失敗例 1**：送信側サブネットの NSG で送信トラフィックが制限されている場合

仮想ネットワーク統合しているサブネットの NSG で送信トラフィックをすべて拒否するよう設定すると、任意のエンドポイントへの接続確認が失敗しました。


![image-fd2e2e99-d636-4248-9456-5406ddfbba67.png]({{site.baseurl}}/media/2024/02/image-fd2e2e99-d636-4248-9456-5406ddfbba67.png)

**接続失敗例 2**： VNET を経由させず外部リソースに接続確認しようとした場合

App Service が 仮想ネットワーク統合していない場合や、仮想ネットワーク統合しているが Route All が無効になっている場合、インターネットへのトラフィックは VNet を経由しないため下記のエラーメッセージが表示されます。

![image-228fd359-03f9-435f-a750-05c17c06f09d.png]({{site.baseurl}}/media/2024/02/image-228fd359-03f9-435f-a750-05c17c06f09d.png)


ツールを使用した接続確認が成功しない場合、問題点や推奨の手順が表示されることがあります。
上記の場合、Route All 設定を有効にすると接続が成功する可能性があります。

詳細についてはこちらをご参照ください。

[アプリを Azure 仮想ネットワークと統合する](https://learn.microsoft.com/ja-jp/azure/app-service/overview-vnet-integration#routing-app-settings)

## Route All 設定を有効にする方法

Azure Portal から Route All の設定を有効にしたい場合、 [ネットワークブレード] - [仮想ネットワーク統合] ページから送信インターネットトラフィックをチェックします。これにより送信インターネットトラフィックが VNET の該当サブネット経由で送信されます。


![image-1dc493bd-813f-46d5-b33b-1b487a18cea8.png]({{site.baseurl}}/media/2024/02/image-1dc493bd-813f-46d5-b33b-1b487a18cea8.png)

## 2. Configuration issues
---

サブスクリプション、VNET とそのサブネットを選択することで、App Service が指定のサブネットと仮想ネットワーク統合しているか確認することができます。統合できていない場合、下図のようなエラーメッセージが表示されます。


![image-5e10bb3c-07c4-4491-85cd-3454eb07435e.png]({{site.baseurl}}/media/2024/02/image-5e10bb3c-07c4-4491-85cd-3454eb07435e.png)



## 3.  Learn more about VNet integration
---

ここでは App Service と仮想ネットワークの統合や Azure VNET についての基本的なコンセプトについて参照できるドキュメントへのリンクが提供されています。

![image-0e4adc10-6363-4724-be3f-4fb9d366a0dc.png]({{site.baseurl}}/media/2024/02/image-0e4adc10-6363-4724-be3f-4fb9d366a0dc.png)

# 参考ドキュメント
https://azure.github.io/AppService/2021/04/13/Network-and-Connectivity-Troubleshooting-Tool.html

<br>
<br>

2024 年 02 月 29 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>