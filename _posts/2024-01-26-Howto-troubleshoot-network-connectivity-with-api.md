---
title: API を用いた App Service のネットワークトラブルシューティング
author_name: "Tsutomu Mori"
tags:
    - App Service
---


# 2024-02-13 追記

本記事で紹介している API は現時点でプレビュー段階となり、あくまでもトラブルシューティング目的でご紹介させていただいております。
また、`WEBSITE_DNS_SERVER` が設定されている場合等において、想定通りに動作しない場合があることを確認しております。
Kudu 経由で、`tcpping` や `ping`、`nameresolver` や` nslookup` が利用可能な場合はまずは該当コマンドにて結果確認を実施いただけますと幸いです。本記事でご紹介する API は、Kudu にアクセスできない場合など上記コマンドを用いた問題切り分けが行えない場合の代替案として捉えていただけますと幸いです。

2024-02-13 追記 ここまで


# はじめに
本記事では、ネットワークトラブルシューティングに利用可能な新しい API の利用方法をご紹介します。

App Service における外部接続のトラブルシューティングでは、当ブログや公式ドキュメントで Kudu を利用した tcpping や nameresolver の実行方法もご案内しております。

- [仮想ネットワークとAzure App Serviceの統合のトラブルシューティング](https://learn.microsoft.com/ja-jp/troubleshoot/azure/app-service/troubleshoot-vnet-integration-apps)

- [Kudu サイトの使い方 (Tips 4 選)](/blog/2022/11/28/How-to-use-Kudu-site.html#tcp-ping-%E3%81%AE%E5%AE%9F%E8%A1%8C%E6%96%B9%E6%B3%95) 


上記の方法では Kudu UI を利用する必要があり、アプリケーションの動作環境 (OS) 毎に実行コマンドが異なる等の課題がありました。2024 年 1 月時点では
[公式ドキュメント](https://learn.microsoft.com/ja-jp/rest/api/appservice/)に未記載でございますが、
OS 毎の違いを抽象化した、より便利な API がリリースされています。


# APIを利用した疎通確認方法

Resource Manager API として、TCP 接続確認用の `tcpPingCheck` と、DNS 名前解決確認用の `dnsCheck` が新たに提供されています。
呼び出し方法として任意の HTTP クライアントを用いることも可能ですが、ここでは `az rest` コマンドを利用した呼び出し方法をご紹介します。
 
Azure CLI から以下の記法で resource URI とアクセス先 FQDN （完全修飾ドメイン名）を指定し、コマンドを実行することで疎通確認が行えます。

## 1.tcpPingCheck 

指定された host:port に対して TCP 接続を試みた結果を出力します。

```bash
az rest --method POST --url "/subscriptions/<サブスクリプションid>/resourcegroups/<リソースグループ名>/providers/microsoft.web/sites/<サイト名>/tcpPingCheck?api-version=2022-03-01" --body "{'properties': {'host': <アクセス先ホスト>, 'port': <アクセス先ポート>}}"
```

## 2.dnsCheck

指定された hostname の DNS 名前解決を試みた結果を出力します。

```
az rest --method POST --url "/subscriptions/<サブスクリプションid>/resourcegroups/<リソースグループ名>/providers/microsoft.web/sites/<サイト名>/dnsCheck?api-version=2022-03-01" --body "{'properties': {'dnsName': 'アクセス先ホスト'}}"
```
	

# 検証

## tcpPingCheck を実行

PowerShell を使用する場合、 `az login` でログインします。

![image-7acb6066-3d54-4867-9a8a-15711519d498.png]({{site.baseurl}}/media/2024/01/image-7acb6066-3d54-4867-9a8a-15711519d498.png)

`tcpPing` を行う API エンドポイントを指定します。<br>
`body` の `properties` では接続するホスト名とポート番号を指定します。

![image-ed07ac37-3c17-47e4-96ba-9dd22a201dda.png]({{site.baseurl}}/media/2024/01/image-ed07ac37-3c17-47e4-96ba-9dd22a201dda.png)

**成功例**

`TCP Ping` が成功し、外部サイトへの接続結果が表示されます。<br>

出力結果の `properties.connectionStatus` から接続が `Success` であったこと、`connectionStatusDetails` から、`externalservicepingtest.azurewebsites.net:443` に対して4回の接続施行がそれぞれ 48ms、10ms、10ms、10ms で完了したことがわかります。

![image-c3fd2f19-d843-441e-8527-c78214a2f75f.png]({{site.baseurl}}/media/2024/01/image-c3fd2f19-d843-441e-8527-c78214a2f75f.png)

**失敗例**

開いていないポート (22番) にアクセスすると、`Gateway Timeout` となっていることがわかります。

![image-f687ccff-5418-4a7d-9c67-8f43ceb0f74c.png]({{site.baseurl}}/media/2024/01/image-f687ccff-5418-4a7d-9c67-8f43ceb0f74c.png)


## dnsCheck を実行

`dnsCheck` を行う API エンドポイントを指定します。<br>
`body` の `properties` では接続するホストを指定します。<br>

![image-b7f292f6-84e5-4974-b57c-c22fba9fb9a8.png]({{site.baseurl}}/media/2024/01/image-b7f292f6-84e5-4974-b57c-c22fba9fb9a8.png)

**成功例**

`properties.connectionsStatus` から接続が `Success` であったことが分かります。
また、DNS 解決の結果として 実際のホスト名 `blob.tyo22prdstr07a.store.core.windows.net` と対応する IP アドレスが出力されています。<br>

![image-ea8574a4-f3aa-4229-accf-0b285629c843.png]({{site.baseurl}}/media/2024/01/image-ea8574a4-f3aa-4229-accf-0b285629c843.png)


**失敗例**

 アクセス先に存在しない DNS 名を指定すると `properties.connectionStatus` は `UnknownError` となり、名前解決に失敗したことが分かります。<br>

![image-d71e5b1b-c08d-4374-8d18-ed4e79120c33.png]({{site.baseurl}}/media/2024/01/image-d71e5b1b-c08d-4374-8d18-ed4e79120c33.png)

# API プレイグラウンド（プレビュー）

Azure Portal 上から [APIプレイグラウンド（プレビュー）](https://portal.azure.com/?feature.customportal=false#view/Microsoft_Azure_Resources/ArmPlayground)
にアクセスすると、同様の確認を行うことができます。

## 1.tcpPing 

メソッドとして `POST` を指定し、API バージョンを含む ARM 相対パスを入力します。
要求本文で host と port を指定して実行を押すと、応答を確認することができます。

![image-a5b72ad7-eeb9-417c-b53e-882ee912414c.png]({{site.baseurl}}/media/2024/01/image-a5b72ad7-eeb9-417c-b53e-882ee912414c.png)

## 2.dnsCheck

`dnsCheck` の場合は ARM 相対パスの "tcpPingCheck" の部分を "dnsCheck" に変更し、要求本文も "dnsName" : "ドメイン名（FQDN)" を設定します。

![image-a8408634-ffa2-428c-80f2-b4b07fa90ba4.png]({{site.baseurl}}/media/2024/01/image-a8408634-ffa2-428c-80f2-b4b07fa90ba4.png)

<br>
<br>

---

2024 年 01 月 26 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。
