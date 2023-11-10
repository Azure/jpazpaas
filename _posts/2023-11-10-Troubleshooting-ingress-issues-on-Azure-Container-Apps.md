---
title: "Azure Container Apps でのイングレスに関する問題のトラブルシューティング"
author_name: "Hidenori Yatsu"
tags:
    - Container Apps
---


※本記事は Azure OSS Developer Support ブログ [Troubleshooting DNS connectivity on Azure Container Apps](https://azureossd.github.io/2023/03/22/Troubleshooting-ingress-issues-on-Azure-Container-Apps/index.html) の日本語抄訳版です。

Container Apps サポート担当の谷津です。

この投稿では、Azure Container Apps でのさまざまな外部イングレスの問題のトラブルシューティングについて説明します。

# 概要
Azure Container Apps は、[Envoy](https://www.envoyproxy.io/) をプロキシとして使用してイングレスを処理します。これにより、HTTP 1.1 と HTTP2 のトラフィックを処理するだけでなく、gRPC の使用も可能です。

設定ミスやアプリケーションの問題などが原因で、Envoy からエラーメッセージが返されることがあります。これらのメッセージは、発生しているシナリオによって異なる場合があります。

この投稿の範囲では、Container Apps での外部イングレスの使用にのみに焦点を当てます。

# 一般的なエラー
## HTTP
---
### HTTP 404 / Not Found
HTTP 404は、通常はリソースが存在しないことが原因です(例： URI が実際にアプリケーションにマッピングされていない)。 しかしながら、他のいくつかの理由でも発生する可能性があります。

- Container App Environment の DNS は名前解決可能だが、Container Apps 名が無効。
- イングレスが有効になっていない。
- イングレスは有効になっているが、外部に設定されていない。
- Container Apps の DNS は解決可能だが、サイトで要求されたリソースが見つからない、もしくは無効。

#### 推奨されるアクション:

- Container Apps の FQDN が実際に正しいことを確認します。
- イングレスが有効になっていて、**external (外部)** かつ **Accepting traffic from anywhere (どこからでもトラフィックを受け入れます)** に設定されていることを確認します。これが **internalOnly** に設定されている場合、またはポータルで "Limited to Container Apps Environment (Container Apps 環境に限定)" もしくは "Limited to VNet (VNet に限定)" に設定されている場合、HTTP 404 が返されることがあります。
  - これは、Container Apps ポータルの **イングレス** ブレードで確認できます。

    <IMG  src="https://azureossd.github.io/media/2023/03/azure-oss-blog-aca-ingress-1.png"  alt="イングレスブレード"/>

- イングレスが環境外の外部トラフィックに設定されている場合は、要求されているリソースがアプリケーションに実際に存在することを確認します。

**注**: 404 の応答がどのように見えるか (例えば、ブラウザを介して) は、問題をより正確に特定するのに役立ちます。404 がアプリケーションの Web サーバーから明確に返された場合、これはイングレスが意図したとおりに機能していることを意味します。Chrome ならびに Edge 固有の 404 が返される場合、これはイングレスが内部に設定され、正しく構成されていないことを示している可能性があります。

### HTTP 500 
これらは傾向としてアプリケーションのエラーであり、ここでは説明しません。アプリケーションのログを確認する方法については、[Azure Container Apps での監視](https://learn.microsoft.com/ja-jp/azure/container-apps/observability)を参照してください。

### HTTP 502
Envoy から返されるエラーとして表れる可能性があります。

`upstream connect error or disconnect/reset before headers <some_connection_error />`

`stream timeout`

#### 考えられる原因:

- アプリケーションからのタイムアウト ※タイムアウトするリクエスト内からの外部接続先との依存関係
- アプリケーションの起動に失敗している。

#### 推奨されるアクション:

- この要求によって返されたタイムアウトを確認します。240 秒より前に返された場合、これはアプリケーションの問題である可能性があります。Envoy のリクエストのタイムアウト制限は 240 秒に設定されています。
- 呼び出されるルートへの依存関係 (例えば、サードパーティの API、データベースなど) を確認します。
- コンテナーの起動に失敗しているかどうかを確認します。ログを確認する方法はいくつかありますが、確認する方法についての一例は、[Azure Container Apps での監視](https://learn.microsoft.com/ja-jp/azure/container-apps/observability)を参照してください。 

HTTP 502 は `upstream connect error or disconnect/reset before headers. retried and the latest reset reason: protocol error`
 のような別の形式で返される場合があります。こちらも Envoy から返され、より具体的な原因が潜在しています。

#### 考えられる原因:

- プロトコルが想定と不一致
- イングレスのポートが正しくない
- アプリケーションの起動に失敗する
- 要求のタイムアウト

#### 推奨されるアクション:

- イングレスポートが何であるかを確認します。これは、アプリケーションが開放しているポートと一致する必要があります。
- 実行されている要求のプロトコルを確認します。アプリケーションで HTTP/2 の **transport** が設定されているときに HTTP/1.1 要求を実行しようとすると、これが発生する可能性があります。
- 同様に、gRPC アプリケーションの transport を HTTP/2 に設定し、HTTP/1.x 要求を行うとこれが発生する可能性があります。
- transport が "auto" に設定されていて、これが HTTP/2 のサポートを必要とするアプリケーション (gRPC アプリケーションなど) で発生する場合は、代わりにトランスポートを HTTP/2 に設定することを検討してください。
- コンテナーの起動に失敗しているかどうかを確認します。ログを確認する方法はいくつかありますが、確認する方法についての一例は、[Azure Container Apps での監視](https://learn.microsoft.com/ja-jp/azure/container-apps/observability)を参照してください。

### HTTPの504
これは、Envoy からの `stream timeout` を表します。

#### 考えられる原因:

- 前述の 「HTTP 502」における前半の理由のほとんどは、ここでも当てはまります。
- Envoy のリクエストタイムアウトは 240 秒に設定されており、それまでにリクエストが完了しない場合、リクエストはキャンセルされます。
- 応答しない、または 240 秒の制限内に作業を完了しないアップストリームの依存関係がこれを引き起こす可能性があります。

#### 推奨されるアクション:

- 「HTTP 502」における前半の理由のほとんどがここでも当てはまります。

### ネットワーク固有の例
#### エラー コード: ERR_CONNECTION_TIMED_OUT
これは、`リソース名.funnyname1234.region.azurecontainerapps.io` への応答に時間がかかりすぎたため表れる可能性があります

#### 考えられる原因:

- ドメインは有効であるが、到達できないことを示します。これは、VNET 統合のシナリオに適用されます。

#### 推奨されるアクション:

- Container App Environment と統合されているこの VNET に関連付けられているネットワーク セキュリティ グループ (NSG) に、必要なサービス タグ  が欠落していないことを確認します。これには 168.63.129.16 の IP が含まれます。([IP アドレス 168.63.129.16 とは](https://learn.microsoft.com/ja-jp/azure/virtual-network/what-is-ip-address-168-63-129-16))
  - NSG 許可規則については [こちら](https://learn.microsoft.com/ja-jp/azure/container-apps/firewall-integration?tabs=workload-profiles#nsg-allow-rules) を参照してください 。
- 接続しようとしているクライアントのサブネットまたは IP アドレスが NSG 規則によってブロックされていないことを確認します。これは、VNET とインターネットから受信する接続の両方に適用されます。
  - VNET トラフィックの場合、トラフィックを許可するには [VirtualNetwork](https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/virtual-network/service-tags-overview.md) サービス タグで十分です 。
  - インターネットトラフィックの場合、 [Internet Service Tag](https://learn.microsoft.com/ja-jp/azure/virtual-network/service-tags-overview) を使用できる可能性があります。(ただし、一部の企業ではオプションにならない可能性があります)

#### ERR_NAME_NOT_RESOLVED
これは `リソース名.funnyname1234.region.azurecontainerapps.io` サーバーのIPアドレスが見つからなかったために表示されることがあります。

#### 考えられる原因:

- DNS が解決できない： アプリケーションが VNET 内にあるときに、インターネットから Container Apps にアクセスしようとしているクライアントで発生し得ます。

#### 推奨されるアクション:

- アプリケーションにアクセスしようとしているクライアントが VNET 内にあることを確認します。
  - そうでない場合、または VNET が既存の Container App Environment の VNET にピアリングされていない場合、接続は失敗します。
- ネットワークに接続されていない環境では、完全な FQDN が正しく、アプリケーションが存在することを確認します。

## GRPC
---
gRPC アプリケーション (HTTP2 を使用) のイングレスに関する問題は、さまざまな形で現れる可能性があります。

### Could not invoke method - Error: Invalid protocol: https
#### 考えられる原因:

- HTTP/S 要求を行おうとしているクライアントまたはツールを介して gRPC 要求を実行しようとしています。

#### Recomended actions:

- これ自体は、必ずしも Envoy によるイングレスの問題ではありません。
- これは、gRPC サービスが https:// に関与しようとしているという事実によるものです。 代わりに、ツールに応じてプロトコルを省略するか (推測される可能性があるため)、grpc:// または grpcs:// (セキュリティで保護するため) を使用します。

### Failed to compute set of methods to expose: server does not support the reflection API
#### 考えられる原因:

- gRPC サーバー (アプリケーション) がリフレクションをサポートしていないか、リフレクションが有効になっていない。
- コンテナー アプリのイングレスは内部に設定されているが、クライアントは外部にある。

#### 推奨されるアクション:

- リフレクションと、リフレクションがアプリケーションとの通信にどのように影響するかについては、[こちら](https://azureossd.github.io/2022/07/07/Running-gRPC-with-Container-Apps/index.html#reflection)の記事をお読みください。
- 入力が内部のみに設定されていないことを確認します。(デフォルトのFQDNは `リソース名.internal.funnyname12345a.azurecontainerapps.io` のようになります)。クライアントが外部 (ブラウザーなど) で、サーバーリフレクションが実際に有効になっている場合は、イングレスを **external** に設定します。

### stream timeout
#### 考えられる原因:

- イングレスポートが gRPC サーバーがリッスンしているものと一致しない。
- アプリケーションが 240 秒の制限時間内に応答を返さない。 (「HTTP 502」または「HTTP 504」の項を参照)

#### 推奨されるアクション:

- イングレスポートの値を確認し、これが gRPC サーバーのポートと想定される受信リクエストと一致することを確認します。

<br>
<br>

---

<br>
<br>

2023 年 11 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>