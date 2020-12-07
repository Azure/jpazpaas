---
title: "Operation list (grouped) テンプレートの編集方法について"
author_name: "Yo Motoya"
tags:
    - API Management
---

APIM の開発者ポータル（レガシ）をカスタマイズする方法として、テンプレートを利用する方法があります。管理者として開発者ポータル（レガシ）にアクセスし、カスタマイズアイコン内で用意されているテンプレートを編集する方法です。

[Azure API Management 開発者ポータルをテンプレートを使用してカスタマイズする方法](https://docs.microsoft.com/ja-jp/azure/api-management/api-management-developer-portal-templates)

本ブログでは、上記ドキュメントに記載されていない Operation list (grouped) テンプレートの編集方法について説明します。

# テンプレート編集時に発生するエラーについて

基本的には、他のテンプレートと編集方法は変わりません。カスタマイズアイコンをクリックして Operation list (grouped) をクリックすると、編集画面が表示されます。

![2020-12-07-operation-list-grouped.png]({{site.baseurl}}/media/2020/12/2020-12-07-operation-list-grouped.png)

しかし、その編集画面にてテンプレートを編集して保存する際、以下のようなエラーが発生する場合があります。

![2020-12-07-template-save-error.png]({{site.baseurl}}/media/2020/12/2020-12-07-template-save-error.png)

# Save Error エラーの回避方法

Operation list (grouped) テンプレートは、API の各操作をタグでグルーピングして表示する際にご利用いただけます。ご利用いただく場合は、少なくとも一つのタグを追加いただく必要があります。

![2020-12-07-apis-tag.png]({{site.baseurl}}/media/2020/12/2020-12-07-apis-tag.png)

タグを追加した後に、開発者ポータル（レガシ）の対象 API のページに遷移すると、Search フィルターが追加されています。Operation list (grouped) テンプレートを編集する場合は、以下画面キャプチャの赤枠の ”Group by tag” を選択してから、カスタマイズアイコン内の対象テンプレートを選択します。

![2020-12-07-group-by-tag.png]({{site.baseurl}}/media/2020/12/2020-12-07-group-by-tag.png)

この状態でテンプレートを編集すると、問題なく保存できることが確認できます。

![2020-12-07-template-saved.png]({{site.baseurl}}/media/2020/12/2020-12-07-template-saved.png)

# 開発者ポータル（レガシ）について

本記事では、開発者ポータル（レガシ）のテンプレートについて記載しておりますが、開発者ポータル（レガシ）は、2023 年 10 月に廃止される予定です。2023 年 10 月までは、これまで通りご利用いただけますが、新規機能の追加等はなく、セキュリティ更新のみ適応される状況となります。そのため、新しい開発者ポータルのご利用についても、ご検討いただければと思います。その際は、以下ページが参考になるかと思いますので、ご確認ください。

[新しい開発者ポータルへの移行](https://docs.microsoft.com/ja-jp/azure/api-management/developer-portal-deprecated-migration)

<br>
<br>

---

<br>
<br>

2020 年 12 月 7 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>