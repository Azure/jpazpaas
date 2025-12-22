---
title: "ドキュメントキー文字数制限エラーの解消法"
author_name: "Yohei Ueda"
tags:
    - AI Search
---

# 質問
Blob ストレージをデータソースとする AI Search を利用して、インデクサーを実行した際に 「`Document key cannot be longer than 1024 characters`」というエラーが発生し、インデクサーの実行が失敗しました。


# 回答
      
AI Search のインデックス内のドキュメントを一意に識別するドキュメントキーは、エラーにある通り、1,024文字を超えることはできません。
このドキュメントキーは一般的には、BLOBのパス（`metadata_storage_path`）を Base64 でエンコードすることで生成されます。
Blob の エンコードされたURLは、ファイル名に日本語等のマルチバイト文字列が含まれる場合、長さが大きくなり、1,024文字の制限に到達しやすくなります。

エラーの解消方法としては 3 つございます。



### 1. fixedLengthEncode 関数 の利用


      
ファイル名の長さによって、エラーになることを防ぐには、インデクサー定義内で `fixedLengthEncode` 関数を利用します。
`fixedLengthEncode` 関数は、任意の長さの文字列を一意の固定長文字列に変換します。

この関数を使用すると、任意の長さの文字列が固定長文字列に変換されます。  
[インデクサーのフィールドをマッピングする - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-field-mappings?tabs=rest#fixedlengthencode-function)


例えばインデクサー定義を以下のようにします。


```
"fieldMappings" : [
 {
  "sourceFieldName" : "metadata_storage_path",
  "targetFieldName" : "your key field",
   "mappingFunction" : {
    "name" : "fixedLengthEncode"
   }
 }
]
```

<br>
<br>

一方で、テキストを分割し、チャンクごとにインデックスのドキュメントを作成する方法を利用されている場合などは、現時点では `fixedLengthEncode` 関数をサポートしておりません。

そのため、 `fixedLengthEncode` 関数を利用できない場合や、インデクサーの編集が困難な場合には、以下の方法にてエラーを解消することも可能です。

<br>

### 2.ファイル名を短い名前に変更する
2.1. 対象のファイルを特定します。インデクサーの実行履歴の画面を確認します。以下の画像のように、「Target field 'chunk_id' is...」にカーソルを合わせると、エラーの原因となっているファイルを確認することができます。<br/>
![image-4236d7ec-8c2d-42f3-90ce-4f30750e3312.png]({{site.baseurl}}/media/2025/12/image-4236d7ec-8c2d-42f3-90ce-4f30750e3312.png)
2.2. Blobのファイル名自体は変更できないため、一度ファイルをダウンロードし、名前変更の上アップロードし、前のファイルは削除します。

<br>


### 3.インデクサーが対象の BLOB をスキップするように設定する
ファイル名変更が難しい場合は、該当Blobのメタデータに、キー : AzureSearch_Skip、値 : true を設定いただき、インデクサーが該当Blobを処理しないようにスキップさせます。
![image-ff0cd6ea-7a08-4283-a2eb-5257e800b4ff.png]({{site.baseurl}}/media/2025/12/image-ff0cd6ea-7a08-4283-a2eb-5257e800b4ff.png)

AzureSearch_Skipについては以下のドキュメントに記載がございます。

 **抜粋**

>

> `"AzureSearch_Skip" : "true"`

>

> BLOB を完全にスキップするように BLOB インデクサーに指示します。 

> メタデータとコンテンツのどちらの抽出も行われません。 

> 特定の BLOB で何度もエラーが発生し、インデックス作成プロセスが中断されるときに利用できます。

[Azure BLOB インデクサー - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/search-how-to-index-azure-blob-storage#determine-which-blobs-to-index)

<br>

# 参考資料
- [インデクサーのエラーと警告 - Azure AI Search | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-common-errors-warnings#error-could-not-parse-document)


<br>
<br>

---

<br>
<br>

2025 年 12 月 22 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>