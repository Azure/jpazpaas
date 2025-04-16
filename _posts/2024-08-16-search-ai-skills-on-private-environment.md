---
title: "AI Search で OCR 等の組み込みスキルを Azure AI Service でパブリックアクセスを無効化した状態で実行したい"
author_name: "Takumi Nagaya"
tags:
    - AI Search
---

# 質問
AI Search のスキルセットで、AI マルチサービス リソースの OCR スキルを設定しています。  
しかし、AI マルチサービス リソースのパブリックネットワークアクセスを無効にするとエラーが発生しました。回避する方法はありますか?

# 回答
まず、本ブログは **Azure OpenAI の Embedding スキル以外** の、「AI マルチサービス リソース」によって提供される組み込みスキル (OCR スキル等) をターゲットとしております。Azure OpenAI の Embedding スキルをご利用の場合は以下のブログに詳細がございますので、参考になれば幸いです。<br/>
[Azure AI Search から Azure OpenAI Service へ可能な限りセキュアに接続したい](https://azure.github.io/jpazpaas/2024/01/25/search-private-access-to-openai.html)

AI マルチサービス リソースのパブリックネットワークアクセスを無効にする場合、共有プライベートリンクを利用することで、AI マルチサービス リソースの OCR 等の組み込みスキルを実行することが可能です。<br>
以下のドキュメントに記載があります。

>Azure AI マルチサービス アカウントへの接続のための共有プライベート リンクは (2024 年 11 月現在) サポートされるようになりました。 Azure AI 検索は、[課金のため](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-attach-cognitive-services)に Azure AI マルチサービスに接続します。 これらの接続は、共有プライベート リンクを介してプライベートにできるようになりました。 共有プライベート リンクは、スキルセット定義で、[マネージド ID (キーレス構成)](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-attach-cognitive-services#bill-through-a-keyless-connection) を構成している場合にのみサポートされます。<br>

[共有プライベート リンク経由で接続する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#supported-resource-types)

おおまかに 2 つの作業が必要となります。<br/>
1. **AI Search のマネージド ID を設定し、AI マルチサービス リソースへの操作に必要なロールを割り当て、設定したマネージド ID をスキル実行時に使用するようにスキルセットを構成します。**
2. **AI Search から AI マルチサービス リソースへの共有プライベートリンクを経由して接続するように設定します。**

## 1. AI マルチサービス リソースへのアクセスでマネージド ID を利用する。
[キーレス接続での課金](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-attach-cognitive-services?tabs=portal%2Cportal-remove#bill-through-a-keyless-connection) の手順にしたがいます。

## 2.  AI Search から AI マルチサービス リソースへの共有プライベートリンクを利用する設定
共有プライベートリンクの利用には以下の[前提条件](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#prerequisites) があります。

|ワークロード|階層の要件|リージョンの要件|サービス作成の要件|
|---|---|---|---|
|埋め込みスキルを含むスキルセット ([垂直統合](https://learn.microsoft.com/ja-jp/azure/search/vector-search-integrated-vectorization))|Basic 以上|[大容量リージョン](https://learn.microsoft.com/ja-jp/azure/search/search-limits-quotas-capacity#partition-storage-gb)|[2024 年 4 月 3 日より後](https://learn.microsoft.com/ja-jp/azure/search/vector-search-index-size#how-to-check-service-creation-date)|
|他の[組み込み](https://learn.microsoft.com/ja-jp/azure/search/cognitive-search-predefined-skills)またはカスタム スキルを使うスキルセット|Standard 2 (S2) 以上|なし|[2024 年 4 月 3 日より後](https://learn.microsoft.com/ja-jp/azure/search/vector-search-index-size#how-to-check-service-creation-date)|

つまり、[垂直統合](https://learn.microsoft.com/ja-jp/azure/search/vector-search-integrated-vectorization) を利用し、Azure OpenAI の 埋め込みスキルを含むスキルセットを利用し、2024 年 4 月 3 日より後に作成された Azure AI Search サービス、かつ大容量リージョンであれば、Basic 以上のレベルで共有プライベートリンクが利用可能ですが、<br/>
そうでない場合は、Standard 2 (S2) 以上のご利用が必要となりますので、ご注意ください。
<br/>
<br/>
<br/>
前提条件を満たしていれば、
[1 - 共有プライベート リンクを作成する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#1---create-a-shared-private-link) の手順にて、リソースの種類を `Microsoft.CognitiveServices/accounts`、サブリソースを `cognitiveservices_account` とします。

| リソースの種類 | サブリソース (またはグループ ID) |
| --- | --- |
| Microsoft.CognitiveServices/accounts | `cognitiveservices_account` |

[サポートされているリソースの種類](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create#supported-resource-types)


あとは [共有プライベート リンク経由で接続する](https://learn.microsoft.com/ja-jp/azure/search/search-indexer-howto-access-private?tabs=portal-create) のドキュメントに記載の内容にしたがってください。
<br>
<br>

---

<br>
<br>

2025 年 04 月 16 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>