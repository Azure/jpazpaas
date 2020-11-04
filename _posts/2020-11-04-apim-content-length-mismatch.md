---
title: "APIM のリトライポリシーでリトライ時に content-length mismatch 発生する時の原因と回避方法"
author_name: "Yusuke Yoneda"
tags:
    - API Management
---

# 質問
API Management でバックエンドからエラーが応答された場合に備えてリトライポリシーを設定しています。
しかし、リトライ動作時に Content length mismatch ProtocolViolation at forward-request のエラーが発生する場合があります。この原因と対処について教えてください。


# 回答
指定されたバックエンド サービスに要求を転送する forward-request ポリシーは既定でリクエスト body をバッファしない動作となるため、リトライが発生した場合 body データを含めず再リクエストが発生いたします。
この時、POST などの body を含むリクエストは Content-Length ヘッダで指定されたデータ長と実際の body データサイズに差異が生じるため Content length mismatch がバックエンドから応答されることがあります。  
この対処として forward-request ポリシーのオプションで、buffer-request-body="true" を指定いただくことでリクエスト body のバッファを有効化することが可能です。これにより、リトライのリクエストにも body データが含まれ Content length mismatch が発生しなくなります。

例：
```xml
<retry condition="@(context.Response.StatusCode==503 || context.Response.StatusCode==502)" count="3" interval="1" max-interval="3" delta="1" first-fast-retry="false">
    <forward-request buffer-request-body="true" />
</retry>
```

# 参考ドキュメント

[API Management の高度なポリシー - 再試行](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-advanced-policies#retry)


[API Management の高度なポリシー - 要求を転送する](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-advanced-policies#forward-request)

---
<br>

2020 年 11 月 04 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>