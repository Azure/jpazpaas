---
title: "パラメータや関数を含むポリシー定義をARMテンプレートデプロイするときの注意点"
author_name: "Hirotaka Shionoiri"
tags:
    - Policy
---

# 質問
ARMテンプレートデプロイによってAzure Policyの定義を作成しようと考えています。<br>
ポリシー定義内にパラメータや関数を利用していると構文エラーとなりテンプレートデプロイに失敗してしまいます。<br>
どのようにすればこのようなポリシー定義をARMテンプレートを利用してデプロイできますか？

# 回答
パラメータや関数の"[]"をエスケープする必要があります。<br>
関数を利用した文字列の先頭に"["を追加してエスケープ処理をしてください。<br>
<br>

[ARM テンプレートの構文と式 - エスケープ文字](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/template-expressions#escape-characters)


# 解説
ポリシー定義内でパラメータやconcat関数を利用するときには、"[]"を利用する必要があります。<br>
例えば下記のドキュメントをご確認頂くと、concatやparametersの利用のために"[]"を利用している様が<br>
改めてご確認頂けるかと思います。<br>
<br>

[Azure Policy パターン: パラメーター](https://docs.microsoft.com/ja-jp/azure/governance/policy/samples/pattern-parameters)

```
"if": {
    "not": {
        "field": "[concat('tags[', parameters('tagName'), ']')]",
        "equals": "[parameters('tagValue')]"
    }
},
```
この、「[concat...)]」の部分や、「[parameters....)]」の部分がARMテンプレートデプロイ用の式であると解釈されてしまいます。<br>
これはポリシー定義用のものであり、ARMテンプレートに解釈されるタイミングでは通常の文字列として認識させたいため<br>
上記の例の場合には下記の様にエスケープ文字を利用します。<br>
"[[concat"や"[[parameters"と、先頭に"["が追加されている部分にご注目下さい。

```
"if": {
    "not": {
        "field": "[[concat('tags[', parameters('tagName'), ']')]",
        "equals": "[[parameters('tagValue')]"
    }
},
```

# 参考ドキュメント
- [ARM テンプレートの構文と式 - エスケープ文字](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/template-expressions#escape-characters)

- [Azure Policy パターン: パラメーター](https://docs.microsoft.com/ja-jp/azure/governance/policy/samples/pattern-parameters)


<br>
<br>

---

<br>
<br>

2021 年 6 月 4 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>