---
title: "エクスポートできないリソースタイプがあると指摘されてしまいます。"
author_name: "Hirotaka Shionoiri"
tags:
    - ARM Template
    - API Management
---

# 質問
API Management において ARM テンプレートのエクスポートを実行したところ、<br>
下記画像の様にエクスポートできないリソースタイプに関するエラーが表示されました。<br>
![image.png]({{site.baseurl}}/media/2021/07/2021-07-09-warning-arm-template-export.png)<br>

特に悪い設定などはしていないつもりなのですが、どうしたら解決できますか？<br>
また、このようなエラーが出てしまうリソースタイプの一覧はありますか？

# 回答
こちらはユーザーの設定等による問題ではなく、現時点でのプラットフォーム側の動作仕様となります。<br>
このエラーの表示される原因は Azure の ARM テンプレートと各サービスの互換性がないためとなります。

# 詳細
この警告は、各リソースと ARM テンプレートの間での互換性が提供されておらず、まだエクスポートのための対応ができていないために発生するものです。<br>
各リソースにおいて利用ができる ARM テンプレ―トのエクスポート機能については、各サービスと ARM テンプレートの間に互換性が必要となります。<br>
各サービスが新しい更新をリリースした際、稀に既存のリソースと互換性がない場合があります。<br>
この時、この互換性のない部分は一時的にエクスポートができない状態となります。<br>
また、一部サービスにおいては互換性がない状態が続いているものもあります。<br>
典型的な例としては本件で話題になっている API Management のテンプレートが挙げられます。<br>
上記のような理由から、こちらはユーザーの設定による問題ではなく、またユーザー側で解決できるものではありません。<br>

大変恐縮ながら、これらのエクスポートができないリソースタイプの一覧については 2021 年 6 月現在ではご用意がございません。<br>
これは、上述の通り各サービスの更新や対応によって各サービスと ARM の間の互換状況がすぐに変化するためです。

# 補足
なお、API Management において、2021 年 7 月 9 日現在では、テンプレートのエクスポートを実施すると、<br>
最大で下記の4つのリソースタイプがエクスポートできないと警告されました。

- Microsoft.ApiManagement/service/contentType
- Microsoft.ApiManagement/service/apis/policy
- Microsoft.ApiManagement/service/apis/operations/policy
- Microsoft.ApiManagement/service/products/policy

Microsoft.ApiManagement/service/contentType は開発者ポータル編集時に表示される編集項目を示しています。<br>
Microsoft.ApiManagement/service/apis/policy、Microsoft.ApiManagement/service/apis/operations/policy、<br>
Microsoft.ApiManagement/service/products/policyはそれぞれ、<br>
各API、各APIのOperation、各製品のポリシーを示しています。<br>
<br>
![image.png]({{site.baseurl}}/media/2021/07/2021-07-09-apim-policy.png)<br>
<br>
また、contentType 以外の3つについては、エクスポートできないエラーは表示されているものの、<br>
テンプレート内にポリシーが出力されているようにも見受けられました。<br>
<br>
![image.png]({{site.baseurl}}/media/2021/07/2021-07-09-apim-arm-template.png)<br>
<br>
contentType については下記の API を利用して確認することが可能です。<br>
[**Content Type - List By Service**](https://docs.microsoft.com/ja-jp/rest/api/apimanagement/2021-01-01-preview/contenttype/listbyservice)

<br>
<br>

---

<br>
<br>

2021 年 7 月 9 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>