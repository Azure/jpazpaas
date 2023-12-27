---
title: "Azure Container Apps で gRPC を動作させる"
author_name: "Daisuke Sekiguchi"
tags:
    - Container Apps
---

※本記事は Azure OSS Developer Support ブログ [Container Apps: Running gRPC with Container Apps
](https://azureossd.github.io/2022/07/07/Running-gRPC-with-Container-Apps/index.html) の日本語抄訳版です。

Container Apps サポート担当の関口です。

[Azure Container Apps](https://learn.microsoft.com/ja-jp/azure/container-apps/overview) は、[Kubernetes](https://kubernetes.io/docs/concepts/overview/) 上で動作する新しいサービスですが、 [gRPC](https://grpc.io/) (トランスポート プロトコル として HTTP/2 を使用する) をサポートしている点が、有用な特徴の 1 つとしてあげられます。

gRPC の使用にあたっては、いくつかの新しい概念を意識する必要がありますが、その一部について本投稿にて記載します。


# 設定
***
**transport プロパティ**

コンテナー アプリの作成において、特に明示的な指定をしない場合、デフォルトで `transport` プロパティは `Auto` に設定されます。

`transport` プロパティの値には、`Auto`、`http`、或いは `http2` を設定する事ができます。 `transport` の設定は、以下のようにポータルからコンテナー アプリの概要 (Overview) ブレードにて確認できます。

<IMG  src="https://azureossd.github.io/media/2022/07/azure-grpc-blog-1.png"  alt="Overview Blade"/>

現時点では、`transport` の設定を直接ポータル上で `http2` に変更する方法はありませんが、[AZ CLI](https://learn.microsoft.com/ja-jp/cli/azure/containerapp/ingress?view=azure-cli-latest#az-containerapp-ingress-enable)、ARM、BICEP 等を用いて、以下のように作成時か或いは作成後に指定することが可能です。

BICEP/ARM:
```
..code..
ingress: {
    external: true
    targetPort: 50051
    // Set transport to http2 for gRPC
    transport: 'http2'
}
..code..
```

Azure CLI:
```
az containerapp ingress enable --name mycontainerapp --resource-group my-rg --target-port 50051 --type external --transport http2
```

上記リクエストが成功すると、`transport` プロパティは新しい値 (`http2`) で表示されます。

**トラブルシューティング**

`transport` が `Auto` または `http` に設定されているときに gRPC サービスを呼び出すと、次のエラーとなるでしょう。

```
{
  "error": "14 UNAVAILABLE: upstream connect error or disconnect/reset before headers. retried and the latest reset reason: protocol error"
}
```

このエラーとなった場合には、上述の方法や [AZ CLI](https://learn.microsoft.com/en-us/cli/azure/containerapp/ingress?view=azure-cli-latest#az-containerapp-ingress-show) にて次のようなコマンドを使用し、`transport` が `http2` に設定されていることを再確認してください。

`az containerapp ingress show --name mycontainerapp --resource-group my-rg`

# リフレクション
***
**責任範囲**

リフレクションを明示的に有効にしない限り、本質的に gRPC サーバーは呼び出し元に対して非公開となります。

使用している言語の gRPC パッケージによっては、リフレクションを利用できる場合とできない場合があります。 公式の gRPC リフレクションをサポートするパッケージは[こちら](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md#known-implementations)にあります。

**リフレクションを実装するか否かは開発者の判断次第となります。 リフレクションはコードで実装され、Azure の責任範囲ではありません。**

リフレクションが有効になっていない場合、クライアントに `proto` ファイルが供給されない限り、Azure Container Apps (或いは本質的にその他の環境においても) にデプロイされている gRPC サーバーを呼び出すことはできないという点に注意が必要です。

リフレクションが有効な場合には、gRPC サーバーを呼び出すクライアントに `proto` ファイルまたはコントラクトを供給する必要はありません。 この場合は、gRPC サーバーの RPC メソッドを公開しているような状態と言えます。

>**注**: リフレクションを実装する、或いは実装しないアプリケーションの例は、[こちら](https://github.com/azureossd/grpc-container-app-examples)で確認することができます。

>**Tips**: 一部の言語では、`grpc` パッケージを介したリフレクション サポートがない場合があります。 このような場合には、必要に応じて他のオープンソース パッケージの使用を検討する必要があります。

**トラブルシューティング**

リフレクションが有効になっていないアプリケーションについて、`proto` ファイルが供給されていないクライアントが Azure Container Apps でホストされている gRPC サーバーを呼び出そうとした場合、次のようなエラーがクライアントに返される可能性があります。

`Failed to compute set of methods to expose: server does not support the reflection API`

この場合、`proto` ファイルは呼び出し側のクライアントから提供されるか、gRPC サーバーでのリフレクションを有効にする必要があります。

#クライアント テスト ツール
***
gRPC がサポートされている、UI または CLI ベースのクライアント ツールがいくつかあります。

- [grpcui](https://github.com/fullstorydev/grpcui#grpc-ui) - GUI based
- [grpcurl](https://github.com/fullstorydev/grpcurl#grpcurl) - CLI based
- [BloomRPC](https://github.com/bloomrpc/bloomrpc) - GUI based
- [Insomnia](https://docs.insomnia.rest/insomnia/grpc) - GUI based
- [Postman](https://www.postman.com/) - GUI based

各ツールには、テスト用の gRPC リクエストを作成する独自の方法があります。 クライアント ツールからテストする場合、FQDN にプレフィックスとして https:// を追加しないでください。 これをすると、ローカルでのテストか Azure Container Apps 上でのテストかに関係なく、即時でエラーとなります。具体的には次のようなエラーとなる場合があります。

`Could not invoke method - Error: Invalid protocol: https`

そもそもプロトコルを追加しないようにするか、或いはもしツールがサポートしている場合には、適切に `grpc://` または `grpcs://` をプレフィックスとして付けてください。

---

<br>
<br>

2023 年 12 月 27 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>