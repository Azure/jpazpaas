---
title: "Azure Container Apps での DNS 接続のトラブルシューティング"
author_name: "Hidenori Yatsu"
tags: 
    - Container Apps
---

※本記事は Azure OSS Developer Support ブログ
 [Troubleshooting DNS connectivity on Azure Container Apps](https://azureossd.github.io/2023/03/03/azure-container-apps-testing-DNS-connectivity/index.html) の日本語抄訳版です。
# はじめに
Container Apps サポート担当の谷津です。Azure Container Apps は、コンテナー化されたアプリケーションをクラウドにデプロイするための便利で効率的な方法を提供し、アプリケーションを簡単にすばやくスケーリングして管理できるようにします。ただし、他のテクノロジと同様に、アプリケーションの可用性とパフォーマンスに影響を与える可能性のある接続の問題が発生することは珍しくありません。

この投稿では、ネットワークDNS接続の問題を特定するためのツールのインストールと実行に焦点を当てます。

# 一般的な問題 - Name or service not known
DNSの設定ミスがある場合に発生する一般的なアプリケーション例外は、**Name or service not known** です。この問題のトラブルシューティングを行うときは、まず DNS レコードを確認して、リモート サーバーの適切なエントリが追加され、正しい場所を指していることを確認する必要があります。

VNET で既定の Azure 提供の DNS サーバーではなくカスタム DNS サーバーを使用する場合は、未解決の DNS クエリを 168.63.129.16 に転送するように DNS サーバーを構成します。

Container Apps と DNS の詳細については、[こちら](https://learn.microsoft.com/ja-jp/azure/container-apps/networking?tabs=azure-cli#dns) を参照してください。

# コンテナー コンソールに接続する
コンテナーのコンソールへの接続は、接続テストを実行する必要がある場合に便利です。

ポータルまたは Azure CLI を使用してコンテナー コンソールに接続する方法については、[こちら](https://learn.microsoft.com/ja-jp/azure/container-apps/container-console?tabs=bash)を参照してください。

# nslookup と dig のインストール

コンテナー コンソールに接続したら、`cat /etc/os-release` を実行して、コンテナーが実行されている Linux ディストリビューションを再確認します。
<IMG  src="https://azureossd.github.io/media/2023/02/azure-blog-container-apps-check-linux-distro.png"  alt="cat / etc / os-releaseを実行してLinuxディストリビューションを確認する"/>

Ubuntu、Debian、Jessie ベースのイメージの場合は、以下を実行する必要があります。


```
apt-get update

apt install dnsutils
```

Alpine ベースのイメージの場合は、以下を実行する必要があります。

```
apk update

apk add bind-tools
```

# nslookup と dig の実行
nslookup コマンドと dig コマンドは、Linux システムでの DNS (ドメイン・ネーム・システム) 解決に使用されます。DNS における名前解決は、ドメイン名がIPアドレスに変換され、コンピューターがインターネットを介して相互に通信できるようにするプロセスです。

nslookup コマンドは、DNS を照会してドメイン名または IP アドレスのマッピング情報を取得するための基本的なツールです。指定された DNS サーバーに DNS クエリを送信し、特定のホスト名またはドメイン名に対応する IP アドレスを返します。nslookup コマンドを使用するための構文は次のとおりです。

`nslookup domain_name`

上記の domain_name は検索するドメイン名です。

たとえば、Microsoft ドメインの IP アドレスを検索するには、次のコマンドを使用します。

`nslookup microsoft.com`

出力には、Microsoft ドメインの IP アドレスと、関連付けられているネーム サーバーの IP アドレスが含まれます。

dig コマンドは nslookup コマンドよりも強力で、DNS クエリに関するより詳細な情報を提供します。A、AAAA、MX、NS、PTR、SOA、SRV、TXTレコードなど、あらゆるDNSレコードタイプのクエリを実行できます。dig コマンドを使用するための構文は次のとおりです。

`dig domain_name record_type`

上記の domain_name は検索するドメイン名で、record_type は照会する DNS レコードの種類です。

たとえば、Microsoft ドメインの MX レコードを検索するには、次のコマンドを使用します。

`dig microsoft.com MX`

nslookup コマンドと dig コマンドはどちらも、DNS サーバーに接続できること、および DNS エントリが正しい場所を指していることを検証するのに役立ちます。

<IMG  src="https://azureossd.github.io/media/2023/02/azure-blog-container-apps-run-dig-nslookup.png"  alt="nslookup と dig の実行"/>

# 参考記事
- [Azure Container Apps 環境でのネットワーク](https://learn.microsoft.com/ja-jp/azure/container-apps/networking?tabs=azure-cli)
- [ネットワーク セキュリティ グループを使用した Azure Container Apps でのカスタム VNET のセキュリティ保護](https://learn.microsoft.com/ja-jp/azure/container-apps/firewall-integration?tabs=workload-profiles)

<br>
<br>

---

<br>
<br>

2023 年 11 月 07 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>