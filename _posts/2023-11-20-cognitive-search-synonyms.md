---
title: "Cognitive Search のシノニム（同意語）検索機能を使用する方法"
author_name: "Chansik Lee"
tags:
    - Cognitive Search
---

# はじめに
お世話になっております。PaaS Dev サポート担当の李です。<br>
Cognitive Search にはインデックス検索時にそのキーワードと共に事前に登録した同意語を一緒に検索する、シノニム検索機能をサポートしております。<br>
本日はシノニム検索の使用方法をご案内致します。手順の一部やシノニムの詳細に関しては以下の弊社公式ドキュメントからもご参照頂けます。<br>
[Azure Cognitive Search でのシノニム](https://learn.microsoft.com/ja-jp/azure/search/search-synonyms)<br>

# サンプル環境の用意
Cognitive Search にデータソース・インデックスなどを Azure ポータルの UI で簡単に登録する手順は以下のページをご参照ください。<br>
[Azure Cognitive Search のデータのインポート ウィザード](https://learn.microsoft.com/ja-jp/azure/search/search-import-data-portal)<br>
今回のデータソースである BLOB ストレージには以下のテキストファイルを用意して検索結果を検証します。<br>
|ファイル名|本文|
|---|---|
|テスト1.txt|マイクロソフト|
|テスト2.txt|日本マイクロソフト|
|テスト3.txt|MS|
|テスト4.txt|Office365|

![image-a07787b8-f56e-4a7c-a144-8ad27e56a0a5.png]({{site.baseurl}}/media/2023/11/image-a07787b8-f56e-4a7c-a144-8ad27e56a0a5.png)<br>
![image-05d895d1-9762-4adc-b7f6-41bb7ad6125a.png]({{site.baseurl}}/media/2023/11/image-05d895d1-9762-4adc-b7f6-41bb7ad6125a.png)<br>

# シノニムマップの登録と確認
冒頭でご案内致しました公式ドキュメントにも記載されておりますが、本記事が投稿された現時点では Azure ポータル UI でのシノニムマップの登録はサポートされておらず、REST API または C#・Python などで登録を行う必要がございます。本記事では REST API を用いて登録する手順をご案内致します。<br>

① Cognitive Search の「キー」から API キーを取得します。<br>
![image-e8e933ff-9cf9-4bae-a4e4-e9f75fdc84c6.png]({{site.baseurl}}/media/2023/11/image-e8e933ff-9cf9-4bae-a4e4-e9f75fdc84c6.png)<br>

② 下記の内容で POST します。201 応答が返されます。<br>
```
POST https://[service name].search.windows.net/synonymmaps?api-version=[api-version]

Header
  Content-Type: application/json  
  api-key: [admin key]

Body
  {
    "name" : "[synonymmap name]",  
    "format" : "solr",  
    "synonyms" : "マイクロソフト, 日本マイクロソフト, MS"
  }
```
![image-45b9885b-59e7-4af4-b38b-7480eadd3b19.png]({{site.baseurl}}/media/2023/11/image-45b9885b-59e7-4af4-b38b-7480eadd3b19.png)<br>
![image-7b490c4a-24ce-4719-84f7-cf040a45b2be.png]({{site.baseurl}}/media/2023/11/image-7b490c4a-24ce-4719-84f7-cf040a45b2be.png)<br>

③ 登録されたシノニムマップを確認する場合は、同じ内容で GET すると登録されたシノニムマップのリストを取得することができます。<br>
```
GET https://[service name].search.windows.net/synonymmaps?api-version=[api-version]

Header
  Content-Type: application/json  
  api-key: [admin key]
```
![image-c69a5241-273f-4c44-a0d7-8722a1a6e8b4.png]({{site.baseurl}}/media/2023/11/image-c69a5241-273f-4c44-a0d7-8722a1a6e8b4.png)<br>

④ インデックスの JSON 定義で、「content」の「synonymMaps」に②で登録したシノニムマップ名を入力します。これでシノニムマップの登録手順は完了です。<br>
![image-734bfa7d-e0e1-4428-bd71-f4dfcb229d0a.png]({{site.baseurl}}/media/2023/11/image-734bfa7d-e0e1-4428-bd71-f4dfcb229d0a.png)<br>
![image-56188cdd-58b7-4a3e-a019-b1464c24b69d.png]({{site.baseurl}}/media/2023/11/image-56188cdd-58b7-4a3e-a019-b1464c24b69d.png)<br>

⑤ クエリに「日本マイクロソフト」と検索しても、「マイクロソフト」及び「MS」の検索も同時に行われます。これで同意語の検索ができるようになりました。<br>
![image-03610b09-c9f9-49ab-9152-7608f2b53bee.png]({{site.baseurl}}/media/2023/11/image-03610b09-c9f9-49ab-9152-7608f2b53bee.png)<br>

# 明示的なマッピングの設定
明示的なマッピングの規則は、矢印「=>」によって示され、「=>」の左側に一致する検索クエリの用語のシーケンスが、クエリ時に右側の代替語で置き換えられる機能となります。
例えば下記の様にマイクロソフト, 日本マイクロソフト, MS => MS でシノニムマップを登録した場合は、いずれのキーワードで検索した場合でも「MS」が含まれている「テスト3.txt」のみが検索される機能となっております。<br>
```
{
  "name" : "[synonymmap name]",  
  "format" : "solr",  
  "synonyms" : "マイクロソフト, 日本マイクロソフト, MS => MS"
}
```
![image-b0ee49c0-35d4-43d3-9dca-2e04a150eebd.png]({{site.baseurl}}/media/2023/11/image-b0ee49c0-35d4-43d3-9dca-2e04a150eebd.png)<br>

# シノニムマップの更新または削除
下記の内容で更新は PUT、削除は DELETE リクエストすると、シノニムマップを更新・削除することができます。
```
PUT/DELETE https://[service name].search.windows.net/synonymmaps/[synonymmap name]?api-version=[api-version]  
  Content-Type: application/json  
  api-key: [admin key]  
```

各REST APIの詳細は以下のページをご参照ください。<br>
[シノニム マップの作成 (REST API Azure Cognitive Search)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/create-synonym-map)<br>
[シノニム マップの一覧表示 (Azure Cognitive Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/list-synonym-maps)<br>
[シノニム マップの更新 (REST API Azure Cognitive Search)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/update-synonym-map)<br>
[シノニム マップの削除 (Azure Cognitive Search REST API)](https://learn.microsoft.com/ja-jp/rest/api/searchservice/delete-synonym-map)<br>


# 登録可能なシノニムマップ数
シノニム マップの最大数はレベルによって異なり、最大20の拡張を含めることができます。<br>
![image-59f3b9b5-a4ce-48ae-a065-0600f66df729.png]({{site.baseurl}}/media/2023/11/image-59f3b9b5-a4ce-48ae-a065-0600f66df729.png)<br>
詳細は以下のページをご参照ください。<br>
[シノニムの制限](https://learn.microsoft.com/ja-jp/azure/search/search-limits-quotas-capacity#synonym-limits)<br>


以上、Cognitive Search のシノニム検索機能を紹介いたしました。<br>


<br>
<br>

---

<br>
<br>

2023 年 10 月 28 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>