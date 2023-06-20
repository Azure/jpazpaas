---
title: "Cognitive Search で検索結果が想定されたものでない"
author_name: "Takumi Nagaya"
tags:
    - Cognitive Search
---

# 質問
日本語でインデックスを検索した場合に、想定された検索結果になりません。例えば「ねこ」と検索すると「こねる」といったコンテンツも検索されてしまいます。

# 回答
想定された検索結果が表示されない場合、もしくは想定された検索スコアでない場合のよくある事象として、
検索文字列およびインデックスに格納された検索対象のコンテンツが、ご利用の言語 (日本語) に対して、適切にトークン化されていない可能性が考えられます。<br/><br/>
以下のドキュメントにも記載があります通り、検索コンテンツの言語に対応した言語アナライザーが使用されないと想定外の検索結果になる場合があります。

>存在するとわかっている用語でも一致数が 0 件なのはなぜですか?<br/><br/>
よくあるケースは、クエリの型ごとにサポートされる検索ビヘイビアーと言語分析のレベルが異なることを把握していない場合です。 主要なワークロードである全文検索には、用語を原形に分解する言語分析フェーズが含まれています。 トークン化された用語は、より多くの数の変形と一致するため、このようなクエリ分析はより広い網を一致候補にかけます。<br/><br/>
[Cognitive Search についてよく寄せられる質問](https://learn.microsoft.com/ja-jp/azure/search/search-faq-frequently-asked-questions#--------------------0-----------)

<br/>

**対処策としては、検索対象のインデックスのフィールドにて、検索コンテンツの言語に対応した言語アナライザーを設定し、インデックスを再構築することで、事象が解消する可能性があります。**
<br/>
<br/>
以下の順番で詳細を記載します。<br/>

1. ご利用の言語アナライザーでの検索文字列の字句解析結果の確認
1. 言語アナライザーの選定
1. 新しい言語アナライザーをインデックスのフィールドに設定する

## 1. ご利用の言語アナライザーでの検索文字列の字句解析結果の確認
まず、現状ご利用いただいている言語アナライザーの検索文字列に対する挙動を確認します。<br/>

一般的に Cognitive Search のインデックスの検索時には、以下のドキュメントに記載の4つの手順が実行されます。


>クエリの実行には次の 4 つの段階があります。
>- クエリ解析
>- 字句解析
>- 文書検索
>- ポイントの計算<br/>
>
>[Azure Cognitive Search でのフルテキスト検索](https://learn.microsoft.com/ja-jp/azure/search/search-lucene-query-architecture)

2 番目の「字句解析」の段階では、検索文字列 (例えば「ねこ」) をインデックスのフィールドの言語アナライザーの設定値に応じてトークン化します。<br/>
例えばインデックスの content というフィールドの定義が以下のように `"analyzer": "standard.lucene"` なっていれば、言語アナライザーは `standard.lucene` というものになります。
インデックス作成時に特に明示的に言語アナライザーを指定しない場合は、`standard.lucene` となります。

```json
  "fields": [
    {
      "name": "content",
      "type": "Edm.String",
      "searchable": true,
      "filterable": false,
      "retrievable": true,
      "sortable": false,
      "facetable": false,
      "key": false,
      "indexAnalyzer": null,
      "searchAnalyzer": null,
      "analyzer": "standard.lucene",
      "normalizer": null,
      "synonymMaps": []
    },
```

トークン化の挙動の確認については、ドキュメント [アナライザーの動作テスト](https://learn.microsoft.com/ja-jp/azure/search/search-lucene-query-architecture#testing-analyzer-behaviors) に記載がございます。<br/>
挙動確認には [テキストの分析 (REST API Azure Cognitive Search)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/test-analyzer) を利用しますが、事前に Cognitive Search サービスと、インデックスを用意し、認証のため [API キーの取得](https://learn.microsoft.com/ja-jp/azure/search/search-security-api-keys?tabs=portal-use%2Cportal-find%2Cportal-query#find-existing-keys) をしておく必要があります。

例えば「ねこ」が言語アナライザー `standard.lucene` によってどのように解析されるか確認するために、
以下のようにリクエストを送信します。

```bash
# <searchName> にご利用の Cognitive Search 名
# <index-name> にご利用のインデックス名
# api-key: <***> に API キーを指定します。

curl --location 'https://<searchName>.search.windows.net/indexes/<index-name>/analyze?api-version=2020-06-30' \
--header 'Content-Type: application/json' \
--header 'api-key: <*******>' \
--data '{
  "text": "ねこ",
  "analyzer": "standard.lucene"
}'
```

応答は以下のようになります。「ねこ」が「ね」「こ」に分割されるため、
「ね」「こ」のそれぞれの検索でヒットした回数の合計が多いほど、検索上位に表示されるかたちになります。<br/>
「ねこ」という連続した文字列では検索されていない状況になります。

```json
{
    "@odata.context": https://********.search.windows.net/$metadata#Microsoft.Azure.Search.V2020_06_30.AnalyzeResult,
    "tokens": [
        {
            "token": "ね",
            "startOffset": 0,
            "endOffset": 1,
            "position": 0
        },
        {
            "token": "こ",
            "startOffset": 1,
            "endOffset": 2,
            "position": 1
        }
    ]
}
```

補足しますと、トークン化は検索時のみではなく、インデックスにデータを登録する際にも検索対象のコンテンツに対しても実行され、検索のためにインデックス内に保持されます。<br/>
検索対象のコンテンツがトークン化され、「ねこ」という単位のトークンが生成されれば、「ねこ」という文字列の検索にマッチすることになります。<br/>


## 2. 言語アナライザーの選定
この挙動を改善するには、検索に使用する言語に応じた言語アナライザーを利用します。<br/>
言語アナライザーは、各言語に対応したものが提供されております。
日本語の場合、`ja.microsoft`、`ja.lucene` が提供されております。<br/>詳細はドキュメント [サポートされる言語アナライザー](https://learn.microsoft.com/ja-jp/azure/search/index-add-language-analyzers#supported-language-analyzers) をご確認ください。

言語アナライザーを日本語対応した `ja.lucene` としてトークン化を検証しますと、

```bash
curl --location 'https://<searchName>.search.windows.net/indexes/<index-name>/analyze?api-version=2020-06-30' \
--header 'Content-Type: application/json' \
--header 'api-key: <*******>' \
--data '{
  "text": "ねこ",
  "analyzer": "ja.lucene"
}'
```

今度は以下のように「ねこ」が一語として認識されており、検索精度が向上することが期待されます。

```json
{
    "@odata.context": https://******.search.windows.net/$metadata#Microsoft.Azure.Search.V2020_06_30.AnalyzeResult,
    "tokens": [
        {
            "token": "ねこ",
            "startOffset": 0,
            "endOffset": 2,
            "position": 0
        }
    ]
}
```

### 3. 新しい言語アナライザーをインデックスのフィールドに設定する
言語アナライザーが選定できましたら、インデックスのフィールドに対して設定し検索時に利用できるようにします。<br/>
検索対象のコンテンツが格納されるフィールドの `analyzer` を `ja.microsoft` または `ja.lucene` として、
インデックスを再構築します。

```json
  "fields": [
    {
      "name": "content",
      "type": "Edm.String",
      "searchable": true,
      "filterable": false,
      "retrievable": true,
      "sortable": false,
      "facetable": false,
      "key": false,
      "indexAnalyzer": null,
      "searchAnalyzer": null,
      "analyzer": "ja.lucene",
      "normalizer": null,
      "synonymMaps": []
    },
```

恐れ入りますが、以下のドキュメントに記載の通り、<br/>
言語アナライザーの変更にはインデックスの再作成が必要となりますのでご注意ください。


>既存のフィールドのアナライザーを変更するには、インデックス全体を削除して再作成する必要があります (個々のフィールドを再構築することはできません)。 <br/>
運用環境のインデックスの場合は、新しいアナライザーの割り当てを指定した新しいフィールドを作成することで再構築を延期し、古いものの代わりに使用を開始することができます。<br/><br/>
[アナライザーを追加する時期](https://learn.microsoft.com/ja-jp/azure/search/search-analyzers#when-to-add-analyzers)

<br/>

実際に検索の挙動を確認すると、インデックスのフィールドの言語アナライザーが `standard.lucene` の場合、<br/>
「ねこ」と検索すると、「こねる」といった別のコンテンツも検索結果に表示されてしまいます。

![image-b35c740c-f95c-49d4-81b9-434f0b0d2da5.png]({{site.baseurl}}/media/2023/06/image-b35c740c-f95c-49d4-81b9-434f0b0d2da5.png)

<br/>

一方で、インデックスのフィールドの言語アナライザーが `ja.lucene` の場合、<br/>「ねこ」を含むコンテンツのみが検索されており、想定された検索結果になっております。

![image-2d095939-6f5f-4e13-93e1-7443f0a0c80e.png]({{site.baseurl}}/media/2023/06/image-2d095939-6f5f-4e13-93e1-7443f0a0c80e.png)

# 参考ドキュメント
- [Cognitive Search についてよく寄せられる質問](https://learn.microsoft.com/ja-jp/azure/search/search-faq-frequently-asked-questions#--------------------0-----------)
- [Azure Cognitive Search でのフルテキスト検索](https://learn.microsoft.com/ja-jp/azure/search/search-lucene-query-architecture)
- [Azure Cognitive Search でのテキスト処理のためのアナライザー](https://learn.microsoft.com/ja-jp/azure/search/search-analyzers)
- [アナライザーの動作テスト](https://learn.microsoft.com/ja-jp/azure/search/search-lucene-query-architecture#testing-analyzer-behaviors)

<br>
<br>

---

<br>
<br>

2023 年 06 月 20 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>