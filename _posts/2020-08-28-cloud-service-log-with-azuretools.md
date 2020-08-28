---
title: "AzureTools を使って Cloud Service のログを採取する方法"
author_name: "Yo Motoya"
tags:
    - Cloud Service 
---

Cloud Service で発生したエラーの原因を調査する際、お客様にログの採取をお願いする場合があります。本記事では、AzureTools を使って Cloud Service のログを採取する方法について説明します。

Azure Tools では、Cloud Service 環境上の各種ログ、および構成情報を一括して採取することができます。採取できる情報は以下の通りです。
- Azure クラウド サービスを動作させているエージェント プロセスのログ
- Azure 診断ログ
- IIS アクセス ログ
- Windows イベント ログ
- デプロイされているアプリケーション アセンブリ
- デプロイ構成情報

英文のみのご提供となりますが、AzureTools の詳細については以下のページがございますのでご参考ください。<br>
[AzureTools - The Diagnostic Utility used by the Windows Azure Developer Support Team](http://blogs.msdn.com/b/kwill/archive/2013/08/26/azuretools-the-diagnostic-utility-used-by-the-windows-azure-developer-support-team.aspx)

# リモートデスクトップ接続の設定

対象インスタンス内で採取したログをローカル環境にダウンロードするために、リモートデスクトップ接続の設定を行い、対象インスタンスからローカルの C ドライブにアクセスできるようにします。以下の手順は Windows 10 の環境で行っています。

1. リモートデスクトップ接続 を立ち上げ、[ローカル リソース] タブに移動します。
2. [ローカル デバイスとリソース] の "詳細" をクリックします。
  ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-remote-desktop-connection1.png)
3. "OSDisk (C:)" にチェックを入れます。
  ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-remote-desktop-connection2.png)

設定は以上となります。上記設定を実施してリモートデスクトップ接続を行い、対象インスタンス内のエクスプローラーを開くと、以下のようにローカルの C ドライブにアクセスできる状況が確認できます。

![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-remote-desktop-connection3.png)

# AzureTools のインストール

調査対象のインスタンスに AzureTools をインストールする必要があります。デフォルトでは、Cloud Service にインストールされていません。

1. 調査対象のインスタンスにログインし、Windows PowerShell を管理者権限で起動します。
2. 以下のコマンドを実行し、Azure Tools を対象インスタンスにインストールします。

```
md c:\tools; Import-Module bitstransfer; Start-BitsTransfer http://dsazure.blob.core.windows.net/azuretools/AzureTools.exe c:\tools\AzureTools.exe
```
上記コマンドでは、`c:\tools` フォルダの作成および BitsTransfer モジュールのインストールを実施した後、BitsTransfer モジュールを使って、`AzureTools.exe` ファイルを `c:\tools` フォルダにダウンロードしています。

# ログの採取手順

1. ` c:\tools\AzureTools.exe` を実行し、C ドライブの tools フォルダに保存した `AzureTools.exe` を起動します。(Windows PowerShell や コマンドプロンプトなど任意の方法で構いません。)
2. Utils タブの “Auto Gather Logs” のリンクをクリックします。
  ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-azuretools1.png)
3. ウィザードを進めると、ログの収集がスタートし、zip ファイルが作成されます。作成された zip ファイルの場所がエクスプローラーで開きます。ログをご提供いただく場合は、その zip ファイルをお寄せください。
  ![image.png]({{site.baseurl}}/media/2020/08/2020-08-28-azuretools1.png)

<br>
<br>

---

<br>
<br>

2020 年 8 月 28 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>