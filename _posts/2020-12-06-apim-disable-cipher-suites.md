---
title: "API Managementの特定の暗号スイートを無効化する"
author_name: "Takumi Nagaya"
tags:
    - API Management
---

# 質問
API Management の特定の暗号スイートを無効化することは可能ですか。
# 回答
API Management の暗号スイートは Azure Portal では無効化することはできませんが、
REST API を使用して無効化することが可能です。  
この記事では REST API を用いて特定の暗号スイートを無効化する方法を紹介します。

REST API [Api Management Service - Create Or Update](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2019-12-01/apimanagementservice/createorupdate#request-body) で、リクエストボディ内の `properties.customProperties` に、無効化する暗号スイートを指定します。  

例えば以下のように、リクエストボディに含めれば、暗号スイート `TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA` を無効化できます。
~~~
"Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA": true,`
~~~

以下に詳細な手順を記載します。

## 手順

1. あらかじめ API Management を作成しておき、以下の情報をメモしておきます。

    - API Management が属するサブスクリプション
    - API Management が属するリソースグループ
    - API Management のサービス名（リソース名）

1. API Management を更新する前に現在の設定を REST API Api Management Service - Get を実行し、確認します。
    1. 簡単に REST API を試すため、「使ってみる」機能を使用します。[Api Management Service - Get](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2019-12-01/apimanagementservice/get) のページにアクセスし、「使ってみる」をクリックします。  
       使用する Azure アカウントを選択しログインします。

    1. パラメータに手順 1 でメモした値を入力します。以下の画像を参考にしてください。

        <img alt="apim-parameters" src="{{site.baseurl}}/media/2020/12/2020-12-06-apim-parameters.jpg" width="70%">

    1. 「実行」を押して REST API を実行します。
        レスポンスに含まれる以下の `customProperties` の値をメモしておきます。

    ```json
        "customProperties": {
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Ssl30": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TripleDes168": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls10": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls11": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Ssl30": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Protocols.Server.Http2": "False"
        }
    ```

1. REST API Api Management Service - Create Or Update を使用して暗号スイートを無効化します。
    1. 簡単に REST API を試すため、「使ってみる」機能を使用します。[Api Management Service - Create Or Update](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2019-12-01/apimanagementservice/createorupdate#request-body) のページにアクセスし、「使ってみる」をクリックします。  
       使用する Azure アカウントを選択しログインします。

        <img alt="try-it" src="{{site.baseurl}}/media/2020/12/2020-12-06-apim-try-it.jpg" width="100%">


    1. パラメータに手順 1 でメモした値を入力します。以下の画像を参考にしてください。

        <img alt="apim-parameters" src="{{site.baseurl}}/media/2020/12/2020-12-06-apim-parameters.jpg" width="70%">

    1. リクエスト本文を入力し、「実行」を押して REST API を実行します。  
    `location` や `sku` などの必須項目があるため、[Api Management Service - Create Or Update](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2019-12-01/apimanagementservice/createorupdate#request-body) のドキュメントを参考に、値を入力します。
    この際に、手順 2 の 3 でメモしたプロパティを参考にします。
    以下のサンプルのリクエスト本文では、API Management で無効化できる暗号スイートを全て無効化します。

    ```json
    {
      "location": "westus2",
      "properties": {
        "publisherEmail": "publisher.email@microsoft.com",
        "publisherName": "publisherName",
        "customProperties": {
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_GCM_SHA256": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA256": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA256": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA": false,
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Ssl30": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TripleDes168": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls10": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls11": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Ssl30": "False",
          "Microsoft.WindowsAzure.ApiManagement.Gateway.Protocols.Server.Http2": "False"
        }
      },
      "sku": {
        "name": "Developer",
        "capacity": 1
      }
    }
    ```

1. 暗号スイートを無効化する設定が適用されたか確認するために、REST API [Api Management Service - Get](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2019-12-01/apimanagementservice/get) で API Management の情報を取得します。  
   手順 1 と同様です。  
   レスポンスを確認すると、暗号スイートが無効化されていることがわかります。

```json
    "customProperties": {
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_GCM_SHA256": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA256": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA256": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA": "false",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Ssl30": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TripleDes168": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls10": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Tls11": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Backend.Protocols.Ssl30": "False",
      "Microsoft.WindowsAzure.ApiManagement.Gateway.Protocols.Server.Http2": "False"
    },
```

## 注意事項
### 無効化できない暗号スイートがあります
以下の暗号スイートは API Management を構成する Cloud Services の内部コンポーネントで必要とされるため、無効にすることはできません。

- `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`
- `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
- `TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384`
- `TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256`
- `TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384`
- `TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256`
- `TLS_RSA_WITH_AES_256_GCM_SHA384`

### PATCH操作で `properties.customProperties` のプロパティを全て指定しない場合、省略した値が規定値にリセットされます
例えば以下のように暗号スイートを無効化するようにプロパティを構成した場合、PUT の操作によって、HTTP2 や Triple DES の設定が規定値に戻ってしまいます。  
したがって、手順 3 の 3 で示したように、`properties.customProperties` の全てのプロパティを含めるようにリクエストボディを構成するようご留意いただければと思います。

以下の例だと、未指定のプロパティが規定値に戻ってしまいます。
```json
"customProperties": {
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_GCM_SHA256": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA256": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA256": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_256_CBC_SHA": false,
  "Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_RSA_WITH_AES_128_CBC_SHA": false,
}
```

---

<br>

2020 年 12 月 6 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>