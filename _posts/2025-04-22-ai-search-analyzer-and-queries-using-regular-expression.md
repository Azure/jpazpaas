---
title: "AI Search アナライザーのトークナイズと正規表現による検索の動作について"
author_name: "yiwenchen"
tags:
    - "AI Search"
---
こんにちは、PaaS Developer チームの陳です。

AI Search インデックスを作成する際に、選択したアナライザーでドキュメントがどのようにトークナイズされるのかを確認する方法、および正規表現を使用したクエリで検索を行う際の注意点について例を交えて解説します。

# 質問

以下のように日本語ドキュメントをクエリで検索したいが、フィルターがうまくかからず期待された結果が返ってこない、なぜですか？

※フィールドにはデフォルトの standard.lucene アナライザーを使用しています。
~~~
"filter": "not search.ismatch('/.* 階層 .*/','field_1,field_2','full','any')"
~~~

# 回答

まず、上記のクエリは正規表現を使用しており、フィールド "field_1" と "field_2" に "階層" を含むドキュメントを検索するといった動作となります。<br>
[Lucene クエリ構文 - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/query-lucene-syntax#bkmk_regex)

ドキュメントをインデックスに保存された際のトークナイズ動作については、ドキュメントをインデックスに保存した時点で、各フィールドに設定されたアナライザーによってトークナイズされてから保存されます。

たとえば、"階層関係" というドキュメントをそれぞれ、standard.lucene アナライザーのフィールドと、ja.lucene アナライザーのフィールドに保存された場合、

異なる方法でトークナイズされます。

具体的にドキュメントは異なるアナライザーによってどのようにトークナイズされるかを確認するには、以下の REST API をご使用ください。<br>
[Indexes - Analyze - REST API (Azure Search Service) | Microsoft Learn](https://learn.microsoft.com/ja-jp/rest/api/searchservice/indexes/analyze?view=rest-searchservice-2024-07-01&tabs=HTTP)

ここで上記の REST API を使用して "階層関係" はどのようにトークナイズされたかを見てみましょう。

### standard.lucene アナライザー

![image-b7a4dfde-b265-4c59-917d-18d09598910b.png]({{site.baseurl}}/media/2025/04/image-b7a4dfde-b265-4c59-917d-18d09598910b.png)

上記の結果から分かるように、standard.lucene アナライザーは日本語の語彙を考えずに単純な一つ一つの文字（Unicodeでエンコードされた表示）にトークナイズされました。

では、ja.lucene はどうでしょうか

![image-f9410ceb-0119-4368-94cd-938efedb5ec1.png]({{site.baseurl}}/media/2025/04/image-f9410ceb-0119-4368-94cd-938efedb5ec1.png)

上記の結果から、ja.lucene アナライザーは日本語の語彙を考え、"階層" と "関係" でトークナイズしました。

ここで先にクエリで正規表現を使用する場合と使用しない場合のトークナイズの動作の違いについて説明します。

1. クエリで正規表現を使用しない場合、検索語句（ここでは "階層"）はフィールドに設定されているアナライザーによってトークナイズされてから、トークン単位で一致するものはないか検索し、一致するものがあったら、そのドキュメントを結果に返します。

2. **クエリで正規表現を使用する場合、検索語句はアナライザーによってトークナイズされず、そのままで検索にかけます**。<br>
[部分的な語句、パターン、特殊文字 - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/search-query-partial-matching#about-partial-term-search)
~~~
正規表現、ワイルドカード、あいまい検索の場合は、クエリ時にアナライザーは使用されません。 演算子と区切り記号の存在を基にパーサーによって検出されるこれらのクエリ フォームの場合、クエリ文字列は字句解析なしでエンジンに渡されます。 これらのクエリ フォームでは、フィールドに指定されたアナライザーは無視されます。
~~~

そのため、上記に説明した動作から分かるように、standrard.lucene アナライザーは "階層" を "階" と "層" でトークナイズし、正規表現はアナライザーを適用しないので、

"階層" で検索しても、"階" と "層" はどちらでも一致しないことが分かります。

この場合は、日本語アナライザー ja.lucene、もしくは ja.microsoft を使用する必要があります。

それでは、もう一つの例を見てみましょう

# 質問

フィールドに日本語アナライザー ja.microsoft を設定したのに、以下のクエリで検索したら、"階層関係" が含まれているものが取れてしまいます。なぜでしょうか。
~~~
'filter' : 'not search.ismatch'/.* 階層関係 .*/','field_1,field_2','full','any''
~~~

# 回答

さきほどの説明から、正規表現を使用するときは検索語句がトークナイズされずそのまま検索にかける、"階層関係" は日本語アナライザーによって、"階層" と "関係" にトークナイズされることがわかります。

そのため、"階層関係" と一致するものはなく、全件返ってくる動作も理解できます。

さて、この状況では期待された動作にするためには、どうすればいいでしょうか。

今の状況から、正規表現の部分はトークナイズされず検索されているため、ドキュメントにある "階層関係" の部分もトークナイズされなかったら、正規表現と一致するようになることが分かります。

そのため、別のアナライザーで "階層関係" を分割しないものがあればよいわけです。

その要件に合致するのは Keyword アナライザーとなります。<br>
[KeywordAnalyzer (Lucene 6.6.1 API)](https://lucene.apache.org/core/6_6_1/analyzers-common/org/apache/lucene/analysis/core/KeywordAnalyzer.html)

Keyword アナライザーはドキュメントを分割せずにそのまあ一つのトークンとして保存します。これを使用すれば正規表現を使用して検索するときも期待された結果になります。

ただし、Keyword アナライザーを使用することで、逆に正規表現を使用しないクエリで検索すると、トークナイズされた検索語句で分割されていないトークンと一致しなくなるため、使用の場面を考慮してどのアナライザーを使用するかを検討する必要があります。

※また、正規表現のクエリや正規表現は特にコストがかかります。 高度な検索には非常に便利ですが、正規表現が複雑な場合や大量のデータを検索する場合は特に、実行に多くの処理能力が必要になる場合があります。 これらの要因に寄って、検索待ち時間が長くなる可能性があります。 軽減策として、正規表現を簡略化するか、複雑なクエリをより小さく管理しやすいクエリに分割してみてください。<br>
[https://learn.microsoft.com/ja-jp/azure/search/search-performance-tips#tip-consider-alternatives-to-regular-expression-queries](https://learn.microsoft.com/ja-jp/azure/search/search-performance-tips#tip-consider-alternatives-to-regular-expression-queries)


