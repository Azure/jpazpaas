---
title: "Azure Functions ランタイムにおける潜在的な重大変更（2025年10月）"
author_name: "Shota Nakano"
tags:
    - Function App
---
## はじめに 
お世話になっております。App Service サポート担当の仲野です。いつも Azure Functions（関数アプリ）をご利用いただき誠にありがとうございます。

本記事は [Potential breaking change for older versions (October 2025)](https://github.com/Azure/app-service-announcements/issues/503) にてアナウンスされた内容の日本語抄訳です。最新情報につきましてはリンク先の Github の Issue をご確認ください。

2025 年 10 月に Azure Functions のプラットフォームのアップデートが予定されています。それに伴い、マイナー ランタイム バージョンに固定されている Azure Functions が正常に動作しなくなる潜在リスクがあります。

以下に当該アップデートにより潜在的なリスクが存在するバージョンと対策について解説します。<br/>

## 影響を受ける可能性のあるバージョン情報と対応内容

プラットフォームのアップデートに伴い、特定のセキュリティ機能がない古いランタイム バージョンで動作する Azure Functions に影響を及ぼす可能性があります。現時点ではプラットフォームの変更は、**2025年10月28日**に有効になる見込みです。そのため当該アップデート以前に Azure Functions のランタイムの更新をすることを推奨します。

下記の表に示すランタイム バージョンで動作している Azure Functions につきましては、直ちに対策をご検討ください。

| ランタイム メジャー バージョン | ランタイム バージョン | 対応内容 |
|-------------|----|----------------|
| v1.x | 1.0.20776.0 未満| 常に最新の 1.x バージョンを使うようにアプリを更新してください。1.x のサポートは 2026年9月14日に終了します。できるだけ早く[4.x への移行](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-version-1-version-4)を行ってください。 |
| v3.x | 3.20.0.0 未満 | 直ちに[4.x への移行](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-version-3-version-4)を行ってください。2.x および 3.x のサポートは 2022年12月13日に終了しています。暫定対処として最新の 3.x へ更新できますが、4.x への移行をご検討ください。 |
| v4.x | 4.24.0.0 未満 | 常に最新の 4.x バージョンを使うようにアプリを更新してください。 |

影響を受ける可能性があるランタイム バージョンで動作する Azure Functions をご利用中の場合、メールによる通知がされ、App Service Advisor で新たな推奨事項が作成されます。Azure ポータルの「診断と問題の解決」ビューで推奨事項を確認できます。

## 重要事項
前述の表に記載したバージョン範囲は、今回の変更のためにフラグが立てられたものですが、記載されたランタイム バージョンを「最低限必要なバージョン」と解釈しないようご注意ください。他にもセキュリティ修正が後のバージョンに含まれています。古いバージョンに固定されている Azure Functions は必ず更新すべきであり、1.0.20776.0、3.20.0.0、4.24.0.0 に移行しても十分ではありません。これらの範囲は、影響を受ける可能性のある Azure Functions を特定するための目安となります。

ランタイム バージョンにつきましては、原則マイナー バージョンに固定せず、メジャー バージョンのみを指定し、最新バージョンへ移行するよう努めてください。


## 対策方法
### 常に最新バージョンで動作させる方法

Azure Functions が古いバージョンで動作している主な理由は、カスタム コンテナー イメージの基本レイヤーが更新されていないことが多いです。

カスタム コンテナー イメージを最新の 4.x ランタイムで動作させるには、以下の表にある適切な [基本イメージ タグ](https://mcr.microsoft.com/catalog?search=functions) をプルし、カスタム イメージを定期的にリビルドして、新しいバージョンを使用してください。

| 言語スタック | 推奨される基本イメージ タグの例 |
|--------------|-------------------------|
| .NET（分離ワーカー モデル） | `mcr.microsoft.com/azure-functions/dotnet-isolated:4-dotnet-isolated8.0`<br/>または<br/>`mcr.microsoft.com/azure-functions/dotnet-isolated:4-dotnet-isolated8.0-appservice`<br/>(これらの例は .NET 8 を対象とします。必要な .NET バージョンに適したイメージを選択します)。 |
| .NET（レガシ インプロセスモデル） | `mcr.microsoft.com/azure-functions/dotnet:4-dotnet8.0`<br/>または<br/>`mcr.microsoft.com/azure-functions/dotnet:4-dotnet8.0-appservice`<br/>(インプロセス モデルのサポートは 2026 年 11 月 10 日に終了します。できるだけ早く[分離ワーカー モデルへの移行](https://learn.microsoft.com/ja-jp/azure/azure-functions/migrate-dotnet-to-isolated-model)する必要があります) |
| Java | `mcr.microsoft.com/azure-functions/java:4-java21`<br/>または<br/>`mcr.microsoft.com/azure-functions/java:4-java21-appservice`<br/>(これらの例は Java 21 を対象とします。必要な Java バージョンに適したイメージを選択します)。 |
| Node.js（JavaScript または TypeScript） | `mcr.microsoft.com/azure-functions/node:4-node22`<br/>または<br/>`mcr.microsoft.com/azure-functions/node:4-node22-appservice`<br/>(これらの例では、Node.js 22 を対象とします。必要な Node.js バージョンに適したイメージを選択します)。 |
| PowerShell | `mcr.microsoft.com/azure-functions/powershell:4-powershell7.4`<br/>または<br/>`mcr.microsoft.com/azure-functions/powershell:4-powershell7.4-appservice`<br/>(これらの例は PowerShell 7.4 を対象とします。必要な PowerShell バージョンに適したイメージを選択します)。 |
| Python | `mcr.microsoft.com/azure-functions/python:4-python3.12`<br/>または<br/>`mcr.microsoft.com/azure-functions/python:4-python3.12-appservice`<br/>(これらの例は Python 3.12 を対象とします。必要な Python バージョンに適したイメージを選択します)。)|
| カスタムハンドラー/その他 | `mcr.microsoft.com/azure-functions/base:4`<br/>または<br/>`mcr.microsoft.com/azure-functions/base:4-appservice` |

詳細は [カスタム コンテナーのメンテナンス](https://learn.microsoft.com/ja-jp/azure/azure-functions/container-concepts#maintaining-custom-containers) を参照してください。

また、Azure Functions の設定で古いバージョンに固定している場合もあります。
4.x ランタイムの最新バージョンを対象とするには、アプリ設定`FUNCTIONS_EXTENSION_VERSION` の値を `~4` と設定してください。

詳細は [Azure Functions ランタイム バージョンをターゲットにする方法](https://learn.microsoft.com/ja-jp/azure/azure-functions/set-runtime-version) を参照してください。

---

<br>
<br>

2025 年 07 月 31 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>