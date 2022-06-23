---
title: "Azure Functions v2,v3 ランタイムのサポート終了について(追跡 ID MT12-HD0)"
author_name: "Shogo Ohe"
tags:
    - Function App
---

Azure Functions をお使いいただいております環境において、追跡 ID MT12-HD0 として以下のアナウンスメントが行われています。
今回はこのアナウンスメントに関して、多数のお問い合わせをいただいておりますので、よくお問い合わせ頂くご質問と回答をご案内いたします。

```
最終更新日 (2022-06-13T19:25:05.3668203Z)

You're receiving this notice because you have Azure Functions apps that use runtime version 2.x or 3.x.

On 3 December 2022, .NET Core 3.1 will be retired. As a result, Azure Functions runtime versions 2.x and 3.x, which use .NET Core 3.1, will also be retired on that date.

Your Functions applications that use runtime version 2.x or 3.x will continue to run and your existing workloads won't be affected. However, we'll no longer provide updates or customer service for applications that use these versions.

Functions runtime version 4.x includes:

    The latest security updates.
    Support for the latest programming languages such as .NET 6 and Node 16.

Recommended action

To avoid potential service disruptions or security vulnerabilities, [update your Functions applications to use runtime version 4.x](https://aka.ms/functions-view-and-update-runtime-version) before 3 December 2022.
```

## 前提：Azure Functions のアーキテクチャについて
Azure Functions のアーキテクチャは、大きく分けて Functions Host と Language Worker の 2 つのコンポーネントから構成されます。

HTTP Trigger や Blob トリガーなどのイベント検知や各種ログの記録といった共通処理を Functions Host が行っております。Functions Host が検知したイベントを gRPC を通じて (Language Worker 上で実行されている) お客様が作成されたコードを呼び出すことで、Azure Functions は成り立っています。

