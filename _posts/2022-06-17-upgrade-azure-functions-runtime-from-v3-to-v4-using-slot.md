---
title: "スロットを使った Azure Functions ランタイムのアップグレード"
author_name: "Yoichi Ozaki"
tags:
    - Function App
---

# はじめに
お世話になっております。App Service サポート担当の尾崎です。

.NET Core 3.1 は 2022 年 12 月 3 日 にサポートが終了することになっております。Azure Functions にてホストされている .NET Core 3.1 の関数コードアプリケーションは、その日を境に動作をしなくなる状況ではございません。

.NET Core 3.1 は Azure Functions ランタイムバージョン 3.x 内にて使用をしております。しかしながら、.NET Core 3.1 についての更新プログラムやカスタマーサービスの提供は終了いたします。Azure Functions ランタイムバージョンのサポート終了に伴い潜在的なサービス中断の恐れや、セキュリティの脆弱性の回避のためのアップデートの適応外となる可能性がございますため、2022 年 12 月 3 日までの、Functions ランタイムバージョン 4.x へのアップデートを推奨させていただいている状況でございます。

▼ [Extended support for Microsoft .NET Core 3.1 will end on 3 December 2022](https://azure.microsoft.com/ja-jp/updates/extended-support-for-microsoft-net-core-31-will-end-on-3-december-2022/) より引用

> On 3 December 2022, extended support for Microsoft .NET Core 3.1 will end. After that date, your applications that are hosted on Functions will continue to run and your applications will not be impacted. However, we'll no longer provide patches or customer service for .NET Core 3.1. Update your Functions applications to runtime version 4.x, which uses .NET 6.


そのため、.NET 6 への移行（それに伴う Azure Function ランタイムのバージョン 4 へアップグレード）を検討いただいているかと存じます。

本稿では App Service のスロット機能を使用して C# を用いた関数アプリの Azure Functions ランタイムをバージョン 4 へアップグレードする方法について解説します。

みなさまの関数アプリの滞りないアップグレードに役立ちましたら幸いです。

# スロットを使用した Azure Functions のランタイムのバージョン 4 へアップグレード手順
弊社提供の公開情報より検証を実施いたしました。

[スロットを使用した移行](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-versions?tabs=in-process%2Cazure-cli%2Cv4&pivots=programming-language-csharp#migration-with-slots)

## 【手順 0 】関数アプリの実装を用意

以下では一例として .NET Core 3.1 をフレームワークとして使用した関数アプリ（ランタイムバージョン 3.x ）を、.NET 6 を使用した関数アプリ（ランタイムバージョン 4.x ）にアップグレードするものとします。またすでに .NET Core 3.1 を使用した関数アプリがデプロイされている状況を前提とします。

- アップグレード前アプリ
    - `<プロジェクト名>.csproj` から使用しているフレームワークと Azure Functions ランタイムのバージョンを確認することができます。
    - わかりやすさのため今回実装する関数アプリは HTTP リクエストでトリガーされ、自身の使用しているフレームワークを表す文字列を返すものとします。

![before-3e983670-99f9-47fc-b61f-e3c492ff8091.png]({{site.baseurl}}/media/2022/06/before-3e983670-99f9-47fc-b61f-e3c492ff8091.png)

- アップグレード後アプリ

![after-f97cb789-b8b4-45de-8316-391093962e0b.png]({{site.baseurl}}/media/2022/06/after-f97cb789-b8b4-45de-8316-391093962e0b.png)

## 【手順 1 】すでにデプロイされている Azure Functions リソースに `stage` スロットを用意する

Azure Portal から `デプロイメント` ブレードを開き、 `＋スロットの追加` から `stage` スロットを追加します。

![prepare-stage-slot-22be0254-acd6-429c-8ab5-2cf7f4c503e3.png]({{site.baseurl}}/media/2022/06/prepare-stage-slot-22be0254-acd6-429c-8ab5-2cf7f4c503e3.png)

`stage` スロットを追加した後、アップグレード前のアプリ（今回の場合は .NET Core 3.1 を使用しているアプリ）を `stage` スロットにデプロイします。

デプロイの完了を確認したら、Azure Portal やブラウザを用いて `production`/`stage` 両スロットのアプリが正しく動作することを確認しておきましょう。

![before-swap-c12c3ae1-cdc3-4f98-8435-6eb1764648ca.png]({{site.baseurl}}/media/2022/06/before-swap-c12c3ae1-cdc3-4f98-8435-6eb1764648ca.png)

## 【手順 2 】`production` スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0` を設定する

次に行うことは `production` スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0` を設定することです。

`stage` スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0` を設定しスワップすることで `production` スロットにも設定します。こうすることで、（関数アプリの再起動はするがその後の）ウォーム アップが完了次第スロット間をスワップするので、アプリケーション設定の更新に伴うダウンタイムを回避することができます。

**`stage` スロットの** `構成` ブレードから `アプリケーション設定` タブ より `＋新しいアプリケーション設定` ボタンを押下し `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS` を `0` に設定します。

![set-sticky-on-stage-e2d7e76e-13de-47e1-883e-00e7d04ad42a.png]({{site.baseurl}}/media/2022/06/set-sticky-on-stage-e2d7e76e-13de-47e1-883e-00e7d04ad42a.png)

続けて `stage` スロットを `production` スロットとスワップします。

![swap-stage-and-production-1d507078-8035-45b9-8f4b-a5b9a605b44d.png]({{site.baseurl}}/media/2022/06/swap-stage-and-production-1d507078-8035-45b9-8f4b-a5b9a605b44d.png)

再び **`stage` スロットの** `構成` ブレードから `アプリケーション設定` タブ より `＋新しいアプリケーション設定` ボタンを押下し `FUNCTIONS_EXTENSION_VERSION` を `~3` に設定します。

![reset-functionsextensionversion-on-stage-13d41d47-6955-43da-bf7c-58fb16b75f77.png]({{site.baseurl}}/media/2022/06/reset-functionsextensionversion-on-stage-13d41d47-6955-43da-bf7c-58fb16b75f77.png)

同様に、`stage` スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0` を設定します。

![set-sticky-on-stage-one-more-time-fe271358-a686-46d9-a490-65e8559ea072.png]({{site.baseurl}}/media/2022/06/set-sticky-on-stage-one-more-time-fe271358-a686-46d9-a490-65e8559ea072.png)

最終的に `production`/`stage` 両スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0`、`FUNCTIONS_EXTENSION_VERSION=~3` が設定されることになります。

## 【手順 3 】アップグレード後のアプリ（今回の場合は .NET 6 アプリ）を `stage` スロットにデプロイする

`stage` スロットにアップグレード後のアプリ（今回の場合は .NET 6 アプリ）をデプロイします。 Visual Studio からアプリをデプロイする場合、次のようなメッセージが表示されますが、ここでは `はい(Y)` を選択しデプロイを行います。

![popup-on-vs-d54bfad1-e18c-41cf-b5a2-df32c5e85bf6.png]({{site.baseurl}}/media/2022/06/popup-on-vs-d54bfad1-e18c-41cf-b5a2-df32c5e85bf6.png)

デプロイが完了したら、`stage` スロットに `WEBSITE_OVERRIDE_STICKY_EXTENSION_VERSIONS=0` を再び設定します。これは先ほどのデプロイによって上書きされてしまっているからです。

![set-sticky-on-stage-again-86b43662-fe73-4c56-b511-6f0f415fd59c.png]({{site.baseurl}}/media/2022/06/set-sticky-on-stage-again-86b43662-fe73-4c56-b511-6f0f415fd59c.png)

次に `stage` スロットのアプリケーション設定に `FUNCTIONS_EXTENSION_VERSION=~4` を設定します。

![set-functionsextensionversions-to-stage-489c7e21-49c7-4758-af3d-b4b66540ff0d.png]({{site.baseurl}}/media/2022/06/set-functionsextensionversions-to-stage-489c7e21-49c7-4758-af3d-b4b66540ff0d.png)

> Visual Studio から関数アプリをデプロイする場合、アプリケーション設定の `FUNCTIONS_EXTENSION_VERSION` が （デプロイしたものに合わせて）`~4` に自動的に更新されます。

最後に、Windows 上で .NET 6 アプリを動かす場合は追加で次のコマンドを Cloud Shell から実行します。コマンド実行の応答として返されるJSONデータの中に `"netFrameworkVersion": "v6.0"` があれば正しく実行できたことが確認できます。


```shell
$ az functionapp config set --net-framework-version v6.0 -g <RESOURCE_GROUP_NAME>  -n <APP_NAME> --slot <SLOT_NAME>
```

![update-netframeworkversion-on-stage-78478e24-1be4-4e67-8d42-71879f84dc61.png]({{site.baseurl}}/media/2022/06/update-netframeworkversion-on-stage-78478e24-1be4-4e67-8d42-71879f84dc61.png)

最終的な `stage` スロットの設定は次のようになっています。

![app-settings-on-stage-9cea58cf-8508-4808-95ce-e670fc81b657.png]({{site.baseurl}}/media/2022/06/app-settings-on-stage-9cea58cf-8508-4808-95ce-e670fc81b657.png)

## 【手順 4 】`stage` スロットを `production` スロットとスワップ

最後に `stage` スロットを `production` スロットとスワップします。

![last-swap-b51b7552-0705-4f76-97f6-b3e965b3ebab.png]({{site.baseurl}}/media/2022/06/last-swap-b51b7552-0705-4f76-97f6-b3e965b3ebab.png)

実行結果を確認すると、正しく関数アプリのバージョンがアップデートされたことが確認できました。

![well-done-12e97a6d-2f47-45f1-a279-859bb03fb7f9.png]({{site.baseurl}}/media/2022/06/well-done-12e97a6d-2f47-45f1-a279-859bb03fb7f9.png)

# Azure Functions ランタイムのバージョンアップに際して気を付けるべきこと

上記手順以外にも次に挙げる二点に注意する必要があります。

- Function App の実装が参照するパッケージのバージョン
- Azure Functions Core Tools のバージョン

## Function App の実装が参照するパッケージのバージョン

Azure Functions のランタイムのバージョンを 3 から 4 へアップグレードするとき、お客様の実装する関数アプリの使用するフレームワークのバージョンも（例えば .NET Core 3.1 から .NET 6 へ）アップグレードすることになります。

その際、例えば C# な関数アプリ内で参照していたパッケージのうち、.NET 6 に対応していないパッケージがある場合は、.NET 6 対応版への変更が必要となる場合があります。

各言語におけるパッケージ／ライブラリの更新方法については、各ツールのドキュメントなどを参照して実行しましょう。

## Azure Functions Core Tools のバージョン

Azure Functions Core Tools (以下 Core Tools と記載します) とは ローカルコンピュータ上で関数アプリを実行・デバッグする際に用いるツールです。

Core Tools には Azure Functions のランタイムバージョンに対応する形で四つのバージョンがあり、Azure Functions のランタイムバージョンに対応するバージョンの Core Tools を利用する必要があります。

Core Tools のバージョンの切り替えについては、Core Tools の再インストール、もしくは非公式記事となり恐縮ですが Core Tools のバージョンを切り替えるツールが OSS にて提供されておりますので、利用をご検討ください。
	
Core Tools のバージョンの変更についての公式ドキュメントの記載はこちらになります。
- [Azure Functions Core Tools の操作](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash#changing-core-tools-versions)
	
Core Tools のバージョン切り替えツール
- [anthonychu/funcvm: Unofficial version manager for Azure Functions Core Tools (github.com)](https://github.com/anthonychu/funcvm)
- なお、切り替えツールにつきましては弊社が公式に提供しているものではない点、ご了承ください。
	

関数アプリのランタイムのアップグレード後のデプロイに際しては、ぜひローカル環境にて十分なテストを行った後、デプロイすることをお勧めいたします。

# 参考ドキュメント

- [Azure Functions ランタイム バージョンの概要](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-versions?tabs=in-process%2Cazure-cli%2Cv4&pivots=programming-language-csharp#migration-with-slots)


<br>

<br>

  

---

<br>

<br>

2022 年 06 月 21 日時点の内容となります。<br>

本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>

<br>