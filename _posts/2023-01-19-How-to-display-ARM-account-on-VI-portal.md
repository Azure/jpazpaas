---
title: "ARM ベースのアカウントを Azure Video Indexer ポータルで表示させる方法"
author_name: "Yudai Kurashia"
tags:
    - Azure Media Services
    - Azure Video Indexer
---


# 質問
ARM ベースの Video Indexer アカウントを作成しましたが、Video Indexer ポータル上に該当アカウントが表示されません。どのようにすれば表示されますでしょうか？

# 回答
上記事象の原因として、下記の 2 点が考えられます。

1. Video Indexer ポータルにアクセスしたユーザーが、前提条件を満たせていないため
1. 異なるテナントのユーザーから、Video Indexer ポータルにログインしているため

下記にて各項目ごとにご案内いたします。

# Video Indexer のアカウントについて
各項目をご案内する前に、Video Indexer のアカウントについてご紹介いたします。

Video Indexer では、大きく下記の 3 つのアカウントがございます。
1. 試用版アカウント
1. 無制限アカウント（ARM ベースのアカウント）
1. 無制限アカウント（クラシックアカウント）

試用版アカウントでは、[Video Indexer ポータル](https://www.videoindexer.ai/) から最大 600 分、[開発者ポータル](https://api-portal.videoindexer.ai/) から最大 2400 分の index を無料で作成できます。（SLA に制限がある旨ご注意ください）


一方、無制限アカウントでは、無制限に index の作成が可能となります。無制限アカウントには ARM ベースのアカウントとクラシックアカウントの 2 種類がございますが、現在は ARM ベースのアカウントのご利用をお勧めしております。

Video Indexer ポータルから、各アカウントについて下記からご確認いただけます。
![video-indexer-image2-8db5d536-ccb3-4aeb-becb-c61915713c2a.png]({{site.baseurl}}/media/2023/01/video-indexer-image2-8db5d536-ccb3-4aeb-becb-c61915713c2a.png)

各アカウントの詳細につきましては、[Azure Video Indexer アカウントの種類](https://learn.microsoft.com/ja-jp/azure/azure-video-indexer/accounts-overview) に記載されておりますのでご参照ください。


## 回答 1. Video Indexer ポータルにアクセスしたユーザーが、前提条件を満たせていないため
ARM ベースの Video Indexer アカウントをご利用いただくためには、下記の前提条件を満たす必要があります。

![video-indexer-image1-1a7644bd-96db-4917-af1b-55bf4abb22b7.png]({{site.baseurl}}/media/2023/01/video-indexer-image1-1a7644bd-96db-4917-af1b-55bf4abb22b7.png)

[チュートリアル: Azure portal を使用して ARM ベースのアカウントを作成する](https://learn.microsoft.com/ja-jp/azure/azure-video-indexer/create-account-portal#prerequisites)


そのため、上記の前提条件を満たせていないユーザーで、Video Indexer ポータルへサインインしますと、ARM ベースの Video Indexer アカウントが表示されない事象が発生します。事象が発生しているユーザーで、上記の前提条件を満たせているかどうかご確認いただくことで、事象を解消できる可能性がございます。

リソースプロバイダーの追加方法、（Azure Media Service/マネージド ID への）各ロールの割り当て方法、につきましては、それぞれ下記のドキュメントに記載されておりますのでご参照ください。

[リソース プロバイダーの登録](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider-1)

[Azure portal を使用して Azure ロールを割り当てる](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/role-assignments-portal)

## 回答 2. 異なるテナントのユーザーから、Video Indexer ポータルにログインしているため
ARM ベースの Video Indexer アカウントを作成したユーザーが属するテナントと、Video Indexer ポータルにログインしたユーザーが属するテナントが異なる場合、Video Indexer ポータルに該当アカウントが表示されない事象が発生します。


そのため、Video Indexer ポータルにサインインしているユーザーが、正しいテナントに属しているかどうかご確認いただくことで、本事象を解消できる可能性がございます。具体的な手順につきましては、下記のドキュメントに記載されておりますので、ご参照ください。

[複数のテナント間で切り替える](https://learn.microsoft.com/ja-jp/azure/azure-video-indexer/switch-tenants-portal)



<br>
<br>

---

<br>
<br>

2023 年 01 月 19 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>