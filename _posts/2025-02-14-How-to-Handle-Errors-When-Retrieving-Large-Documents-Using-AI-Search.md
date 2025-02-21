---
title: "AI Search で 10 万件を超えるドキュメントを取得する際に発生するエラーの回避法"
author_name: "Yohei Ueda"
tags:
    - AI Search
---

# 質問
10 万件を超えるドキュメントが格納されたインデックスに対して、Python SDK を使用してすべてのドキュメントを取得しようとした時、エラーが出て取得できない。  
大量データのドキュメントを取得するにはどのようにしたら良いでしょうか。

## エラー内容  
```
azure.core.exceptions.HttpResponseError: (InvalidRequestParameter) Value must be between 0 and 100000.  
Parameter name: $skip  
Code: InvalidRequestParameter  
Message: Value must be between 0 and 100000.  
```

## エラーが発生するコード例
```python
search_client = SearchClient(
    endpoint=ENDPOINT,
    index_name=index_name,
    credential=CREDENTIAL,
    api_version=AI_SEARCH_API_VERSION,
)

results = search_client.search(search_text='*', select=['id'])
get_document = [{'id': doc['id']} for doc in results]

with open('output.json', 'w', encoding='utf-8') as f:
    json.dump(get_document, f, ensure_ascii=False, indent=4)
```


# 回答
このエラーは、SDK の内部処理で利用している skip クエリパラメーターの制限により発生しているエラーです。  
一度に取得するドキュメント数が 100,000 件を超える場合、このエラーが発生いたします。

### Ms Lean のドキュメント抜粋
> $skip はスキップする検索結果の数です。  この値は 100,000 を超えることはできません。 この制限により使用 $skip できない場合は、インデックス内のすべてのドキュメントに対して一意の値を持つフィールド (たとえば、ドキュメント キーなど) に対して$orderbyを使用し、代わりに範囲クエリで$filterすることを検討してください。<br/>
[ドキュメントの検索 - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/rest/api/searchservice/search-documents#query-parameters:~:text=%E3%81%93%E3%81%AE%E5%80%A4%E3%81%AF%20100%2C000%20%E3%82%92%E8%B6%85%E3%81%88%E3%82%8B%E3%81%93%E3%81%A8%E3%81%AF%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%9B%E3%82%93%E3%80%82)

<br>


この制限の回避策として 2 つの方法をご提案しますので、  
それぞれのご状況に合わせて「[シナリオ A : facets と filter を利用して複数回に分けてデータを取得する場合](#unchor-scenario_a)」もしくは「[シナリオ B : orderby と filter を利用して複数回に分けてデータを取得する場合](#unchor-scenario_b)」をご確認ください。


#  <a id="unchor-scenario_a"></a>シナリオ A : facets と filter を利用して複数回に分けてデータを取得する場合 
facets と filter を適用しているフィールドがある場合は、facets と filter を利用して複数回に分けてデータを取得することができます。  
以下の手順で行うことができます。

１．facets パラメータを使用して、各 facets のデータを取得します。

２．facets で取得した各データに対して、filter を利用してインデックスのすべてのデータを取得します。


## サンプルコード
```python
# クライアントの作成
search_client = SearchClient(
   endpoint=ENDPOINT,
   index_name=index_name,
   credential=CREDENTIAL,
   api_version=AI_SEARCH_API_VERSION,
)

# facets の設定
facet_results = search_client.search(search_text='*', facets=["category,count:0"])
facets = facet_results.get_facets()
all_docs = []

# facets と filter の適用
for facet in facets['category']:
   filter_expression = f"category eq '{facet['value']}'"
   results = search_client.search(search_text='*', filter=filter_expression, select=['id'])
   docs_to_update = [{'id': doc['id'], 'category': facet['value']} for doc in results]
   all_docs.extend(docs_to_update)

# JSON file　の出力
with open('output.json', 'w', encoding='utf-8') as f:
   json.dump(all_docs, f, ensure_ascii=False, indent=4)
```
 
<br>

ただ、facets および filterable 属性は、フィールドが最初にインデックスに追加されたときにのみ有効にできます。  
既存のフィールドを修正して有効にすることはできないので、その場合はフィールドの再作成をしていただけますと幸いです。  

また、facets により集約したデータが 100,000 件以上の場合は、制限により取得ができません。

補足ですが、 facets による集約後のデータの取得件数は、既定値では 10 件となっているのでご注意ください。  
サンプルコードでは facets=["category,count:0"] のように設定して、ファセットを無制限に変更しています。  
[ファセット ナビゲーション カテゴリ階層を追加する - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/search-faceted-navigation#:~:text=%E3%83%95%E3%82%A1%E3%82%BB%E3%83%83%E3%83%88%E3%82%92%E7%84%A1%E5%88%B6%E9%99%90%E3%81%AB%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AB%E3%80%81%22count%22%3A%20%220%22%20%E3%82%92%E6%8C%87%E5%AE%9A%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82)

<br>

#  <a id="unchor-scenario_b"></a>シナリオ B : orderby と filter を利用して複数回に分けてデータを取得する場合
orderby と filter を適用しているフィールドがある場合は、orderby と filter を利用して複数回に分けてデータを取得することができます。  
以下の手順で行うことができます。
 

１．順番付けされた一意のフィールドに対して最初の 100,000 件の結果を取得します。

２．取得した結果の一意のフィールドの最大値を確認します。(取得した最後のデータ)

３．次に一意のフィールドに対してフィルターを適用して、先ほど取得した最大値よりも大きい結果を100,000件取得します。

４．これを繰り返すことで、全てのインデックスが取得可能です。

## サンプルコード
```python
search_client = SearchClient(
    endpoint=ENDPOINT,
    index_name=index_name,
    credential=CREDENTIAL,
    api_version=AI_SEARCH_API_VERSION,
)

all_docs = []
batch_size = 100000
unique_field = 'id'
last_value = None

while True:
    filter_expression = f"{unique_field} gt {last_value}" if last_value else None
    results = search_client.search(
        search_text='*',
        filter=filter_expression,
        order_by=[unique_field],
        select=['id'],
        top=batch_size
    )

    docs_to_update = [{'id': doc['id']} for doc in results]
    all_docs.extend(docs_to_update)

    if len(docs_to_update) < batch_size:
        break

    last_value = docs_to_update[-1]['id']

with open('output-orderby.json', 'w', encoding='utf-8') as f:
    json.dump(all_docs, f, ensure_ascii=False, indent=4)
```

ただ、filterable および sortable 属性は、フィールドが最初にインデックスに追加されたときにのみ有効にできます。  
既存のフィールドを修正して有効にすることはできないので、その場合はフィールドの再作成をしていただけますと幸いです。

# 免責事項
当記事で紹介するサンプルコードは、要件を満たす最低限の処理を実装したコードであり、弊社にてその動作を保証するものではございません。ご使用の際には、お客様の環境に合わせて変更およびエラー処理を追加していただき、検証、動作確認をご実施くださいますようお願いいたします。サンプル内で使用しております API などの詳細な情報に関しては、弊社公開情報等をご参照くださいますようお願い申し上げます。

# 参考ドキュメント
- [ドキュメントの検索](https://learn.microsoft.com/ja-jp/rest/api/searchservice/search-documents)
- [ファセット ナビゲーション カテゴリ階層を追加する](https://learn.microsoft.com/ja-jp/azure/search/search-faceted-navigation)

---

<br>
<br>

2025 年 2 月 14 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>