Functions Host に関しては以下のプロジェクトで開発が行われておりますので、ご参考となれば幸いです。<br />
  [github Azure/azure-functions-host](https://github.com/Azure/azure-functions-host)

Azure Functions の Language Worker や gRPC に関しては以下の記事をご確認ください。<br />
- [Language Extensibility](https://github.com/Azure/azure-functions-host/wiki/Language-Extensibility)
- [Helpful links: GitHub repositories](https://github.com/Azure/Azure-Functions#github-repositories)

## なぜ Azure Functions v2.x, v3.x ランタイムが廃止されるのですか？
Azure Functions v2.x, v3.x ランタイム (正確には Functions Host コンポーネント) はそれぞれ、.NET Core 2.2 と 3.1 を実行基盤として使用してきました。
.NET Core 2.2 は 2019年12月23日 にサポートが終了しており、.NET Core 3.1 は 2022年12月3日 にサポートが終了いたします。

[ライフサイクル - Microsoft .NET および .NET Core](https://docs.microsoft.com/ja-jp/lifecycle/products/microsoft-net-and-net-core)

.NET Core 3.1 のサポートが終了するため、Functions Host 機能の継続提供を行うことが困難となりますため、2022年12月3日を持って、Azure Functions v2.x, v3.x のサポート並びに提供が終了いたします。


## 私たちが使用している Node.js 12 や Java 11 はまだサポート期間内なのに Functions v2.x, v3.x ランタイムで使用できなくなるのはなぜですか？

今回の .NET Core 3.1 のサポート終了に伴い、イベントの検知を行う Functions Host の使用が継続できなくなるため、イベントの監視や各プログラミング言語で作成されたコードの呼出も行うことが出来なくなります。そのため、まだコミュニティが開発を継続しているプログラミング言語、バージョンであっても、ランタイムバージョン v2.x, v3.x での使用が出来なくなります。


## 私が使用している Functions は対象になりますか？
恐れ入りますが、対象かどうかに関しては、ポータルを通じて、お客様にてご確認いただくことが可能です。

具体的には以下で確認可能です。
ポータルから該当の Functions を確認し、[構成] > [関数のランタイム設定] にて、ランタイムバージョン "~2", "~3" といった表記があった場合が対象です。

![image-d68e050c-b6a8-41fd-8a1f-c3e402e957cb.png]({{site.baseurl}}/media/2022/06/image-d68e050c-b6a8-41fd-8a1f-c3e402e957cb.png)

または [構成] > [アプリケーション設定] の FUNCTIONS_EXTENSION_VERSION で "~2", "~3", "2.x", "3.x" といったバージョンが指定されている場合が対象です。

![image-5183d856-a05c-4bf0-a3ed-0cb38eceacab.png]({{site.baseurl}}/media/2022/06/image-5183d856-a05c-4bf0-a3ed-0cb38eceacab.png)

また、Azure CLI や Azure PowerShell などを通じて確認が可能ですので、リソースが複数ある場合はこちらの使用もご検討ください。

[az functionapp config appsettings list](https://docs.microsoft.com/ja-jp/cli/azure/functionapp/config/appsettings?view=azure-cli-latest#az-functionapp-config-appsettings-list)<br />
 例: az functionapp config appsettings list --name MyWebapp --resource-group MyResourceGroup  

Azure CLI での出力例：
```
[
  ...
  {
    "name": "FUNCTIONS_EXTENSION_VERSION",
    "slotSetting": false,
    "value": "~3"
  },
  ...
]
```

[Get-AzFunctionAppSetting](https://docs.microsoft.com/ja-jp/powershell/module/az.functions/get-azfunctionappsetting?view=azps-8.0.0)<br />
 例: Get-AzFunctionAppSetting -Name MyAppName -ResourceGroupName MyResourceGroupName

Azure PowerShell での出力例：
```
Name                           Value
----                           -----
 :
FUNCTIONS_EXTENSION_VERSION    ~3
 :
```

## 2022年12月3日を過ぎたら、私の作成した Functions v2.x,v3.x で動作する関数は動かなくなりますか？
2022年12月3日に予定されているのは、サポートの終了のみとなります。<br />
そのため、現時点では 2022年12月3日に Functions v2.x,v3.x の実行環境が突然停止するといったことはなく、2022年12月3日を過ぎても、ある程度の期間は Functions v2.x,v3.x での実行は可能である可能性が高いと考えられます。

ただし、Azure Functions では、提供の終了したバージョンやモジュールは定期的に削除を行っております。
そのため、2022年12月3日以降のどこかのタイミングで、Functions v2.x,v3.x や .NET Core 3.1 が削除される可能性があります。

削除のタイミングなどについては現時点では未定となります。実際の削除作業は、サポート並びに提供が終了後の対応となりますため、事前または事後の情報提供は行われない可能性があります。弊社テクニカルサポートにお問い合わせいただきましても、削除予定時期に関しては回答いたしかねます。


## .NET Framework を使用して Functions v1.x を使用していますが、影響を受けますか？
いいえ。現時点で対応が予定されているのは、Azure Functions v2.x, v3.x 系のランタイムのみです。

Functions 1.x は .NET Framework 4.8 で動作しています。
今回のサポート終了は .NET Core 3.1 となりますので、影響は受けません。

また、.NET Framework のサポート終了は宣言されておらず、Azure Functions におけるサポート終了時期も現時点においては未定です。<br />
[Microsoft .NET Framework](https://docs.microsoft.com/en-us/lifecycle/products/microsoft-net-framework)


## Functions v4.x への移行について
Functions v2.x, v3.x を使用いただいている場合、Functions v4 への移行が必要となり、ご利用いただいておりますプログラミング言語に応じて対応が変わります。

例えば Functions v2.x で .NET Core 2.1 や Node.js 8 を使用している場合は、Functions v4.x を使用するにあたり、.NET 6.0 や Node 14, 16 にアップデートいただく必要がございます。
一方で、すでに Funcitons v3.x で Node 14 や Java 11 を使用されている場合、ランタイムを Functions v4.x に変更するだけでそのまま動作する可能性もございます。

詳細は以下のドキュメントに記載がございますのでご確認いただけますと幸いです。<br />
[Languages](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-versions?tabs=in-process%2Cazure-cli%2Cv4&pivots=programming-language-csharp#languages)

移行後は、想定した動作になっているか、お客様にて動作確認を行っていただけますと幸いです。

---

<br>
<br>

2022 年 06 月 24 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>