---
title: "Azure Functions インプロセス モデル のサポート終了について(追跡 ID FVN7-7PZ)"
author_name: "Kohei Mayama"
tags:
    - Function App
---

Azure Functions をお使いいただいております .NET のインプロセスモデル環境において、追跡 ID FVN7-7PZ として以下のアナウンスメントが行われています。 今回はこのアナウンスメントに関して、多数のお問い合わせをいただいておりますので、よくお問い合わせ頂くご質問と回答をご案内いたします。

**2024 年 3 月現在のアナウンスメント**
```
Beginning 10 November 2026, the in-process model for .NET apps in Azure Functions will no longer be supported. To ensure that your apps that use this model continue being supported, you’ll need to transition to the isolated worker model by that date.
 
You may still use your .NET apps with the in-process model beyond 10 November 2026, but they will no longer receive security and feature updates from Microsoft.
```

## 概要
2026 年 11 月 10 日より、Azure Functions .NET アプリケーションの インプロセスモデル がサポート終了となります。
インプロセスモデルにてお使いの .NET の関数コード アプリケーションを引き続き製品サポートを受ける場合には、分離ワーカー プロセスモデル への変更をする必要がございます。2026 年 11 月 10 日以降も、インプロセスモデルを使用した .NET の関数コード アプリケーションを使用することはできますが、Microsoft からのセキュリティおよび機能アップデートは受けられなくなります。


## Azure Functions アーキテクチャ と インプロセスモデル について
インプロセスモデルを理解する上で Azure Functions アーキテクチャが必要となります。Azure Functions が構成するプロセスは、大きく分けて 2 つございます。ホストプロセス（Functions Host）と呼ばれる関数アプリを制御するプロセスと 言語ワーカー（Language Worker）と呼ばれる関数アプリにて指定されたランタイムスタック（Ex. .NET, Java, JavaScript, Python）の関数コードを動作するプロセスに分かれております。
Azure Functions アーキテクチャの詳細につきましては、以下の弊社サポートブログよりご参考ください。

[Azure Functions（関数アプリ）の内部アーキテクチャ概要や関連する用語について](https://azure.github.io/jpazpaas/2023/08/24/azure-functions-words-relative-management.html)

Azure Functions では、Functions Host と Language Worker のプロセスに分かれております。しかしながら、.NET のインプロセスモデル を利用する場合には、Functions Host と Language Worker は同じプロセスとして動作をします。

**インプロセスモデルの概略図**

![image-9430a7b2-3df3-4619-a39d-c7f2634594fa.png]({{site.baseurl}}/media/2024/04/image-9430a7b2-3df3-4619-a39d-c7f2634594fa.png)


## 背景
Azure Functions の .NET バージョンの開発プロセスとしては、分離ワーカープロセス から適用が行われ、ユーザへの提供が行われます。以下の Azure Functions のロードマップにて .NET 8 の 分離ワーカープロセスモデルとして 一般提供しておりますが、インプロセスモデルでの同等機能は同時期には提供されません。また、分離ワーカプロセス モデルを推奨しており、将来的にはインプロセス モデルをサポートしなくなることが挙げられます。そのため、アプリケーションコード変更もあり、お手数おかけして申し訳ございませんが、将来性を鑑み分離ワーカプロセスモデルへの移行をご検討ください。

[.NET on Azure Functions – August 2023 roadmap update](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/net-on-azure-functions-august-2023-roadmap-update/ba-p/3910098)

![image-b0152014-438a-48aa-ae36-4e0755d028a0.png]({{site.baseurl}}/media/2024/04/image-b0152014-438a-48aa-ae36-4e0755d028a0.png)


[.NET on Azure Functions – March 2024 roadmap update](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/net-on-azure-functions-march-2024-roadmap-update/ba-p/4097744)


また、分離ワーカープロセス モデルを推奨する理由としては、Functions Host と Language Worker のプロセスに分かれておりますため、アセンブリの競合が少ないことや依存関係の挿入をすることが出来ます。詳細は以下の弊社提供の公開情報にてご案内をしております。

[分離ワーカー モデルの利点](https://learn.microsoft.com/ja-jp/azure/azure-functions/dotnet-isolated-process-guide?tabs=windows#benefits-of-the-isolated-worker-model)

---

## 対象となる 関数アプリ リソースを特定する
今回の通知に対しての対応は以下の 2 パターンが考えられます。以下の移行する関数アプリを特定するに従い、まずはインプロセスモデルをご利用の 関数アプリ　リソースが無いかご確認ください。

### Azure Portal を使用した確認方法
対象となる 関数アプリ リソースの Azure Portal より、「環境変数」> 「アプリ設定」にて `FUNCTIONS_WORKER_RUNTIME` の値が `dotnet` であることを確認します。

![image-13d350ca-ebc3-4571-9818-18abc5616696.png]({{site.baseurl}}/media/2024/04/image-13d350ca-ebc3-4571-9818-18abc5616696.png)

### Azure PowerShell を使用した確認方法
Azure PowerShell スクリプトを使用して、サブスクリプション内でインプロセスモデルを利用している関数アプリ リソースの一覧を出力することができます。詳細な方法につきましては、以下の弊社提供の公開情報にてご案内しておりますためご参考ください。

[移行する関数アプリを特定する](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8#identify-function-apps-to-migrate)



## 対象となる 関数アプリ リソースにて移行を開始する
前項のインプロセスモデルをご利用の環境がある場合には 1. を、分離プロセスモデルを利用なものの .NET6 である場合には 2. のご対応をお願いいたします。

### 1. 対応前の状態が .NET6 の インプロセスモデル をご利用の場合
以下の 2 つの手順を実施いただく必要がございます。.NET6 のインプロセスモデルから .NET8 の分離プロセスモデルへの移行となるため、Azure Functions の環境上の変更（Azure Functions で言語スタックのバージョンを更新する）と実際に動作するアプリケーションの変更（.NET アプリをインプロセス モデルから分離ワーカー モデルに移行する）を実施いただく必要がございます。

- [Azure Functions で言語スタックのバージョンを更新する](https://learn.microsoft.com/ja-jp/azure/azure-functions/update-language-versions?tabs=azure-portal%2Cwindows&pivots=programming-language-csharp)

- [.NET アプリをインプロセス モデルから分離ワーカー モデルに移行する](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model?tabs=net8)

### 2. 対応前の状態が .NET6 の 分離プロセスモデル をご利用の場合
以下の手順を実施いただく必要がございます。.NET6 の分離プロセスモデルから .NET8 の分離プロセスモデルへの移行となるため、Azure Functions の環境上の変更（Azure Functions で言語スタックのバージョンを更新する）のみを実施いただく必要があるためでございます。

- [Azure Functions で言語スタックのバージョンを更新する](https://learn.microsoft.com/ja-jp/azure/azure-functions/update-language-versions?tabs=azure-portal%2Cwindows&pivots=programming-language-csharp)

<br>
<br>

---

<br>
<br>

2024 年 04 月 01 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>