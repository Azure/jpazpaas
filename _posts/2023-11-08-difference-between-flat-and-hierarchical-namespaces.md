---
title: "フラット型名前空間と階層型名前空間が有効な BLOB の違いについて"
author_name: "Eiji Mocihda"
tags:
    - Azure Storage
---

# 質問
Azure Data Lake Gen2 の特徴として、階層型名前空間が追加された、とありますが、通常の BLOB もポータル等でみるとディレクトリができているから階層構造になっているように見えます。通常の BLOB と Azure Data Lake Gen2 の違いはどこにあるのですが。

[Azure Data Lake Storage Gen2 の階層型名前空間](https://learn.microsoft.com/ja-jp/azure/storage/blobs/data-lake-storage-namespace)

# 回答

質問にあるように、確かに Azure ポータルなどで BLOB を参照すると、ディレクトリがあるように見えます。

## 回答の前提
以下、例を出して説明します。まず該当のストレージアカウント配下にコンテナーが一つあることとします。そのコンテナーの配下には dir1 と dir2 という二つのディレクトリがあり、dir1 の下には test1.txt と test2.txt があり、dir2 の下には abc.txt と def.txt があるとします。
以下のようなイメージです。

コンテナー直下には dir1、dir2 が見えます。<br/>
![image-73caf6a8-0024-4c54-adb4-f754dfef525f.png]({{site.baseurl}}/media/2023/11/image-73caf6a8-0024-4c54-adb4-f754dfef525f.png)

dir1 の中には test1.txt、test2.txt が見えます<br/>
![image-ac398462-acb9-4438-9c0a-5712dcc1a6f3.png]({{site.baseurl}}/media/2023/11/image-ac398462-acb9-4438-9c0a-5712dcc1a6f3.png)

dir2 の中には abc.txt、def.txt が見えます<br/>
![image-b02d6fb6-e395-4b7b-96b9-020dab77a367.png]({{site.baseurl}}/media/2023/11/image-b02d6fb6-e395-4b7b-96b9-020dab77a367.png)

こちらを前提に通常の BLOB と Azure Data Lake Gen2 の違いについて説明します。

## BLOB と Azure Data Lake Gen2 の違い


まず、通常の BLOB の構造については以下のようなイメージとなります。<br/>
![image-b0bee377-28ca-4869-a1cd-365e7320c79c.png]({{site.baseurl}}/media/2023/11/image-b0bee377-28ca-4869-a1cd-365e7320c79c.png)

ポイントとしては、通常の BLOB については、フラット型名前空間になるため、ディレクトリという概念がなく、今回の例ですと以下のような名前の 4つの BLOB がコンテナー配下にある、ということになります。 


- dir1/test1.txt
- dir1/test2.txt
- dir2/abc.txt
- dir2/def.txt

"/" が含まれるとディレクトリのように見えますが、フラット型名前空間の場合は "/" も単なる文字になり、特別な意味はありません。ただ、Azure ポータルなどでは "/" をディレクトリ区切りのように見せて、ディレクトリのように見せているだけとなります。

次に、Azure Data Lake Gen2 の場合は以下のようなイメージとなります。<br/>
![image-4d281c6d-28f6-42ad-abf1-7c6d16a0ea24.png]({{site.baseurl}}/media/2023/11/image-4d281c6d-28f6-42ad-abf1-7c6d16a0ea24.png)


こちらは Azure ポータルの見た目と同様にディレクトリがある階層構造となります。
コンテナー配下に dir1 と dir2 というディレクトリがあり、dir1 の中に test1.txt、test2.txt、dir2 の中に abc.txt、def.txt が存在している、という構成となります。

これらの構造の違いによって、通常の BLOB と Azure Data Lake Gen2 において、以下のような違いが出ます。

- 通常の BLOB の場合はディレクトリを削除、という操作はできません。今回の例ですと、仮に通常の BLOB で dir1 を消したい、というときには dir1/ で始まる BLOB を全部削除する、という操作になります。一方 Azure Data Lake Gen2 の場合はディレクトリ、という概念があるため、dir1 というディレクトリを削除する、という操作ができます。ポータル上ですとどちらのパターンでも dir1 というディレクトリを削除という操作はできますが、具体的なストレージへの操作内容は異なることになります。
- 通常の BLOB はディレクトリという概念がないため、dir1 という空のディレクトリを作るということはできません。dir1/test1.txt というような BLOB が存在することで、はじめてポータル上で dir1 というディレクトリがあるような見え方をします
- 以下の画像のようにAzure Storage Explorer ですと、通常の BLOB でも、ディレクトリを作るという操作ができます。しかし通常の BLOB の場合においてはこの操作は一時的にディレクトリがいるように見せかけているだけで、作ったディレクトリ配下に何かしらの BLOB を配置しないと、ディレクトリだけ存在させる、ということはできません。以下、通常の BLOB で"新しいフォルダー" をクリックすると、作成するフォルダー名を指定する画面で以下のような注意書きもご確認いただけます。

![image-6dd2b5a9-5ac9-404f-b32e-8ea0acd0010f.png]({{site.baseurl}}/media/2023/11/image-6dd2b5a9-5ac9-404f-b32e-8ea0acd0010f.png)

![image-0b3e93c0-5caa-4468-a300-8c610478aab2.png]({{site.baseurl}}/media/2023/11/image-0b3e93c0-5caa-4468-a300-8c610478aab2.png)

## まとめ
Azure ポータルなどでコンテナーの内容を確認すると、一見、通常の BLOB と Azure Data Lake Gen2 の両方ともディレクトリをもったような構造に見えますが、実際の構造はこのような違いがある、ということはご理解頂ければ幸いです。


# 参考ドキュメント
[Azure Data Lake Storage Gen2 の階層型名前空間](https://learn.microsoft.com/ja-jp/azure/storage/blobs/data-lake-storage-namespace)

<br>
<br>

---

<br>
<br>

2023 年 10 月 30 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>