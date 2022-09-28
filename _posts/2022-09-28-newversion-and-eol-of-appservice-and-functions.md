---
title: "App Service, Functions における未サポートのバージョンと EOL について"
author_name: "Shogo Ohe"
tags:
    - app-service
    - function-app
---
Azure App Service ならびに Azure Functions (以降 App Service, Functions) では、弊社 (マイクロソフト) が提供するコンポーネント以外にも、オープンソースやコミュニティが提供する製品が組まれています。
こうした製品がリリース後、どの程度の期間で使用できるようになるのか、また EOL (End Of Life) を迎える場合についてご案内いたします。


# 新しいバージョンの展開について
App Service や Functions においてサポートされているプログラミング言語やアプリケーションサーバにおいて、各プロジェクトやコミュニティで新しいマイナーバージョンが提供された場合についてご説明いたします。

新機能を使用したい場合、不具合の修正対応、CVE で報告されている脆弱性への対応などの観点で「いつ新しいバージョンが使えるようになるか」といった形で、お問い合わせを多くいただいております。

## App Service, Functions におけるランタイムの構成について
前提として App Service, Functions ではどのように言語環境やミドルウェアを提供しているのかをご案内します。

App Service, Functions では実行基盤を共有しており、同じ OS イメージを使用しています。
Windows の場合、インスタンス (VM) を実行する OS イメージは共通であり、[サンドボックス](https://github.com/projectkudu/kudu/wiki/Azure-Web-App-sandbox) の中から OS イメージに含まれる各言語環境やミドルウェアにアクセスすることになります。
Linux の場合、インスタンスには docker engine を含むレイヤーが共通化されています。ただし、ユーザーはこの環境にアクセスすることはできず、App Service, Functions ともに各言語毎に提供される Docker イメージを使用することになります。

App Service, Functions が提供している OS の機能に関しては以下のドキュメントをご参照ください。<br />
- [Azure App Service におけるオペレーティング システムの機能](https://learn.microsoft.com/ja-jp/azure/app-service/operating-system-functionality)


まとめると、App Service, Functions では Windows, Linux 関係なく、提供される OS イメージにより実行できるランタイムが決まることになります。


## App Service, Functions での対応について
App Service, Functions では定期的にプラットフォームのアップデートを行っています。このアップデートでは、OS のパッチ適用や、利用可能な言語ランタイムやミドルウェアの更新、新機能の追加といった変更が含まれ、これらをまとめた OS イメージを作成します。
開発部門では、次期プラットフォームのアップデートの計画にあたり、提供する言語ランタイムやミドルウェアのマイナーバージョンを最初に確定します。

提供する OS のパッチレベルや、言語のマイナーバージョンなどの構成を凍結した上で、機能追加などを含むリグレッションテストを行い、OS イメージとしてビルドを行います。
ここで作成した OS イメージを使用して検証した結果、問題なく動作するかの確認を行っています。
また、[カナリア リージョン](https://azure.microsoft.com/ja-jp/blog/advancing-safe-deployment-practices/) と呼ばれる一般公開前の検証で使用されるリージョンでの動作、展開テストも行われます。

こうした理由から最初のバージョンの選定から、一般環境へのリリースまでのタイムラグが 2022 年時点で約 1~2 ヶ月程度存在します。さらに、リリースされた各種 OS イメージが全てのリージョンで使用可能になるまでにさらに 1 ヶ月以上かかる場合があります。

こうした背景から、App Service, Functions において使用される言語やミドルウェアの実行バージョンは、多くの場合リリースから 2~3 か月経過したものが使用されます。
また、コミュニティやプロジェクトが公開した直後の言語やライブラリ、ミドルウェアを提供することが難しくなっています。

新しくリリースされた製品が提供されるまでのイメージ図が以下になります。
![new-eol2.jpg]({{site.baseurl}}/media/2022/09/new-eol2.jpg)

# EOL (End of Life) を迎えた製品について
多くの製品開発に共通しますが、それぞれの製品では Lifecycle が設定されており、製品のリリースやサポート期限が決められています。
リリース後のしばらくは製品開発は行われますが、次期バージョンがリリースされた日以降のどこかで、すべてのメンテナンス (パッチ開発) が停止される End of Life と呼ばれる状態になります。

End of Life を迎えた製品は、新しい脆弱性に対する対応ができなくなる、技術的負債になる可能性があるなどの様々な理由から、利用は推奨されず、サポートのある製品への移行が強く推奨されます。
担当者の経験上も、オンプレミス環境では、特別延長保守契約や塩漬けといった形で、サポートレベルの低下することを許容し、可能な限り古いバージョンのまま使用する事例を見てきています。

App Service, Functions では PaaS として提供されているため、[プラットフォームの運用と健全性はマイクロソフトが責任を持っています](https://learn.microsoft.com/ja-jp/azure/security/fundamentals/shared-responsibility#division-of-responsibility)。
[Azure App Service のセキュリティ](https://learn.microsoft.com/ja-jp/azure/app-service/overview-security) にも記載がありますが、脆弱な実行環境を残しておくことが、お客様環境への侵入のきっかけとなってしまう可能性があることから適宜アップデートを行っています。

また、こうした背景から App Service, Functions では過去のバージョンを永続的に使用することはできず、EOL を迎えた製品は使用せず、適宜新しいバージョンに追従いただく必要がございます。


## EOL 以降におきること
App Service, Functions では多くの場合、EOL を迎える前に予め通知を行っています。一例をあげると以下のようなタイミングで通知が行われています。
- Node 14 LTS, EOL 2023年4月30日 (2022年3月に通知)
- Python 3.7, EOL 2021年12月23日 (2021年2月に通知)
- PHP 7.3, EOL 2021年12月6日 (2021年8月に通知)

EOL を迎えた製品を使用している環境に関して、サポートへのお問い合わせは受け付けは可能ですが、対応はベストエフォートとなります。
EOL を迎えた製品に関係しない一般的な内容に関しては対応は可能であるものの、EOL を迎えた製品に関連・または起因するお問い合わせの場合、対応をお断りする場合がございます。
また、EOL は Azure をご利用いただいております、すべてのお客様が対象です。特定のお客様のみ終了日を変更できないか、といったご要望も恐れ入りますが対応することは叶いません。
恐れ入りますが、お客様がご利用いただいております製品 (プログラミング言語のマイナーバージョン、ミドルウェア) に関してお客様にて EOL の予定を把握いただき、計画的に新しいバージョンへの移行を計画いただく必要がございます。

App Service, Functions では EOL 日以降のいずれかのタイミングで対象の言語環境やミドルウェアの削除を行います。
この削除は、先にご案内した OS イメージの更新とともに実行されるため、EOL 当日に行われることは基本的にはありません。

EOL を迎えた製品の削除までのイメージ図が以下になります。
![new-eol1.jpg]({{site.baseurl}}/media/2022/09/new-eol1.jpg)


<font color='red'>**また、重要度 A でお問い合わせいただいても旧バージョンの復旧などの対応はできません。**</font><br />
重要度 A でお問い合わせいただいた場合でも、新しいバージョンへの移行を、お客様にて実施いただく事になります。


## EOL にあたって良くいただくご質問
よくいただくご質問と回答を以下に記載いたします

**Q. EOL 日までにどういった対応が必要ですか？**<br />
A. 指定された EOL 日までにアプリケーションを新しいバージョンで動作するように改修、テストを行ってください。

**Q. 今実行しているアプリは EOL 日に動かなくなりますか？**<br />
A. いいえ。多くの場合、EOL の当日には何も起きず、多くの場合、翌日以降も動作はします。
ただし、EOL の翌日以降は別のバージョンに強制的に変更される、App Service, Functions のアップデートを通じてイメージを削除するといった操作が行われる可能性があり、アプリケーションが正常に動作しない可能性もあります。

**Q. EOL 日以降の実際に使用不可になる日を教えてもらえますか？**<br />
A. EOL 日以降はサポートの対象外であり、使用していない前提で対応が進むため、ご案内することは叶いません。
また、削除操作も全世界で一斉に実施されるわけではなく、各リージョン毎に (アップデートと同じタイミングで) 適宜実行されますため、ご案内は叶いません。

EOL 日以降も使用できることを前提に対応を進めず、EOL 日までに余裕をもって移行を行ってください。

**Q. 新しいバージョンに移行するためにはどうすればいいですか？**<br />
A. App Service, Functions リソースに対する対応と、リソース上で実行している、お客様の開発されたアプリケーションの 2 つの観点で対応が必要です。<br />
「App Service, Functions リソースに対する対応」は、新しいバージョンが動作するように設定を変更する操作になります。多くの場合、通知の中に変更方法のご案内が含まれておりますので、ご確認いただけますと幸いです。

以下にリンクの一例をご案内いたします。
- App Service, PHP 7.4 の EOL: [リンク](https://github.com/Azure/app-service-linux-docs/blob/master/Runtime_Support/php_support.md#end-of-life-for-php-74)
- App Service, JavaScript(Node) の EOL: [リンク](https://github.com/Azure/app-service-linux-docs/blob/master/Runtime_Support/node_support.md#how-to-update-your-app-to-target-a-different-version-of-node)
- Functions, JavaScript(Node) の EOL: [リンク](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-node?tabs=v2-v3-v4-export%2Cv2-v3-v4-done%2Cv2%2Cv2-log-custom-telemetry%2Cv2-accessing-request-and-response%2Cwindows-setting-the-node-version#setting-the-node-version)
- Functions, Python の EOL: [リンク](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-reference-python?tabs=asgi%2Capplication-level#python-version)
- Functions, PowerShell 7.0 の EOL: [リンク](https://github.com/Azure/azure-functions-powershell-worker/wiki/Upgrading-your-Azure-Function-Apps-to-run-on-PowerShell-7.2)

アプリケーションの改修 (新しいバージョンへの対応) に関してはお客様にてご確認ください。
多くの場合、バージョン間の移植は問題がないことが多いですが、使用しているライブラリや構文、内部的な処理が変わったことで、アプリ自体の動作が変わる可能性があります。
十分な準備の下でテストと修正を行ってください。

[ステージングスロット](https://learn.microsoft.com/ja-jp/azure/app-service/deploy-staging-slots) を使用して新しいバージョンでの動作テストや入れ替えなどを実施可能です。必要に応じてご利用をご検討いただけますと幸いです。
ステージングスロットは独立した環境になりますので、異なるランタイムバージョンを実行することが可能です (PHP 7.4 と 8.0、Node 14 と 16 等)。

<!--
**Q. **<br />
A.
-->

# Azure でサポートされていないバージョンを使用する方法
App Service, Functions で使用できるバージョンには範囲が決まっており、プロジェクトやコミュニティからリリースされた直後の製品や、 EOL を迎えた製品は使用できない点は先にご案内した通りとなります。

こうした場合に有効な方法は、Azure が提供するイメージを使用せず、お客様にてイメージを作成いただきご利用いただく方法があります。

- [Azure でカスタム コンテナーを実行する](https://learn.microsoft.com/ja-jp/azure/app-service/quickstart-custom-container?tabs=dotnet&pivots=container-linux-vscode)
- [カスタム コンテナーを使用して Linux で関数を作成する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-linux-custom-image?tabs=in-process%2Cbash%2Cazure-cli&pivots=programming-language-csharp)

こちらの方法を使用することで、App Service, Functions がサポートしていない (提供していない) バージョンの言語環境やミドルウェアを使用することが可能となります。
ただし、App Service, Functions が提供している Dockerfile や各種構成に関しては提供することは叶わず、既に提供されている環境を元にお客様にて構成いただく必要がございます。

こちらの方法は、EOL 後の塩漬け運用に使用した場合、脆弱性などが修正されず放置されることになるため、セキュリティ上のリスクが高まります。
あくまでも一時的な延命用とし、極力 EOL を迎えた後は、速やかにサポートのあるバージョンへ移行することを強くお勧めいたします。

また、カスタムコンテナーを使用された場合でも、アップデートは必要となります。
App Service, Functions が提供する環境を使用している場合、最新の環境で動作するように弊社が適宜更新や調整を行っています。
カスタムコンテナーを使用される場合、カスタムコンテナー内のベースイメージ、使用される言語ランタイムやミドルウェアに関してはお客様に管理責任があり、構成やバージョンの管理といった内容はお客様に管理いただく必要がございます。
大変恐縮ではございますが、お客様にてライフサイクルをご確認いただき、適宜アップデートや EOL への対応をを実施ください。


<br>
<br>

---

<br>
<br>

2022 年 09 月 28 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>